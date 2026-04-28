# TP0 : Mission — Remise en route d'une stack de monitoring "Legacy"

**Durée : 2h**

Contexte : Vous intégrez une entreprise qui a une stack de monitoring en service, mais mal documentée et exposée.

Mission principale

- Remonter la stack "Legacy" fournie (Prometheus, Grafana, Node Exporter) et rendre l'état initial vérifiable pour l'équipe de sécurité.

Mission 1 — Remonter la stack "Legacy"

- Consigne : Utilisez le `docker-compose.yml` fourni pour lancer Prometheus, Grafana et Node Exporter. Vérifiez que les services démarrent correctement.
- Preuve attendue : Validation visuelle par le formateur — Grafana accessible sur le port 3000 (http://localhost:3000).

Question d'analyse (court texte attendu) : En observant l'architecture déployée, pourquoi peut-on qualifier cette stack de "passoire" côté sécurité ? Donnez au moins trois risques concrets (ex : ports exposés, mots de passe par défaut, volumes montés dangereux) et une remédiation prioritaire pour chacun.

Commande et tests utile:

- `docker-compose up -d` lance Prometheus, Grafana et Node Exporter.
- Prometheus : http://localhost:9090 — vérifier `Targets` → `node-exporter:9100` est UP.
- Grafana : http://localhost:3000 accessible.

Format de rendu attendu

- Pas de rendu écrit obligatoire. Un court paragraphe Markdown répondant à la question d'analyse sera à joindre avec les prochains TPs.



Ressources

- [Documentation Docker Compose](https://docs.docker.com/compose/)
- [Guide Prometheus](https://prometheus.io/docs/introduction/overview/)