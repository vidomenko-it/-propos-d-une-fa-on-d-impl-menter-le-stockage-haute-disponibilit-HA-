---
title: "À propos d'une façon d'implémenter le stockage haute disponibilité (HA) partagé RAID1 pour un cluster Proxmox de deux nœuds à partir de leurs disques."
categories:
  - Blog
tags:
  - Shared Volume
  - HA
  - Live Migration 
  - RAID
  - Proxmox
  - nvme
  - scsi
  - multipath
---
# 2. À propos d'une façon d'implémenter le stockage haute disponibilité (HA) partagé RAID1 pour un cluster Proxmox de deux nœuds à partir de leurs disques.

**La solution suivante, à mon avis, est simple et peu coûteuse à mettre en œuvre.**

**Il consiste en ce qui suit :**  

  1.	Il y a deux nœuds avec des lecteurs NVME (facultatif). Sur chacun, le paquet : "nvme-cli" ("nvme-tcp", on diffuse les disques sur le réseau depuis chaque nœud, on peut aussi utiliser le paquet icsi "tgt") et le paquet : "open-iscsi", le paquet : "multipath-tools" (comme l'une des méthodes, on peut utiliser pour améliorer les performances, on peut se lier, quand il est possible de connecter des nœuds avec plusieurs connexions réseau).  
  2.	Un réseau est établi entre ces nœuds.    
  3.	Sur l'un des nœuds, nous créons une machine virtuelle basée sur Linux "VSAN105" avec approximativement les caractéristiques suivantes : RAM 512 Mo, disque dur 3 Go et installons les packages : "mdadm" (nous allons construire une matrice RAID1 à partir des partitions correspondantes des disques nvme des nœuds : nvme discovery... , connect... , puis la supporter), "tgt" (pour diffuser la cible iscsi md0 à deux nœuds), "lvm2" (nous allons faire pvcreate, vgcreate à partir de la matrice md0 résultante).  
  4.	Nous configurons « open-iscsi » sur les nœuds. 
  5.	Nous lançons VSAN. Il diffuse iscsi md0 avec le VG correspondant.
  6.	Nous créons un stockage LVM partagé dans le centre de données Proxmox.
  7.	Faisons de cette machine virtuelle une machine qui fonctionne en RAM sans utiliser de disque. Il peut désormais fonctionner en mode migration en direct et prendre en charge le stockage partagé.
  8.	Nous transférons la machine virtuelle VSAN vers HA. 
  9.	La notification, le diagnostic et le dépannage des problèmes RAID sont gérés par les utilitaires du package « mdadm ».

## 1.	Préparation des nœuds. 
### 1.1. Il y a deux nœuds avec des lecteurs NVME. Chacun installe le paquet : « nvme-cli », et le paquet : « open-iscsi », le paquet : « multipath-tools » :
```
apt update 
apt install nvme-cli open-iscsi multipath-tools
```
### 1.2. Nous diffusons des disques sur le réseau à partir de chaque nœud :  

**Sur le nœud pve1 10.10.1.1**  

**Créons un fichier exécutable :**  
```
nano  /root/nvme_start.sh
#!/bin/bash
modprobe nvmet
modprobe nvmet-tcp
# Налаштування порта
mkdir /sys/kernel/config/nvmet/ports/1
cd /sys/kernel/config/nvmet/ports/1
echo 10.10.1.1 | tee -a addr_traddr > /dev/null
echo tcp | tee -a addr_trtype > /dev/null
echo 4420 | tee -a addr_trsvcid > /dev/null
echo ipv4 | tee -a addr_adrfam > /dev/null
echo "port 4420 IP 10.10.1.1 tcp ipv4"
mkdir /sys/kernel/config/nvmet/ports/2
cd /sys/kernel/config/nvmet/ports/2
echo 10.10.2.1 | tee -a addr_traddr > /dev/null
echo tcp | tee -a addr_trtype > /dev/null
echo 4420 | tee -a addr_trsvcid > /dev/null
echo ipv4 | tee -a addr_adrfam > /dev/null
echo "port 4420 IP 10.10.2.1 tcp ipv4"
name_nvme_disk=$(udevadm info --query=name /dev/disk/by-path/pci-0000:02:00.0-nvme-1)
echo "$name_nvme_disk"
        cd /sys/kernel/config/nvmet/subsystems
        mkdir ora10
        cd ora10
        echo "oral0"
        echo -n 1 > attr_allow_any_host
        nvme list | grep -w $name_nvme_disk | awk '{print "pve1-"$3}' > attr_serial
        nvme list | grep -w $name_nvme_disk | awk '{print "pve1-"$4}' > attr_model
        cd namespaces/
        mkdir "10"; cd "10"
        echo -n "/dev/${name_nvme_disk}" > device_path; echo -n 1 > enable;
        # Кожну підсистему, опісля конфигурації, приєднуємо до порту
        sleep 5
        ln -s /sys/kernel/config/nvmet/subsystems/ora10 /sys/kernel/config/nvmet/ports/1/subsystems/ora10
        ln -s /sys/kernel/config/nvmet/subsystems/ora10 /sys/kernel/config/nvmet/ports/2/subsystems/ora10
exit 0
```
```
root@pve1:~# ./root/nvme_start.sh
root@pve1:~# apt install tree

root@pve1:~#  tree /sys/kernel/config/nvmet/ 
/sys/kernel/config/nvmet/
├── hosts
├── ports
│   ├── 1
│   │   ├── addr_adrfam
│   │   ├── addr_traddr
│   │   ├── addr_treq
│   │   ├── addr_trsvcid
│   │   ├── addr_trtype
│   │   ├── addr_tsas
│   │   ├── ana_groups
│   │   │   └── 1
│   │   │       └── ana_state
│   │   ├── param_inline_data_size
│   │   ├── param_pi_enable
│   │   ├── referrals
│   │   └── subsystems
│   │       └── ora10 -> ../../../../nvmet/subsystems/ora10
│   └── 2
│       ├── addr_adrfam
│       ├── addr_traddr
│       ├── addr_treq
│       ├── addr_trsvcid
│       ├── addr_trtype
│       ├── addr_tsas
│       ├── ana_groups
│       │   └── 1
│       │       └── ana_state
│       ├── param_inline_data_size
│       ├── param_pi_enable
│       ├── referrals
│       └── subsystems
│           └── ora10 -> ../../../../nvmet/subsystems/ora10
└── subsystems
    └── ora10
        ├── allowed_hosts
        ├── attr_allow_any_host
        ├── attr_cntlid_max
        ├── attr_cntlid_min
        ├── attr_firmware
        ├── attr_ieee_oui
        ├── attr_model
        ├── attr_pi_enable
        ├── attr_qid_max
        ├── attr_serial
        ├── attr_version
        ├── namespaces
        │   └── 10
        │       ├── ana_grpid
        │       ├── buffered_io
        │       ├── device_nguid
        │       ├── device_path
        │       ├── device_uuid
        │       ├── enable
        │       ├── p2pmem
        │       └── revalidate_size
        └── passthru
            ├── admin_timeout
            ├── clear_ids
            ├── device_path
            ├── enable
            └── io_timeout

21 directories, 41 files
root@pve1:~# 
```
**Sur le nœud pve99 10.10.1.2**  

**Créons un fichier exécutable similaire :**  
```
nano  /root/nvme_start.sh 
#!/bin/bash
modprobe nvmet
modprobe nvmet-tcp
# # Налаштування порта
mkdir /sys/kernel/config/nvmet/ports/1
cd /sys/kernel/config/nvmet/ports/1
echo 10.10.1.2 | tee -a addr_traddr > /dev/null
echo tcp | tee -a addr_trtype > /dev/null
echo 4420 | tee -a addr_trsvcid > /dev/null
echo ipv4 | tee -a addr_adrfam > /dev/null
echo "port 4420 IP 10.10.1.2 tcp ipv4"
mkdir /sys/kernel/config/nvmet/ports/2
cd /sys/kernel/config/nvmet/ports/2
echo 10.10.2.2 | tee -a addr_traddr > /dev/null
echo tcp | tee -a addr_trtype > /dev/null
echo 4420 | tee -a addr_trsvcid > /dev/null
echo ipv4 | tee -a addr_adrfam > /dev/null
echo "port 4420 IP 10.10.2.2 tcp ipv4"
name_nvme_disk=$(udevadm info --query=name /dev/disk/by-path/pci-0000:05:00.0-nvme-1)
echo "$name_nvme_disk"
        cd /sys/kernel/config/nvmet/subsystems
        mkdir ora20
        cd ora20
        echo "ora20"
        echo -n 1 > attr_allow_any_host
        nvme list | grep -w $name_nvme_disk | awk '{print "pve99-"$3}' > attr_serial
        nvme list | grep -w $name_nvme_disk | awk '{print "pve99-"$4}' > attr_model
        cd namespaces/
        mkdir "20"; cd "20"
        echo -n "/dev/${name_nvme_disk}" > device_path; echo -n 1 > enable;
        # Кожну підсистему, опісля конфигурації, приєднуємо до порту
        sleep 5
        ln -s /sys/kernel/config/nvmet/subsystems/ora20 /sys/kernel/config/nvmet/ports/1/subsystems/ora20
        ln -s /sys/kernel/config/nvmet/subsystems/ora20 /sys/kernel/config/nvmet/ports/2/subsystems/ora20
exit 0
```
```
[root@pve99 ~]$ ./root/nvme_start.sh
[root@pve99 ~]$ apt install tree

[root@pve99 ~]$ tree /sys/kernel/config/nvmet/
/sys/kernel/config/nvmet/
├── hosts
├── ports
│   ├── 1
│   │   ├── addr_adrfam
│   │   ├── addr_traddr
│   │   ├── addr_treq
│   │   ├── addr_trsvcid
│   │   ├── addr_trtype
│   │   ├── addr_tsas
│   │   ├── ana_groups
│   │   │   └── 1
│   │   │       └── ana_state
│   │   ├── param_inline_data_size
│   │   ├── param_pi_enable
│   │   ├── referrals
│   │   └── subsystems
│   │       └── ora20 -> ../../../../nvmet/subsystems/ora20
│   └── 2
│       ├── addr_adrfam
│       ├── addr_traddr
│       ├── addr_treq
│       ├── addr_trsvcid
│       ├── addr_trtype
│       ├── addr_tsas
│       ├── ana_groups
│       │   └── 1
│       │       └── ana_state
│       ├── param_inline_data_size
│       ├── param_pi_enable
│       ├── referrals
│       └── subsystems
│           └── ora20 -> ../../../../nvmet/subsystems/ora20
└── subsystems
    └── ora20
        ├── allowed_hosts
        ├── attr_allow_any_host
        ├── attr_cntlid_max
        ├── attr_cntlid_min
        ├── attr_firmware
        ├── attr_ieee_oui
        ├── attr_model
        ├── attr_pi_enable
        ├── attr_qid_max
        ├── attr_serial
        ├── attr_version
        ├── namespaces
        │   └── 20
        │       ├── ana_grpid
        │       ├── buffered_io
        │       ├── device_nguid
        │       ├── device_path
        │       ├── device_uuid
        │       ├── enable
        │       ├── p2pmem
        │       └── revalidate_size
        └── passthru
            ├── admin_timeout
            ├── clear_ids
            ├── device_path
            ├── enable
            └── io_timeout

21 directories, 41 files
[root@pve99 ~]$ 
```
## 2.	Un réseau est établi entre ces nœuds.
Dans ce cas, deux ports de nœud sont connectés directement  

## 3.	Sur l'un des nœuds, nous créons une machine virtuelle "VSAN105" (10.10.1.3) basée sur Linux avec les caractéristiques suivantes (512 Mo, 3 Go) et installons les packages : "mdadm" (nous allons construire une matrice RAID1 à partir des partitions correspondantes des disques nvme des nœuds : nvme discovery... , connect... , puis la supporter), "tgt" (pour diffuser la cible iscsi md0 à deux nœuds), "lvm2" (à partir de la matrice md0 résultante, nous ferons pvcreate, vgcreate).

### 3.1. Préparation d’une machine virtuelle à utiliser comme fournisseur de stockage en cluster.

### 3.2. Choisissons Debian comme système d’exploitation.
**Nous installons tgt**
`apt install tgt`

`nano /etc/tgt/conf.d/tgtpve.conf` 
**Remplissage:**  
```
<target iqn.1993-08.org.debian:01:9e746ebec3e>
   backing-store /dev/md0
   initiator-address 10.10.1.1
   initiator-address 10.10.1.2
</target>
```

### 3.3. Installer « lvm2 » : 
`apt install lvm2`

### 3.4. Installer « nvme-cli » :
`apt install nvme-cli`

**Chargement des modules au démarrage du système :**  

`modprobe nvme_tcp && echo "nvme_tcp" > /etc/modules-load.d/nvme_tcp.conf`  

**Connecter:**  
```
nvme discover -t tcp -a 10.10.1.1 -s 4420
nvme connect -t tcp -n ora10 -a 10.10.1.1 -s 4420
nvme discover -t tcp -a 10.10.1.2 -s 4420
nvme connect -t tcp -n ora20 -a 10.10.1.2 -s 4420
```
**Pour se connecter au démarrage :**

`nano /etc/nvme/discovery.conf`  

**Insérons :**  
```
# Used for extracting default parameters for discovery
#
# Example:
# --transport=<trtype> --traddr=<traddr> --trsvcid=<trsvcid> --host-traddr=<host-traddr> --host-iface=<host-iface>
discover -t tcp -a 10.10.1.1 -s 4420
discover -t tcp -a 10.10.1.2 -s 4420
```
**Alors:**
```
systemctl enable nvmf-autoconnect.service
```
### 3.5. Préparation des disques (modification de la taille des secteurs selon les besoins)

**Node 1**  
```
root@pve1:~# nvme list
Node Generic SN Model Namespace Usage Format FW Rev
/dev/nvme1n1 /dev/ng1n1 S4EUNG0M328258D Samsung SSD 970 EVO Plus 250GB 1 214.99 GB / 250.06 GB 512 B + 0 B 1B2QEXM7
/dev/nvme0n1 /dev/ng0n1 50026B7282A726A4 KINGSTON SKC3000S512G 1 512.11 GB / 512.11 GB 4 KiB + 0 B EIFK31.6
```
**Vérification de la taille du bloc**  
```
root@pve1:~# nvme id-ns /dev/nvme0 -n 1 -H | grep &quot;LBA Format&quot;
[6:5] : 0 Most significant 2 bits of Current LBA Format Selected
[3:0] : 0x1 Least significant 4 bits of Current LBA Format Selected
LBA Format 0 : Metadata Size: 0 bytes - Data Size: 512 bytes - Relative Performance: 0x2 Good
LBA Format 1 : Metadata Size: 0 bytes - Data Size: 4096 bytes - Relative Performance: 0x1 Better (in use)
root@pve1:~#
```
**Si nécessaire, passez à 4k**  
``` 
root@pve1:~# nvme id-ns /dev/format --lbaf=1 /dev/nvme0n1
```
### 3.6. Nous partitionnons le disque.
```
root@pve1:~# fdisk /dev/nvme0n1
```
**Nous sauvegardons le balisage à utiliser lorsque nous remplacerons le disque et maintenant pour une sauvegarde :**  
```
root@pve1:~# sfdisk -d /dev/nvme0n1 > nvmeKINGSTON512.dump
root@pve1:~# cat nvmeKINGSTON512.dump
label: gpt
label-id: A1F37274-73E6-864F-B0B6-9BDD551BBD45
device: /dev/nvme0n1
unit: sectors
first-lba: 256
last-lba: 125026896
sector-size: 4096
/dev/nvme0n1p1 : start= 4096, size= 262144, type=0657FD6D-A4AB-43C4-84E5-0933C84B4F4F, uuid=B21C1B97-64EE-6948-AA0F-0BBA8797EB91
/dev/nvme0n1p2 : start= 266240, size= 104857600, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=E71CF74F-B553-5246-A649-3A5C45619225
/dev/nvme0n1p3 : start= 105123840, size= 16777216, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=47A78E4D-FC8E-F54B-8DAF-D52A7908590C
/dev/nvme0n1p4 : start= 121901056, size= 3125760, type=0657FD6D-A4AB-43C4-84E5-0933C84B4F4F, uuid=7F23E207-9E1A-1B42-962F-98BED3C1F479
```
**Pour restaurer ce modèle plus tard, vous pouvez faire :**  
```
# sfdisk /dev/nvme0n1 < nvmeKINGSTON512.dump
root@pve1:~#
```
**Nous cassons le deuxième disque de la même manière.**  
```
[root@pve99 ~]$ sfdisk /dev/nvme0n1 < nvmeKINGSTON512.dump
[root@pve99 ~]$ sfdisk -d /dev/nvme0n1 > nvmeKINGSTON512_pve99.dump
[root@pve99 ~]$ cat nvmeKINGSTON512_pve99.dump
label: gpt
label-id: BB0E5F37-05C1-104A-8CB5-6D4A7DA9073A
device: /dev/nvme0n1
unit: sectors
first-lba: 256
last-lba: 125026896
sector-size: 4096
/dev/nvme0n1p1 : start= 4096, size= 262144, type=C12A7328-F81F-11D2-BA4B-00A0C93EC93B, uuid=B139BB2A-C95D-1641-968F-3ED977E40B6A
/dev/nvme0n1p2 : start= 266240, size= 104857600, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=3CF0D8B4-A0C3-1145-B923-05D59A56B406
/dev/nvme0n1p3 : start= 105123840, size= 16777216, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=35726BB3-0E33-C440-B5A1-14F31EE527FD
/dev/nvme0n1p4 : start= 121901056, size= 3125760, type=0657FD6D-A4AB-43C4-84E5-0933C84B4F4F, uuid=4512150A-2074-5D44-90E3-58826E7A6152
[root@pve99 ~]$
```
## Passons maintenant à la machine virtuelle :

### 3.7. Installer mdadm :
`apt install mdadm`

### 3.8. Nous configurons les périphériques Raid1 /dev/nvme1n1p2 , /dev/nvme2n1p2 : 
```  
nvme-list
```

`root@debvsan:/home/vov# nvme list`    

```
|   Node     |  Generic    |           SN            |      Model        |  Namespace Usage    |         Format           |       FW      |   Rev     |
|-----------:|:-----------:|:-----------------------:|:-----------------:|:-------------------:|:------------------------:|:-------------:|:----------|
|/dev/nvme2n1|  /dev/ng2n1 |  9782709cba71d57d8e6b   |   pve99-KINGSTON  |           20        |  512.11  GB / 512.11  GB |  4 KiB +  0 B |  6.8.12-5 |
|/dev/nvme1n1|  /dev/ng1n1 |  f1f5c6929bb211d228f4   |   pve1-KINGSTON   |           10        |  512.11  GB / 512.11  GB |  4 KiB +  0 B |  6.8.12-5 |

```
```
root@debvsan:/home/vov#
mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/nvme1n1p2 /dev/nvme2n1p2
```
**Configurons « mdadm » pour reconstruire la matrice au redémarrage, puis mettons à jour l'initrd pour permettre à « mdadm » d'y rester.**  
```
mdadm --detail --scan | tee -a /etc/mdadm/mdadm.conf
update-initramfs –u
```
## 4. Nous imposons « open-iscsi » aux nœuds.

## 4.1. Nous installons des poches sur chacun des nœuds :
`apt install open-iscsi multipath-tools`

**Nous démarrons le service :**  

`systemctl start open-iscsi.service`  

### 4.2. Sur le premier nœud, nous nous connectons à la machine virtuelle, trouvons la cible et nous connectons :
`iscsiadm -m discovery -t st -p 10.10.1.3`

### 4.3. Sur le deuxième nœud : 
```
[root@pve99:~iscsiadm -m discovery -t st -p 10.10.1.3
10.10.1.3:3260,1 iqn.1993-08.org.debian:01:9e746ebec3e
```

## 5. Sur la machine virtuelle, nous faisons : 
```
# pvcreate /dev/md0
# vgcreate vg_nvme1 /dev/md0
```
**Sur les nœuds et dans la machine virtuelle, la commande : vgs affiche le groupe : vg_nvme1**  

## 6. Dans le shell graphique LVM de Datacenter Storage, cochez Partagé et sélectionnez les nœuds en conséquence.

## 7. Nous créons une machine virtuelle qui fonctionne uniquement en RAM sans disque. Nous l'avons mis en HA. Le système sera opérationnel jusqu'à ce que les deux nœuds, les deux disques ou plusieurs canaux réseau tombent en panne.

### 7.1. Dans le fichier /usr/share/initramfs-tools/scripts/local, recherchez les lignes 179-185 (après avoir fait une sauvegarde du fichier) :  
```
   checkfs "${ROOT}" root "${FSTYPE}"

	# Mount root
	# shellcheck disable=SC2086
	if ! mount ${roflag} ${FSTYPE:+-t "${FSTYPE}"} ${ROOTFLAGS} "${ROOT}" "${rootmnt?}"; then
		panic "Failed to mount ${ROOT} as root file system."
	fi
```
І **Et nous changeons ce code en ceci :**  
```
   #checkfs "${ROOT}" root "${FSTYPE}"

	# Mount root
	# shellcheck disable=SC2086
	mkdir /ramboottmp
	mount ${roflag} -t ${FSTYPE} ${ROOTFLAGS} ${ROOT} /ramboottmp
	mount -t tmpfs -o size=100% none ${rootmnt}
	cd ${rootmnt}
	cp -rfa /ramboottmp/* ${rootmnt}
	umount /ramboottmp
```
### 7.2. Sauvegarder le fichier. Et entrez la commande dans le terminal en tant que root :  

`mkinitramfs -o /boot/initrd.img-ramboot`  

### 7.3. Nous vérifions que le fichier a été créé dans le dossier /boot et remettons l'ancien fichier local dans le dossier /usr/share/initramfs-tools/scripts/local à sa place (ou supprimons toutes nos modifications que nous avons effectuées à l'étape 1).

### 7.4. Allez dans le dossier : /etc et recherchez le fichier : fstab, enregistrez-en une copie et modifiez-le, en recherchant quelque chose comme ceci dans les premières lignes :

`UUID= 321dba83-9a22-442b-b06b-185d7afe1088 / ext4 defaults 1 1`

**et changer en :**  

`none / tmpfs defaults 0 0` 

### 7.5 Créons un menu approprié lors du chargement :
**déposer:**
```
nano /etc/grub.d/40_custom
```
**Contenu:**
```
#!/bin/sh
exec tail -n +3 $0
# This file provides an easy way to add custom menu entries.  Simply type the
# menu entries you want to add after this comment.  Be careful not to change
# the 'exec tail' line above.
menuentry 'RAM-Debian GNU/Linux' --class debian --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-321dba83-9a22-442b-b06b-185d7afe1088' {
        load_video
        insmod gzio
        insmod part_msdos
        insmod ext2
        set root='hd0,msdos1'
        echo    'Loading Linux 6.1.0-28-amd64 ...'
        linux   /boot/vmlinuz-6.1.0-28-amd64 root=UUID=321dba83-9a22-442b-b06b-185d7afe1088 ro  quiet splash toram
        echo    'Loading initial ramdisk ...'
        initrd  /boot/initrd.img-ramboot
}
```
```
update-grub
```
**Nous obtenons le menu du grub.**

### 7.5. Créons maintenant ram.tar.gz, éteignons la machine virtuelle, démarrons la nouvelle machine virtuelle en mode liveCD et connectons le disque de cette machine virtuelle.
**Montez-le dans /mnt. Faisons-le:**  
```
# cd /mnt
# tar -czf /mnt/boot/ram.tar.gz .
```
### 7.6. Maintenant, lors du chargement de la machine virtuelle, nous sélectionnons le menu de démarrage RAM approprié. Après le chargement, déverrouillez le disque : 

#### 7.6.1. Arrêtons l'accès au disque :

**Pour ce faire, utilisez la commande :**  
```
echo 1 > /sys/block/sda/device/delete`
```
     
**Cela désactivera le périphérique /dev/sda au niveau du noyau. Le lecteur ne sera plus visible sur le système.**  

#### 7.6.2. Vérifions le statut :

**Assurons-nous que le disque n’apparaît plus dans la liste des périphériques :**  
```
lsblk
```

**Cela ressemblera à ceci :**  
```
root@debvsan:/home/vov# lsblk
NAME                            MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
sda                               8:0    0     8G  0 disk  
└─sda1                            8:1    0     8G  0 part  
root@debvsan:/home/vov#
```
```
echo 1 > /sys/block/sda/device/delete
```
```
lsblk
```
#### 7.6.3. Détacher un disque d'une machine virtuelle via le moniteur QEMU :  

##### 7.6.3.1. Connectez-vous au moniteur QEMU pour une VM spécifique : 
```
qm monitor 105
```

**Cela affichera tous les lecteurs connectés. Vous verrez quelque chose comme ceci :**  
```
info block  
```
**Cela affichera tous les lecteurs connectés. Vous verrez quelque chose comme ceci :**  
```
root@pve1:~# qm monitor 105 
Entering QEMU Monitor for VM 105 - type 'help' for help
qm> info block
drive-scsi0 (#block190): /dev/pve/vm-105-disk-1 (raw)
    Attached to:      scsi0
    Cache mode:       writeback, direct
    Detect zeroes:    unmap
qm>
```

**Démonter le lecteur :**
```
device_del scsi0
```

**Vérifiez quels appareils sont connectés :**
```
info pci
```

**Vous verrez une liste de périphériques PCI, y compris le contrôleur SCSI.
Par exemple:**  
```
Bus  9, device   1, function 0:
    SCSI controller: PCI device 1af4:1004
      PCI subsystem 1af4:0008
      IRQ 10, pin A
      BAR0: I/O at 0x1000 [0x103f].
      BAR1: 32 bit memory at 0xfd800000 [0xfd800fff].
      BAR4: 64 bit prefetchable memory at 0xfc000000 [0xfc003fff].
      id "virtioscsi0"
```
**Retirer:**  
```
qm> device_del virtioscsi0
```
**Sortir:**  
```
q
```

##### 7.3.4.2. Coller dans le fichier de configuration :
```
root@pve1:~# nano /etc/pve/qemu-server/105.conf
```
**Suivant : desabled=1
Exemple :**  
```
scsi0: local-lvm:vm-105-disk-1,disabled=1,aio=native,backup=0,discard=on,iothread=1,size=8G
scsihw: virtio-scsi-single,disabled=1
```
##### 7.3.4.3. Et pendant la migration nous verrons : 
```
()
Task viewer: VM 105 - Migrate
OutputStatus
Stop
Download
task started by HA resource agent
2025-01-04 00:34:24 use dedicated network address for sending migration traffic (10.10.1.1)
2025-01-04 00:34:24 starting migration of VM 105 to node 'pve1' (10.10.1.1)
2025-01-04 00:34:24 starting VM 105 on remote node 'pve1'
2025-01-04 00:34:28 start remote tunnel
2025-01-04 00:34:29 ssh tunnel ver 1
2025-01-04 00:34:29 starting online/live migration on unix:/run/qemu-server/105.migrate
2025-01-04 00:34:29 set migration capabilities
2025-01-04 00:34:29 migration downtime limit: 100 ms
2025-01-04 00:34:29 migration cachesize: 512.0 MiB
2025-01-04 00:34:29 set migration parameters
2025-01-04 00:34:29 start migrate command to unix:/run/qemu-server/105.migrate
2025-01-04 00:34:30 migration active, transferred 357.9 MiB of 4.0 GiB VM-state, 586.6 MiB/s
2025-01-04 00:34:31 migration active, transferred 735.9 MiB of 4.0 GiB VM-state, 543.8 MiB/s
2025-01-04 00:34:32 migration active, transferred 1.1 GiB of 4.0 GiB VM-state, 399.1 MiB/s
2025-01-04 00:34:33 migration active, transferred 1.3 GiB of 4.0 GiB VM-state, 502.5 MiB/s
2025-01-04 00:34:34 migration active, transferred 1.6 GiB of 4.0 GiB VM-state, 243.0 MiB/s
2025-01-04 00:34:35 migration active, transferred 1.8 GiB of 4.0 GiB VM-state, 320.5 MiB/s
2025-01-04 00:34:36 migration active, transferred 2.2 GiB of 4.0 GiB VM-state, 462.0 MiB/s
2025-01-04 00:34:37 migration active, transferred 2.5 GiB of 4.0 GiB VM-state, 443.7 MiB/s
2025-01-04 00:34:38 average migration speed: 457.0 MiB/s - downtime 73 ms
2025-01-04 00:34:38 migration status: completed
2025-01-04 00:34:42 migration finished successfully (duration 00:00:18)
TASK OK
```
### 8. Nous transférons la machine virtuelle de Dtacenter vers HA.
En utilisant l'interface graphique Proxmox, nous transférons la VM 105 vers HA.
### 9. La notification, le diagnostic et le dépannage de RAID1 relèvent des utilitaires du package « mdadm ».

#HauteDisponibilité #RepriseActivité #CyberSecurité
