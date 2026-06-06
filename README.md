# TPs Linux — Ubuntu 26.04 Desktop

Série de travaux pratiques Linux pour débutants et intermédiaires, conçue pour Ubuntu 26.04 Desktop.  
Les TPs sont progressifs : chaque TP s'appuie sur les compétences acquises dans les précédents.

---

## Liste des TPs

| # | Titre | Niveau | Prérequis |
|---|---|---|---|
| [TP01](TP01%20Linux%20-%20Shell.md) | Shell | Débutant | Aucun |
| [TP02](TP02%20Linux%20-%20Services.md) | Services | Débutant | TP01 |
| [TP03](TP03%20Linux%20-%20Network.md) | Réseau | Débutant | TP01, TP02 |
| [TP04](TP04%20Linux%20-%20Script%20Bash.md) | Script Bash | Intermédiaire | TP01 à TP03 |
| [TP05](TP05%20Linux%20-%20Docker.md) | Docker | Intermédiaire | TP01 à TP03 |
| [TP06](TP06%20Linux%20-%20Vagrant%20+%20Nextcloud.md) | Vagrant + Nextcloud | Intermédiaire | TP01 à TP03, VirtualBox + Vagrant installés |
| [TP07](TP07%20Linux%20-%20Sécurité.md) | Sécurité | Avancé | TP06 complété |

---

## Contenu détaillé

### TP01 — Shell
Prise en main du terminal Linux. Navigation dans l'arborescence, manipulation de fichiers, redirections, permissions, éditeur nano.

### TP02 — Services
Gestion des services système avec systemd. Démarrage, arrêt, activation au démarrage, consultation des logs avec journalctl.

### TP03 — Réseau
Configuration réseau sous Linux. Interfaces, adresses IP, Netplan, résolution DNS avec systemd-resolved, diagnostic réseau.

### TP04 — Script Bash
Écriture de scripts shell. Variables, conditions, boucles, fonctions, arguments, traitement de fichiers, automatisation de tâches.

### TP05 — Docker
Installation de Docker via le dépôt officiel. Création d'une application Flask, écriture d'un Dockerfile, build d'image, Docker Compose avec WordPress. Cheatsheet Docker complète.

### TP06 — Vagrant + Nextcloud
Déploiement d'une infrastructure multi-VMs avec Vagrant et VirtualBox. Deux VM Debian 13 : Nextcloud (box01) et MariaDB (box02), reliées en réseau privé. Installation et configuration complètes via CLI.

### TP07 — Sécurité
Scan réseau avec nmap, attaques par brute force avec hydra (MariaDB et Nextcloud WebDAV), protection avec fail2ban, analyse de trafic avec tcpdump, durcissement UFW et MariaDB. S'appuie sur l'infrastructure du TP06.

---

## Environnement requis

- **OS hôte** : Ubuntu 26.04 Desktop
- **TP06 et TP07** nécessitent en plus : VirtualBox et Vagrant installés, minimum 8 Go de RAM disponible
