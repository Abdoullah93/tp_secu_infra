# TP3 : Missions — Sécurité de la supply chain et gestion des secrets

**Durée : 3h30**

> **Contexte :** Vous prenez en charge le dépôt d'une équipe qui vient de partir en vacances. On vous signale qu'une image utilisée en production aurait des vulnérabilités connues, et qu'un stagiaire aurait "peut-être" commité des mots de passe il y a quelques semaines. Votre mission : auditer, corriger, et mettre en place les garde-fous pour que ça ne se reproduise pas.

---

## Mise en place — Dépôt de démo compromis

Commencez par créer une situation réaliste : un dépôt avec des secrets commités et des images non auditées.

```bash
mkdir tp3-legacy && cd tp3-legacy
git init
```

Créez ce `docker-compose.yml` volontairement problématique :

```yaml
# docker-compose.yml  (NE PAS utiliser tel quel en production)
version: "3.8"

services:
  prometheus:
    image: prom/prometheus:latest          # Tag flottant — version inconnue
    container_name: prometheus
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest          # Tag flottant
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: supersecret123   # Secret en clair !
      GF_DATABASE_PASSWORD: dbpassword_prod        # Secret en clair !

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
```

Commitez-le (simulation du "stagiaire") :

```bash
git add docker-compose.yml
git commit -m "init stack monitoring"

# Simulation : le stagiaire crée un .env avec des secrets et le commite par erreur
cat > .env << 'EOF'
GF_SECURITY_ADMIN_PASSWORD=supersecret123
DB_PASSWORD=dbpassword_prod
API_KEY=sk-prod-abcdef123456
EOF

git add .env
git commit -m "add env config"

# Il réalise son erreur et supprime le fichier... mais trop tard
git rm .env
git commit -m "remove .env oops"
```

Vous avez maintenant un dépôt qui ressemble à de nombreux projets réels.

---

## Mission 3.1 — Audit de la supply chain : CVE et images

> **Objectif :** Identifier les vulnérabilités dans les images utilisées, comprendre leur impact, pratiquer la remédiation par changement de tag.

### Étape 1 — Installer Trivy

```bash
# Sur Ubuntu/Debian
sudo apt install -y wget apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" \
  | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt update && sudo apt install -y trivy
#### INSTALLATION CORRIGéE ####
wget https://github.com/aquasecurity/trivy/releases/download/v0.70.0/trivy_0.70.0_Linux-64bit.deb
sudo dpkg -i trivy_0.70.0_Linux-64bit.deb

# Vérification
trivy --version
```

### Étape 2 — Scanner les images avec le tag `latest`

```bash
# Scanner Prometheus latest — noter le nombre de CVE par sévérité
trivy image --severity HIGH,CRITICAL prom/prometheus:latest

# Scanner Grafana latest
trivy image --severity HIGH,CRITICAL grafana/grafana:latest

# Scanner Node Exporter latest
trivy image --severity HIGH,CRITICAL prom/node-exporter:latest
```

Pour chaque image, notez dans votre rendu :
- Le nombre de CVE `CRITICAL` et `HIGH`
- Les 2 ou 3 packages les plus touchés

### Étape 3 — Générer un rapport HTML exploitable

```bash
# Télécharger le template HTML de Trivy ### DEJA FAIT POUR VOUS
wget https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl

# Générer les rapports
trivy image --format template --template "@html.tpl" \
  -o trivy-prometheus-latest.html prom/prometheus:latest

trivy image --format template --template "@html.tpl" \
  -o trivy-grafana-latest.html grafana/grafana:latest
```

Ouvrez `trivy-prometheus-latest.html` dans votre navigateur et identifiez une CVE CRITICAL. Notez son identifiant (ex: `CVE-2024-XXXXX`).

### Étape 4 — Analyser une CVE en détail

Prenez la CVE CRITICAL la plus récente trouvée et recherchez-la sur [https://nvd.nist.gov/vuln/search](https://nvd.nist.gov/vuln/search).

Pour votre rendu, documentez :
- L'identifiant CVE et le score CVSS
- Le package vulnérable et la version affectée
- Le vecteur d'attaque (réseau ? local ? authentifié ?)
- La version qui corrige la vulnérabilité (si disponible)

### Étape 5 — Remédiation : fixer les tags et rescanner

Identifiez des tags stables et récents pour chaque image :

```bash
# Lister les tags disponibles pour Prometheus
# (sur hub.docker.com, chercher "prom/prometheus" → Tags)
# Choisir un tag sémantique récent, ex: v2.51.0

# Scanner le tag fixé et comparer
trivy image --severity HIGH,CRITICAL prom/prometheus:v2.51.0
trivy image --severity HIGH,CRITICAL grafana/grafana:10.4.2
trivy image --severity HIGH,CRITICAL prom/node-exporter:v1.7.0
```

Comparez le nombre de CVE entre `latest` et le tag fixé. Documentez la différence.

**Questions d'analyse :**

- Pourquoi `latest` est-il dangereux en production, même si l'image est à jour aujourd'hui ?
- Qu'est-ce qu'un score CVSS et comment l'interpréter pour prioriser les correctifs ?
- Un tag fixé (ex: `v2.51.0`) garantit-il l'absence de vulnérabilités ? Quelle pratique complémentaire faudrait-il adopter en CI/CD ?
- Si une image n'a aucun tag maintenu sans CVE CRITICAL, quelle est la démarche à suivre ?

---

## Mission 3.2 — Détection et élimination des secrets

> **Objectif :** Comprendre qu'un secret commité reste dans l'historique Git même après suppression, détecter les fuites, et mettre en place les pratiques qui les préviennent.

### Étape 1 — Le secret supprimé existe toujours dans Git

Démontrez que supprimer un fichier ne supprime pas son historique :

```bash
# On est dans tp3-legacy/ (créé en introduction)

# Le .env a été "supprimé" — mais est-il vraiment parti ?
git log --oneline

# Retrouver le commit qui contenait .env
git show HEAD~1:.env

# Ou de manière plus explicite
git log --all --full-history -- .env
```

Vous verrez les secrets apparaître dans l'historique Git. C'est ce qu'un attaquant ayant accès au dépôt (même partiel) pourrait faire en quelques secondes.

### Étape 2 — Scanner le dépôt avec Gitleaks

Gitleaks est l'outil de référence pour détecter les secrets dans l'historique Git.

```bash
# Installation
wget https://github.com/gitleaks/gitleaks/releases/download/v8.18.4/gitleaks_8.18.4_linux_x64.tar.gz
tar -xzf gitleaks_8.18.4_linux_x64.tar.gz
sudo mv gitleaks /usr/local/bin/

# Scanner tout l'historique du dépôt
gitleaks detect --source . --report-format json --report-path gitleaks-report.json

# Afficher le rapport lisible
cat gitleaks-report.json | python3 -m json.tool
```

Gitleaks détecte automatiquement les patterns de secrets connus (clés AWS, tokens GitHub, mots de passe, clés API...).

**Question :** Combien de secrets détecte-t-il ? Sur quels commits ?

### Étape 3 — Corriger la situation : migrer vers `.env`

Créez la structure correcte :

```bash
# .env.example — commité dans Git, sans valeurs réelles
cat > .env.example << 'EOF'
# Copier ce fichier en .env et renseigner les valeurs
GF_SECURITY_ADMIN_USER=
GF_SECURITY_ADMIN_PASSWORD=
GF_DATABASE_PASSWORD=
API_KEY=
EOF

# .env — jamais commité, contient les vraies valeurs
cat > .env << 'EOF'
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=VotreMotDePasseFort!2024
GF_DATABASE_PASSWORD=AutreMotDePasse!
API_KEY=sk-local-dev-uniquement
EOF

# Protéger .env
echo ".env" >> .gitignore
echo "*.env" >> .gitignore

# Vérifier que .env ne sera pas commité accidentellement
git status
```

### Étape 4 — Mettre à jour `docker-compose.yml` pour utiliser `.env`

```yaml
services:
  grafana:
    image: grafana/grafana:10.4.2
    container_name: grafana
    ports:
      - "3000:3000"
    env_file:
      - .env          # Les variables sont chargées depuis .env, jamais écrites ici
```

Testez :

```bash
docker compose up -d grafana
docker exec grafana env | grep GF_SECURITY   # Vérifier que la variable est bien injectée
```

### Étape 5 — Ajouter un hook Git pour prévenir les futures fuites

```bash
# Créer un pre-commit hook qui bloque si .env est dans les fichiers stagés
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash
if git diff --cached --name-only | grep -qE '\.env$|\.env\.local$'; then
  echo "❌ ERREUR : Tentative de commit d'un fichier .env détectée."
  echo "   Retirez le fichier avec : git reset HEAD .env"
  exit 1
fi
EOF

chmod +x .git/hooks/pre-commit

# Tester le hook
git add .env 2>/dev/null || true
git commit -m "test hook"   # Doit être bloqué
```

**Questions d'analyse :**

- Un secret supprimé avec `git rm` puis commité est-il définitivement effacé du dépôt ? Comment un attaquant peut-il le retrouver ?
- Quelle est la différence entre `.env`, `.env.example` et `.env.local` ? Lequel doit être commité ?
- Si un secret de production a été exposé dans un dépôt (même privé), quelle est la **première action** à faire, avant même de corriger le code ?
- Citez deux outils ou mécanismes qui permettent de ne jamais stocker de secrets en clair, même dans `.env` (ex : HashiCorp Vault, variables d'environnement CI/CD chiffrées...).

---

## Mission 3.3 — Images non-root et Dockerfile sécurisé

> **Objectif :** Comprendre pourquoi exécuter un processus en root dans un conteneur est risqué, et construire une image qui applique le principe du moindre privilège dès la construction.

### Contexte

Par défaut, beaucoup d'images Docker officielles s'exécutent en tant que `root` à l'intérieur du conteneur. Si le processus est compromis et qu'une évasion de conteneur est possible (via une mauvaise configuration ou une CVE noyau), l'attaquant arrive sur l'hôte en tant que root.

```bash
# Vérifier quel utilisateur tourne dans node-exporter:latest
docker run --rm --entrypoint /bin/sh prom/node-exporter:latest -c "id"
```

### Étape 1 — Créer un Dockerfile non-root pour Node Exporter

```dockerfile
# Dockerfile.node-exporter
FROM prom/node-exporter:v1.7.0

# Vérification : node-exporter:v1.7.0 tourne déjà en nobody
# Pour les images qui ne le font pas, voici le pattern à appliquer :

# Créer un utilisateur dédié sans shell ni répertoire home
USER nobody

# Bonne pratique : déclarer explicitement l'utilisateur même si l'image le fait déjà
# Cela documente l'intention et protège contre une future régression de l'image upstream
```

Pour une image qui tourne réellement en root (exemple avec Ubuntu) :

```dockerfile
# Dockerfile.secure-exemple
FROM ubuntu:22.04

# Créer un groupe et utilisateur applicatif dédié
RUN groupadd --gid 1001 appgroup && \
    useradd --uid 1001 --gid appgroup --no-create-home --shell /bin/false appuser

# Copier les binaires applicatifs
# COPY --chown=appuser:appgroup ./app /app

# Basculer vers l'utilisateur non-root
USER appuser

# Aucun processus ne peut escalader vers root
CMD ["/bin/sh"]
```

### Étape 2 — Construire et vérifier

```bash
docker build -f Dockerfile.node-exporter -t node-exporter-secure:local .

# Vérifier l'utilisateur effectif
docker run --rm --entrypoint /bin/sh node-exporter-secure:local -c "id"


# Scanner l'image custom avec Trivy
trivy image node-exporter-secure:local
```

### Étape 3 — Choisir une image de base minimaliste

Comparez l'empreinte et la surface d'attaque de différentes bases :

```bash
# Comparer les tailles et le nombre de packages
docker pull ubuntu:22.04
docker pull alpine:3.19
docker pull gcr.io/distroless/static-debian12

docker images | grep -E "ubuntu|alpine|distroless"

# Scanner chaque base
trivy image ubuntu:22.04 --severity CRITICAL,HIGH | tail -5
trivy image alpine:3.19 --severity CRITICAL,HIGH | tail -5
trivy image gcr.io/distroless/static-debian12 --severity CRITICAL,HIGH | tail -5
```

**Questions d'analyse :**

- Quelle différence observez-vous en termes de nombre de CVE entre `ubuntu:22.04`, `alpine:3.19` et `distroless` ?
- Pourquoi une image `distroless` est-elle plus difficile à attaquer, même si une vulnérabilité est présente ?
- Si votre application a besoin de `bash` pour fonctionner, `distroless` est-elle adaptée ? Quelle alternative proposez-vous ?
- En quoi l'exécution non-root *dans* le conteneur réduit-elle le risque en cas d'évasion ?

---

## Format de rendu attendu

Un document (Markdown ou PDF) contenant :

**Mission 3.1 — Supply chain :**
- Tableau comparatif CVE : `latest` vs tag fixé pour chaque image (Prometheus, Grafana, Node Exporter)
- Analyse détaillée d'une CVE CRITICAL (identifiant, score CVSS, vecteur, remédiation)
- Le rapport Trivy HTML en pièce jointe ou capture d'écran

**Mission 3.2 — Secrets :**
- Capture de `gitleaks detect` montrant les secrets détectés dans l'historique
- Le `docker-compose.yml` corrigé avec `env_file`
- Le `.env.example` commité
- Capture de `git status` confirmant que `.env` est ignoré
- Démonstration du hook `pre-commit` qui bloque un ajout accidentel de `.env`

**Mission 3.3 — Images non-root :**
- Le `Dockerfile.node-exporter` ou équivalent
- Capture de `docker run --rm <image> whoami` retournant un utilisateur non-root
- Tableau comparatif des surfaces d'attaque (ubuntu vs alpine vs distroless)
- Réponses aux questions d'analyse

---

## Checklist de validation

- Rapport Trivy généré pour les trois images en tag `latest`
- Rapport Trivy généré pour les trois images en tag fixé — comparaison documentée
- Une CVE analysée en détail (CVSS, vecteur, remédiation)
- `gitleaks detect` exécuté et résultat commenté
- `.env` dans `.gitignore`, `.env.example` commité
- `docker-compose.yml` utilise `env_file` (plus de secrets en clair)
- Hook `pre-commit` en place et testé
- Image construite avec utilisateur non-root + capture `whoami`
- Réponses aux questions d'analyse des trois missions

---

## Ressources

- [Trivy — Documentation](https://aquasecurity.github.io/trivy/)
- [Gitleaks — GitHub](https://github.com/gitleaks/gitleaks)
- [NVD — Base de données CVE](https://nvd.nist.gov/vuln/search)
- [Docker — Bonnes pratiques Dockerfile](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Distroless images — Google](https://github.com/GoogleContainerTools/distroless)
- [CVSS Calculator](https://www.first.org/cvss/calculator/3.1)