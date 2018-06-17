# Installing your software onto the Vim's eMMC
## Use U-Boot to create some basic GPT partitions on eMMC
As you can see below, I leave a 64MiB gap at the beginning of the eMMC.  This is
to make sure we don't overlap with U-Boot and it's environment.  We could
tighten this up with math and testing some other time.

Because of our issue where GPT and U-Boot are sharing some space on the eMMC
(U-Boot is actaully written over part of the GPT table) you will want to get
your partition sizes set right the first time.  Every time you change them,
GPT will overwrite U-boot and you'll need to reinstall U-Boot.
```
partitions="uuid_disk=0fa9206e-6bcc-11e8-9392-c3cf9b293922;name=BOOT,start=64MiB,size=100MiB,uuid=ae178b6a-6bcb-11e8-a042-63bcdf179117;name=ROOTFS,size=3900MiB,uuid=a4b19cee-6bd2-11e8-b662-1b9914e11155;name=ZPOOL,size=-,uuid=ebc99d44-71fd-11e8-b490-f72d8dbd6d7e"
gpt write mmc 2 ${partitions}
```

Before you issue a `reset`, make sure you have your U-Boot on SD Card ready so
you can reinstall U-Boot.

## Install u-boot onto eMMC:
1.  Copy the u-boot.bin* files into the small boot partition on either the SD
    card or eMMC
2.  Restart/reset the device with the SD card inserted and start pressing space
    repeatedly until you get the u-boot prompt.
3.  Load the u-boot.bin file into memory:
  * From SD Card (partition 1)
```
setenv initrd_loadaddr "0x13000000"
fatload mmc 0:1 ${initrd_loadaddr} u-boot.bin
```
  * From TFTP Server
```
TODO
```
  * Then
```
store rom_write ${initrd_loadaddr} 0 ${filesize}
```

Maybe instead
```
mmc write ${initrd_loadaddr} #unknown_address# ${filesize}
```

4.  Unplug the SD card and reboot
```
reset
```
5.  If all went well it either booted the OS on eMMC or booted to a u-boot
    prompt

## Install a root filesystem
### Use a Root filesystem hosted on an NFS server
TODO

### Install a Root filesystem from a DD image on your TFTP server
https://www.reddit.com/r/commandline/comments/5psivn/piping_tftp_to_dd/

This requires a running Linux OS though.  Can we do something similar from
within u-boot itself?
```
curl tftp://172.30.0.2/ubuntu-minimal_18.04-1G_ROOTFS.img | \
  sudo dd of=/dev/disk/by-partlabel/ROOTFS
```
This method is a bit slow.  40 minutes over 100Mbit/s LAN for a 1GiB image.  
Should also use something like "| tee >(md5sum)" to validate the image that was
downloaded.
```
curl tftp://172.30.0.2/ubuntu-minimal_18.04-1G_ROOTFS.img |\
  tee >(md5sum) | \
  sudo dd of=/dev/disk/by-partlabel/ROOTFS
```

Expand the filesytem you just wrote so that it fills the entire partition
```
sudo resize2fs /dev/disk/by-partlabel/ROOTFS
```

Get your FIT image (kernel, initramfs and DTB) from the TFTP server:
```
mkdir /mnt/boot
mount /dev/disk/by-partlabel/BOOT /mnt/boot
cd /mnt/boot
sudo curl tftp://172.30.0.2/image-4.16.13.itb --output image-4.16.13.itb
ln -s image-4.16.13.itb image.itb
```

### Install a Root filesystem from a DD image on your build machine, over SSH
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

If all went well then this should have built any DKMS modules for your new
kernel so when you do boot with it later, ZFS (for example) will be ready and
working for you.

## Final U-Boot configuration
To make U-Boot load the FIT image from the BOOT partition and then boot it
automatically, boot into the U-Boot prompt and:
```
setenv bootcmd "ext4load mmc 2:1 ${kernel_loadaddr} image.itb; bootm ${kernel_loadaddr}"
saveenv
reset
```
If all went well, you will now boot into Linux automatically without
interaction.

## Extra kernel command line arguments
As mentioned elsewhere, the "gpt" argument forces the kernel to read the
secondary GPT header if the primary is corrupt (i.e. slightly overwritten by
U-Boot).

I also use LXD/LXC containers and snaps so I need AppArmor so I add
"apparmor=1 security=apparmor"

My complete bootargs looks like this: TODO

## Installing kernel updates and creating updated boot images
Keeping your kernel up-to-date isn't as simple as it is with Ubuntu on x86,
obviously since you need to build it yourself.  After it's built and you have
installed the .deb files, you still need to update the FIT image.

This can probably be scripted and I'm reasonably sure that you can hook into
the update-initramfs tool to finish this process automatically but I haven't
worked on this element yet.
