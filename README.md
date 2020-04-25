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
Boot partition has to be formatted with FAT32. It cannot be labeled (FAT file system does not support labels):

    mkfs.fat -F32 /dev/sda1

Swap partition is formatted with `mkswap`. Use `-L SWAP` to label it:

    mkswap -L SWAP /dev/sd2

All other partitions should be ext4. Label them with parameter `-L <name>`:

    mkfs.ext4 -L HOME /dev/sda3
    mkfs.ext4 -L ROOT /dev/sda4
    mkfs.ext4 -L DATA /dev/sdb1

You can check your disks and partitions with lsblk (list block) This example setup was generated by lsblk -o `NAME,PARTTYPENAME,FSTYPE,LABEL,TYPE,MOUNTPOINT`:

    NAME   PARTTYPENAME     FSTYPE LABEL   SIZE TYPE MOUNTPOINT
    sda                                  238.5G disk 
    ├─sda1 EFI System       vfat           260M part /boot
    ├─sda2 Linux swap       swap   SWAP     20G part [SWAP]
    ├─sda3 Linux filesystem ext4   HOME     20G part /home
    └─sda4 Linux filesystem ext4   ROOT  198.2G part /
    sdb                                  698.7G disk 
    └─sdb1 Linux filesystem ext4   DATA  698.7G part /data
    sr0                                   1024M rom  

Other convenient opstion is `lsblk -f`, or use `blkid`

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
### Connect to the internet

A working internet connection is required and assumed. A wired connection is setup automatically, if found. Wireless ones must be configured by the user. Verify your connection and update the repositories:

    ping 8.8.8.8
    pacman -Syy

### Install base system

Use `basestrap` to install the `base` and optionally the `base-devel` package groups and your preferred init (currently available: `openrc`, `runit`, and `s6`):

    basestrap /mnt base base-devel runit elogind-runit

### Install a kernel

Artix provides `linux-lts` and `linux` kernels. It is possible to experiment with other kernels, e.g. (`-ck`, `-pf`, `-zen`, etc.). Linux-firmware is recommended for recognizing devices (like network card)

    basestrap /mnt linux linux-firmware

### Generate the File System Table

Use fstabgen to generate /etc/fstab, use `-U` for UUIDs or `-L` for partition labels:

    fstabgen -U /mnt > /mnt/etc/fstab

check and verify /etc/fstab. See [ArchWiki/fstab](https://wiki.archlinux.org/index.php/Fstab).

## Base System Configuration

The base system configuration is done from the future root location. This is _especially_ practical for the boot loader configuration and installation. Therefore change to the new root location:

    artools-chroot /mnt

### Set the system clock

Browse regions and cities in `/usr/share/zoneinfo/`. Then create a symbolic link to your region and city as `/etc/localtime`.

    ln -sf /usr/share/zoneinfo/<Region>/<City>`

Run hwclock to generate /etc/adjtime:

    hwclock --systohc

### Localization

Install a text editor of your choice (let's use vim here) and edit `/etc/locale.gen`, uncommenting the locales you desire:

    pacman -Sy vim
    vim /etc/locale.gen

To set the locale systemwide, create or edit `/etc/locale.conf` (which is sourced by `/etc/profile`) or `/etc/bash/bashrc.d/artix.bashrc` or `/etc/bash/bashrc.d/local.bashrc`; user-specific changes may be made to their respective `~/.bashrc`, for example:

    export LANG="en_US.UTF-8"
    export LC_COLLATE="C"

Example `/etc/locale.conf`:

    LANG=en_US.UTF-8
    LC_TIME=en_DK.UTF-8
    LC_PAPER=de_DE.UTF-8
    LC_COLLATE=C

For more info see [ArchWiki/Locale](https://wiki.archlinux.org/index.php/Locale).

### Boot Loader

First, install `grub` and `os-prober` (for detecting other installed operating systems) and EFI boot manager:

    pacman -S grub os-prober efibootmgr

Modify Grub's default settings in `/etc/default/grub` or the configuration generating scripts in `/etc/grub.d/`. See [ArchWiki/GRUB](https://wiki.archlinux.org/index.php/GRUB#Generate_the_main_configuration_file).

Here I added `random.trust_cpu=on` (Arch Forums post [1](https://bbs.archlinux.org/viewtopic.php?id=248132) and [2](https://bbs.archlinux.org/viewtopic.php?pid=1816982#p1816982)) and the UUID of the swap partition to `resume` the system from hiberation. Also I let highlight the last used system to boot (`GRUB_DEFAULT=saved`, `GRUB_SAVEDEFAULT=true`):

    # GRUB boot loader configuration
    
    GRUB_DEFAULT=saved
    GRUB_TIMEOUT=5
    GRUB_DISTRIBUTOR="Artix"
    GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 random.trust_cpu=on resume=UUID=8a8c0b83-8b6f-4fea-bd4a-905e7b51ebb4 quiet"
    GRUB_CMDLINE_LINUX=""
    
    # Preload both GPT and MBR modules so that they are not missed
    GRUB_PRELOAD_MODULES="part_gpt part_msdos"
    
    # Uncomment to enable booting from LUKS encrypted devices
    #GRUB_ENABLE_CRYPTODISK=y
    
    # Set to 'countdown' or 'hidden' to change timeout behavior,
    # press ESC key to display menu.
    GRUB_TIMEOUT_STYLE=menu
    
    # Uncomment to use basic console
    GRUB_TERMINAL_INPUT=console
    
    # Uncomment to disable graphical terminal
    #GRUB_TERMINAL_OUTPUT=console
    
    # The resolution used on graphical terminal
    # note that you can use only modes which your graphic card supports via VBE
    # you can see them in real GRUB with the command `vbeinfo'
    GRUB_GFXMODE=auto
    
    # Uncomment to allow the kernel use the same resolution used by grub
    GRUB_GFXPAYLOAD_LINUX=keep
    
    # Uncomment if you want GRUB to pass to the Linux kernel the old parameter
    # format "root=/dev/xxx" instead of "root=/dev/disk/by-uuid/xxx"
    #GRUB_DISABLE_LINUX_UUID=true
    
    # Uncomment to disable generation of recovery mode menu entries
    GRUB_DISABLE_RECOVERY=true
    
    # Uncomment and set to the desired menu colors.  Used by normal and wallpaper
    # modes only.  Entries specified as foreground/background.
    GRUB_COLOR_NORMAL="light-blue/black"
    GRUB_COLOR_HIGHLIGHT="light-cyan/blue"
    
    # Uncomment one of them for the gfx desired, a image background or a gfxtheme
    #GRUB_BACKGROUND="/path/to/wallpaper"
    #GRUB_THEME="/path/to/gfxtheme"
    
    # Uncomment to get a beep at GRUB start
    #GRUB_INIT_TUNE="480 440 1"
    
    # Uncomment to make GRUB remember the last selection. This requires
    # setting 'GRUB_DEFAULT=saved' above.
    GRUB_SAVEDEFAULT="true"



Install grub for UEFI boot systems:
    grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=grub

For BIOS boot systems refer to [ArtixWiki/Installation](https://wiki.artixlinux.org/Main/Installation).

Create grub config
    grub-mkconfig -o /boot/grub/grub.cfg

It is paramount that `/boot` contains `initramfs-linux-fallback.img`, `initramfs-linux.img`, `vmlinuz-linux` otherwise it will not boot. Here is my `tree /boot`:

    /boot
    ├── EFI
    │   └── grub
    │       └── grubx64.efi
    ├── grub
    │   ├── fonts
    │   │   └── unicode.pf2
    │   ├── grub.cfg
    │   ├── grubenv
    │   ├── themes
    │   │   └── starfield
    │   │       ├── COPYING.CC-BY-SA-3.0
    │   │       ├── README
    │   │       ├── blob_w.png
    │   │       ├── boot_menu_c.png
    │   │       ├── boot_menu_e.png
    │   │       ├── boot_menu_n.png
    │   │       ├── boot_menu_ne.png
    │   │       ├── boot_menu_nw.png
    │   │       ├── boot_menu_s.png
    │   │       ├── boot_menu_se.png
    │   │       ├── boot_menu_sw.png
    │   │       ├── boot_menu_w.png
    │   │       ├── dejavu_10.pf2
    │   │       ├── dejavu_12.pf2
    │   │       ├── dejavu_14.pf2
    │   │       ├── dejavu_16.pf2
    │   │       ├── dejavu_bold_14.pf2
    │   │       ├── slider_c.png
    │   │       ├── slider_n.png
    │   │       ├── slider_s.png
    │   │       ├── starfield.png
    │   │       ├── terminal_box_c.png
    │   │       ├── terminal_box_e.png
    │   │       ├── terminal_box_n.png
    │   │       ├── terminal_box_ne.png
    │   │       ├── terminal_box_nw.png
    │   │       ├── terminal_box_s.png
    │   │       ├── terminal_box_se.png
    │   │       ├── terminal_box_sw.png
    │   │       ├── terminal_box_w.png
    │   │       └── theme.txt
    │   └── x86_64-efi
    │       ├── acpi.mod
    │       ├── adler32.mod
    │       ├── affs.mod
    │       ├── afs.mod
    │       ├── ahci.mod
    │       ├── all_video.mod
    │       ├── aout.mod
    │       ├── appleldr.mod
    │       ├── archelp.mod
    │       ├── at_keyboard.mod
    │       ├── ata.mod
    │       ├── backtrace.mod
    │       ├── bfs.mod
    │       ├── bitmap.mod
    │       ├── bitmap_scale.mod
    │       ├── blocklist.mod
    │       ├── boot.mod
    │       ├── boottime.mod
    │       ├── bsd.mod
    │       ├── bswap_test.mod
    │       ├── btrfs.mod
    │       ├── bufio.mod
    │       ├── cacheinfo.mod
    │       ├── cat.mod
    │       ├── cbfs.mod
    │       ├── cbls.mod
    │       ├── cbmemc.mod
    │       ├── cbtable.mod
    │       ├── cbtime.mod
    │       ├── chain.mod
    │       ├── cmdline_cat_test.mod
    │       ├── cmp.mod
    │       ├── cmp_test.mod
    │       ├── command.lst
    │       ├── configfile.mod
    │       ├── core.efi
    │       ├── cpio.mod
    │       ├── cpio_be.mod
    │       ├── cpuid.mod
    │       ├── crc64.mod
    │       ├── crypto.lst
    │       ├── crypto.mod
    │       ├── cryptodisk.mod
    │       ├── cs5536.mod
    │       ├── ctz_test.mod
    │       ├── date.mod
    │       ├── datehook.mod
    │       ├── datetime.mod
    │       ├── disk.mod
    │       ├── diskfilter.mod
    │       ├── div.mod
    │       ├── div_test.mod
    │       ├── dm_nv.mod
    │       ├── echo.mod
    │       ├── efi_gop.mod
    │       ├── efi_uga.mod
    │       ├── efifwsetup.mod
    │       ├── efinet.mod
    │       ├── ehci.mod
    │       ├── elf.mod
    │       ├── eval.mod
    │       ├── exfat.mod
    │       ├── exfctest.mod
    │       ├── ext2.mod
    │       ├── extcmd.mod
    │       ├── f2fs.mod
    │       ├── fat.mod
    │       ├── file.mod
    │       ├── fixvideo.mod
    │       ├── font.mod
    │       ├── fs.lst
    │       ├── fshelp.mod
    │       ├── functional_test.mod
    │       ├── gcry_arcfour.mod
    │       ├── gcry_blowfish.mod
    │       ├── gcry_camellia.mod
    │       ├── gcry_cast5.mod
    │       ├── gcry_crc.mod
    │       ├── gcry_des.mod
    │       ├── gcry_dsa.mod
    │       ├── gcry_idea.mod
    │       ├── gcry_md4.mod
    │       ├── gcry_md5.mod
    │       ├── gcry_rfc2268.mod
    │       ├── gcry_rijndael.mod
    │       ├── gcry_rmd160.mod
    │       ├── gcry_rsa.mod
    │       ├── gcry_seed.mod
    │       ├── gcry_serpent.mod
    │       ├── gcry_sha1.mod
    │       ├── gcry_sha256.mod
    │       ├── gcry_sha512.mod
    │       ├── gcry_tiger.mod
    │       ├── gcry_twofish.mod
    │       ├── gcry_whirlpool.mod
    │       ├── geli.mod
    │       ├── gettext.mod
    │       ├── gfxmenu.mod
    │       ├── gfxterm.mod
    │       ├── gfxterm_background.mod
    │       ├── gfxterm_menu.mod
    │       ├── gptsync.mod
    │       ├── grub.efi
    │       ├── gzio.mod
    │       ├── halt.mod
    │       ├── hashsum.mod
    │       ├── hdparm.mod
    │       ├── hello.mod
    │       ├── help.mod
    │       ├── hexdump.mod
    │       ├── hfs.mod
    │       ├── hfsplus.mod
    │       ├── hfspluscomp.mod
    │       ├── http.mod
    │       ├── iorw.mod
    │       ├── iso9660.mod
    │       ├── jfs.mod
    │       ├── jpeg.mod
    │       ├── keylayouts.mod
    │       ├── keystatus.mod
    │       ├── ldm.mod
    │       ├── legacy_password_test.mod
    │       ├── legacycfg.mod
    │       ├── linux.mod
    │       ├── linux16.mod
    │       ├── loadbios.mod
    │       ├── loadenv.mod
    │       ├── loopback.mod
    │       ├── ls.mod
    │       ├── lsacpi.mod
    │       ├── lsefi.mod
    │       ├── lsefimmap.mod
    │       ├── lsefisystab.mod
    │       ├── lsmmap.mod
    │       ├── lspci.mod
    │       ├── lssal.mod
    │       ├── luks.mod
    │       ├── lvm.mod
    │       ├── lzopio.mod
    │       ├── macbless.mod
    │       ├── macho.mod
    │       ├── mdraid09.mod
    │       ├── mdraid09_be.mod
    │       ├── mdraid1x.mod
    │       ├── memdisk.mod
    │       ├── memrw.mod
    │       ├── minicmd.mod
    │       ├── minix.mod
    │       ├── minix2.mod
    │       ├── minix2_be.mod
    │       ├── minix3.mod
    │       ├── minix3_be.mod
    │       ├── minix_be.mod
    │       ├── mmap.mod
    │       ├── moddep.lst
    │       ├── modinfo.sh
    │       ├── morse.mod
    │       ├── mpi.mod
    │       ├── msdospart.mod
    │       ├── mul_test.mod
    │       ├── multiboot.mod
    │       ├── multiboot2.mod
    │       ├── nativedisk.mod
    │       ├── net.mod
    │       ├── newc.mod
    │       ├── nilfs2.mod
    │       ├── normal.mod
    │       ├── ntfs.mod
    │       ├── ntfscomp.mod
    │       ├── odc.mod
    │       ├── offsetio.mod
    │       ├── ohci.mod
    │       ├── part_acorn.mod
    │       ├── part_amiga.mod
    │       ├── part_apple.mod
    │       ├── part_bsd.mod
    │       ├── part_dfly.mod
    │       ├── part_dvh.mod
    │       ├── part_gpt.mod
    │       ├── part_msdos.mod
    │       ├── part_plan.mod
    │       ├── part_sun.mod
    │       ├── part_sunpc.mod
    │       ├── partmap.lst
    │       ├── parttool.lst
    │       ├── parttool.mod
    │       ├── password.mod
    │       ├── password_pbkdf2.mod
    │       ├── pata.mod
    │       ├── pbkdf2.mod
    │       ├── pbkdf2_test.mod
    │       ├── pcidump.mod
    │       ├── pgp.mod
    │       ├── play.mod
    │       ├── png.mod
    │       ├── priority_queue.mod
    │       ├── probe.mod
    │       ├── procfs.mod
    │       ├── progress.mod
    │       ├── raid5rec.mod
    │       ├── raid6rec.mod
    │       ├── random.mod
    │       ├── rdmsr.mod
    │       ├── read.mod
    │       ├── reboot.mod
    │       ├── regexp.mod
    │       ├── reiserfs.mod
    │       ├── relocator.mod
    │       ├── romfs.mod
    │       ├── scsi.mod
    │       ├── search.mod
    │       ├── search_fs_file.mod
    │       ├── search_fs_uuid.mod
    │       ├── search_label.mod
    │       ├── serial.mod
    │       ├── setjmp.mod
    │       ├── setjmp_test.mod
    │       ├── setpci.mod
    │       ├── sfs.mod
    │       ├── shift_test.mod
    │       ├── shim_lock.mod
    │       ├── signature_test.mod
    │       ├── sleep.mod
    │       ├── sleep_test.mod
    │       ├── spkmodem.mod
    │       ├── squash4.mod
    │       ├── strtoull_test.mod
    │       ├── syslinuxcfg.mod
    │       ├── tar.mod
    │       ├── terminal.lst
    │       ├── terminal.mod
    │       ├── terminfo.mod
    │       ├── test.mod
    │       ├── test_blockarg.mod
    │       ├── testload.mod
    │       ├── testspeed.mod
    │       ├── tftp.mod
    │       ├── tga.mod
    │       ├── time.mod
    │       ├── tpm.mod
    │       ├── tr.mod
    │       ├── trig.mod
    │       ├── true.mod
    │       ├── udf.mod
    │       ├── ufs1.mod
    │       ├── ufs1_be.mod
    │       ├── ufs2.mod
    │       ├── uhci.mod
    │       ├── usb.mod
    │       ├── usb_keyboard.mod
    │       ├── usbms.mod
    │       ├── usbserial_common.mod
    │       ├── usbserial_ftdi.mod
    │       ├── usbserial_pl2303.mod
    │       ├── usbserial_usbdebug.mod
    │       ├── usbtest.mod
    │       ├── verifiers.mod
    │       ├── video.lst
    │       ├── video.mod
    │       ├── video_bochs.mod
    │       ├── video_cirrus.mod
    │       ├── video_colors.mod
    │       ├── video_fb.mod
    │       ├── videoinfo.mod
    │       ├── videotest.mod
    │       ├── videotest_checksum.mod
    │       ├── wrmsr.mod
    │       ├── xfs.mod
    │       ├── xnu.mod
    │       ├── xnu_uuid.mod
    │       ├── xnu_uuid_test.mod
    │       ├── xzio.mod
    │       ├── zfs.mod
    │       ├── zfscrypt.mod
    │       ├── zfsinfo.mod
    │       └── zstd.mod
    ├── initramfs-linux-fallback.img
    ├── initramfs-linux.img
    └── vmlinuz-linux
    
    7 directories, 314 files

### Add users

First, set the root passwd:

    passwd

Second, create a regular user and password:

    useradd -m user
    passwd user

### Network Configuration

Create a hostname (the network name of the computer), see [ArchWiki/hostname](https://wiki.archlinux.org/index.php/Network_configuration#Set_the_hostname):

    echo coolhostname > /etc/hostname

Install the DHCP client daemon:

    pacman -S dhcpcd

Tell Runit about it:

    ln -s /etc/runit/sv/dhcpcd /etc/runit/runsvdir/default/

Setup `/etc/hosts`, see [ArchWiki/Domain_name_resolution](https://wiki.archlinux.org/index.php/Domain_name_resolution). Here is an example (columns divided by tabs):

    # Static table lookup for hostnames.
    # See hosts(5) for details.
    127.0.0.1	localhost
    ::1	localhost
    127.0.1.1	lappi.localdomain	lappi

For wireless netowrking, install `wpa_supplicant`. It contains the main program `wpa_supplicant`, the passphrase tool `wpa_passphrase`, and the text front-end `wpa_cli`.

    pacman -S wpa_supplicant

Follow instructions in [ArchWiki/Network_configuration/Wireless](https://wiki.archlinux.org/index.php/Network_configuration/Wireless) and then in [ArchWiki/Wpa_supplicant](https://wiki.archlinux.org/index.php/Wpa_supplicant). It probably boils down to this:

Presuming the WiFi card is on PCI, list all PCI devides, find it there as `Network controller`, kernel module `iwlwifi`:
    lspci -k

Also check the output of the `ip link` command to see if a wireless interface was created, usually the interface is named `wlp2s0` or `wlan0`:

Then bring the interface up with:

    ip link set wlp2s0 up

Now use a network manager to establish a connection. In our case we are using `wpa_supplicant`. In order to use `wpa_cli`, a control interface must be specified for `wpa_supplicant`, and it must be given the rights to update the configuration. Do this by creating a minimal configuration file `/etc/wpa_supplicant/wpa_supplicant.conf`:

    ctrl_interface=/run/wpa_supplicant
    update_config=1

Now start wpa_supplicant with:

    wpa_supplicant -B -i interface -c /etc/wpa_supplicant/wpa_supplicant.conf

At this point run:

    wpa_cli

This will present an interactive prompt (`>`), which has tab completion and descriptions of completed commands.

    > scan
    OK
    <3>CTRL-EVENT-SCAN-RESULTS
    > scan_results
    bssid / frequency / signal level / flags / ssid
    00:00:00:00:00:00 2462 -49 [WPA2-PSK-CCMP][ESS] MYSSID
    11:11:11:11:11:11 2437 -64 [WPA2-PSK-CCMP][ESS] ANOTHERSSID

To associate with MYSSID, add the network, set the credentials and enable it:

    > add_network
    0
    > set_network 0 ssid "MYSSID"
    > set_network 0 psk "passphrase"
    > enable_network 0
    <2>CTRL-EVENT-CONNECTED - Connection to 00:00:00:00:00:00 completed (reauth) [id=0 id_str=]

Continue adding your networks. Each time you type `add_network`, the network ID will count up: `0, 1, 2, ...`. Finally save this network in the configuration file and quit wpa_cli:

    > save_config
    OK
    > quit

The `/etc/wpa_supplicant/wpa_supplicant.conf` should be now updated with SSIDs and passwords. However, it is safer to connect using the PSK passphrase. `wpa_passphrase` is a command line tool which generates the minimal configuration needed by wpa_supplicant. For example:

    wpa_passphrase MYSSID passphrase
    network={
        ssid="MYSSID"
        #psk="passphrase"
        psk=59e0d07fa4c7741797a4e394f38a5c321e3bed51d54ad5fcbd3f84bc7415d73d
    }

You can use this to append the output of `wpa_passphrase` to `/etc/wpa_supplicant/wpa_supplicant.conf` and then edit out the entries created by `wpa_cli` and delete the commented human-readable password lines from the appended `wpa_passphrase` outputs.

Once association is complete, you must obtain an IP address, for example, using `dhcpcd` [ArchWiki/Dhcpcd](https://wiki.archlinux.org/index.php/Dhcpcd).

Tell Runit about wpa_supplicant to make it start automatically:

    ln -s /etc/runit/sv/wpa_supplicant /etc/runit/runsvdir/default/

### Reboot the system

Exit chroot environment:

    exit

Unmount drives:

    umount -R /mnt

Reboot:

    reboot

If everything was right, system will boot and have a wireless internet connection.

## Post-Installation Configuration
