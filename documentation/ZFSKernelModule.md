# ZFS Kernel Module
Regardless of the system you're working with, needing a lot of build tools on your system that exist only to build the ZFS kernel module whenever the kernel is updated is a pain.  It's additional packages with additional potential security vulnrabilities.  It's space consumed, which is especially important if your working with an SBC with limited (and maybe slow) local storage.

You can work around this need by building the ZFS kernel modules on another machine (even if it's a different architecture) and packaging them up in a nice .deb.

## Requirements
### Chroot (optional)
I will be using a chroot on my laptop to build modules for an ARM64 Single Board Computer.  All of the build dependencies are installed into the chroot.  When I'm done I could simply delete the chroot and have almost no trace left on my build machine.  Nice and tidy.

TODO: install qemu dependency

To create the chroot, download.... TODO

TODO: bind mounts

Then you can chroot into your new environment:
TODO

### Required packages
```
apt install python3-distutils dh-autoreconf tzdata
```

### Avoid DKMS automatically attempting to build all modules when you install a new kernel
```
rm  /etc/kernel/postinst.d/dkms
```

### Kernel and Headers
You need to install the kernel and headers for the kernel you will be building modules for

TODO

### Download and prepare the ZFS source
```
ZFSVERSION="0.8.0"
KERNELVERSION="4.20.0"
echo -e "#!/bin/sh\ncp \"$@\"" >zfs-${ZFSVERSION}/cp
chmod +x zfs-${ZFSVERSION}/cp
cd zfs-${ZFSVERSION}/scripts/
./dkms.mkconf -n zfs -v ${ZFSVERSION} -c kernel -f ../dkms.conf
cd ..
./autogen.sh
```

## Build the module
```
dkms build -m zfs -v ${ZFSVERSION} -k ${KERNELVERSION}
```

## Package the module
```
dkms mkbmdeb -m zfs -v ${ZFSVERSION} -k ${KERNELVERSION}
```

## Install the module on the target machine

### Installation issue
```
dpkg: dependency problems prevent configuration of zfs-modules-4.20.3:
 zfs-modules-4.20.3 depends on linux-image-4.20.3; however:
  Package linux-image-4.20.3 is not installed.

dpkg: error processing package zfs-modules-4.20.3 (--install):
 dependency problems - leaving unconfigured
Errors were encountered while processing:
 zfs-modules-4.20.3
```
At some point recently the Linux kernel package went from being named
linux-image-<version> to just linux-image.  Either DKMS or ZFS needs to update
how they do dependencies.  In the mean time:
```
mkdir linux-image-4.20.3
cd linux-image-4.20.3
dpkg-deb -R . /path/to/linux-image_4.20.3-2_arm64.deb
vim DEBIAN/control
```
Change:
```
Package: linux-image
```
to
```
Package: linux-image-4.20.3
```

And create a new .deb:
```
dpkg-deb -b . ../linux-image_4.20.3-2_arm64_fixed.deb
```

Uninstall the old package and install the new one:
```
sudo dpkg -r linux-image
sudo dpkg -i ../linux-image_4.20.3-2_arm64_fixed.deb
```

Now try installing the ZFS modules .deb again.

## Keeping up to date
The biggest issue with this process is that there is no automatic way have all of this happen each time there is a kernel update.  Can we come up with something?
