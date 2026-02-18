# Modified Arch Installation Guide

# Preliminary steps  

First set up your keyboard layout  
```
loadkeys us
```
<br>

Check that we are in UEFI mode  
```Zsh
# If this command prints 64 or 32 then you are in UEFI
cat /sys/firmware/efi/fw_platform_size
```
<br>

Setup the internet connection
```
# start the service
systemctl start iwd 

# when in iwctl
[iwd]# device list

# device name is most likely wlan0; scan for avail networks 
[iwd]# station wlan0 scan 
[iwd]# station wlan0 get-networks 
[iwd]# station wlan0 connect <network_name> 
```

Check the internet connection  

```Zsh
ping -c 5 archlinux.org 
```

<br>

Check the system clock

```Zsh
# check if ntp is active and if the time is right
timedatectl

# if not active
timedatectl set-ntp true
```

<br>

# Main installation
There are two partitions:  
| Number | Type | Size |
| --- | --- | --- |
| 1 | EFI | 512 Mb |
| 2 | Linux Filesystem | 999.5Gb \(all of the remaining space \) |  

## Disk partitioning

<br>

```Zsh
# check drive name (ex. /dev/nvme0n1)
fdisk -l

# invoke fdisk to partition
fdisk /dev/nvme0n1

# now press the following commands, when i write ENTER press enter
g
ENTER
n
ENTER
ENTER
ENTER
+1G
ENTER
t
ENTER
ENTER
1
ENTER
n
ENTER
ENTER
+1T
p
ENTER # check if you got the partitions right

# if so write the changes
w
ENTER

# if not you can quit without saving and redo from the beginning
q
ENTER
```
<br>

## Disk formatting  

```Zsh
# find the efi partition with fdisk -l or lsblk
mkfs.fat -F 32 /dev/nvme0n1p1

# find the root partition and format it
mkfs.btrfs /dev/nvme0n1p2

# Mount the root fs to make it accessible
mount /dev/nvme0n1p2 /mnt
```

<br>

## Disk mounting

```Zsh
# Create the subvolumes. Subvolumes are identified by prepending @
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@swap 

# Unmount the root fs
umount /mnt
```
<br>

For this guide I'll compress the btrfs subvolumes with **Zstd**, which has proven to be [a good algorithm among the choices](https://www.phoronix.com/review/btrfs-zstd-compress)  

```Zsh
# Mount the root and home subvolume. If you don't want compression just remove the compress option.
mount -o compress=zstd,subvol=@ /dev/nvme0n1p2 /mnt
mkdir -p /mnt/home
mount -o compress=zstd,subvol=@home /dev/nvme0n1p2 /mnt/home
mkdir -p /mnt/swap 
mount -o compress=zstd,subvol=@swap /dev/nvme0n1p2 /mnt/swap

# create the swapfile 
btrfs filesystem mkswapfile --size 4g --uuid clear /swap/swapfile 
swapon /swap/swapfile 
```
<br>

Now we have to mount the efi partition. In general there are 2 main mountpoints to use: `/efi` or `/boot` but in this configuration i am **forced** to use `/efi`, because by choosing `/boot` we could experience a **system crash** when trying to restore `@` _\( the root subvolume \)_ to a previous state after kernel updates. This happens because `/boot` files such as the kernel won't reside on `@` but on the efi partition and hence they can't be saved when snapshotting `@`. Also this choice grants separation of concerns and also is good if one wants to encrypt `/boot`, since you can't encrypt efi files. Learn more [here](https://wiki.archlinux.org/title/EFI_system_partition#Typical_mount_points)

```Zsh
mkdir -p /mnt/efi
mount /dev/nvme0n1p1 /mnt/efi
```
<br>

## Packages installation  

```Zsh
# This will install some packages to "bootstrap" methaphorically our system. Feel free to add the ones you want
# "base, linux, linux-firmware" are needed. If you want a more stable kernel, then swap linux with linux-lts
pacstrap -K /mnt base base-devel linux linux-firmware git btrfs-progs grub efibootmgr grub-btrfs inotify-tools timeshift vim networkmanager pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber reflector zsh zsh-completions zsh-autosuggestions openssh man sudo iwctl intel-ucode 
```
<br>

## Fstab  

```Zsh
# Fetch the disk mounting points as they are now ( we mounted everything before ) and generate instructions to let the system know how to mount the various disks automatically
genfstab -U /mnt >> /mnt/etc/fstab

# Check if fstab is fine ( it is if you've faithfully followed the previous steps )
cat /mnt/etc/fstab
```
<br>

## Context switch to our new system  

```Zsh
# To access our new system we chroot into it
arch-chroot /mnt
```
<br>

## Set up the time zone

```Zsh
# In our new system we have to set up the local time zone, find your one in /usr/share/zoneinfo mine is /usr/share/zoneinfo/Europe/Rome and create a symbolic link to /etc/localtime
ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime

# Now sync the system time to the hardware clock
hwclock --systohc
```

<br>

## Set up the language and tty keyboard map

Edit `/etc/locale.gen` and uncomment the entries for your locales. We will uncomment _\( ie: remove the # \)_ for `en_US.UTF-8 UTF-8`

```Zsh
vim /etc/locale.gen

# after uncommenting, regenerate the locales
locale-gen
```
<br>

Since the locale is generated but still not active, we will create the configuration file `/etc/locale.conf` and set the locale to the desired one by setting the `LANG` variable accordingly. We will write `LANG=en_US.UTF-8` in `locale.conf`. More on this [here](https://wiki.archlinux.org/title/Locale#Variables)

```Zsh
touch /etc/locale.conf
vim /etc/locale.conf
```
<br>

Now to make the current keyboard layout permanent for tty sessions , create `/etc/vconsole.conf` and write `KEYMAP=us`

```Zsh
vim /etc/vconsole.conf
```
<br>

## Hostname and Host configuration

```Zsh
# create /etc/hostname then choose and write the name of your pc in the first line. In this example, we will use `Arch`
touch /etc/hostname
vim /etc/hostname

# create the /etc/hosts file. This is very important because it will resolve the listed hostnames locally and not over Internet DNS.
touch /etc/hosts
```

Write the following ip, hostname pairs inside /etc/hosts, replacing `Arch` with **YOUR** hostname:

```
127.0.0.1 localhost
::1 localhost
127.0.1.1 Arch
```

```Zsh
# Edit the file with the information above
vim /etc/hosts
```
<br>

## Root and users  

```Zsh
# Set up the root password
passwd

# Add a new user, in my case mjkstra.
# -m creates the home dir automatically
# -G adds the user to an initial list of groups, in this case wheel, the administration group. If you are on a Virtualbox VM and would like to enable shared folders between host and guest machine, then also add the group vboxsf besides wheel.
useradd -mG wheel mjkstra
passwd mjkstra

# The command below is a one line command that will open the /etc/sudoers file with your favourite editor.
# Once opened, you have to look for a line which says something like "Uncomment to let members of group wheel execute any action"
# and uncomment exactly the line BELOW it, by removing the #. This will grant superuser priviledges to your user.
EDITOR=vim visudo
```
<br>

## Grub configuration  

Now I'll [deploy grub](https://wiki.archlinux.org/title/GRUB#Installation)  

```Zsh
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB  
```
<br>

Generate the grub configuration ( it will include the microcode installed with pacstrap earlier )  

```Zsh
grub-mkconfig -o /boot/grub/grub.cfg
```
<br>

## Unmount everything and reboot 

```Zsh
# enable newtork manager before rebooting otherwise, you won't be able to connect
systemctl enable NetworkManager

# exit from chroot
exit

# unmount everything to check if the drive is busy
umount -R /mnt

# reboot the system and unplug the installation media
reboot

# now you'll be presented at the terminal. Log in with your user account, for me its "mjkstra".

# enable and start the time synchronization service
timedatectl set-ntp true

# connect to the internet using nmcli 
sudo nmcli wlan0 connect <network_name> password <password>
```
<br>

## Automatic snapshot boot entries update  

Each time a system snapshot is taken with timeshift, it will be available for boot in the bootloader, however you need to manually regenerate the grub configuration, this can be avoided thanks to `grub-btrfs`, which can automatically update the grub boot entries.  

Edit the **`grub-btrfsd`** service and because I will rely on timeshift for snapshotting, I am going to replace `ExecStart=...` with `ExecStart=/usr/bin/grub-btrfsd --syslog --timeshift-auto`. If you don't use timeshift or prefer to manually update the entries then lookup [here](https://github.com/Antynea/grub-btrfs)  

```Zsh 
sudo systemctl edit --full grub-btrfsd

# Enable grub-btrfsd service to run on boot
sudo systemctl enable grub-btrfsd
```

<br>

## Aur helper and additional packages installation  

To gain access to the arch user repository we need an aur helper, I will choose yay which also works as a pacman wrapper \( which means you can use yay instead of pacman \). 

To learn more about yay read [here](https://github.com/Jguer/yay#yay)  

```Zsh
# install yay
sudo pacman -S --needed git base-devel && git clone https://aur.archlinux.org/yay.git && cd yay && makepkg -si

# install "timeshift-autosnap", a configurable pacman hook which automatically makes snapshots before pacman upgrades.
yay -S timeshift-autosnap
```
> Learn more about timeshift autosnap [here](https://gitlab.com/gobonja/timeshift-autosnap)
<br>

## Finalization

```Zsh
# to complete the main/basic installation reboot the system
reboot
```
<br>

# Video drivers

In order to have the smoothest experience on a graphical environment, **Gaming included**, we first need to install video drivers. To help you choose which one you want or need, read [this section](https://wiki.archlinux.org/title/Xorg#Driver_installation) of the arch wiki.  
<br>

```
sudo pacman -S mesa vulkan-intel libva-mesa-driver mesa-vdpau
```
<br>

# Setting up a graphical environment

## HyDE

Download the repo here: [https://github.com/HyDE-Project/HyDE](https://github.com/HyDE-Project/HyDE) and follow accordingly

# Notes
- need to look more into the swapfile / swap partition portion
- is yay necessary if HyDE comes with the chaotic aur helper
- did i include all the wifi steps cuz idk if i did
