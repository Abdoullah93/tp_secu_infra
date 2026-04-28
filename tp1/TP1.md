# TP1 : Missions — Audit et durcissement de l'hôte

**Durée : 3h**

> **Contexte :** Avant de protéger les conteneurs, l'hôte doit être sécurisé. En l'état, des services et ports exposés représentent une entrée pour un attaquant. Dans ce TP, vous allez cartographier l'exposition de votre machine, durcir l'accès SSH, puis filtrer le trafic réseau — d'abord via UFW pour les services de l'hôte, puis via un reverse proxy Nginx pour isoler vos conteneurs Docker.

---

## Mission 1.1 — Reconnaissance (se placer en attaquant)

**Consigne :** Utilisez `nmap` pour cartographier la machine hôte (ports et versions visibles).

```bash
sudo nmap -sV -p 22,3000,9090,9100 localhost
```

> **Note :** `sudo` est requis pour la détection de versions (`-sV`). Sans lui, nmap peut ne pas identifier correctement les services.

**Question d'analyse :**

Quels services sont visibles depuis l'extérieur ? Pour chaque service découvert, décrivez brièvement ce qu'un attaquant pourrait en tirer (ex : exploitation d'une version vulnérable d'OpenSSH, accès non autorisé à Grafana, scraping des métriques Prometheus, etc.).

> En pratique, un attaquant utiliserait l'adresse IP publique de votre machine plutôt que `localhost`.

---

## Mission 1.2 — Verrouillage et sécurisation des accès administrateurs (SSH)

**Contexte :** Par défaut, une machine Ubuntu fraîche peut ne pas avoir `openssh-server` installé. Vous allez installer SSH, l'isoler sur la boucle locale, créer un compte d'administration dédié et désactiver l'authentification par mot de passe.

### Étape 1 — Installation et isolation du serveur SSH

```bash
sudo apt update && sudo apt install -y openssh-server
```

Éditez ensuite `/etc/ssh/sshd_config` et ajoutez ou modifiez la ligne suivante pour que SSH n'écoute que sur la boucle locale :

```
ListenAddress 127.0.0.1
```

### Étape 2 — Création de l'utilisateur `devops`

```bash
sudo useradd -m -s /bin/bash devops
sudo usermod -aG sudo devops
```

### Étape 3 — Génération et mise en place de la paire de clés Ed25519

```bash
# Créer le dossier .ssh pour l'utilisateur devops
sudo -u devops mkdir -p /home/devops/.ssh

# Générer la paire de clés (la clé privée reste sur la machine du client qui se connecte)
sudo -u devops ssh-keygen -t ed25519 -C "devops@local" -f /home/devops/.ssh/id_ed25519 -N ""

# Vérifier les fichiers créés
ls /home/devops/.ssh/

# Autoriser la clé publique pour la connexion
sudo -u devops bash -c "cat /home/devops/.ssh/id_ed25519.pub >> /home/devops/.ssh/authorized_keys"
sudo -u devops chmod 600 /home/devops/.ssh/authorized_keys
sudo chown -R devops:devops /home/devops/.ssh
```

### Étape 4 — Durcissement de la configuration SSH

Éditez `/etc/ssh/sshd_config` et vérifiez ou ajoutez les directives suivantes :

```
ListenAddress 127.0.0.1
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

### Étape 5 — Application et test

```bash
sudo systemctl restart ssh

# Tester la connexion en utilisant la clé privée de devops
ssh -i /home/devops/.ssh/id_ed25519 devops@127.0.0.1
```

**Questions d'analyse :**

- En situation réelle, sur quelle machine (poste client de l'administrateur ou serveur cible) la paire de clés doit-elle être générée ? Justifiez votre réponse.
- Si vous devez donner un accès à un collaborateur sur un serveur distant, quel fichier transmettez-vous — la clé publique (`.pub`) ou la clé privée ? Expliquez pourquoi transmettre l'autre fichier constituerait une faille de sécurité majeure.
- Pourquoi Ed25519 est-il préférable à RSA face aux attaques par force brute ?
- Vous avez configuré `ListenAddress 127.0.0.1`. Si vous deviez administrer ce serveur à distance depuis une autre machine (via VPN par exemple), en quoi cette configuration poserait-elle problème, et quelle serait la configuration la plus sécurisée à adopter ?

---

## Mission 1.3 — Stratégie "Drop by Default" avec UFW

> **Avertissement pédagogique :** UFW agit sur les règles `iptables` de l'hôte, mais **Docker contourne UFW** en modifiant directement iptables pour exposer les ports des conteneurs. Ainsi, après activation d'UFW, vos ports `3000`, `9090` et `9100` (Grafana, Prometheus, Node Exporter) **resteront accessibles** depuis le réseau. UFW protège efficacement les services natifs de l'hôte (comme SSH), mais pas les conteneurs Docker. C'est précisément ce que vous allez constater — et corriger dans la mission suivante.

**Consigne :** Configurez UFW pour bloquer tout trafic entrant par défaut, en n'autorisant que SSH.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable
sudo ufw status
```

Essayez ensuite d'accéder à Grafana depuis votre navigateur : `http://localhost:3000`

**Questions d'analyse :**

- Que constatez-vous ? Grafana est-il bloqué ou toujours accessible ? Pourquoi ?
- Si UFW ne suffit pas à isoler les conteneurs Docker, quelle couche de sécurité complémentaire faut-il mettre en place ?
- Le durcissement manuel d'un serveur est chronophage et sujet aux erreurs humaines. Comment garantir que ces configurations (SSH, UFW, création d'utilisateurs) sont appliquées de manière identique sur un parc de 100 serveurs ? Comment détecter si un administrateur modifie manuellement `sshd_config` ?

---

## Mission 1.4 — Reverse proxy Nginx : isoler les conteneurs

**Contexte :** UFW ne contrôle pas les ports ouverts par Docker. La solution professionnelle est d'utiliser un **reverse proxy** : un seul service écoute publiquement (port 80 ou 443), et redirige le trafic vers les conteneurs en interne. Les conteneurs ne sont plus exposés directement sur le réseau.

```
Navigateur → :80 (Nginx) → :3000 (Grafana, réseau interne Docker)
                          → :9090 (Prometheus, réseau interne Docker)
```

### Étape 1 — Retirer l'exposition directe des ports dans `docker-compose.yml`

Supprimez les mappings de ports publics pour Grafana et Prometheus. Remplacez :

```yaml
ports:
  - "3000:3000"
```

par l'absence de section `ports` pour ces services. Assurez-vous que tous vos conteneurs partagent un réseau Docker interne commun :

```yaml
networks:
  monitoring:
    driver: bridge
```

### Étape 2 — Ajouter Nginx dans le Compose

```yaml
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - monitoring
    depends_on:
      - grafana
```

### Étape 3 — Créer la configuration Nginx (`nginx.conf`)

```nginx
events {}

http {
    server {
        listen 80;

        # Grafana accessible sur /grafana/
        location /grafana/ {
            proxy_pass http://grafana:3000/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # Prometheus accessible sur /prometheus/
        # En production : restreindre l'accès par IP
        location /prometheus/ {
            proxy_pass http://prometheus:9090/;
            proxy_set_header Host $host;
        }
    }
}
```

### Étape 4 — Relancer la stack et vérifier

```bash
docker compose down
docker compose up -d
```

Vérifiez que :

- `http://localhost/grafana/` fonctionne correctement
- `http://localhost:3000` n'est **plus** accessible directement
- `http://localhost:9090` n'est **plus** accessible directement
- http://localhost/grafana/login doit être accessible mettez la capture dans votre rendu

**Questions d'analyse :**

- Quel est l'avantage d'avoir un seul point d'entrée (Nginx) plutôt que d'exposer chaque conteneur sur un port différent ?
- En production, Nginx serait configuré en HTTPS (port 443) avec un certificat TLS. Pourquoi est-ce indispensable, même sur un réseau d'entreprise ?
- Prometheus expose des métriques potentiellement sensibles sur votre infrastructure. Quelle directive Nginx permettrait de restreindre son accès à certaines adresses IP seulement ?
- En quoi le reverse proxy et UFW sont-ils complémentaires, et non redondants ?

---

## Format de rendu attendu

Un document **"Bilan de Reconnaissance et Durcissement"** (Markdown ou PDF) contenant :

- Les résultats du scan `nmap` et la liste des services visibles avec leur analyse.
- Les réponses aux questions d'analyse de chaque mission.
- Une capture d'écran montrant que Grafana est accessible via le reverse proxy (`/grafana/`) mais que le port `3000` direct est refusé.
- Toute réflexion complémentaire sera valorisée.

---

## Ressources

- [Guide SSH hardening](https://www.ssh.com/academy/ssh/sshd_config)
- [Documentation UFW](https://help.ubuntu.com/community/UFW)
- [Documentation Nginx — proxy_pass](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
- [Docker et iptables — pourquoi UFW ne suffit pas](https://docs.docker.com/network/packet-filtering-firewalls/)