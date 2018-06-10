# Ubuntu root filesystem from a "base" archive
This provides the slimmest root filesytem that is completely vanilla.  No bells,
no whistles.  You do get a little more freedom with being able to follow
Ubuntu's release schedule rather than waiting for Khadas to release their
version.  You also get to support yourself if Khadas stop providing up-top-date
images.

Other Linux distributions may also provide a similar "base" archive.  The
following instructions should work the same for any such distribution.

Start out by creating an empty 1GiB disk image file:
```
dd if=/dev/zero of=ubuntu-18.04-1G.img bs=1M count=1024
```

Now partition the image file as if it was a real block device:
```
sudo parted ubuntu-18.04-1G.img
```
We are going to create two partitions.  One small (100MiB) fat32 partition that
we might put u-boot binaries and kernels on and a second that will be our root
filesystem.
```
(parted) mklabel msdos
(parted) mkpart
Partition type?  primary/extended? p                                      
File system type?  [ext2]? fat16                                          
Start? 2048s                                                              
End? 100M
(parted) mkpart                                                           
Partition type?  primary/extended? p                                      
File system type?  [ext2]?                                                
Start? 100M                                                               
End? -1s
quit
```

Use our system's loopback mechanism to have it present our image file as a block
device:
```
sudo losetup -P /dev/loop101 ubuntu-18.04-1G.img
```

Format the partitions we created
```
sudo mkfs.fat -F 16 -n BOOT /dev/loop101p1
sudo mkfs.ext4 -L ROOTFS -m 0 /dev/loop101p2
```

Mount the root filesystem partition
```
sudo mkdir /mnt/target
sudo mount /dev/loop101p2 /mnt/target
```

Extract the Ubuntu (or other) base filesystem into our root filesystem
```
cd /mnt/target/
sudo tar -xzf ~/Downloads/bionic-base-arm64.tar.gz
```

**Note**: The "base" archive doesn't seem to contain any type of init making it
useless without adding at least systemd?  I just installed "ubuntu-server" and
got a system that booted.  TODO: Work out some minimum packages to get an init
or suggest "init=/bin/bash" on the kernel command line for first boot.

Optional: Chroot into our root file system and make customisations<br/>
Use the process detailed on the [Setup an Arm64 Chroot on an X86 build machine (Ubuntu/Debian)](SetupArm64ChrootOnX86_64.md)
page.

I would install, at minimum:
```
apt --no-install-recommends --no-install-suggests install ifupdown net-tools \
  udev sudo ssh dialog openssh-server
```

You still won't have an init system so maybe add "systemd"
```
TODO
```

If you're not super worried about making your image as small as possible you
could just install "ubuntu-minimal" or, for a little more stuff you might not
need strictly need "ubuntu-server"

I'm slowly moving towards having NetworkManager manage network connections for
me rather than needing to mess about with configurations manually.
NetworkManager only adds another 16MiB on top of "ubuntu-minimal".

Install the kernel and headers packages
```
TODO
```

Create a user
```
useradd -G sudo -m -s /bin/bash ubuntu
echo ubuntu:ubuntu | chpasswd
```

If you plan to use NetworkManager to manage the ethernet connection:
```
rm /usr/lib/NetworkManager/conf.d/10-globally-managed-devices.conf
```

If you're going to try to pull the root filesystem image from a TFTP server
```
apt install curl
```

Create a /etc/fstab
```
TODO
```

Setup timezone and locale (replace "en_AU.UTF-8" with your desired locale):
```
dpkg-reconfigure tzdata
apt-get install --reinstall locales
locale-gen en_AU.UTF-8
update-locale --reset LANG=en_AU.UTF-8
```

Unmount the partitions within the image
```
sudo umount $(mount |grep /mnt/target/ | awk '{print $3}')
sudo umount /mnt/target
```

If you need just the root filesystem, not the partitions (deploy from TFTP to
eMMC), etc. you can extract it like so:
```
sudo dd if=/dev/loop101p2 of=ubuntu-minimal_18.04-1G_ROOTFS.img bs=4k
```

Remove the pseudo-block device
```
sudo losetup -d /dev/loop101
```

Add u-boot to the SD card image (optional):
The Vim will only look for u-boot on the SD card if it doesn't have one on MMC.
You can add u-boot to the SD card image so you have a fallback u-boot if needed.
```
TODO
```

## First boot
### Set hostname
TODO

### Generate unique SSH host keys
TODO

### Install kernel packages and create boot images

### U-Boot load kernel and initrd from root filesystem
**Note**: My version of u-boot clearly has ext4 issues so I'm not using the
below until I work out an updated u-boot.
```
ext4load mmc 1:5 ${dtb_mem_addr} kvim_linux-${kver}.dtb;fdt addr ${dtb_mem_addr}
ext4load mmc 1:5 ${kernel_loadaddr} uImage-${kver}
ext4load mmc 1:5 ${initrd_loadaddr} uInitrd-4.9.40
bootm ${kernel_loadaddr} ${initrd_loadaddr} ${dtb_mem_addr}
```

### U-Boot load kernel and initrd from seperate filesystem
Since the above isn't working as I'd like, I'm going to try creating a fat32
filesystem in the partition that's supposed to be for the boot ramdisk image and
drop the needed files in there.

We will need to keep in mind that when we update our kernel, etc. we need to
update the files in this partition.
```
sudo mkfs.fat -F 16 -n RAMDISK /dev/ramdisk
sudo mkdir /boot/ramdisk
sudo mount /dev/ramdisk /boot/ramdisk
sudo cp /boot/uImage-4.9.40 /boot/uInitrd-4.9.40 /boot/kvim_linux-4.9.40.dtb \
  /boot/ramdisk
```
Add the following to /etc/fstab (the "nofail" and "-systemd.device-timeout=1"
stop the system going into a panic if the filesystem cannot be found or will not
mount cleanly):
```
LABEL=RAMDISK /boot/ramdisk       vfat    ro,umask=0077,nofail,x-systemd.device-timeout=1      0       1
```

Reboot and from the u-boot prompt:
```
fatload mmc 1:4 ${dtb_mem_addr} kvim_linux-${kver}.dtb;fdt addr ${dtb_mem_addr}
fatload mmc 1:4 ${kernel_loadaddr} uImage-${kver}
fatload mmc 1:4 ${initrd_loadaddr} uInitrd-${kver}
bootm ${kernel_loadaddr} ${initrd_loadaddr} ${dtb_mem_addr}
```

If that works, make it persistant as u-boot's default boot command:
```
setenv bootcmd "fatload mmc 1:4 ${dtb_mem_addr} kvim_linux-${kver}.dtb;fdt addr ${dtb_mem_addr};fatload mmc 1:4 ${kernel_loadaddr} uImage-${kver};fatload mmc 1:4 ${initrd_loadaddr} uInitrd-${kver};bootm ${kernel_loadaddr} ${initrd_loadaddr} ${dtb_mem_addr}"
saveenv
reset
```

If all goes well the Vim will load the kernel, initrd and dtb from the small
fat32 partition and boot the OS without any manual steps.
