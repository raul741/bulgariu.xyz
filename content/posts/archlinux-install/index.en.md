+++ 
draft = false
date = 2023-05-21T12:32:26+02:00
title = "Arch Linux LUKS encrypted UEFI installation"
description = "A simple Arch Linux installation guide"
slug = ""
authors = ["Raul Bulgariu Suciu"]
tags = ["Linux", "Tutorial"]
categories = []
externalLink = ""
series = []
+++

This is a guide meant to help you get an Arch Linux install working on a UEFI computer with full disk encryption, if your needs differ in any way, you can consult the original installation guide [here](https://wiki.archlinux.org/title/Installation_guide). Let's get started:

# Important disclaimer
Please, read each step carefully and if possible, first try to perform the installation within a [virtual machine](https://www.virtualbox.org/), I don't know the full extent of any possible errors that this guide might have so if you spot any, please feel free to [tell me about it!](/en/contact/).

# Download the .iso
1. Download the latest .iso from the official [Arch Linux website](https://archlinux.org/download/)

# Create a bootable USB drive
1. Download and use [Ventoy](https://www.ventoy.net/en/download.html) to create a multi-bootable USB drive
2. Drag the Arch Linux .iso into the Ventoy partition of your USB
3. Find out the [BIOS button](https://www.tomshardware.com/reviews/bios-keys-to-access-your-firmware,5732.html) for your computer brand and use it to boot into your USB drive
4. Once you reach the boot screen, just select the default options
> I might make a Ventoy installation tutorial in the future, stay tuned.

# Setting keyboard layout
1. Find out the codename for your keyboard layout, the layout files can be seen with `ls /usr/share/kbd/keymaps`
2. Run ``loadkeys KB`` while replacing "KB" with the code for your correct layout

> For example, in my case I would run `loadkeys es` to use the spanish keyboard layout.

# Check for UEFI support
1. Once we got our keyboard working properly, we can go ahead and check real quick that we do have UEFI support by running ```ls /sys/firmware/efi/efivars```
2. If the command above didn't spit out a bunch of files then your computer's not running in UEFI mode, check BIOS settings and if you don't have any UEFI support, refer to the [official Arch Linux installation guide](https://wiki.archlinux.org/title/Installation_guide), as an Arch Linux BIOS installation exceeds the scope of this guide

# Establish an Internet connection
> If you're connected through ethernet then it should work out of the box.
1. If you want to use Wi-Fi for the installation, first run `iwctl`
2. While in this interactive prompt, run `station list` to find out the names for your wireless interface
3. Afterwards, run: ```station WIRELESS_INTERFACE connect SSID``` while replacing WIRELESS_INTERFACE with your own *(for example, mine is called `wlan0`)* and SSID with the name of your wireless network
4. Once connected, hit **Ctrl+D** to exit the prompt

# Partitioning the disk
> Please, remember that disk names will ***usually*** vary depending on each computer, I urge you to run `lsblk` and check yourself which is the disk you wish to partition, especially if you're running a multi-disk setup. ***This will probably save you from accidentally messing up the wrong disk.***

> Also, if you're using an NVMe disk drive then the naming will change from sda, sdb, etc... to something similar like **"nvme0n1"**, `lsblk` will also help you there.

1. Now we're getting into the danger zone, please read the text above and make sure you know the correct name for your disk drive, from now on I will be using /dev/sdX to refer to the installation disk, ***replace the X with the correct letter for your disk***
2. Once everything is figured out, we'll run `cfdisk /dev/sdX`, from here we'll choose a GPT partition table for our disk if it's completely empty or delete the existing partitions
3. Once we have our free space, select "New" and create a 1G partition **(Boot partition)**, then change its "Type" to ***"EFI System"***
4. Then we'll select the remaining free space, hit "New" again and create a new partition with the remaining space **(Root partition)**
5. Once everything is done, hit "Write", then "Quit"

# Encrypting the root partition
> If everything went correctly, running `lsblk` should now show the newly created partitions.

1. Now we can begin to encrypt our system by running `cryptsetup luksFormat /dev/sdX2`, you will be prompted to enter the passphrase for booting up your system, please, ***do NOT forget this passphrase***

> Remember you're supposed to run the above command on the root partition, not on the entire disk itself.

2. Run `cryptsetup open /dev/sdX2 crypt` to open your newly encrypted partition

# Creating filesystems
1. Create the filesystem for your EFI boot partition by running `mkfs.vfat -F32 /dev/sdX1`
2. Create the root filesystem with `mkfs.ext4 /dev/mapper/crypt`

# Mounting filesystems
1. Run `mount /dev/mapper/crypt /mnt` to mount the root filesystem
2. Run `mount --mkdir /dev/sdX1 /mnt/boot` to mount your boot filesystem
3. Run `lsblk` to make sure that everything went well

# Create the swap file
> The swap file is disk memory that'll be utilized when there's not enough RAM, if you skip this step then your system will freeze everytime it uses up too much RAM.

> The amount of swap you need will depend on your needs, if you have no intention to configure hibernation then you can leave it at a far smaller number, I personally use 2GB of swap with 8GB of RAM.
1. Run `dd if=/dev/zero of=/mnt/swapfile bs=1M count=xxxx status=progress` while replacing "xxxx" with the amount of megabytes you're gonna give to your swapfile
2. Run `chmod 600 /mnt/swapfile` to set the right permissions
3. Run `mkswap /mnt/swapfile` to turn it into an actual swapfile
4. Run `swapon /mnt/swapfile` to activate it

# Pacstrapping
1. Now for installing the Arch Linux files, run `pacstrap -K /mnt base base-devel linux linux-firmware neovim`
> You can replace neovim with your preferred terminal editor of choice

# Generating /etc/fstab
> This is a pretty important step, it'll tell your operating system which partitions to mount and where when booting up.
1. Run `genfstab -U /mnt >> /mnt/etc/fstab` to generate an fstab file using partition UUIDs

# Chrooting into the new environment
1. Run `arch-chroot /mnt` to switch to your Arch Linux installation

# Setting locales
1. Run `ln -sf /usr/share/zoneinfo/Europe/Madrid /etc/localtime` *(replace /Europe/Madrid with your timezone)*
2. Run `hwclock --systohc`
3. Edit /etc/locale.gen with your editor of choice and uncomment the locales you wish to use *(I personally use en_US and es_ES)*
4. Run `locale-gen` to generate your selected locales
5. Run `echo 'LANG=en_US.UTF-8' > /etc/locale.conf`
> The above command will change the display language of your OS, if you wish to use spanish or whatnot, modify accordingly.
6. Run `echo 'KEYMAP=es' > /etc/vconsole.conf`
> This one takes care of the keymap used by default in TTYs, will save you a headache later on when booting into the installation, and as always, if you use a different keymap, modify accordingly.

# Setting hostname
1. Run `echo 'genesis' > /etc/hostname` and replace genesis with your preferred hostname
2. Modify /etc/hosts with your editor of choice and insert the following lines:
```bash
127.0.0.1   localhost
::1         localhost
```

# Configure initramfs for encrypted booting
1. Modify /etc/mkinitcpio.conf with your editor of choice and in the `HOOKS` array, add `encrypt` between `block` and `filesystems`so that it looks something like this:
```
HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block encrypt filesystems fsck)
```
2. Run `mkinitcpio -P`

# Installing the bootloader
> Replace amd-ucode with intel-ucode if you have an Intel processor.
1. Run `pacman -S grub efibootmgr amd-ucode` to install the bootloader and CPU microcode
2. Run `echo "GRUB_CMDLINE_LINUX=cryptdevice=UUID=$(blkid -s UUID -o value /dev/sdX2):crypt" >> /etc/default/grub`
> Remember to replace sdX2 with the correct disk partition, otherwise you won't be able to boot!
3. After running the command above, open `/etc/default/grub` with your editor of choice and replace the original "GRUB_CMDLINE_LINUX" with the one you echoed into the file
4. Without closing your editor, please remember to also uncomment the `GRUB_ENABLE_CRYPTODISK=y` line within the file
5. Run `grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB`
6. Run `grub-mkconfig -o /boot/grub/grub.cfg`

# Setting the root password
1. Run `passwd` to set your root password

# Final touches
That's actually it! Now we ***could*** reboot if we wanted and we'd have a "working" system, but there are a few things we should take care of first while we're still here:

## Installing NetworkManager / iwd
These two will help you actually connect to the Internet once you boot into Arch Linux, ***however you can only choose one***, NetworkManager is more novice friendly although a bit heavy on resources, meanwhile iwd is a far more minimalist network daemon, if you don't know what to pick, just go with NetworkManager:

### NetworkManager
1. Run `pacman -S networkmanager` to install the network daemon
2. Once done, run `systemctl enable NetworkManager` for the daemon to start next reboot
> By the way, keep in mind the capital letters when dealing with NetworkManager, if you run `systemctl enable networkmanager` then it won't do anything.
> After you reboot the system, all you have to do is run `nmtui` to bring up a fancy TUI menu for connecting to your wireless network.

### iwd
1. Run `pacman -S iwd` and install it
2. Run `systemctl enable iwd`
> For connecting to Wi-Fi post-reboot, you just have to follow the same steps at the beginning of the guide, `iwctl`, `station wlan0 connect`, etc...

## Creating an user account
> Remember to replace "raul" with your preferred username!
1. Run `useradd -m -G wheel,games,network,audio,video -s /bin/bash raul`
2. Run `EDITOR=nvim visudo` and uncomment the `%wheel ALL=(ALL:ALL) ALL` line
> Replace nvim above with your preferred editor, the step above will give your user administrator privileges.
3. Run `passwd raul` or whatever your username is supposed to be, and give your account a password as well

## Reducing swappiness
> Swappiness is how often your system will make use of swap memory, unless you have around 4 GB of RAM, you'll most likely want to lower this value to increase system performance, however feel free to adjust the value to whatever fits right for you.
1. Run `echo 'vm.swappiness=20' > /etc/sysctl.d/99-swappiness.conf`

# Finishing up
That's about all of it! Now that everything finished up, hit Ctrl+D to quit the chroot session and run `reboot` so you can boot into your newly installed Arch Linux system *(remember to remove the bootable USB drive)*, once you get past the login screen you'll realize that there's nothing but a terminal, that's because this is where the real journey starts, you'll most likely want a desktop environment to install so you can make actual use of the PC.

> By the way remember the tip from earlier to use `nmtui` to connect to your wireless network.

While this might go against the essence of building your own work environment, if you want something that just works out of the box, just run the following command:
```bash
sudo pacman -Syu xorg xorg-server ffmpeg4.4 ffmpegthumbnailer tumbler gvfs ttf-roboto ttf-roboto-mono xfce4 xfce4-goodies lightdm lightdm-gtk-greeter lightdm-gtk-greeter-settings pulseaudio pulseaudio-alsa pulseaudio-jack && sudo systemctl enable lightdm
```
And then reboot your system! XFCE is a lightweight and great desktop environment that *"just works"*.
