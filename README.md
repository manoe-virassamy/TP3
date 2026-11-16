# Compte rendu TP 3 : Nginx dans un chroot (fat puis thin)

Le but de ces deux TP est de faire tourner le Nginx qu'on a compilé nous-mêmes (version 1.30.6, installé dans `/opt/nginx-1.30.6`) dans un chroot. Un chroot, c'est un dossier que le processus voit comme sa racine `/` : il ne peut plus accéder au reste du système. Si Nginx se fait attaquer, l'attaquant reste coincé dedans. La séance 3 porte sur un chroot « fat », avec un système Debian complet. La séance 4 porte sur un chroot « thin », avec seulement ce dont Nginx a besoin.

## 1. Fat Nginx chroot

### 1.1 Création du dossier

```
mkdir -p /chroot/fat
```

L'option `-p` crée aussi les dossiers parents, ici `/chroot`.

### 1.2 Installation d'un système Debian dans le chroot avec debootstrap

```
debootstrap stable /chroot/fat/
```

debootstrap télécharge les paquets de base de Debian depuis un miroir et les installe dans le dossier. On obtient un système complet (`/bin`, `/lib`, `/etc`...) dans lequel on pourra entrer avec `chroot`. C'est ce qui le rend « fat » : il est gros alors que Nginx n'a besoin que de très peu de choses.

### 1.3 Copie de Nginx

```
cp -r /opt/nginx-1.30.6 /chroot/fat/opt/
```

Le chroot contient maintenant Nginx. Comme il a été compilé avec `--prefix=/opt/nginx-1.30.6`, il retrouve ses fichiers au même chemin une fois dans le chroot, puisque `/chroot/fat` devient `/`.

### 1.4 Modification de la page d'accueil

La page à modifier est donc celle du chroot, pas celle de l'hôte. On l'a éditée avec `vim.tiny` :

```
vim.tiny /chroot/fat/opt/nginx-1.30.6/html/index.html
```

On a ajouté un texte pour la reconnaître dans le navigateur. La consigne le fait en une ligne :

```
echo "In chroot" > /chroot/fat/opt/nginx-1.30.6/html/index.html
```

### 1.5 Entrée dans le chroot

```
chroot /chroot/fat /bin/bash
```

On obtient un bash qui voit `/chroot/fat` comme racine. Un `ls /` affiche le contenu du Debian installé par debootstrap, et non plus celui de l'hôte.

### 1.6 Vérification de la page

Depuis ce bash :

```
cat /opt/nginx-1.30.6/html/index.html
```

On voit bien « In chroot ». Le chemin est le même que sur l'hôte, mais sans le préfixe `/chroot/fat`.

### 1.7 Recréation du lien symbolique

En sortant du chroot, on a essayé de lancer Nginx directement :

```
chroot /chroot/fat /opt/nginx/sbin/nginx
```

Ça ne démarre pas : on a copié `/opt/nginx-1.30.6` mais pas le lien `/opt/nginx` qui pointe dessus, donc le chemin n'existe pas dans le chroot. On le recrée :

```
cd /chroot/fat/opt
ln -s nginx-1.30.6 nginx
```

Le lien est relatif, il marche aussi bien de l'intérieur que de l'extérieur du chroot.

### 1.8 Recréation de l'utilisateur nginx

Le lancement échoue encore, cette fois avec une erreur `[emerg]`. Nginx doit faire tourner ses workers sous l'utilisateur `nginx` et il ne le trouve pas. On compare les deux fichiers passwd :

```
grep nginx /etc/passwd
grep nginx /chroot/fat/etc/passwd
```

Le premier affiche la ligne de nginx (UID, GID, home, shell), le deuxième ne renvoie rien. Dans le chroot, c'est le `/etc/passwd` du chroot qui est lu, pas celui de l'hôte. Les UID sont les mêmes partout puisque le noyau est le même, mais c'est ce fichier qui associe un nom à un UID. Ici ce n'est donc pas une dépendance qui manque, c'est un problème de résolution du nom nginx.

On copie la ligne de l'hôte dans le passwd du chroot :

```
grep nginx /etc/passwd >> /chroot/fat/etc/passwd
```

Il faut bien mettre `>>` pour ajouter à la fin. Avec un seul `>`, on écraserait tout le fichier, root compris.

On a aussi regardé `/etc/shadow` (là où sont stockés les mots de passe) :

```
grep nginx /etc/shadow
```

Ça n'a pas suffi : en relançant Nginx, ça échouait toujours. Il lui manquait le groupe, qui est dans `/etc/group`. On copie aussi cette ligne :

```
grep nginx /etc/group
nginx:x:988:

grep nginx /etc/group >> /chroot/fat/etc/group
```

### 1.9 Lancement de Nginx dans le chroot

On sort du chroot avec `exit` et on lance :

```
chroot /chroot/fat /opt/nginx/sbin/nginx
```

Cette fois ça fonctionne. Dans le navigateur, `http://localhost` affiche « In chroot », donc c'est bien le Nginx du chroot qui répond. Pour voir sur quelle socket il écoute :

```
ss -tlnp | grep nginx
```

Il écoute sur le port 80. Le chroot isole les fichiers mais pas le réseau, donc c'est le même port que sur l'hôte.

### 1.10 UID utilisé par nginx

```
ps -eo user,uid,gid,pid,cmd | grep nginx
```

Le master tourne en root et les workers sous l'utilisateur nginx. Le problème possible, c'est que si on crée l'utilisateur dans le chroot avec `useradd`, l'UID est choisi sans regarder l'hôte. Si ce numéro est déjà celui d'un autre compte de l'hôte, Nginx aurait les droits de ce compte. Dans notre cas, on a copié les lignes de l'hôte, donc l'UID et le GID (988) sont les mêmes des deux côtés et il n'y a pas de collision.

### 1.11 Nouvel utilisateur avec un autre UID

La consigne propose de supprimer l'utilisateur du chroot et d'en créer un avec un UID fixe, que personne n'utilise sur l'hôte. On le fait dans le chroot :

```
groupadd -g 3000 nginx
useradd -r -d /var/empty -s /bin/false -u 3000 -g 3000 nginx
```

`-r` crée un compte système, `-d /var/empty` donne un home vide, `-s /bin/false` interdit d'ouvrir un shell avec ce compte, et `-u 3000 -g 3000` fixe l'UID et le GID.

## 2. Thin Nginx chroot 

Le chroot fat marche mais il est lourd. Ici on construit un chroot qui ne contient que Nginx et ce qu'il lui faut pour tourner.

### 2.1 Création du dossier

```
mkdir -p /chroot/thin/opt
```

Cette fois pas de debootstrap, le dossier est vide.

### 2.2 Copie de Nginx

```
cp -r /opt/nginx-1.30.6 /chroot/thin/opt/
cd /chroot/thin/opt/
ln -s nginx-1.30.6 nginx
```

On refait tout de suite le lien symbolique, que `cp -r` ne copie pas.

### 2.3 Modification de la page d'accueil

```
echo "In chroot" > /chroot/thin/opt/nginx-1.30.6/html/index.html
```

### 2.4 Analyse des dépendances

```
ldd /chroot/thin/opt/nginx-1.30.6/sbin/nginx
```

`ldd` liste les bibliothèques dont le binaire a besoin. Dans l'exemple de la consigne, on a `libdl`, `libpthread`, `libcrypt`, `libc.so.6` et `/lib64/ld-linux-x86-64.so.2` (la liste dépend de la version de la libc de la machine). Comme ces bibliothèques ne sont pas présentes dans le chroot thin, Nginx ne peut pas démarrer. Il faut donc les recopier dans le chroot en conservant la même arborescence que sur l'hôte.

### 2.5 Recréation de l'arborescence des bibliothèques

Il faut recopier les bibliothèques avec la même arborescence que sur l'hôte, sinon Nginx ne les trouve pas :

```
mkdir /chroot/thin/lib
mkdir -p /chroot/thin/lib/x86_64-linux-gnu/
mkdir /chroot/thin/lib64
```

Avant de copier, on regarde avec `file` si c'est un vrai fichier ou un lien symbolique :

```
file /lib/x86_64-linux-gnu/libdl.so.2
/lib/x86_64-linux-gnu/libdl.so.2: symbolic link to libdl-2.31.so

file /lib/x86_64-linux-gnu/libdl-2.31.so
/lib/x86_64-linux-gnu/libdl-2.31.so: ELF 64-bit LSB shared object, x86-64, ...
```

Ici `libdl.so.2` n'est qu'un lien vers `libdl-2.31.so`. Si on ne copie que le lien, il pointe vers rien dans le chroot, il faut copier le vrai fichier. On a copié les bibliothèques une par une dans `/chroot/thin/lib/x86_64-linux-gnu/`, en vérifiant avec `ls -al` que le dossier se remplissait.

Pour `libc.so.6`, `file` montre que c'est directement le fichier, donc on le copie tel quel :

```
file /lib/x86_64-linux-gnu/libc.so.6
cp /lib/x86_64-linux-gnu/libc.so.6 /chroot/thin/lib/x86_64-linux-gnu/
```

Pour `ld-linux`, on copie le vrai fichier au même endroit, puis on refait dans `lib64` le lien que demande `ldd` :

```
cd /chroot/thin/lib64
ln -s ../lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 ./ld-linux-x86-64.so.2
ls -al
```

Le lien pointe bien vers la bonne bibliothèque. Côté exécutable, on a tout.

### 2.6 Import des utilisateurs root et nginx

Comme pour le fat, il n'y a ni passwd ni group dans le chroot, et même pas de dossier `/etc` :

```
mkdir /chroot/thin/etc
grep root /etc/passwd > /chroot/thin/etc/passwd
grep nginx /etc/passwd >> /chroot/thin/etc/passwd
grep root /etc/group > /chroot/thin/etc/group
grep nginx /etc/group >> /chroot/thin/etc/group
```

Le premier `grep` de chaque fichier utilise `>` parce qu'on crée le fichier, le deuxième `>>` pour ajouter. On vérifie avec `cat /chroot/thin/etc/passwd` et `cat /chroot/thin/etc/group`.

### 2.7 Essai de lancement

```
chroot /chroot/thin /opt/nginx/sbin/nginx
```

Malgré toutes les bibliothèques copiées avec `ldd`, Nginx ne démarre toujours pas. En regardant l'erreur, on se rend compte qu'il manque encore `libnss_files.so.2`, qui sert à retrouver les utilisateurs et groupes définis dans les fichiers `passwd` et `group`. Cette bibliothèque n'apparaît pas dans la sortie de `ldd`, mais elle est pourtant nécessaire au démarrage de Nginx.

### 2.8 Installation du module NSS

```
ls -l /usr/lib/x86_64-linux-gnu/libnss_files*
cp /usr/lib/x86_64-linux-gnu/libnss_files.so.2 /chroot/thin/lib/x86_64-linux-gnu/
```

On regarde d'abord ce qu'il y a sur l'hôte, puis on copie le vrai fichier au même endroit dans le chroot, comme pour les autres bibliothèques.

### 2.9 Relancement de Nginx

```
chroot /chroot/thin /opt/nginx/sbin/nginx
```

Avec le module en place, Nginx retrouve son utilisateur et démarre. On vérifie dans le navigateur avec `http://localhost` : on doit voir « In chroot ».

### Comparaison de la taille

```
du -sh /chroot/fat
314M    /chroot/fat

du -sh /chroot/thin
6.2M    /chroot/thin
```

Le fat fait 314 Mo et le thin 6,2 Mo, soit environ 50 fois moins. Il y a aussi moins de choses à exploiter si Nginx est attaqué : il n'y a même pas de shell dans le thin.

## Conclusion

Ce TP montre deux façons d'isoler Nginx dans un chroot. Le chroot fat est simple à mettre en place puisqu'il contient un système Debian complet, mais il est volumineux. Le chroot thin demande davantage de travail car il faut identifier et recopier manuellement les bibliothèques et fichiers nécessaires au fonctionnement de Nginx. En contrepartie, il est beaucoup plus léger (6,2 Mo contre 314 Mo) et réduit encore davantage la surface d'attaque.
