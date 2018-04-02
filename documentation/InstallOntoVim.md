## Install u-boot onto eMMC:
1.  Copy the u-boot.bin* files into the small boot partition on either the SD
    card or eMMC
2.  Restart/reset the device with the SD card inserted and start pressing space
    repeatedly until you get the u-boot prompt.
3.  Load the u-boot.bin file into memory:
```
setenv initrd_loadaddr "0x13000000"
fatload mmc 0:1 ${initrd_loadaddr} u-boot.bin
store rom_write ${initrd_loadaddr} 0 100000
```
4.  Unplug the SD card and reboot
      reset
5.  If all went well it either booted the OS of eMMC or booted to a u-boot
    prompt

## Install the device tree (.dtb file) to emmc
```
setenv initrd_loadaddr "0x13000000"
fatload mmc 0:1 ${initrd_loadaddr} kvim.dtb
fatload mmc 0:1 ${initrd_loadaddr} meson-gxl-s905x-khadas-vim.dtb
store dtb write ${initrd_loadaddr}
reset
```

## Install the boot.img to eMMC
* Copy the boot.img file to an SD card that is formatted with fat32
* Connect with the serial adaptor to the Vim
* Powerup the Vim and immediately press CRTL+C to interupt the boot sequence
```
sdc_update ramdisk boot.img
reset
```

## Install the Root filesystem
* DD the rootfs into a partition on an SD card and boot the Vim with the card
inserted.  It should find the partition labled ROOTFS on the SD card and use it
as a root fs.  From here you can DD a rootfs image into the eMMC.

```
dd if=rootfs.PARTITION | ssh -t ubuntu@172.30.1.242 sudo dd of=/dev/rootfs
```

## Install kernel headers
**Note:** If you uninstalled any of the build tools and you use the zfs DKMS
module, reinstall the following:
```
sudo apt install build-essential libtool autoconf
```

**Note:** No shortcuts.  Install the debs one at a time in the following order.
```
sudo dpkg -i linux-headers-blah
cd /usr/src/linux-headers-4.9.26-blah
sudo tar -xzf linux-headers-extra.tar.gz
sudo make modules_prepare
sudo dpkg -i linux-image-blah
```

If all went well then this should have built the DKMS modules for your new
kernel so when you do boot with it later, zfs will be ready and working for you.
