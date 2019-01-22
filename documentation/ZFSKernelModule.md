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
dkms build -m ${ZFSVERSION} -v 0.8.0 -k ${KERNELVERSION}
```

## Package the module
```
dkms mkbmdeb -m zfs -v ${ZFSVERSION} -k ${KERNELVERSION}
```

## Install the module on the target machine

## Keeping up to date
The biggest issue with this process is that there is no automatic way have all of this happen each time there is a kernel update.  Can we come up with something?
