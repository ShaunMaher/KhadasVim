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
sudo losetup -P /dev/loop24 ubuntu-18.04-1G.img
```

Format the partitions we created
```
sudo mkfs.fat -F 16 -n BOOT /dev/loop24p1
sudo mkfs.ext4 -L ROOTFS -m 0 /dev/loop24p2
```

Mount the root filesystem partition
```
sudo mkdir /mnt/target
sudo mount /dev/loop24p2 /mnt/target
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
apt --no-install-recommends install ifupdown net-tools udev sudo ssh dialog openssh-server
```

Unmount the partitions within the image
```
sudo umount -l /mnt/target
```

Remove the pseudo-block device
```
sudo losetup -d /dev/loop24
```
