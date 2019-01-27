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
partitions="uuid_disk=0fa9206e-6bcc-11e8-9392-c3cf9b293922;name=ROOTFS,start=64MiB,size=4000MiB,uuid=a4b19cee-6bd2-11e8-b662-1b9914e11155;name=ZPOOL,size=-,uuid=ebc99d44-71fd-11e8-b490-f72d8dbd6d7e"
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

## Set a persistant MAC address for the NIC
```
setenv ethaddr <generate a unique mac address and put it here>
saveenv
```

## Install a root filesystem
### Use a Root filesystem hosted on an NFS server
TODO

### Install a Root filesystem from a DD image on your TFTP server from within U-Boot (experimental)

Get the ROOTFS image into memory.  Obviously this will only work if the image is smaller than the VIM's total memory.  I stick to 1GiB images so I should be
OK(?).
```
${tftpcmd} ${initrd_loadaddr} ubuntu-minimal_18.04_900M-f2fs-ROOTFS.img
```
This seemed faster than the curl download from the same server.

We need to know how many "blocks" the image is.  ${filesize} in bytes.
```
mmc dev 2
mmc info
```
```
Device: mmc@74000
Manufacturer ID: 15
OEM: 100
Name: AWPD3
Bus Speed: 52000000
Mode : MMC High Speed (52MHz)
Rd Block Len: 512
MMC version 5.0
High Capacity: Yes
Capacity: 14.6 GiB
Bus Width: 8-bit
Erase Group Size: 512 KiB
HC WP Group Size: 8 MiB
User Capacity: 14.6 GiB WRREL
Boot Capacity: 4 MiB ENH
RPMB Capacity: 4 MiB ENH
```
So, our blocks are 512b.  We need to divide ${filesize} by 512.  To complicate
things, ${filesize} is in hex and the block size is decimal.  512 = 0x200 so we
want to do (0x${filesize}/0x200).  I don't know how to do math in the U-Boot
shell so I'm just going to use Gnome Calculator in Programmer mode.

My ~500MiB image file comes to: 0xF9800

Work out where to write it.
```
mmc dev 2
mmc part
```
```
Partition Map for MMC device 2  --   Partition Type: EFI

GUID Partition Table Header signature is wrong: 0x6791EB71709C9E05 != 0x5452415020494645
part_print_efi: *** ERROR: Invalid GPT ***
part_print_efi: ***        Using Backup GPT ***
Part    Start LBA       End LBA         Name
        Attributes
        Type GUID
        Partition GUID
  1     0x00020000      0x007effff      "ROOTFS"
        attrs:  0x0000000000000000
        type:   ebd0a0a2-b9e5-4433-87c0-68b6b72699c7
        type:   data
        guid:   a4b19cee-6bd2-11e8-b662-1b9914e11155
  2     0x007f0000      0x01d1efde      "ZPOOL"
        attrs:  0x0000000000000000
        type:   ebd0a0a2-b9e5-4433-87c0-68b6b72699c7
        type:   data
        guid:   ebc99d44-71fd-11e8-b490-f72d8dbd6d7e
```
So, my rootfs partition starts at "0x00020000"
```
mmc write ${initrd_loadaddr} 0x00020000 0xF9800
```

### Install a Root filesystem from a DD image on your TFTP server from a running Linux OS
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
sudo apt install build-essential libtool autoconf bison flex libssl-dev
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
