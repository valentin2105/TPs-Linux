# TP07 Linux - Sécurité : Scan, Attaque et Protection

**Niveau** : Avancé  
**Prérequis** : TP06 complété (box01 = Nextcloud fonctionnel, box02 = MariaDB configuré), connaissances réseau de base  
**Convention du TP** : le dossier personnel (`HOME`) de l'apprenant est `/home/PRENOM`. Remplacez `PRENOM` par votre prénom dans les chemins absolus.

---

## 📚 Objectifs pédagogiques

À la fin de ce TP, vous saurez :
- ✅ Scanner un réseau et identifier les services exposés avec nmap
- ✅ Comprendre le fonctionnement d'une attaque par brute force sur une BDD et une application web
- ✅ Mettre en place fail2ban pour bloquer automatiquement les intrusions
- ✅ Analyser le trafic réseau avec tcpdump et mesurer l'impact d'un chiffrement absent
- ✅ Durcir une infrastructure Linux avec UFW et les bonnes pratiques MariaDB

---

## 🗺️ Architecture du TP

```
Machine hôte (Ubuntu 26.04 Desktop)

  Réseau privé 192.168.56.0/24

  ┌──────────────────────────────┐
  │  box01 — 192.168.56.10       │  ← CIBLE WEB
  │  Debian 13 · Apache · PHP    │
  │  Nextcloud · fail2ban (→P5)  │
  └──────────────────────────────┘

  ┌──────────────────────────────┐
  │  box02 — 192.168.56.20       │  ← CIBLE BDD
  │  Debian 13 · MariaDB         │
  └──────────────────────────────┘

  ┌──────────────────────────────┐
  │  malicious — 192.168.56.30   │  ← ATTAQUANT
  │  Debian 13 · nmap · hydra    │
  └──────────────────────────────┘
```

> **Cadre légal** : les outils de scan et de brute force utilisés ici (nmap, hydra) sont légaux uniquement sur des systèmes dont vous êtes propriétaire ou sur lesquels vous avez une autorisation explicite. Leur utilisation contre des systèmes tiers est un délit pénal. Ce TP se déroule intégralement dans un réseau isolé Vagrant.

---

## 🖥️ Partie 1 : Ajouter la VM "malicious"

### 1.1 Modifier le Vagrantfile

Depuis le dossier TP06 sur l'hôte :

```bash
cd /home/PRENOM/TP06-Nextcloud
nano Vagrantfile
```

Ajoutez la VM `malicious` dans le tableau `servers`, après `box02` :

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
  {
    :hostname => "malicious",
    :ip       => "192.168.56.30",
    :box      => "bento/debian-13",
    :ram      => 1024,
    :cpu      => 1
  },
]
```

> **box01 et box02 ne sont pas redémarrées** : Vagrant crée uniquement les VMs nouvelles ou modifiées. `malicious` sera provisionnée seule.

---

### 1.2 Démarrer la VM attaquante

```bash
vagrant up malicious
```

**Vérifier l'état des trois VMs** :

```bash
vagrant status
```

**Résultat attendu** :

```text
box01        running (virtualbox)
box02        running (virtualbox)
malicious    running (virtualbox)
```

---

### 1.3 Installer les outils d'attaque sur malicious

```bash
vagrant ssh malicious
sudo apt update
sudo apt install -y nmap hydra
```

**Vérifier les installations** :

```bash
nmap --version
hydra -h 2>&1 | head -3
```

**Quitter malicious** :

```bash
exit
```

---

## 🔍 Partie 2 : Reconnaissance réseau avec nmap

Toutes les commandes de cette partie s'exécutent **depuis malicious**.

```bash
vagrant ssh malicious
```

---

### 2.1 Découverte des hôtes actifs (ping sweep)

La première étape de tout audit réseau est de cartographier les hôtes actifs.

```bash
sudo nmap -sn 192.168.56.0/24
```

**Résultat attendu** :

```text
Nmap scan report for box01 (192.168.56.10)
Host is up (0.00Xs latency).
Nmap scan report for box02 (192.168.56.20)
Host is up (0.00Xs latency).
Nmap scan report for malicious (192.168.56.30)
Host is up (0.00Xs latency).
Nmap done: 256 IP addresses (3 hosts up)
```

> **Ce que ça révèle** : 3 hôtes actifs, leurs IPs, et leurs noms de machine. Point de départ de toute reconnaissance.

---

### 2.2 Scan complet de box01 — la cible web

**Scan de tous les ports ouverts avec détection de version** :

```bash
sudo nmap -sV -p- --open 192.168.56.10
```

**Résultat attendu** :

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH X.X (Debian Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.X ((Debian))
```

**Scan avec scripts NSE (Nmap Scripting Engine)** :

```bash
sudo nmap -sV -sC 192.168.56.10
```

**Résultat attendu (extrait)** :

```text
80/tcp open  http    Apache httpd 2.4.X
|_http-title: Nextcloud
| http-robots.txt: 14 disallowed entries
|_http-server-header: Apache/2.4.X (Debian)
```

> **Ce que ça révèle** : l'application Nextcloud est identifiée sans authentification. La version d'Apache et d'OpenSSH est visible — suffisant pour chercher des CVE publiées.

---

### 2.3 Scan de box02 — la cible base de données

```bash
sudo nmap -sV -p 22,3306 192.168.56.20
```

**Résultat attendu** :

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH X.X (Debian Linux; protocol 2.0)
3306/tcp open  mysql   MariaDB (unauthorized)
```

**Script dédié à MySQL/MariaDB** :

```bash
sudo nmap -p 3306 --script mysql-info 192.168.56.20
```

**Résultat attendu** :

```text
3306/tcp open  mysql
| mysql-info:
|   Protocol: 10
|   Version: X.X.XX-MariaDB-X+debian
|   Thread ID: X
|   Capabilities flags: XXXXX
|   Status: Autocommit
```

> **Problème identifié** : le port 3306 de MariaDB est accessible depuis toute VM du réseau privé. L'attaquant connaît le SGBD et sa version exacte — suffisant pour rechercher des vulnérabilités connues.

---

### 2.4 Détection du système d'exploitation

```bash
sudo nmap -O 192.168.56.10
sudo nmap -O 192.168.56.20
```

**Résultat attendu** :

```text
OS details: Linux 5.X - 6.X
```

> La détection d'OS est approximative dans les VMs VirtualBox mais confirme le système cible (Linux, noyau 5.x ou 6.x).

---

### 2.5 Résumé de la surface d'attaque identifiée

| VM | IP | Ports ouverts | Services exposés |
|---|---|---|---|
| box01 | 192.168.56.10 | 22, 80 | SSH (OpenSSH), HTTP (Apache + Nextcloud) |
| box02 | 192.168.56.20 | 22, 3306 | SSH (OpenSSH), MySQL (MariaDB) |

**Quitter malicious** :

```bash
exit
```

---

## 🎯 Partie 3 : Mise en place de cibles vulnérables

> **Contexte pédagogique** : Dans cette partie, nous créons délibérément des comptes avec des mots de passe faibles pour simuler une mauvaise pratique réelle. Ces comptes seront supprimés en Partie 6.

---

### 3.1 Créer un compte MariaDB vulnérable (sur box02)

```bash
vagrant ssh box02

sudo mysql << 'EOF'
CREATE USER 'dbadmin'@'%' IDENTIFIED BY 'password123';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'dbadmin'@'%';
FLUSH PRIVILEGES;
EOF
```

> `'dbadmin'@'%'` autorise ce compte depuis n'importe quelle IP source — mauvaise pratique délibérée pour la démonstration.

**Vérifier** :

```bash
sudo mysql -e "SELECT User, Host FROM mysql.user WHERE User = 'dbadmin';"
```

**Résultat attendu** :

```text
+---------+------+
| User    | Host |
+---------+------+
| dbadmin | %    |
+---------+------+
```

**Quitter box02** :

```bash
exit
```

---

### 3.2 Créer un compte Nextcloud vulnérable (sur box01)

```bash
vagrant ssh box01
cd /var/www/nextcloud

OC_PASS="support123" sudo -E -u www-data php occ user:add --password-from-env support
```

**Vérifier** :

```bash
sudo -u www-data php occ user:list
```

**Résultat attendu** :

```text
  - admin: admin
  - support: support
```

**Désactiver temporairement la protection anti-brute-force native de Nextcloud** :

> Nextcloud intègre un mécanisme qui ralentit progressivement les tentatives depuis une même IP. Nous le désactivons pour que la démonstration soit visible — fail2ban remplacera ce rôle avec une approche plus robuste (ban au niveau firewall).

```bash
sudo -u www-data php occ config:system:set auth.bruteforce.protection.enabled \
  --value='false' --type=boolean
```

**Quitter box01** :

```bash
exit
```

---

## ⚔️ Partie 4 : Attaques par brute force

Toutes les commandes de cette partie s'exécutent **depuis malicious**.

```bash
vagrant ssh malicious
```

---

### 4.1 Créer la wordlist d'attaque

Une wordlist est un fichier de mots de passe courants. En situation réelle, les attaquants utilisent des listes comme `rockyou.txt` (14 millions d'entrées). Pour ce TP, nous créons une liste courte qui contient les mots de passe cibles.

```bash
cat > /tmp/passwords.txt << 'EOF'
admin
password
root
123456
nextcloud
mariadb
support123
password123
qwerty
letmein
secret
debian
vagrant
EOF
```

---

### 4.2 Brute force sur MariaDB (hydra)

Hydra teste chaque mot de passe de la liste contre le compte `dbadmin` sur le port 3306 de box02.

```bash
hydra -l dbadmin -P /tmp/passwords.txt mysql://192.168.56.20
```

**Résultat attendu** :

```text
[DATA] max 4 tasks per 1 server, overall 4 tasks, 13 login tries
[DATA] attacking mysql://192.168.56.20:3306/
[3306][mysql] host: 192.168.56.20   login: dbadmin   password: password123
1 of 1 target successfully completed, 1 valid password found
```

> **Analyse** : le mot de passe a été trouvé en testant seulement quelques combinaisons. Avec `rockyou.txt` et plusieurs threads (`-t 16`), un attaquant trouverait un mot de passe commun en quelques secondes.

**Vérifier l'accès obtenu** :

```bash
mysql -u dbadmin -ppassword123 -h 192.168.56.20 nextcloud \
  -e "SHOW TABLES;" 2>/dev/null | head -10
```

> L'attaquant a maintenant un accès complet en lecture/écriture à la base Nextcloud : données utilisateurs, fichiers, tokens de session.

---

### 4.3 Brute force sur Nextcloud (WebDAV via hydra)

Nextcloud expose une API WebDAV sur `/remote.php/dav/`. Cette interface utilise l'**authentification HTTP Basic** — directement ciblable par hydra, sans CSRF ni JavaScript requis.

```bash
hydra -l support -P /tmp/passwords.txt 192.168.56.10 \
  http-get /remote.php/dav/files/support/ -t 4
```

**Résultat attendu** :

```text
[DATA] attacking http://192.168.56.10:80/remote.php/dav/files/support/
[80][http-get] host: 192.168.56.10   login: support   password: support123
1 of 1 target successfully completed, 1 valid password found
```

**Vérifier l'accès obtenu** :

```bash
curl -s -u support:support123 \
  http://192.168.56.10/remote.php/dav/files/support/ | head -20
```

**Résultat attendu** : une réponse XML WebDAV listant les fichiers du compte — l'attaquant a accès aux données.

---

### 4.4 Observer les traces laissées côté victime

**Sur box01**, depuis un second terminal, observez les logs générés par l'attaque :

```bash
vagrant ssh box01
sudo tail -30 /var/log/apache2/nextcloud_access.log
```

**Résultat attendu** :

```text
192.168.56.30 - - [DATE] "GET /remote.php/dav/files/support/ HTTP/1.1" 401 ...
192.168.56.30 - - [DATE] "GET /remote.php/dav/files/support/ HTTP/1.1" 401 ...
192.168.56.30 - - [DATE] "GET /remote.php/dav/files/support/ HTTP/1.1" 401 ...
...
192.168.56.30 - - [DATE] "GET /remote.php/dav/files/support/ HTTP/1.1" 207 ...
```

> Chaque `401 Unauthorized` est une tentative échouée. La ligne `207` (Multi-Status WebDAV) correspond au mot de passe correct. Toutes ces lignes mentionnent l'IP source `192.168.56.30` : c'est ce que fail2ban va exploiter.

**Quitter box01** :

```bash
exit
```

---

## 🛡️ Partie 5 : Protection avec fail2ban

fail2ban surveille les fichiers de log et bannit automatiquement les IPs qui génèrent trop d'erreurs dans une fenêtre de temps donnée. Il agit en ajoutant des règles iptables/nftables temporaires.

Toutes les commandes de cette partie s'exécutent **sur box01**.

```bash
vagrant ssh box01
```

---

### 5.1 Installer fail2ban

```bash
sudo apt update
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
sudo systemctl status fail2ban
```

**Validation** : `Active: active (running)`

---

### 5.2 Créer le filtre Nextcloud

Le filtre définit l'expression régulière que fail2ban recherche dans les logs pour détecter une tentative échouée.

```bash
sudo nano /etc/fail2ban/filter.d/nextcloud-webdav.conf
```

Contenu :

```ini
[Definition]
failregex = ^<HOST> .* "(GET|POST|PUT|DELETE|PROPFIND|HEAD) /remote\.php/.* HTTP/.*" 401
ignoreregex =
```

> `<HOST>` est un marqueur spécial de fail2ban remplacé par le regex de détection d'adresse IP. Le filtre matche toute requête vers l'API WebDAV qui retourne un code `401`.

**Tester le filtre sur les logs existants** :

```bash
sudo fail2ban-regex /var/log/apache2/nextcloud_access.log \
  /etc/fail2ban/filter.d/nextcloud-webdav.conf
```

**Résultat attendu** :

```text
Results
=======
Failregex: X match(es)
Ignoreregex: 0 match(es)
```

> Le nombre de matches doit correspondre aux tentatives échouées de la Partie 4. Si le compteur est à 0, vérifiez que le fichier de log contient bien des lignes `401`.

---

### 5.3 Créer le jail fail2ban

Le jail associe un filtre à un fichier de log et définit la politique de bannissement.

```bash
sudo nano /etc/fail2ban/jail.d/nextcloud.conf
```

Contenu :

```ini
[nextcloud-webdav]
enabled  = true
port     = http,https
filter   = nextcloud-webdav
logpath  = /var/log/apache2/nextcloud_access.log
maxretry = 5
findtime = 60
bantime  = 300
```

| Paramètre | Valeur | Signification |
|---|---|---|
| `maxretry` | 5 | 5 échecs déclenchent le ban |
| `findtime` | 60 | fenêtre d'analyse de 60 secondes |
| `bantime` | 300 | IP bannie pendant 5 minutes |

**Recharger fail2ban** :

```bash
sudo systemctl reload fail2ban
```

**Vérifier que le jail est actif** :

```bash
sudo fail2ban-client status
```

**Résultat attendu** :

```text
Status
|- Number of jail:      1
`- Jail list:   nextcloud-webdav
```

**Détail du jail** :

```bash
sudo fail2ban-client status nextcloud-webdav
```

**Résultat attendu** :

```text
Status for the jail: nextcloud-webdav
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     X
|  `- File list:        /var/log/apache2/nextcloud_access.log
`- Actions
   |- Currently banned: 0
   |- Total banned:     0
   `- Banned IP list:
```

**Quitter box01** :

```bash
exit
```

---

### 5.4 Démonstration du blocage en temps réel

Préparez **deux terminaux** simultanément.

**Terminal 1 — Surveiller fail2ban (box01)** :

```bash
vagrant ssh box01
sudo journalctl -fu fail2ban
```

Laissez ce terminal ouvert et visible.

**Terminal 2 — Relancer l'attaque (malicious)** :

```bash
vagrant ssh malicious
hydra -l support -P /tmp/passwords.txt 192.168.56.10 \
  http-get /remote.php/dav/files/support/ -t 4
```

**Dans le terminal fail2ban (Terminal 1), vous verrez apparaître** :

```text
[nextcloud-webdav] Found 192.168.56.30
[nextcloud-webdav] Found 192.168.56.30
[nextcloud-webdav] Found 192.168.56.30
[nextcloud-webdav] Found 192.168.56.30
[nextcloud-webdav] Found 192.168.56.30
[nextcloud-webdav] Ban 192.168.56.30
```

**Vérifier le ban sur box01** :

```bash
sudo fail2ban-client status nextcloud-webdav
```

**Résultat attendu** :

```text
`- Actions
   |- Currently banned: 1
   `- Banned IP list:   192.168.56.30
```

**Constater l'effet du ban depuis malicious** :

```bash
curl -m 5 http://192.168.56.10/ 2>&1
```

**Résultat attendu** : `curl: (28) Operation timed out` — la connexion est bloquée au niveau du firewall, plus aucune réponse HTTP ne parvient à l'attaquant.

**Débannir manuellement pour continuer le TP** :

```bash
# Sur box01
sudo fail2ban-client set nextcloud-webdav unbanip 192.168.56.30
```

---

## 🔬 Partie 6 : Analyse réseau avancée

### 6.1 Capture de trafic avec tcpdump — MySQL en clair

> MariaDB transmet les données **sans chiffrement** par défaut. Un attaquant capable d'écouter le réseau (ARP spoofing, accès à un switch managé, VM compromise sur le même réseau) voit les requêtes SQL et les mots de passe en clair.

**Sur box02 — Identifier l'interface réseau host-only** :

```bash
vagrant ssh box02
ip a | grep -B1 '192.168.56'
```

**Résultat attendu** : une interface nommée `eth1` ou `enp0s8` (VirtualBox).

**Lancer la capture sur le port 3306** :

```bash
sudo tcpdump -i eth1 -A -s 0 port 3306 2>/dev/null
```

Laissez cette capture active.

**Sur box01 (autre terminal) — Exécuter une requête SQL** :

```bash
vagrant ssh box01
mysql -u ncuser -pncpassword -h 192.168.56.20 nextcloud \
  -e "SELECT uid, displayname FROM oc_users LIMIT 3;" 2>/dev/null
```

**Dans la capture tcpdump sur box02, observez** :

```text
...ncpassword...
...SELECT uid, displayname FROM oc_users LIMIT 3...
...admin...
```

> Le mot de passe `ncpassword` et les données de la table `oc_users` apparaissent en clair. Sur un réseau réel partagé (Wi-Fi d'entreprise, VLAN compromis), toute personne en position d'écoute passive récupère ces informations.

**Stopper tcpdump** : `Ctrl+C`

**Quitter box02** :

```bash
exit
```

---

### 6.2 Durcissement du firewall avec UFW

UFW (Uncomplicated Firewall) restreint les connexions entrantes aux seuls ports légitimes et depuis les seules sources autorisées.

**Sur box01 — Autoriser uniquement SSH et HTTP** :

```bash
vagrant ssh box01

sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
sudo ufw enable
```

> `ufw enable` peut afficher un avertissement sur la connexion SSH en cours. La règle `allow ssh` garantit que la session ne sera pas coupée.

**Vérifier les règles actives** :

```bash
sudo ufw status verbose
```

**Résultat attendu** :

```text
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
```

**Quitter box01** :

```bash
exit
```

**Sur box02 — Restreindre MariaDB à box01 uniquement** :

```bash
vagrant ssh box02

sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow from 192.168.56.10 to any port 3306
sudo ufw enable
```

**Vérifier** :

```bash
sudo ufw status verbose
```

**Résultat attendu** :

```text
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
3306/tcp                   ALLOW IN    192.168.56.10
```

**Quitter box02** :

```bash
exit
```

**Depuis malicious — Vérifier que le port 3306 est maintenant inaccessible** :

```bash
vagrant ssh malicious
nmap -p 3306 192.168.56.20
```

**Résultat attendu** :

```text
PORT     STATE    SERVICE
3306/tcp filtered mysql
```

> `filtered` signifie que le firewall absorbe la sonde sans répondre. Avant UFW, le port était `open`. L'attaquant ne peut plus atteindre MariaDB depuis `malicious`.

**Vérifier que box01 peut toujours accéder à MariaDB** :

```bash
vagrant ssh box01
mysql -u ncuser -pncpassword -h 192.168.56.20 nextcloud \
  -e "SELECT 1 AS acces_ok;" 2>/dev/null
```

**Résultat attendu** : `1` — le trafic légitime box01 → box02 n'est pas bloqué.

---

### 6.3 Durcissement MariaDB et audit des comptes

**Sur box02** :

```bash
vagrant ssh box02
```

**Supprimer le compte vulnérable créé pour la démonstration** :

```bash
sudo mysql -e "DROP USER 'dbadmin'@'%'; FLUSH PRIVILEGES;"
```

**Audit complet des comptes MariaDB** :

```bash
sudo mysql -e "SELECT User, Host, plugin FROM mysql.user ORDER BY User;"
```

**Résultat attendu (uniquement les comptes légitimes)** :

```text
+-------------+---------------+-----------------------+
| User        | Host          | plugin                |
+-------------+---------------+-----------------------+
| mariadb.sys | localhost     | mysql_native_password |
| mysql       | localhost     | mysql_native_password |
| ncuser      | 192.168.56.10 | mysql_native_password |
| root        | localhost     | unix_socket           |
+-------------+---------------+-----------------------+
```

> `root` utilise `unix_socket` : authentification via le système d'exploitation, pas de mot de passe transmis sur le réseau.  
> `ncuser` est restreint à une seule IP source : une éventuelle fuite de ce mot de passe ne permet pas de connexion depuis une autre machine.

**Vérifier l'absence d'utilisateurs anonymes** :

```bash
sudo mysql -e "SELECT User, Host FROM mysql.user WHERE User = '';"
```

**Résultat attendu** : aucune ligne.

**Quitter box02** :

```bash
exit
```

---

### 6.4 Nettoyage et réactivation des protections Nextcloud

**Sur box01** :

```bash
vagrant ssh box01
cd /var/www/nextcloud

# Supprimer le compte de démonstration
sudo -u www-data php occ user:delete support

# Réactiver la protection anti-brute-force native de Nextcloud
sudo -u www-data php occ config:system:set auth.bruteforce.protection.enabled \
  --value='true' --type=boolean

# Vérifier
sudo -u www-data php occ config:system:get auth.bruteforce.protection.enabled
```

**Résultat attendu** : `true`

**Quitter box01** :

```bash
exit
```

---

## 🧪 Partie 7 : Exercices progressifs

### Exercice 1 : Scan NSE avancé ⭐

Depuis malicious, utilisez les scripts nmap spécialisés pour extraire davantage d'informations :

```bash
vagrant ssh malicious

# En-têtes HTTP de box01
sudo nmap --script http-headers 192.168.56.10

# Méthodes HTTP autorisées sur WebDAV
sudo nmap -p 80 --script http-methods \
  --script-args http-methods.url-path=/remote.php/dav/ 192.168.56.10
```

Analysez ce que chaque script révèle et réfléchissez à comment ces informations seraient exploitées par un attaquant réel.

---

### Exercice 2 : Jail fail2ban pour SSH ⭐⭐

Configurez un second jail fail2ban sur box01 pour protéger SSH :

1. Créez `/etc/fail2ban/jail.d/sshd.conf` avec le filtre intégré `sshd`
2. Paramétrez `maxretry = 3`, `bantime = 600`, `findtime = 120`
3. Rechargez fail2ban et vérifiez que le jail `sshd` apparaît dans `fail2ban-client status`
4. Depuis malicious, tentez des connexions SSH avec un mauvais mot de passe :

```bash
for i in {1..4}; do
  ssh -o StrictHostKeyChecking=no -o PasswordAuthentication=yes \
    vagrant@192.168.56.10 exit 2>/dev/null || true
done
```

Vérifiez que l'IP est bannie avec `sudo fail2ban-client status sshd`.

---

### Exercice 3 : Lire les logs JSON de Nextcloud ⭐⭐

Nextcloud conserve ses propres logs dans `/var/www/nextcloud/data/nextcloud.log` (format JSON, un objet par ligne).

```bash
vagrant ssh box01
sudo tail -20 /var/www/nextcloud/data/nextcloud.log | \
  python3 -c "import sys,json; [print(json.dumps(json.loads(l), indent=2)) for l in sys.stdin]" \
  2>/dev/null | head -60
```

Identifiez les champs `remoteAddr`, `level`, `message` et analysez ce que Nextcloud a enregistré lors des attaques de la Partie 4.

---

### Exercice 4 : Filtre fail2ban sur les logs Nextcloud ⭐⭐⭐

En complément du filtre WebDAV (logs Apache), créez un filtre qui analyse directement les logs JSON de Nextcloud pour détecter les `Login failed`.

Structure d'un log Nextcloud (simplifié) :
```json
{"remoteAddr":"192.168.56.30","user":"--","message":"Login failed: 'support' (Remote IP: '192.168.56.30')"}
```

1. Créez `/etc/fail2ban/filter.d/nextcloud-app.conf` avec ce failregex :
```ini
[Definition]
failregex = .*"remoteAddr":"<HOST>".*"message":"Login failed
ignoreregex =
```

2. Créez un jail correspondant dans `/etc/fail2ban/jail.d/nextcloud-app.conf` qui surveille `/var/www/nextcloud/data/nextcloud.log`

3. Testez le filtre avec `fail2ban-regex` sur le fichier de log

---

### Exercice 5 : Comparer avant/après UFW avec nmap ⭐⭐

Recréez le tableau de la surface d'attaque (section 2.5) en relançant les scans nmap depuis malicious **après** l'activation d'UFW sur box01 et box02.

Comparez les états `open` / `filtered` / `closed` des ports et analysez ce que l'attaquant peut encore voir.

---

## 🩺 Partie 8 : Dépannage

### fail2ban ne banne pas l'IP après l'attaque

```bash
# Vérifier que le jail est actif
sudo fail2ban-client status

# Tester le filtre sur les logs réels
sudo fail2ban-regex /var/log/apache2/nextcloud_access.log \
  /etc/fail2ban/filter.d/nextcloud-webdav.conf

# Vérifier les logs fail2ban
sudo journalctl -u fail2ban -n 50

# Vérifier que le logpath existe et n'est pas vide
ls -la /var/log/apache2/nextcloud_access.log
sudo wc -l /var/log/apache2/nextcloud_access.log
```

---

### UFW bloque des connexions légitimes

```bash
# Voir les règles avec leurs numéros
sudo ufw status numbered

# Supprimer une règle par son numéro
sudo ufw delete 3

# Désactiver UFW temporairement pour diagnostic
sudo ufw disable

# Ajouter une règle manquante
sudo ufw allow from 192.168.56.10 to any port 3306
sudo ufw reload
```

---

### hydra ne trouve pas le mot de passe

```bash
# Vérifier que le port est accessible depuis malicious
nc -zv 192.168.56.20 3306
nc -zv 192.168.56.10 80

# Tester la connexion WebDAV manuellement
curl -v -u support:support123 \
  http://192.168.56.10/remote.php/dav/files/support/ 2>&1 | grep "< HTTP"

# Vérifier que le compte existe côté MariaDB
vagrant ssh box02
sudo mysql -e "SELECT User, Host FROM mysql.user WHERE User='dbadmin';"
```

---

### Le filtre fail2ban retourne 0 matches

```bash
# Vérifier le format des logs (les lignes doivent contenir l'IP et le code 401)
sudo grep '401' /var/log/apache2/nextcloud_access.log | tail -5

# Vérifier le failregex avec des logs de test
echo '192.168.56.30 - - [01/Jan/2025:10:00:00 +0000] "GET /remote.php/dav/files/support/ HTTP/1.1" 401 401' | \
  sudo fail2ban-regex - /etc/fail2ban/filter.d/nextcloud-webdav.conf
```

---

## ✅ Checklist de validation finale

- [ ] `vagrant status` : box01, box02 et malicious en `running`
- [ ] Depuis malicious : `sudo nmap -sn 192.168.56.0/24` détecte les 3 VMs
- [ ] `nmap -sV 192.168.56.10` identifie Apache et Nextcloud
- [ ] `nmap -sV -p 3306 192.168.56.20` identifie MariaDB avant UFW
- [ ] Hydra a trouvé `password123` sur le compte `dbadmin` MariaDB
- [ ] Hydra a trouvé `support123` sur le compte `support` Nextcloud WebDAV
- [ ] Les logs Apache montrent les `401` de l'attaque hydra
- [ ] fail2ban est actif sur box01 (`sudo fail2ban-client status`)
- [ ] `fail2ban-regex` sur les logs Apache retourne des matches
- [ ] Après 5 tentatives hydra, fail2ban banne 192.168.56.30
- [ ] `curl` depuis malicious retourne un timeout après le ban
- [ ] UFW est actif sur box01 (`sudo ufw status`)
- [ ] UFW est actif sur box02 avec règle `3306 ALLOW 192.168.56.10`
- [ ] `nmap -p 3306 192.168.56.20` depuis malicious retourne `filtered` après UFW
- [ ] box01 peut toujours accéder à MariaDB via `ncuser` (trafic légitime non bloqué)
- [ ] Le compte `dbadmin` a été supprimé de MariaDB
- [ ] Le compte `support` a été supprimé de Nextcloud
- [ ] La protection anti-brute-force Nextcloud est réactivée (`true`)

---

## 📌 Récapitulatif des commandes importantes

### nmap (depuis malicious)

```bash
sudo nmap -sn 192.168.56.0/24                          # Découverte des hôtes
sudo nmap -sV -p- --open <IP>                           # Scan complet + versions
sudo nmap -sV -sC <IP>                                  # Scan + scripts NSE par défaut
sudo nmap -O <IP>                                       # Détection OS
sudo nmap -p 3306 --script mysql-info <IP>              # Info spécifique MariaDB
sudo nmap --script http-headers <IP>                    # En-têtes HTTP
```

### hydra (depuis malicious)

```bash
hydra -l USER -P wordlist.txt mysql://IP                        # Brute force MariaDB
hydra -l USER -P wordlist.txt IP http-get /chemin/              # Brute force HTTP Basic
hydra -l USER -P wordlist.txt IP http-get /chemin/ -t 4         # Avec 4 threads
```

### fail2ban (sur box01)

```bash
sudo fail2ban-client status                              # Jails actifs
sudo fail2ban-client status <jail>                       # Détail d'un jail
sudo fail2ban-client set <jail> unbanip <IP>             # Débannir une IP
sudo fail2ban-regex <logfile> <filter>                   # Tester un filtre
sudo journalctl -fu fail2ban                             # Logs en temps réel
sudo systemctl reload fail2ban                           # Recharger la config
```

### UFW (sur box01 / box02)

```bash
sudo ufw default deny incoming                           # Refuser par défaut
sudo ufw allow ssh                                       # Autoriser SSH
sudo ufw allow http                                      # Autoriser HTTP
sudo ufw allow from <IP> to any port <PORT>              # Règle source spécifique
sudo ufw enable                                          # Activer le firewall
sudo ufw disable                                         # Désactiver
sudo ufw status verbose                                  # Voir les règles
sudo ufw status numbered                                 # Voir les règles numérotées
sudo ufw delete <NUM>                                    # Supprimer une règle
```

### tcpdump (sur box02)

```bash
sudo tcpdump -i eth1 -A -s 0 port 3306                 # Capturer trafic MySQL en clair
sudo tcpdump -i eth1 -w /tmp/capture.pcap port 3306    # Sauvegarder en fichier pcap
```

### MariaDB — audit et nettoyage (sur box02)

```bash
sudo mysql -e "SELECT User, Host, plugin FROM mysql.user ORDER BY User;"
sudo mysql -e "SELECT User, Host FROM mysql.user WHERE User = '';"
sudo mysql -e "DROP USER 'USER'@'HOST'; FLUSH PRIVILEGES;"
```
