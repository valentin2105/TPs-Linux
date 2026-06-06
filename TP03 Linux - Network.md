# TP03 Linux - Réseau

**Niveau** : Débutant  
**Prérequis** : VM Ubuntu 26.04, terminal ouvert, droits `sudo`  
**Convention du TP** : le dossier personnel (`HOME`) de l'apprenant est `/home/PRENOM`. Remplacez `PRENOM` par votre prénom dans les chemins absolus.

---

## 📚 Objectifs pédagogiques

À la fin de ce TP, vous saurez :
- ✅ Identifier les interfaces réseau avec `ip a`
- ✅ Lire la table de routage avec `ip r`
- ✅ Voir les ports en écoute avec `ss -tulpn`
- ✅ Tester la connectivité avec `ping`
- ✅ Comprendre un chemin réseau avec `traceroute`
- ✅ Tester la résolution DNS avec `dig` et `host`
- ✅ Comprendre la différence entre `/etc/hosts` et DNS
- ✅ Configurer le réseau d'une VM Ubuntu Server 26.04 avec Netplan
- ✅ Configurer un pare-feu simple avec `ufw`
- ✅ Diagnostiquer et corriger une panne réseau simulée
- ✅ Documenter une démarche de dépannage

---

## 🧭 Scénario du TP

Vous administrez une VM Ubuntu Server 26.04.

Votre mission :
1. observer la configuration réseau actuelle ;
2. configurer proprement le réseau avec Netplan ;
3. tester la connectivité IP et DNS ;
4. activer un pare-feu simple avec `ufw` ;
5. recevoir un *trouble ticket* contenant une panne simulée ;
6. diagnostiquer avant de corriger ;
7. documenter la démarche.

Ubuntu Server 26.04 utilise Netplan pour la configuration réseau persistante. Dans ce TP, toute configuration réseau persistante se fait donc avec Netplan.

---

## ⚠️ Règles de sécurité du TP

Avant toute modification réseau :

```bash
mkdir -p /home/PRENOM/TP03-Reseau/sauvegardes
sudo cp -a /etc/netplan /home/PRENOM/TP03-Reseau/sauvegardes/netplan-original
```

Lorsque vous modifiez Netplan, préférez :

```bash
sudo netplan try
```

Cette commande applique la configuration avec confirmation. Si vous perdez la connexion, elle revient automatiquement en arrière.

---

## 🧰 Partie 1 : Installer les outils utiles

### 1.1 Mettre à jour le catalogue des paquets

```bash
sudo apt update
```

### 1.2 Installer les outils du TP

```bash
sudo apt install -y traceroute dnsutils ufw curl
```

**À quoi servent-ils ?**
- `traceroute` : afficher le chemin réseau vers une destination ;
- `dnsutils` : fournit `dig` et `host` ;
- `ufw` : pare-feu simplifié Ubuntu ;
- `curl` : tester une connexion HTTP.

**Validation** :

```bash
traceroute --version
host -V
ufw version
curl --version
```

---

## 🌐 Partie 2 : Observer l'état réseau de la VM

### 2.1 Identifier les interfaces avec `ip a`

**Commande à taper** :

```bash
ip a
```

**À repérer** :
- `lo` : interface locale de la machine ;
- une interface réseau de VM, souvent nommée `ens33`, `enp0s3`, `ens18` ou similaire ;
- une adresse IPv4, par exemple `192.168.1.50/24` ;
- l'état `UP` si l'interface est active.

**Notez le nom de votre interface** :

```text
Interface de la VM : ____________________
Adresse IPv4 : __________________________
Masque CIDR : ___________________________
```

Dans la suite du TP, remplacez `INTERFACE` par le nom réel de votre interface.

---

### 2.2 Lire la table de routage avec `ip r`

**Commande à taper** :

```bash
ip r
```

**Exemple de sortie** :

```text
default via 192.168.1.1 dev enp0s3
192.168.1.0/24 dev enp0s3 proto kernel scope link src 192.168.1.50
```

**À comprendre** :
- `default via` indique la passerelle par défaut ;
- sans route `default`, la VM ne sait pas sortir de son réseau local ;
- la route locale indique le réseau directement connecté.

**Notez la passerelle** :

```text
Passerelle par défaut : __________________
```

---

### 2.3 Voir les ports en écoute avec `ss -tulpn`

**Commande à taper** :

```bash
sudo ss -tulpn
```

**À repérer** :
- `tcp` ou `udp` ;
- `LISTEN` pour les services en écoute ;
- `Local Address:Port` ;
- le programme associé.

**Exemple** :

```text
tcp LISTEN 0 128 0.0.0.0:22 0.0.0.0:* users:(("sshd",pid=700,fd=3))
```

Cela signifie que le service SSH écoute sur le port `22`.

---

## 🧩 Partie 3 : Configurer le réseau avec Netplan

### 3.1 Lire la configuration actuelle

**Commandes à taper** :

```bash
ls -l /etc/netplan
sudo cat /etc/netplan/*.yaml
```

**Cas fréquent en DHCP** :

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
```

**Validation** : Vous savez trouver le fichier Netplan actif et le nom de l'interface configurée.

---

### 3.2 Préparer les informations réseau

Demandez au formateur les valeurs à utiliser pour votre VM.

```text
Interface : ______________________________
Adresse IP statique : ____________________
Masque CIDR : ____________________________
Passerelle : _____________________________
DNS 1 : __________________________________
DNS 2 : __________________________________
```

Exemple :

```text
Interface : enp0s3
Adresse IP statique : 192.168.1.50
Masque CIDR : 24
Passerelle : 192.168.1.1
DNS 1 : 1.1.1.1
DNS 2 : 8.8.8.8
```

---

### 3.3 Écrire une configuration Netplan en IP statique

**Sauvegarder le fichier existant** :

```bash
sudo cp /etc/netplan/*.yaml /home/PRENOM/TP03-Reseau/sauvegardes/
```

**Éditer le fichier Netplan** :

```bash
sudo nano /etc/netplan/01-netcfg.yaml
```

Si le fichier porte un autre nom, utilisez le nom existant dans `/etc/netplan`.

**Exemple à adapter** :

```yaml
network:
  version: 2
  ethernets:
    INTERFACE:
      dhcp4: false
      addresses:
        - 192.168.1.50/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
      mtu: 1500
```

Remplacez :
- `INTERFACE` par votre interface réelle ;
- `192.168.1.50/24` par votre adresse ;
- `192.168.1.1` par votre passerelle ;
- les DNS par ceux fournis par le formateur.

---

### 3.4 Vérifier et appliquer la configuration

**Vérifier la syntaxe** :

```bash
sudo netplan generate
```

**Appliquer prudemment** :

```bash
sudo netplan try
```

Si la connexion fonctionne, validez la configuration lorsque Netplan le demande.

**Vérifier** :

```bash
ip a
ip r
```

**Validation** :
- l'interface possède l'adresse IP attendue ;
- `ip r` affiche une route `default via` ;
- la passerelle correspond aux informations du formateur.

---

## 🧪 Partie 4 : Tester la connectivité

### 4.1 Tester la boucle locale

```bash
ping -c 4 127.0.0.1
```

**Validation** : Si ce test échoue, le problème est local à la machine.

---

### 4.2 Tester la passerelle

```bash
ping -c 4 PASSERELLE
```

Exemple :

```bash
ping -c 4 192.168.1.1
```

**Validation** : Si la passerelle répond, votre VM communique avec son réseau local.

---

### 4.3 Tester Internet par adresse IP

```bash
ping -c 4 1.1.1.1
```

**Validation** : Si ce test fonctionne, la route vers Internet est correcte.

---

### 4.4 Tester Internet par nom DNS

```bash
ping -c 4 example.com
```

**Validation** : Si `1.1.1.1` répond mais `example.com` échoue, la panne est probablement DNS.

---

### 4.5 Observer le chemin réseau avec `traceroute`

```bash
traceroute 1.1.1.1
traceroute example.com
```

**À comprendre** : Chaque ligne représente un saut réseau. Certains sauts peuvent afficher `* * *` si les réponses sont filtrées.

---

## 🔎 Partie 5 : Comprendre DNS et `/etc/hosts`

### 5.1 Lire la configuration DNS actuelle

```bash
cat /etc/resolv.conf
```

**À repérer** :

```text
nameserver X.X.X.X
```

Un `nameserver` est un serveur DNS utilisé par la machine.

---

### 5.2 Tester DNS avec `dig` et `host`

```bash
dig example.com
host example.com
```

**À observer avec `dig`** :
- `ANSWER SECTION` si une réponse existe ;
- l'adresse IP retournée ;
- le serveur DNS interrogé.

**Validation** : Vous obtenez au moins une adresse IP pour `example.com`.

---

### 5.3 Comprendre `/etc/hosts`

**Afficher le fichier** :

```bash
cat /etc/hosts
```

**Ajouter une entrée locale de test** :

```bash
echo "10.10.10.10 serveur-test.local" | sudo tee -a /etc/hosts
```

**Tester la résolution locale** :

```bash
getent hosts serveur-test.local
ping -c 1 serveur-test.local
```

**À comprendre** :
- `/etc/hosts` est local à la machine ;
- il peut résoudre un nom sans interroger DNS ;
- DNS est externe et permet de résoudre des noms à grande échelle.

**Nettoyage** :

```bash
sudo nano /etc/hosts
```

Supprimez la ligne ajoutée si le formateur le demande.

---

## 🧱 Partie 6 : Configurer un pare-feu simple avec `ufw`

### 6.1 Vérifier l'état du pare-feu

```bash
sudo ufw status verbose
sudo ufw status numbered
```

---

### 6.2 Définir une politique simple

**Important** : Si vous êtes connecté en SSH à la VM, autorisez SSH avant d'activer le pare-feu.

```bash
sudo ufw allow 22/tcp
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
```

**Validation** :

```bash
sudo ufw status verbose
```

Vous devez voir :
- pare-feu actif ;
- connexions entrantes refusées par défaut ;
- connexions sortantes autorisées par défaut ;
- port `22/tcp` autorisé.

---

### 6.3 Ajouter et supprimer une règle

**Autoriser HTTP** :

```bash
sudo ufw allow 80/tcp
sudo ufw status numbered
```

**Supprimer la règle HTTP** :

```bash
sudo ufw delete allow 80/tcp
sudo ufw status numbered
```

**Validation** : Vous savez lire, ajouter et supprimer une règle `ufw`.

---

## 🎫 Partie 7 : Trouble tickets à diagnostiquer

Chaque binôme reçoit une panne à injecter sur la VM de l'autre binôme.

Règles :
1. Le binôme qui corrige ne doit pas connaître la commande d'injection.
2. Le diagnostic doit être écrit avant la correction.
3. Toute correction doit être validée par des tests.
4. La démarche doit être documentée dans `/home/PRENOM/TP03-Reseau/rapport.md`.

Créer le rapport :

```bash
mkdir -p /home/PRENOM/TP03-Reseau
nano /home/PRENOM/TP03-Reseau/rapport.md
```

---

## 🧯 Ticket A : Route par défaut supprimée

### Commande d'injection pour le binôme formateur

**À exécuter sur la VM à dépanner** :

```bash
sudo ip route del default
```

Cette panne est temporaire. Elle ne modifie pas Netplan.

---

### Symptômes attendus

- La VM peut encore avoir une adresse IP.
- La VM peut joindre son réseau local.
- La VM ne peut plus joindre Internet.
- Les noms DNS peuvent échouer indirectement car aucun serveur externe n'est joignable.

---

### Diagnostic attendu

**Étape 1 : vérifier l'adresse IP**

```bash
ip a
```

**Étape 2 : vérifier les routes**

```bash
ip r
```

**Indice** : la ligne `default via ...` est absente.

**Étape 3 : tester local puis externe**

```bash
ping -c 4 PASSERELLE
ping -c 4 1.1.1.1
```

---

### Correction temporaire

```bash
sudo ip route add default via PASSERELLE dev INTERFACE
```

Exemple :

```bash
sudo ip route add default via 192.168.1.1 dev enp0s3
```

---

### Correction persistante avec Netplan

Vérifiez que le fichier Netplan contient bien une route par défaut :

```yaml
routes:
  - to: default
    via: 192.168.1.1
```

Puis appliquez :

```bash
sudo netplan try
```

---

### Validation

```bash
ip r
ping -c 4 1.1.1.1
ping -c 4 example.com
```

---

## 🧯 Ticket B : Mauvais DNS dans `/etc/resolv.conf`

### Commande d'injection pour le binôme formateur

**Sauvegarder puis casser la résolution DNS** :

```bash
sudo cp -a /etc/resolv.conf /home/PRENOM/TP03-Reseau/sauvegardes/resolv.conf.original
printf 'nameserver 203.0.113.254\n' | sudo tee /etc/resolv.conf
```

`203.0.113.254` est une adresse de documentation qui ne doit pas être utilisée comme DNS réel.

---

### Symptômes attendus

- `ping 1.1.1.1` fonctionne.
- `ping example.com` échoue.
- `dig example.com` ne retourne pas de réponse utilisable.

---

### Diagnostic attendu

**Étape 1 : distinguer IP et DNS**

```bash
ping -c 4 1.1.1.1
ping -c 4 example.com
```

**Étape 2 : tester DNS**

```bash
dig example.com
host example.com
```

**Étape 3 : lire les DNS configurés**

```bash
cat /etc/resolv.conf
```

**Cause probable** : le fichier contient un mauvais `nameserver`.

---

### Correction temporaire

```bash
printf 'nameserver 1.1.1.1\nnameserver 8.8.8.8\n' | sudo tee /etc/resolv.conf
```

---

### Correction persistante avec Netplan

Éditez Netplan :

```bash
sudo nano /etc/netplan/01-netcfg.yaml
```

Vérifiez la section :

```yaml
nameservers:
  addresses:
    - 1.1.1.1
    - 8.8.8.8
```

Appliquez :

```bash
sudo netplan try
```

---

### Validation

```bash
dig example.com
host example.com
ping -c 4 example.com
```

---

## 🧯 Ticket C : Règle `ufw` bloquante

### Commande d'injection pour le binôme formateur

Cette panne bloque les requêtes DNS sortantes. Elle est visible dans `ufw` et ne coupe pas SSH.

```bash
sudo ufw enable
sudo ufw deny out 53
```

---

### Symptômes attendus

- `ping 1.1.1.1` fonctionne.
- `ping example.com` échoue.
- `dig example.com` échoue ou expire.
- Le pare-feu contient une règle bloquante.

---

### Diagnostic attendu

**Étape 1 : vérifier IP puis DNS**

```bash
ping -c 4 1.1.1.1
ping -c 4 example.com
```

**Étape 2 : vérifier DNS explicitement**

```bash
dig example.com
```

**Étape 3 : vérifier le pare-feu**

```bash
sudo ufw status numbered
sudo ufw status verbose
```

**Cause probable** : une règle `DENY OUT` bloque le port `53`.

---

### Correction

Afficher les règles numérotées :

```bash
sudo ufw status numbered
```

Supprimer la règle bloquante par son numéro :

```bash
sudo ufw delete NUMERO
```

Remplacez `NUMERO` par le numéro de la règle `DENY OUT 53`.

---

### Validation

```bash
sudo ufw status numbered
dig example.com
ping -c 4 example.com
```

---

## 🧯 Ticket D : MTU cassé

### Commande d'injection pour le binôme formateur

```bash
sudo ip link set dev INTERFACE mtu 576
```

Exemple :

```bash
sudo ip link set dev enp0s3 mtu 576
```

Cette panne est temporaire. Elle ne modifie pas Netplan.

---

### Symptômes attendus

- Certains petits paquets passent.
- Des connexions semblent lentes ou instables.
- Les tests avec paquets plus gros échouent.

---

### Diagnostic attendu

**Étape 1 : lire l'état de l'interface**

```bash
ip a
ip link show INTERFACE
```

**Indice** : la valeur `mtu` est anormalement basse.

**Étape 2 : tester avec différentes tailles**

```bash
ping -c 4 1.1.1.1
ping -c 2 -s 1200 1.1.1.1
ping -c 2 -M do -s 1472 1.1.1.1
```

**À comprendre** : avec une MTU standard de `1500`, un ping de taille `1472` plus les en-têtes IP/ICMP correspond à `1500` octets.

---

### Correction temporaire

```bash
sudo ip link set dev INTERFACE mtu 1500
```

Exemple :

```bash
sudo ip link set dev enp0s3 mtu 1500
```

---

### Correction persistante avec Netplan

Vérifiez que Netplan contient :

```yaml
mtu: 1500
```

Exemple :

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.1.50/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
      mtu: 1500
```

Appliquez :

```bash
sudo netplan try
```

---

### Validation

```bash
ip link show INTERFACE
ping -c 4 1.1.1.1
ping -c 2 -s 1200 1.1.1.1
```

---

## 📝 Partie 8 : Documenter la démarche

Chaque ticket doit être documenté dans :

```text
/home/PRENOM/TP03-Reseau/rapport.md
```

### 8.1 Modèle à remplir

````markdown
# Rapport TP03 Linux - Réseau

## Informations

- Prénom : PRENOM
- Binôme :
- Interface réseau :
- Adresse IP :
- Passerelle :

## Ticket reçu

- Ticket : A / B / C / D
- Symptômes observés :

## Diagnostic

### Commandes exécutées

```bash
# Coller ici les commandes utilisées
```

### Observations

- Adresse IP :
- Route par défaut : présente / absente
- DNS : fonctionnel / non fonctionnel
- Pare-feu : règle bloquante / pas de règle bloquante
- MTU : normale / anormale

## Cause identifiée

Décrire la cause avec vos mots.

## Correction appliquée

```bash
# Coller ici les commandes de correction
```

## Validation

```bash
# Coller ici les commandes de validation
```

## Ce que j'ai appris

- 
````

---

## ✅ Partie 9 : Validation finale du TP

### 9.1 Commandes de validation globale

```bash
ip a
ip r
sudo ss -tulpn
cat /etc/resolv.conf
dig example.com
host example.com
ping -c 4 1.1.1.1
ping -c 4 example.com
sudo ufw status numbered
```

### 9.2 Résultats attendus

- l'interface possède une adresse IP correcte ;
- une route par défaut est présente ;
- DNS fonctionne ;
- `ping` vers une IP externe fonctionne ;
- `ping` vers un nom de domaine fonctionne ;
- `ufw` est actif et compréhensible ;
- aucune panne injectée ne reste active ;
- le rapport décrit diagnostic, cause, correction et validation.

---

## 📌 Récapitulatif des commandes importantes

### Observer le réseau

```bash
ip a
ip r
ip link show INTERFACE
sudo ss -tulpn
```

### Tester la connectivité

```bash
ping -c 4 PASSERELLE
ping -c 4 1.1.1.1
ping -c 4 example.com
traceroute example.com
```

### Tester DNS

```bash
cat /etc/resolv.conf
dig example.com
host example.com
getent hosts serveur-test.local
```

### Configurer Netplan

```bash
sudo nano /etc/netplan/01-netcfg.yaml
sudo netplan generate
sudo netplan try
ip a
ip r
```

### Gérer `ufw`

```bash
sudo ufw status verbose
sudo ufw status numbered
sudo ufw allow 22/tcp
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
sudo ufw delete NUMERO
```

### Corriger les pannes fréquentes

```bash
sudo ip route add default via PASSERELLE dev INTERFACE
printf 'nameserver 1.1.1.1\nnameserver 8.8.8.8\n' | sudo tee /etc/resolv.conf
sudo ufw delete NUMERO
sudo ip link set dev INTERFACE mtu 1500
```
