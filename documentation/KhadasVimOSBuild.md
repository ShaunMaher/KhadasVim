# Building an OS for the Khadas Vim (and possibly other Arm64 SBCs)
## References:
http://forum.khadas.com/t/port-linux-mainline-on-khadas-vim/848

## Create a working directory
Set yourself up a directory to work in.  Give yourself at least 15GiB and if
possible, use local SSD storage.  The compiling stages do quite a bit of I/O so
slow storage slows down everything.

```
mkdir KhadasVim
cd KhadasVim
```

## Tip: Speedup
The Linux sources are going to take some time to download from github.com.  Jump
ahead to the "Build Kernel" section below and get the "git clone" started in
another tab/window/terminal so it will be ready by the time you need it.

## Install the cross-compiling tool chain
**Note**: I used the 5.x toolchain initially but found that none of the kernels
I built actaully booted.  There were no errors during the build process.  Maybe
I was still doing something wrong.  I did try with the latest toolchain (7.x at
the time of writing) but the build failed.
```
mkdir toolchains
cd toolchains
```
Browse to https://releases.linaro.org/components/toolchain/binaries/latest-4/aarch64-linux-gnu/
and get the URL for the latest package for the 5.x branch (latest supported
version)
```
wget <URL>
tar -xJf gcc-linaro-4.*.tar.xz
PATH=$PATH:TODO
```

## Khadas utils
```
git clone https://github.com/khadas/utils.git
```

## Build u-boot
**Note:** Don't try to do this on the Vim.  It makes use of a x86_64 binary.

```
sudo apt install bc build-essential libtool gcc-arm-none-eabi \
  binutils-arm-none-eabi lib32ncurses5 python-minimal lib32stdc++6 \
  libncurses5-dev libncursesw5-dev
git clone https://github.com/khadas/u-boot -b ubuntu
make kvim_defconfig
make -j8 CROSS_COMPILE=aarch64-linux-gnu-
```

## Build kernel
I setup two copies of the kernel source.  One (linux.orig) is completely
untouched upstream source the other (linux) gets local modifications and
patches, etc.  This can be handy for creating patches or just when you have made
an unfixable mess and want to start over but don't want to wait for the download
again.

When you're building a kernel from source, the builder assumes that you're doing
so for debugging purposes and it kindly makes the kernel identify itself by the
git revision number.  This just makes things difficult when you want to build
modules later.  I can't remember exactly what issue I had, something about
headers and modules failing to load because the kernel had a different version
(because one had the git revision on the end and the built modules didn't).  It
was a pain.  The `touch .scmversion` lines (multiple) prevent this behaviour.

```
sudo apt install bc build-essential libtool gcc-arm-none-eabi \
  binutils-arm-none-eabi libssl-dev
mkdir linux.orig
cd linux.orig
```
For the Khadas kernel sources:
```
git clone https://github.com/khadas/linux -b ubuntu-4.9 .
```

For the mainline sources (assuming you want to build v4.15):
```
git clone https://github.com/torvalds/linux.git .
git checkout tags/v4.15
```
```
cd ..
mkdir linux
cp -ra linux.orig/* linux/
```
Optional: Apply my patches:
```
TODO: Um... What patches?
```

Set some environment variables to maek things easier later
For the Khadas kernel:
```
DEFCONFIG="kvim_defconfig"
DTB="kvim_linux"
```

For the mainline kernel with Khadas patches:
```
DEFCONFIG="kvim_defconfig"
DTB="meson-gxl-s905x-khadas-vim"
```

For the mainline kernel without patches:
```
DEFCONFIG="defconfig"
DTB="meson-gxl-s905x-khadas-vim"
```

For either kernel:
```
make ARCH=arm64 ${DEFCONFIG}
touch .scmversion
make -j3 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image ${DTB}.dtb modules
KERNELVERSION=$(make kernelversion)
tar -czf ../packages/linux-headers-extra-${KERNELVERSION}.tar.gz \
  arch/arm64/kernel/vdso/* security/selinux/include/* tools/include/tools/*
cp arch/arm64/boot/dts/amlogic/${DTB}.dtb \
  ../tmp/mkbootimg/${DTB}-${KERNELVERSION}.dtb
cp arch/arm64/boot/Image ../tmp/mkbootimg/Image-${KERNELVERSION}
```
What does this next step achieve?  We never go and fetch these files
```
mkdir -p /tmp/kvimX
make -j3 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
  INSTALL_MOD_PATH=/tmp/kvim7/ modules_install
```
```
make clean
touch .scmversion
make -j3 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- deb-pkg
```

Tidy up a little:
```
mv ../linux-*_arm64.deb ../linux-*_arm64.changes \
  ../linux-${KERNELVERSION}*.tar.gz ../linux-${KERNELVERSION}*.dsc ../packages/
```

## Future Experiment: Build kernel with ZFS built in
http://www.linuxquestions.org/questions/linux-fromeson-gxl-s905x-khadas-vimm-scratch-13/%5Bhow-to%5D-add-zfs-to-the-linux-kernel-4175514510/

## Initrd: Using Device Emulation
If you already have a running device, the simplist way to produce the boot image
and initrd is to use said device.  Alternativly, you can emulate an arm64 device
on your build machine.

[Setup an Arm64 Chroot on an X86 build machine (Ubuntu/Debian)](SetupArm64ChrootOnX86_64.md)

Copy some of the generated packages and files into the chroot
```
cp packages/linux-image-${KERNELVERSION}*.deb \
  packages/linux-headers-${KERNELVERSION}*.deb \
  packages/linux-headers-extra-${KERNELVERSION}*.tar.gz "${ROOTFSPATH}/tmp/"
```

Launch the chroot (with a trick so that the KERNELVERSION environment variable
follows us into the chroot)
```
sudo chroot "${ROOTFSPATH}" bash -c "KERNELVERSION=${KERNELVERSION} bash"
```

Install kernel and headers
```
dpkg -i /tmp/linux-headers*.deb
cd /usr/src/linux-headers-${KERNELVERSION}
tar -xzf /tmp/linux-headers-extra-${KERNELVERSION}.tar.gz
make modules_prepare
dpkg -i /tmp/linux-image-*.deb
```
If you get something like:
```
depmod: WARNING: could not open /lib/modules/4.15.0/modules.order: No such file or directory
depmod: WARNING: could not open /lib/modules/4.15.0/modules.builtin: No such file or directory
depmod: WARNING: could not open /var/tmp/mkinitramfs_LcLam8/lib/modules/4.15.0/modules.order: No such file or directory
depmod: WARNING: could not open /var/tmp/mkinitramfs_LcLam8/lib/modules/4.15.0/modules.builtin: No such file or directory
```
Then your modules directory may have had "-dirty" appended to it's name
(because files in the kernel build tree were modified/patched?).  Try this:
```
cd /lib/modules/
mv 4.15.0 4.15.0.orig
ln -s 4.15.0-dirty 4.15.0
```
And then try installing the package again:
```
dpkg -i /tmp/linux-image-*.deb
```

Make the initramfs:
```
update-initramfs -k ${KERNELVERSION} -c
```

We're done inside the chroot for now:
```
exit
```

Move to the top level of your build environment and then:
```
cp "${ROOTFSPATH}/boot/initrd.img-${KERNELVERSION}" tmp/mkbootimg/
```

## Initrd: Using the device itself
I had a running Vim with the stock 4.9.26 kernel running so I installed the .deb
for the kernel I compiled above and then:
```
sudo update-initramfs -k 4.9.26-g8bc293d -c
```
Then I took the resulting /boot/initrd.img-4.9.26-g8bc293d

## Make the boot.img
I'm in two minds as to what to do here.  The upstream method is to create a
boot.img that gets written to it's own "ramdisk" partition on the emmc.  I would
prefer an old school kernel and initrd in a /boot directory.  The latter is
possible but I can't get u-boot to read the ext4 file system right now.  Intil I
resolve this issue, I'm using the ramdisk method.

```
../../utils/mkbootimg --kernel Image-${KERNELVERSION} --ramdisk \
  initrd.img-${KERNELVERSION} -o boot-${KERNELVERSION}.img
```

## Make a ROOTFS using an Ubuntu "base"
This provides the slimmest root filesytem that is completely vanilla.  No bells,
no whistles.  You do get a little more freedom with being able to follow
Ubuntu's release schedule rather than waiting for Khadas to release their
version.  You also get to support yourself if Khadas stop providing up-top-date
images.

TODO

## Use an existing ROOTFS: Provided by Khadas
The Khadas images have a few extra goodies, such as making the Vim's red LED
"breathe" and "pulse" depending on system load, etc.

TODO

## Testing
This step is not mandatory but if you find yourself making multiple attempts to
get everything just right, you might find that moving your SD card between build
machine and Vim gets tedious.  You can setup a TFTP server on your network and
test each kernel as you build it without using the SD card.

[TestingViaTFTP.md](TestingViaTFTP.md)

I would be SCPing the resulting files onto my TFTP server like this:
```
TFTPSERVER=172.30.0.1
scp tmp/mkbootimg/boot-${KERNELVERSION}.img root@$TFTPSERVER:/tftpboot/
scp tmp/mkbootimg/${DTB}-${KERNELVERSION}.dtb root@$TFTPSERVER:/tftpboot/
```

## Installation
Head over to the [InstallOntoVim.md](InstallOntoVim.md) document for information
regarding the installation of your newly built OS.
