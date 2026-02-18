# Modern Arch linux installation guide

# Table of contents

- [Introduction](#introduction)
- [Preliminary Steps](#preliminary-steps)
- [Main installation](#main-installation)
  - [Disk partitioning](#disk-partitioning)
  - [Disk formatting](#disk-formatting)
  - [Disk mounting](#disk-mounting)
  - [Packages installation](#packages-installation)
  - [Fstab](#fstab)
  - [Context switch to our new system](#context-switch-to-our-new-system)
  - [Set up the time zone](#set-up-the-time-zone)
  - [Set up the language and tty keyboard map](#set-up-the-language-and-tty-keyboard-map)
  - [Hostname and Host configuration](#hostname-and-host-configuration)
  - [Root and users](#root-and-users)
  - [Grub configuration](#grub-configuration)
  - [Unmount everything and reboot](#unmount-everything-and-reboot)
  - [Automatic snapshot boot entries update](#automatic-snapshot-boot-entries-update)
  - [Virtualbox support](#virtualbox-support)
  - [Aur helper and additional packages installation](#aur-helper-and-additional-packages-installation)
  - [Finalization](#finalization)
- [Video drivers](#video-drivers)
  - [Amd](#amd)
    - [32 Bit support](#32-bit-support)
  - [Nvidia](#nvidia)
  - [Intel](#intel)
- [Setting up a graphical environment](#setting-up-a-graphical-environment)
  - [Option 1: KDE Plasma](#option-1-kde-plasma)
  - [Option 2: Hyprland \[WIP\]](#option-2-hyprland-wip)
- [Adding a display manager](#adding-a-display-manager)
- [Gaming](#gaming)
  - [Gaming clients](#gaming-clients)
  - [Windows compatibility layers](#windows-compatibility-layers)
  - [Generic optimizations](#generic-optimizations)
  - [Overclocking and monitoring](#overclocking-and-monitoring)
- [Additional notes](#additional-notes)
- [Things to add](#things-to-add)

# Preliminary steps  

First set up your keyboard layout  

```
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
# Check if ntp is active and if the time is right
timedatectl

# In case it's not active you can do
timedatectl set-ntp true
```

<br>

# Main installation

## Disk partitioning

<br>

```Zsh
# Check the drive name. Mine is /dev/nvme0n1
# If you have an hdd is something like sdax
fdisk -l

# Invoke fdisk to partition
fdisk /dev/nvme0n1

# Now press the following commands, when i write ENTER press enter
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
ENTER # Now check if you got the partitions right

# If so write the changes
w
ENTER

# If not you can quit without saving and redo from the beginning
q
ENTER
```

<br>

## Disk formatting  


```Zsh
# Find the efi partition with fdisk -l or lsblk. For me it's /dev/nvme0n1p1 and format it.
mkfs.fat -F 32 /dev/nvme0n1p1

# Find the root partition. For me it's /dev/nvme0n1p2 and format it. I will use BTRFS.
mkfs.btrfs /dev/nvme0n1p2

# Mount the root fs to make it accessible
mount /dev/nvme0n1p2 /mnt
```

<br>

## Disk mounting

```Zsh
# Create the subvolumes, in my case I choose to make a subvolume for / and one for /home. Subvolumes are identified by prepending @
# NOTICE: the list of subvolumes will be increased in a later release of this guide, upon proper testing and judgement. See the "Things to add" chapter.
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

Edit `/etc/locale.gen` and uncomment the entries for your locales. Each entry represent a language and its formats for time, date, currency and other country related settings. By uncommenting we will mark the entry to be generated when the generate command will be issued, but note that it won't still be active. In my case I will uncomment _\( ie: remove the # \)_ `en_US.UTF-8 UTF-8` and `it_IT.UTF-8 UTF-8` because I use English as a display language and Italian for date, time and other formats.  

```Zsh
# To edit I will use vim, feel free to use nano instead.
vim /etc/locale.gen

# Now issue the generation of the locales
locale-gen
```

<br>

Since the locale is generated but still not active, we will create the configuration file `/etc/locale.conf` and set the locale to the desired one, by setting the `LANG` variable accordingly. In my case I'll write `LANG=it_IT.UTF-8` to apply Italian settings to everything and then override only the display language to English by setting \( on a new line \) `LC_MESSAGES=en_US.UTF-8`. _\( if you want formats and language to stay the same **DON'T** set `LC_MESSAGES`  \)_. More on this [here](https://wiki.archlinux.org/title/Locale#Variables)

```Zsh
touch /etc/locale.conf
vim /etc/locale.conf
```

<br>

Now to make the current keyboard layout permanent for tty sessions , create `/etc/vconsole.conf` and write `KEYMAP=your_key_map` substituting the keymap with the one previously set [here](#preliminary-steps). In my case `KEYMAP=us`

```Zsh
vim /etc/vconsole.conf
```

<br>

## Hostname and Host configuration

```Zsh
# Create /etc/hostname then choose and write the name of your pc in the first line. In my case I'll use Arch
touch /etc/hostname
vim /etc/hostname

# Create the /etc/hosts file. This is very important because it will resolve the listed hostnames locally and not over Internet DNS.
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
# You can choose a different editor than vim by changing the EDITOR variable
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
# Enable newtork manager before rebooting otherwise, you won't be able to connect
systemctl enable NetworkManager

# Exit from chroot
exit

# Unmount everything to check if the drive is busy
umount -R /mnt

# Reboot the system and unplug the installation media
reboot

# Now you'll be presented at the terminal. Log in with your user account, for me its "mjkstra".

# Enable and start the time synchronization service
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

## Virtualbox support  

Follow these steps if you are running Arch on a Virtualbox VM.
This will enable features such as **clipboard sharing**, **shared folders** and **screen resolution tweaks**

```Zsh
# Install the guest utils
pacman -S virtualbox-guest-utils

# Enable this service to automatically load the kernel modules
systemctl enable vboxservice.service
```

> Note: the utils will only work after a reboot is performed.

> Warning: the utils seems to only work in a graphical environment.

<br>

## Aur helper and additional packages installation  

To gain access to the arch user repository we need an aur helper, I will choose yay which also works as a pacman wrapper \( which means you can use yay instead of pacman \). Yay has a CLI, but if you later want to have an aur helper with a GUI you can install [`pamac`](https://gitlab.manjaro.org/applications/pamac) \( a Manjaro software, so use at your own risk \), **however** note that front\-ends like `pamac` and also any store \( KDE discovery, Ubuntu store etc. \) are not officially supported and should be avoided, because of the high risk of performing [partial upgrades](https://wiki.archlinux.org/title/System_maintenance#Partial_upgrades_are_unsupported). This is also why later when installing KDE, I will exclude the KDE discovery store from the list of packages.  

To learn more about yay read [here](https://github.com/Jguer/yay#yay)  

> Note: you can't execute makepkg as root, so you need to log in your main account. For me it's mjkstra

```Zsh
# Install yay
sudo pacman -S --needed git base-devel && git clone https://aur.archlinux.org/yay.git && cd yay && makepkg -si

# Install "timeshift-autosnap", a configurable pacman hook which automatically makes snapshots before pacman upgrades.
yay -S timeshift-autosnap
```

> Learn more about timeshift autosnap [here](https://gitlab.com/gobonja/timeshift-autosnap)

<br>

## Finalization

```Zsh
# To complete the main/basic installation reboot the system
reboot
```

> After these steps you **should** be able to boot on your newly installed Arch Linux, if so congrats !  

> The basic installation is complete and you could stop here, but if you want to to have a graphical session, you can continue reading the guide.

<br>

# Video drivers

In order to have the smoothest experience on a graphical environment, **Gaming included**, we first need to install video drivers. To help you choose which one you want or need, read [this section](https://wiki.archlinux.org/title/Xorg#Driver_installation) of the arch wiki.  

> Note: skip this section if you are on a Virtual Machine

<br>

## Amd  

For this guide I'll install the [**AMDGPU** driver](https://wiki.archlinux.org/title/AMDGPU) which is the open source one and the recommended, but be aware that this works starting from the **GCN 3** architecture, which means that cards **before** RX 400 series are not supported. _\( I have an RX 5700 XT \)_  

```Zsh

# What are we installing ?
# mesa: DRI driver for 3D acceleration.
# xf86-video-amdgpu: DDX driver for 2D acceleration in Xorg. I won't install this, because I prefer the default kernel modesetting driver.
# vulkan-radeon: vulkan support.
# libva-mesa-driver: VA-API h/w video decoding support.
# mesa-vdpau: VDPAU h/w accelerated video decoding support.

sudo pacman -S mesa vulkan-radeon libva-mesa-driver mesa-vdpau
```

### 32 Bit support

If you want to add **32-bit** support, we need to enable the `multilib` repository on pacman: edit `/etc/pacman.conf` and uncomment the `[multilib]` section _\( ie: remove the hashtag from each line of the section. Should be 2 lines \)_. Now we can install the additional packages.

```Zsh
# Refresh and upgrade the system
yay

# Install 32bit support for mesa, vulkan, VA-API and VDPAU
sudo pacman -S lib32-mesa lib32-vulkan-radeon lib32-libva-mesa-driver lib32-mesa-vdpau
```

<br>

## Nvidia  

In summary if you have an Nvidia card you have 2 options:  

1. [**NVIDIA** proprietary driver](https://wiki.archlinux.org/title/NVIDIA)
2. [**Nouveau** open source driver](https://wiki.archlinux.org/title/Nouveau)

The recommended is the proprietary one, however I won't explain further because I don't have an Nvidia card and the process for such cards is tricky unlike for AMD or Intel cards. Moreover for reason said before, I can't even test it.

<br>

## Intel

Installation looks almost identical to the AMD one, but every time a package contains the `radeon` word substitute it with `intel`. However this does not stand for [h/w accelerated decoding](https://wiki.archlinux.org/title/Hardware_video_acceleration), and to be fair I would recommend reading [the wiki](https://wiki.archlinux.org/title/Intel_graphics#Installation) before doing anything.

<br>

# Setting up a graphical environment

I'll provide 2 options:  

1. **KDE-plasma**  
2. **Hyprland**

On top of that I'll add a **display manager**, which you can omit if you don't like ( if so, you have additional configuration steps to perform ).  

<br>

## Option 1: KDE-plasma

**KDE Plasma** is a very popular DE which comes bundled in many distributions. It supports both the older **Xorg** and the newer **Wayland** protocols. It's **user friendly**, **light** and it's also used on the Steam Deck, which makes it great for **gaming**. I'll provide the steps for a minimal installation and add some basic packages.

```Zsh
# plasma-desktop: the barebones plasma environment.
# plasma-pa: the KDE audio applet.
# plasma-nm: the KDE network applet.
# plasma-systemmonitor: the KDE task manager.
# plasma-firewall: the KDE firewall.
# plasma-browser-integration: cool stuff, it lets you manage things from your browser like media currently played via the plasma environment. Make sure to install the related extension on firefox ( you will be prompted automatically upon boot ).
# kscreen: the KDE display configurator.
# kwalletmanager: manage secure vaults ( needed to store the passwords of local applications in an encrypted format ). This also installs kwallet as a dependency, so I don't need to specify it.
# kwallet-pam: automatically unlocks secure vault upon login ( without this, each time the wallet gets queried it asks for your password to unlock it ).
# bluedevil: the KDE bluetooth manager.
# powerdevil: the KDE power manager.
# power-profiles-daemon: adds 3 power profiles selectable from powerdevil ( power saving, balanced, performance ). Make sure that its service is enabled and running ( it should be ).
# kdeplasma-addons: some useful addons.
# xdg-desktop-portal-kde: better integrates the plasma desktop in various windows like file pickers.
# xwaylandvideobridge: exposes Wayland windows to XWayland-using screen sharing apps ( useful when screen sharing on discord, but also in other instances ).
# kde-gtk-config: the native settings integration to manage GTK theming.
# breeze-gtk: the breeze GTK theme.
# cups, print-manager: the CUPS print service and the KDE front-end.
# konsole: the KDE terminal.
# dolphin: the KDE file manager.
# ffmpegthumbs: video thumbnailer for dolphin.
# firefox: the web browser.
# kate: the KDE text editor.
# okular: the KDE pdf viewer.
# gwenview: the KDE image viewer.
# ark: the KDE archive manager.
# pinta: a paint.net clone written in GTK.
# spectacle: the KDE screenshot tool.
# dragon: a simple KDE media player. A more advanced alternative based on libmpv is Haruna.
sudo pacman -S plasma-desktop plasma-pa plasma-nm plasma-systemmonitor plasma-firewall plasma-browser-integration kscreen kwalletmanager kwallet-pam bluedevil powerdevil power-profiles-daemon kdeplasma-addons xdg-desktop-portal-kde xwaylandvideobridge kde-gtk-config breeze-gtk cups print-manager konsole dolphin ffmpegthumbs firefox kate okular gwenview ark pinta spectacle dragon
```

Now don't reboot your system yet. If you want a display manager, which is generally recommended, head to the [related section](#adding-a-display-manager) in this guide and proceed from there otherwise you'll have to [manually configure](https://wiki.archlinux.org/title/KDE#From_the_console) and launch the graphical environment each time \(which I would advise to avoid\).

<br>

## Option 2: Hyprland [WIP]  

> Note: this section needs configuration and is basically empty, I don't know when and if I will expand it but at least you have a starting point.

<br>

**Hyprland** is a **tiling WM** that sticks to the wayland protocol. It looks incredible and it's one of the best Wayland WMs right now. It's based on **wlroots** the famous library used by Sway, the most mature Wayland WM there is. I don't know if I would recommend this to beginners because it's a totally different experience and it may not be better. Moreover it requires you to read the [wiki](https://wiki.hyprland.org/) for configuration but it also features a [master tutorial](https://wiki.hyprland.org/Getting-Started/Master-Tutorial). The good part is that even if it seems discouraging, it's actually an easy read because it is written beautifully.  

```Zsh
# Install hyprland from tagged releases and other utils:
# swaylock: the lockscreen
# wofi: the wayland version of rofi, an application launcher, extremely configurable
# waybar: a status bar for wayland wm's
# dolphin: a powerful file manager from KDE applications
# alacritty: a beautiful and minimal terminal application, super configurable
pacman -S --needed hyprland swaylock wofi waybar dolphin alacritty

# wlogout: a logout/shutdown menu
yay -S wlogout
```

<br>

# Adding a display manager

**Display managers** are useful when you have multiple DE or WMs and want to choose where to boot from or select the display protocol \( Wayland or Xorg \) in a GUI fashion, also they take care of the launch process. I'll show the installation process of **SDDM**, which is highly customizable and compatible.

> Note: hyprland does not support any display manager, however SDDM is reported to work flawlessly from the [wiki](https://wiki.hyprland.org/Getting-Started/Master-Tutorial/#launching-hyprland)

```Zsh
# Install SDDM
sudo pacman -S sddm

# Enable SDDM service to make it start on boot
sudo systemctl enable sddm

# If using KDE I suggest installing this to control the SDDM configuration from the KDE settings App
pacman -S --needed sddm-kcm

# Now it's time to reboot the system
reboot
```

<br>

# Gaming

Gaming on linux has become a very fluid experience, so I'll give some tips on how to setup your arch distro for gaming.  
Before going further I'll assume that you have installed the video drivers, also make sure to install with pacman, if you haven't done it already: `lib32-mesa`, `lib32-vulkan-radeon` and additionally `lib32-pipewire` \( Note that the `multilib` repository must be enabled, [here](#32-bit-support) I've explained how to do it ).

Let's break down what is needed to game:  

1. **Gaming client** ( eg: Steam, Lutris, Bottles, etc..)
2. **Windows compatibility layers** ( eg: Proton, Wine, DXVK, VKD3D )

Optionally we can have:  

1. **Generic optimization** ( eg: gamemode )
2. **Overclocking and monitoring software** ( eg: CoreCtrl, Mangohud )
3. **Custom kernels**

<br>

## Gaming clients  

I'll install **Steam** and to access games from other launchers I'll use **Bottles**, which should be installed through **flatpak**.

```Zsh
# Install steam and flatpak
sudo pacman -S steam flatpak

# Install bottles through flatpak
flatpak install flathub com.usebottles.bottles
```

<br>

## Windows compatibility layers

Proton is the compatibility layer developed by Valve, which includes **DXVK**( DirectX 9-10-11 to Vulkan), **VKD3D** ( DirectX 12 to Vulkan ) and a custom version of **Wine**. It is embedded in Steam and can be enabled for **non** native games direclty in Steam: `Steam > Settings > Compatibility > Enable Steam Play for all other titles`. A custom version of proton, **Proton GE** exists and can be used as an alternative if something is broken or doesn't perform as expected. Can be either [downloaded manually](https://github.com/GloriousEggroll/proton-ge-custom#installation) or through yay as below.  

```Zsh
# Installation through yay
yay -S proton-ge-custom-bin
```

<br>

## Generic optimizations

We can use gamemode to gain extra performance. To enable it read [here](https://github.com/FeralInteractive/gamemode#requesting-gamemode)

```Zsh
# Install gamemode
sudo pacman -S gamemode
```

<br>

## Overclocking and monitoring

To live monitor your in-game performance, you can use **mangohud**. To enable it read [here](https://github.com/flightlessmango/MangoHud#normal-usage).  

In order to easily configure mangohud, I'll use **Goverlay**.

```Zsh
# Install goverlay which includes mangohud as a dependency
sudo pacman -S goverlay
```

To overclock your system, i suggest installing [**corectrl**](https://gitlab.com/corectrl/corectrl) if you have an AMD Gpu or [**TuxClocker**](https://github.com/Lurkki14/tuxclocker) for NVIDIA.

<br>

# Additional notes

- On KDE disabling mouse acceleration is simple, just go to the settings via the GUI and on the mouse section enable the flat acceleration profile. If not using KDE then read [here](https://wiki.archlinux.org/title/Mouse_acceleration)

- To enable Freesync or Gsync you can read [here](https://wiki.archlinux.org/title/Variable_refresh_rate), depending on your session \( Wayland or Xorg \) and your gfx provider \( Nvidia, AMD, Intel \) the steps may differ. On a KDE wayland session, you can directly enable it from the monitor settings under the name of **adaptive sync** 

- Some considerations if you are thinking about switching to a custom kernel:
  - You have to manually recompile it each time there is a new update unless you use a precompiled kernel from pacman or aur such as `linux-zen`.
  - There is no such thing as the best kernel, all kernels make tradeoffs \( eg: latency for throughtput \) and this it why it's generally advised to stick with the generic one.
  - If you are mainly a gamer you MAY consider the **TKG** or **CachyOS** kernel. These kernels contain many optimizations and are highly customizable, however the TKG kernel has to be compiled \( mainly it's time consuming, not hard \), while CachyOS kernel comes already packaged and optimized for specific hardware configurations, and can be simply installed with pacman upon adding their repos to `pacman.conf`. Some users have reported to experience a smoother experience with lower latency, however I couldn't find consistent information about this and it seems that is all backed by a personal sensation and not a result obtained through objective measurements \( Also this is difficult because in Linux there can be countless configuration variables and it also depends by the graphic card being used \). 
    
- Some recommended reads:
  - [How to build your KDE environment](https://community.kde.org/Distributions/Packaging_Recommendations)
  - [Gaming on Wayland](https://zamundaaa.github.io/wayland/2021/12/14/about-gaming-on-wayland.html) \( old article but still relevant \)
  - [Improving Linux Gaming Performance](https://linux-gaming.kwindu.eu/index.php?title=Improving_performance)
  - [Arch linux system maintenance](https://wiki.archlinux.org/title/System_maintenance)
  - [Arch linux general recommendations](https://wiki.archlinux.org/title/General_recommendations)
<br>

# Things to add

1. Additional pacman configuration \( paccache, colors, download packages simultaneously \)
2. Reflector configuration
3. Snapper: a more advanced snapshot program as a timeshift alternative.
4. Overhaul the subvolumes partitioning into a richer set including @log @cache @tmp @snapshots. This way they they won't be included when snapshotting the root subvolume ( ie: @ ).
5. Better fstab structure

<br>
