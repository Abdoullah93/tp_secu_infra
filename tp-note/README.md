# TP Noté Final — Audit de conformité et remédiation

**Durée : 2h (indicatif)**

> **Mise en situation :** Vous êtes consultant DevSecOps missionné pour auditer l'infrastructure d'une équipe qui affirme avoir "sécurisé" sa stack de monitoring. Votre mission : vérifier cette affirmation, identifier les failles restantes, les corriger, et rédiger un rapport d'audit professionnel à destination du RSSI.

---

## Barème

| Domaine | Points |
|---|---|
| Mission A — Audit Docker Bench (analyse commentée) | 5 pts |
| Mission B — Remédiation documentée (avant/après) | 8 pts |
| Mission C — Rapport d'audit professionnel | 7 pts |
| **Total** | **20 pts** |

Détail du barème par mission en fin de document.

---

## Infrastructure de départ — obligatoire

Tout le monde part de la même base. Créez le répertoire de travail suivant :

```bash
mkdir tp-final && cd tp-final
```

**`docker-compose.audit.yml`** — infrastructure volontairement imparfaite :

```yaml
services:
  prometheus:
    image: prom/prometheus:latest              # Faille 1 : tag flottant
    container_name: prometheus-audit
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - /var/run/docker.sock:/var/run/docker.sock  # Faille 2 : socket Docker monté

  grafana:
    image: grafana/grafana:latest              # Faille 3 : tag flottant
    container_name: grafana-audit
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin        # Faille 4 : mot de passe par défaut en clair

  node-exporter:
    image: prom/node-exporter:latest           # Faille 5 : tag flottant
    container_name: node-exporter-audit
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc
      - /sys:/host/sys
    # Faille 6 : aucune limite CPU/RAM
    # Faille 7 : aucun cap_drop
    # Faille 8 : pas de no-new-privileges
```

**`prometheus.yml`** minimal :

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter-audit:9100']
```

Lancez la stack :

```bash
docker compose -f docker-compose.audit.yml up -d
docker ps
```

---

## Mission A — Audit automatisé avec Docker Bench for Security (5 pts)

### Lancement de l'audit

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /etc:/etc:ro \
  -v /usr/bin/containerd:/usr/bin/containerd:ro \
  -v /usr/bin/runc:/usr/bin/runc:ro \
  --label docker_bench_security \
  docker/docker-bench-security 2>/dev/null | tee docker-bench-report.txt
```

### Guide de lecture — filtrer ce qui compte

Docker Bench génère beaucoup de résultats. Voici comment les lire :

| Préfixe | Signification | Action |
|---|---|---|
| `[PASS]` | Conforme | Rien à faire |
| `[WARN]` | Non conforme, correctible | À corriger en priorité |
| `[INFO]` | Informatif | À documenter |
| `[NOTE]` | Recommandation | À évaluer selon le contexte |

**Sections à prioriser pour ce TP :**

```bash
# Filtrer uniquement les WARN sur les conteneurs en cours d'exécution (section 5)
grep "\[WARN\]" docker-bench-report.txt | grep -E "5\."

# Filtrer les WARN sur la configuration des images (section 4)
grep "\[WARN\]" docker-bench-report.txt | grep -E "4\."

# Compter le total de WARN
grep -c "\[WARN\]" docker-bench-report.txt
```

> **Note :** Certains WARN concernent la configuration du daemon Docker en production (TLS, auditd) et ne sont pas applicables dans votre contexte. Vous devez les identifier et justifier pourquoi vous ne les corrigez pas — c'est aussi ça, un vrai audit.

### Ce que vous rendez pour la Mission A

Un tableau dans votre rapport listant **au minimum 6 WARN** identifiés, avec pour chacun :

| ID Docker Bench | Description | Criticité estimée (H/M/L) | Applicable ? (O/N) | Justification si N |
|---|---|---|---|---|
| 5.x | ... | ... | ... | ... |

---

## Mission B — Remédiation (8 pts)

Produisez un fichier `docker-compose.hardened.yml` corrigeant les failles identifiées. Chaque correction doit être **commentée dans le fichier** pour expliquer ce qu'elle corrige.

### Corrections obligatoires (6 pts)

Chaque correction vaut des points. Elles doivent apparaître dans le fichier et être justifiées dans le rapport.

**B1 — Tags fixés sur toutes les images (1 pt)**

Remplacer tous les `latest` par des tags sémantiques fixés. Joindre la preuve du scan Trivy sur les nouveaux tags.

```yaml
# Avant
image: prom/prometheus:latest

# Après — tag fixé, version auditée
image: prom/prometheus:v2.51.0   # Scanné avec Trivy le XX/XX/XXXX — 0 CVE CRITICAL
```

**B2 — Suppression du socket Docker monté (1 pt)**

```yaml
# Avant — accès root au daemon Docker depuis le conteneur
volumes:
  - /var/run/docker.sock:/var/run/docker.sock

# Après — supprimé ; Prometheus ne scrape pas Docker directement ici
# Si nécessaire, utiliser l'exporteur dédié docker-exporter avec droits minimaux
```

**B3 — Secrets dans `.env` (1 pt)**

```yaml
# Avant
environment:
  GF_SECURITY_ADMIN_PASSWORD: admin

# Après
env_file:
  - .env
```

Fournir le `.env.example` commité et prouver que `.env` est dans `.gitignore`.

**B4 — Limites CPU et mémoire sur tous les services (1 pt)**

```yaml
deploy:
  resources:
    limits:
      cpus: "0.50"
      memory: 256M
```

Justifier les valeurs choisies pour chaque service.

**B5 — Capabilities et sécurité runtime (1 pt)**

```yaml
cap_drop:
  - ALL
cap_add:
  - DAC_READ_SEARCH    # Uniquement si nécessaire pour node-exporter
security_opt:
  - no-new-privileges:true
pids_limit: 128
```

**B6 — Utilisateur non-root (1 pt)**

Vérifier que chaque service tourne en non-root :

```bash
docker exec prometheus-hardened whoami
docker exec grafana-hardened whoami
docker exec node-exporter-hardened whoami
```

Joindre les captures.

### Vérification finale après remédiation (2 pts)

Relancez Docker Bench sur la stack corrigée :

```bash
docker compose -f docker-compose.hardened.yml up -d

docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /etc:/etc:ro \
  --label docker_bench_security \
  docker/docker-bench-security 2>/dev/null | tee docker-bench-report-after.txt

# Comparer avant / après
echo "WARN avant : $(grep -c '\[WARN\]' docker-bench-report.txt)"
echo "WARN après : $(grep -c '\[WARN\]' docker-bench-report-after.txt)"
```

Documentez la réduction du nombre de WARN dans votre rapport.

---

## Mission C — Rapport d'audit professionnel (7 pts)

Rédigez un rapport structuré d'une à deux pages, comme si vous le remettiez à un RSSI. Il doit être clair, factuel et lisible par quelqu'un qui ne fait pas de technique.

### Structure attendue

---

**RAPPORT D'AUDIT — Infrastructure de monitoring**
*Date · Auditeur · Version*

**1. Résumé exécutif** *(5-8 lignes)*
Synthèse à destination du management : état initial, risques principaux identifiés, niveau de conformité après remédiation.

**2. Périmètre et méthodologie**
- Infrastructure auditée (services, versions)
- Outils utilisés (Docker Bench, Trivy, Gitleaks si applicable)
- Durée et conditions de l'audit

**3. Résultats de l'audit initial**
Tableau des failles identifiées, classées par criticité :

| # | Faille | Criticité | Impact potentiel |
|---|---|---|---|
| F1 | Tag `latest` sur toutes les images | Haute | Déploiement incontrôlé de versions vulnérables |
| F2 | Socket Docker monté dans Prometheus | Critique | Évasion de conteneur → accès root hôte |
| ... | ... | ... | ... |

**4. Remédiations appliquées**
Pour chaque faille : ce qui a été fait, comment le vérifier, et la preuve (capture ou commande).

**5. Résultats post-remédiation**
Comparaison avant/après Docker Bench. Failles résiduelles non corrigées et justification.

**6. Recommandations pour la suite**
Deux ou trois actions que vous recommanderiez si la mission continuait (ex : intégrer Trivy en CI/CD, mettre en place une rotation des secrets, configurer le daemon Docker avec TLS).

---

### Critères de notation Mission C

| Critère | Points |
|---|---|
| Résumé exécutif lisible par un non-technicien | 1 pt |
| Tableau des failles complet et classé par criticité | 2 pts |
| Remédiations documentées avec preuves | 2 pts |
| Comparaison avant/après chiffrée (nb de WARN) | 1 pt |
| Recommandations pertinentes et réalistes | 1 pt |

---

## Checklist de rendu

Déposez une archive `tp-final-<prenom-nom>.zip` contenant :

- [ ] `docker-compose.audit.yml` — infrastructure de départ (non modifiée)
- [ ] `docker-compose.hardened.yml` — version corrigée et commentée
- [ ] `.env.example` — template de variables sans valeurs réelles
- [ ] `.gitignore` — contenant `.env`
- [ ] `docker-bench-report.txt` — rapport avant remédiation
- [ ] `docker-bench-report-after.txt` — rapport après remédiation
- [ ] `trivy-*.html` ou captures des scans Trivy sur les nouveaux tags
- [ ] Captures `docker exec ... whoami` pour chaque service
- [ ] `rapport-audit.md` ou `rapport-audit.pdf` — rapport professionnel structuré

---

## Barème détaillé

### Mission A — Audit (5 pts)

| Critère | Points |
|---|---|
| Docker Bench lancé, rapport produit | 1 pt |
| 6 WARN identifiés et listés dans le tableau | 2 pts |
| Criticité correctement estimée | 1 pt |
| WARN non applicables identifiés et justifiés | 1 pt |

### Mission B — Remédiation (8 pts)

| Critère | Points |
|---|---|
| B1 Tags fixés + preuve Trivy | 1 pt |
| B2 Socket Docker supprimé + justification | 1 pt |
| B3 Secrets dans `.env` + `.gitignore` | 1 pt |
| B4 Limites CPU/RAM sur tous les services | 1 pt |
| B5 `cap_drop`, `no-new-privileges`, `pids_limit` | 1 pt |
| B6 Utilisateur non-root vérifié (`whoami`) | 1 pt |
| Réduction documentée du nombre de WARN | 2 pts |

### Mission C — Rapport (7 pts)

*Voir critères détaillés dans la section Mission C.*

---

## Ressources

- [Docker Bench for Security](https://github.com/docker/docker-bench-security)
- [Trivy Documentation](https://aquasecurity.github.io/trivy/)
- [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- [NVD — Base CVE](https://nvd.nist.gov/vuln/search)