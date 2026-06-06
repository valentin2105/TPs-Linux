# TP02 Linux - Services

**Niveau** : Débutant / intermédiaire  
**Prérequis** : Accès à une machine Ubuntu, terminal ouvert, droits `sudo`  
**Convention du TP** : le dossier personnel (`HOME`) de l'apprenant est `/home/PRENOM`. Remplacez `PRENOM` par votre prénom dans les chemins absolus.

---

## 📚 Objectifs pédagogiques

À la fin de ce TP, vous saurez :
- ✅ Rechercher, consulter, installer et bloquer des paquets avec `apt`
- ✅ Lister les paquets installés avec `dpkg -l`
- ✅ Observer et contrôler des processus avec `ps`, `top`, `htop`, `kill`, `nice`, `jobs`, `nohup`
- ✅ Gérer un service avec `systemctl`
- ✅ Lire les logs d'un service avec `journalctl`
- ✅ Écrire une unit file systemd personnalisée
- ✅ Créer un service `mon-app.service` qui démarre au boot
- ✅ Exposer une petite application Python qui répond `Hello World`

---

## 🧰 Partie 1 : Gérer les paquets sur Ubuntu

### 1.1 Mettre à jour le catalogue des paquets

**Concept** : `apt` utilise un catalogue local pour connaître les paquets disponibles. Avant d'installer ou de chercher un paquet, il est conseillé de le mettre à jour.

**Commande à taper** :
```bash
sudo apt update
```

**Validation** : La commande doit se terminer sans erreur et afficher que les listes de paquets ont été lues.

---

### 1.2 Rechercher un paquet avec `apt search`

**Concept** : `apt search` permet de chercher un paquet par nom ou mot-clé.

**Commandes à taper** :
```bash
apt search htop
apt search python3
```

**À observer** :
- le nom du paquet ;
- une courte description ;
- l'état éventuel `[installé]` si le paquet est déjà présent.

**Validation** : Vous devez trouver un paquet nommé `htop`.

---

### 1.3 Afficher les détails d'un paquet avec `apt show`

**Concept** : `apt show` affiche les métadonnées d'un paquet : version, dépendances, taille, description.

**Commande à taper** :
```bash
apt show htop
```

**Points à repérer** :
- `Package` : nom du paquet ;
- `Version` : version disponible ;
- `Depends` : dépendances ;
- `Description` : description longue.

**Validation** : Vous savez expliquer à quoi sert le paquet `htop` avant de l'installer.

---

### 1.4 Installer des paquets avec `apt install`

**Concept** : `apt install` installe un paquet et ses dépendances. L'installation modifie le système, donc elle nécessite `sudo`.

**Commandes à taper** :
```bash
sudo apt install -y htop curl python3
```

**Vérification** :
```bash
htop --version
curl --version
python3 --version
```

**Validation** : Les trois commandes affichent une version.

---

### 1.5 Lister les paquets installés avec `dpkg -l`

**Concept** : `dpkg -l` liste les paquets installés ou connus par le système.

**Commandes à taper** :
```bash
dpkg -l

dpkg -l | grep htop

dpkg -l | grep python3
```

**À comprendre** :
- `ii` signifie que le paquet est installé correctement ;
- le nom du paquet apparaît dans la deuxième colonne ;
- la version apparaît dans la troisième colonne.

**Validation** : La ligne de `htop` doit commencer par `ii`.

---

### 1.6 Bloquer et débloquer une version avec `apt-mark hold`

**Concept** : `apt-mark hold` empêche un paquet d'être mis à jour automatiquement. C'est utile quand une version précise doit être conservée.

**Commandes à taper** :
```bash
sudo apt-mark hold htop
apt-mark showhold
```

**Validation** : `htop` doit apparaître dans la liste des paquets bloqués.

**Débloquer le paquet** :
```bash
sudo apt-mark unhold htop
apt-mark showhold
```

**Validation** : `htop` ne doit plus apparaître dans la liste.

---

## ⚙️ Partie 2 : Observer et contrôler les processus

### 2.1 Afficher les processus avec `ps aux`

**Concept** : Un processus est un programme en cours d'exécution. Chaque processus possède un identifiant unique : le `PID`.

**Commandes à taper** :
```bash
ps aux
ps aux | head
ps aux | grep python
```

**Colonnes importantes** :
- `USER` : utilisateur qui exécute le processus ;
- `PID` : identifiant du processus ;
- `%CPU` : usage processeur ;
- `%MEM` : usage mémoire ;
- `COMMAND` : commande lancée.

**Validation** : Vous savez repérer le `PID` d'un processus dans la sortie de `ps aux`.

---

### 2.2 Surveiller avec `top` et `htop`

**Concept** : `top` et `htop` affichent les processus de façon interactive.

**Commandes à taper** :
```bash
top
```

Dans `top` :
- appuyez sur `q` pour quitter.

**Lancer `htop`** :
```bash
htop
```

Dans `htop` :
- utilisez les flèches pour naviguer ;
- appuyez sur `F10` ou `q` pour quitter.

**Validation** : Vous savez identifier les processus qui consomment le plus de CPU ou de mémoire.

---

### 2.3 Lancer un processus en arrière-plan avec `&`

**Concept** : Ajouter `&` à la fin d'une commande lance le processus en arrière-plan et libère le terminal.

**Commande à taper** :
```bash
sleep 1000 &
```

**Résultat attendu** :
```text
[1] 12345
```

Le premier nombre est le numéro de job. Le second est le `PID`.

**Afficher les jobs du terminal courant** :
```bash
jobs
```

**Validation** : Le job `sleep 1000` apparaît dans la liste.

---

### 2.4 Arrêter un processus avec `kill`

**Concept** : `kill` envoie un signal à un processus. Par défaut, il demande un arrêt propre.

**Étape 1 : lancer un processus**
```bash
sleep 1000 &
```

**Étape 2 : récupérer son PID**
```bash
echo $!
```

**Étape 3 : arrêter le processus**
```bash
kill PID
```

Remplacez `PID` par le numéro obtenu.

**Vérification** :
```bash
jobs
ps aux | grep sleep
```

**Arrêt forcé si nécessaire** :
```bash
kill -9 PID
```

**Validation** : Le processus `sleep` n'est plus actif.

---

### 2.5 Modifier la priorité avec `nice`

**Concept** : `nice` lance un processus avec une priorité ajustée. Plus la valeur est élevée, moins le processus est prioritaire.

**Commande à taper** :
```bash
nice -n 10 sleep 1000 &
```

**Afficher la priorité** :
```bash
ps -o pid,ni,cmd -p $!
```

**À observer** : La colonne `NI` doit afficher `10`.

**Validation** : Vous savez lancer un processus avec une priorité réduite.

---

### 2.6 Lancer un processus avec `nohup`

**Concept** : `nohup` permet à un processus de continuer même si le terminal est fermé.

**Commande à taper** :
```bash
nohup sleep 1000 > /home/PRENOM/nohup-sleep.log 2>&1 &
```

**Vérifier le processus** :
```bash
ps aux | grep sleep
```

**Vérifier le fichier de sortie** :
```bash
ls -l /home/PRENOM/nohup-sleep.log
```

**Nettoyage** :
```bash
pkill -f "sleep 1000"
```

**Validation** : Vous comprenez la différence entre `commande &` et `nohup commande &`.

---

## 🐍 Partie 3 : Créer une application Python `Hello World`

### 3.1 Préparer le dossier de l'application

**Objectif** : Créer une petite application HTTP dans `/home/PRENOM/mon-app`.

**Commandes à taper** :
```bash
cd /home/PRENOM
mkdir -p mon-app
cd mon-app
pwd
```

**Validation** : `pwd` doit afficher :
```text
/home/PRENOM/mon-app
```

---

### 3.2 Écrire l'application Python

**Commande à taper** :
```bash
cat > app.py << 'EOF'
#!/usr/bin/env python3

from http.server import BaseHTTPRequestHandler, HTTPServer
import os

HOST = "0.0.0.0"
PORT = 8080

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        message = "Hello World\n"
        self.send_response(200)
        self.send_header("Content-Type", "text/plain; charset=utf-8")
        self.send_header("Content-Length", str(len(message.encode("utf-8"))))
        self.end_headers()
        self.wfile.write(message.encode("utf-8"))
        print(f"GET {self.path} from {self.client_address[0]} pid={os.getpid()}", flush=True)

if __name__ == "__main__":
    print(f"mon-app listening on {HOST}:{PORT} pid={os.getpid()}", flush=True)
    HTTPServer((HOST, PORT), Handler).serve_forever()
EOF
```

**Rendre le script exécutable** :
```bash
chmod +x app.py
```

**Validation** :
```bash
ls -l app.py
```

Le fichier doit avoir le droit d'exécution (`x`).

---

### 3.3 Tester l'application manuellement

**Terminal 1 : lancer l'application**
```bash
cd /home/PRENOM/mon-app
python3 app.py
```

**Terminal 2 : tester la réponse HTTP**
```bash
curl http://localhost:8080
```

**Résultat attendu** :
```text
Hello World
```

**Arrêter l'application dans le terminal 1** :
```text
Ctrl+C
```

**Validation** : L'application répond correctement avant d'être transformée en service.

---

## 🧩 Partie 4 : Créer le service systemd `mon-app.service`

### 4.1 Comprendre une unit file `.service`

**Concept** : systemd gère les services avec des fichiers appelés *unit files*. Pour un service système, ils sont placés dans `/etc/systemd/system/`.

Une unit file contient généralement :
- `[Unit]` : description et dépendances ;
- `[Service]` : commande à lancer, utilisateur, comportement ;
- `[Install]` : configuration du démarrage automatique.

---

### 4.2 Écrire le fichier `/etc/systemd/system/mon-app.service`

**Important** : Remplacez `PRENOM` par votre prénom dans la commande ci-dessous avant de l'exécuter.

**Commande à taper** :
```bash
sudo tee /etc/systemd/system/mon-app.service > /dev/null << 'EOF'
[Unit]
Description=Mon application Python Hello World
After=network.target

[Service]
Type=simple
User=PRENOM
WorkingDirectory=/home/PRENOM/mon-app
ExecStart=/usr/bin/python3 /home/PRENOM/mon-app/app.py
Restart=always
RestartSec=3
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF
```

**Vérifier le contenu** :
```bash
sudo systemctl cat mon-app
```

**Validation** : Le fichier doit contenir :
```text
User=PRENOM
WorkingDirectory=/home/PRENOM/mon-app
ExecStart=/usr/bin/python3 /home/PRENOM/mon-app/app.py
```

Dans votre machine, `PRENOM` doit être remplacé par votre vrai nom d'utilisateur.

---

### 4.3 Recharger systemd et démarrer le service

**Concept** : Après création ou modification d'une unit file, il faut demander à systemd de relire sa configuration.

**Commandes à taper** :
```bash
sudo systemctl daemon-reload
sudo systemctl start mon-app
sudo systemctl status mon-app
```

**À observer dans `status`** :
- `Loaded: loaded` ;
- `Active: active (running)` ;
- le chemin `/etc/systemd/system/mon-app.service`.

**Tester l'application** :
```bash
curl http://localhost:8080
```

**Résultat attendu** :
```text
Hello World
```

**Validation** : Le service est lancé par systemd et l'application répond.

---

### 4.4 Arrêter, relancer et redémarrer le service

**Commandes utiles** :
```bash
sudo systemctl stop mon-app
sudo systemctl status mon-app

sudo systemctl start mon-app
sudo systemctl status mon-app

sudo systemctl restart mon-app
sudo systemctl status mon-app
```

**Validation** :
- après `stop`, le service n'est plus actif ;
- après `start`, le service redevient actif ;
- après `restart`, le service est actif avec un nouveau processus Python.

---

### 4.5 Activer le démarrage au boot

**Concept** : `enable` configure le service pour qu'il démarre automatiquement au démarrage de la machine.

**Commandes à taper** :
```bash
sudo systemctl enable mon-app
systemctl is-enabled mon-app
```

**Résultat attendu** :
```text
enabled
```

**Validation** : Le service `mon-app.service` est configuré pour démarrer au boot.

**Désactivation si besoin** :
```bash
sudo systemctl disable mon-app
```

---

## 📜 Partie 5 : Lire les logs avec `journalctl`

### 5.1 Consulter les logs du service

**Concept** : systemd envoie les sorties standard et erreur du service dans le journal système. `journalctl` permet de les lire.

**Commandes à taper** :
```bash
sudo journalctl -u mon-app
sudo journalctl -u mon-app -n 30
```

**À observer** :
- le message de démarrage `mon-app listening on ...` ;
- les lignes `GET / ...` générées par les requêtes HTTP ;
- les éventuelles erreurs Python.

**Validation** : Les logs de votre application sont visibles.

---

### 5.2 Suivre les logs en continu

**Terminal 1 : suivre les logs**
```bash
sudo journalctl -u mon-app -f
```

**Terminal 2 : générer des requêtes**
```bash
curl http://localhost:8080
curl http://localhost:8080/test
```

**Arrêter le suivi dans le terminal 1** :
```text
Ctrl+C
```

**Validation** : Chaque requête doit produire une ligne de log visible dans `journalctl`.

---

## 🧪 Partie 6 : Dépannage guidé

### 6.1 Le service ne démarre pas

**Commandes de diagnostic** :
```bash
sudo systemctl status mon-app
sudo journalctl -u mon-app -n 50
```

**Causes fréquentes** :
- `PRENOM` n'a pas été remplacé ;
- le fichier `/home/PRENOM/mon-app/app.py` n'existe pas ;
- Python n'est pas installé ;
- le port `8080` est déjà utilisé ;
- l'unit file a été modifiée sans `daemon-reload`.

---

### 6.2 Vérifier le port utilisé

**Commande à taper** :
```bash
sudo ss -ltnp | grep ':8080'
```

**À comprendre** : Si une ligne apparaît, un processus écoute sur le port `8080`.

**Arrêter le service puis vérifier** :
```bash
sudo systemctl stop mon-app
sudo ss -ltnp | grep ':8080'
```

**Validation** : Après arrêt du service, le port `8080` doit être libéré.

---

### 6.3 Corriger et recharger une unit file

**Éditer le service** :
```bash
sudo nano /etc/systemd/system/mon-app.service
```

**Recharger et relancer** :
```bash
sudo systemctl daemon-reload
sudo systemctl restart mon-app
sudo systemctl status mon-app
```

**Validation** : Toute correction du fichier `.service` est suivie d'un `daemon-reload`.

---

## 🎓 Partie 7 : Livrable final

### 7.1 Ce que vous devez rendre

Votre livrable est un service Linux complet nommé `mon-app.service`.

Il doit respecter les critères suivants :
- ✅ le fichier `/home/PRENOM/mon-app/app.py` existe ;
- ✅ l'application Python répond `Hello World` sur `http://localhost:8080` ;
- ✅ le fichier `/etc/systemd/system/mon-app.service` existe ;
- ✅ le service démarre avec `sudo systemctl start mon-app` ;
- ✅ le service est actif selon `sudo systemctl status mon-app` ;
- ✅ le service est activé au boot avec `sudo systemctl enable mon-app` ;
- ✅ les logs sont consultables avec `sudo journalctl -u mon-app` ;
- ✅ les logs peuvent être suivis avec `sudo journalctl -u mon-app -f`.

---

### 7.2 Vérification finale

**Commandes à exécuter** :
```bash
systemctl is-enabled mon-app
systemctl is-active mon-app
curl http://localhost:8080
sudo journalctl -u mon-app -n 10
```

**Résultats attendus** :
```text
enabled
active
Hello World
```

**Validation finale** : Si ces résultats sont obtenus, le TP est réussi.

---

## 📌 Récapitulatif des commandes importantes

### Paquets
```bash
sudo apt update
apt search htop
apt show htop
sudo apt install -y htop curl python3
sudo apt-mark hold htop
apt-mark showhold
sudo apt-mark unhold htop
dpkg -l
dpkg -l | grep htop
```

### Processus
```bash
ps aux
top
htop
sleep 1000 &
jobs
echo $!
kill PID
kill -9 PID
nice -n 10 sleep 1000 &
nohup sleep 1000 > /home/PRENOM/nohup-sleep.log 2>&1 &
```

### Service systemd
```bash
sudo systemctl daemon-reload
sudo systemctl start mon-app
sudo systemctl stop mon-app
sudo systemctl restart mon-app
sudo systemctl enable mon-app
sudo systemctl disable mon-app
sudo systemctl status mon-app
systemctl is-enabled mon-app
systemctl is-active mon-app
```

### Logs
```bash
sudo journalctl -u mon-app
sudo journalctl -u mon-app -n 30
sudo journalctl -u mon-app -f
```
