Arch-Notes
==========

Instructions for Arch installation notes + Arch on IMac/Book installation.

__References:__

- [pandeiro/arch-on-air](https://github.com/keinohguchi/arch-on-air/blob/master/README.md)
- [ArchLinux Installation With OS X on Macbook Air (Dual Boot)](http://panks.me/posts/2013/06/arch-linux-installation-with-os-x-on-macbook-air-dual-boot/)
- [Arch Installation guide](https://wiki.archlinux.org/index.php/installation_guide)
- [ArchLinux on MacBook(Air)](https://wiki.archlinux.org/index.php/MacBook)

- - - -

- [Partitioning](#3-create-partitions)
- [Installation](#5-installation)
- [Configuration](#7-configure-system)
- [Bootloaders](#11-bootloader)
- [Network](#12-network)
- [Post Installation](#post-installation)


Procedure
---------

### 1. Make bootable USB media with Arch ISO image ([wiki](https://wiki.archlinux.org/index.php/USB_Flash_Installation_Media), [video guide](https://www.youtube.com/watch?v=EnepPp7Xl8Y))

```bash
dd if = archlinux-2015.01.01-dual.iso | pv | of = /dev/sd*
```

### 2. Hold the _alt/option_ key and boot into USB

### 3. Create partitions

```bash
fdisk -l
cgdisk /dev/sd*
```

#### Partitions:

If the installation is on Apple device the partition table should look
like this:

Mount point   | Partition     | Name         | Partition type      | Suggested size
------------- | ------------- | ------------ | ------------------- | ------------- 
NONE          | /dev/sdX4     | Boot Loader  | Apple HFS+          | 128M  
/             | /dev/sdX5     | Boot         | Linux filesystem    | 256MB  
/             | /dev/sdX6     | Root         | Linux filesystem    | [Rest of space] 

Check if the partition scheme of the disk is right:

```bash
lsblk
```

### 4. Format and mount partitions

```bash
mkfs.ext4 /dev/sdX5
mkfs.ext4 /dev/sdX6
mount /dev/sdX6 /mnt
mkdir /mnt/boot && mount /dev/sdX5 /mnt/boot
```
Create a swapfile :

```bash
dd if=/dev/zero of=/mnt/swapfile bs=1G count=8
chmod 600 /mnt/swapfile
mkswap /mnt/swapfile
swapon /mnt/swapfile
```

#### 4.1 Use this method for creating a swap partition instead of a swapfile:

Mount point   | Partition     | Name         | Partition type      | Suggested size
------------- | ------------- | ------------ | ------------------- | ------------- 
NONE          | /dev/sdX4     | Boot Loader  | Apple HFS+          | 128M  
/             | /dev/sdX5     | Boot         | Linux filesystem    | 256MB  
NONE          | /dev/sdX6     | Swap         | Linux Swap          | [X] 
/             | /dev/sdX7     | Root         | Linux filesystem    | [Rest of space] 

Check if the partition scheme of the disk is right:

```bash
lsblk
```

Create Filesystem:

```bash
mkfs.ext4 /dev/sdX5
mkfs.ext4 /dev/sdX7
mount /dev/sdX7 /mnt
mkdir /mnt/boot && mount /dev/sdX5 /mnt/boot
mkswap /dev/sdX6
swapon /dev/sdX6
```

#### 4.2 Installation with GPT in BIOS:


Mount point   | Partition     | Name         | Partition type      | Suggested size
------------- | ------------- | ------------ | ------------------- | ------------- 
NONE          | /dev/sdX1     | Boot         | BIOS boot partition | 2M   
/             | /dev/sdX2     | Root         | Linux filesystem    | [Rest of space]   

Check if the partition scheme of the disk is right:

```bash
lsblk
```


Create Filesystem:

```bash
mkfs.ext4 /dev/sdX2
mount /dev/sdX2 /mnt
```
Create a swapfile :

```bash
dd if=/dev/zero of=/mnt/swapfile bs=1G count=8
chmod 600 /mnt/swapfile
mkswap /mnt/swapfile
swapon /mnt/swapfile
```

### 5. Install the base system
Internet connection required, for wireless option use:

```bash
wifi-menu
```

For DHCP:

```bash
systemctl enable dhcpcd.service
```


Test:

```bash
ping www.google.com
```

Run
```bash
pacstrap /mnt base base-devel
genfstab -p /mnt >> /mnt/etc/fstab
```

### 6. Optimize fstab for SSD, add swap (Apple only)

```bash
vi /mnt/etc/fstab
```

```example
/dev/sd*6  /       ext4  defaults,noatime,discard,data=writeback 0 1
/dev/sd*5  /boot   ext4  defaults,relatime,stripe=4 0 2
/swapfile  none    swap  defaults 0 0
```

If your drive is HDD, you fstab file should look like this (Apple only):

```example
/dev/sd*6  /      ext4  rw,defaults,noatime,data=writeback 0 1
/dev/sd*5  /boot  ext4  rw,defaults,data=ordered 0 2
```

### 7. Configure system

```bash
arch-chroot /mnt /bin/bash
passwd
echo myhostname > /etc/hostname
ln -s /usr/share/zoneinfo/Canada/Eastern /etc/localtime
hwclock –systohc –utc
useradd -m -g users -G wheel -s /bin/bash myusername
passwd myusername
pacman -S sudo
```

### 8. Grant sudo

```bash
echo “%wheel ALL=(ALL) ALL” > /etc/sudoers.d/10-grant-wheel-group
```

### 9. Set up locale

```bash
nano /etc/locale.gen
locale-gen
echo LANG=en_US.UTF8 > /etc/locale.conf
export LANG=en_US.UTF-8
```

### 10. Set up mkinitcpio hooks and run

Insert "keyboard" after "autodetect" if it's not already there.

```bash
vi /etc/mkinitcpio.conf
```

Then run it:

```bash
mkinitcpio -p linux
```

### 11. Bootloader 

To boot up the computer we will continue to use Apple’s EFI bootloader, so we need GRUB-EFI:

```bash
pacman -S grub-efi-x86_64
```

#### Configuring GRUB

```bash
vi /etc/default/grub
```

A special kernal parameter must be set to avoid system (CPU/IO)
hangs:

```example
GRUB_CMDLINE_LINUX_DEFAULT="quiet rootflags=data=writeback libata.force=1:noncq"
```
Additionally, the grub template is broken and requires this adjustment:

```example
#Fix broken grub.cfg gen
GRUB_DISABLE_SUBMENU=y
```

Generate the configuration file of grub:

```bash
grub-mkconfig -o boot/grub/grub.cfg
grub-mkstandalone -o boot.efi -d usr/lib/grub/x86_64-efi -O x86_64-efi --compress=xz boot/grub/grub.cfg
```

Copy boot.efi (generated in the command above) to a USB stick for use
later in OS X:

```bash
mkdir /mnt/usbdisk && mount /dev/sdb /mnt/usbdisk 
cp boot.efi /mnt/usbdisk/
```

### 11.1 BIOS

For GRUB/EFI bootloader configuration 

Install GRUB:

```bash
 pacman -S grub
```

#### Configuring GRUB

```bash
vi /etc/default/grub
```

The grub template is broken and requires this adjustment:

```example
#Fix broken grub.cfg gen
GRUB_DISABLE_SUBMENU=y
```

Generate the configuration file of grub:

```bash
grub-install --target=i386-pc --recheck /dev/sdX
grub-mkconfig -o boot/grub/grub.cfg
```

### 12. Network
Arch doesn't have wireless included in the base packages. Install the following packages before rebooting for the WiFi to work on reboot:

```bash
pacman -S iw wireless_tools wpa_supplicant dialog
```

### 13. Setup boot in OS X
Before rebooting, exit from the *chroot* and unmount partition/s:

```bash
exit
umount -R /mnt
reboot
```

### 14. Launch Disk Utility in OS X

Format (“Erase”) /dev/sda4 using Mac journaled filesystem.

### 15. Create boot file structure

This procedure allows the Apple bootloader to see our Arch Linux system and present it as the default boot option.

```bash
cd /Volumes/disk0s4
mkdir System mach_kernel
cd System
mkdir Library
cd Library
mkdir CoreServices
cd CoreServices
touch SystemVersion.plist
```

```bash
nano SystemVersion.plist
```

```example
<xml version="1.0" encoding="utf-8"?>
<plist version="1.0">
<dict>
    <key>ProductBuildVersion</key>
    <string></string>
    <key>ProductName</key>
    <string>Linux</string>
    <key>ProductVersion</key>
    <string>Arch Linux</string>
</dict>
</plist>
```

Copy boot.efi from your USB stick to this CoreServices directory. The tree should look like this:

```example
|___mach_kernel
|___System
       |
       |___Library
              |
              |___CoreServices
                      |
                      |___SystemVersion.plist
                      |___boot.efi
```

### 16. Make Boot Loader partition bootable

```bash
sudo bless --device /dev/disk0s4 --setBoot
```

You may need to disable the System Integrity Projection of OS X:

-   Restart the computer, while booting hold down Command-R to boot
    into recovery mode.
-   Once booted, navigate to the “Utilities > Terminal” in the top menu bar.
-   Enter “csrutil disable” in the terminal window and hit the returnkey.
-   Restart the machine and System Integrity Protection will now be disabled.

End of Arch Linux is installation.

Reboot the computer and hold the alt/option key to select operating system.

----

Post Installation
-----------------
[GUI](#gui)

#### GUI
 1. [Installation](#1-installation)
 2. [Starting Session](#2-starting session)
  2. [.xinitrc](#i-xinitrc)

          
#### 1. Installation:

```bash
sudo pacman -S gnome
```

#### 2. Starting Session
I will be using [xinit](https://wiki.archlinux.org/index.php/Xinit) to start the desktop enviroment, a display manager line [GDM](https://wiki.archlinux.org/index.php/GDM#Autostarting_applications_with_GDM) can be used and enable with [systemd](https://wiki.archlinux.org/index.php/Systemd#Using_units).

```bash
sudo pacman -S xorg-xinit
```

##### i. .xinitrc
The _.xinitrc_ file should look like this, __first__ run:

```bash
cp /etc/X11/xinit/xinitrc ~/.xinitrc
sudo nano .xinitrc
```
__then__

```example
# merge in defaults and keymaps

if [ -f $sysresources ]; then







    xrdb -merge $sysresources

fi

if [ -f $sysmodmap ]; then
    xmodmap $sysmodmap
fi

if [ -f "$userresources" ]; then







    xrdb -merge "$userresources"

fi

if [ -f "$usermodmap" ]; then
    xmodmap "$usermodmap"
fi

# start some nice programs

if [ -d /etc/X11/xinit/xinitrc.d ] ; then
 for f in /etc/X11/xinit/xinitrc.d/?*.sh ; do
  [ -x "$f" ] && . "$f"
 done
 unset f
fi

# twm &
#xclock -geometry 50x50-1+1 &
#xterm -geometry 80x50+494+51 &
#xterm -geometry 80x20+494-0 &
#exec xterm -geometry 80x66+0+0 -name login
exec gnome-session
```







