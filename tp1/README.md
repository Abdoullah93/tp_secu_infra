# TP1 : Missions — Audit et durcissement de l'hôte

**Durée : 3h**

Contexte : Avant de protéger les conteneurs, l'hôte doit être sécurisé. En l'état, des services et ports exposés représentent une entrée pour un attaquant.

## Mission 1.1 — Reconnaissance (se placer en attaquant)

- Consigne : Utilisez `nmap` pour cartographier la machine hôte (ports et versions visibles). Exemple :

```bash
nmap -sV -p 22,3000,9090,9100 localhost
```

- Question d'analyse : Quels services sont visibles depuis l'extérieur ? Pour chaque service découvert, décrivez brièvement ce qu'un attaquant pourrait en tirer (ex : exploitation d'une version vulnérable d'OpenSSH, accès non autorisé à Grafana, etc.).
Vous pouvez répondre à la question avec localhost mais en pratique un attaquant mettrait l'adresse ip de votre machine/serveur.  

## Mission 1.2 — Verrouillage et sécurisation des accès administrateurs (SSH)

Contexte : Par défaut, une machine Ubuntu fraîche peut ne pas avoir `openssh-server` installé. Vous devrez : Installer SSH localement (si nécessaire), l'isoler sur la boucle locale pour éviter une exposition sur le réseau du campus, créer un compte d'administration dédié et désactiver les méthodes d'authentification vulnérables (login/mdp).

Consignes détaillées :

1) Installation et isolation du serveur SSH (si manquant)

```bash
sudo apt update && sudo apt install -y openssh-server
# Forcer sshd à n'écouter que sur la boucle locale (édition de /etc/ssh/sshd_config)
# Ajouter ou modifier :
ListenAddress 127.0.0.1
```

2) Création de l'utilisateur `devops` et assignation groupe sudo

```bash
sudo useradd -m -s /bin/bash devops
sudo usermod -aG sudo devops
```

3) Génération et mise en place de la paire de clés (Ed25519)

```bash
# on crée le dossier pour y créer la clé Ed25519
sudo -u devops mkdir -p /home/devops/.ssh
# création de la paire de clé, /!\ la clé privée doit être detenu par la personne qui va se connecter au serveur.
sudo -u devops ssh-keygen -t ed25519 -C "devops@local" -f /home/devops/.ssh/id_ed25519 -N ""
# lisez la sortie et identifié avec un "ls" la paire de clé crée

# ajouter la clé au registre des clés autorisées.
sudo -u devops bash -c "cat /home/devops/.ssh/id_ed25519.pub >> /home/devops/.ssh/authorized_keys"
sudo -u devops chmod 600 /home/devops/.ssh/authorized_keys
sudo chown -R devops:devops /home/devops/.ssh
```

4) Durcissement du service SSH

Éditez `/etc/ssh/sshd_config` et vérifiez / ajoutez :

```
ListenAddress 127.0.0.1 # cela a normalement déjà été fait plus haut
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

5) Application et test

```bash
sudo systemctl restart ssh || sudo systemctl restart sshd
# Tester la connexion locale via la clé privée, rappelez vous à qui vous avez donné la clé privé. 
ssh -i /home/<username>/.ssh/id_ed25519 devops@127.0.0.1
```
Question d'analyse : 
- En situation réelle, sur quelle machine (le poste client de l'administrateur ou le serveur cible) la paire de clés doit-elle être générée ? Justifiez votre réponse.
- Si vous devez transmettre votre accès à un serveur distant, quel fichier (.pub ou clé privée) doit être transféré ? Expliquez pourquoi le transfert de l'autre fichier constituerait une faille de sécurité majeure.

Question d'analyse : 
- Pourquoi l'algorithme Ed25519 est-il préférable aujourd'hui face aux attaques par force brute comparé aux anciens standards (ex: RSA) ? 
- Actuellement, vous avez configuré ListenAddress 127.0.0.1. Si vous deviez administrer ce serveur à distance depuis une autre machine (via un VPN par exemple), pourquoi cette configuration poserait-elle problème et quelle serait la configuration la plus sécurisée à adopter ?

## Mission 1.3 — Stratégie "Drop by Default" (Pare-feu)

- Consigne : Configurez UFW pour bloquer par défaut tout en autorisant SSH. Testez l'accès à Grafana depuis votre navigateur.

Exemples :

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable
sudo ufw status
```

Question d'analyse : 
- Que constatez-vous lorsque vous tentez d'accéder à Grafana après activation ? Si Grafana est un service métier critique, quelle stratégie recommanderiez-vous pour le rendre disponible sans sacrifier la sécurité ?
- Le durcissement manuel d'un serveur est chronophage et sujet à l'erreur humaine. Comment garantiriez-vous que ces configurations (SSH, UFW, création d'utilisateurs) sont appliquées de manière identique sur un parc de 100 serveurs, et comment détecter si un administrateur modifie manuellement le fichier sshd_config

## Format de rendu attendu

- Un document **"Bilan de Reconnaissance"** (Markdown ou PDF) rassemblant :
  - Les résultats du scan `nmap` et la liste des services visibles.
  - Les réponses aux questions d'analyse pour chaque mission.
  - Une capture d'écran démontrant le blocage (ex : message de timeout dans le navigateur pour Grafana).
  - Toute réflexion complémentaire sur le sujet sera valorisée.


Ressources

- [Guide SSH hardening](https://www.ssh.com/academy/ssh/sshd_config)
- [Documentation UFW](https://help.ubuntu.com/community/UFW)