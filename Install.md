Installing archlinux with encrypted home
========================================

Partition the disk
------------------

    parted /dev/disk/by-id/ata-ST3000DM001-9YN166_S1F0LGEY

Parted usage
------------
Select sectors as unit, where 1s=512b. Modern hard drives have 4 kb or 8s
physical sector size.  In a gpt partition table sector 0 contains the mbr and
sectors 1-33 contains the partition information. This means the first usable
sector is 34 and if we want it to be divisible by 8 the first useful sector is
40.

For SSDs there is a similar quantity that we need to take into account, the
erase sector size, which usually is 512k or 1M. This means the first useful
sector for an ssd would be 2048.

Since at the end of the hard drive the gpt table is replicated, sectors -1 -33
are not free, to be properly aligned the last partion must end at -41 (which is
fine for HDs because the total number of sectors in the hd is a multiple of 8,
but it is not generally true for SSDs)

     $ sudo parted /dev/sda
     [sudo] password for federico:
     GNU Parted 3.2
     Using /dev/sda
     Welcome to GNU Parted! Type 'help' to view a list of commands.
     (parted) u s
     (parted) p
     Model: ATA SAMSUNG SSD 830 (scsi)
     Disk /dev/sda: 500118192s
     Sector size (logical/physical): 512B/512B
     Partition Table: gpt
     Disk Flags:

     Number  Start    End         Size        File system  Name  Flags
     1      2048s    206847s     204800s     fat32              boot, esp
     2      206848s  500117503s  499910656s  ext4

In the above partion there is a fat32 partition of 200M for the UEFI boot,
and a partion for ext4 beginning on a sector divisible by 8 and ending at -40.

The partition type must be created:

    mklabel gpt

Then the partition:

     mkpart

Ignore the warning about the partition not being properly aligned.

The efi partition should have the `boot`, and `esp` flag:

    set 1 boot
    set 1 esp

 The partition table for the 8TB hd is:

     Number  Start  End           Size          File system  Name  Flags
     1      40s    15628053127s  15628053088s  ext4


Creating the uefi partition
---------------------------

    mkdosfs -F32 -nEFI /dev/sdb1
    mkdir /tmp/boot/
    mv /boot/* /tmp/boot/
    mount /dev/sdb1 /boot/
    mv /tmp/boot/* /boot

Add the following fstab entry:

    LABEL=EFI   /boot   vfat  defaults,rw,relatime,fmask=0133,dmask=0022  0      0

Loading the system using systemd bootloader
------------------------------------------
This supersedes the previous method

    $ pacman -S intel-ucode
    $ bootctl update

    $ mkdir -p  /boot/EFI/BOOT/
    $ cp /usr/lib/systemd/boot/efi/systemd-bootx64.efi /boot/EFI/BOOT/
    $ mv /boot/EFI/BOOT/systemd-bootx64.efi /boot/EFI/BOOT/BOOTX64.EFI
    $ mkdir /boot/loader
    $ cp /usr/share/systemd/bootctl/loader.conf /boot/loader/
    $ vi /boot/loader/loader.conf

    $ cat /boot/loader/loader.conf
    default arch

    $ mkdir /boot/loader/entries
    $ cp /usr/share/systemd/bootctl/arch.conf /boot/loader/entries/
    $ vi /boot/loader/entries/
    $ cat /boot/loader/entries/arch.conf
    title   Arch Linux
    linux   /vmlinuz-linux
    initrd  /intel-ucode.img
    initrd  /initramfs-linux.img
    options root=UUID=eee538a2-8f02-42c2-97d6-15df80269f05 rootfstype=ext4 add_efi_memmap

Creating the encrypted home partition
-------------------------------------

    $ mkfs.ext4 -L HOME -O encrypt /dev/sdb1
    $ e4crypt add_key
    $ mount /dev/sdb1 /home
    $ mkdir /home/federico
    $ e4crypt set_policy 7211b15f45cbc441 /home/federico

Misc configuration
------------------

Set the mobo time to UTC (greenwich) and set the timezone:

    timedatectl list-timezones
    timedatectl set-timezone Europe/Rome
    $ hwclock --systohc
    $ vi /etc/locale.gen
    $ locale-gen
    $ cat /etc/locale.conf
    LANG=en_US.UTF-8

Set the hostname:

    hostnamectl set-hostname hobbithole

    $ cat /etc/hostname
    hobbithole
    $ cat /etc/hosts
    #
    # /etc/hosts: static lookup table for host names
    #

    #<ip-address>	<hostname.domain.org>	<hostname>
    127.0.0.1	localhost.localdomain	localhost
    ::1		localhost.localdomain	localhost
    127.0.0.1	hobbithole.localdomain	hobbithole

Configure the kernel image:

    $ cat /etc/mkinitcpio.conf
    ...
    MODULES="i915"
    ...
    HOOKS="base systemd autodetect modconf block filesystems keyboard fsck"

    $ mkinitcpio -p linux

    Set root password:
    passwd

    $ systemctl enable systemd-resolved
    $ systemctl enable systemd-networkd
    $ systemctl start systemd-networkd
    $ cat /etc/systemd/network/wired.network

    [Match]
    Name=enp3s0

    [Network]
    DHCP=ipv4

    $ systemctl restart systemd-networkd
    $ systemctl start systemd-resolved

    Set the time server

    $ timedatectl set-ntp true

Network, first check the name of: your interface:

    ip addr

For example 'enp3s0', then enable dhcpd:

    systemctl enable dhcpcd@enp3s0




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

Installing lightdm
------------------

    pacman -S lightdm lightdm-gtk3-greeter
    systemctl enable lightdm

install 'xsession.desktop' from aur.

Themes
-----
For gtk I use Adwaita from gnome themes:

    pacman -S gnome-themes-standard

For icons, Faience azur:

   pacman -S faience-icon-theme


Audio
-----

    pacman -S jack2-dbus
    pacman -S qjackctl
    pacman -S lib32-jack2   (for wine compatibility)

Configure quodlibet

    preferences -> playback
    outputpipeline: jackaudiosink

