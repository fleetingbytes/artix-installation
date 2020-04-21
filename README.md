# artix-installation
Artix linux installation process, later to be used to improve the Artix wiki or to create a pdf manual

## Artix Live Environment

### Swap
If Artix live environment finds a swap partition when it is booting, it will automatically use it. You will need to turn off swap by `swapoff` if you want to modify or delete the swap partition

### Partition disks
    cfdisk /dev/sda
Create partitions as needed.

#### Partition types
Boot partition must be of type `EFI System`.
Swap partition must be of type `Linux swap`.
All other partitions should be `Linux filesystem`.

### Format disks
Boot partition has to be formatted with FAT32, like this:

    mkfs.fat -F32 /dev/sda1

All other partitions should be ext4. Label them with parameter `-L <name>`:

    mkfs.ext4 -L HOME /dev/sda2

This is my setup:

    NAME   PARTTYPENAME     FSTYPE LABEL   SIZE TYPE MOUNTPOINT
    sda                                  238.5G disk 
    ├─sda1 EFI System       vfat           260M part /boot
    ├─sda2 Linux swap       swap   SWAP     20G part [SWAP]
    ├─sda3 Linux filesystem ext4   HOME     20G part /home
    └─sda4 Linux filesystem ext4   ROOT  198.2G part /
    sdb                                  698.7G disk 
    └─sdb1 Linux filesystem ext4   DATA  698.7G part /data
    sr0                                   1024M rom  

### Mount partitions
You will have to create directories for the non-root partitions. 

    mkdir /mnt/boot
    mkdir /mnt/home
    mkdir /mnt/data

If you have freshly created partitions, you can use their labels when mounting them. Swap partition is not mounted. Just activate it with `swapon` or deactivate with `swapoff`.

    swapon /dev/disk/by-label/SWAP
    mount /dev/disk/by-label/ROOT /mnt
    mount /dev/disk/by-label/HOME /mnt/home
    mount /dev/disk/by-label/BOOT /mnt/boot
    mount /dev/disk/by-label/DATA /mnt/data

If the partitions were not created in this session, they have to be mounted by their device names, `mount /dev/sda1 /mnt/boot` etc. 
