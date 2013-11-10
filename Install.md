Installing archlinux with zfs
=============================

Update pacman

    pacman -Sy
    pacman -S pacman

Add the [zfs repository](https://wiki.archlinux.org/index.php/ZFS#Archiso_tracking_repositoryrepository)
synchronized with the iso to /etc/pacman.conf.

    [demz-repo-archiso]
    Server = http://demizerone.com/$repo/$arch

Import the repo key:

    pacman-key -r 0EE7A126
    pacman-key --lsign-key 0EE7A126

Install zfs

     pacman -Sy
     pacman -S zfs

Partition the disk:

    parted /dev/disk/by-id/ata-ST3000DM001-9YN166_S1F0LGEY

Parted usage
------------
Select sectors as unit, where 1s=512b. Modern hard drives have 4 kb or 8s physical sector size.
In a gpt partition table sector 0 contains 
the mbr and sectors 1-33 contains the partition information. 
This means the first usable sector is 34 and if we want it to be divisible by 8 the
first useful sector is 40.

For SSDs there is a similar quantity that we need to take into account, the erase
sector size, which usually is 512k or 1M. This means the first useful sector for
an ssd would be 2048.

Since at the end of the hard drive the gpt table is replicated, sectors -1 -33
are not free, to be properly aligned the last partion must end at -41
(which is fine for HDs because the total number of sectors in the hd is a multiple of 8,
but it is not generally true for SSDs)

    Number  Start    End          Size         File system  Name  Flags
     1      40s      409639s      409600s      fat32              boot
     2      409640s  5860533127s  5860123488s

In the above partion there is a fat32 partition of 200M for the UEFI boot,
and a partion for the zfs pool beginning on a sector divisible by 8 and ending at -40.

The efi partition should have the boot flag:

    set 1 boot

Creating the zpool
------------------
Create a pool named wirh the hard drive model (ST3000DM001),
and with 2^12=4kb sector size:

    zpool create -o ashift=12 -O canmount=off -O checksum=sha256 \
     -R /mnt ST3000DM001 /dev/disk/by-id/ata-ST3000DM001-9YN166_S1F0LGEY-part2

Check that everything is fine:

    zpool list
    zpool status
    zpool get all

Create the root filesystem:

    zfs create -o compression=lz4 -o mountpoint=/ ST3000DM001/archlinux

The output of `mount` should show it as mounted.

Install the base packages:

    pacstrap /mnt base 

Check that everyting is fine:

    zfs list

Set the boot filesystem:

    zpool set bootfs=ST3000DM001/archlinux ST3000DM001

Chroot:

arch-chroot /mnt

Working in the chroot
---------------------
First add the repository:
    vi /etc/pacman.conf 

    [demz-repo-core]
    Server = http://demizerone.com/$repo/$arch

Install zfs:

    pacman -Sy
    pacman -S zfs

Set the cachefile

    zpool set cachefile=/etc/zfs/zpool.cache ST3000DM001
    systemctl enable zfs

Create the home filesystem (compression off):

    zfs create -o mountpoint=/home ST3000DM001/home

Then add the zfs module to the init ram disk

    vi /etc/mkinitcpio.conf 

Add 'zfs' to HOOKS, and delete fsck:

    HOOKS="base udev autodetect modconf block zfs filesystems keyboard"

Rebuild the ram disk:

    mkinitcpio -p linux

Creating the uefi partition
---------------------------

    mkdosfs -F32 -nEFI /dev/sdb1
    mkdir /boot/efi
    mount /dev/sdb1 /boot/efi/
    mkdir -p /boot/efi/EFI/arch
    cp /boot/vmlinuz-linux /boot/efi/EFI/arch/vmlinuz-arch.efi
    cp /boot/initramfs-linux.img /boot/efi/EFI/arch/initramfs-arch.img
    cp /boot/initramfs-linux-fallback.img /boot/efi/EFI/arch/initramfs-arch-fallback.img

Installing the UEFI shell
-------------------------
As in the [arch wiki](https://wiki.archlinux.org/index.php/Unified_Extensible_Firmware_Interface#UEFI_Shell_download_links)
download the [uefi shell](https://edk2.svn.sourceforge.net/svnroot/edk2/trunk/edk2/ShellBinPkg/UefiShell/X64/Shell.efi).

Move it to the following file:

     /boot/efi/shellx64.efi

Configuring the Uefi boot
-------------------------
Reboot the machine and start the UEFI shell from the UEFI motherboard menu. Then install
the kernel:

      bcfg boot add 3 fs0:\EFI\arch\vmlinuz-arch.efi "ArchLinux"

And add the boot options:

      bcfg boot -opt 3 "initrd=EFI/arch/initramfs-arch.img zfs=bootfs ro"

Single user:

      bcfg boot add 4 fs0:\EFI\arch\vmlinuz-arch.efi "ArchLinux single user"
      bcfg boot -opt 4 "initrd=EFI/arch/initramfs-arch.img zfs=bootfs single ro"

Single shell (for emergency):

      bcfg boot add 5 fs0:\EFI\arch\vmlinuz-arch.efi "ArchLinux init=/bin/sh"
      bcfg boot -opt 5 "initrd=EFI/arch/initramfs-arch.img zfs=bootfs init=/bin/sh"

Misc configuration
------------------

Network, first check the name of your interface:

    ip addr

For example 'enp3s0', then enable dhcpd:

    systemctl enable dhcpcd@enp3s0

Set the hostname:

    hostnamectl set-hostname hobbithole

Set the mobo time to UTC (greenwich) and set the timezone:

    timedatectl list-timezones
    timedatectl set-timezone Europe/Rome

Install xorg:

    pacman -S xorg

Add [haskell](https://wiki.archlinux.org/index.php/Haskell_package_guidelines#Haskell_packages)
servers to /etc/pacman.conf:

    [core]
    Include = /etc/pacman.d/mirrorlist

    [haskell-core]
    Server = http://xsounds.org/~haskell/core/$arch
    Server = http://www.kiwilight.com/haskell/core/$arch

    [extra]
    Include = /etc/pacman.d/mirrorlist

Add [yaourt](https://wiki.archlinux.org/index.php/Yaourt) repo:

    [archlinuxfr]
    SigLevel = Never
    Server = http://repo.archlinux.fr/$arch

package groups
--------------

Find [here](https://www.archlinux.org/groups/)

Adding users
------------

    useradd -m federico
    passwd federico

Add to wheel for sudo rights:

   gpasswd -a federico wheel

Installing avahi for hostname resolution
-----------------------------------------
Install avahi and nss-mdns 
Edit /etc/nsswitch.conf:

    hosts: files myhostname dns -> hosts: files myhostname mdns_minimal [NOTFOUND=return] dns

Enable avahi:

    avahi-daemon.service

Enabling ssh on demand
----------------------

    systemctl enable sshd.socket

Installing cpu microcode update
-------------------------------
Just add the package:

     pacman -S intel-ucode

Installing lightdm
------------------

    pacman -S lightdm lightdm-gtk3-greeter
    install -m 744 /usr/share/xsessions/

Add an entry for xmonad:

    cat > /usr/share/xsessions/xmonad.desktop << EOF
    [Desktop Entry]
    Name=Xmonad
    Comment=Tiling window manager
    Exec = xmonad
    TryExec= /usr/bin/xmonad
    Type=XSession
    EOF

    systemctl enable lightdm

Themes
-----
For gtk I use Adwaita from gnome themes:

    pacman -S gnome-themes

For icons, Ubuntu Human:

   pacman -S human-icon-theme


IMPORTANT!!!
------------
When updating the kernel remember to mount the uefi partition first.
After the update the module for FAT32 won't be there anymore.

    mount /dev/sdb1 /boot/efi/
    cp /boot/vmlinuz-linux /boot/efi/EFI/arch/vmlinuz-arch.efi
    cp /boot/initramfs-linux.img /boot/efi/EFI/arch/initramfs-arch.img
    cp /boot/initramfs-linux-fallback.img /boot/efi/EFI/arch/initramfs-arch-fallback.img

Infinality
----------
For better font rendering  you can add the infinality patched freetype
engine. There are precompiled packages. See
(wiki)[https://wiki.archlinux.org/index.php/Infinality-bundle%2Bfonts]

You must change the installed fonts with the infinality ones, e.g.
ttf-dejavu is replaced by t1-dejavu-ib.
