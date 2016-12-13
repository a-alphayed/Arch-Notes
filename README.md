Arch-Notes
==========

Instructions for Arch installation notes + Arch on IMac/MacBook installation.

The information in this documentation was taken from these sources:

- [pandeiro/arch-on-air](https://github.com/keinohguchi/arch-on-air/blob/master/README.md)
- [ArchLinux Installation With OS X on Macbook Air (Dual Boot)](http://panks.me/posts/2013/06/arch-linux-installation-with-os-x-on-macbook-air-dual-boot/)

Procedure
---------

### 1. Make bootable USB media with Arch ISO image ([wiki](https://wiki.archlinux.org/index.php/USB_Flash_Installation_Media), [video guide](https://www.youtube.com/watch?v=EnepPp7Xl8Y))

### 2. Hold the <alt/option> key and boot into USB

### 3. Create partitions

If the installation is on Apple device the partition table should look
like this:

```bash
fdisk -l
cgdisk /dev/sd*
```

#### Partitions:

```example
##### /dev/sd*4 - [128MB] Apple HFS+ “Boot Loader”

##### /dev/sd*5 - [256MB] Linux filesystem “Boot”

##### /dev/sd*6 - [Rest of space] Linux filesystem “Root”
```

### 4. Format and mount partitions

```bash
mkfs.ext4 /dev/sd*5
mkfs.ext4 /dev/sd*6
mount /dev/sd*6 /mnt
mkdir /mnt/boot && mount /dev/sd*5 /mnt/boot
```
Create a swapfile :

```bash
dd if=/dev/zero of=/mnt/swapfile bs=1G count=8
chmod 600 /mnt/swapfile
mkswap /mnt/swapfile
swapon /mnt/swapfile
```

Use this method for creating a swap partition instead of a swapfile:

#### Partitions:
```example
##### /dev/sd*4 - [128MB]         Apple HFS+ "Boot Loader"
##### /dev/sd*5 - [256MB]         Linux filesystem "Boot"
##### /dev/sd*6 - [X]             Linux Swap "Swap"
##### /dev/sd*7 - [Rest of space] Linux filesystem "Root"
```

Run:

```bash
mkfs.ext4 /dev/sd*5
mkswap /dev/sd*6
mkfs.ext4 /dev/sd*7
mount /dev/sd*7 /mnt
mkdir /mnt/boot && mount /dev/sda5 /mnt/boot
swapon /dev/sd*6
```
### 5. Installation
Internet connection required, for wireless option use:

```bash
wifi-menu
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

### 6. Optimize fstab for SSD, add swap

```bash
nano /mnt/etc/fstab
```

```example
/dev/sd*6 / ext4 defaults,noatime,discard,data=writeback 0 1
/dev/sd*5 /boot ext4 defaults,relatime,stripe=4 0 2
/swapfile none swap defaults 0 0
```

If your drive is HDD, you fstab file should look like this:

```example
/dev/sd*6 / ext4 defaults,noatime,discard,data=writeback 0 1
/dev/sd*5 /boot ext4 defaults,relatime,stripe=4 0 2
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
nano /etc/mkinitcpio.conf
```

Then run it:

```bash
mkinitcpio -p linux
```

### 11. Set up GRUB/EFI

To boot up the computer we will continue to use Apple’s EFI bootloader, so we need GRUB-EFI:

```bash
pacman -S grub-efi-x86_64
```

#### Configuring GRUB

```bash
nano /etc/default/grub
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

Run:

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

Arch doesn't have wireless included in the base packages. Install the following packages before rebooting for the WiFi to work on reboot:

```bash
pacman -S iw wireless_tools wpa_supplicant dialog
```

### 12. Setup boot in OS X
Before reboot, exit from the *chroot* and unmount all the partition:

```bash
exit
umount /mnt/{boot,root}
reboot
```

### 13. Launch Disk Utility in OS X

Format (“Erase”) /dev/sda4 using Mac journaled filesystem.

### 14. Create boot file structure

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

### 15. Make Boot Loader partition bootable

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
