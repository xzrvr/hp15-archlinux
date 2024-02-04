# hp15-archlinux
A guide to install Arch Linux on a old AMD laptop I own.

# Table of contents
wip

# Arch Linux on Asus ROG Zephyrus G15 (GA502IV)
Guide to install Arch Linux on a HP 15-db0026au


# Basic Install

## Prepare and Booting ISO

Boot Arch Linux using a prepared USB stick. [Rufus](https://rufus.ie/en/) can be used on windows, [Etcher](https://www.balena.io/etcher/) can be used on Windows or Linux. Or use Ventoy

## Networking

For Network i use wireless, if you need wired please check the [Arch WiKi](https://wiki.archlinux.org/index.php/Network_configuration).  it should work automatically for wired.

Launch `iwctl` and connect to your AP like this:
* `station wlan0 scan`
* `station wlan0 get-networks`
* `station wlan0 connect YOURSSID` 

Type `exit` to leave.

Update System clock with `timedatectl set-ntp true`

## Format Disk (Using gdisk)

* My Disk is `sda`, check with `lsblk`
* Format Disk using `gdisk /dev/sda` with this simple layout:

	* `o` for new partition table
	* `n,1,<ENTER>,+1024M,ef00` for EFI Boot
	* `n,2,<ENTER>,<ENTER>,8300` for the linux partition
	* `w` to save layout


Format the EFI Partition

`mkfs.vfat -F 32 -n EFI /dev/sda1`

## Create and Mount btrfs Subvolumes

`mkfs.btrfs -f -L ROOTFS /dev/sda2` btrfs filesystem for root partition


Mount Partitions und create Subvol for btrfs.

* `mount -t btrfs LABEL=ROOTFS /mnt` Mount root filesystem to /mnt
* `btrfs sub create /mnt/@`
* `btrfs sub create /mnt/@home`
* `btrfs sub create /mnt/@swap`

## Create a btrfs swapfile and remount subvols

```
truncate -s 0 /mnt/@swap/swapfile
chattr +C /mnt/@swap/swapfile
btrfs property set /mnt/@swap/swapfile compression none
fallocate -l ${SWAP_SIZE} /mnt/@swap/swapfile
chmod 600 /mnt/@swap/swapfile
mkswap /mnt/@swap/swapfile
mkdir /mnt/@/swap

```

if u want to increase or decrease swap size, just change `--size 16g` to `--size (size-in-gb's)g`
make sure that it is `g` not `gb`

Just unmount with `umount /mnt/` and remount with subvolumes

```
mount -o noatime,compress=zstd,space_cache,commit=120,subvol=@ /dev/mapper/luks /mnt
mkdir -p /mnt/boot
mkdir -p /mnt/home
mkdir -p /mnt/.snapshots
mkdir -p /mnt/btrfs

mount -o noatime,compress=zstd,space_cache,commit=120,subvol=@home /dev/mapper/luks /mnt/home/
mount -o noatime,compress=zstd,space_cache,commit=120,subvol=@snapshots /dev/mapper/luks /mnt/.snapshots/
mount -o noatime,space_cache,commit=120,subvol=@swap /dev/mapper/luks /mnt/swap/

mount /dev/nvme0n1p1 /mnt/boot/

mount -o noatime,compress=zstd,space_cache,commit=120,subvolid=5 /dev/mapper/luks /mnt/btrfs/
```


Check mountpoints with `df -Th` 

## Install the system using pacstrap

```
pacstrap /mnt base base-devel linux linux-firmware btrfs-progs nano networkmanager amd-ucode
```

After this, generate the filesystem table using 
`genfstab -Lp /mnt >> /mnt/etc/fstab` 

Add swapfile 
`echo "/swap/swapfile none swap defaults 0 0" >> /mnt/etc/fstab `

## Chroot into the new system and change language settings
You can use a hostname of your choice, I have gone with "zephyrus"
```
arch-chroot /mnt
echo zephyrus > /etc/hostname
echo LANG=en_NZ.UTF-8 > /etc/locale.conf
echo LANGUAGE=en_NZ >> /etc/locale.conf
ln -sf /usr/share/zoneinfo/Pacific/Auckland /etc/localtime
hwclock --systohc
```

Modify `nano /etc/hosts` with these entries. For static IPs, remove 127.0.1.1. Replace "zephyrus" with your hostname that you chosen previously.

```
127.0.0.1		localhost
::1				localhost
127.0.1.1		zephyrus.localdomain	zephyrus
```

`nano /etc/locale.gen` to uncomment the following line

```
en_US.UTF-8
```
Execute `locale-gen` to create the locales now

Add a password for root using `passwd root`


## Add btrfs and encrypt to Initramfs

`nano /etc/mkinitcpio.conf` and add `encrypt btrfs` to hooks between block/filesystems. **NOTE:** If you chose to not encrypt your home partition, do not add `encrypt` to `HOOKS`. and put `resume` udev after filesystems/keyboard and before fsck if u want hibernation

`HOOKS="base udev autodetect modconf block encrypt btrfs filesystems keyboard resume fsck `

Also include `amdgpu` in the MODULES section
`MODULES=(amdgpu)`

create Initramfs using `mkinitcpio -P`

## Install Systemd Bootloader

`bootctl --path=/boot install` installs bootloader

`nano /boot/loader/loader.conf` delete everything and add these few lines and save the file

```
default	arch.conf
timeout	3
editor	0
```

` nano /boot/loader/entries/arch.conf` with these lines and save. 

```
title	Arch Linux
linux	/vmlinuz-linux
initrd	/amd-ucode.img
initrd	/initramfs-linux.img
```

copy boot-options with
` echo "options	cryptdevice=UUID=$(blkid -s UUID -o value /dev/nvme0n1p2):luks root=/dev/mapper/luks rootflags=subvol=@ rw" >> /boot/loader/entries/arch.conf` 

**NOTE:** If you chose to not encrypt your home partition, use this command:

`ROOT_PARTITION=<!!!YOUR_ROOT_PARTITION_HERE!!!> && echo "options root=UUID=$(blkid -s UUID -o value ${ROOT_PARTITION}) rootflags=subvol=@ rw" >> /boot/loader/entries/arch.conf`

## Set nvidia-nouveau onto blacklist 

using `nano /etc/modprobe.d/blacklist-nvidia-nouveau.conf` with these lines and save file

```
	blacklist nouveau
	options nouveau modeset=0
```

## Leave Chroot and Reboot to installed

Type `exit` to exit chroot

`umount -R /mnt/` to unmount all volumes

Now its time to `reboot` into the new system!

# Finetuning after first Reboot

## Enable Networkmanager

Configure WiFi Connection.

```
systemctl enable NetworkManager
systemctl start NetworkManager
nmcli device wifi connect YOURSSID password SSIDPASSWORD
```
you can also use nmtui for a tui instead of cli

## Create a new user 

First create my new local user and point it to zsh

```
useradd -m -g users -G wheel,lp,power,audio -s /bin/bash MYNEWUSERNAME
passwd MYNEWUSERNAME
```

Edit `nano /etc/sudoers` and uncomment `%wheel ALL=(ALL:ALL) ALL` or `%wheel ALL=(ALL) ALL`

Now `exit` and relogin with the new MYNEWUSERNAME

## Update your system
The first thing you should do after installing is to update your system. Open a command line and run:

`sudo pacman -Syyu`

Install some daemons before we reboot

```
sudo pacman -S acpid dbus 
sudo systemctl enable acpid
```

# Setup automatic Snapshots for Pacman

The goal is to have automatic snapshots each time i made changes with pacman. The hook creates a snapshot
to ".snapshots\STABLE". So if something goes wrong i can boot from this snapshot and rollback my system.


### Create the STABLE snapshot and modify Bootloader

First i create the snapshot and changes by hand to test if anythink is working. After that it will be done automaticly by our hook and script.

```bash
sudo -i
btrfs sub snap / /.snapshots/STABLE
cp /boot/vmlinuz-linux /boot/vmlinuz-linux-stable
cp /boot/amd-ucode.img /boot/amd-ucode-stable.img
cp /boot/initramfs-linux.img /boot/initramfs-linux-stable.img
cp /boot/loader/entries/arch.conf /boot/loader/entries/stable.conf
```

Edit `/boot/loader/entries/stable.conf` to boot from STABLE snapshot

```bash
title   Arch Linux Stable
linux   /vmlinuz-linux-stable
initrd  /amd-ucode-stable.img
initrd  /initramfs-linux-stable.img
options ... rootflags=subvol=@snapshots/STABLE rw
```

Now edit the `/.snapshots/STABLE/etc/fstab` to change the root to the new snapshot/STABLE

ˋˋˋ
...
LABEL=ROOTFS  /  btrfs  rw,noatime,.....subvol=@snapshots/STABLE
...
ˋˋˋ


reboot and test if you can boot from the stable snapshot.

### Script for auto-snapshots

Copy the script from my Repo in /usr/bin/autosnap to

`/usr/bin/autosnap` and make it executable with `chmod +x /usr/bin/autosnap`

### Hook for pacman

Copy the script from my Repo in /etc/pacman.d/hooks/00-autosnap.hook to

`/etc/pacman.d/hooks/00-autosnap.hook`

Now each time pacman executes, it launches the `autosnap`script which takes a snapshot from the current system.


# Install Xfce4 or kde Desktop Environment

## Get X.Org and Xfce4

Install xorg and xfce4 packages

```bash
sudo pacman -Sy xorg xfce4 xfce4-goodies xf86-input-synaptics gvfs xdg-user-dirs ttf-dejavu network-manager-applet firefox-i18n-de git git-lfs curl wget

sudo localectl set-x11-keymap de pc105 deadgraveacute
xdg-user-dirs-update
```

LightDM Loginmanager

```bash
sudo pacman -S lightdm lightdm-gtk-greeter lightdm-gtk-greeter-settings
sudo systemctl enable lightdm
```
if you want to use kde instead, go [here](#kde-stuff)

## Fixing Audio on Linux
Audio was exceptionally low on linux. To fix, first remove everything pulseaudio related by running:
```
sudo pacman -Rdd pulseaudio pulseaudio-alsa pulseaudio-bluetooth pulseaudio-ctl pulseaudio-equalizer pulseaudio-jack pulseaudio-lirc pulseaudio-rtp pulseaudio-zeroconf pulseaudio-equalizer-ladspa
```
Most of this may not be installed already so remove it from the command.
Then, install pipewire and its related packages.
```
sudo pacman -S pipewire pipewire-pulse gst-plugin-pipewire pipewire-alsa pipewire-media-session
```

Reboot and login to your new Desktop.

## Setup Plymouth for nice Password Prompt during Boot

Plymouth is in [AUR](https://aur.archlinux.org) , just clone the Repo and make the Package (i create a subfolder AUR within my homefolder)

```
cd AUR
git clone https://aur.archlinux.org/plymouth-git.git
makepkg -is
```
or get a [aur manager such as yay or paru](#aur-helper)

Do not replace encrypt with plymouth-encrypt, as plymouth-encrypt no longer exists.
Now modify the Hooks for the Initramfs, "plymouth" must be right after "base udev" and before "encrypt". 

```
HOOKS=(base udev plymouth encrypt autodetect modconf kms keyboard keymap consolefont block btrfs filesystems fsck)
```

Run `mkinitcpio -P` and wait for it to finish

Add some kernel-parameters after "root=/dev/mapper/luks" to make boot smooth. Edit `/boot/loader/entries/arch.conf` and append to options

```
... rootflags=subvol=@ quiet splash loglevel=3 rd.systemd.show_status=auto rd.udev.log_priority=3 vt.global_cursor_default=0 rw
```


For Plymouth Theming and Options, check [Plymouth on Arch Wiki](https://wiki.archlinux.org/title/plymouth). Issue the following command to set ROG logo as the plymouth theme.
```
sudo plymouth-set-default-theme -R bgrt
```

For KDE Theming you could check this nice [Youtube Video from Linux Scoop](https://www.youtube.com/watch?v=2GYT7BK41zk)
For XFCE4 Theming you could check this nice [Youtube Video from Linux Scoop](https://www.youtube.com/watch?v=X3siZNJN3ec)

# Nvidia propietary drivers
Install `nvidia` package from official repos. Double check to see if linux-headers are installed to avoid blackscreen on reboot.  Do NOT run `nvidia-xconfig` on Arch as it results in black screen. Install the following packages:
```
sudo pacman -S nvidia-dkms nvidia-settings nvidia-prime acpi_call
```

# Useful Customizations

## Install asusctl tool

Add [Luke Jones](https://asus-linux.org/)'s Repo to pacman.conf (/etc/pacman.conf)
```
sudo bash -c "echo -e '\r[g14]\nSigLevel = DatabaseNever Optional TrustAll\nServer = https://arch.asus-linux.org\n' >> /etc/pacman.conf"

sudo pacman -S asusctl
```
Activate DBUS Messaging for the new asus deamon to get asus notifications upon changing fan profile etc.

```
systemctl --user enable asus-notify
systemctl --user start asus-notify
```

Run the following commands: 
```
asusctl -c 85 		# Sets charge limit to 85% if you do not want this, do not execute this line
asusctl fan-curve -m Quiet -f cpu -e true
asusctl fan-curve -m Quiet -f gpu -e true 
asusctl fan-curve -m Performance -f cpu -e true
asusctl fan-curve -m Performance -f gpu -e true
asusctl fan-curve -m Balanced -f cpu -e true
asusctl fan-curve -m Balanced -f gpu -e true
```

For fine-tuning read the [Arch Linux Wiki](https://wiki.archlinux.org/title/ASUS_GA401I#ASUSCtl) or the [Repository from Luke](https://gitlab.com/asus-linux/asusctl). asusctl requires kernel 5.15+, which at the time of writing has not been released. Install the ROG kernel instead.

## Install ROG/asusctl Kernel
After adding the above repo, install the ROG kernel by running
```
sudo pacman -S linux-g14 linux-g14-headers 
#kernel headers are very important otherwise nvidia module will not load, resulting in black screen.	
```
After installing the kernel, edit `/boot/loader/loader.conf` and add the following to it:
```
default arch-g14.conf
timeout 3
editor 0
```

Then run `sudo cp /boot/loader/entries/arch.conf /boot/loader/entries/arch-g14.conf && sudo nano /boot/loader/entries/arch-g14.conf`

Replace the lines that start with `title`, `linux`, and `initrd` with this:
```
title   Arch Linux (g14 kernel)
linux   /vmlinuz-linux-g14
initrd  /amd-ucode.img
initrd  /initramfs-linux-g14.img
```

and finally do 
```
sudo mkinitcpio -P
```

## Switch Profile On Charger Connect
To automatically turn Performance profile on charger connect and Quiet profile on charger disconnect, run the following
```
echo "#\!/bin/bash\nasusctl profile -P Performance\n" > ~/.local/share/scripts/switch_performance.sh && echo "#\!/bin/bash\nasusctl profile -P Quiet\n" > ~/.local/share/scripts/switch_quiet.sh
```
Make the scripts executable by running
```
chmod +x  ~/.local/share/scripts/switch_performance.sh ~/.local/share/scripts/switch_quiet.sh
```
Go to Settings -> Power Management -> Energy Saving -> On AC Power. enable Run Script, select switch_performance script from ~/.local/share/scripts/ using open file dialogue button. Go to On Battery Tab, enable Run Script and select switch_quiet script from ~/.local/.local/share/scripts/ using open file dialogue button. Manually giving path does not appear to work for some reason.

**Optional:** Enable battery full charge notification. Go to KDE Settings -> Notifications -> Application Settings -> Configure Events. Select Charge Complete and Select Show a message in popup

## ROG Key Map
Go to KDE Settings->Shortcuts->Custom Shortcuts. Click Edit->New->Global Shortcut->Command/URL. Name it `NvidiaSettings`. Set trigger to `ROG Key` and set action to `nvidia-settings`  

## Change Fan Profile
Go to KDE Settings->Shortcuts->Custom Shortcuts. Click Edit->New->Global Shortcut->Command/URL. Name it  `ChangeFanProfile`, set trigger to `fn + f5` and action to `asusctl profile -n`

## Mic Mute Key
Run `usb-devices` and look for the device that says `Product=N-KEY Device`. Note the vendor id. For my zephyrus it is `0b05`.  Run 
```
sudo find /sys -name modalias | xargs grep -i 0b05
```

Find the line that goes like:
```
.../input/input18/modalias:input:b0003v0B05p1866e0110-e0...
```
Copy the part after `input:`, before the first `-e`. In my case, it is `b0003v0B05p1866e0110`.  Create a file named `/etc/udev/hwdb.d/90-nkey.hwdb` and add
```
/etc/udev/hwdb.d/90-nkey.hwdb
evdev:input:b0003v0B05p1866*
 KEYBOARD_KEY_ff31007c=f20 # x11 mic-mute, space in start is important in this line
```
After that, update `hwdb`.
```
sudo systemd-hwdb update
sudo udevadm trigger
```

## Gamma Correction
In display and monitor -> gamma, change gamma to 0.9 for better colors

## Touchpad Gestures
 Use [fusuma](https://github.com/iberianpig/fusuma) to get touchpad gestures. Create `~/.local/share/scripts/fusuma.sh` and add
 ```
 #!/bin/bash
 fusuma -d #for running in daemon mode
```
Add this script to autostart in KDE settings. For macOS like gestures use [this config](https://github.com/iberianpig/fusuma/wiki/KDE-to-mimic-MacOS.). 4 finger gestures are not working. My config is in the repo.

Install bluetooth related packages
```
sudo pacman -S bluez bluez-utils
sudo systemctl enable bluetooth.service
sudo systemctl start bluetooth.service
```

# Miscellaneous
## Fetch on Terminal Start
After installing and enabling zsh and oh-my-zsh with powerlevel10k, create file `~/.zshenv` and do the following:
- Install fastfetch-git from pamac.
- Add `fastfetch --load-config paleofetch` in `~/.zshenv`

## Key delay
Reduce key input delay to 250 ms for a better keyboard experience.

## AUR Helper
You can install `paru`, an AUR helper like this:
`cd ~ && mkdir paru && git clone https://aur.archlinux.org/paru.git && cd paru && makepkg -sci && cd ~ && rm -r paru/`

After installing `paru`, you can use it like pacman to install AUR packages.

Examples:
* paru -S \<package\>
* paru -Syu
* paru -Rns \<package\>

_Do not run paru as root. It will ask for permission when it needs to._

Alternatively, you can use [pamac](https://aur.archlinux.org/packages/pamac-aur/), which also has a gui.
Or, you can use [yay](https://aur.archlinux.org/packages/yay)

## Install Viper4Linux
```
yay -S viper4linux-git
yay -S viper4linux-gui
```

# KDE Stuff

## Install KDE Desktop Environment (optional) 

### Get X.Org and KDE Plasma

Install xorg and kde packages

```
pacman -S xorg 
sudo pacman -S plasma kde-applications pulseaudio
```

### SDDM Loginmanager

```
sudo pacman -S sddm
sudo systemctl enable sddm
```

## Remove extra KDE Packages (for kde)
`kde-applications` installs a bunch of packages that I do not need so I removed them. First remove the following groups of applications.
```
sudo pacman -Rns kdepim kde-games kde-education kde-multimedia
```
Then, remove some apps from other groups too.
```
sudo pacman -R kwrite kcharselect yakuake kdebugsettings kfloppy filelight kteatime konqueror konversation kopete
```
To see what various applications do, check out the [kde-applications](https://archlinux.org/groups/x86_64/kde-applications/) group on Arch website.


## Yet Another Magic Lamp

A better [magic lamp](https://github.com/zzag/kwin-effects-yet-another-magic-lamp) effect. In latest plasma versions, exclude "disable unsupported effects" next to the search bar in settings for the effect to appear.

## Maximize to new desktop
In Kwin scripts, install "kwin-maximize-to-new-desktop" and run:
```
mkdir -p ~/.local/share/kservices5
ln -s ~/.local/share/kwin/scripts/max2NewVirtualDesktop/metadata.desktop ~/.local/share/kservices5/max2NewVirtualDesktop.desktop
```
Then install kdesignerplugin through `pacman -S kdesignerplugin`. Logout and login again after configuration changes. My config:
```
Trigger: Maximize only
Position: Next to current
```
([git](https://github.com/Aetf/kwin-maxmize-to-new-desktop#window-class-blacklist-in-configuration-is-blank))

## KDE Tweaks (for kde)
### Window Size
I prefer the apps to open in windowed mode, in the center of the screen with 1280 * 720 resolution. To do this, add Window Rule, name it `Window Size`. Set the following rules

- Window Class: Unimportant
- Match whole window class: No
- Window Types: Normal window (deselect all others)
- Size: Apply Initially, 1280x720
- Ignore Requested Geometry: Force, Yes.

To make brave launch in 1280x720, do the following:
- Edit `/usr/share/applications/brave-browser.desktop`
- line 111: change `Exec=brave %U` to `Exec=brave %U --window-size="1280,720"`
- Repeat for line 111 and 223

# Optimus
**NOTE: `asusctl` and `optimus-manager` may conflinct with eachother. If using `asusctl`, it is recommended to uninstall `optimus-manager` with `sudo pacman -Rns optimus-manager optimus-manager-qt`**


## Optimus Manager (for some laptops like mux switch)
Install optimus-manager and optimus-manager-qt. After rebooting, it should work fine. 

- **Type C external display**
HDMI works out of the box, displays can be connected to type C port but require switching to dedicated graphics. Can be done either through asusctl or Optimus Manager QT.
	
- **Optimus Manager configuration**
In optimus manager qt, go to optimus tab, and select ACPI call switching method and set PCI reset to No. Enable PCI remove. Startup mode is integrated. In Nvidia, set dynamic power management to fine. Modeset, Pat and overclocking options.

- **Sleep/Shutdown issues**
System was only going to sleep once and after that got stuck on shutdown and sleep. This happened because I had set switching method to bbswitch in optimus manager. Swithced to acpi_call to fix.

# Oh-My-ZSH (for zsh)

I like to use oh-my-zsh with Powerlevel10K theme

```
sudo pacman -S git git-lfs curl wget zsh zsh-completions firefox-i18n-de
chsh -s /bin/zsh # (relogin to activate)
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
mkdir .local/share/fonts
cd .local/share/fonts
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Regular.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Italic.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold%20Italic.ttf
fc-cache -v
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
```

Set `ZSH_THEME="powerlevel10k/powerlevel10k"` in `~/.zshrc`

**Fixing terminal resize issue:** On Konsole and some other terminals, prompt is expected to be on the left side only. If right side of the prompt is enabled in powerlevel10k, then resizing Konsole results in the prompt becoming jittery and breaking to multiple lines as shown [here](https://github.com/romkatv/powerlevel10k/blob/master/README.md#horrific-mess-when-resizing-terminal-window). To fix the problem, open Konsole, go to settings -> edit current profile -> Scrolling and disable "Reflow lines" option.

https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Hibernation
https://wiki.archlinux.org/title/btrfs#Swap_file
https://confluence.jaytaala.com/display/TKB/Use+a+swap+file+and+enable+hibernation+on+Arch+Linux+-+including+on+a+LUKS+root+partition
