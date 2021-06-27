<h1 align="center">Arch Linux Installation Guide</h1>

## Preface
I can hear you say _Great, yet another Arch install guide_. And you are right, we (thankfully!) have a wide variety of excellent guides, tutorials and videos on how to install Arch Linux. So why write another? While I believe that there does not exist a single bad installation guide, none of the existing ones fully met my requirements. Talking about _my requirements_ already implies that this guide might not fully suit _your needs_. Feel free to adapt this guide to your liking. Your [feedback](mailto:arch-installation-guide-dev@branchonequal.com) is highly appreciated!

## Overview
Alright, this is what we are going to work with:
* UEFI,
* LVM on LUKS,
* Btrfs with subvolumes,
* systemd-boot,
* nftables,
* Reflector,
* Snapper with [Time Warp](https://github.com/branchonequal/timewarp) (shameless plug alert!),
* yay,
* GNOME 3.

## First Steps
Load the keymap if required:
```sh
loadkeys de-latin1  # Run "localectl list-keymaps" for a list of available keymaps.
```

Verify that the system was booted with UEFI:
```sh
ls /sys/firmware/efi/efivars  # Directory must not be empty.
```

Enable NTP and set the time zone:
```sh
timedatectl set-ntp true
timedatectl set-timezone Europe/Berlin
```

## Disk Setup
Start the partition editor:
```sh
cgdisk /dev/nvme0n1  # Replace "nvme0n1" with your device.
```

### Partition Layout
| Partition        | Size      | Type                |
| ---------------- | --------- | ------------------- |
| `/dev/nvme0n1p1` | 512M      | `EF00` (EFI system) |
| `/dev/nvme0n1p2` | Remainder | `8E00` (Linux LVM)  |

### LUKS
Initialize the LUKS partition and create the mapping for it:
```sh
cryptsetup luksFormat --type luks2 --cipher aes-xts-plain64 --key-size 512 /dev/nvme0n1p2
cryptsetup open /dev/nvme0n1p2 cryptlvm
```

### LVM
Create the physical volume:
```sh
pvcreate /dev/mapper/cryptlvm
```

Create the volume group:
```sh
vgcreate vg0 /dev/mapper/cryptlvm
```

Create the logical volumes:
```sh
lvcreate -L 48G vg0 -n swap  # 32G of memory * 1.5 to allow for hibernation.
lvcreate -l 100%FREE vg0 -n root
```

### File Systems
#### `/`
##### Subvolume Layout
We are roughly using the same subvolume layout as [openSUSE](https://en.opensuse.org/SDB:BTRFS) with the main difference of not creating a subvolume for `/var`. You are probably asking why.

When we are updating our system later on, we want to create a snapshot of our `/` subvolume before and after the transaction to be able to roll back the modification if necessary. A snapshot of the `/` subvolume does not include subvolumes which are mounted below `/` though which means that `/var` would also not be included. As the `pacman` database of installed packages is located under `/var/lib/pacman/local`, our database would become inconsistent after a rollback.

These are the subvolumes which we are going to create:
* `ROOT` &mdash; `/` subvolume,
* `.bootenv` &mdash; Boot environments for Time Warp,
* `.snapshots` &mdash; Snapper snapshots,
* `home`,
* `opt`,
* `root`,
* `srv`,
* `tmp`,
* `usr/local`,
* `var/cache`,
* `var/log`,
* `var/spool`,
* `var/tmp`.

Create the root file system:
```sh
mkfs.btrfs /dev/vg0/root
```

Mount it:
```sh
mount /dev/vg0/root /mnt
cd /mnt
```

Create the subvolumes:
```sh
mkdir usr
mkdir var
for name in ROOT .bootenv .snapshots home opt root srv tmp usr/local var/cache var/log var/spool var/tmp; do btrfs subvolume create $name; done
```

Make the `ROOT` subvolume the new `/`:
```sh
cd
umount /mnt
mount -o compress=lzo,noatime,subvol=ROOT /dev/vg0/root /mnt
cd /mnt
```

Now create the mountpoints for the newly created subvolumes:
```sh
for name in .bootenv .snapshots home opt root srv tmp usr/local var/cache var/log var/spool var/tmp; do mkdir -p $name; done  # No ROOT this time!
```

Mount the subvolumes:
```sh
for name in .bootenv .snapshots home opt root srv tmp usr/local var/cache var/log var/spool var/tmp; do mount -o compress=lzo,noatime,subvol=$name /dev/vg0/root $name; done
cd
```

#### `/boot`
Create and mount the `/boot` file system:
```sh
mkfs.fat -F32 /dev/nvme0n1p1
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```

### Swap
Create and activate swap:
```sh
mkswap /dev/vg0/swap
swapon /dev/vg0/swap
```

## Base System
Install the base system:
```sh
pacstrap /mnt base base-devel btrfs-progs git intel-ucode linux linux-firmware lvm2 man-db man-pages nano networkmanager vim zsh  # Choose amd-ucode if you have an AMD CPU; zsh is optional.
```

### `fstab`
Generate the `fstab`:
```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

We need to slightly modify the `fstab` as it will also be used by boot environments created from `/` snapshots. These use different `subvolid` and `subvol` mount options for `/` so we remove them from the corresponding entry. The correct mount options will be passed by the boot loader later on.

The resulting line in `/mnt/etc/fstab` should look something like this:
```
UUID=<UUID of your root device>    /    btrfs    rw,noatime,compress=lzo,space_cache    0 0
```

### System Setup
Enter the new system:
```sh
arch-chroot /mnt
```

Activate periodic TRIM:
```sh
systemctl enable fstrim.timer
```

#### Time Zone and Localization
Unfortunately, `timedatectl` and `localectl` do not work inside a chroot environment.

Set the time zone and sync the hardware clock:
```sh
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
hwclock --systohc --utc
```

Enable the desired locales in `/etc/locale.gen` and generate them:
```sh
locale-gen
```

Set the default locale by editing `/etc/locale.conf` if required:
```
LANG=de_DE.UTF-8
```

Set the keymap in `/etc/vconsole.conf` if required:
```
KEYMAP=de-latin1
```

#### Networking
Set the hostname in `/etc/hostname`:
```
arch
```

Set up the necessary entries in `/etc/hosts`:
```
127.0.0.1    localhost
::1          localhost
127.0.1.1    arch.localdomain arch
```

Enable NetworkManager:
```sh
systemctl enable NetworkManager
```

#### Root Password
Set the root password:
```sh
passwd
```

## initramfs
The next step is to set up the initramfs which is the initial root file system used during system boot. Modify the `HOOKS` line in `/etc/mkinitcpio.conf` as follows:
```
HOOKS=(base systemd keyboard autodetect modconf block sd-vconsole sd-encrypt lvm2 fsck filesystems)
```

Now generate the initramfs:
```sh
mkinitcpio -P
```

## Boot Loader
First install the boot loader:
```sh
bootctl install
```

Create the boot loader configuration file, `/boot/loader/entries/arch.conf`:
```
title   Arch Linux
linux   /vmlinuz-linux
initrd  /intel-ucode.img  # Or amd-ucode.img.
initrd  /initramfs-linux.img
options rd.luks.name=<UUID of your root partition>=cryptlvm root=/dev/vg0/root rootflags=subvol=ROOT rw
```

To get the UUID of the root partition, you can append it to the configuration file by entering
```sh
blkid -s UUID -o value /dev/nvme0n1p2 >> /boot/loader/entries/arch.conf
```

Modify the boot loader configuration in `/boot/loader/loader.conf`:
```
timeout 3
#console-mode keep
default arch.conf
```

## User Account
In order to be able to use the `sudo` command we first need to enable it for members of the `wheel` group. To do so, uncomment the line
```
%wheel ALL=(ALL) ALL
```
in the file `/etc/sudoers`. Do not edit the file directly though; use `visudo` instead. If you are like me and do not like editors from the '70s (sorry!), you can invoke `visudo` with
```sh
EDITOR=nano visudo
```
to use `nano` instead of `vi` for editing the file.

Now create the new user:
```sh
useradd -m -G wheel -c '<Full name>' -s /usr/bin/zsh <User name>  # Or /bin/bash if you prefer.
passwd <User name>
```

By passing the option `-G wheel` to `useradd`, the new user is added to the group `wheel` on creation.

## Reboot
Time to reboot!
```sh
exit
umount -R /mnt
reboot
```

## NTP
As we are out of the chroot environment now, we can enable NTP synchronization:
```sh
timedatectl set-ntp true
```

## Firewall
Now we are going to set up a basic nftables firewall. Install the `nftables` package:
```sh
pacman -S nftables
```

Enable the firewall:
```sh
systemctl enable nftables
systemctl start nftables
```

## Reflector
Reflector is a tool which helps keeping the `mirrorlist` up-to-date.

First, install Reflector and run it manually:
```sh
pacman -S reflector
reflector --verbose -l 5 -p https --sort rate --save /etc/pacman.d/mirrorlist
```
This will verbosely rate and sort the five most recently synchronized HTTPS mirrors, overwriting `/etc/pacman.d/mirrorlist`.

Now we want to run Reflector every time the `pacman-mirrorlist` package gets updated. To achieve this, first create the hook directory under `/etc/pacman.d`:
```sh
mkdir /etc/pacman.d/hooks
```

Then create a new hook file, [`reflector-update-mirrorlist.hook`](https://raw.githubusercontent.com/branchonequal/arch-installation-guide/main/data/reflector-update-mirrorlist.hook) under `/etc/pacman.d/hooks`:
```
[Trigger]
Type = Package
Operation = Upgrade
Target = pacman-mirrorlist

[Action]
Description = Updating the mirrorlist with Reflector...
Depends = reflector
When = PostTransaction
Exec = /bin/sh -c "reflector -l 5 -p https --sort rate --save /etc/pacman.d/mirrorlist; rm -f /etc/pacman.d/mirrorlist.pacnew"
```

## Snapper
Next we will install and set up Snapper.
```sh
pacman -S snapper
```

Unfortunately, Snapper insists on creating a new `.snapshots` subvolume when creating the root configuration and fails if the directory `.snapshots` already exists.

First, unmount the `.snapshots` subvolume and delete the mountpoint:
```sh
umount /.snapshots
rm -r /.snapshots
```

Now create the root configuration:
```sh
snapper -c root create-config /
```

Delete the `.snapshots` subvolume created by Snapper, recreate the `.snapshots` mountpoint and remount the original `.snapshots` subvolume:
```sh
btrfs subvolume delete /.snapshots
mkdir /.snapshots
mount -o compress=lzo,noatime,subvol=.snapshots /dev/vg0/root /.snapshots
```

The default root configuration created by Snapper will take hourly snapshots of `/` which we do not want so we disable this feature and also tweak some other settings in `/etc/snapper/configs/root`:
```
(...)
NUMBER_LIMIT="2-10"
NUMBER_LIMIT_IMPORTANT="4-10"
(...)
TIMELINE_CREATE="no"
(...)
```

## yay
`yay` is an AUR helper which makes it possible to install AUR packages just like normal ones. Install it like this (as you cannot run `makepkg` as root, we quickly change the user with `su`):
```sh
su - <User name>
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd
rm -rf yay
logout
```

## Graphical Environment
Yes, we are actually not setting up a server.

Install GNOME:
```sh
pacman -S gnome gnome-extra
```
This will automatically pull in the required Xorg packages.

Now install the driver for your graphics card:
```sh
pacman -S nvidia nvidia-utils  # Or xf86-video-ati + mesa or xf86-video-intel + mesa.
```

Set the keymap for Xorg:
```sh
localectl --no-convert set-x11-keymap de
```

Before activating GDM permanently, it might be a good idea to try it out first:
```sh
systemctl start gdm
```

If GDM starts and your GNOME session is working properly, you can enable GDM on start-up:
```sh
systemctl enable gdm
```

## Time Warp
When you have finalized the initial setup, you can install Time Warp now.

Install the package:
```sh
su - <User name>
git clone https://github.com/branchonequal/timewarp-arch-pkg.git
cd timewarp-arch-pkg
makepkg -si
cd
rm -rf timewarp-arch-pkg
logout
```

Create `/etc/xdg/timewarp/timewarp.conf` using the provided example. If you followed this guide, the configuration in `/etc/xdg/timewarp/timewarp.conf.example` should work out-of-the-box.

Enable the Time Warp service:
```sh
systemctl enable timewarp
systemctl start timewarp
```

For more information on Time Warp, see the [documentation](https://github.com/branchonequal/timewarp).

## Acknowledgments
* This installation guide was heavily influenced by [angristan](https://github.com/angristan)'s excellent [Arch Linux installation guide](https://github.com/angristan/arch-linux-install).
* [unicks.eu](https://www.youtube.com/channel/UCnZIn_CYjz0ErPs1ktH-2lQ) made some great videos about installing Arch Linux on Btrfs which helped me to better understand Btrfs and its features. Check him out on YouTube!
* The [Btrfs Wiki](https://btrfs.wiki.kernel.org/index.php/Main_Page) provided some valuable insight on the different [subvolume layouts](https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Layout).
