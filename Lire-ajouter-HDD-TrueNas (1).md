## Prérequis

 *- Brancher le disque dur à l'ordinateur*
 *- Avoir une machine Windows à disposition*
 *- Faire un `apt update & apt upgrade -y`*

# Lire contenu du disque

## Step 1 : Lire et monter le disque

### Interface Proxmox
 - Dans l'interface graphique Proxmox, sélectionner notre **Noeud** et se rendre dans **Disques**
 - Identifier notre disque précedemment brancher, si c'est un ancien disque Windows, on devrait voir la partition **ntfs** écrite dans l'onglet **Usage**

### Identifier la partition NTFS
- Installer le package **ntfs-3g**, qui permet de monter et de lire les partitions NTFS sous Linux : 
	- `apt install ntfs-3g`
- Identifier la partition NTFS : 
	- `lsblk`
- Confirmer le système de fichiers : 
	- `blkid`
	
### Monter la partition NTFS
- Créer un point de montage : 
	- `mkdir /mnt/ntfs-disk`
- Monter la partition avec **ntfs-3g**,  : 
	- `mount -t ntfs-3g /dev/sdX1 /mnt/ntfs-disk`
	- *Remplacez `/dev/sdX1` par le chemin de votre partition NTFS
- Vérifiez le contenu et lister les fichiers de la nouvelle partition monté : 
	- `ls -lh /mnt/ntfs-disk`


## Step 2 : Créer un partage Samba

### Installer Samba sur Proxmox
- `apt install samba`
- `smbd --version`

### Configurer le partage Samba
- Modifier le fichier de configuration Samba : 
	- `nano /etc/samba/smb.conf`
- Ajoutez une configuration de partage pour le disque NTFS à la fin du fichier : 
			*[ntfs-disk]
			path = /mnt/ntfs-disk
			browseable = yes
			read only = no
			guest ok = yes
			force user = root*
- Redémarrez le service Samba : 
	- `systemctl restart smbd`

### Accéder au partage
- Ouvrez **Explorateur de fichiers**
- Dans la barre d'adresse, entrez l'adresse IP de votre serveur Proxmox : 
	- `\\<IP_du_serveur>`
	

# Formater le disque et le monter dans TrueNas

## Prérequis
- *Avoir identifié le disque dans le menu Disque du noeud Proxmox*
- *Démonter le disque avec la commande suivante : `umount /mnt/ntfs-disk`*
- *Arrêter la VM TrueNas*
 
## Step 1 : Supprimer les partitions et formater en ext4

### Nettoyer les données
- Sélectionnez le disque cible
- Cliquez sur **Wipe Data** pour supprimer toutes les données et confirmer

### Supprimer et créer une nouvelle partition
- Vérifier que le disque a bien été démonté avec `lsblk`
- On va utiliser l'outil **gdisk** pour supprimer les partitions existantes et créer une nouvelle table de partition : 
	- `gdisk /dev/sdX`
	- Tapez `o` pour créer une nouvelle table de partition.
	- Tapez `w` pour sauvegarder et quitter.
- On créé une partition unique sur le disque avec `parted` *(installer l'utilitaire si il n'existe pas)*
	- `parted /dev/sdX mklabel gpt`
	- `parted /dev/sdX mkpart primary ext4 0% 100%`

### Formater la partition en ext4
- Formatez la partition nouvellement créée en **ext4** : 
	- `mkfs.ext4 /dev/sdX1`


## Step 2 : Monter le disque dans la VM TrueNas

### Récupérer le device-id de la partition du nouveau disque
- Avec cette commande : 
	- `ls -l /dev/disk/by-id/`

### Configurer le pass-through du disque physique
- Aller dans **/etc/pve/qemu-server/** et ouvrir le fichier de conf de la VM TrueNas : 
	- `nano /etc/pve/qemu-server/xxx.conf`
- Ajoutez une ligne pour passer le disque physique ext4 à la VM
	- `scsi1: /dev/disk/by-id/<device-id>,format=raw`
	- *Remplacez `<device-id>` par l'identifiant unique du disque.*
- Redémarrer la VM TrueNas

### Vérifier le disque dans TrueNAS
- Dans l'interface TrueNas, dans **Storage**, vérifier que le disque est bien remonté







