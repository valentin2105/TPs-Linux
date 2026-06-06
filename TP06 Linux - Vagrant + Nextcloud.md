# TP06 Linux - Vagrant + Nextcloud

**Niveau** : Intermédiaire  
**Prérequis** : Ubuntu 26.04 Desktop, VirtualBox et Vagrant installés, connexion Internet, au moins **6 Go de RAM disponible**, TP01 à TP03 maîtrisés  
**Convention du TP** : le dossier personnel (`HOME`) de l'apprenant est `/home/PRENOM`. Remplacez `PRENOM` par votre prénom dans les chemins absolus.

---

## 📚 Objectifs pédagogiques

À la fin de ce TP, vous saurez :
- ✅ Lire et adapter un `Vagrantfile` multi-VMs en Ruby
- ✅ Gérer les VMs avec `vagrant up`, `ssh`, `halt`, `destroy`
- ✅ Configurer MariaDB sur une VM dédiée avec accès réseau restreint
- ✅ Installer Nextcloud sur une VM Apache + PHP
- ✅ Connecter Nextcloud à une base de données sur une VM séparée
- ✅ Accéder à Nextcloud depuis le navigateur de la machine hôte

---

## 🗺️ Architecture du TP

```
Machine hôte (Ubuntu 26.04 Desktop)
│
│  http://localhost:8080 ──────────────┐
│                                      │
│   Réseau privé 192.168.56.0/24      │
│  ┌──────────────────────────────┐   │
│  │  box01 — 192.168.56.10       │◄──┘
│  │  Debian 13 · 3 vCPU · 3 Go  │
│  │  Apache + PHP + Nextcloud    │
│  └──────────┬───────────────────┘
│             │ MariaDB :3306
│  ┌──────────▼───────────────────┐
│  │  box02 — 192.168.56.20       │
│  │  Debian 13 · 2 vCPU · 2 Go  │
│  │  MariaDB · base : nextcloud  │
│  └──────────────────────────────┘
```

**Pourquoi MariaDB ?**  
Nextcloud recommande officiellement MariaDB comme base de données principale pour ses performances sur les requêtes de métadonnées de fichiers et sa gestion des jeux de caractères `utf8mb4`.

---

## 🧰 Partie 1 : Préparer l'environnement

### 1.1 Créer le dossier de travail

```bash
mkdir -p /home/PRENOM/TP06-Nextcloud
cd /home/PRENOM/TP06-Nextcloud
```

---

### 1.2 Écrire le Vagrantfile

Le Vagrantfile suit le même format que le dépôt VagrantLab : une liste de machines Ruby, un bloc de configuration partagé, et un script de provisionnement externe.

```bash
nano Vagrantfile
```

Contenu :

```ruby
servers = [
  {
    :hostname => "box01",
    :ip       => "192.168.56.10",
    :box      => "bento/debian-13",
    :ram      => 3072,
    :cpu      => 3
  },
  {
    :hostname => "box02",
    :ip       => "192.168.56.20",
    :box      => "bento/debian-13",
    :ram      => 2048,
    :cpu      => 2
  },
]

Vagrant.configure("2") do |config|
  config.vm.box_check_update = false

  servers.each do |machine|
    config.vm.define machine[:hostname] do |node|
      node.vm.box      = machine[:box]
      node.vm.hostname = machine[:hostname]

      # Réseau host-only pour le lab
      node.vm.network "private_network", ip: machine[:ip]

      # Port forwarding uniquement pour box01 (Nextcloud → navigateur hôte)
      if machine[:hostname] == "box01"
        node.vm.network "forwarded_port", guest: 80, host: 8080
      end

      # Ressources VirtualBox
      node.vm.provider "virtualbox" do |vb|
        vb.customize ["modifyvm", :id, "--cpus",   machine[:cpu]]
        vb.customize ["modifyvm", :id, "--memory", machine[:ram]]
      end

      # Provisionnement initial
      node.vm.provision "shell",
        path: "bootstrap.sh",
        env: { "DEBIAN_FRONTEND" => "noninteractive" }
    end
  end
end
```

**Points clés du Vagrantfile** :

| Élément | Rôle |
|---|---|
| `bento/debian-13` | Image Debian 13 (Trixie) stable avec VirtualBox Guest Additions |
| `box_check_update = false` | Ne vérifie pas les mises à jour de la box à chaque `vagrant up` |
| `private_network` | Réseau interne isolé entre les VMs |
| `forwarded_port` | Accès Nextcloud depuis le navigateur hôte sur le port 8080 |
| `vb.customize` | Allocation CPU et RAM via les outils VirtualBox CLI |
| `bootstrap.sh` | Script exécuté automatiquement au premier démarrage |

---

### 1.3 Écrire le script `bootstrap.sh`

Le script `bootstrap.sh` est exécuté automatiquement par Vagrant au premier `vagrant up`. Il prépare l'environnement de base de chaque VM.

```bash
nano bootstrap.sh
```

Contenu :

```bash
#!/bin/bash
set -e

apt-get update -qq
apt-get install -y -qq \
  curl \
  wget \
  vim \
  htop \
  net-tools \
  ca-certificates \
  gnupg \
  lsb-release

echo "==> Bootstrap terminé sur $(hostname) ($(date))"
```

---

### 1.4 Démarrer les VMs

```bash
vagrant up
```

> **Première exécution** : Vagrant télécharge la box `bento/debian-13` (~500 Mo) puis exécute `bootstrap.sh` sur chaque VM. Cela peut prendre plusieurs minutes.

**Vérifier l'état des VMs** :

```bash
vagrant status
```

**Résultat attendu** :

```text
box01                     running (virtualbox)
box02                     running (virtualbox)
```

---

### 1.5 Vérifier la connectivité réseau entre les VMs

**Connexion à box01** :

```bash
vagrant ssh box01
```

**Pinguer box02 depuis box01** :

```bash
ping -c 4 192.168.56.20
```

**Validation** : les 4 paquets passent sans perte. Si le ping échoue, vérifiez que VirtualBox a bien créé l'interface `vboxnet0` sur l'hôte.

**Quitter box01** :

```bash
exit
```

---

## 🗄️ Partie 2 : Configurer box02 — MariaDB

Toutes les commandes de cette partie s'exécutent **depuis box02**.

```bash
vagrant ssh box02
```

---

### 2.1 Installer MariaDB

```bash
sudo apt update
sudo apt install -y mariadb-server
sudo systemctl enable --now mariadb
sudo systemctl status mariadb
```

**Validation** : le statut affiche `Active: active (running)`.

---

### 2.2 Créer la base de données et l'utilisateur Nextcloud

MariaDB sur Debian utilise l'authentification `unix_socket` pour root : `sudo mysql` ouvre un accès root sans mot de passe.

```bash
sudo mysql << 'EOF'
CREATE DATABASE nextcloud
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_general_ci;

CREATE USER 'ncuser'@'192.168.56.10'
  IDENTIFIED BY 'ncpassword';

GRANT ALL PRIVILEGES ON nextcloud.*
  TO 'ncuser'@'192.168.56.10';

FLUSH PRIVILEGES;
EOF
```

**Vérifier** :

```bash
sudo mysql -e "SELECT User, Host FROM mysql.user WHERE User = 'ncuser';"
sudo mysql -e "SHOW DATABASES LIKE 'nextcloud';"
```

**Résultat attendu** :

```text
+--------+--------------+
| User   | Host         |
+--------+--------------+
| ncuser | 192.168.56.10|
+--------+--------------+

+--------------------+
| Database (nextcloud|
+--------------------+
| nextcloud          |
+--------------------+
```

---

### 2.3 Autoriser les connexions distantes depuis box01

Par défaut, MariaDB n'écoute que sur `127.0.0.1`. Il faut lui faire écouter sur l'IP du réseau privé.

**Modifier la configuration** :

```bash
sudo sed -i 's/^bind-address\s*=.*/bind-address = 192.168.56.20/' \
  /etc/mysql/mariadb.conf.d/50-server.cnf
```

**Vérifier la modification** :

```bash
grep 'bind-address' /etc/mysql/mariadb.conf.d/50-server.cnf
```

**Résultat attendu** :

```text
bind-address = 192.168.56.20
```

**Redémarrer MariaDB** :

```bash
sudo systemctl restart mariadb
```

**Vérifier que MariaDB écoute sur le bon port** :

```bash
sudo ss -ltnp | grep ':3306'
```

**Résultat attendu** :

```text
LISTEN  0  ...  192.168.56.20:3306  ...  mariadbd
```

**Quitter box02** :

```bash
exit
```

---

### 2.4 Vérifier la connexion depuis box01

```bash
vagrant ssh box01
```

**Installer le client MariaDB sur box01** :

```bash
sudo apt update
sudo apt install -y mariadb-client
```

**Tester la connexion vers box02** :

```bash
mysql -u ncuser -pncpassword -h 192.168.56.20 nextcloud -e "SELECT 1 AS connexion_ok;"
```

**Résultat attendu** :

```text
+--------------+
| connexion_ok |
+--------------+
|            1 |
+--------------+
```

**Validation** : box01 peut se connecter à la base `nextcloud` sur box02. Si la connexion échoue, consultez la section Dépannage.

---

## 🌐 Partie 3 : Configurer box01 — Nextcloud

Toutes les commandes de cette partie s'exécutent **depuis box01** (`vagrant ssh box01`).

---

### 3.1 Installer Apache et PHP avec les extensions requises

```bash
sudo apt update
sudo apt install -y \
  apache2 \
  libapache2-mod-php \
  php \
  php-mysql \
  php-gd \
  php-curl \
  php-xml \
  php-zip \
  php-intl \
  php-mbstring \
  php-bcmath \
  php-gmp \
  php-apcu \
  php-imagick
```

**Activer les modules Apache nécessaires** :

```bash
sudo a2enmod rewrite headers env dir mime
sudo systemctl restart apache2
```

**Vérifier la version PHP installée** :

```bash
php --version
```

**Résultat attendu** : PHP 8.2.x (version par défaut sur Debian 13)

---

### 3.2 Configurer PHP pour Nextcloud

Les valeurs par défaut de PHP ne sont pas adaptées à Nextcloud.

**Modifier les paramètres** :

```bash
PHP_INI=/etc/php/8.2/apache2/php.ini

sudo sed -i 's/^memory_limit = .*/memory_limit = 512M/'              "$PHP_INI"
sudo sed -i 's/^upload_max_filesize = .*/upload_max_filesize = 512M/' "$PHP_INI"
sudo sed -i 's/^post_max_size = .*/post_max_size = 512M/'            "$PHP_INI"
sudo sed -i 's/^max_execution_time = .*/max_execution_time = 300/'   "$PHP_INI"
sudo sed -i 's/^max_input_time = .*/max_input_time = 300/'           "$PHP_INI"
```

**Vérifier les valeurs appliquées** :

```bash
grep -E '^(memory_limit|upload_max_filesize|post_max_size|max_execution_time)' \
  /etc/php/8.2/apache2/php.ini
```

**Résultat attendu** :

```text
memory_limit = 512M
upload_max_filesize = 512M
post_max_size = 512M
max_execution_time = 300
```

**Activer APCu pour PHP-CLI** (nécessaire pour les commandes `occ`) :

```bash
echo 'apc.enable_cli=1' | sudo tee -a /etc/php/8.2/mods-available/apcu.ini
sudo phpenmod apcu
```

**Redémarrer Apache** :

```bash
sudo systemctl restart apache2
```

---

### 3.3 Télécharger et déployer Nextcloud

**Télécharger la dernière version stable** :

```bash
cd /tmp
wget https://download.nextcloud.com/server/releases/latest.tar.bz2
```

> Cette archive fait environ 200 Mo. La vitesse dépend de la connexion Internet de la VM.

**Extraire et déplacer dans le répertoire web** :

```bash
tar -xjf latest.tar.bz2
sudo mv nextcloud /var/www/nextcloud
```

**Configurer les permissions** :

```bash
sudo chown -R www-data:www-data /var/www/nextcloud
sudo chmod -R 750 /var/www/nextcloud
```

**Vérifier** :

```bash
ls -ld /var/www/nextcloud
ls /var/www/nextcloud
```

**Résultat attendu** :

```text
drwxr-x--- ... www-data www-data ... /var/www/nextcloud

3rdparty  AUTHORS  config  core  index.php  ...
```

---

### 3.4 Configurer Apache pour Nextcloud

**Créer le VirtualHost** :

```bash
sudo nano /etc/apache2/sites-available/nextcloud.conf
```

Contenu :

```apache
<VirtualHost *:80>
    ServerName  box01
    ServerAlias localhost
    DocumentRoot /var/www/nextcloud

    <Directory /var/www/nextcloud>
        Options +FollowSymlinks
        AllowOverride All
        Require all granted
    </Directory>

    <IfModule mod_headers.c>
        Header always set Strict-Transport-Security "max-age=15552000"
    </IfModule>

    ErrorLog  ${APACHE_LOG_DIR}/nextcloud_error.log
    CustomLog ${APACHE_LOG_DIR}/nextcloud_access.log combined
</VirtualHost>
```

**Activer le site Nextcloud, désactiver le site par défaut** :

```bash
sudo a2ensite nextcloud.conf
sudo a2dissite 000-default.conf
sudo systemctl reload apache2
```

**Vérifier la configuration Apache** :

```bash
sudo apache2ctl configtest
```

**Résultat attendu** : `Syntax OK`

---

### 3.5 Installer Nextcloud via la commande `occ`

`occ` (Nextcloud console) est l'outil CLI officiel de Nextcloud pour l'installation et l'administration. Il évite de passer par l'installateur web.

**Lancer l'installation** :

```bash
cd /var/www/nextcloud

sudo -u www-data php occ maintenance:install \
  --database      "mysql" \
  --database-host "192.168.56.20" \
  --database-name "nextcloud" \
  --database-user "ncuser" \
  --database-pass "ncpassword" \
  --admin-user    "admin" \
  --admin-pass    "Nextcloud2024!" \
  --data-dir      "/var/www/nextcloud/data"
```

> Cette commande prend 1 à 3 minutes. Elle initialise la structure de la base de données et crée les dossiers de données.

**Résultat attendu** :

```text
Nextcloud was successfully installed
```

**Si une erreur apparaît** :
- `Exception: Could not connect to database` → retournez à la section 2.4 et vérifiez la connexion
- `Error while trying to create admin user` → même cause, problème BDD

---

### 3.6 Configurer les domaines de confiance

Nextcloud refuse les connexions depuis des domaines absents de sa liste `trusted_domains`.

```bash
cd /var/www/nextcloud

sudo -u www-data php occ config:system:set trusted_domains 0 --value='localhost'
sudo -u www-data php occ config:system:set trusted_domains 1 --value='192.168.56.10'
```

**Vérifier** :

```bash
sudo -u www-data php occ config:system:get trusted_domains
```

**Résultat attendu** :

```text
localhost
192.168.56.10
```

---

### 3.7 Activer le cache mémoire APCu

```bash
cd /var/www/nextcloud

sudo -u www-data php occ config:system:set memcache.local \
  --value='\OC\Memcache\APCu'
```

**Vérifier** :

```bash
sudo -u www-data php occ config:system:get memcache.local
```

**Résultat attendu** :

```text
\OC\Memcache\APCu
```

---

### 3.8 Vérification finale sur box01

```bash
sudo systemctl status apache2
curl -s -o /dev/null -w "%{http_code}\n" http://localhost/
```

**Résultat attendu** : `200` ou `302` (redirection vers la page de login).

**Quitter box01** :

```bash
exit
```

---

## 🖥️ Partie 4 : Accéder à Nextcloud depuis le navigateur

### 4.1 Ouvrir Nextcloud

> **Prérequis** : Le port 8080 ne doit pas être occupé sur l'hôte. S'il l'est (TP02 ou TP05), arrêtez le service concerné, ou changez le port dans le Vagrantfile puis exécutez `vagrant reload box01`.

Ouvrez un navigateur sur votre machine hôte :

```text
http://localhost:8080
```

**Vous devez voir** : la page de connexion Nextcloud.

---

### 4.2 Se connecter

| Champ | Valeur |
|---|---|
| Identifiant | `admin` |
| Mot de passe | `Nextcloud2024!` |

**Validation** : vous accédez au tableau de bord Nextcloud.

---

### 4.3 Vérifier le statut Nextcloud

```bash
vagrant ssh box01
cd /var/www/nextcloud
sudo -u www-data php occ status
```

**Résultat attendu** :

```text
  - installed: true
  - version: XX.X.X.X
  - versionstring: Nextcloud XX.X.X
  - edition: Community
  - maintenance: false
  - needsDbUpgrade: false
```

---

## 🔧 Partie 5 : Commandes Vagrant utiles

### Gestion des VMs

```bash
vagrant status                  # État de toutes les VMs
vagrant up                      # Démarrer toutes les VMs
vagrant up box01                # Démarrer une VM spécifique
vagrant halt                    # Arrêter toutes les VMs
vagrant halt box02              # Arrêter une VM spécifique
vagrant reload box01            # Redémarrer + relire le Vagrantfile
vagrant suspend                 # Suspendre (libère la RAM)
vagrant resume                  # Reprendre après suspension
vagrant destroy -f              # Supprimer toutes les VMs (irréversible)
vagrant destroy box01 -f        # Supprimer une VM spécifique
```

### Connexion et diagnostic

```bash
vagrant ssh box01               # Se connecter à box01
vagrant ssh box02               # Se connecter à box02
vagrant ssh-config box01        # Afficher la config SSH
vagrant port box01              # Voir les ports redirigés
```

### Snapshots

```bash
vagrant snapshot save box01 avant-nextcloud   # Créer un snapshot nommé
vagrant snapshot list box01                   # Lister les snapshots
vagrant snapshot restore box01 avant-nextcloud # Restaurer
vagrant snapshot delete box01 avant-nextcloud  # Supprimer
```

**Conseil** : créez un snapshot après la validation de chaque partie du TP pour pouvoir revenir en arrière facilement.

---

## 🧪 Partie 6 : Exercices progressifs

### Exercice 1 : Créer un utilisateur Nextcloud via `occ` ⭐

Sans utiliser l'interface web, créez un utilisateur depuis le terminal :

```bash
vagrant ssh box01
cd /var/www/nextcloud
sudo -u www-data php occ user:add --display-name="Apprenant Test" apprenant
```

Nextcloud demande un mot de passe interactivement. Connectez-vous ensuite dans le navigateur avec ce nouvel utilisateur.

---

### Exercice 2 : Sauvegarder la base de données ⭐⭐

Depuis **box02**, exportez la base `nextcloud` dans un fichier de sauvegarde :

```bash
vagrant ssh box02
mysqldump -u ncuser -pncpassword -h 192.168.56.20 nextcloud \
  > /home/vagrant/nextcloud-backup-$(date +%Y%m%d).sql

ls -lh /home/vagrant/nextcloud-backup-*.sql
grep 'CREATE TABLE' /home/vagrant/nextcloud-backup-*.sql | wc -l
```

**Validation** : le dump contient plusieurs dizaines de tables.

---

### Exercice 3 : Ajouter bootstrap.sh au provisionnement Nextcloud ⭐⭐

Modifiez `bootstrap.sh` pour qu'il installe automatiquement les paquets Apache, PHP et MariaDB selon le hostname de la VM :

- Si `hostname == box01` : installe Apache + PHP + extensions
- Si `hostname == box02` : installe MariaDB et configure `bind-address`

Testez avec `vagrant destroy -f && vagrant up`.

**Aide** :

```bash
if [ "$(hostname)" = "box01" ]; then
  # installation Nextcloud
elif [ "$(hostname)" = "box02" ]; then
  # installation MariaDB
fi
```

---

### Exercice 4 : Snapshot avant/après ⭐⭐⭐

1. Créez un snapshot de box01 et box02 **avant** l'installation de Nextcloud (`vagrant snapshot save`)
2. Installez Nextcloud normalement
3. Restaurez les snapshots (`vagrant snapshot restore`)
4. Vérifiez que les VMs reviennent à leur état initial
5. Réinstallez Nextcloud depuis les snapshots

---

## 🩺 Partie 7 : Dépannage

### Nextcloud affiche "Access through untrusted domain"

```bash
vagrant ssh box01
cd /var/www/nextcloud
sudo -u www-data php occ config:system:set trusted_domains 2 --value='DOMAINE_MANQUANT'
sudo -u www-data php occ config:system:get trusted_domains
```

---

### La page Nextcloud ne s'affiche pas (port 8080)

```bash
# Sur l'hôte : vérifier le port forwarding
vagrant port box01

# Sur box01 : diagnostiquer Apache
vagrant ssh box01
sudo systemctl status apache2
sudo apache2ctl configtest
sudo tail -20 /var/log/apache2/nextcloud_error.log
```

---

### Nextcloud ne se connecte pas à MariaDB

```bash
# Depuis box01 : tester la connexion manuellement
vagrant ssh box01
mysql -u ncuser -pncpassword -h 192.168.56.20 nextcloud -e "SELECT 1;"

# Si la connexion échoue, diagnostiquer sur box02 :
vagrant ssh box02
grep 'bind-address' /etc/mysql/mariadb.conf.d/50-server.cnf   # doit être 192.168.56.20
sudo systemctl status mariadb
sudo ss -ltnp | grep ':3306'                                    # doit écouter sur 192.168.56.20
```

---

### OCC retourne une erreur PHP

```bash
vagrant ssh box01
php --version                        # Vérifier PHP 8.2
php -m | grep -E 'pdo_mysql|mysqli'  # Vérifier les extensions BDD
ls /etc/php/8.2/apache2/php.ini      # Vérifier que le fichier de config existe
```

---

## ✅ Checklist de validation finale

- [ ] `vagrant status` affiche `box01` et `box02` en `running`
- [ ] `vagrant ssh box01` et `vagrant ssh box02` fonctionnent
- [ ] `ping -c 2 192.168.56.20` depuis box01 passe sans perte
- [ ] MariaDB écoute sur `192.168.56.20:3306` (`sudo ss -ltnp | grep 3306` sur box02)
- [ ] La base `nextcloud` et l'utilisateur `ncuser` existent sur box02
- [ ] `mysql -u ncuser -pncpassword -h 192.168.56.20 nextcloud` répond depuis box01
- [ ] `sudo apache2ctl configtest` retourne `Syntax OK` sur box01
- [ ] `curl -s -o /dev/null -w "%{http_code}" http://localhost/` retourne `200` ou `302` sur box01
- [ ] `sudo -u www-data php occ status` retourne `installed: true` sur box01
- [ ] Nextcloud est accessible sur `http://localhost:8080` depuis le navigateur hôte
- [ ] La connexion avec `admin` / `Nextcloud2024!` fonctionne

---

## 📌 Récapitulatif des commandes importantes

### Vagrant (depuis l'hôte)

```bash
vagrant up                         # Démarrer les VMs
vagrant halt                       # Arrêter les VMs
vagrant status                     # État des VMs
vagrant ssh box01                  # Se connecter à box01
vagrant ssh box02                  # Se connecter à box02
vagrant reload box01               # Redémarrer + relire le Vagrantfile
vagrant destroy -f                 # Supprimer toutes les VMs
vagrant snapshot save box01 NOM    # Créer un snapshot
vagrant snapshot restore box01 NOM # Restaurer un snapshot
```

### MariaDB (sur box02)

```bash
sudo systemctl status mariadb
sudo mysql                         # Connexion root locale (sans password)
mysql -u ncuser -pncpassword -h 192.168.56.20 nextcloud -e "SHOW TABLES;"
mysqldump -u ncuser -pncpassword -h 192.168.56.20 nextcloud > backup.sql
grep 'bind-address' /etc/mysql/mariadb.conf.d/50-server.cnf
```

### Nextcloud OCC (sur box01, dans /var/www/nextcloud)

```bash
sudo -u www-data php occ status
sudo -u www-data php occ user:list
sudo -u www-data php occ user:add NOM_UTILISATEUR
sudo -u www-data php occ config:system:set trusted_domains N --value='DOMAINE'
sudo -u www-data php occ config:system:get trusted_domains
sudo -u www-data php occ maintenance:mode --on
sudo -u www-data php occ maintenance:mode --off
sudo -u www-data php occ db:add-missing-indices
sudo -u www-data php occ upgrade
```

### Apache (sur box01)

```bash
sudo systemctl status apache2
sudo apache2ctl configtest
sudo a2ensite nextcloud.conf
sudo a2enmod rewrite
sudo tail -f /var/log/apache2/nextcloud_error.log
```
