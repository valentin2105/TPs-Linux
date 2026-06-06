# TP Linux Débutant : Maîtriser le Shell et l'Arborescence

**Niveau** : Débutant  
**Prérequis** : Accès à un terminal Linux/macOS  
**Convention du TP** : le dossier personnel (`HOME`) de l'apprenant est `/home/PRENOM`. Remplacez `PRENOM` par votre prénom dans les chemins absolus.

---

## 📚 Objectifs pédagogiques

À la fin de ce TP, vous saurez :
- ✅ Naviguer efficacement dans l'arborescence de fichiers
- ✅ Utiliser les commandes essentielles : `pwd`, `ls`, `cd`
- ✅ Créer, copier, déplacer et supprimer fichiers et dossiers
- ✅ Afficher et éditer des fichiers texte simples
- ✅ Comprendre et utiliser les chemins absolus et relatifs
- ✅ Modifier les permissions avec `chmod`
- ✅ Mettre en place une organisation cohérente

---

## 🎯 Partie 1 : Les Fondations

### 1.1 Découvrir votre position : `pwd`

**Concept** : Le terminal est toujours dans un répertoire courant. `pwd` affiche son chemin complet.

**Commande à taper** :
```bash
pwd
```

**Résultat attendu** :
```
/home/PRENOM
```

**Ce que cela signifie** : Vous êtes actuellement dans le répertoire personnel `/home/PRENOM`.

---

### 1.2 Voir le contenu d'un dossier : `ls`

**Concept** : `ls` liste les fichiers et dossiers du répertoire courant.

**Commandes à taper** :
```bash
ls                    # Liste simple
ls -l                 # Liste détaillée avec permissions
ls -la                # Inclut les fichiers cachés (commençant par .)
ls -lh                # Taille en format lisible (Ko, Mo, etc.)
```

**Exemple de résultat** :
```
drwxr-xr-x   3 user  group   96B 6 juin 10:30 Documents
-rw-r--r--   1 user  group  2.1K 6 juin 10:15 notes.txt
-rw-r--r--   1 user  group   512B 6 juin 10:00 projet.md
```

**Explication** :
- `d` = répertoire (directory)
- `-` = fichier simple
- `rwxr-xr-x` = permissions (propriétaire / groupe / autres)
- `user` / `group` = propriétaire et groupe
- `96B` ou `2.1K` = taille
- Date et nom = dernière modification et nom du fichier

---

### 1.3 Changer de répertoire : `cd`

**Concept** : `cd` permet de naviguer entre dossiers.

**Commandes à taper** :
```bash
cd Documents          # Accéder au dossier Documents (chemin relatif)
pwd                   # Vérifier sa position
cd /tmp               # Aller au dossier /tmp (chemin absolu)
pwd
cd ..                 # Remonter d'un niveau
pwd
cd ~                  # Retour au dossier personnel
cd /                  # À la racine du système
ls
cd ~                  # Revenir chez soi
```

**Points clés** :
- `.` = répertoire courant
- `..` = répertoire parent
- `/` = racine du système (chemin absolu commence toujours par `/`)
- `~` = dossier personnel de l'utilisateur, ici `/home/PRENOM`

**Validation** : À chaque déplacement, tapez `pwd` pour vérifier votre position.

---

## 🛠️ Partie 2 : Manipuler les Fichiers

### 2.1 Créer et organiser

**Étape 1 : Créer un dossier de travail**

```bash
cd ~
mkdir TP-Linux                  # Créer le dossier
cd TP-Linux                     # Y accéder
pwd                             # Vérifier
```

**Validation** : `pwd` doit afficher `/home/PRENOM/TP-Linux`

---

**Étape 2 : Créer des fichiers texte simples**

```bash
echo "Bonjour Linux!" > bienvenue.txt
echo "Ceci est mon premier fichier." > test.txt
echo "Exercice de shell" > exercice.txt
```

**Vérification** :
```bash
ls -l
```

**Attente** : 3 fichiers `.txt` doivent s'afficher.

---

### 2.2 Afficher le contenu d'un fichier

**Concept** : Différentes façons de lire un fichier.

```bash
cat bienvenue.txt               # Afficher complètement
cat test.txt exercice.txt       # Afficher plusieurs fichiers
head -2 test.txt                # Les 2 premières lignes
wc -l test.txt                  # Compter les lignes
```

**Résultat attendu** :
```
Bonjour Linux!
Ceci est mon premier fichier.
Exercice de shell
```

---

### 2.3 Éditer un fichier simple

**Option A : Avec `nano` (plus facile)** :
```bash
nano bienvenue.txt
```
- Tapez du texte ou modifiez l'existant
- Appuyez sur `Ctrl+O` puis `Entrée` pour sauvegarder
- Appuyez sur `Ctrl+X` pour quitter

**Option B : Avec `vi` ou `vim` (plus avancé)** :
```bash
vi bienvenue.txt
```
- Appuyez sur `i` pour entrer en mode insertion
- Tapez du texte
- Appuyez sur `Échap` puis `:wq` pour sauvegarder et quitter

**Exercice** : Ouvrez `bienvenue.txt`, ajoutez une ligne `"Aujourd'hui, j'apprends le shell!"` et sauvegardez.

**Validation** :
```bash
cat bienvenue.txt
```
Doit afficher les deux lignes.

---

### 2.4 Copier des fichiers

**Commande** : `cp source destination`

```bash
cp test.txt test_copie.txt      # Créer une copie
cp test.txt backup.txt          # Autre copie
ls -l                           # Vérifier
```

**Attente** : `test_copie.txt` et `backup.txt` existent maintenant.

**Copier un dossier entier** :
```bash
mkdir dossier1
echo "Contenu" > dossier1/fichier.txt
cp -r dossier1 dossier1_backup   # -r = récursif
ls -l
```

**Validation** : `dossier1_backup` et son contenu existent.

---

### 2.5 Déplacer et renommer des fichiers

**Commande** : `mv source destination`

```bash
# Renommer un fichier
mv test_copie.txt ancien_test.txt
ls -l

# Déplacer un fichier vers un dossier
mkdir archives
mv backup.txt archives/
ls -l
ls archives/

# Déplacer ET renommer
mv ancien_test.txt archives/test_ancien.txt
```

**Validation** :
```bash
ls -l              # test_copie.txt a disparu
ls archives/       # Les fichiers s'y trouvent
```

---

### 2.6 Supprimer des fichiers

**Commande** : `rm fichier` (⚠️ IRRÉVERSIBLE)

```bash
echo "À supprimer" > temp.txt
rm temp.txt
ls                 # temp.txt n'existe plus

# Supprimer plusieurs fichiers
echo "Fichier 1" > delete1.txt
echo "Fichier 2" > delete2.txt
rm delete1.txt delete2.txt

# Supprimer un dossier vide
mkdir vide
rmdir vide

# Supprimer un dossier et son contenu
mkdir -p dossier_temp/sous
echo "Contenu" > dossier_temp/sous/fichier.txt
rm -r dossier_temp      # -r = récursif
```

**⚠️ Attention** : `rm -r` supprime TOUT sans confirmation. À utiliser avec prudence.

---

## 📍 Partie 3 : Chemins Absolus et Relatifs

### 3.1 Concept

**Chemin absolu** : Commence par `/`, valide depuis n'importe où
```bash
/home/PRENOM/TP-Linux/bienvenue.txt
```

**Chemin relatif** : Relatif au répertoire courant, utilise `.` et `..`
```bash
./bienvenue.txt              # Dans le répertoire courant
../documents/mon_fichier.txt # Un niveau au-dessus
```

---

### 3.2 Exercice pratique

```bash
# Vous êtes dans ~/TP-Linux
pwd

# Créer une arborescence
mkdir -p projet/src
mkdir -p projet/docs

# Naviguer avec des chemins absolus
ls /home/PRENOM/TP-Linux

# Naviguer avec des chemins relatifs
cd projet/src
pwd
cat ../../bienvenue.txt                  # Remonte 2 niveaux

# Revenir
cd ..
pwd
```

**Résultat attendu** : Vous pouvez accéder aux fichiers des deux façons.

---

## 🔐 Partie 4 : Les Permissions

### 4.1 Comprendre les permissions

```bash
ls -l bienvenue.txt
```

**Résultat** :
```
-rw-r--r-- 1 user group 50 6 juin 10:30 bienvenue.txt
```

**Explication** : `rw-r--r--` = 9 caractères divisés en 3 groupes de 3

| Groupe | Signification |
|--------|---------------|
| `rw-` | Propriétaire : lecture (r) + écriture (w), pas exécution (x) |
| `r--` | Groupe : lecture seule |
| `r--` | Autres : lecture seule |

**Légende** :
- `r` = read (lecture) = 4
- `w` = write (écriture) = 2
- `x` = execute (exécution) = 1

---

### 4.2 Modifier les permissions avec `chmod`

**Syntaxe numérique** : `chmod xyz fichier`
- `x` = permissions propriétaire (r=4, w=2, x=1)
- `y` = permissions groupe
- `z` = permissions autres

**Exemples** :
```bash
# Rendre un fichier exécutable pour le propriétaire
chmod 755 bienvenue.txt
# 755 = rwxr-xr-x (lecture/exécution pour tous, écriture propriétaire)

# Restreindre : propriétaire seul peut lire/écrire
chmod 600 test.txt
# 600 = rw------- 

# Rendre un dossier accessible (x = traversable)
chmod 755 projet
```

**Vérification** :
```bash
ls -l
```

---

**Syntaxe symbolique** (plus intuitive) :
```bash
chmod +x script.sh              # Ajouter exécution
chmod -w fichier.txt            # Retirer écriture
chmod u+rwx fichier.txt         # Propriétaire : r+w+x
chmod go-x dossier              # Groupe et autres : retirer x
```

**Exercice** :
```bash
cd ~/TP-Linux
chmod 644 bienvenue.txt         # rw-r--r--
ls -l bienvenue.txt
chmod 755 projet                # rwxr-xr-x
ls -ld projet                   # -d pour afficher le dossier lui-même
```

**Validation** : Les permissions s'affichent correctement dans `ls -l`.

---

## 🎓 Partie 5 : Exercices Progressifs

### Exercice 1 : Organisation de base ⭐

**Objectif** : Créer une arborescence cohérente

**Consignes** :
1. Créez un dossier `Mon_Projet` dans votre répertoire personnel
2. Créez 3 sous-dossiers : `sources`, `donnees`, `resultats`
3. Créez 2 fichiers dans `sources` : `main.txt` et `config.txt`
4. Afficher l'arborescence avec `tree` ou `ls -R`

**Commandes attendues** :
```bash
mkdir -p ~/Mon_Projet/{sources,donnees,resultats}
echo "Code principal" > ~/Mon_Projet/sources/main.txt
echo "Configuration" > ~/Mon_Projet/sources/config.txt
ls -R ~/Mon_Projet
```

**Critère de validation** : L'arborescence s'affiche correctement avec tous les fichiers.

---

### Exercice 2 : Manipulation de fichiers ⭐⭐

**Objectif** : Pratiquer cp, mv, rm

**Consignes** :
1. Allez dans `~/Mon_Projet/sources`
2. Copiez `main.txt` en `main_backup.txt`
3. Copiez `config.txt` dans le dossier `donnees`
4. Renommez `main.txt` en `principal.txt`
5. Déplacez `principal.txt` dans le dossier `resultats`
6. Supprimez `main_backup.txt`
7. Listez le contenu de chaque dossier

**Commandes attendues** :
```bash
cd ~/Mon_Projet/sources
cp main.txt main_backup.txt
cp config.txt ../donnees/
mv main.txt principal.txt
mv principal.txt ../resultats/
rm main_backup.txt
echo "=== sources ===" && ls sources
echo "=== donnees ===" && ls donnees
echo "=== resultats ===" && ls resultats
```

**Critère de validation** :
- `sources/` : contient `config.txt` uniquement
- `donnees/` : contient `config.txt`
- `resultats/` : contient `principal.txt`

---

### Exercice 3 : Chemins relatifs et absolus ⭐⭐

**Objectif** : Maîtriser les chemins

**Consignes** :
1. Positionnez-vous dans `~/Mon_Projet/donnees`
2. Afficher le contenu de `resultats/principal.txt` avec un chemin relatif
3. Afficher le contenu de `sources/config.txt` avec un chemin absolu
4. Copiez `../resultats/principal.txt` dans le dossier courant
5. Vérifiez avec `ls -l`

**Commandes attendues** :
```bash
cd ~/Mon_Projet/donnees
pwd
cat ../resultats/principal.txt          # Chemin relatif
cat $HOME/Mon_Projet/sources/config.txt # Chemin absolu
cp ../resultats/principal.txt ./
ls -l
```

**Critère de validation** : Les fichiers s'affichent correctement, `principal.txt` est dans `donnees/`.

---

### Exercice 4 : Édition et consultation ⭐⭐

**Objectif** : Éditer et consulter des fichiers

**Consignes** :
1. Allez dans `~/Mon_Projet/resultats`
2. Ouvrez `principal.txt` avec `nano`
3. Ajoutez la ligne `"Version: 1.0 - Finalisée"`
4. Sauvegardez et quittez
5. Affichage le contenu avec `cat` puis `head -1`

**Commandes attendues** :
```bash
cd ~/Mon_Projet/resultats
nano principal.txt
# (Ajouter la ligne, Ctrl+O, Entrée, Ctrl+X)
cat principal.txt
head -1 principal.txt
```

**Critère de validation** : La nouvelle ligne s'affiche, le fichier contient au moins 2 lignes.

---

### Exercice 5 : Permissions et sécurité ⭐⭐⭐

**Objectif** : Gérer les permissions

**Consignes** :
1. Allez dans `~/Mon_Projet`
2. Rendez `sources/config.txt` lisible par le propriétaire seul (600)
3. Rendez le dossier `donnees` exécutable (755)
4. Vérifiez les permissions avec `ls -l` et `ls -ld`
5. Essayez de modifier `sources/config.txt` (vous devez pouvoir)
6. Changez ses permissions en 444 (lecture seule) et essayez de modifier

**Commandes attendues** :
```bash
cd ~/Mon_Projet
chmod 600 sources/config.txt
chmod 755 donnees
ls -l sources/config.txt
ls -ld donnees
# Essayer d'éditer
nano sources/config.txt    # Ctrl+C pour quitter
chmod 444 sources/config.txt
nano sources/config.txt    # Tentative (error)
chmod 644 sources/config.txt  # Retrouver les droits
```

**Critère de validation** : Les permissions s'affichent correctement (600, puis 755, puis 444).

---

## 📋 Checklist de Validation Finale

Avant de conclure ce TP, vérifiez que vous pouvez :

- [ ] Utiliser `pwd` pour connaître votre position
- [ ] Utiliser `ls`, `ls -l`, `ls -la` pour explorer un dossier
- [ ] Naviguer avec `cd`, `cd ..`, `cd ~`, `cd /`
- [ ] Créer des fichiers avec `echo`, `touch`, `nano`
- [ ] Créer des dossiers avec `mkdir` (y compris `-p`)
- [ ] Copier des fichiers et dossiers avec `cp` et `cp -r`
- [ ] Déplacer et renommer avec `mv`
- [ ] Supprimer des fichiers avec `rm` et des dossiers avec `rmdir` ou `rm -r`
- [ ] Afficher le contenu avec `cat`, `head`, `wc`
- [ ] Éditer avec `nano` ou `vi`
- [ ] Utiliser les chemins absolus (commençant par `/`)
- [ ] Utiliser les chemins relatifs (`.`, `..`, `./`)
- [ ] Comprendre les permissions `rwx` et leur notation numérique
- [ ] Modifier les permissions avec `chmod`

---

## 🚀 Pour Aller Plus Loin (Bonus)

### Commandes supplémentaires à explorer

```bash
# Rechercher des fichiers
find ~/Mon_Projet -name "*.txt"

# Compter le nombre de lignes
wc -l ~/Mon_Projet/sources/config.txt

# Chercher du texte dans un fichier
grep "Version" ~/Mon_Projet/resultats/principal.txt

# Afficher le début/fin d'un fichier
head -5 mon_fichier.txt
tail -3 mon_fichier.txt

# Voir l'espace disque utilisé
du -sh ~/Mon_Projet

# Voir les dossiers en arborescence
tree ~/Mon_Projet       # (si disponible)
ls -R ~/Mon_Projet      # Alternative
```

### Concepts avancés pour plus tard

- Redirection : `>`, `>>`, `<`
- Pipes : `|`
- Variables d'environnement
- Scripts shell (`.sh`)
- Gestion des processus (`ps`, `kill`)
- Permissions avancées (`umask`, `chown`)

---

## 📞 Aide-Mémoire Rapide

| Commande | Usage | Exemple |
|----------|-------|---------|
| `pwd` | Position actuelle | `pwd` |
| `ls` | Lister fichiers | `ls -la` |
| `cd` | Changer dossier | `cd Documents` |
| `mkdir` | Créer dossier | `mkdir -p a/b/c` |
| `touch` | Créer fichier vide | `touch fichier.txt` |
| `echo` | Afficher/créer texte | `echo "Hello" > f.txt` |
| `cat` | Afficher fichier | `cat fichier.txt` |
| `nano` | Éditer fichier | `nano fichier.txt` |
| `cp` | Copier | `cp fichier.txt copie.txt` |
| `mv` | Déplacer/renommer | `mv old.txt new.txt` |
| `rm` | Supprimer fichier | `rm fichier.txt` |
| `rmdir` | Supprimer dossier vide | `rmdir dossier` |
| `rm -r` | Supprimer dossier | `rm -r dossier` |
| `chmod` | Permissions | `chmod 755 fichier` |
| `ls -l` | Voir permissions | `ls -l` |

---

## ✅ Conclusion

Vous maîtrisez maintenant les bases essentielles du shell Linux ! Ces compétences sont fondamentales pour :
- Gérer des serveurs
- Développer des scripts
- Automatiser des tâches
- Travailler efficacement dans un environnement Linux/Unix

**Prochaines étapes** : Explorez les redirection, pipes, variables d'environnement et scripts shell.

Bon apprentissage ! 🎉
