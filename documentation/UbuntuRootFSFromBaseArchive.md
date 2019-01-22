# Ubuntu root filesystem from a "base" archive
## Initial Build
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

TODO: F2FS or JFFS2 may be better choices for the ROOTFS as they balance writes
amongst blocks to optimise block lifespan.  This is not so much an issue on SSD
storage where the SSDs controller balances the writes but controllerless devices
such as eMMC and SD Cards don't have these algorithms built in.  That said, I'm
going to use ZFS on the data volume which will probably thrash the device to
death in short order.

**Note:** If you use F2FS for your ROOTFS, you need to remove the
"rootflags=data=writeback" from "bootargs" in U-Boot.

Nilfs2 ran out of space before I got the ~300MiB of OS packages installed.  It
requires a constantly running garbage collector service in order to not run out
of space but apparently you can't trust the GC to free up space before you run
out.  Even though is seems like a really interesting concept, I'm disregarding
Nilfs2 for now.

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
useless without adding at least systemd?  I just installed "ubuntu-minimal" and
"systemd" and got a system that booted.

Optional: Chroot into our root file system and make customisations<br/>
Use the process detailed on the [Setup an Arm64 Chroot on an X86 build machine (Ubuntu/Debian)](SetupArm64ChrootOnX86_64.md)
page.

I would install, at minimum:
```
apt --no-install-recommends --no-install-suggests install ifupdown net-tools \
  udev sudo ssh dialog openssh-server ubuntu-minimal systemd vim.tiny dosfstools isc-dhcp-common
```

To make vim.tiny work as the default vim:
```
update-alternatives --install /usr/bin/vim vim /usr/bin/vim.tiny 100
```

If you're not super worried about making your image as small as possible you
could just install "ubuntu-server", even if you end up with a little more stuff
you might not need strictly need.

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

If you want to drop in a basic NetPlan coonfiguration:
```
echo -e "network:\n  version: 2\n  renderer: networkd\n  ethernets:\n    eth0:\n      dhcp4: true" >/etc/netplan/defaul
t.yaml
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

TODO: Wifi firmware from here: https://github.com/khadas/fenix/tree/master/archives/hwpacks/wlan-firmware/brcm
```
mkdir -p /lib/firmware/brcm/
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
sudo hostnamectl set-hostname "some.host.name"

### Generate unique SSH host keys
TODO
