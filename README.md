# 🐧ARCH + 🗃️LVM + 🔐LUKS
# DESCRIPTION
This setup guide is for minimal Arch system with following features:
- GNOME desktop environment
- LVM partition
- LUKS encryption
- Secure Boot

Arch will be installed on bare metal using the entire space of the drive.

# PREPARATION
1. Download the latest ISO from [archlinux.org](https://archlinux.org/download/)
2. Transfer the ISO to the USB drive using [Rufus](https://rufus.ie/en/) or [Ventoy](https://www.ventoy.net/en/index.html)
3. Boot your system from the USB drive into Arch live environment


### Set Comfortable Font
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

We'll use it to SSH into the system

```sh
ip a
```

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

### Sync system clock

```sh
timedatectl set-ntp true
```

## Partition the drive for LVM on LUKS

### Check the name of the drive
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

Give it a name you want at the end of the command (e.g. cryptluks)

```sh
cryptsetup open /dev/nvme0n1p3 cryptluks
```

The decrypted container is now available at `/dev/mapper/cryptluks`

### Backup LUKS Header

It's important for your data security.
Keep the backup file in a safe place, such as a USB drive. 

```sh
cryptsetup luksHeaderBackup /dev/nvme1n0p3 --header-backup-file /path/to/backup_header_file
```

#### (Optional) Restore header from backup

```sh
sudo cryptsetup luksHeaderRestore /dev/<your-disk-luks> --header-backup-file /path/to/backup_header_file
```

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
> Replace `intel-ucode` with `amd-ucode` if you have and AMD CPU

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
System
```sh
pacman -S btop curl fastfetch fwupd git inxi lm_sensors pipewire pipewire-audio sbctl sbsigntools sof-firmware mokutil mtools rsync timeshift
```
Tools
```sh
pacman -S firefox rhythmbox veracrypt vlc
```
Connections
```sh
pacman -S bluez bluez-utils firewalld ipset iptables-nft network-manager-applet networkmanager openssh
```
Fonts
```sh
pacman -S ttf-firacode-nerd ttf-dejavu ttf-liberation noto-fonts ttf-jetbrains-mono ttf-fira-code
```

## Install Desktop Environment

- Barebones GNOME setup

```sh
pacman -S gnome-shell gdm gnome-control-center gnome-backgrounds gnome-keyring xdg-user-dirs-gtk network-manager-applet alacritty nautilus-python nautilus
```

- Standard GNOME setup

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

## Enable services
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

List available console font names `ls /usr/share/kbd/consolefonts/`

```sh
vim /etc/vconsole.conf
# Place in the file
FONT=ter-128b
CONSOLEFONT=ter-128b
```

**OR Speedrun**

```sh
echo -e "FONT=ter-128b\nCONSOLEFONT=ter-128b" >> /etc/vconsole.conf && cat /etc/vconsole.conf
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

Add parameters.

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


# Congratulations! 🎉
You've completed the setup
