# TP2 : Missions — Sécurité du runtime Docker et isolation

**Durée : 3h30**

> **Contexte :** Les conteneurs s'exécutent aujourd'hui avec trop de privilèges et sans limites de ressources. Un conteneur compromis peut provoquer un déni de service (DoS) ou s'échapper vers l'hôte. Dans ce TP, vous allez d'abord *constater* ces risques dans un environnement volontairement vulnérable, puis les corriger méthodiquement.

---

## Mise en place — Conteneurs de démonstration

Créez les deux fichiers suivants. Le premier représente un conteneur **intentionnellement non protégé** ; le second servira à démontrer l'effet du `pids_limit` seul, avant d'aborder le durcissement complet.

**`docker-compose.vuln.yml`**

```yaml
services:
  demo-vuln:
    image: ubuntu:22.04
    container_name: demo-vuln
    # Aucune limite CPU ni mémoire
    # Aucune restriction de capabilities
    # Aucune limite de processus
    command: sleep infinity
    stdin_open: true
    tty: true

  demo-secure:
    image: ubuntu:22.04
    container_name: demo-secure
    # Même image, mais avec une limite de processus
    pids_limit: 20
    command: sleep infinity
    stdin_open: true
    tty: true

  node-exporter:
    image: prom/node-exporter:v1.7.0
    container_name: node-exporter
    ports:
      - "9100:9100"
    restart: unless-stopped
```

Lancez-le :

```bash
docker compose -f docker-compose.vuln.yml up -d
docker ps
```

---

## Mission 2.1 — Simulation d'un DoS par Fork Bomb

> **Objectif :** Comprendre pourquoi un conteneur sans limites peut mettre à genoux un serveur entier, et observer cet impact en temps réel.

### Étape 1 — Ouvrir le monitoring en parallèle

Avant de lancer quoi que ce soit, ouvrez **deux terminaux** :

**Terminal A — surveillance en temps réel :**

```bash
# Affichage live des ressources de tous les conteneurs (rafraîchi toutes les secondes)
watch -n 1 docker stats
```

**Terminal B — métriques système :**

```bash
# Charge CPU et nombre de processus en temps réel
watch -n 1 "cat /proc/loadavg && echo '---' && ps aux --no-header | wc -l"
```

Vous pouvez aussi garder Grafana ouvert sur `http://localhost:3000` pour observer les courbes monter.

### Étape 2 — Entrer dans le conteneur vulnérable

```bash
docker exec -it demo-vuln bash
```

### Étape 3 — Lancer la Fork Bomb dans `demo-vuln`

> **⚠️ Sécurité :** La commande suivante doit être exécutée **uniquement dans le shell du conteneur**, jamais directement sur votre machine hôte. Votre hôte Ubuntu dispose de sa propre limite noyau (généralement 32 768 processus) qui empêchera un crash total — mais si votre machine commence à freezer, ouvrez un troisième terminal et exécutez `docker kill demo-vuln`.

```bash
# Commande à exécuter DANS le conteneur demo-vuln
:(){ :|:& };:
```

Cette one-liner définit une fonction `:` qui s'appelle elle-même deux fois en parallèle, créant une explosion exponentielle de processus.

### Étape 4 — Observer l'impact

Dans le Terminal A (`docker stats`), regardez la colonne `PIDS` exploser pour `demo-vuln`. Dans le Terminal B, observez la charge CPU (`loadavg`) et le nombre total de processus sur l'hôte augmenter.

**Après 15 à 20 secondes**, depuis un troisième terminal :

```bash
docker kill demo-vuln
docker compose -f docker-compose.vuln.yml up -d
```

### Étape 5 — Re-tenter sur `demo-secure` (avec `pids_limit: 20`)

```bash
docker exec -it demo-secure bash
# Dans le conteneur :
:(){ :|:& };:
```

Observez dans `docker stats` : la colonne `PIDS` de `demo-secure` ne dépasse pas 20. Le conteneur peut devenir instable ou s'arrêter, mais **l'hôte et `node-exporter` ne sont pas affectés**.

```bash
# Depuis un autre terminal, vérifier que node-exporter répond toujours
curl -s http://localhost:9100/metrics | head -5
```

**Questions d'analyse :**

- Qu'avez-vous observé dans `docker stats` pendant la fork bomb sur `demo-vuln` ? La colonne `PIDS` avait-elle une limite ?
- Quelle différence constatez-vous sur `demo-secure` avec `pids_limit: 20` ?
- Node Exporter a-t-il continué à répondre sur `:9100` pendant l'attaque sur `demo-vuln` ? Pourquoi ?
- Pourquoi le monitoring cesse-t-il de répondre en premier dans un scénario réel sans isolation ?
- Quel serait l'impact métier si ce serveur était mutualisé entre plusieurs équipes en production ?

---

## Mission 2.2 — Application du principe du moindre privilège

> **Objectif :** Comprendre les capabilities Linux et appliquer des limites strictes (CPU, RAM, processus, capabilities) pour qu'un conteneur compromis ne puisse pas nuire au-delà de son périmètre.

### Contexte — Qu'est-ce qu'une capability Linux ?

Par défaut, Docker accorde aux conteneurs un ensemble de **capabilities** (droits noyau granulaires) bien plus large que nécessaire pour la plupart des applications. Parmi les plus dangereuses accordées par défaut :

| Capability | Ce qu'elle permet |
|---|---|
| `NET_RAW` | Forger des paquets réseau bruts (ARP/IP spoofing) |
| `SYS_CHROOT` | Changer la racine du système de fichiers |
| `AUDIT_WRITE` | Écrire dans le journal d'audit du noyau |
| `CHOWN` | Changer le propriétaire de n'importe quel fichier |

La stratégie correcte est : **tout supprimer, puis ne rajouter que ce qui est strictement nécessaire**.

### Étape 1 — Identifier les capabilities actives par défaut

`docker inspect` ne montre que les overrides explicites — un conteneur sans `cap_drop` ni `cap_add` affiche `[]` dans les deux champs, mais dispose quand même de ~14 capabilities actives par défaut. Voici comment les voir réellement :

```bash
# Lire les capabilities du processus principal dans demo-vuln
docker exec demo-vuln cat /proc/1/status | grep Cap
```

Vous obtenez des valeurs hexadécimales (`CapEff`, `CapPrm`, etc.). Pour les décoder en noms lisibles :

```bash
# Installer capsh sur l'HÔTE (pas dans le conteneur)
sudo apt install -y libcap2-bin

# Décoder CapEff (capabilities effectivement actives)
capsh --decode=$(docker exec demo-vuln cat /proc/1/status | grep CapEff | awk '{print $2}')
```

Vous devriez voir apparaître une liste comme :

```
cap_chown, cap_dac_override, cap_fowner, cap_fsetid, cap_kill,
cap_setgid, cap_setuid, cap_setpcap, cap_net_bind_service,
cap_net_raw, cap_sys_chroot, cap_mknod, cap_audit_write, cap_setfcap
```

Notez en particulier `cap_net_raw` et `cap_sys_chroot` — deux capabilities inutiles pour la quasi-totalité des applications et exploitables par un attaquant.

### Étape 2 — Créer `docker-compose.secure.yml`

> **Note technique — limites de ressources :** `deploy.resources` est une directive Docker Swarm ignorée par `docker compose` classique. Pour que les limites CPU et mémoire s'appliquent réellement, utilisez `mem_limit`, `mem_reservation` et `cpus` directement au niveau du service.

```yaml
services:
  node-exporter:
    image: prom/node-exporter:v1.7.0        # Tag fixé — jamais "latest"
    container_name: node-exporter-secure
    ports:
      - "9100:9100"
    restart: unless-stopped

    # Limites de ressources (syntaxe standalone — fonctionne sans Swarm)
    mem_limit: 128m           # Maximum 128 Mo de RAM
    mem_reservation: 32m      # Garantie minimum de 32 Mo RAM
    cpus: 0.25                # Maximum 25% d'un cœur CPU

    # Limite de processus — parade directe à la fork bomb
    pids_limit: 64

    # Sécurité : principe du moindre privilège
    cap_drop:
      - ALL                         # Supprimer TOUTES les capabilities par défaut
    cap_add:
      - DAC_READ_SEARCH             # Nécessaire pour lire /proc et les métriques système

    security_opt:
      - no-new-privileges:true      # Interdit l'élévation de privilèges (setuid/setgid)

    read_only: true                 # Système de fichiers en lecture seule

    user: "65534:65534"             # Exécution en tant que nobody:nogroup (non-root)

    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'

  prometheus:
    image: prom/prometheus:v2.51.0
    container_name: prometheus-secure
    ports:
      - "9090:9090"
    restart: unless-stopped
    mem_limit: 256m
    cpus: 0.50
    pids_limit: 128
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    user: "65534:65534"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro

  grafana:
    image: grafana/grafana:10.4.2
    container_name: grafana-secure
    ports:
      - "3000:3000"
    restart: unless-stopped
    mem_limit: 256m
    cpus: 0.50
    pids_limit: 128
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    env_file:
      - .env                        # Credentials dans .env, jamais en clair ici
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  grafana-data:
```

### Étape 3 — Créer le fichier `.env`

```bash
cat > .env << 'EOF'
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=changeme_mot_de_passe_fort
EOF

echo ".env" >> .gitignore
```

### Étape 4 — Lancer la stack sécurisée et valider

```bash
# Arrêter la stack vulnérable
docker compose -f docker-compose.vuln.yml down

# Lancer la stack sécurisée
docker compose -f docker-compose.secure.yml up -d

# Vérifier que les conteneurs tournent
docker ps

# Vérifier les limites réellement appliquées
docker inspect node-exporter-secure \
  --format='Mem: {{.HostConfig.Memory}} | CPUs: {{.HostConfig.NanoCpus}} | PIDs: {{.HostConfig.PidsLimit}} | CapDrop: {{.HostConfig.CapDrop}} | CapAdd: {{.HostConfig.CapAdd}}'
```

Les valeurs attendues :
- `Mem` : `134217728` (128 Mo en octets)
- `NanoCpus` : `250000000` (0.25 CPU)
- `PIDs` : `64`
- `CapDrop` : `[ALL]`

### Étape 5 — Vérifier les capabilities après durcissement

```bash
# Comparer les capabilities avant et après
echo "=== demo-vuln (non protégé) ==="
capsh --decode=$(docker exec demo-vuln cat /proc/1/status | grep CapEff | awk '{print $2}')

echo "=== node-exporter-secure (cap_drop: ALL) ==="
capsh --decode=$(docker exec node-exporter-secure cat /proc/1/status | grep CapEff | awk '{print $2}')
```

Le second devrait retourner `0x0000000000000000 =` — aucune capability active.

**Questions d'analyse :**

- Pourquoi commence-t-on par `cap_drop: ALL` plutôt que de supprimer les capabilities dangereuses une par une ?
- Quelle est la différence entre `cap_drop: ALL` + `cap_add: [DAC_READ_SEARCH]` et ne rien configurer du tout ?
- En quoi `no-new-privileges: true` protège-t-il contre les binaires `setuid` ?
- Pourquoi `pids_limit` est-il la parade directe à la fork bomb, et non `mem_limit` ?
- Le paramètre `user: "65534:65534"` force l'exécution non-root. Quel risque cela atténue-t-il si le conteneur est compromis ?
- Pourquoi `deploy.resources` est-il ignoré en `docker compose` classique ? Dans quel contexte s'applique-t-il ?

---

## Format de rendu attendu

Un document (Markdown ou PDF) contenant :

- Le fichier `docker-compose.secure.yml` complet et commenté.
- Captures `docker stats` montrant la fork bomb sur `demo-vuln` (PIDS illimités) et son blocage sur `demo-secure` (PIDS ≤ 20).
- Capture de `docker inspect node-exporter-secure` confirmant les limites appliquées.
- Capture de la comparaison `capsh` avant/après durcissement.
- Réponses aux questions d'analyse des deux missions.

---

## Checklist de validation

- [ ] `docker-compose.secure.yml` avec `mem_limit`, `cpus`, `pids_limit`, `cap_drop`, `cap_add`, `security_opt`
- [ ] Les credentials Grafana sont dans `.env` (pas en clair dans le Compose)
- [ ] `.env` est dans `.gitignore`
- [ ] Les images utilisent des tags fixés (pas `latest`)
- [ ] Capture `docker stats` — fork bomb sur `demo-vuln` (PIDS explosent) et sur `demo-secure` (PIDS bloqués à 20)
- [ ] Capture `docker inspect` confirmant `Memory`, `NanoCpus`, `PidsLimit`, `CapDrop`
- [ ] Capture `capsh` comparant les capabilities avant/après
- [ ] Réponses aux questions d'analyse

---

## Ressources

- [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- [Linux Capabilities — man7.org](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [Docker Compose — options de ressources standalone](https://docs.docker.com/compose/compose-file/05-services/#mem_limit)
- [Docker — seccomp et capabilities](https://docs.docker.com/engine/security/seccomp/)