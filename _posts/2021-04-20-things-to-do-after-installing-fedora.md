---
layout: post 
title: "Things to do after installing Fedora"
date: 2021-04-20
author: "sn99"
comments: true 
description: "A guide of things to do after installing fedora"
keywords: "fedora, linux, guide, tricks"
---

A set of things that I do and thing will help others too. Feel free to raise questions and suggest new things.

> Fedora Workstation is a reliable, user-friendly, and powerful operating system for your laptop or desktop computer. It supports a wide range of developers, from hobbyists and students to professionals in corporate environments.

My system does not use a NVIDIA GPU so you might have to [fetch it for yourself](https://rpmfusion.org/Howto/NVIDIA). I
usually apply the settings/tweaks in the same order as I will be writing them.

## DNF Flags

```shell
echo 'fastestmirror=1' | sudo tee -a /etc/dnf/dnf.conf
echo 'max_parallel_downloads=7' | sudo tee -a /etc/dnf/dnf.conf # You can change it to 10 or other depending on your connection speed
echo 'deltarpm=true' | sudo tee -a /etc/dnf/dnf.conf
cat /etc/dnf/dnf.conf
```

## Update the system

```shell
sudo dnf upgrade --refresh
sudo dnf clean all
sudo fwupdmgr get-devices
sudo fwupdmgr refresh --force
sudo fwupdmgr get-updates
sudo fwupdmgr update
```

After this reboot.

## Additional repositories

Software -> Software Repositories -> Third Party Repositories -> Enable All

After this:

```shell
sudo dnf install -y  https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm
sudo dnf install -y https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak update # I do not like snaps

sudo dnf upgrade --refresh
sudo dnf groupupdate core
sudo dnf install -y rpmfusion-free-release-tainted
sudo dnf install -y dnf-plugins-core
```

## Fonts & few other stuff that require restart

I cannot stress how much good fonts matter to me and the most hassle free way is:

```shell
sudo dnf copr enable dawid/better_fonts
sudo dnf install fontconfig-enhanced-defaults fontconfig-font-replacements
```

### Btrfs filesystem optimizations

1. `sudo gedit /etc/fstab`, change it to look something like this:
    ```shell
    UUID=<do-not-change> /                       btrfs   subvol=root,ssd,noatime,space_cache,discard=async,compress=zstd:1 0 0
    UUID=<do-not-change> /boot                   ext4    defaults        1 2
    UUID=<do-not-change>          /boot/efi               vfat    umask=0077,shortname=winnt 0 2
    UUID=<do-not-change> /home                   btrfs   subvol=home,ssd,noatime,space_cache,discard=async,compress=zstd:1 0 0
    ```

2. `sudo systemctl daemon-reload`

3. `sudo systemctl enable fstrim.timer`

### Changing swappiness

If you have 8GB or more ram you might benefit from it otherwise leave it as it is.

1. To see current swappiness enter `cat /proc/sys/vm/swappiness`, it should print `60`, we wanna make it 10.

2. `sudo gedit /etc/sysctl.conf`

3. Enter `vm.swappiness=10`, now step 1 should print 10 after a restart.

### Improving boot time

1. Remove startup applications, I use `gnome-tweaks` for a GUI like experience.

2. Run the following to find what service is taking the longest:

   ```shell
   systemd-analyze
   systemd-analyze blame
   systemd-analyze critical-chain
   ```

   This might vary from system to system and distro to distro, in my case(fedora) I disabled `dnf-makecache.service`
   which took around `32s`. To do so:

    ```shell
    systemctl disable NetworkManager-wait-online.service
    systemctl disable dnf-makecache.service 
    systemctl disable dnf-makecache.timer
    gsettings set org.gnome.software download-updates false
    ```

   You might wanna google every service that you think about disabling and what it does, in my case it just updates dnf
   cache which I usually like to do manually.

### Some kernel boot parameters

```shell
     sudo grubby --update-kernel=ALL --args="processor.ignore_ppc=1 nowatchdog"
``` 

After this restart the system.

## Gnome Tweak Tool & Settings

```shell
sudo dnf install gnome-tweak-tool
flatpak install flathub org.gnome.Extensions # preferred way now
```

Now head to https://extensions.gnome.org/local/ and update those extensions.

I usually change the following:

- Tweaks -> General -> Turn off Suspend when laptop is closed
- Tweaks -> Keyboard & Mouse -> Mouse Click Emulation -> Area
- Tweaks -> Top Bar -> enable everything except Seconds


- Settings -> About -> Change Device Name
- Settings -> Users -> Add profile pic
- Settings -> Mouse & Touchpad -> enable Tap to Click

## Themes and Icons

I like to change a bit but nothing too custom or heavy, I liked the Clear Linux theme before they decides to go full
vanilla gnome.

```shell
sudo dnf install papirus-icon-theme # I used to go for paper-icons but they are much less maintained
wget -qO- https://git.io/papirus-folders-install | sh

sudo dnf install materia-gtk-theme
```

Now in gnome-tweaks -> Appearance:

Applications -> Materia-compact Icons -> Papirus Shell -> Materia-light-compact

```shell
papirus-folders -C bluegrey --theme Papirus
```

## Terminal

I have a few settings like using I-beam instead of block and using transparency but it might come down to personal
preferences. I do use a custom shell prompt.

```shell
sh -c "$(curl -fsSL https://starship.rs/install.sh)"
```

Add the following to the end of ~/.bashrc:

```shell
# ~/.bashrc

eval "$(starship init bash)"
```

## Multimedia

```shell
sudo dnf groupupdate sound-and-video
sudo dnf install -y libdvdcss
sudo dnf install -y gstreamer1-plugins-{bad-\*,good-\*,ugly-\*,base} gstreamer1-libav --exclude=gstreamer1-plugins-bad-free-devel ffmpeg gstreamer-ffmpeg 
sudo dnf install -y lame\* --exclude=lame-devel
sudo dnf group upgrade --with-optional Multimedia
sudo dnf install gstreamer1-plugins-base gstreamer1-plugins-good gstreamer1-plugins-ugly gstreamer1-plugins-bad-free gstreamer1-plugins-bad-free gstreamer1-plugins-bad-freeworld gstreamer1-plugins-bad-free-extras ffmpeg
```

## Apps

### Firefox

```shell
sudo dnf config-manager --set-enabled fedora-cisco-openh264
sudo dnf install -y gstreamer1-plugin-openh264 mozilla-openh264
```

Menu -> Addons -> Plugins -> enable OpenH264

### Profile-sync-daemon

```shell
sudo dnf install -y profile-sync-daemon
psd
# First time running psd so please edit /home/$USER/.config/psd/psd.conf to your liking and run again
gedit /home/$USER/.config/psd/psd.conf
# Close your browser now
systemctl --user enable psd.service
systemctl --user start psd.service
systemctl --user status psd.service
psd preview
```

### Flatpak apps

```shell
flatpak install flathub org.videolan.VLC # dnf one just glitches for me
flatpak install org.signal.Signal
flatpak install org.qbittorrent.qBittorrent
flatpak install com.discordapp.Discord 
flatpak install com.github.micahflee.torbrowser-launcher
flatpak install com.github.tchx84.Flatseal
```

### Some other Packages and tools related to coding

```shell
sudo dnf install -y java-latest-openjdk
sudo dnf groupupdate c-development


sudo dnf copr enable kwizart/fedy
sudo dnf install fedy -y

sudo dnf install fedpkg fedora-packager rpmdevtools ncurses-devel pesign grubby
sudo /usr/libexec/pesign/pesign-authorize # Authorize user for kernel build
sudo dnf install openssl libXi-devel gcc-c++ qt5-devel gcc flex make bison openssl-devel elfutils-libelf-devel llvm
```

### Steam

```shell
sudo dnf install steam
```