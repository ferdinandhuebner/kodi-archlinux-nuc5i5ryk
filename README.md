# Kodi on an Intel NUC5i5RYK with Arch Linux

## NUC Preparation

### Firmware update

Update the firmware on the NUC to the latest version (fixes bugs with the IR receiver)

https://downloadcenter.intel.com/download/25228/BIOS-Update-RYBDWi35-86A-

* Download the `.BIO` file and copy it to a FAT32 formatted USB drive (`mkfs.fat -F32`)
* Plug in th USB drive, power on the NUC and press F7
* Follow the instructions to update the firmware

### Boot mode

If you don't want UEFI boot, disable UEFI boot in the NUC's BIOS. If you don't disable UEFI boot, 
the Arch Linux installation media will boot a system in UEFI mode. You have to make sure you can
boot from the SSD in UEFI mode as well which didn't work for me.

### Arch Linux ISO on an USB stick

`dd bs=4M if=/tmp/archlinux-2015.09.01-dual.iso of=/dev/sdf && sync`

