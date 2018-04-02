## Speedup
The Linux sources are going to take some time to download from github.com.  Jump
ahead to the "Build Kernel" section below and get the "git clone" started in
another tab/window/terminal so it will be ready by the time you need it.

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

## Install the cross-compiling tool chain
```
mkdir toolchains
cd toolchains
```
Browse to https://releases.linaro.org/components/toolchain/binaries/latest-5/aarch64-linux-gnu/
and get the URL for the latest package for the 5.x branch (latest supported
version)
```
wget <URL>
tar -xJf gcc-linaro-5.*.tar.xz
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
```
cd linux
touch .scmversion
```
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
tar -czf ../linux-headers-extra.tar.gz arch/arm64/kernel/vdso/* \
  security/selinux/include/* tools/include/tools/*
cp arch/arm64/boot/dts/amlogic/${DTB}.dtb ../tmp/mkbootimg
cp arch/arm64/boot/Image ../tmp/mkbootimg
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

## Future Experiment: Build kernel with ZFS built in
http://www.linuxquestions.org/questions/linux-fromeson-gxl-s905x-khadas-vimm-scratch-13/%5Bhow-to%5D-add-zfs-to-the-linux-kernel-4175514510/

## Install the device tree (.dtb file) to emmc
```
setenv initrd_loadaddr "0x13000000"
fatload mmc 0:1 ${initrd_loadaddr} kvim.dtb
fatload mmc 0:1 ${initrd_loadaddr} meson-gxl-s905x-khadas-vim.dtb
store dtb write ${initrd_loadaddr}
reset
```
## Initrd: Using Device Emulation
If you already have a running device, the simplist way to produce the boot image
and initrd is to use said device.  Alternativly, you can emulate an arm64 device
on your build machine.

Install the Qemu binaries:
```
sudo apt-get install qemu qemu-user-static binfmt-support debootstrap
```

Create a root filesystem:
```
cd roots
mkdir ubuntu-base-18.04
cd ubuntu-base-18.04
tar -xzf /path/to/bionic-base-arm64.tar.gz
```

Set an environment variable to to the path of the root to save time later:
```
ROOTFSPATH=/home/shaun/Documents/KhadasVim/roots/ubuntu-base-18.04
```

Copy in the Qemu emulator binary
```
sudo cp -a /usr/bin/qemu-aarch64-static "${ROOTFSPATH}/usr/bin"
```

Mount some loactions on your build machine into the root:
```
sudo mount -o bind /proc "${ROOTFSPATH}/proc"
sudo mount -o bind /sys "${ROOTFSPATH}/sys"
sudo mount -o bind /dev "${ROOTFSPATH}/dev"
sudo mount -o bimake -j3 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image ${DTB}.dtb modules
nd /dev/pts "${ROOTFSPATH}/dev/pts"
```

Start a shell inside the root (bash has a bug that should be resolved shortly):
```
sudo chroot "${ROOTFSPATH}"
uname -a
```
```
Linux e6540 4.15.0-13-generic #14-Ubuntu SMP Sat Mar 17 13:44:27 UTC 2018 aarch64 aarch64 aarch64 GNU/Linux
```
Note the 'aarch64' near the end.

Configure a nameserver:
```
echo "nameserver 8.8.8.8" >>/etc/resolv.conf
```

Update all packages:
```
apt update
apt -y dist-upgrade
```

Install some prerequsite packages for networking, editing and creating
initramfs:
```
apt-get install ifupdown net-tools udev vim sudo ssh initramfs-tools dialog \
  build-essential libssl-dev
```

Install kernel and headers
```
dpkg -i /tmp/linux-headers-*.deb
cd /usr/src/linux-headers-*
tar -xzf /tmp/linux-headers-extra.tar.gz
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
sudo update-initramfs -k 4.15.0 -c
```

We're done inside the chroot for now:
```
exit
```

Move to the top level of your build environment and then:
```
cd tmp/mkbootimg
cp "${ROOTFSPATH}/boot/initrd.img-4.15.0" .
../../utils/mkbootimg --kernel Image --ramdisk initrd.img-4.15.0 -o boot.img
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
../../utils/mkbootimg --kernel Image --ramdisk initrd.img-4.15.0 -o boot.img
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

## update-initramfs always fails with an error referring to flash-kernel
```
sudo vim /etc/initramfs/post-update.d/flash-kernel
```
Add the following line near the top of the script, after the #!
```
exit 0
```
