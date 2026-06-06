# TP05 Linux - Docker

**Niveau** : Débutant  
**Prérequis** : Ubuntu 26.04 Desktop, terminal ouvert, connexion Internet active, droits `sudo`  
**Convention du TP** : le dossier personnel (`HOME`) de l'apprenant est `/home/PRENOM`. Remplacez `PRENOM` par votre prénom dans les chemins absolus.

---

## 📚 Objectifs pédagogiques

À la fin de ce TP, vous saurez :
- ✅ Installer Docker via le dépôt officiel apt
- ✅ Comprendre la différence entre image et conteneur
- ✅ Écrire un `Dockerfile` pour une application Python Flask
- ✅ Construire une image avec `docker build`
- ✅ Lancer, inspecter, arrêter et supprimer des conteneurs
- ✅ Utiliser les commandes Docker essentielles (cheatsheet)
- ✅ Orchestrer plusieurs services avec `docker compose`
- ✅ Déployer WordPress + MySQL avec `docker compose`

---

## 🧰 Partie 1 : Installer Docker sur Ubuntu 26.04

### 1.1 Supprimer les anciennes versions

Si Docker a déjà été installé depuis les dépôts Ubuntu (version non officielle), supprimez-le :

```bash
sudo apt remove -y docker docker-engine docker.io containerd runc
```

**Note** : Cette commande peut afficher des erreurs si les paquets ne sont pas installés. C'est normal, continuez.

---

### 1.2 Ajouter le dépôt officiel Docker

**Étape 1 : mettre à jour et installer les prérequis**

```bash
sudo apt update
sudo apt install -y ca-certificates curl
```

**Étape 2 : ajouter la clé GPG officielle Docker**

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

**Étape 3 : ajouter le dépôt apt Docker**

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
```

> **Note Ubuntu 26.04** : si `apt update` retourne une erreur indiquant que la distribution n'est pas trouvée dans le dépôt Docker, remplacez `$VERSION_CODENAME` par le nom de code de la dernière Ubuntu LTS supportée. Demandez au formateur le nom à utiliser.

**Validation** : `apt update` se termine sans erreur liée à Docker.

---

### 1.3 Installer Docker Engine et Docker Compose

```bash
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

**Vérifier les versions installées** :

```bash
docker --version
docker compose version
```

**Résultat attendu** :

```text
Docker version 27.x.x, build ...
Docker Compose version v2.x.x
```

---

### 1.4 Démarrer Docker et activer au boot

```bash
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker
```

**Validation** : le statut affiche `Active: active (running)`.

---

### 1.5 Utiliser Docker sans `sudo`

Par défaut, chaque commande Docker nécessite `sudo`. Pour l'éviter, ajoutez votre utilisateur au groupe `docker` :

```bash
sudo usermod -aG docker $USER
```

**Appliquer le changement dans le terminal courant** :

```bash
newgrp docker
```

**Vérification finale** :

```bash
docker run hello-world
```

**Résultat attendu** : un message `Hello from Docker!` s'affiche. Docker est installé et fonctionnel.

---

## 🐋 Partie 2 : Concepts fondamentaux

### 2.1 Image vs Conteneur

| Concept | Définition | Analogie |
|---|---|---|
| **Image** | Modèle figé contenant l'application et ses dépendances | Recette de cuisine |
| **Conteneur** | Instance en cours d'exécution d'une image | Plat cuisiné |
| **Registry** | Dépôt d'images (Docker Hub par défaut) | Bibliothèque de recettes |
| **Dockerfile** | Fichier texte décrivant comment construire une image | La recette elle-même |

---

### 2.2 Premières commandes

**Télécharger une image depuis Docker Hub** :

```bash
docker pull nginx:alpine
```

**Lister les images locales** :

```bash
docker images
```

**Lancer un conteneur en arrière-plan** :

```bash
docker run -d -p 8080:80 --name mon-nginx nginx:alpine
```

- `-d` : mode détaché (arrière-plan) ;
- `-p 8080:80` : port hôte `8080` → port conteneur `80` ;
- `--name mon-nginx` : nom lisible du conteneur.

**Vérifier que le conteneur tourne** :

```bash
docker ps
```

**Tester** :

```bash
curl http://localhost:8080
```

**Arrêter et supprimer le conteneur** :

```bash
docker stop mon-nginx
docker rm mon-nginx
```

**Validation** : `docker ps` ne doit plus afficher `mon-nginx`.

---

## 🐍 Partie 3 : Application Flask avec Dockerfile

### 3.1 Préparer le dossier de travail

```bash
mkdir -p /home/PRENOM/TP05-Docker/mon-flask
cd /home/PRENOM/TP05-Docker/mon-flask
pwd
```

**Validation** : `pwd` affiche `/home/PRENOM/TP05-Docker/mon-flask`.

---

### 3.2 Écrire l'application Flask

```bash
cat > app.py << 'EOF'
from flask import Flask
import os

app = Flask(__name__)

@app.route('/')
def hello():
    env = os.environ.get('APP_ENV', 'développement')
    return f"Hello depuis Flask dans Docker ! Environnement : {env}\n"

@app.route('/sante')
def sante():
    return "OK\n"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
```

**Vérifier le fichier** :

```bash
cat app.py
```

---

### 3.3 Écrire le fichier de dépendances

```bash
cat > requirements.txt << 'EOF'
flask
EOF
```

---

### 3.4 Écrire le Dockerfile

```bash
cat > Dockerfile << 'EOF'
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
EOF
```

**Explication ligne par ligne** :

| Instruction | Rôle |
|---|---|
| `FROM python:3.12-slim` | Image de base (Python 3.12 allégée) |
| `WORKDIR /app` | Dossier de travail dans le conteneur |
| `COPY requirements.txt .` | Copie les dépendances dans le conteneur |
| `RUN pip install ...` | Installe les dépendances au moment du build |
| `COPY app.py .` | Copie le code de l'application |
| `EXPOSE 5000` | Documente le port utilisé par l'application |
| `CMD [...]` | Commande lancée au démarrage du conteneur |

**Vérifier l'arborescence** :

```bash
ls -l
```

**Résultat attendu** :

```text
-rw-r--r-- 1 PRENOM PRENOM  ... Dockerfile
-rw-r--r-- 1 PRENOM PRENOM  ... app.py
-rw-r--r-- 1 PRENOM PRENOM  ... requirements.txt
```

---

### 3.5 Construire l'image

```bash
docker build -t mon-flask:v1 .
```

- `-t mon-flask:v1` : nom et tag de l'image ;
- `.` : répertoire contenant le Dockerfile (répertoire courant).

**Vérifier que l'image apparaît** :

```bash
docker images
```

**Résultat attendu** :

```text
REPOSITORY   TAG   IMAGE ID       CREATED          SIZE
mon-flask    v1    xxxxxxxxxxxx   X seconds ago    ...
```

---

### 3.6 Lancer et tester le conteneur

**Lancer** :

```bash
docker run -d -p 5000:5000 --name mon-flask mon-flask:v1
```

**Vérifier** :

```bash
docker ps
```

**Tester les deux routes** :

```bash
curl http://localhost:5000
curl http://localhost:5000/sante
```

**Résultat attendu** :

```text
Hello depuis Flask dans Docker ! Environnement : développement
OK
```

---

### 3.7 Inspecter le conteneur

**Lire les logs** :

```bash
docker logs mon-flask
```

**Suivre les logs en temps réel** (Ctrl+C pour quitter) :

```bash
docker logs -f mon-flask
```

**Entrer dans le conteneur** :

```bash
docker exec -it mon-flask bash
```

Dans le conteneur :

```bash
ls /app
python --version
exit
```

---

### 3.8 Passer une variable d'environnement

Arrêtez et relancez le conteneur avec une variable d'environnement :

```bash
docker stop mon-flask
docker rm mon-flask

docker run -d -p 5000:5000 -e APP_ENV=production --name mon-flask mon-flask:v1
curl http://localhost:5000
```

**Résultat attendu** :

```text
Hello depuis Flask dans Docker ! Environnement : production
```

**Nettoyage** :

```bash
docker stop mon-flask
docker rm mon-flask
```

---

## 📋 Partie 4 : Cheatsheet Docker

### Images

```bash
docker images                          # Lister les images locales
docker pull IMAGE:TAG                  # Télécharger une image depuis Docker Hub
docker build -t NOM:TAG .              # Construire une image depuis un Dockerfile
docker rmi NOM:TAG                     # Supprimer une image locale
docker image prune                     # Supprimer les images inutilisées
docker tag NOM:TAG NOM:NEWTAG          # Renommer/re-tagger une image
```

### Conteneurs — cycle de vie

```bash
docker run -d -p HOTE:CONT --name NOM IMAGE   # Lancer en arrière-plan
docker run -it IMAGE bash                      # Lancer en mode interactif
docker run --rm IMAGE commande                 # Lancer et supprimer après exécution
docker ps                                      # Conteneurs en cours d'exécution
docker ps -a                                   # Tous les conteneurs (arrêtés inclus)
docker stop NOM                                # Arrêter proprement
docker start NOM                               # Démarrer un conteneur arrêté
docker restart NOM                             # Redémarrer
docker rm NOM                                  # Supprimer un conteneur arrêté
docker rm -f NOM                               # Forcer la suppression
docker container prune                         # Supprimer tous les conteneurs arrêtés
```

### Logs et inspection

```bash
docker logs NOM                        # Voir les logs
docker logs -f NOM                     # Suivre les logs en temps réel
docker logs --tail 50 NOM              # Dernières 50 lignes
docker inspect NOM                     # Détails complets (JSON)
docker stats                           # CPU / RAM en temps réel
docker top NOM                         # Processus actifs dans le conteneur
```

### Exécution dans un conteneur

```bash
docker exec -it NOM bash               # Ouvrir un shell dans le conteneur
docker exec NOM commande               # Exécuter une commande sans shell
docker cp NOM:/chemin/fichier ./local  # Copier depuis le conteneur
docker cp ./local NOM:/chemin/         # Copier vers le conteneur
```

### Volumes

```bash
docker volume ls                             # Lister les volumes
docker volume create mon-volume              # Créer un volume nommé
docker volume rm mon-volume                  # Supprimer un volume
docker run -v mon-volume:/data IMAGE         # Monter un volume nommé
docker run -v /chemin/local:/data IMAGE      # Monter un dossier local (bind mount)
docker volume prune                          # Supprimer les volumes inutilisés
```

### Réseaux

```bash
docker network ls                      # Lister les réseaux
docker network inspect bridge          # Inspecter un réseau
```

### Nettoyage global

```bash
docker system df                       # Espace disque utilisé par Docker
docker system prune                    # Supprimer tout ce qui est inutilisé
docker system prune -a                 # Tout supprimer (images comprises)
```

---

## 🐳 Partie 5 : Docker Compose — WordPress

### 5.1 Concept de Docker Compose

**Docker Compose** permet de décrire et lancer plusieurs conteneurs ensemble à partir d'un fichier `docker-compose.yml`. C'est l'outil standard pour les applications multi-services.

| Commande | Rôle |
|---|---|
| `docker compose up -d` | Démarrer tous les services en arrière-plan |
| `docker compose down` | Arrêter et supprimer les conteneurs et réseaux |
| `docker compose ps` | Lister l'état des services |
| `docker compose logs -f` | Suivre les logs de tous les services |
| `docker compose logs SERVICE` | Logs d'un service spécifique |
| `docker compose exec SERVICE bash` | Entrer dans un conteneur du compose |
| `docker compose restart` | Redémarrer tous les services |

---

### 5.2 Préparer le dossier WordPress

```bash
mkdir -p /home/PRENOM/TP05-Docker/wordpress
cd /home/PRENOM/TP05-Docker/wordpress
pwd
```

**Validation** : `pwd` affiche `/home/PRENOM/TP05-Docker/wordpress`.

> **Note** : si le port `8080` est déjà utilisé sur votre machine (par exemple par un service du TP précédent), arrêtez-le ou changez le port dans le fichier `docker-compose.yml` ci-dessous.

---

### 5.3 Écrire le fichier `docker-compose.yml`

```bash
cat > docker-compose.yml << 'EOF'
services:

  db:
    image: mysql:8.0
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass
    volumes:
      - db_data:/var/lib/mysql

  wordpress:
    image: wordpress:latest
    restart: always
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wp_data:/var/www/html
    depends_on:
      - db

volumes:
  db_data:
  wp_data:
EOF
```

**Explication des sections** :

| Section | Rôle |
|---|---|
| `services` | Déclare les conteneurs à lancer |
| `db` | Conteneur MySQL 8.0 |
| `wordpress` | Conteneur WordPress, connecté à `db` |
| `environment` | Variables de configuration (identifiants BDD) |
| `volumes` | Données persistantes : survivent aux redémarrages |
| `depends_on` | WordPress attend que `db` soit démarré |
| `ports: "8080:80"` | WordPress accessible sur `http://localhost:8080` |

**Vérifier le fichier** :

```bash
cat docker-compose.yml
```

---

### 5.4 Lancer WordPress

```bash
docker compose up -d
```

Docker télécharge les images si elles ne sont pas encore locales, puis démarre les deux conteneurs.

**Observer le démarrage** :

```bash
docker compose ps
```

**Résultat attendu** (après quelques secondes) :

```text
NAME                    IMAGE              STATUS
wordpress-db-1          mysql:8.0          Up X seconds
wordpress-wordpress-1   wordpress:latest   Up X seconds
```

**Suivre les logs de démarrage** (Ctrl+C pour quitter) :

```bash
docker compose logs -f
```

---

### 5.5 Accéder à WordPress

Ouvrez un navigateur et allez sur :

```text
http://localhost:8080
```

L'assistant d'installation WordPress s'affiche. Suivez les étapes :
1. Choisissez la langue
2. Renseignez le titre du site, un identifiant admin et un mot de passe
3. Cliquez sur **Installer WordPress**

**Validation** : la page d'accueil WordPress s'affiche après l'installation.

---

### 5.6 Inspecter les services

**Logs du service WordPress uniquement** :

```bash
docker compose logs wordpress
```

**Logs de la base de données** :

```bash
docker compose logs db
```

**Entrer dans le conteneur WordPress** :

```bash
docker compose exec wordpress bash
ls /var/www/html
exit
```

**Entrer dans MySQL et vérifier les tables** :

```bash
docker compose exec db mysql -u wpuser -pwppass wordpress
```

Dans MySQL :

```sql
SHOW TABLES;
EXIT;
```

**Validation** : vous voyez les tables WordPress (`wp_posts`, `wp_users`, etc.).

---

### 5.7 Arrêter et relancer

**Arrêter les conteneurs sans supprimer les données** :

```bash
docker compose stop
```

**Relancer** :

```bash
docker compose start
```

**Arrêter et supprimer conteneurs + réseaux (données conservées)** :

```bash
docker compose down
```

**Arrêter et supprimer tout, y compris les volumes (données perdues)** :

```bash
docker compose down -v
```

⚠️ `docker compose down -v` supprime définitivement les volumes. Toutes les données WordPress et MySQL seront perdues.

---

## 🎓 Partie 6 : Exercices progressifs

### Exercice 1 : Nouvelle route Flask ⭐

1. Allez dans `/home/PRENOM/TP05-Docker/mon-flask`
2. Modifiez `app.py` pour ajouter une route `/version` qui retourne `v2.0\n`
3. Reconstruisez l'image avec le tag `mon-flask:v2`
4. Lancez un conteneur depuis `mon-flask:v2`
5. Vérifiez avec `curl http://localhost:5000/version`

---

### Exercice 2 : Fichier de logs persistant ⭐⭐

Lancez le conteneur Flask en montant un dossier local pour conserver les logs :

```bash
mkdir -p /home/PRENOM/TP05-Docker/logs
docker run -d -p 5000:5000 \
  -v /home/PRENOM/TP05-Docker/logs:/logs \
  --name mon-flask mon-flask:v1
```

Modifiez `app.py` pour écrire chaque requête dans `/logs/acces.log`, relancez, testez avec `curl`, puis vérifiez le fichier log depuis l'hôte :

```bash
cat /home/PRENOM/TP05-Docker/logs/acces.log
```

---

### Exercice 3 : Compose Flask + Nginx ⭐⭐⭐

Créez un fichier `docker-compose.yml` dans un nouveau dossier `~/TP05-Docker/flask-nginx` qui lance :
- votre image `mon-flask:v1` (port interne `5000`, sans exposition directe) ;
- un conteneur Nginx comme reverse proxy, exposé sur le port `80`.

Dans la configuration Nginx, le proxy doit passer les requêtes vers `http://flask:5000` (le nom du service dans Compose).

---

## ✅ Checklist de validation finale

Avant de conclure ce TP, vérifiez que vous pouvez :

- [ ] Installer Docker Engine via le dépôt officiel apt
- [ ] Lancer `docker run hello-world` sans `sudo`
- [ ] Télécharger une image avec `docker pull`
- [ ] Lister les images avec `docker images` et les conteneurs avec `docker ps`
- [ ] Construire une image Flask avec `docker build`
- [ ] Lancer un conteneur avec `-d`, `-p`, `--name`, `-e`
- [ ] Lire les logs avec `docker logs` et entrer dans un conteneur avec `docker exec -it`
- [ ] Arrêter et supprimer un conteneur avec `docker stop` et `docker rm`
- [ ] Déployer WordPress avec `docker compose up -d`
- [ ] Accéder à WordPress sur `http://localhost:8080`
- [ ] Arrêter les services avec `docker compose down`

---

## 📌 Récapitulatif des commandes importantes

### Installation

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
```

### Images et conteneurs

```bash
docker images
docker pull IMAGE:TAG
docker build -t NOM:TAG .
docker run -d -p HOTE:CONT --name NOM IMAGE
docker run -d -p HOTE:CONT -e VAR=valeur --name NOM IMAGE
docker ps
docker ps -a
docker stop NOM
docker rm NOM
docker logs NOM
docker logs -f NOM
docker exec -it NOM bash
docker system prune
```

### Docker Compose

```bash
docker compose up -d
docker compose down
docker compose down -v
docker compose ps
docker compose logs -f
docker compose logs SERVICE
docker compose exec SERVICE bash
docker compose stop
docker compose start
docker compose restart
```
