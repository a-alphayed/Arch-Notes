# Arch-Notes
Instructions for Arch installation notes + Arch on IMac installation.  



Credit for this information was taken from these sources:
- [[https://raw.githubusercontent.com/pandeiro/arch-on-air/master/README.org][pandeiro/arch-on-air]]
- [[http://panks.me/posts/2013/06/arch-linux-installation-with-os-x-on-macbook-air-dual-boot/][ArchLinux Installation With OS X on Macbook Air (Dual Boot)]]


** Procedure
*** 1. Make bootable USB media with Arch ISO image ([[https://wiki.archlinux.org/index.php/USB_Flash_Installation_Media][wiki]])
*** 2. Hold the <alt/option> key and boot into USB
*** 3. Create partitions
If the installation is on apple device the partition table should look like this: 

#+begin_src sh
fdisk -l
cgdisk /dev/sd*
#+end_src
**** Partitions:
***** /dev/sd*4 - [128MB]         Apple HFS+ "Boot Loader"
***** /dev/sd*5 - [256MB]         Linux filesystem "Boot"
***** /dev/sd*6 - [Rest of space] Linux filesystem "Root"
*** 4. Format and mount partitions
#+begin_src sh
mkfs.ext4 /dev/sda5
mkfs.ext4 /dev/sda6
mount /dev/sda6 /mnt
mkdir /mnt/boot && mount /dev/sda5 /mnt/boot
#+end_src
Create a swapfile :
#+begin_src sh
dd if=/dev/zero of=/mnt/swapfile bs=1G count=8
chmod 600 /mnt/swapfile
mkswap /mnt/swapfile
swapon /mnt/swapfile
#+end_src
Use this method for creatind a swap partition instead of a swapfile:
**** Partitions:
***** /dev/sd*4 - [128MB]         Apple HFS+ "Boot Loader"
***** /dev/sd*5 - [256MB]         Linux filesystem "Boot"
***** /dev/sd*6 - [X]             Linux Swap "Swap"
***** /dev/sd*7 - [Rest of space] Linux filesystem "Root"
#+begin_src sh
mkfs.ext4 /dev/sd*5
mkswap /dev/sd*6
mkfs.ext4 /dev/sd*7
mount /dev/sd*7 /mnt
mkdir /mnt/boot && mount /dev/sda5 /mnt/boot
swapon /dev/sd*6
#+end_src
*** 5. Installation
Internet connection required, for wireless option use:
#+begin_src sh
wifi-menu
#+end_src
#+begin_src sh
pacstrap /mnt base base-devel
genfstab -U -p /mnt >> /mnt/etc/fstab
#+end_src
*** 6. Optimize fstab for SSD, add swap
#+begin_src sh
nano /mnt/etc/fstab
#+end_src
#+begin_example
/dev/sda6 /     ext4 defaults,noatime,discard,data=writeback 0 1
/dev/sda5 /boot ext4 defaults,relatime,stripe=4              0 2
/swapfile none  swap defaults                                0 0
#+end_example
*** 7. Configure system
#+begin_src sh
arch-chroot /mnt /bin/bash
passwd
echo myhostname > /etc/hostname
ln -s /usr/share/zoneinfo/Canada/Eastern /etc/localtime
hwclock --systohc --utc
useradd -m -g users -G wheel -s /bin/bash myusername
passwd myusername
pacman -S sudo
#+end_src
*** 8. Grant sudo
#+begin_src sh
echo "%wheel ALL=(ALL) ALL" > /etc/sudoers.d/10-grant-wheel-group
#+end_src
*** 9. Set up locale
#+begin_src sh
nano /etc/locale.gen
#+end_src
#+begin_src sh
locale-gen
echo LANG=en_US.UTF8 > /etc/locale.conf
export LANG=en_US.UTF-8
#+end_src
*** 10. Set up mkinitcpio hooks and run
Insert "keyboard" after "autodetect" if it's not already there.
#+begin_src sh
nano /etc/mkinitcpio.conf
#+end_src
Then run it:
#+begin_src sh
mkinitcpio -p linux
#+end_src
*** 11. Set up GRUB/EFI
To boot up the computer we will continue to use Apple's EFI
bootloader, so we need GRUB-EFI:
#+begin_src sh
pacman -S grub-efi-x86_64
#+end_src
**** Configuring GRUB
#+begin_src sh
nano /etc/default/grub
#+end_src
Aside from setting the quiet and rootflags kernel parameters,
a special parameter must be set to avoid system (CPU/IO)
hangs related to ATA, as per [[https://bbs.archlinux.org/viewtopic.php?pid%3D1295212#p1295212][this thread]]:
#+begin_example
GRUB_CMDLINE_LINUX_DEFAULT="quiet rootflags=data=writeback libata.force=1:noncq"
#+end_example
Additionally, the grub template is broken and requires this adjustment:
#+begin_example
# Fix broken grub.cfg gen
GRUB_DISABLE_SUBMENU=y
#+end_example
#+begin_src sh
grub-mkconfig -o boot/grub/grub.cfg
grub-mkstandalone -o boot.efi -d usr/lib/grub/x86_64-efi -O x86_64-efi --compress=xz boot/grub/grub.cfg
#+end_src
Copy boot.efi (generated in the command above) to a USB stick for use later in OS X:
#+begin_src sh
mkdir /mnt/usbdisk && mount /dev/sdb /mnt/usbdisk 
cp boot.efi /mnt/usbdisk/
#+end_example
Arch doesn't have wireless included in the base package. Install the following packages before rebooting for the WiFi to work on reboot:
#+begin_src sh
pacman -S iw wireless_tools wpa_supplicant dialog
#+end_example

*** 12. Setup boot in OS X
Before reboot, exit from the *chroot* and unmount all the partition:

#+begin_src sh
exit
umount /mnt/{boot,root}
reboot
#+end_src
*** 13. Launch Disk Utility in OS X
Format ("Erase") /dev/sda4 using Mac journaled filesystem
*** 14. Create boot file structure
This procedure allows the Apple bootloader to see our Arch
Linux system and present it as the default boot option.
#+begin_src sh
cd /Volumes/disk0s4
mkdir System mach_kernel
cd System
mkdir Library
cd Library
mkdir CoreServices
cd CoreServices
touch SystemVersion.plist
#+end_src
#+begin_src sh
nano SystemVersion.plist
#+end_src
#+begin_example
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
#+end_example
Copy boot.efi from your USB stick to this CoreServices directory. 
The tree should look like this:
#+begin_example
|___mach_kernel
|___System
       |
       |___Library
              |
              |___CoreServices
                      |
                      |___SystemVersion.plist
                      |___boot.efi
#+end_example
*** 15. Make Boot Loader partition bootable
#+begin_src sh
sudo bless --device /dev/disk0s4 --setBoot
#+end_src

You may need to disable the System Integrity Projection of OS X:
- Restart the computer, while booting hold down Command-R to boot into recovery mode.
- Once booted, navigate to the “Utilities > Terminal” in the top menu bar.
- Enter "csrutil disable" in the terminal window and hit the return key.
- Restart the machine and System Integrity Protection will now be disabled.

End of Arch Linux is installation.

Reboot the computer and hold the alt/option key to
select which operating system to boot.
