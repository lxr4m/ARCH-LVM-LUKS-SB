# 🐧ARCH + 🗃️LVM + 🔐LUKS + 🛅Secure Boot
# DESCRIPTION
This setup guide is for minimal Arch system with following features:
- GNOME desktop environment
- LVM partition
- LUKS encryption
- Secure Boot

Arch will be installed on bare metal using the entire space of the drive.

# Table of Contents

- [Preparation](#preparation)
  - [Set Comfortable Font](#set-comfortable-font)
  - [(Optional) Configure Wireless Network](#optional-configure-wireless-network)
  - [Set Root Password for SSH Connection](#set-root-password-for-ssh-connection)
  - [Get Your Local IP](#get-your-local-ip)
  - [SSH from Another Computer on Your Network](#ssh-from-another-computer-on-your-network)
- [Begin Installation](#begin-installation)
  - [Update Arch Keyring](#update-arch-keyring)
  - [Enable Network Time Protocol (NTP)](#enable-network-time-protocol-(NTP))
  - [Partition the Drive for LVM on LUKS](#partition-the-drive-for-lvm-on-luks)
    - [Check the Name of the Drive](#check-the-name-of-the-drive)
    - [Use cfdisk to Create the Partitions](#use-cfdisk-to-create-the-partitions)
  - [Format Partitions](#format-partitions)
    - [EFI Partition](#efi-partition)
    - [Boot Partition](#boot-partition)
  - [Encrypt LVM Partition](#encrypt-lvm-partition)
    - [Create Encrypted Container](#create-encrypted-container)
    - [Check LUKS Header Information](#optional-check-luks-header-information)
    - [Open Encrypted LUKS Container](#open-encrypted-luks-container)
  - [Create Volumes](#create-volumes)
    - [Create Physical Volume](#create-physical-volume)
    - [Create Volume Group for Logical Volumes](#create-volume-group-for-logical-volumes)
    - [Create Logical Volumes](#create-logical-volumes)
    - [Format Logical Volumes](#format-logical-volumes)
    - [Mount Filesystems](#mount-filesystems)
    - [Enable Swap Space](#enable-swap-space)
  - [Install Base System](#install-base-system)
    - [Pacstrap System Packages](#pacstrap-system-packages)
    - [Generate /etc/fstab File](#generate-etcfstab-file)
    - [Make /tmp Write to RAM](#make-tmp-write-to-ram)
  - [Chroot into the New System](#chroot-into-the-new-system)
    - [Refresh Pacman Keyring](#refresh-pacman-keyring)
  - [Set Up the Environment](#set-up-the-environment)
    - [Set Your Time Zone](#set-your-time-zone)
    - [Enable the Specified Locales](#enable-the-specified-locales)
    - [Generate Locales](#generate-locales)
    - [Update locale.conf File](#update-localedef-file)
    - [Set Machine Hostname in /etc/hostname](#set-machine-hostname-in-etchostname)
    - [Add Local Domain Info to /etc/hosts](#add-local-domain-info-to-etchosts)
    - [Enable Pacman Animation and Color](#enable-pacman-animation-and-color)
  - [Install Additional Packages](#install-additional-packages)
    - [System](#system)
    - [Tools](#tools)
    - [Connections](#connections)
    - [Fonts](#fonts)
  - [Install Desktop Environment](#install-desktop-environment)
    - [Barebones GNOME Setup](#barebones-gnome-setup)
    - [Standard GNOME Setup](#standard-gnome-setup)
  - [Set Autologin for DE](#set-autologin-for-de)
  - [Enable Services](#enable-services)
  - [Set Up User Accounts](#set-up-user-accounts)
    - [Set Root Password](#set-root-password)
    - [Create a User](#create-a-user)
    - [Edit the Sudoers](#edit-the-sudoers)
    - [(Optional) Set Persistent Console Font](#optional-set-persistent-console-font)
  - [Configure mkinitcpio.conf Hooks for systemd Initramfs](#configure-mkinitcpio.conf-hooks-for-systemd-initramfs)
  - [Recreate the Initramfs Image](#recreate-the-initramfs-image)
  - [Configure GRUB](#configure-grub)
    - [Install Grub with Secure Boot Support](#install-grub-with-secure-boot-support)
    - [Get UUID for the Encrypted Drive](#get-uuid-for-the-encrypted-drive)
    - [Specify Root in Grub](#specify-root-in-grub)
  - [Re-generate Grub Config](#re-generate-grub-config)
  - [Exit Chroot and Reboot](#exit-chroot-and-reboot)
- [Enable Secure Boot](#enable-secure-boot)
  - [UEFI Config](#uefi-config)
  - [Setup Using SBCTL](#setup-using-sbctl)
    - [Install sbctl](#install-sbctl)
    - [Check Secure Boot Status](#check-secure-boot-status)
    - [Create Secure Boot Keys](#create-secure-boot-keys)
    - [Enroll Custom Secure Boot Keys](#enroll-custom-secure-boot-keys)
    - [Confirm Setup Mode is Disabled](#confirm-setup-mode-is-disabled)
    - [Sign Bootloader and Kernels Before Rebooting](#sign-bootloader-and-kernels-before-rebooting)
 - [Notes](#notes)
    - [Backup LUKS Header](#backup-luks-header)
- [Congratulations! 🎉](#congratulations)


# PREPARATION
1. Download the latest ISO from [archlinux.org](https://archlinux.org/download/)
2. Transfer the ISO to the USB drive using [Rufus](https://rufus.ie/en/) or [Ventoy](https://www.ventoy.net/en/index.html)
3. Boot your system from the USB drive into Arch live environment


### Set Font
To a more readable size for better visibility in the terminal
```sh
setfont ter-128b
```

### (Optional) Configure Wireless Network

In order to be able to SSH into the system and access the Internet

Ethernet should be enabled by default

```sh
iwctl
station list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect NETWORKNAME
exit
ping 8.8.8.8
```

At this point, you should have a successful connection to the Internet

### Set root password for SSH connection

Can be a generic one since it'll only be used for the setup

```sh
passwd
```

### Get your local IP
We'll use it to SSH into the system.
```sh
ip a
```
Should see something like `192.168.1.100`

## SSH from another computer on your network

Use the IP and password from the previous steps

```sh
ssh root@192.168.1.100
```

# BEGIN INSTALLATION

### Update Arch Keyring
To ensure the security and integrity of packages
```sh
pacman -Sy archlinux-keyring
```

### Enable Network Time Protocol (NTP)
To synchronize the system clock with internet time servers
```sh
timedatectl set-ntp true
```

## Partition the drive for LVM on LUKS

### Check the name of the drive
Lists all block devices
```sh
lsblk
```
For example, _nvme0n1_

### Use _cfdisk_ to create the partitions
```sh
cfdisk /dev/nvme0n1
```

> [!caution] DATA LOSS RISK!
> Perform the next steps only if you're ready to wipe all data from the drive

1. Delete all existing partitions
2. New Partition -> Partition Size: **1G** -> Type: **EFI System**
3. New Partition -> Partition Size: **1G** -> Type: **Linux Filesystem**
4. New Partition -> Partition Size: **510G** (rest of the space) -> Type: **Linux LVM**
5. Write partition table to disk


## Format Partitions

### EFI partition
```sh
mkfs.fat -F32 -n EFI /dev/nvme0n1p1
```
- `-F32`: specifies FAT32 file system
- `-n`: sets the label

### Boot Partition
```sh
mkfs.ext4 -L BOOT /dev/nvme0n1p2
```
- `-L`: sets the label

## Encrypt LVM partition

### Create Encrypted Container

```sh
cryptsetup luksFormat -v --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 5000 --use-random /dev/nvme0n1p3
```
- `-v`: verbose mode, gives detailed output
- `--cipher aes-xts-plain64`: specifies the encryption algorithm 
- `--key-size 512`: sets the size of the encryption key to 512 bits
- `--hash sha512`: produces a 512-bit hash and enhances security
- `--iter-time 5000`: target time in milliseconds for the key derivation function, helps slow down brute-force attacks by increasing the time required to derive the encryption key
- `--use-random`: use random data from the system's entropy pool to generate the encryption key, makes the key less predictable

> Read about different keys for cryptsetup command and adjust it to your needs

> [!important] USE A STRONG PASSWORD!
>
> High entropy is your friend. At least 65 bits. Which means 14 random chars from a-z or a random English sentence of \> 108 characters length


#### (Optional) Check LUKS header information
```sh
cryptsetup luksDump /dev/nvme0n1p3
```

### Open encrypted LUKS container
Map it to a device a name you want at the end of the command (e.g. cryptluks)
```sh
cryptsetup open /dev/nvme0n1p3 cryptluks
```
The decrypted container is now available at `/dev/mapper/cryptluks`

## Create Volumes

### Create Physical Volume
```sh
pvcreate /dev/mapper/cryptluks
```

### Create Volume Group for Logical Volumes
Assign it the name you like (e.g. vgroup)
```sh
vgcreate vgroup /dev/mapper/cryptluks
```

### Create Logical Volumes
```sh
lvcreate -L 64G vgroup -n root
lvcreate -L 32G vgroup -n var
lvcreate -L 8G vgroup -n tmp
lvcreate -L 16G vgroup -n swap
lvcreate -l +100%FREE vgroup -n home
lvreduce -L -256M vgroup/home
lvdisplay
```
- `-L`: size of the volume
- `-n`: name of the volume

Leave 256MiB of free space in the volume group the e2scrub command. It requires the LVM volume group to have at least 256MiB of unallocated space to dedicate to the snapshot

### Format Logical Volumes
```sh
mkfs.ext4 -L ROOT /dev/vgroup/root
mkfs.ext4 -L VAR /dev/vgroup/var
mkfs.ext4 -L TMP /dev/vgroup/tmp
mkfs.ext4 -L HOME /dev/vgroup/home
lsblk -f
```

### Mount Filesystems
```sh
mount /dev/vgroup/root /mnt
mount --mkdir /dev/nvme0n1p1 /mnt/efi
mount --mkdir /dev/nvme0n1p2 /mnt/boot
mount --mkdir /dev/vgroup/var /mnt/var
mount --mkdir /dev/vgroup/tmp /mnt/tmp
mount --mkdir /dev/vgroup/home /mnt/home
lsblk
```

### Enable Swap space
```sh
mkswap /dev/vgroup/swap
swapon /dev/vgroup/swap
lsblk
```

## Install Base System
### Pacstrap system packages
```sh
pacstrap -K /mnt base base-devel coreutils cryptsetup dkms e2fsprogs efibootmgr grub intel-ucode linux linux-firmware linux-headers lvm2 man-db man-pages nano texinfo terminus-font tpm2-tools vim
```
> [!important] INTEL or AMD?
> Replace `intel-ucode` with `amd-ucode` if you have an AMD CPU

### Generate `/etc/fstab` file
To ensure that partitions are mounted at boot
```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

#### Make `/tmp` write to RAM
`/tmp` directory will use RAM for storage, which is faster than using disk storage, and it will be cleared on reboot, making it suitable for temporary files. Increases speed and reduce SSD wear
```sh
echo 'tmpfs /tmp tmpfs defaults,noatime,mode=1777 0 0' >> /mnt/etc/fstab
```

## Chroot into the new system

### Refresh pacman keyring
```sh
pacman-key --init && pacman-key --populate archlinux
```

## Set up the environment

#### Set your time zone
By creating a symbolic link to the appropriate timezone file
```sh
ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
```

(Optional) See available timezones:
```sh
ls /usr/share/zoneinfo/
```

#### Enable the specified locales 
Edit `/etc/locale.gen`
Uncomment desired locales (`en_US.UTF-8`)
**OR Speedrun**
```sh
sed -i 's/^#\s*en_US.UTF-8/en_US.UTF-8/' /etc/locale.gen
```

#### Generate locales
```sh
locale-gen
```

#### Update `locale.conf` file
Set the system language to English (US)
```sh
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

#### Set machine Hostname in `/etc/hostname`
Replace "_arch_" with desired hostname
```sh
echo "arch" > /etc/hostname | tee -a /etc/hostname
```

#### Add Local domain info to `/etc/hosts`
Replace "_arch_" with desired hostname
```sh
echo -e "127.0.0.1\tlocalhost\n::1\t\tlocalhost\n127.0.1.1\tarch.localdomain\tarch" | tee -a /etc/hosts
```

#### Enable pacman animation and color
```sh
sed -i -e 's/^#Color/Color/' -e '/^Color/a ILoveCandy' /etc/pacman.conf
```

## Install Additional Packages
Feel free to adjust to your needs
**System**
```sh
pacman -S btop curl fastfetch fwupd git inxi lm_sensors pipewire pipewire-audio sbctl sbsigntools sof-firmware mokutil mtools rsync timeshift
```
**Tools**
```sh
pacman -S firefox rhythmbox veracrypt vlc
```
**Connections**
```sh
pacman -S bluez bluez-utils firewalld ipset iptables-nft network-manager-applet networkmanager openssh
```
**Fonts**
```sh
pacman -S ttf-firacode-nerd ttf-dejavu ttf-liberation noto-fonts ttf-jetbrains-mono ttf-fira-code
```

## Install Desktop Environment
Options:
1. Barebones GNOME setup
```sh
pacman -S gnome-shell gdm gnome-control-center gnome-backgrounds gnome-keyring xdg-user-dirs-gtk network-manager-applet alacritty nautilus-python nautilus
```

2. Standard GNOME setup
```sh
pacman -S gdm gnome nautilus-python
```

### Set Autologin for DE
```sh
nano /etc/gdm/custom.conf
```
Add under [daemon] section
```sh
AutomaticLoginEnable=True
AutomaticLogin=iron
```

## Enable SystemD services
```sh
systemctl enable bluetooth.service gdm.service NetworkManager sshd
```

### Set up User accounts
#### Set Root password
```sh
passwd
```

#### Create a User
Replace "_user_" with a desired user name
```sh
useradd -m -G wheel user && passwd user
```

##### Edit the Sudoers 
To allow your user to use SUDO
```sh
visudo
```
Uncomment `%wheel ALL=(ALL) ALL`

### (Optional) Set persistent console font
```sh
vim /etc/vconsole.conf
```
Append to the file:
```sh
FONT=ter-128b
CONSOLEFONT=ter-128b
```

**OR Speedrun**

```sh
echo -e "FONT=ter-128b\nCONSOLEFONT=ter-128b" >> /etc/vconsole.conf && cat /etc/vconsole.conf
```

To list available console font names:
```sh
ls /usr/share/kbd/consolefonts/
```


## Configure `mkinitcpio.conf` HOOKS for **systemd** initramfs
> Make sure the [lvm2](https://archlinux.org/packages/?name=lvm2) package is [installed](https://wiki.archlinux.org/title/Install "Install")

```sh
vim /etc/mkinitcpio.conf
```

Add `systemd keyboard sd-vconsole sd-encrypt lvm2` hooks to [mkinitcpio.conf](https://wiki.archlinux.org/title/Mkinitcpio.conf "Mkinitcpio.conf")

```sh
HOOKS=(base systemd keyboard autodetect microcode modconf kms sd-vconsole block sd-encrypt lvm2 filesystems fsck shutdown)
```

**OR Speedrun** using `sed`

```sh
sed -i 's/^HOOKS=.*/HOOKS=(base systemd keyboard autodetect microcode modconf kms sd-vconsole block sd-encrypt lvm2 resume filesystems fsck shutdown)/' /etc/mkinitcpio.conf && cat /etc/mkinitcpio.conf
```

### Recreate the initramfs image
Based on the current configuration, ensuring that the necessary modules and hooks are included
```sh
mkinitcpio -P
```

## Configure GRUB
#### Install Grub with Secure Boot support
```sh
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB --modules="tpm" --disable-shim-lock
```

#### Get UUID for the Encrypted drive
Replace `/dev/nvme0n1p3` with the appropriate path to the LUKS partition
```sh
blkid -o value -s UUID /dev/nvme0n1p3
```
Copy the UUID for use in `/etc/default/grub`

#### Specify root in Grub
```sh
vim /etc/default/grub
```
Add parameters:
```sh
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash loglevel=3 audit=0"
GRUB_CMDLINE_LINUX="rd.luks.name=<UUID>=cryptluks root=/dev/vgroup/root"
GRUB_ALLOWEDSECUREBOOT_MODULES="linux tpm"
```
Replace `<UUID>` with UUID from `blkid` output

> [!important] Execution priority
> GRUB_CMDLINE_LINUX supersedes GRUB_CMDLINE_LINUX_DEFAULT in case of the emergency

### Re-generate Grub config
```sh
grub-mkconfig -o /boot/grub/grub.cfg
```

## Exit Chroot and Reboot
```sh
exit
umount -R /mnt && swapoff -a # This allows noticing any "busy" partitions, and finding the cause with fuser
reboot
```

# Enable Secure Boot
## UEFI config
1. Load into UEFI
2. Security -> Secure Boot
3. Clear existing Secure Boot keys
4. If not already, put Secure Boot into Setup Mode 
5. Save and Reboot the system

> [!Important]
> Set a strong UEFI password

## Setup using SBCTL
### Install `sbctl`
```sh
sudo pacman -S sbctl
```
### Check Secure Boot status
```sh
sbctl status
```
Should be:
- Installed:   ✘ sbctl is not installed
- Setup Mode:  ✘ Enabled
- Secure Boot: ✘ Disabled
- Vendor Keys: none

### Create Secure Boot keys
```sh
sudo sbctl create-keys
```

### Enroll custom secure boot keys 
```sh
sudo sbctl enroll-keys
```
Use `-m` key to add Microsoft keys if needed

### Confirm setup mode is disabled
```sh
sbctl status
```
Should be:
- Installed:   ✔ Sbctl is installed
- Owner GUID:  GUID string...
- Secure Boot: ✔ Disabled
- Vendor Keys: ✘ Disabled

### Sign Bootloader and Kernels before rebooting!

Sign **EFI** binaries
```sh
sudo sbctl sign -s /efi/EFI/GRUB/grubx64.efi
```

Sign **Kernel** binaries
```sh
sudo sbctl sign -s /boot/vmlinuz-linux
```

Sign **fwupd** binaries
```sh
sudo sbctl sign -s /usr/lib/fwupd/efi/fwupdx64.efi
```

Regenerate initramfs images
```sh
sudo mkinitcpio -P
```

Reboot into UEFI and turn Secure Boot back on, if it's disabled. 
If the bootloader and OS load, Secure Boot should be working. Check with:
```sh
sbctl status
```
Should be:
- Installed:      ✓ sbctl is installed
- Owner GUID:     GUID string...
- Setup Mode:     ✓ Disabled
- Secure Boot:    ✓ Enabled
- Vendor Keys:    none

# NOTES
### Backup LUKS Header
It's important for your data security in case something goes wrong and the header is damaged.
Keep the backup file in a safe place, such as a USB drive. 
```sh
sudo cryptsetup luksHeaderBackup /dev/nvme1n0p3 --header-backup-file /path/to/backup_header_file/luks-header-backup-$(date -I)
```
- Replace _/path/to/backup_header_file/_ with the actual directory where you want to save the backup file
- `$(date -I)` will append the current date in ISO format to the filename

#### (Optional) Restore header from backup
```sh
sudo cryptsetup luksHeaderRestore /dev/<your-disk-luks> --header-backup-file /path/to/backup_header_file
```



# Congratulations! 🎉
You've completed the setup
