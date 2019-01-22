# Building an OS for the Khadas Vim (and possibly other Arm64 SBCs)
## References:
  * http://forum.khadas.com/t/port-linux-mainline-on-khadas-vim/848
  * https://github.com/umiddelb/armhf/wiki/How-To-compile-a-custom-Linux-kernel-for-your-ARM-device

## Caveats
#### Missing u-boot command limits ability to update u-boot in the future
The mainline u-boot doesn't have a "store rom_write" command so I'm not sure
how to have an a mainline U-Boot install U-Boot to eMMC.  For now I'm going
to keep a Khadas U-Boot handy so it can be put onto a SD card, chain booted
and then used to write U-Boot to eMMC.  I'm fairly sure that "write mmc ..."
can do the job but I'm not sure where on the eMMC U-Boot should be written.

#### Cannot manipulate U-Boot environment from within Linux
I put the environment in an arbitrary location near the start of the eMMC and I
don't know how to configure u-boot-tools to this location, yet.

https://github.com/lentinj/u-boot/blob/master/tools/env/fw_env.config

#### MTD Partitions
I didn't end up utilising the MTD partitioning that the Khadas U-Boot and
Kernel use.  I just couldn't get the mainline U-Boot to recognise any
partitions.

There is one slight quirk.  U-Boot and the GPT partition table expect to be
in the same place (U-Boot is more than the 512b in LBA 0 and GPT uses LBA
1).  This can be worked around though because there are two GPT tables on
the storage.  One starting at LBA 1 and one starting at the last LBA (LBA
-1).  If you provide the additional Linux kernel argument "gpt" it will
search for and find the valid secondary GPT and make use of it.

#### Khadas forked kernel is quite stable
I have been using the Khadas fork of the 4.9 kernel for several months with
no issues.  I have modified my Vim by adding a 25x25x15mm heatsink on the
CPU which required cutting a square hole in the top of the case for the
heatsink to stick through.

## Automated method
Before you press forward on all of the manual steps below, you can use the
[Khadas fenix scripts](https://github.com/khadas/fenix) which do all the work
for you.

## Build machine
I did all builds on an x86_64 laptop running Ubuntu 18.04 Desktop.  I assume
that most of the commands below could be modified to run directly on the Vim or
another Arm64 machine but I'm starting from scratch with no operating Arm64
machine to use.  At minimum, the following tasks **require** an x86_64 machine:
  1.  "Build u-boot":  Somewhere in the process it makes use of an x86_64 binary
  2.  "Make the boot.img": the Khadas provided "mkbootimg" is an x86_64 binary.
    This issue can be worked around by not using a boot.img by sticking with the
    more traditional seperate kernel and initrd files ("mkimage" from
    "u-boot-tools" package).  

## Create a working directory
Set yourself up a directory to work in.  Give yourself at least 15GiB and if
possible, use local SSD storage.  The compiling stages do quite a bit of I/O so
slow storage slows down everything.

```
mkdir KhadasVim
cd KhadasVim
```

## Tip: Time saver
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
and get the URL for the latest package for the 4.x branch (latest supported
version)
```
wget <URL>
tar -xJf gcc-linaro-4.*.tar.xz
if [ "${ORIGPATH}" == "" ]; then export ORIGPATH=$PATH; fi
PATH=$(pwd)/../toolchains/gcc-linaro-4.9.4-2017.01-x86_64_aarch64-linux-gnu/bin:$ORIGPATH
```

While we're here, download a couple of toolchain versions we may also make use
of:
```
wget https://releases.linaro.org/components/toolchain/binaries/6.3-2017.02/aarch64-linux-gnu/gcc-linaro-6.3.1-2017.02-x86_64_aarch64-linux-gnu.tar.xz
wget https://releases.linaro.org/components/toolchain/binaries/7.2-2017.11/aarch64-elf/gcc-linaro-7.2.1-2017.11-x86_64_aarch64-elf.tar.xz

```

## Build U-Boot
#### Khadas fork
**Note:** Don't try to do this on the Vim.  It makes use of a x86_64 binary.

```
sudo apt install bc build-essential libtool gcc-arm-none-eabi \
  binutils-arm-none-eabi lib32ncurses5 python-minimal lib32stdc++6 \
  libncurses5-dev libncursesw5-dev
git clone https://github.com/khadas/u-boot -b ubuntu
make kvim_defconfig
make -j8 CROSS_COMPILE=aarch64-linux-gnu-
```

#### Mainline
**Notes**:
1.  As mentioned earlier, I have been unable to get this U-Boot to see MDT
partitions (even when they were manually added to the DTB).
2.  I'm also unable to get this U-Boot to boot the boot.img style files that I
used with the Khadas provided U-Boot.  I'll be using FIT images so this poses no issue.
3.  The method I used to store the environment on eMMC is a bit blunt and could
use refinement.
4.  Again, as mentioned earlier, I cannot get this U-Boot to install U-Boot to
eMMC properly.

This entire process is basically pulled directly from the Khasas [fenix](https://github.com/khadas/fenix) scripts.

We're going to use another toolchain for this build so:
```
cd toolchains
wget https://releases.linaro.org/components/toolchain/binaries/7.2-2017.11/aarch64-elf/gcc-linaro-7.2.1-2017.11-x86_64_aarch64-elf.tar.xz
tar -xJf gcc-linaro-7.2.1-2017.11-x86_64_aarch64-elf.tar.xz
if [ "${ORIGPATH}" == "" ]; then export ORIGPATH=$PATH; fi
PATH=$(pwd)/gcc-linaro-7.2.1-2017.11-x86_64_aarch64-elf/bin:$ORIGPATH
```

Fetch the source tree
```
mkdir u-boot
cd u-boot
wget https://github.com/u-boot/u-boot/archive/680a52c.tar.gz
tar --strip-components=1 -xzf *.tar.gz
```

Add the Khadas/amlogic firmware blobs.  We need to download just this directory
from github:
https://github.com/khadas/fenix/tree/master/packages/u-boot-mainline/fip.  To do
so there are a few options:
  * Use GitZip: http://kinolien.github.io/gitzip/
    * Ignore the request for a token.  I don't know what that's for but it works
      without it
    * Enter the GitHub URL above in the text field near the top right of the
      page
    * Click Download
  * Just "git clone" the entire tree and pull out the directory we need
  * svn trickery:
    https://stackoverflow.com/questions/7106012/download-a-single-folder-or-directory-from-a-github-repo

However you got the directory, put it in the root of the u-boot source
directory.

Now do the build:
```
make distclean
make khadas-vim_defconfig
```
At this point you might want to modify the default configuration slightly.
```
make menuconfig
```
I enable:
* Boot images -> Enable support for Android Boot Images
* Boot images -> Support Flattened Image Tree
* delay in seconds before automatically booting -> 10 (useful while still testing)
* Display information about the board during early start up
* Display information about the board during late start up
* Partition Types -> Enable support of GUID for partition type
* Command line interface -> Device access commands -> GPT (GUID Partition Table) command
* Command line interface -> Device access commands -> GPT Random UUID generation
* Environment -> Environment in an MMC device
* Command line interface -> Misc commands -> uuid, guid - generation of unique IDs
* Command line interface -> Shell prompt -> "kvim> "

The following extra configuration was added to the end of (just before the
`#endif /* __CONFIG_H */`  on the last line of the file):
include/configs/khadas-vim.h

```
#undef CONFIG_EXTRA_ENV_SETTINGS
#define CONFIG_EXTRA_ENV_SETTINGS \
        "fdt_addr_r=0x01000000\0" \
        "scriptaddr=0x1f000000\0" \
        "kernel_addr_r=0x01080000\0" \
        "pxefile_addr_r=0x01080000\0" \
        "ramdisk_addr_r=0x13000000\0" \
        MESON_FDTFILE_SETTING \
        BOOTENV \
        "serverip=172.30.0.2\0" \
        "kernel_loadaddr=0x11000000\0" \
        "initrd_loadaddr=0x13000000\0" \
        "tftpcmd=dhcp\0" \
        "bootargs=\"root=LABEL=ROOTFS rootflags=data=writeback rw no_console_suspend consoleblank=0 fsck.repair=yes net.ifnames=0 jtag=disable gpt\"\0"

#define CONFIG_SYS_MMC_ENV_DEV 2
#define CONFIG_ENV_OFFSET 0x800000
#define CONFIG_SYS_MAX_NAND_DEVICE 3
#define CONFIG_SYS_NAND_BASE_LIST   {0}
#define CONFIG_MTD_DEVICE y
```
The above causes U-Boot to store it's environment somewhere around 8-10MiB from
the beginning of the eMMC storage.  I could do math and make it closer to U-Boot
itself at the beginning of the storage but that's a task for another day.

```
make -j8 CROSS_COMPILE=aarch64-elf-
```

While the Khadas u-boot source does these final steps for us, we need to do them
manually:
```
cp u-boot.bin fip/bl33.bin
bash fip/blx_fix.sh fip/bl30.bin fip/zero_tmp fip/bl30_zero.bin fip/bl301.bin \
  fip/bl301_zero.bin fip/bl30_new.bin bl30
chmod +x fip/acs_tool.pyc
fip/acs_tool.pyc fip/bl2.bin fip/bl2_acs.bin fip/acs.bin 0
bash fip/blx_fix.sh fip/bl2_acs.bin fip/zero_tmp fip/bl2_zero.bin fip/bl21.bin \
  fip/bl21_zero.bin fip/bl2_new.bin bl2
chmod +x fip/aml_encrypt_gxl
fip/aml_encrypt_gxl --bl3enc --input fip/bl30_new.bin
fip/aml_encrypt_gxl --bl3enc --input fip/bl31.img
fip/aml_encrypt_gxl --bl3enc --input fip/bl33.bin
fip/aml_encrypt_gxl --bl2sig --input fip/bl2_new.bin --output fip/bl2.n.bin.sig
fip/aml_encrypt_gxl --bootmk --output fip/u-boot.bin --bl2 fip/bl2.n.bin.sig \
  --bl30 fip/bl30_new.bin.enc --bl31 fip/bl31.img.enc --bl33 fip/bl33.bin.enc
```

The resulting u-boot images are in fip/u-boot.bin*

## Build the Linux kernel
When you're building a kernel from source, the builder assumes that you're doing
so for debugging purposes and it kindly makes the kernel identify itself by the
git revision number.  This just makes things difficult when you want to build
modules later.  I can't remember exactly what issue I had, something about
headers and modules failing to load because the kernel had a different version
(because one had the git revision on the end and the built modules didn't).  It
was a pain.  The `touch .scmversion` lines (multiple) prevent this behaviour.

```
sudo apt install bc build-essential libtool gcc-arm-none-eabi \
  binutils-arm-none-eabi libssl-dev bison flex
mkdir linux.orig
cd linux.orig
```
For the Khadas kernel sources:
```
git clone https://github.com/khadas/linux -b ubuntu-4.9 .
git apply $(find ../patches/linux/khadas-4.9.40/ -mindepth 1 | sort)
```

For the mainline kernel as used/built by the fenix scripts, with Khadas/amlogic
patches:
```
wget http://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.20.tar.xz
tar --strip-components=1 -xJf linux-4.20.tar.xz
git apply $(find ../patches/linux/mainline-4.20/ -mindepth 1 | sort)
```

For the mainline sources (assuming you want to build v4.15):
```
git clone https://github.com/torvalds/linux.git .
git checkout tags/v4.15
```

Optional: Apply patches:
```
cd linux-4.16.13
find ../../patches/linux/mainline-4.16.13/ -maxdepth 1 -mindepth 1 | sort | while read PATCH; do patch -N -u --strip 1 -i "${PATCH}"; done
```

Set some environment variables to make things easier later
* For the Khadas kernel:
```
if [ "${ORIGPATH}" == "" ]; then export ORIGPATH=$PATH; fi
PATH=$(pwd)/../toolchains/gcc-linaro-4.9.4-2017.01-x86_64_aarch64-linux-gnu/bin:$ORIGPATH
DEFCONFIG="kvim_defconfig"
DTBRELPATH=""
DTB="kvim_linux"
```

* For the mainline kernel with Khadas patches:
```
if [ "${ORIGPATH}" == "" ]; then export ORIGPATH=$PATH; fi
PATH=$(pwd)/../toolchains/gcc-linaro-6.3.1-2017.02-x86_64_aarch64-linux-gnu/bin:$ORIGPATH
DEFCONFIG="defconfig"
DTBRELPATH="amlogic/"
DTB="meson-gxl-s905x-khadas-vim"
```

* For the mainline kernel without patches:
```
if [ "${ORIGPATH}" == "" ]; then export ORIGPATH=$PATH; fi
PATH=$(pwd)/../toolchains/gcc-linaro-6.3.1-2017.02-x86_64_aarch64-linux-gnu/bin:$ORIGPATH
DEFCONFIG="defconfig"
DTBRELPATH="amlogic/"
DTB="meson-gxl-s905x-khadas-vim"
```

TODO: Prevent "-dirty" suffix: https://stackoverflow.com/questions/25090803/linux-kernel-kernel-version-string-appended-with-either-or-dirty

For either kernel, you might want to manually select some features
```
make ARCH=arm64 ${DEFCONFIG}
make ARCH=arm64 menuconfig
```
**Important:** Make sure you load the existing config from .config before
making changes.

I enabled:
* File systems -> F2FS filesystem support
* File systems -> F2FS consistency checking feature
* File systems -> Miscellaneous filesystems -> (SquashFS) Include support for XZ compressed file systems (needed for Ubuntu snaps)
* File systems -> Miscellaneous filesystems -> (SquashFS) Decompressor parallelisation options (Use multiple decompressors for parallel I/O)
* File systems -> Miscellaneous filesystems -> Journalling Flash File System v2 (JFFS2) support
* File systems -> Miscellaneous filesystems -> JFFS2 XATTR support
* File systems -> Network File Systems ->  NFS server support (as a module and with all it's sub-items)
* File systems -> Network File Systems ->  Ceph distributed file system (as a module)
* File systems -> Network File Systems ->  SMB3 and CIFS support (as a module)
* File systems -> Btrfs (disabled, it's existance offends me)
* Security options -> AppArmor support (needed for snaps and lxd)

There are PLENTY of device drivers that could be dropped or moved to modules
that would ultimately reduce the size of the kernel and therefore speed up the
boot process slightly.

Now start the build:
```
touch .scmversion
make -j3 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image \
  ${DTBRELPATH}${DTB}.dtb modules
KERNELVERSION=$(make kernelversion)
tar -czf ../packages/linux-headers-extra-${KERNELVERSION}.tar.gz \
  arch/arm64/kernel/vdso/* security/selinux/include/* tools/include/tools/*
cp arch/arm64/boot/dts/amlogic/${DTB}.dtb \
  ../tmp/mkbootimg/${DTB}-${KERNELVERSION}.dtb
cp arch/arm64/boot/Image ../tmp/mkbootimg/Image-${KERNELVERSION}
```

Now build run the build again, this time outputting some .debs<br/>
TODO: Can we just add deb-pkg to the first build command?  It seems not.  Hmmm.
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

## Future Experiment: Integrate Ubuntu's custom kernel patches?

## Future Experiment: Build kernel with ZFS built in
http://www.linuxquestions.org/questions/linux-from-scratch-13/%5Bhow-to%5D-add-zfs-to-the-linux-kernel-4175514510/

## Initrd
#### Using Device Emulation
If you already have a running device, the simplist way to produce the boot image
and initrd is to use said device.  Alternativly, you can emulate an arm64 device
on your build machine.

[Setup an Arm64 Chroot on an X86 build machine (Ubuntu/Debian)](SetupArm64ChrootOnX86_64.md)

Copy some of the generated packages and files into the chroot
```
rm "${ROOTFSPATH}/tmp/"*
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
dpkg -i /tmp/linux-headers*${KERNELVERSION}*.deb
cd /usr/src/linux-headers-${KERNELVERSION}
tar -xzf /tmp/linux-headers-extra-${KERNELVERSION}.tar.gz
make modules_prepare
dpkg -i /tmp/linux-image*${KERNELVERSION}*.deb
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
mv ${KERNELVERSION} ${KERNELVERSION}.orig
ln -s ${KERNELVERSION}-dirty ${KERNELVERSION}
```
And then try installing the package again:
```
dpkg -i /tmp/linux-image*${KERNELVERSION}*.deb
```

Make the initramfs and kernel (uImage) files:
```
update-initramfs -k ${KERNELVERSION} -c
cd /boot
mkimage -A arm64 -O linux -T ramdisk -a 0x0 -e 0x0 -n \
  initrd.img-${KERNELVERSION} -d /boot/initrd.img-${KERNELVERSION} \
  uInitrd-${KERNELVERSION}
mkimage -A arm64 -O linux -T kernel -C none -a 0x1080000 -e 0x1080000 -n \
  ${KERNELVERSION} -d /boot/vmlinuz-${KERNELVERSION} \
  /boot/uImage-${KERNELVERSION}
```

We're done inside the chroot for now:
```
exit
```

Move to the top level of your build environment and then:
```
cp "${ROOTFSPATH}/boot/initrd.img-${KERNELVERSION}" tmp/mkbootimg/
cp "${ROOTFSPATH}/boot/uImage-${KERNELVERSION}" tmp/mkbootimg/
cp "${ROOTFSPATH}/boot/uInitrd-${KERNELVERSION}" tmp/mkbootimg/
```

#### Using the device itself
I had a running Vim with the stock 4.9.26 kernel running so I installed the .deb
for the kernel I compiled above and then:
```
sudo update-initramfs -k 4.9.26-g8bc293d -c
```
Then I took the resulting /boot/initrd.img-4.9.26-g8bc293d

## Make a FIT image
A FIT image is a single file that contains Kernel, initramfs and DTB.  You start
with these three components as seperate files plus a .its file and then use
`mkimage` to create the final .itb file.

The .its file (which I simply call image-${KERNELVERSION}.its) looks like this:
```
/dts-v1/;

/ {
    description = "U-Boot fitImage for plnx_aarch64 kernel";
    #address-cells = <1>;

    images {
        kernel@0 {
            description = "Linux Kernel";
            data = /incbin/("./Image-4.16.13.gz");
            type = "kernel";
            arch = "arm64";
            os = "linux";
            compression = "gzip";
            load = <0x80000>;
            entry = <0x80000>;
            hash@1 {
                algo = "sha1";
            };
        };
        fdt@0 {
            description = "Flattened Device Tree blob";
            data = /incbin/("./meson-gxl-s905x-khadas-vim-4.16.13.dtb");
            type = "flat_dt";
            arch = "arm64";
            compression = "none";
            hash@1 {
                algo = "sha1";
            };
        };
        ramdisk@0 {
            description = "ramdisk";
            data = /incbin/("./initrd.img-4.16.13");
            type = "ramdisk";
            arch = "arm64";
            os = "linux";
            compression = "gzip";
            hash@1 {
                algo = "sha1";
            };
        };
    };
    configurations {
        default = "conf@1";
        conf@1 {
            description = "Boot Linux kernel with FDT blob + ramdisk";
            kernel = "kernel@0";
            fdt = "fdt@0";
            ramdisk = "ramdisk@0";
            hash@1 {
                algo = "sha1";
            };
        };
    };
};
```

To create the gzip compressed file `Image-4.16.13.gz`:
```
gzip -k Image-4.16.13
```

* TODO: Can we add things like the default "bootargs"?

```
mkimage -f image-${KERNELVERSION}.its image-${KERNELVERSION}.itb
```

## Make a root filesystem using an Ubuntu "base"
[Ubuntu root filesystem from a "base" archive](UbuntuRootFSFromBaseArchive.md)

## Use an existing root filesystem: Provided by Khadas
The Khadas images have a few extra goodies, such as making the Vim's red LED
"breathe" and "pulse" depending on system load, etc.

TODO

## Use an existing root filesystem: Provided by Armbian
TODO

## Testing
This step is not mandatory but if you find yourself making multiple attempts to
get everything just right, you might find that moving your SD card between build
machine and Vim gets tedious.  You can setup a TFTP server on your network and
test each kernel as you build it without using the SD card.

[TestingViaTFTP.md](TestingViaTFTP.md)

I would be SCPing the resulting files onto my TFTP server like this:
```
TFTPSERVER=172.30.0.2
scp tmp/mkbootimg/image-${KERNELVERSION}.itb root@$TFTPSERVER:/tftpboot/
```

## Installation
Head over to the [InstallOntoVim.md](InstallOntoVim.md) document for information
regarding the installation of your newly built OS.

## Troubleshooting
### Buffer I/O error on dev <device>, logical block <block id>, lost async page write
https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=879072
