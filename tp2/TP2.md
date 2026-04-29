# TP2 : Missions — Sécurité du runtime Docker et isolation

**Durée : 3h30**

> **Contexte :** Les conteneurs s'exécutent aujourd'hui avec trop de privilèges et sans limites de ressources. Un conteneur compromis peut provoquer un déni de service (DoS) ou s'échapper vers l'hôte. Dans ce TP, vous allez d'abord *constater* ces risques dans un environnement volontairement vulnérable, puis les corriger méthodiquement.

---

## Mise en place — Conteneur de démonstration vulnérable

Avant de commencer les missions, créez le fichier suivant. Il représente un conteneur **intentionnellement non protégé**, tel qu'on en trouve trop souvent en production.

**`docker-compose.vuln.yml`**

```yaml
services:
  demo-vuln:
    image: ubuntu:22.04
    container_name: demo-vuln
    # Aucune limite CPU ni mémoire
    # Aucune restriction de capabilities
    # Accès complet au système de processus de l'hôte
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

> **Objectif :** Comprendre pourquoi un conteneur sans limites peut mettre à genoux un serveur entier, et observer cet impact en temps réel via les métriques.

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

### Étape 3 — Lancer la Fork Bomb (dans le conteneur uniquement)

> **⚠️ Sécurité :** La commande suivante doit être exécutée **uniquement dans le shell du conteneur**, jamais directement sur votre machine hôte. Le conteneur `demo-vuln` n'a pas de `pids_limit`, ce qui permet de démontrer l'impact — mais votre hôte Ubuntu dispose de sa propre limite système (généralement 32 768 processus) qui empêchera un crash total. Si votre machine commence à freezer, ouvrez un troisième terminal et exécutez `docker kill demo-vuln`.

```bash
# Commande à exécuter DANS le conteneur (après docker exec -it demo-vuln bash)
:(){ :|:& };:
```

Cette one-liner définit une fonction `:` qui s'appelle elle-même deux fois en parallèle, créant une explosion exponentielle de processus.

### Étape 4 — Observer l'impact

Dans le Terminal A (`docker stats`), regardez la colonne `PIDS` exploser pour le conteneur `demo-vuln`. Observez aussi comment la charge CPU de l'hôte augmente dans le Terminal B.

**Après 15 à 20 secondes**, depuis un troisième terminal, stoppez proprement le conteneur :

```bash
docker kill demo-vuln
```

Puis redémarrez-le pour la suite :

```bash
docker compose -f docker-compose.vuln.yml up -d
```

**Questions d'analyse :**

- Qu'avez-vous observé dans `docker stats` pendant la fork bomb ? La colonne `PIDS` a-t-elle une limite ?
- Node Exporter a-t-il continué à répondre sur `:9100` pendant l'attaque ? Pourquoi ?
- Pourquoi le monitoring cesse-t-il de répondre en premier dans un scénario réel sans isolation ?
- Quel serait l'impact métier si ce serveur était mutualisé entre plusieurs équipes en production ?

---

## Mission 2.2 — Application du principe du moindre privilège

> **Objectif :** Comprendre les capabilities Linux et appliquer des limites strictes (CPU, RAM, processus, capabilities) pour qu'un conteneur compromis ne puisse pas nuire au-delà de son périmètre.

### Contexte — Qu'est-ce qu'une capability Linux ?

Par défaut, Docker accorde aux conteneurs un ensemble de **capabilities** (droits noyau granulaires) bien plus large que nécessaire pour la plupart des applications. Parmi les plus dangereuses accordées par défaut :

| Capability | Ce qu'elle permet |
|---|---|
| `NET_RAW` | Forger des paquets réseau bruts (utile pour des attaques ARP/IP spoofing) |
| `SYS_CHROOT` | Changer la racine du système de fichiers |
| `AUDIT_WRITE` | Écrire dans le journal d'audit du noyau |
| `CHOWN` | Changer le propriétaire de n'importe quel fichier |

La stratégie correcte est : **tout supprimer, puis ne rajouter que ce qui est strictement nécessaire**.

### Étape 1 — Identifier les capabilities utilisées par un service

Avant de restreindre, identifiez ce dont votre service a réellement besoin :

```bash
# Lister les capabilities actives d'un conteneur en cours d'exécution
docker inspect demo-vuln --format='{{.HostConfig.CapAdd}} / Drop: {{.HostConfig.CapDrop}}'

# Pour node-exporter (lecture seule du système, pas besoin de capabilities réseau avancées)
docker inspect node-exporter --format='{{.HostConfig.CapAdd}} / Drop: {{.HostConfig.CapDrop}}'
```

### Étape 2 — Créer `docker-compose.secure.yml`

Copiez votre fichier vulnérable et appliquez les protections suivantes. Chaque paramètre est commenté pour vous expliquer son rôle.

```yaml
version: "3.8"

services:
  node-exporter:
    image: prom/node-exporter:v1.7.0        # Tag fixé — jamais "latest"
    container_name: node-exporter-secure
    ports:
      - "9100:9100"
    restart: unless-stopped

    deploy:
      resources:
        limits:
          cpus: "0.25"      # Maximum 25% d'un cœur CPU
          memory: 128M      # Maximum 128 Mo de RAM
        reservations:
          cpus: "0.05"      # Garantie minimum de 5% CPU
          memory: 32M       # Garantie minimum de 32 Mo RAM

    # Sécurité : principe du moindre privilège
    cap_drop:
      - ALL                 # On supprime TOUTES les capabilities par défaut

    cap_add:
      - DAC_READ_SEARCH     # Nécessaire pour lire /proc et les métriques système

    security_opt:
      - no-new-privileges:true   # Interdit l'élévation de privilèges (setuid/setgid)

    read_only: true         # Système de fichiers du conteneur en lecture seule

    pids_limit: 64          # Maximum 64 processus dans ce conteneur (fork bomb impossible)

    user: "65534:65534"     # Exécution en tant que nobody:nogroup (non-root)

    volumes:
      - /proc:/host/proc:ro       # Montage en lecture seule uniquement
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
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: 256M
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    pids_limit: 128
    user: "65534:65534"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro

  grafana:
    image: grafana/grafana:10.4.2
    container_name: grafana-secure
    ports:
      - "3000:3000"
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: 256M
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    pids_limit: 128
    env_file:
      - .env                # Les credentials dans un fichier .env, jamais en clair ici
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

# S'assurer que .env ne sera jamais commité
echo ".env" >> .gitignore
```

### Étape 4 — Lancer la stack sécurisée et valider

```bash
# Arrêter la stack vulnérable
docker compose -f docker-compose.vuln.yml down

# Lancer la stack sécurisée
docker compose -f docker-compose.secure.yml up -d

# Vérifier que les conteneurs sont bien lancés
docker ps

# Inspecter les limites appliquées
docker inspect node-exporter-secure | grep -A 10 '"HostConfig"' | grep -E "Memory|NanoCpus|PidsLimit|CapDrop|CapAdd"
```

### Étape 5 — Re-tenter la Fork Bomb sur le conteneur sécurisé

```bash
docker exec -it node-exporter-secure sh
# Dans le conteneur :
:(){ :|:& };:
```

Observez dans `docker stats` ce qui se passe maintenant. Le `pids_limit: 64` doit bloquer la prolifération de processus. Le conteneur peut devenir instable mais l'hôte et les autres conteneurs sont protégés.

**Questions d'analyse :**

- Pourquoi commence-t-on par `cap_drop: ALL` plutôt que de supprimer les capabilities dangereuses une par une ?
- Quelle est la différence entre `cap_drop: ALL` + `cap_add: [DAC_READ_SEARCH]` et ne rien mettre du tout ?
- En quoi `no-new-privileges: true` protège-t-il contre les binaires `setuid` ?
- Pourquoi `pids_limit` est-il la parade directe à la fork bomb ?
- Le paramètre `user: "65534:65534"` force l'exécution non-root. Quel risque cela atténue-t-il si le conteneur est compromis ?

---

## Format de rendu attendu

Un document (Markdown ou PDF) contenant :

- Le fichier `docker-compose.secure.yml` complet et commenté.
- Captures `docker stats` pendant et après la fork bomb sur le conteneur vulnérable (Mission 2.1).
- Capture de la commande `docker inspect` montrant les limites appliquées (Mission 2.2).
- Réponses aux questions d'analyse des deux missions.

---

## Checklist de validation

- `docker-compose.secure.yml` déposé avec `deploy.resources`, `cap_drop`, `cap_add`, `pids_limit` et `security_opt`
- Les credentials Grafana sont dans `.env` (pas en clair dans le Compose)
- `.env` est dans `.gitignore`
- Les images utilisent des tags fixés (pas `latest`)
- Capture `docker stats` montrant l'impact de la fork bomb sur le conteneur vulnérable
- Capture `docker inspect` confirmant les limites sur le conteneur sécurisé
- Réponses aux questions d'analyse

---

## Ressources

- [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- [Linux Capabilities — man7.org](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [Docker — Limites de ressources](https://docs.docker.com/compose/compose-file/deploy/)
- [Docker — seccomp et capabilities](https://docs.docker.com/engine/security/seccomp/)