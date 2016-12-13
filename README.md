Arch-Notes
==========

Instructions for Arch installation notes + Arch on IMac installation.

Credit for this information was taken from these sources:

-   [pandeiro/arch-on-air]
-   [ArchLinux Installation With OS X on Macbook Air (Dual Boot)]

Procedure
---------

### 1. Make bootable USB media with Arch ISO image ([wiki])

### 2. Hold the <alt/option> key and boot into USB

### 3. Create partitions

If the installation is on apple device the partition table should look
like this:

    fdisk -l
    cgdisk /dev/sd*

#### Partitions:

##### /dev/sd\*4 - \[128MB\] Apple HFS+ “Boot Loader”

##### /dev/sd\*5 - \[256MB\] Linux filesystem “Boot”

##### /dev/sd\*6 - \[Rest of space\] Linux filesystem “Root”

### 4. Format and mount partitions

    mkfs.ext4 /dev/sda5
    mkfs.ext4 /dev/sda6
    mount /dev/sda6 /mnt
    mkdir /mnt/boot && mount /dev/sda5 /mnt/boot

Create a swapfile :

    dd if=/dev/zero of=/mnt/swapfile bs=1G count=8
    chmod 600 /mnt/swapfile
    mkswap /mnt/swapfile
    swapon /mnt/swapfile

Use this method for creatind a swap partition instead of a swapfile:

    #### Partitions:
    ##### /dev/sd*4 - [128MB]         Apple HFS+ "Boot Loader"
    ##### /dev/sd*5 - [256MB]         Linux filesystem "Boot"
    ##### /dev/sd*6 - [X]             Linux Swap "Swap"
    ##### /dev/sd*7 - [Rest of space] Linux filesystem "Root"

mkfs.ext4 /dev/sd*5\
mkswap /dev/sd*6\
mkfs.ext4 /dev/sd*7\
mount /dev/sd*7 /mnt\
mkdir /mnt/boot && mount /dev/sda5 /mnt/boot\
swapon /dev/sd\*6

    ### 5. Installation
    Internet connection required, for wireless option use:

wifi-menu

pacstrap /mnt base base-devel\
genfstab -U -p /mnt &gt;&gt; /mnt/etc/fstab

    ### 6. Optimize fstab for SSD, add swap

nano /mnt/etc/fstab\
``/dev/sda6 / ext4 defaults,noatime,discard,data=writeback 0 1\
/dev/sda5 /boot ext4 defaults,relatime,stripe=4 0 2\
/swapfile none swap defaults 0 0

    ### 7. Configure system

arch-chroot /mnt /bin/bash\
passwd\
echo myhostname &gt; /etc/hostname\
ln -s /usr/share/zoneinfo/Canada/Eastern /etc/localtime\
hwclock –systohc –utc\
useradd -m -g users -G wheel -s /bin/bash myusername\
passwd myusername\
pacman -S sudo

    ### 8. Grant sudo

echo “%wheel ALL=(ALL) ALL” &gt; /etc/sudoers.d/10-grant-wheel-group

    ### 9. Set up locale

nano /etc/locale.gen

locale-gen\
echo LANG=en\_US.UTF8 &gt; /etc/locale.conf\
export LANG=en\_US.UTF-8

    ### 10. Set up mkinitcpio hooks and run
    Insert "keyboard" after "autodetect" if it's not already there.

nano /etc/mkinitcpio.conf

Then run it:

    mkinitcpio -p linux

### 11. Set up GRUB/EFI

To boot up the computer we will continue to use Apple’s EFI\
bootloader, so we need GRUB-EFI:

    pacman -S grub-efi-x86_64

#### Configuring GRUB

    nano /etc/default/grub

Aside from setting the quiet and rootflags kernel parameters,\
a special parameter must be set to avoid system (CPU/IO)\
hangs related to ATA, as per
\[\[<https://bbs.archlinux.org/viewtopic.php?pid%3D1295212#p1295212>\]\[this
thread\]\]:

    GRUB_CMDLINE_LINUX_DEFAULT="quiet rootflags=data=writeback libata.force=1:noncq"

Additionally, the grub template is broken and requires this adjustment:

    # Fix broken grub.cfg gen
    GRUB_DISABLE_SUBMENU=y

    grub-mkconfig -o boot/grub/grub.cfg
    grub-mkstandalone -o boot.efi -d usr/lib/grub/x86_64-efi -O x86_64-efi --compress=xz boot/grub/grub.cfg

Copy boot.efi (generated in the command above) to a USB stick for use
later in OS X:

``` {.{.bash}}
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
```

### 13. Launch Disk Utility in OS X

Format (“Erase”) /dev/sda4 using Mac journaled filesystem

### 14. Create boot file structure

This procedure allows the Apple bootloader to see our Arch Linux system\
and present it as the default boot option.

``` {.{.bash}}
cd /Volumes/disk0s4
mkdir System mach_kernel
cd System
mkdir Library
cd Library
mkdir CoreServices
cd CoreServices
touch SystemVersion.plist
```

``` {.{.bash}}
nano SystemVersion.plist
```

``` {.{.example}}
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

Copy boot.efi from your USB stick to this CoreServices directory. The\
tree should look like this:

``` {.example}
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

``` {.bash}
sudo bless --device /dev/disk0s4 --setBoot
```

You may need to disable the System Integrity Projection of OS X:

-   Restart the computer, while booting hold down Command-R to boot
    into\
    recovery mode.
-   Once booted, navigate to the “Utilities &gt; Terminal” in the top\
    menu bar.
-   Enter “csrutil disable” in the terminal window and hit the return\
    key.
-   Restart the machine and System Integrity Protection will now be\
    disabled.

End of Arch Linux is installation.

Reboot the computer and hold the alt/option key to select operating system.
