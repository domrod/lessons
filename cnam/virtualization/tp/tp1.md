# TP1 - QEMU

### Objectifs
**Le but de ce TP est de préparer un environnement complet de virtualisation basé sur qemu-kvm, que l'on récupèrera en ligne avant de le compiler.
Puis, il s'agit de créer une machine virtuelle qui servira de base pour des environnements de développement, pour, par exemple, tester différentes plateformes comme le cas d'un site web.**

### Informations pratiques
Ce TP fait appel à la ligne de commandes, et vous demande donc un effort spécial pour entrer correctement au clavier les différentes lignes présentées ci-après.

Pour information, il ne faut pas confondre les symbôles **-** et **_** , ni le **O** (la voyelle o majuscule) et le **0** (zéro). La variable **$HOME** désigne le répertoire utilisateur courant.

Par ailleurs, lorsque les commandes sont trop longues pour figurer sur une seule ligne dans cet énoncé, on utilise ici le symbole « **\** », qui veut juste dire que la commande à taper sur l'ordinateur continue sur la ligne suivante.

L'environnement à utiliser sera celui des PC du CNAM sous Linux. Veillez donc à booter votre machine uniquement sous cet OS. Si vous avez un PC portable sous Linux, vous pouvez également faire le TP dessus, à condition que vous ayez l'accélération matérielle KVM activée. Pour vérifier cela, taper dans une console :

```bash
$ /sbin/lsmod | grep kvm
```
Si vous obtenez quelque chose du genre :
```bash
kvm_intel             121968  0
kvm                   287708 1 kvm_intel
```

vous avez l'accélération matérielle sur votre machine. Sinon, essayez (sous **root**)
```bash
$ /sbin/modprobe kvm
```
Il peut parfois être nécessaire de rebooter son PC et d'activer l'accélération matérielle dans le BIOS (si son matériel dispose effectivement de cette fonctionnalité, ce qui n'est pas le cas de tous les ordinateurs portables, même récents).

### Configuration

#### Récupérer QEMU

- Créer un répertoire pour QEMU et se rendre dans celui-ci :

```bash
$ mkdir -p $HOME/tp/virt
$ cd $HOME/tp/virt
```

- Aller dans ce répertoire et récupérer **qemu-kvm** à partir du lien suivant :

http://wiki.qemu.org/Download

- Prendre la dernière version : **2.9.0**

ou directement :
```bash
$ wget http://download.qemu.org/qemu-2.9.0.tar.xz
```

#### Récupérer une image ISO
Récupérer en parallèle une image iso de **Linux Debian** *netinstall*
```bash
$ mkdir -p $HOME/tp/virt/iso/debian
$ cd $HOME/tp/virt/iso/debian

$ wget -c http://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-8.8.0-amd64-netinst.iso
```

#### Compiler QEMU

- Revenir sur l'archive QEMU pour l'extraire :

```bash
$ cd $HOME/tp/virt/
$ tar xvJf qemu-2.9.0.tar.xz
```
- Se rendre dans le nouveau répertoire (pour configurer/compiler/installer qemu)

```bash
$ cd qemu-2.9.0
```
- Lister les options possibles :

```bash
$ ./configure --help | more
```

- Configurer et compiler QEMU :

```bash
$ ./configure --prefix=$HOME/tp/virt/qemu \
--target-list=x86_64-softmmu \
--enable-kvm \
--cpu=$(uname -m) \
--enable-vnc
```

puis

```bash
$ make && make install
```

- Vérifier l'install

```bash
$ ls $HOME/tp/virt/qemu
```

## Création de machines virtuelles

- Créer un répertoire pour une VM **debian**

```bash
$ mkdir -p $HOME/tp/virt/vm/debian
$ cd $HOME/tp/virt/vm/debian
```
- Créer votre premier disque dur virtuel et le vérifier

```bash
$ $HOME/tp/virt/qemu/bin/qemu-img create -f qcow2 vm_debian.qcow2 5G

$ $HOME/tp/virt/qemu/bin/qemu-img info vm_debian.qcow2
```

- Lancer sa première installation de VM

```bash
$ $HOME/tp/virt/qemu/bin/qemu-system-x86_64 \
--enable-kvm \
-drive file=vm_debian.qcow2,media=disk \
-drive file=$HOME/tp/virt/iso/debian/debian-8.8.0-amd64-netinst.iso,media=cdrom \
-boot order=dc \
-m 512 -smp 2 \
-netdev type=user,id=tp1 \
-device driver=e1000,netdev=tp1,mac=00:52:a1:b2:c3:d4 \
-usb -device usb-tablet -k fr \
-monitor stdio
```

- Pour visualiser celle-ci (si une fenêtre ne s'ouvre pas automatiquement) :

```bash
$ vncviewer localhost:5900
```

**Nota : à la fin de l'installation de la VM, dans la partie "Sélection des logiciels",
ne gardez pas l'option "Environnement de bureau Debian". Celle-ci installe un trop
 grand nombre de logiciels, ce qui rend la création de la VM très longue.
 Préférez plutôt la case "serveur SSH"**.

#### Arrêter sa VM
- Soit en tapant la commande suivante dans la *console qemu*

```bash
(qemu) system_powerdown
```
- Soit avec la commande suivante *dans la VM*

```bash
(debian)$ poweroff
```

#### Tests de différents paramétrages pour la VM

- CPU :
```bash
	-cpu host
```
- Performance :
```bash
	driver=virtio-net-pci (réseau, essais en performance)
```
- Élasticité :
```bash
	-balloon virtio (élasticité mémoire)
```
- Accès SSH
```bash
	-redir tcp:5522::22
```
puis, utiliser le « user » créé à l'installation
```bash
ssh debian_user@localhost -p 5522
```

- Tester si le système virtualisé détecte la virtualisation :
```bash
dmesg | grep -i kvm
```

## Environnement de test

- Utiliser des clones pour la suite

Création d'un clone :
```bash
$ qemu-img create -b vm_master.qcow2 -f qcow2 vm_clone.qcow2
```

- Inspecter à froid l'image disque de la VM

Conversion au format raw :
```bash
$ $HOME/tp/virt/qemu/bin/qemu-img convert -f qcow2 -O raw vm_clone.qcow2 vm_clone.raw
```
Inspection avec fdisk :
```bash
$ fdisk -l vm_clone.raw
```

#### Application : création d'un site WEB

- Accès WEB + site WEB dans la VM
```bash
-redir tcp:5580::80
```

##### Dans la **VM**
- Installer Apache et Php
```bash
$ aptitude install apache2 php5
```
- Éditer */var/www/html/index.html*
- Relancer apache
- Ouvrir un navigateur à l'adresse


	http://localhost:5580

- Créér /var/www/index.php :

```php
<?php
phpinfo();
?>
```
- Puis, créer un message :
```php
<?php
echo «<h1>CNAM></h1>»;
echo «<p>En direct du TP de virtualisation></p>»;
echo «<p>Sign&eacute; : ...></p>»;
?>
```
- Recharger la page web
