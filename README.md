# Compte rendu TP 3 : Nginx, compilation puis chroot fat et thin

Le but de ce TP est de compiler Nginx nous-mêmes (version 1.30.6, installé dans `/opt/nginx-1.30.6`), puis de le faire tourner dans un chroot. Un chroot, c'est un dossier que le processus voit comme sa racine `/` : il ne peut plus accéder au reste du système. Si Nginx se fait attaquer, l'attaquant reste coincé dedans. On fait d'abord un chroot « fat », avec un système Debian complet, puis un chroot « thin », avec seulement ce dont Nginx a besoin.

## Partie 1 : compilation de Nginx

On installe le serveur web Nginx en le compilant nous-mêmes à partir de son code source, au lieu de passer par le paquet Debian. Compiler soi-même permet de choisir exactement ce qu'on installe : le dossier d'installation, l'utilisateur sous lequel tourne le serveur et les modules qu'on garde ou qu'on enlève.

### Mise à jour du système

On commence par mettre à jour tout le système, y compris le noyau, avec :

```bash
apt-get dist-upgrade
```

Contrairement à `apt-get upgrade`, qui met à jour les paquets installés sauf le noyau, `dist-upgrade` met tout à jour, y compris le noyau. On a maintenant le nouveau paquet linux-image. On vérifie la version du noyau avec `uname -a` :

```bash
eleve@master-tp:~$ uname -a
Linux master-tp 6.12.115+deb13.1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.115-2 (2026-11-01) x86_64 GNU/Linux
```

On voit le nom de la machine (master-tp), la version du noyau (6.12.115) et l'architecture (x86_64).

### 2. Téléchargement de Nginx

Ensuite on télécharge les sources de Nginx depuis le site officiel :

```bash
root@master-tp:~# wget https://nginx.org/download/nginx-1.30.6.tar.gz
--2026-11-06 15:42:29--  https://nginx.org/download/nginx-1.30.6.tar.gz
Resolving nginx.org (nginx.org)... 3.125.197.172, 2a05:d014:5c0:2600::6, 2a05:d014:5c0:2601::6
Connecting to nginx.org (nginx.org)|3.125.197.172|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1331698 (1.3M) [application/octet-stream]
Saving to: 'nginx-1.30.6.tar.gz'

nginx-1.30.6.tar.gz    100%[=========================>]   1.27M  2.65MB/s    in 0.5s

2026-11-06 15:42:30 (2.65 MB/s) - 'nginx-1.30.6.tar.gz' saved [1331698/1331698]
```

On récupère une archive `.tar.gz` d'environ 1,3 Mo, qui contient tout le code source de Nginx.

### 3. Décompression de l'archive

Puis on désarchive et on décompresse l'archive :

```bash
tar -xzf nginx-1.30.6.tar.gz
```

Le `.tar.gz` est en fait deux choses : une archive tar qui regroupe tous les fichiers en un seul, puis une compression gzip. Dans la commande, `-x` sert à extraire, `-z` à décompresser le gzip et `-f` à indiquer le fichier. On obtient un dossier `nginx-1.30.6` avec les sources.

### 4. Installation de build-essential

Pour compiler, il faut gcc et make. On les installe via le paquet build-essential, qui contient gcc, make, etc. :

```bash
apt install build-essential
```

gcc est le compilateur qui transforme le code C de Nginx en exécutable, et make lance la compilation en suivant les instructions du Makefile.

### 5. Dossier des sources

On se place dans le dossier des sources :

```bash
cd nginx-1.30.6
```

### 6. Options de compilation

On affiche toutes les options de compilation disponibles :

```bash
root@master-tp:~/nginx-1.30.6# ./configure --help
```

La liste est longue, mais on s'intéresse surtout à trois options :

- `--prefix` : le dossier où Nginx sera installé ;
- `--user` : l'utilisateur non privilégié sous lequel tournent les processus worker ;
- `--group` : le groupe non privilégié de ces mêmes processus.

L'intérêt de `--user` et `--group` est que les processus qui traitent les requêtes ne tournent pas en root. Si quelqu'un exploite une faille dans Nginx, il n'aura que les droits de cet utilisateur limité.

### 7. Génération du Makefile

On configure le Makefile avec le script configure :

```bash
./configure --prefix=/opt/nginx-1.30.6 --user=nginx --group=nginx --without-http_rewrite_module --without-http_gzip_module
```

Le script vérifie que tout ce qu'il faut est présent sur la machine, puis génère le Makefile avec nos options. Ici, Nginx sera installé dans `/opt/nginx-1.30.6` et tournera sous l'utilisateur et le groupe nginx. Les options `--without-...` désactivent deux modules : le module rewrite (réécriture d'URL) et le module gzip (compression des réponses). Ils ne sont pas utiles pour ce TP, et ça évite d'installer les bibliothèques dont ils ont besoin.

### 8. Compilation

Maintenant on compile tout le code avec `make` :

```bash
make
```

make lit le Makefile généré à l'étape précédente et compile tous les fichiers sources un par un pour obtenir l'exécutable nginx.

### 9. Installation

Puis on l'installe avec `make install` :

```bash
make install
```

Cette commande copie l'exécutable et les fichiers de configuration dans le dossier choisi avec `--prefix`, c'est-à-dire `/opt/nginx-1.30.6`.

## Partie 2 : chroot fat

### 1. Création du dossier

```bash
mkdir -p /chroot/fat
```

L'option `-p` crée aussi les dossiers parents, ici `/chroot`.

### 2. Installation d'un système Debian dans le chroot avec debootstrap

```bash
debootstrap stable /chroot/fat/
```

debootstrap télécharge les paquets de base de Debian depuis un miroir et les installe dans le dossier. On obtient un système complet (`/bin`, `/lib`, `/etc`...) dans lequel on pourra entrer avec `chroot`. C'est ce qui le rend « fat » : il est gros alors que Nginx n'a besoin que de très peu de choses.

### 3. Copie de Nginx

```bash
cp -r /opt/nginx-1.30.6 /chroot/fat/opt/
```

Le chroot contient maintenant Nginx. Comme il a été compilé avec `--prefix=/opt/nginx-1.30.6`, il retrouve ses fichiers au même chemin une fois dans le chroot, puisque `/chroot/fat` devient `/`.

### 4. Modification de la page d'accueil

La page à modifier est donc celle du chroot, pas celle de l'hôte. On l'a éditée avec `vim.tiny` :

```bash
vim.tiny /chroot/fat/opt/nginx-1.30.6/html/index.html
```

On a ajouté un texte pour la reconnaître dans le navigateur. La consigne le fait en une ligne :

```bash
echo "In chroot" > /chroot/fat/opt/nginx-1.30.6/html/index.html
```

### 5. Entrée dans le chroot

```bash
chroot /chroot/fat /bin/bash
```

On obtient un bash qui voit `/chroot/fat` comme racine. Un `ls /` affiche le contenu du Debian installé par debootstrap, et non plus celui de l'hôte.

### 6. Vérification de la page

Depuis ce bash :

```bash
cat /opt/nginx-1.30.6/html/index.html
```

On voit bien « In chroot ». Le chemin est le même que sur l'hôte, mais sans le préfixe `/chroot/fat`.

### 7. Recréation du lien symbolique

En sortant du chroot, on a essayé de lancer Nginx directement :

```bash
chroot /chroot/fat /opt/nginx/sbin/nginx
```

Ça ne démarre pas : on a copié `/opt/nginx-1.30.6` mais pas le lien `/opt/nginx` qui pointe dessus, donc le chemin n'existe pas dans le chroot. On le recrée :

```bash
cd /chroot/fat/opt
ln -s nginx-1.30.6 nginx
```

Le lien est relatif, il marche aussi bien de l'intérieur que de l'extérieur du chroot.

### 8. Recréation de l'utilisateur nginx

Le lancement échoue encore, cette fois avec une erreur `[emerg]`. Nginx doit faire tourner ses workers sous l'utilisateur `nginx` et il ne le trouve pas. On compare les deux fichiers passwd :

```bash
grep nginx /etc/passwd
grep nginx /chroot/fat/etc/passwd
```

Le premier affiche la ligne de nginx (UID, GID, home, shell), le deuxième ne renvoie rien. Dans le chroot, c'est le `/etc/passwd` du chroot qui est lu, pas celui de l'hôte. Les UID sont les mêmes partout puisque le noyau est le même, mais c'est ce fichier qui associe un nom à un UID. Ici ce n'est donc pas une dépendance qui manque, c'est un problème de résolution du nom nginx.

On copie la ligne de l'hôte dans le passwd du chroot :

```bash
grep nginx /etc/passwd >> /chroot/fat/etc/passwd
```

Il faut bien mettre `>>` pour ajouter à la fin. Avec un seul `>`, on écraserait tout le fichier, root compris.

On a aussi regardé `/etc/shadow` (là où sont stockés les mots de passe) :

```bash
grep nginx /etc/shadow
```

Ça n'a pas suffi : en relançant Nginx, ça échouait toujours. Il lui manquait le groupe, qui est dans `/etc/group`. On copie aussi cette ligne :

```bash
grep nginx /etc/group
nginx:x:988:

grep nginx /etc/group >> /chroot/fat/etc/group
```

### 9. Lancement de Nginx dans le chroot

On sort du chroot avec `exit` et on lance :

```bash
chroot /chroot/fat /opt/nginx/sbin/nginx
```

Cette fois ça fonctionne. Dans le navigateur, `http://localhost` affiche « In chroot », donc c'est bien le Nginx du chroot qui répond. Pour voir sur quelle socket il écoute :

```bash
ss -tlnp | grep nginx
```

Il écoute sur le port 80. Le chroot isole les fichiers mais pas le réseau, donc c'est le même port que sur l'hôte.

### 10. UID utilisé par nginx

```bash
ps -eo user,uid,gid,pid,cmd | grep nginx
```

Le master tourne en root et les workers sous l'utilisateur nginx. Le problème possible, c'est que si on crée l'utilisateur dans le chroot avec `useradd`, l'UID est choisi sans regarder l'hôte. Si ce numéro est déjà celui d'un autre compte de l'hôte, Nginx aurait les droits de ce compte. Dans notre cas, on a copié les lignes de l'hôte, donc l'UID et le GID (988) sont les mêmes des deux côtés et il n'y a pas de collision.

### 11. Nouvel utilisateur avec un autre UID

La consigne propose de supprimer l'utilisateur du chroot et d'en créer un avec un UID fixe, que personne n'utilise sur l'hôte. On le fait dans le chroot :

```bash
groupadd -g 3000 nginx
useradd -r -d /var/empty -s /bin/false -u 3000 -g 3000 nginx
```

`-r` crée un compte système, `-d /var/empty` donne un home vide, `-s /bin/false` interdit d'ouvrir un shell avec ce compte, et `-u 3000 -g 3000` fixe l'UID et le GID.

## Partie 3 : chroot thin

Le chroot fat marche mais il est lourd. Ici on construit un chroot qui ne contient que Nginx et ce qu'il lui faut pour tourner.

### 1. Création du dossier

```bash
mkdir -p /chroot/thin/opt
```

Cette fois pas de debootstrap, le dossier est vide.

### 2. Copie de Nginx

```bash
cp -r /opt/nginx-1.30.6 /chroot/thin/opt/
cd /chroot/thin/opt/
ln -s nginx-1.30.6 nginx
```

On refait tout de suite le lien symbolique, que `cp -r` ne copie pas.

### 3. Modification de la page d'accueil

```bash
echo "In chroot" > /chroot/thin/opt/nginx-1.30.6/html/index.html
```

### 4. Analyse des dépendances

```bash
ldd /chroot/thin/opt/nginx-1.30.6/sbin/nginx
```

`ldd` liste les bibliothèques dont le binaire a besoin. Dans l'exemple de la consigne, on a `libdl`, `libpthread`, `libcrypt`, `libc.so.6` et `/lib64/ld-linux-x86-64.so.2` (la liste dépend de la version de la libc de la machine). Comme ces bibliothèques ne sont pas présentes dans le chroot thin, Nginx ne peut pas démarrer. Il faut donc les recopier dans le chroot en conservant la même arborescence que sur l'hôte.

### 5. Recréation de l'arborescence des bibliothèques

Il faut recopier les bibliothèques avec la même arborescence que sur l'hôte, sinon Nginx ne les trouve pas :

```bash
mkdir /chroot/thin/lib
mkdir -p /chroot/thin/lib/x86_64-linux-gnu/
mkdir /chroot/thin/lib64
```

Avant de copier, on regarde avec `file` si c'est un vrai fichier ou un lien symbolique :

```bash
file /lib/x86_64-linux-gnu/libdl.so.2
/lib/x86_64-linux-gnu/libdl.so.2: symbolic link to libdl-2.31.so

file /lib/x86_64-linux-gnu/libdl-2.31.so
/lib/x86_64-linux-gnu/libdl-2.31.so: ELF 64-bit LSB shared object, x86-64, ...
```

Ici `libdl.so.2` n'est qu'un lien vers `libdl-2.31.so`. Si on ne copie que le lien, il pointe vers rien dans le chroot, il faut copier le vrai fichier. On a copié les bibliothèques une par une dans `/chroot/thin/lib/x86_64-linux-gnu/`, en vérifiant avec `ls -al` que le dossier se remplissait.

Pour `libc.so.6`, `file` montre que c'est directement le fichier, donc on le copie tel quel :

```bash
file /lib/x86_64-linux-gnu/libc.so.6
cp /lib/x86_64-linux-gnu/libc.so.6 /chroot/thin/lib/x86_64-linux-gnu/
```

Pour `ld-linux`, on copie le vrai fichier au même endroit, puis on refait dans `lib64` le lien que demande `ldd` :

```bash
cd /chroot/thin/lib64
ln -s ../lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 ./ld-linux-x86-64.so.2
ls -al
```

Le lien pointe bien vers la bonne bibliothèque. Côté exécutable, on a tout.

### 6. Import des utilisateurs root et nginx

Comme pour le fat, il n'y a ni passwd ni group dans le chroot, et même pas de dossier `/etc` :

```bash
mkdir /chroot/thin/etc
grep root /etc/passwd > /chroot/thin/etc/passwd
grep nginx /etc/passwd >> /chroot/thin/etc/passwd
grep root /etc/group > /chroot/thin/etc/group
grep nginx /etc/group >> /chroot/thin/etc/group
```

Le premier `grep` de chaque fichier utilise `>` parce qu'on crée le fichier, le deuxième `>>` pour ajouter. On vérifie avec `cat /chroot/thin/etc/passwd` et `cat /chroot/thin/etc/group`.

### 7. Essai de lancement

```bash
chroot /chroot/thin /opt/nginx/sbin/nginx
```

Malgré toutes les bibliothèques copiées avec `ldd`, Nginx ne démarre toujours pas. En regardant l'erreur, on se rend compte qu'il manque encore `libnss_files.so.2`, qui sert à retrouver les utilisateurs et groupes définis dans les fichiers `passwd` et `group`. Cette bibliothèque n'apparaît pas dans la sortie de `ldd`, mais elle est pourtant nécessaire au démarrage de Nginx.

### 8. Installation du module NSS

```bash
ls -l /usr/lib/x86_64-linux-gnu/libnss_files*
cp /usr/lib/x86_64-linux-gnu/libnss_files.so.2 /chroot/thin/lib/x86_64-linux-gnu/
```

On regarde d'abord ce qu'il y a sur l'hôte, puis on copie le vrai fichier au même endroit dans le chroot, comme pour les autres bibliothèques.

### 9. Relancement de Nginx

```bash
chroot /chroot/thin /opt/nginx/sbin/nginx
```

Avec le module en place, Nginx retrouve son utilisateur et démarre. On vérifie dans le navigateur avec `http://localhost` : on doit voir « In chroot ».

### Comparaison de la taille

```bash
du -sh /chroot/fat
314M    /chroot/fat

du -sh /chroot/thin
6.2M    /chroot/thin
```

Le fat fait 314 Mo et le thin 6,2 Mo, soit environ 50 fois moins. Il y a aussi moins de choses à exploiter si Nginx est attaqué : il n'y a même pas de shell dans le thin.

## Conclusion

Ce TP montre comment compiler Nginx, puis deux façons de l'isoler dans un chroot. Le chroot fat est simple à mettre en place puisqu'il contient un système Debian complet, mais il est volumineux. Le chroot thin demande davantage de travail car il faut identifier et recopier manuellement les bibliothèques et fichiers nécessaires au fonctionnement de Nginx. En contrepartie, il est beaucoup plus léger (6,2 Mo contre 314 Mo) et réduit encore davantage la surface d'attaque.
