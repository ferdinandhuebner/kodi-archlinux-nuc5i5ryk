# Kodi on an Intel NUC5i5RYK with Arch Linux

## NUC Preparation

### Firmware update

Update the firmware on the NUC to the latest version (fixes bugs with the IR receiver)

https://downloadcenter.intel.com/download/25228/BIOS-Update-RYBDWi35-86A-

* Download the `.BIO` file and copy it to a FAT32 formatted USB drive (`mkfs.fat -F32`)
* Plug in th USB drive, power on the NUC and press F7
* Follow the instructions to update the firmware

### Slow down the fan

In stock settings, the NUC's fan is running at 3000RPM even if the system is idle. 
I switched to `Quiet` mode in the fan settings and lowered the minimum fan speed to 2000 RPM (select 20%).

### Boot mode

If you don't want UEFI boot, disable UEFI boot in the NUC's BIOS. If you don't disable UEFI boot, 
the Arch Linux installation media will boot a system in UEFI mode. You have to make sure you can
boot from the SSD in UEFI mode as well which didn't work for me.

### Arch Linux ISO on an USB stick

`dd bs=4M if=/tmp/archlinux-2015.09.01-dual.iso of=/dev/sdf && sync`

# Arch Linux installation

## SSH 

I performed the installation via SSH:

* boot from the USB drive
* set a root password with `passwd`
* find the NUC's IP address with `ip addr` 
* permit root login with password by setting `PermitRootLogin yes` in `/etc/ssh/sshd_config`
* start sshd with `systemctl start sshd`

## Preparations

Follow the [Arch Linux installation guide](https://wiki.archlinux.org/index.php/Installation_guide)

* Prepare the disks: partition and format
* Mount the partitions at `/mnt` (and at `/mnt/boot` if you have a separate `/boot` partition)
* Configure mirrors (Mirrorlist)[https://www.archlinux.org/mirrorlist/]
* `pacstrap /mnt base`
* `genfstab -p /mnt >> /mnt/etc/fstab`
* chroot with a bash: `arch-chroot /mnt /bin/bash`

## In the chroot

* `echo "nuc" > /etc/hostname`
* `ln -sf /usr/share/zoneinfo/Europe/Zurich /etc/localtime`
* `vi /etc/locale.gen`
* `locale-gen`
* `echo "LANG=en_US.UTF-8" > /etc/locale.conf`
* `passwd`
* `pacman -S grub`
* `grub-install --recheck /dev/sda`
* `grub-mkconfig -o /boot/grub/grub.cfg`

### Additional packages and configurations before you reboot

Make sure to install `dhclient` if you want to configure your network with DHCP. I installed the following
packages prior to rebooting: 

* `dhclient`
* `openssh`
* `sudo`
* `vim`

### Exiting the chroot and rebooting

* Exit the chroot `exit`
* Unmount disks: `umount /mnt/boot && umount /mnt`
* reboot

# System configuration

Configure [DHCP networking with systemd](https://wiki.archlinux.org/index.php/Systemd-networkd#Basic_DHCP_network):

* `systemctl enable systemd-networkd.service`
* `systemctl start systemd-networkd.service`

Create a `kodi` user for everyday use:

`useradd -d /home/kodi -m -G wheel,network,audio,video,users kodi`

Enable and start the sshd

* `systemctl enable sshd`
* `systemctl start sshd`

Login with the `kodi` user and switch user to root.

If you want to allow the `kodi` user to execute commands with `sudo` without a password, 
append the following line with `visudo`:

`kodi ALL=(ALL) NOPASSWD: ALL`

I added the following to `/etc/inputrc`, but that is personal preference

```
# map "page up" and "page down" to search history based on current cmdline
"\e[5~": history-search-backward
"\e[6~": history-search-forward
# fix Home and End for German users
"\e[7~": beginning-of-line
"\e[8~": end-of-line
# konsole (alt + arrow key)
"\e[1;3C": forward-word
"\e[1;3D": backward-word
```