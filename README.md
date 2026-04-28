# TP Sécurité Infrastructure

Ce dépôt contient les travaux pratiques (TP) pour le cours **Sécurisation d'Infrastructure**. Les TPs  se concentrent sur la sécurisation pratique plutôt que sur le développement. Ils s'appuient sur le syllabus du cours (théorie sécurité infra + ateliers monitoring/sécurisation) et progressent d'une infrastructure non sécurisée à une infrastructure "hardened".

## Structure des TPs

- **TP0** : Mise en place d'une stack de monitoring (Prometheus, Grafana, Node Exporter) - 2h
- **TP1** : Audit initial et durcissement de l'hôte (nmap, SSH, UFW, Reverse Proxy) - 3h
- **TP2** : Sécurité du runtime Docker (isolation, ressources, capabilities) - 3.5h
- **TP3** : Sécurité de la supply chain et secrets (Trivy, images, gestion secrets) - 3.5h
- **TP Noté Final** : Audit de conformité (Docker Bench for Security) - 2h

Total : 14h

## Prérequis

- Environnement Linux local avec Docker installé (version >= 20.10).
- Outils : nmap, Trivy, Docker Bench for Security (installés via package manager ou Docker).
- Connaissances de base en ligne de commande Linux.

Chaque TP inclut des templates et des commandes pré-remplies pour éviter la perte de temps sur le développement.

## Objectifs Généraux

- Comprendre les vulnérabilités courantes dans les infrastructures de monitoring.
- Appliquer des bonnes pratiques de sécurisation (defense in depth).
- Valider la conformité via des audits automatisés.
- Lisez les documentations des différents outils que vous utilisez pour comprendre 

Bonne chance !
