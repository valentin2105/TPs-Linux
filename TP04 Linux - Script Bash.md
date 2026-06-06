# TP04 Linux - Script Bash

**Niveau** : Débutant Linux / Bash  
**Prérequis** : Terminal Linux, éditeur de texte (`nano`), droits sur son dossier personnel  
**Convention du TP** : le dossier personnel (`HOME`) de l'apprenant est `/home/PRENOM`. Remplacez `PRENOM` par votre prénom dans les chemins absolus.

---

## 📚 Objectifs pédagogiques

À la fin de ce TP, vous saurez :
- ✅ créer un script Bash ;
- ✅ le rendre exécutable avec `chmod +x` ;
- ✅ utiliser des variables ;
- ✅ lire une saisie utilisateur avec `read` ;
- ✅ utiliser des arguments (`$1`, `$2`, `$#`, `$@`) ;
- ✅ écrire des conditions `if`, `then`, `else`, `elif` ;
- ✅ utiliser des boucles `for` et `while` ;
- ✅ utiliser `case` pour faire un menu simple ;
- ✅ écrire une fonction Bash ;
- ✅ créer un mini-jeu `Guess the number`.

---

## 🧰 Partie 1 : Préparer le dossier du TP

### 1.1 Créer le dossier de travail

```bash
mkdir -p /home/PRENOM/TP04-Bash
cd /home/PRENOM/TP04-Bash
pwd
```

**Validation** : `pwd` doit afficher :

```text
/home/PRENOM/TP04-Bash
```

---

## 📝 Partie 2 : Premier script Bash

### 2.1 Créer un fichier script

```bash
nano bonjour.sh
```

Contenu :

```bash
#!/bin/bash

# Mon premier script Bash
echo "Bonjour depuis Bash !"
```

**Explication** :
- `#!/bin/bash` s'appelle le *shebang* ;
- il indique que le script doit être exécuté avec Bash ;
- les lignes commençant par `#` sont des commentaires ;
- `echo` affiche du texte.

---

### 2.2 Rendre le script exécutable

```bash
chmod +x bonjour.sh
```

**Exécuter le script** :

```bash
./bonjour.sh
```

**Résultat attendu** :

```text
Bonjour depuis Bash !
```

---

## 📦 Partie 3 : Variables

### 3.1 Déclarer et afficher une variable

Créer `variables.sh` :

```bash
nano variables.sh
```

Contenu :

```bash
#!/bin/bash

PRENOM="Alice"
AGE=25

echo "Bonjour $PRENOM"
echo "Tu as $AGE ans"
echo "Ton dossier personnel est $HOME"
```

Rendez le script exécutable :

```bash
chmod +x variables.sh
./variables.sh
```

**Point important** : il ne faut pas mettre d'espaces autour du signe `=`.

Correct :

```bash
NOM="Alice"
```

Incorrect :

```bash
NOM = "Alice"
```

---

### 3.2 Mini-exercice

Créez un script `presentation.sh` qui affiche :

```text
Je m'appelle PRENOM
J'apprends Bash dans /home/PRENOM/TP04-Bash
```

---

## 🎛️ Partie 4 : Arguments d'un script

### 4.1 Lire `$1`, `$2`, `$#`, `$@`

Créer `arguments.sh` :

```bash
nano arguments.sh
```

Contenu :

```bash
#!/bin/bash

echo "Nom du script : $0"
echo "Premier argument : $1"
echo "Deuxième argument : $2"
echo "Nombre d'arguments : $#"
echo "Tous les arguments : $@"
```

Tester :

```bash
chmod +x arguments.sh
./arguments.sh Alice Linux Bash
```

**Validation** : Le script affiche les arguments passés en ligne de commande.

---

### 4.2 Mini-exercice

Créez `salut.sh` qui prend un prénom en argument :

```bash
./salut.sh Alice
```

Résultat attendu :

```text
Bonjour Alice
```

---

## ⌨️ Partie 5 : Saisie utilisateur avec `read`

### 5.1 Demander une valeur

Créer `read.sh` :

```bash
nano read.sh
```

Contenu :

```bash
#!/bin/bash

echo "Quel est votre prénom ?"
read PRENOM

echo "Bonjour $PRENOM !"
```

Tester :

```bash
chmod +x read.sh
./read.sh
```

---

### 5.2 Variante avec prompt sur une seule ligne

```bash
#!/bin/bash

read -p "Votre prénom : " PRENOM
echo "Bienvenue $PRENOM"
```

---

## 🔀 Partie 6 : Conditions `if`, `then`, `else`

### 6.1 Tester un âge

Créer `age.sh` :

```bash
nano age.sh
```

Contenu :

```bash
#!/bin/bash

read -p "Votre âge : " AGE

if [ "$AGE" -ge 18 ]; then
    echo "Vous êtes majeur"
else
    echo "Vous êtes mineur"
fi
```

Tester :

```bash
chmod +x age.sh
./age.sh
```

---

### 6.2 Opérateurs utiles

Comparaisons numériques :

```bash
[ "$A" -eq "$B" ]   # égal
[ "$A" -ne "$B" ]   # différent
[ "$A" -lt "$B" ]   # inférieur
[ "$A" -le "$B" ]   # inférieur ou égal
[ "$A" -gt "$B" ]   # supérieur
[ "$A" -ge "$B" ]   # supérieur ou égal
```

Comparaisons de texte :

```bash
[ "$TEXTE" = "oui" ]
[ "$TEXTE" != "non" ]
[ -z "$TEXTE" ]      # vide
[ -n "$TEXTE" ]      # non vide
```

Tests de fichiers :

```bash
[ -f fichier.txt ]    # existe et est un fichier
[ -d dossier ]        # existe et est un dossier
[ -x script.sh ]      # est exécutable
```

---

### 6.3 Mini-exercice

Créez `positif.sh` qui demande un nombre et affiche :
- `positif` si le nombre est supérieur à 0 ;
- `négatif` si le nombre est inférieur à 0 ;
- `zéro` sinon.

---

## 🔁 Partie 7 : Boucle `for`

### 7.1 Parcourir une liste

Créer `for.sh` :

```bash
nano for.sh
```

Contenu :

```bash
#!/bin/bash

for FRUIT in pomme banane orange; do
    echo "Fruit : $FRUIT"
done
```

Tester :

```bash
chmod +x for.sh
./for.sh
```

---

### 7.2 Compter de 1 à 5

```bash
#!/bin/bash

for NOMBRE in 1 2 3 4 5; do
    echo "Nombre : $NOMBRE"
done
```

Autre syntaxe :

```bash
#!/bin/bash

for NOMBRE in {1..5}; do
    echo "Nombre : $NOMBRE"
done
```

---

### 7.3 Mini-exercice

Créez `carres.sh` qui affiche :

```text
1 x 1 = 1
2 x 2 = 4
3 x 3 = 9
4 x 4 = 16
5 x 5 = 25
```

Aide :

```bash
RESULTAT=$((NOMBRE * NOMBRE))
```

---

## 🔄 Partie 8 : Boucle `while`

### 8.1 Répéter tant qu'une condition est vraie

Créer `while.sh` :

```bash
nano while.sh
```

Contenu :

```bash
#!/bin/bash

COMPTEUR=1

while [ "$COMPTEUR" -le 5 ]; do
    echo "Compteur : $COMPTEUR"
    COMPTEUR=$((COMPTEUR + 1))
done
```

Tester :

```bash
chmod +x while.sh
./while.sh
```

---

### 8.2 Mini-exercice

Créez `somme.sh` qui :
1. demande des nombres à l'utilisateur ;
2. additionne les nombres ;
3. s'arrête quand l'utilisateur tape `0` ;
4. affiche la somme finale.

---

## 🧭 Partie 9 : Menu avec `case`

### 9.1 Créer un menu simple

Créer `menu.sh` :

```bash
nano menu.sh
```

Contenu :

```bash
#!/bin/bash

echo "1. Afficher la date"
echo "2. Afficher le dossier courant"
echo "3. Quitter"
read -p "Choix : " CHOIX

case "$CHOIX" in
    1)
        date
        ;;
    2)
        pwd
        ;;
    3)
        echo "Au revoir"
        ;;
    *)
        echo "Choix invalide"
        ;;
esac
```

Tester :

```bash
chmod +x menu.sh
./menu.sh
```

---

## 🧩 Partie 10 : Fonctions

### 10.1 Déclarer une fonction

Créer `fonctions.sh` :

```bash
nano fonctions.sh
```

Contenu :

```bash
#!/bin/bash

saluer() {
    echo "Bonjour $1"
}

additionner() {
    local A=$1
    local B=$2
    echo $((A + B))
}

saluer "Alice"
RESULTAT=$(additionner 4 7)
echo "Résultat : $RESULTAT"
```

Tester :

```bash
chmod +x fonctions.sh
./fonctions.sh
```

---

### 10.2 Codes de retour

En Bash :
- `0` signifie succès ;
- une autre valeur signifie erreur.

Exemple :

```bash
#!/bin/bash

verifier_fichier() {
    if [ -f "$1" ]; then
        return 0
    else
        return 1
    fi
}

if verifier_fichier "bonjour.sh"; then
    echo "Le fichier existe"
else
    echo "Le fichier est absent"
fi
```

---

## 🎮 Partie 11 : Exercice final — Guess the number

### 11.1 Objectif

Créer un jeu Bash nommé `guess_number.sh`.

Le script doit :
1. choisir un nombre aléatoire entre 1 et 100 ;
2. demander une proposition à l'utilisateur ;
3. répondre `trop petit`, `trop grand` ou `bravo` ;
4. compter le nombre de tentatives ;
5. boucler jusqu'à ce que le joueur trouve.

---

### 11.2 Version guidée

Créer le fichier :

```bash
nano guess_number.sh
```

Contenu :

```bash
#!/bin/bash

NOMBRE_SECRET=$((RANDOM % 100 + 1))
TENTATIVES=0

echo "=== Guess the number ==="
echo "Je pense à un nombre entre 1 et 100."
echo "À vous de deviner !"
echo ""

while true; do
    read -p "Votre proposition : " PROPOSITION

    if ! [[ "$PROPOSITION" =~ ^[0-9]+$ ]]; then
        echo "Veuillez entrer un nombre valide."
        continue
    fi

    TENTATIVES=$((TENTATIVES + 1))

    if [ "$PROPOSITION" -eq "$NOMBRE_SECRET" ]; then
        echo "Bravo ! Vous avez trouvé $NOMBRE_SECRET en $TENTATIVES tentative(s)."
        break
    elif [ "$PROPOSITION" -lt "$NOMBRE_SECRET" ]; then
        echo "Trop petit."
    else
        echo "Trop grand."
    fi
done
```

Rendre exécutable et tester :

```bash
chmod +x guess_number.sh
./guess_number.sh
```

---

### 11.3 Améliorations possibles

Ajoutez une ou plusieurs options :
- limiter le nombre de tentatives ;
- proposer de rejouer ;
- ajouter un niveau facile, moyen, difficile ;
- enregistrer le score dans `/home/PRENOM/TP04-Bash/scores.txt` ;
- afficher un message spécial si le joueur trouve en moins de 5 tentatives.

---

## ✅ Validation finale

Vous devez avoir créé et testé au moins ces scripts :

```text
bonjour.sh
variables.sh
arguments.sh
read.sh
age.sh
for.sh
while.sh
menu.sh
fonctions.sh
guess_number.sh
```

Commandes utiles pour vérifier :

```bash
ls -l /home/PRENOM/TP04-Bash
bash -n /home/PRENOM/TP04-Bash/guess_number.sh
/home/PRENOM/TP04-Bash/guess_number.sh
```

**Validation** : si `guess_number.sh` fonctionne et que vous comprenez chaque partie du script, le TP est réussi.
