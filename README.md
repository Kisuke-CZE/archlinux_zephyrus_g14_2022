# Archlinux setup for Asus Zephyrus G14 2022 (GA402RJ/RK)

## Basic installation

Connect to wifi using `iwctl`:
```
station wlan0 scan
station wlan0 get-networks
station wlan0 connect <YOUR_SSID>
```

Now we will enable NTP synchronization and check time.  
```
timedatectl set-ntp true
timedatectl set-timezone Europe/Prague
timedatectl status
```

Now partition the disk. I choosed this schema for a 1TB drive:  
| Partition      |  Size  |                           Additional info                                   |
|----------------|--------|-----------------------------------------------------------------------------|
| /dev/nvme0n1p1 |  2GB   | LABEL=BOOT, fs=fat32, ESP, future mountpoint = /boot                        |
| /dev/nvme0n1p2 |  929GB | LABEL=ROOTFS, fs=btrfs, all data will be here on btrfs subvolumes           |

**Beware when creating partition for /boot. It mmust be set to type=ESP (EF00).**

I use cgdisk to create partitions: `cgdisk /dev/nvme0n1`

I want to have whole `ROOTFS` encrypted, so I will set up encrypted FS:
```
cryptsetup luksFormat --label CRYPTROOT /dev/nvme0n1p2
cryptsetup open /dev/nvme0n1p2 rootfs
```

**BEWARE**: It's important for my setup to create label on LUKS volume! This label must be totally different from label on partition and also different from filesystem label (we will create fs in next step). If you do not create label (or label will be same as partition for example), then you cannot use that label as cryptdevice parameter later! (Since I do not like to use UUID in manual setups, I preffer label for better readibility)

Partitioning done, so let's make filesystems:  
```
mkfs.vfat -F 32 -n BOOT /dev/nvme0n1p1
mkfs.btrfs -f -L ROOT /dev/mapper/rootfs
```

Now let's mount created rootfs where all data will be and create those subvolumes for that data. Since I want to support hibernation to swapfile, it needs to be larger than my RAM size (to be able to successfully hibernate).  
```
mount -t btrfs LABEL=ROOT /mnt
btrfs subvolume create /mnt/@root
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@swap
btrfs filesystem mkswapfile --size 43g /mnt/@swap/swapfile
```

Now unmount filesystem and mount with subvolumes we just created.  
```
umount /mnt/
mount -o noatime,subvol=@root LABEL=ROOT /mnt
mkdir -p /mnt/boot
mkdir -p /mnt/home
mkdir -p /mnt/swap
mkdir -p /mnt/.snapshots
mkdir -p /mnt/btrfs

mount -o noatime,discard=async,subvol=@home LABEL=ROOT /mnt/home
mount -o noatime,discard=async,subvol=@swap LABEL=ROOT /mnt/swap
mount -o noatime,discard=async,subvol=@snapshots LABEL=ROOT /mnt/.snapshots
mount -o noatime,discard=async LABEL=ROOT /mnt/btrfs
mount /dev/nvme0n1p1 /mnt/boot
swapon /mnt/swap/swapfile
```

Check if it's mounted successfully for sure with `df -Th` and `free -h`

If everithing is OK, let's install the base system with `pacstrap /mnt base base-devel linux linux-firmware btrfs-progs vim networkmanager amd-ucode man-db man-pages texinfo bash-completion`

Create fstab using `genfstab -Lp /mnt >> /mnt/etc/fstab`.

## Setting the target system

Now we finished basic installation. But installed system needs some setup and boot configuration. Let's get to it.

Chroot into installed system using `arch-chroot /mnt`

And do some basic settings:  
```
echo "<HOSTNAME>" > /etc/hostname
ln -sf /usr/share/zoneinfo/Europe/Prague /etc/localtime
hwclock --systohc
echo "LANG=cs_CZ.UTF-8" > /etc/locale.conf
echo "LANGUAGE=cs_CZ" >> /etc/locale.conf
echo "KEYMAP=us" > /etc/vconsole.conf
echo "FONT=lat2-16" >> /etc/vconsole.conf
```

Edit `/etc/hosts` with `vim /etc/hosts` to look like:  
```
127.0.0.1		localhost
::1				localhost
127.0.1.1		<HOSTNAME>.localdomain	<HOSTNAME>
```

Edit locales: `vim /etc/locale.gen`  
For me I only uncommented `cs_CZ.UTF-8 UTF-8` and `en_US.UTF-8 UTF-8`  
After that run `locale-gen`

Set root password with `passwd`.

Add `encrypt btrfs` to hooks in `/etc/mkinitcpio.conf` after `udev` hook.  
Example: `HOOKS=(base udev encrypt autodetect modconf kms keyboard keymap consolefont block btrfs filesystems fsck)`  
Also add `amdgpu` to modules section in `/etc/mkinitcpio.conf`.  
Example: `MODULES=(amdgpu)`  

Edit whe file with `vim /etc/mkinitcpio.conf`  
When you are done editing, run `mkinitcpio -p linux`.

Now let's install a bootloader. systemd-boot is kinda simple and easy to setup, so use `bootctl install`.  
Prepare configuration with `vim /boot/loader/loader.conf`:  
```
default	archlinux
timeout	0
editor	0
```
And now prepare configuration for that `archlinux` item with `vim /boot/loader/entries/archlinux.conf`:  
```
title          Arch Linux
linux          /vmlinuz-linux
initrd	       /amd-ucode.img
initrd         /initramfs-linux.img
options        cryptdevice=LABEL=CRYPTROOT:rootfs:allow-discards root=LABEL=ROOT rootflags=subvol=@root rw
```
For some emergency situations let's prepare `archlinux-textmode` option. Menu can be displayed holding space key on boot. `vim /boot/loader/entries/archlinux-textmode.conf`
```
title          Arch Linux (textmode)
linux          /vmlinuz-linux
initrd	       /amd-ucode.img
initrd         /initramfs-linux.img
options        cryptdevice=LABEL=CRYPTROOT:rootfs:allow-discards root=LABEL=ROOT rootflags=subvol=@root rw 3
```
If label won't work for some reason you can use UUID, but **do not use together!**:  
```
echo "options	cryptdevice=UUID=$(blkid -s UUID -o value /dev/nvme0n1p2):rootfs:allow-discards root=LABEL=ROOT rootflags=subvol=@root rw" >> /boot/loader/entries/archlinux.conf
echo "options	cryptdevice=UUID=$(blkid -s UUID -o value /dev/nvme0n1p2):rootfs:allow-discards root=LABEL=ROOT rootflags=subvol=@root rw 3" >> /boot/loader/entries/archlinux-textmode.conf
```

Consider to set up automatic update of systemd-boot according to this article: https://wiki.archlinux.org/title/Systemd-boot.

Now we should be ready to boot to newly installed system. We will exit the chroot environment, unmount all volumes and reboot.  
```
exit
swapoff /mnt/swap/swapfile
umount -R /mnt/
reboot
```

## Setup the installed system

After reboot you will boot directly into system you just installed. Login as `root` and proceed with the initial setup.

Enable NetworkManager and set wifi connection:  
```
systemctl enable --now NetworkManager
nmcli device wifi connect "{YOURSSID}" password "{SSIDPASSWORD}"
```

Enable time synchronization using `systemctl enable --now systemd-timesyncd`.

Edit `/etc/environment` and set `EDITOR=vim` with `vim /etc/environment`.

Add user for normal usage (we don'w want to run a desktop as root) and set his password:  
```
useradd -m -g users -G wheel,lp,power,audio -s /bin/bash {MYUSERNAME}
passwd {MYUSERNAME}
```
Execute `visudo` and uncomment `%wheel ALL=(ALL) ALL`

Now logout using `exit` and login with user you just created.

We can now disable root login by `sudo passwd -l root` and replacing root's encrypted password in `/etc/shadow` with `!`

Install some essential packages:  
```
sudo pacman -Sy acpid dbus
sudo systemctl enable acpid
```

Now we will enable multilib repository. So edit config with `sudo vim /etc/pacman.conf` and uncomment the `[multilib]` section.

As next step install the graphics drivers (I also want 32bit support for wine and older apps) using `sudo pacman -Sy mesa lib32-mesa vulkan-radeon lib32-vulkan-radeon libva-mesa-driver lib32-libva-mesa-driver mesa-vdpau lib32-mesa-vdpau`

Maybe we can use of `xf86-video-amdgpu`, depends on your choice of environment.

Now this depends on your choose. I want to use modern Gnome desktop and stick with Fedora/RHEL way, so I will install Pipewire: `sudo pacman -S pipewire lib32-pipewire wireplumber pipewire-alsa pipewire-pulse pipewire-jack`.

I also want Gnome on Wayland, so install Wayland: `sudo pacman -S wayland xorg-xwayland`.

Next let's install some [desktop environment](https://wiki.archlinux.org/title/Desktop_environment). I choosed [Gnome desktop](https://wiki.archlinux.org/index.php/GNOME) so `sudo pacman -S gnome gnome-initial-setup`.

As a final step before reboot into graphics environment enable graphical login - for Gnome desktop it is `sudo systemctl enable gdm`.

And reboot coomputer to run desktop environment. We will continue there, it will be lot easier.

## Final setup in the graphics environment.

### AUR helper

Blame me, but since I use a lot of packages from ArchLinux's famous [AUR](https://wiki.archlinux.org/index.php/Arch_User_Repository) I like to use [helper](https://wiki.archlinux.org/title/AUR_helpers) for package installing. I choosed [yay](https://aur.archlinux.org/packages/yay).

So open Gnome Console and install the helper:  
```
sudo pacman -S git
mkdir work
cd work
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

You can also preset the answers for usual questions. But beware - changes are needed when you want to edit PKGBUILD or other files for some reason. See `man yay` for more info.  
My favorite for everyday simple usage is: `yay --save --answerclean All --answerdiff None --removemake`

### Install Gnome Dash to Dock

I like to have dock so I will use previously installed AUR helper to install Dash to Dock extension: `yay -S gnome-shell-extension-dash-to-dock`

### Remove unused applications

There are some applications from default gnome collection I just do not use. Let's remove them: `sudo pacman -Rsncu gnome-maps gnome-weather gnome-photos gnome-clocks gnome-music totem cheese`

### Make booting process pretty with Plymouth

Well almost all important things are in place. Time for some cosmetics. Install plymouth `pacman -S plymouth`

Edit `/etc/mkinitcpio.conf` for example using `sudo gnome-text-editor /etc/mkinitcpio.conf`.

Add `plymouth` hook after `base udev` hooks. (but before `encrypt`).  
Example: `HOOKS=(base udev plymouth encrypt autodetect modconf kms keyboard keymap consolefont block btrfs filesystems fsck)`

If you want, you can install smooth transition for gdm: `yay -S gdm-plymouth`  
It will replace `gdm`. This step is completely optional and cosmetic - if you preffer to use gdm from main repository, you can just skip this.

I want to use `bgrt` theme. So `sudo gnome-text-editor /etc/plymouth/plymouthd.conf` and set `Theme=bgrt`.  
You might like different theme, see wiki for details: https://wiki.archlinux.org/title/plymouth  
Also displaying Arch logo in an option: `sudo cp /usr/share/plymouth/arch-logo.png /usr/share/plymouth/themes/spinner/watermark.png`

And add `quiet splash loglevel=3 rd.systemd.show_status=auto rd.udev.log_priority=3 vt.global_cursor_default=0` as kernel parameters: `sudo gnome-text-editor /boot/loader/entries/archlinux.conf`

Rebuild initramfs: `sudo mkinitcpio -p linux`

Now reboot the system to test it.

### Install Asus-Linux packages

Since we have gaming machine with dual graphics, we will install tools for this machine from https://asus-linux.org/wiki/arch-guide/ .

```
sudo su -
pacman-key --recv-keys 8F654886F17D497FEFE3DB448B15A6B0E9A3FA35
pacman-key --finger 8F654886F17D497FEFE3DB448B15A6B0E9A3FA35
pacman-key --lsign-key 8F654886F17D497FEFE3DB448B15A6B0E9A3FA35
pacman-key --finger 8F654886F17D497FEFE3DB448B15A6B0E9A3FA35
```

Edit pacman config with `vim /etc/pacman.conf` and add these lines at end:
```
[g14]
Server = https://arch.asus-linux.org
```

Continue with:  
```
pacman -Suy
pacman -S asusctl
systemctl enable --now power-profiles-daemon.service
pacman -S supergfxctl
systemctl enable --now supergfxd
pacman -S rog-control-center
pacman -S npm
exit

git clone https://gitlab.com/asus-linux/supergfxctl-gex.git /tmp/supergfxctl-gex && cd /tmp/supergfxctl-gex
npm install
npm run build && npm run install-user
cd ..
rm -rf supergfxctl-gex

git clone https://gitlab.com/asus-linux/asusctl-gex.git /tmp/asusctl-gex && cd /tmp/asusctl-gex
npm install
npm run build && npm run install-user
cd ..
rm -rf asusctl-gex
```

Logout from Gnome session and login again and after that go to `gnome-extensions` and enable installed extensions.

### Enable wayland support in Firefox

If you use Firefox as main browser as I do, it's good idea to enable native wayland support in it. (If you do not it will be running through XWayland compatability layer which from mine experience works OK, but why not run on wayland directly when app can).

So edit environment with `sudo gnome-text-editor /etc/environment`.  
And add `MOZ_ENABLE_WAYLAND=1`.  
You'll need to relogin to make change effective.

### Install firewall

For some basic safety I want to have some basic firewall on my computer. I like simple solutions, so I use just `nftables` and edit firewall rules by hand.

So install and enable nftables:  
```
sudo pacman -S nftables
sudo systemctl enable --now nftables
```

And put your rules into config with `sudo gnome-text-editor /etc/nftables.conf`.

Here is mine simple setup, which you can use as starting point:  
```
#!/usr/sbin/nft -f

flush ruleset

define ssh_admins = {10.11.18.39, 10.1.0.0/24}

table inet filter {
	chain input {
		type filter hook input priority 0; policy drop;

		# established/related connections
		ct state established,related accept

		# invalid connections
		ct state invalid drop

		# loopback interface
		iif lo accept

		# ICMP & IGMP
		ip6 nexthdr icmpv6 icmpv6 type { destination-unreachable, packet-too-big, time-exceeded, parameter-problem, mld-listener-query, mld-listener-report, mld-listener-reduction, nd-router-solicit, nd-router-advert, nd-neighbor-solicit, nd-neighbor-advert, ind-neighbor-solicit, ind-neighbor-advert, mld2-listener-report } accept
		ip protocol icmp icmp type { destination-unreachable, router-solicitation, router-advertisement, time-exceeded, parameter-problem } accept
		ip protocol igmp accept

		# SSH (port 22)
		tcp dport ssh ip saddr $ssh_admins accept

		# Allow Mikrotik discovery
		udp dport 5678 accept

		# Allow Mikrotik Streaming Sniffer
		udp dport 37008 accept

		# Allow Xerox printer discovery
		udp dport 22161 accept
		tcp dport 22161 accept

		# Allow mDSN
		udp dport mdns accept

		# HTTP (ports 80 & 443)
		# tcp dport { http, https } accept
	}

	chain forward {
		type filter hook forward priority 0; policy drop;
	}

	chain output {
		type filter hook output priority 0; policy accept;
	}

}
```

When you done editing, save the config and restart nftables `sudo systemctl restart nftables`.

### Enable Alt+Shift to change keyboard layouts
```
gsettings set org.gnome.desktop.wm.keybindings switch-input-source-backward "['<Alt>Shift_L', '<Shift>XF86Keyboard']"
gsettings set org.gnome.desktop.wm.keybindings switch-input-source "['<Alt>Shift_L', 'XF86Keyboard']"
```

### Automaticaly add bin directories from user's home directory

Create shell function using `sudo gnome-text-editor /etc/profile.d/homebin.sh` and put this content in:  
```
set_path(){

    # Check if user id is 1000 or higher
    [ "$(id -u)" -ge 1000 ] || return

    for i in "${@}";
    do
        # Check if the directory exists
        [ -d "$i" ] || continue

        # Check if it is not already in your $PATH.
        echo "$PATH" | grep -Eq "(^|:)$i(:|$)" && continue

        # Then append it to $PATH and export it
        export PATH="${PATH}:$i"
    done
}

set_path ~/bin ~/.local/bin
```

Then relogin to make change effective.

### Enable hibernation to SWAP file

Edit mkinitcpio config with `sudo gnome-text-editor /etc/mkinitcpio.conf` and add `resume` hook after `filesystem hook`.  
Example: `HOOKS=(base udev plymouth encrypt autodetect modconf kms keyboard keymap consolefont block btrfs filesystems resume fsck)`

Now run `sudo btrfs inspect-internal map-swapfile -r /swap/swapfile` and note the output number as OFFSET.

Now open boot config file with `sudo gnome-text-editor /boot/loader/entries/archlinux.conf` and add `resume=LABEL=ROOT resume_offset=OFFSET`.  
Full options example: `options	cryptdevice=LABEL=CRYPTROOT:rootfs:allow-discards root=LABEL=ROOT rootflags=subvol=@root quiet splash loglevel=3 rd.systemd.show_status=auto rd.udev.log_priority=3 vt.global_cursor_default=0 resume=LABEL=ROOT resume_offset=533760 rw`

Rebuild initrd: `sudo mkinitcpio -p linux`  
Install an extension which will add hibernate to Gnome menu: `yay -S gnome-shell-extension-hibernate-status-git`  
And reboot the machine and test it.

### Install extension to tweak Gnome quick-settings

`yay -S gnome-shell-extension-quick-settings-tweaks-git`

### Install liberation TTF font

It's practical to have Liberation font installed, mainly for Steam. `sudo pacman -S ttf-liberation`

### Solving problem when cursor is off in some fulscreen game - Gamescope

Sometimes it happens that game running in fulscreen has a cursor position off. You are pointing with cursor to some button, but in real, another button has focus/is selected.  
In my case this happened in native linux version of Tropico 6.  
Usualy, there is an option to run windows version of the game using Proton, which by my experience works OK.  
But we have also another option called [gamescope](https://archlinux.org/packages/community/x86_64/gamescope/).

Install gamescope using `sudo pacman -S gamescope`.  
Then you can enter game's properties in Steam and enter launch options to something like this: `gamescope  -W 1920 -H 1200 -f -- %command%`. Check the gamescope help for details.

### Enable mDNS resolution

To resolve local hostnames on network we need to enable mDNS:  
```
sudo pacman -S nss-mdns
sudo systemctl enable --now avahi-daemon
```

Now edit config file with `sudo gnome-text-editor /etc/nsswitch.conf`, find the `hosts` line and add `mdns_minimal [NOTFOUND=return]` (or `mdns4_minimal [NOTFOUND=return]` if you do not have ipv6 connectivity for faster resolution) before `resolve`.  
Example: `hosts: mymachines mdns_minimal [NOTFOUND=return] resolve [!UNAVAIL=return] files myhostname dns`

### Enable systemd-resolved

If you want advanced DNS resolving capabilities (like having multiple VPNs and query the right one DNS server for a domain in that VPN), you need some advanced resolver on your laptop. I found `systemd-resolved` pretty handy.  
For configuration details, see [wiki](https://wiki.archlinux.org/title/Systemd-resolved).  
You can easily change resolving settings from scripts. Example: `systemd-resolve --interface="vpn0" --set-dns="1.1.1.1" --set-domain=example.local`

```
sudo systemctl enable --now systemd-resolved
sudo ln -rsf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

### Make vim editor more sane

I use vim editor a lot, mostly on servers in textmode. So I want to disable mouse interaction and autoformating of pasted text on my desktop - to make vim bave like on most servers.  
To do that run `sudo gnome-text-editor /etc/vimrc` and add those lines:  
```
let skip_defaults_vim=1

" Set the mouse mode to 'r'
if has('mouse')
  set mouse=r
endif

" Disable autoformating when pasting content
filetype plugin indent off
```

### Make QT apps look good in Gnome/GTK

To make look QT apps looking mostly same as GTK apps in Gnome it's easier than ever today. Just install needed tools with `sudo pacman -S qgnomeplatform-qt5 qgnomeplatform-qt6`.

See [wiki](https://wiki.archlinux.org/title/Uniform_look_for_Qt_and_GTK_applications) for more details and options. But step above was good enough for me (I rarely use QT apps - for some reason it seems noone makes them anymore.)

### Remove unnecessary icons from application menu

I don't like crowded menu. There are some apps that are installed just as dependecy to another app, or some I use only in terminal (so I do not need desktop icon for that).  
I've created this script to hide icons from those apps:  
```
#!/bin/bash
UNNECESSARY_APPS=(
                  'avahi-discover'
                  'bssh'
                  'bvnc'
                  'qvidcap'
                  'qv4l2'
                  'libreoffice-base'
                  'libreoffice-draw'
                  'libreoffice-math'
                  'libreoffice-writer'
                  'libreoffice-calc'
                  'libreoffice-impress'
                  'calibre-ebook-edit'
                  'calibre-ebook-viewer'
                  'calibre-lrfviewer'
                  'vim'
                  'easytag'
                  'winetricks'
                  'stoken-gui'
                  'stoken-gui-small'
                  'cups'
                  'jconsole-java-openjdk'
                  'jshell-java-openjdk'
                  'displaycal-3dlut-maker'
                  'displaycal-vrml-to-x3d-converter'
                  'displaycal-curve-viewer'
                  'lstopo'
                  'displaycal-apply-profiles'
                  'displaycal-profile-info'
                  'displaycal-scripting-client'
                  'displaycal-synthprofile'
                  'displaycal-testchart-editor'
                 )

if [ ! -d "${HOME}/.local/share/applications" ]
then
  mkdir -p "${HOME}/.local/share/applications"
fi

for APP in ${UNNECESSARY_APPS[@]}
do
  if [ -f "/usr/share/applications/${APP}.desktop" ]
  then
    if [ ! -f "${HOME}/.local/share/applications/${APP}.desktop" ]
    then
      cp "/usr/share/applications/${APP}.desktop" "${HOME}/.local/share/applications/${APP}.desktop"
      sed -i '/NoDisplay=false/d' "${HOME}/.local/share/applications/${APP}.desktop"
      sed -i -E 's/(\[Desktop Entry\])/\1\nNoDisplay=true/' "${HOME}/.local/share/applications/${APP}.desktop"
    fi
  fi
done
```

### Install and set console font
To make console text more readable on laptop's display (default one is kinda small for this resolution by my perspective) in installed system, i suggest to use font `terminus`.

Install font using `pacman -S terminus-font`.

I found ideal size for me is `Terminus 10x18 bold` for a 1920x1200 screen of my Zephyrus G14. For detailed info about variants/encodings/sizes you can see this [manual page](https://files.ax86.net/terminus-ttf/README.Terminus.txt).

Then set it as default console font by putting `FONT=ter-v18b` in `/etc/vconsole.conf`. It's a good idea to run `sudo mkinitcpio -p linux` after that to propagate font to initramfs.

### Install ZSH
Actually on the start I was not thinking I will do this, since I wanted to have as close shell on my laptop as on servers I manage. But it seems to be most easy way to achieve some highlighting and more readibility in terminal on local machine. So I tired that and it works so far. But be warned, ZSH can have some capabilities that are not available in BASH, so be careful about this.

Anyway, let's install and activate ZSH:  
```
sudo pacman -S zsh zsh-completions zsh-autosuggestions ttf-meslo-nerd zsh-theme-powerlevel10k
chsh -s /bin/zsh
zsh
# Configure initial settings
echo 'bindkey  "^[[H"   beginning-of-line
bindkey  "^[[F"   end-of-line
bindkey  "^[[3~"  delete-char

source /usr/share/zsh-theme-powerlevel10k/powerlevel10k.zsh-theme' >> .zshrc
exec zsh
# After this theme wizard will be shown
```

## That's all

Since kernel 6.1 you do not need to install G14 specific kernel. You are ready to go

## Credits

This is just kinda notes for me how to install Archlinux on Zephyrus G14. Maybe it could be useful to someone else.

It's based on [Unim8trix's guide](https://github.com/Unim8trix/G14Arch). (Let's look here, there are some great ideas like automatic snapshots, which I do not use since I want to make them manually)

Of course on great work of guys at https://asus-linux.org/

I also had to mention great [Archlinux Wiki](https://wiki.archlinux.org/), which is source of most knowledges used in this guide. Please consider reading [Installation guide](https://wiki.archlinux.org/title/Installation_guide) and [General recommendations](https://wiki.archlinux.org/title/General_recommendations), you can get a lot of knowledge here and then make some own adjustments.

And also some mine old notes when I was using Archlinux on some Macbook was used when building this.
