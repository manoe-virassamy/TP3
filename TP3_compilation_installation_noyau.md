# Compte rendu TP3 : Configurer, compiler et installer le noyau

Suite du TP2. Cette séance couvre les étapes 5 à 8 du TP : configuration avec `menuconfig`, compilation, installation du noyau et vérification au redémarrage.

> **À savoir :** je n'ai pas compilé le noyau moi-même. Le prof nous a donné un dossier `linux-build` déjà compilé (une compilation complète prend des heures). On a vérifié que la compilation démarre, mais on s'est appuyés sur le build fourni.

## 1. Reprise de la session

Il faut d'abord relancer VirtualBox et démarrer la VM, puis se reconnecter depuis un terminal de ma machine :

```
ssh -p 2424 user@127.0.0.1
```

## 2. Configuration avec `menuconfig`

```
cd ~/src/linux-6.17
export TMPDIR=/build/tmp
make O=/build/linux-build menuconfig
```

`menuconfig` est une interface en ncurses où on choisit ce qu'on compile dans le noyau. Il y a environ 15 000 paramètres (par exemple la pile réseau : sockets Unix, TCP/IP, IPsec...). Avant `menuconfig`, il fallait éditer à la main le fichier `.config` option par option, ce qui était très long. Ici, le point de départ est le `.config` qui se trouve déjà dans `/build/linux-build`.

Le TP demande de vérifier quatre réglages sur les clés et signatures :

- *Additional X.509 keys for default system keyring* : vide ;
- *File name or PKCS#11 URI of module signing key* : vide ;
- *Automatically sign all modules* : décoché ;
- *Digital signature verification using multiple keyrings* : décoché.

Ils étaient **déjà bons** dans la config du prof, donc je n'ai rien changé. Pour le label (*General setup > Local version*), le TP demande `-custom`, mais il était déjà à `e4cyb`. Je l'ai laissé tel quel : le label est cosmétique, mais le changer aurait forcé une recompilation d'une grande partie du build du prof. Le noyau s'appelle donc `6.17.0e4cyb` (et pas `6.17.0-custom`).

J'ai quitté `menuconfig` avec Échap sans enregistrer, puisque je n'avais rien modifié.

## 3. Compilation

```
make O=/build/linux-build -j"$(nproc)"
```

Résultat :

```
user@debian:~/src/linux-6.17$ make O=/build/linux-build -j"$(nproc)"
make[1]: Entering directory '/build/linux-build'
  GEN     Makefile
  CALL    /home/user/src/linux-6.17/scripts/checksyscalls.sh
  DESCEND objtool
  INSTALL libsubcmd_headers
  DESCEND bpf/resolve_btfids
  INSTALL libsubcmd_headers
Kernel: arch/x86/boot/bzImage is ready  (#1)
make[1]: Leaving directory '/build/linux-build'
```

Ça s'est terminé presque tout de suite avec `bzImage is ready` : `make` ne recompile que ce qui a changé, et tout était déjà compilé. `-j"$(nproc)"` lance autant de compilations en parallèle que la VM a de cœurs.

## 4. Installation du noyau et des modules

Avant d'installer, je regarde la place sur le disque système (les modules s'installent sur `/`, pas sur `/build`) et le nom du noyau :

```
user@debian:~/src/linux-6.17$ df -h /
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        28G  3.5G   23G  14% /
user@debian:~/src/linux-6.17$ make O=/build/linux-build kernelrelease
make[1]: Entering directory '/build/linux-build'
6.17.0e4cyb
make[1]: Leaving directory '/build/linux-build'
```

Puis les deux commandes d'installation :

```
sudo make O=/build/linux-build modules_install
sudo make O=/build/linux-build install
```

`modules_install` copie les modules dans `/lib/modules/6.17.0e4cyb` (fin de la sortie, avec un `DEPMOD` qui génère les dépendances) :

```
  INSTALL /lib/modules/6.17.0e4cyb/kernel/virt/lib/irqbypass.ko
  XZ      /lib/modules/6.17.0e4cyb/kernel/virt/lib/irqbypass.ko.xz
  DEPMOD  /lib/modules/6.17.0e4cyb
make[1]: Leaving directory '/build/linux-build'
```

`install` copie le noyau dans `/boot`, génère l'initramfs et met à jour GRUB. Pas besoin de lancer `update-grub` à la main : sur Debian, l'installation le fait automatiquement (le prof l'a dit aussi).

## 5. Ce que contient `/boot`

```
user@debian:~$ ls /boot
config-6.12.43+deb13-amd64  initrd.img-6.12.43+deb13-amd64  System.map-6.12.48+deb13-amd64  vmlinuz-6.17.0e4cyb
config-6.12.48+deb13-amd64  initrd.img-6.12.48+deb13-amd64  System.map-6.17.0e4cyb
config-6.17.0e4cyb          initrd.img-6.17.0e4cyb          vmlinuz-6.12.43+deb13-amd64
grub                        System.map-6.12.43+deb13-amd64  vmlinuz-6.12.48+deb13-amd64
user@debian:~$ ls /lib/modules
6.12.43+deb13-amd64  6.12.48+deb13-amd64  6.17.0e4cyb
```

Les noyaux `6.12` sont les anciens, qu'on garde : si le nouveau ne démarre pas, on peut choisir l'ancien dans GRUB. Pour chaque version du noyau, il y a **4 fichiers** :

- **`vmlinuz-...`** : le noyau lui-même, un gros exécutable monolithique. C'est lui que GRUB charge (il ne faut surtout pas faire `cat` dessus, ça casse le terminal).
- **`initrd.img-...`** (initramfs) : une petite archive d'un système Linux minimal, chargée en RAM. Le noyau tout seul ne peut pas faire grand-chose, il lui faut des outils côté utilisateur pour monter la vraie partition racine (lire `fstab`, déchiffrer le disque, gérer LVM...). Une fois la racine montée, l'initramfs passe la main à systemd et disparaît.
- **`System.map-...`** : la table qui associe des adresses aux fonctions du noyau (dont les appels système).
- **`config-...`** : une copie du `.config` utilisé pour compiler.

Le prof nous a aussi montré le menu de GRUB : avec `e` sur une entrée, on voit sa configuration (les `insmod` pour lire la partition, la racine, la ligne `linux` pour le noyau et la ligne `initrd`). Donc GRUB charge noyau + initramfs, l'initramfs monte la racine, puis systemd démarre les services.

## 6. Redémarrage et vérification

Je redémarre depuis la VM :

```
sudo shutdown -r now
```

> Attention : j'ai d'abord tapé cette commande dans PowerShell sous Windows, où `sudo` est désactivé. Elle n'a rien fait, c'est dans la session SSH vers la VM qu'il fallait la lancer.

Après reconnexion en SSH, la bannière affichait déjà le nouveau noyau :

```
Linux debian 6.17.0e4cyb #1 SMP PREEMPT_DYNAMIC Fri Oct 16 09:47:59 CEST 2026 x86_64
```

et la vérification finale :

```
user@debian:~$ uname -r
6.17.0e4cyb
user@debian:~$ df -h /build
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1        59G   24G   33G  42% /build
```

La VM tourne bien sur le noyau installé (GRUB démarre par défaut sur le plus récent) et `/build` est remonté automatiquement grâce au `fstab`.

## 7. Lien avec le cours sur l'isolation

Pendant la séance, le prof a aussi expliqué les **conteneurs** :

- Un conteneur, c'est un processus à qui on donne des **instances privées** d'objets du noyau (les *namespaces* : pid, net, mnt, ipc, user, time, cgroup). Dans le noyau, la `task_struct` d'un processus a un pointeur `nsproxy` qui liste ses namespaces, et l'appel système `unshare` permet d'arrêter de partager un objet avec les autres.
- Les **cgroups** limitent la consommation de ressources. Namespaces + cgroups = un processus isolé, ce qui est la base de Docker (via `runc`).
- Docker conteneurise un service ; LXC conteneurise tout un espace utilisateur avec systemd. Ce n'est pas de la virtualisation : le noyau reste le même.

## 8. Ce qui n'a pas été fait

La section 9 du TP (fabriquer des paquets `.deb`) est marquée *optionnelle* et on ne l'a pas faite, le prof s'étant arrêté à « vous avez compilé votre premier noyau ».
