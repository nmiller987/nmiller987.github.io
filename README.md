# nmiller987.github.io

There are three sections: first, the Arch install, then the Docker project, and then finally the Digital Ocean VPN project.

# Arch Install

# Goals

The following list is taken directly from the slides:
Overview: 
    - Follow the Arch Wiki to successfully install Arch in a VMWare VM
    - Completely document your steps
        - Include commands
        - Areas which were unclear or needed troubleshooting
    - Install a desktop environment or window manager
    - Customize the install in some way (different shell, aliases, setup scripts, etc.)

# Pre-installation
I'm downloading from mirrors.bloomu.edu, which is thankfully https. The archlinux site links to https://www2.cs.arizona.edu/stork/packagemanagersecurity/attacks-on-package-managers.html, which has greatly decreased my trust in http over a very small period of time.

### VMWare struggles
     
After opening the ISO with VMWare, it asks what kind of OS the ISO has on it. I hit Linux, only for it to respond with a very long dropdown menu for me to select what kind of Linux I'm using. Arch is not on it.

Out of the available options, Other Linux 6.x kernel 64-bit seems the most likely to work.
When prompted to allocate disk space, I chose 8 GB. That's probably way overkill, but I have spare.

### Powering On
After hitting Power On VM, it reassuringly shows the Arch logo.

### Connecting to the Internet
First, I run `iwctl` to start up an interactive prompt to pick a WiFi network.
`device list` is supposed to output available WiFi networks to connects to, but returns nothing.
The wiki does not seem to say what to do here. 
`help` lists commands to run in the iwctl prompt, but I can't figure out how to scroll up the massive list it sends. 

iwd can apparently assign IP addresses automatically, so I'm going to detour and edit /etc/iwd/main.conf as described in section 4.2 of the arch wiki page for iwd.

Oh my god! After ten minutes I've realized that I am, in fact, already connected to the internet and I don't need to mess around with this section of the wiki.
This was done by running `ping archlinux.org`

### Rest of pre-installation
`timedatectl` says that the system thinks it's in UTC time. This is an issue that I might come back to later.

The last section in the pre-installation guide is complicated stuff about disk partitioning. I could be wrong, but I'm going to gamble that VMWare deals with this for me.

# Installation

Worryingly, letters that I type in my VM take a couple seconds to actually show up. I changed the amount of RAM available to the VM to 2 gigs, but that didn't seem to change anything.

### Installing Packages
`pacstrap -K /mnt base linux linux-firmware` is the next step to take. However, running it leads to an error: not enough free disk space. I have more than enough memory allocated to the VM, so it's not VMWare's fault, I think.
This is probably the fault of the disk partitioning that I didn't do earlier. Time to step back to that.

### Partitioning 
This was frustrating, because the guide didn't do a great job of explaining this step. 
With lsblk, I could determind that /dev/sda had 8GB, but that didn't seem to be used by the filesystem (because it wasn't).
First, I needed to run `fdisk /dev/sda`
Then, running `g` created a new GPT partition table. I don't really know what that means, but it was mentioned in the guide.

With the sequence `n` `1` `Enter` `+512M` `t` `1` `1` , I created the EFI system partition. That's a weird sequence, so let me walk through it.

`n` stands for creating a new partition.
`1` was the default partition number.
`Enter` accepted the default for the first block to assign.
`+512M` defined the last block to write to, which was 512 megabytes after the first block. 
`t` changed partition...
`1` to type...
`1` (EFI system).

Creating the root partition was very similar, but I just accepted the defaults for last block to write to (the very last block available) and didn't bother changing the partition type. 

After that, I needed to save this partitioning with `w`.

I then formatted the EFI partition and root partition with 
`mkfs.fat -F32 /dev/sda1` and
`mkfs.ext4 /dev/sda2` respectively.

Finally, I needed to run 
`mount /dev/sda2 /mnt`
`mkdir /mnt/boot`
`mount /dev/sda1 /mnt/boot`

### Installation for real this time?
If mounting worked as intended, then I should be able to run 
`pacstrap -K /mnt base linux linux-firmware` will finally work.
And it does! It takes a little bit for everything to download, but it goes.

# Configure the system

First, I run `genfstab -U /mnt >> /mnt/etc/fstab`.

`arch-chroot /mnt` changes root into the new system.

`ln -sf /usr/share/zoneinfo/US/Central /etc/localtime` sets the correct time zone.
`hwclock --systohc` generates /etc/adjtime.

### Trying to get Vim

The next step doesn't work - vim doesn't work to edit a file. Turns out I don't have vim installed! Changing root into the new system disabled it, I think. I try to run `pacman -S vim man-db` to get vim and man pages.

Here, I get a weird error: Invalid or corrupted package (PGP signature)
To fix this, I run 
`pacman-key --init` and 
`pacman-key --populate archlinux`, and then my initial pacman commands works.

I run `vim /etc/locale.gen` and uncomment the lines beginning with en_US.
Then, I run `locale-gen`, and edit `/etc/locale.conf` to have `LANG=en_US.UTF-8`

### Network

I create /etc/hostname and write my hostname into it: landscape.
I'm able to ping archlinux.org, so I assume I have internet access.
I thiiiink I'm done with network configuration, but if I have problems later, I'll double check here.

I set the root password to the last name of an author I can see from my desk with `passwd`.

### Boot loader

I'm going to try to work with GRUB. I run `pacman -S grub` to get it.
`grub-install --target=i386-pc /dev/sda1` to install it to disk.
`grub-mkconfig -o /boot/grub/grub.cfg` generates the GRUB configuration file.

# Reboot

I run `exit`, and then `reboot`.
It loads back into the live environment, so something went wrong somewhere that I need to fix.

I think the issue is that I mounted grub to /sda1.
I run `grub-install --target=i386-pc /dev/sda`.
It errors, because I'm in the live installation.

I run `arch-chroot /mnt` to enter the system.
`grub-install --target=i386-pc /dev/sda` to hopefully fix my mistake, and then 
`grub-mkconfig -o /boot/grub/grub.cfg`

I run exit, reboot, and then I think I made an error
From the VMWare boot screen, I selected "Boot existing OS" in the hopes that it would boot my arch install. It got stuck on booting. I closed the tab, and found that I couldn't open it again.
This wiped the ~5 hours that I had put into this at this point.
# Doing the exact same stuff again

I'm not going to bother typing out the commands I run because I already did that once. I'll just run the same ones again.

I think that this reset did a much, much, much better job of teaching the value of this document than could possibly ever be done in class. I got up to GRUB in ten minutes, which is where I made my previous error. From here, I get a second shot at it.

I think I see what went wrong earlier. I forgot that I was working with EFI, not BIOS. 
I run `pacman -S efibootmgr`, as it's needed for working with GRUB.

After a bunch of debugging involving `mount | grep /boot`, I find out that EFI variables aren't even supported on the system.
Now, I'm stuck. I don't think it's possible to turn on EFI support, because I'm running this on a VM, and if I'm not on an EFI system then I might have messed up forever ago when I set up disk partitions.
I don't know how to fix this, but I know how to reset the machine and try again, which will probably be faster than figuring out how to fix it.

# Attempt 3

After `fdisk /dev/sda`, I run `g` to create a GPT disklabel.
To create a BIOS boot partition, I use the sequence `n 1 Enter +1M t 1 4` to create a partition with 1 megabyte of memory and set it to the BIOS boot type.
Then, I create a 2 GB swap partition with `n 2 Enter +2G t 2 19`.
Finally, I create a root partition with the remaining available space: `n 3 Enter Enter`.
`w` to save these partitions, then 
`mkfs.ext4 /dev/sda3` to format root, then 
`mkswap /dev/sda2`
`swapon /dev/sda2`
to set up the swap partition.
`mount /dev/sda3 /mnt` finishes the process.

Now I'm back to following along what I wrote earlier.

Did it!!! `exit` and `reboot` took me to a new screen, which was when I knew that I won.
# Post-Installation

`ping archlinux.org` errors, so I need to set up internet before moving on.
I can't find any way to fix this online that doesn't require certain programs, which I can't download because I don't have those programs to set up my network.
This sucks. I saved a snapshot of the VM before I rebooted into the system, so I'm going to try again.

# Attempt 4

This time, I already have the dull stuff like partitioning done. 
I run `pacman -S dhcpcd` to get a package that can help with network setup.
I run `exit` and `reboot` again to go back to the system.

# Post-Installation 

`ip link set ens33 up`
`dhcpcd ens33`
now works to get me an internet connection!

`pacman -S lxde` downloads LXDE, the desktop environment that I'll be using.
`systemctl enable lxdm` and `systemctl start lxdm` set up and start the display manager, finally getting me into the actual desktop environment.
After inputting my login name and password, the VM froze up.

Holding control and alt and hitting random keys, I can eventually force myself back to the terminal.

After trying to fix LXDM, I tried to reboot, but it errors. I have no clue what happened, so I just revert to an earlier snapshot.

After messing around with it, I can't find a way to fix LXDM, so I tried to find a way to instead remove and replace it. Every time I did that, it bricked the system and I had to revert to an earlier snapshot. I'm going to give up and try again on another fresh iso with a different desktop manager.

# Attempt 5

I'm back to the system. I'm going to try xfce this time.

`pacman -S xfce4 xfce4-goodies` to download it.
To avoid dealing with a display manager, I'm going to 
`echo "exec startxfce4" > ~/.xinitrc`
and try `startx`

It didn't work. Apparently, I need to `pacman -S xorg-server`.

It worked! Like ten hours into this project, I finally get to see a desktop.

# Post-Desktop

To create the three user accounts, I run 
`useradd -m nolan`
`useradd -m justin`
`useradd -m codi`

`echo "justin:GraceHopper1906" | sudo chpasswd`
`echo "codi:GraceHopper1906" | sudo chpasswd`
and
`sudo chage -d 0 justin`
`sudo chage -d 0 codi`
sets the passwords and forces a password change on first login for justin and codi.

`usermod -aG wheel nolan`
`usermod -aG wheel justin`
`usermod -aG wheel codi`
adds each user account to wheel, then 
`pacman -S sudo` because I forgot to do that earlier. Whoops. Then I need to 
`export EDITOR=vim` so that visudo works.
`visudo` and uncommenting `%wheel ALL=(ALL) ALL`.

To install a shell other than bash, I just run 
`pacman -S dash`, which gets the shell that I'm used to working with.

To set up ssh, I run 
`pacman -S openssh`
`systemctl start sshd`
`systemctl enable sshd`

To enable in color in the terminal, I put 
`alias ls='ls --color=auto' `
`alias grep='grep --color=auto' 
`alias egrep='egrep --color=auto' `
`alias fgrep='fgrep --color=auto' `
`export PS1='\[\e[1;32m\]\u@\h \[\e[1;34m\]\w\[\e[0m\] \$ '`
and then `source ~/.bashrc` to apply changes.

In order to boot into the GUI, I will need a display manager.
I run `pacman -S lightdm lightdm-gtk-greeter` to get one.
I then enable it with `systemctl enable lightdm`.
I reboot and test this, and it works!

I add a couple aliases to the ~/.bashrc file:
`alias update='pacman -Syu`
`alias ll='ls -a'`
`alias c='clear'`
When I try to run these in the terminal, it doesn't work.
Turns out I forgot to run 
`source ~/.bashrc`, which solves the issue.



# Docker Project

## Starting Off

I'm going to try to set up PiHole. A network-wide ad blocker sounds really nice, with how bad my YouTube ads have been getting. 

First, I download the docker.desktop app for Windows.

Then, I swap to the command prompt. I make a directory called `pihole` that I'll use for this project. 

https://github.com/pi-hole/docker-pi-hole seems pretty useful. I set up a docker-compose.yml in my directory based off of the guide.

When I try to run `docker-compose up -d`, I get hit with a weird error telling me that port 53 is not permitted, so now I have to figure that out.

Upon further research, it seems like trying to set this up on Windows would be a huge pain. Opening port 53 would require me to disable Windows DNS Client, which I can't find a way to do. I'm just going to try doing this on my laptop with Ubuntu instead.

## Ubuntu

First off, I run `sudo apt-get update` and `sudo apt-get update`. I already have docker.io installed. 

`sudo docker run -d -p 443:443 --name openvas mikesplain/openvas` to get the right container.

Then, open up Firefox, and go to `https://localhost`. The Greenbone UI pops up, prompting for a username and password. The default for both fields is `admin`.

To run a scan, I go to the Scans/Tasks page, then hit the magic want emoji in the top left. I accept the defaults, and then wait a while for the scan to run. 

