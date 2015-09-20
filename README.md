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
* `vimpager`
* `wget`

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

# Kodi installation

## xorg

Check if there are system updates:

`pacman -Syu --confirm`

Install xorg and intel driver, select `mesa-libgl` for `libgl` and some tools

```
# xorg
pacman -S xorg-server xorg-server-utils xf86-video-intel mesa libva-intel-driver
# blackbox to check xorg installation
pacman -S blackbox xorg-xinit xterm
```

Check if xorg starts fine (as `kodi` user):

```
echo "/usr/bin/blackbox" > ~/.xinitrc
startx
```

Remove `.xinitrc` if everything is fine

## kodi

```
pacman -S kodi libnfs libplist shairplay udisks unrar unzip upower lsb-release libbluray
```

Autostart kodi [kodi-standalone-service](https://aur.archlinux.org/packages/kodi-standalone-service/)

```
pacman -S base-devel
```

As `kodi` user

```
mkdir ~/aur && cd ~/aur
wget https://aur.archlinux.org/cgit/aur.git/snapshot/kodi-standalone-service.tar.gz
cd kodi-standalone-service
makepkg -sri
sudo systemctl enable kodi.service
reboot
```

# VRC-1100 remote (AKA Hama Remote)

The NUC's IR receiver doesn't play well with a VRC-1100 remote (or the other way around).
I am currently using the IR receiver bundled with the VRC-1100 until I replace the remote.

I don't use the receiver with LIRC, but use `inputlircd` instead, which emulates a LIRC daemon.
The source repository for inputlircd doesn't seem to exist anymore. However, you can download a tarball
from one of gentoo's mirrors:

http://mirror.switch.ch/ftp/mirror/gentoo/distfiles/inputlircd-0.0.1_pre15.tar.gz

```
wget http://mirror.switch.ch/ftp/mirror/gentoo/distfiles/inputlircd-0.0.1_pre15.tar.gz
tar xvzf inputlircd-0.0.1_pre15.tar.gz
cd inputlircd-0.0.1_pre15
make && sudo make install
sudo systemctl disable kodi.service
```

Create a file /usr/local/bin/create-remote-symlinks.sh with the following content:

```
#!/bin/bash
lines=$(grep -n "Vendor=05a4 Product=9881" /proc/bus/input/devices | awk -F : '{print $1}')
count=0;

for line in $lines; do
    let relevantLine=$line+5;
    event=$(head -n $relevantLine /proc/bus/input/devices | tail -n1 | sed -e 's:=: :' | sed -e 's:\ :\n:g' | grep event);
    ln -sf /dev/input/$event /dev/input/irremote$count;
    let count=$count+1
done

```

Create a file /usr/local/bin/create-lircd-symlinks.sh with the following content:

```
#!/bin/bash
if [ ! -e /var/run/lirc ]; then
  mkdir -p /var/run/lirc
fi
if [ ! -e /var/run/lirc/lircd ]; then
  ln -s /dev/lircd /var/run/lirc/lircd
fi
```

Create a systemd unit for inputlircd (`/etc/systemd/system/inputlircd.service`):

```
[Unit]
Description=inputlircd lirc daemon
After=systemd-udevd.service

[Service]
ExecStartPre=/bin/bash /usr/local/bin/create-remote-symlinks.sh
ExecStartPre=/bin/bash /usr/local/bin/create-lircd-symlinks.sh
ExecStart=/usr/local/sbin/inputlircd -f -u root -g -m0 -c /dev/input/irremote0 /dev/input/irremote1

[Install]
WantedBy=multi-user.target
```

Enable the unit `systemctl enable inputlircd.service`

Create a modified version of `kodi.service`:

```
[Unit]
Description = Starts instance of Kodi using xinit
After = systemd-user-sessions.service network.target sound.target mysqld.service inputlircd.service
Requires = inputlircd.service
Conflicts=getty@tty7.service

[Service]
User = kodi
Group = kodi
PAMName=login
Type = simple
TTYPath=/dev/tty7
ExecStart = /usr/bin/xinit /usr/bin/dbus-launch --exit-with-session /usr/bin/kodi-standalone -- :0 -nolisten tcp vt7
Restart = on-abort
StandardInput = tty

[Install]
WantedBy = multi-user.target
```

Configure a kodi `Lircmap.xml` in `/home/kodi/.kodi/userdata/Lircmap.xml`

```
<lircmap>
    <remote device="irremote1">
        <start>KEY_HOMEPAGE</start>
        <pause>KEY_PLAYPAUSE</pause>
        <stop>KEY_STOPCD</stop>
        <info>BTN_RIGHT</info>
        <title>BTN_MOUSE</title>
        <skipplus>KEY_NEXTSONG</skipplus>
        <skipminus>KEY_PREVIOUSSONG</skipminus>
    </remote>
    <remote device="irremote0">
          <myTV>CTRL_SHIFT_KEY_T</myTV>
          <mymusic>CTRL_KEY_M</mymusic>
          <mypictures>CTRL_KEY_I</mypictures>
          <myvideo>CTRL_KEY_E</myvideo>
          <reverse>CTRL_SHIFT_KEY_B</reverse>
          <forward>CTRL_SHIFT_KEY_F</forward>
          <menu>CTRL_SHIFT_KEY_M</menu>
          <record>CTRL_KEY_R</record>
          <back>KEY_BACKSPACE</back>
          <left>KEY_LEFT</left>
          <right>KEY_RIGHT</right>
          <up>KEY_UP</up>
          <down>KEY_DOWN</down>
          <select>KEY_ENTER</select>
          <pageplus>KEY_PAGEUP</pageplus>
          <pageminus>KEY_PAGEDOWN</pageminus>
          <one>KEY_KP1</one>
          <two>KEY_KP2</two>
          <three>KEY_KP3</three>
          <four>KEY_KP4</four>
          <five>KEY_KP5</five>
          <six>KEY_KP6</six>
          <seven>KEY_KP7</seven>
          <eight>KEY_KP8</eight>
          <nine>KEY_KP9</nine>
          <zero>KEY_KP0</zero>
          <display>KEY_KPASTERISK</display>
          <clear>KEY_ESC</clear>
          <eject>ALT_META_KEY_ENTER</eject>        
    </remote>
</lircmap>
```

Reboot