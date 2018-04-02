# Manually boot Kernel, etc from SDCard
```
setenv cec "cecf"
setenv mesontimer "0"
setenv nographics "0"
setenv condev "console=ttyAML0,115200n8 console=tty0 consoleblank=0"
setenv verbosity "255"
setenv bootargs "root=LABEL=ROOTFS rootflags=data=writeback rw ${condev} no_console_suspend hdmimode=${m} m_bpp=${m_bpp} fsck.repair=yes net.ifnames=0"

setenv bootargs "root=LABEL=EMMCROOTFS rootflags=data=writeback rw ${condev} no_console_suspend hdmimode=${m} m_bpp=${m_bpp} fsck.repair=yes net.ifnames=0"

"0x10000000"
setenv kernel_loadaddr "0x11000000"
setenv initrd_loadaddr "0x13000000"
fatload mmc 0:1 ${initrd_loadaddr} Archive/uInitrd
fatload mmc 0:1 ${kernel_loadaddr} Archive/zImage

fatload mmc 1:1 ${initrd_loadaddr} uInitrd
fatload mmc 1:1 ${kernel_loadaddr} zImage

fatload mmc 0:1 ${dtb_mem_addr} kvim1.dtb
fatload mmc 1:1 ${dtb_mem_addr} dtb/meson-gxl-s905x-khadas-vim.dtb

fdt addr ${dtb_mem_addr}
if test "${mesontimer}" = "1"; then fdt rm /timer; fi
if test "${nographics}" = "1"; then fdt rm /reserved-memory; fdt rm /aocec; fi
booti ${kernel_loadaddr} ${initrd_loadaddr} ${dtb_mem_addr}
```

# Manually boot kernel from ext4 partition on eMMC
```
setenv cec "cecf"
setenv mesontimer "0"
setenv nographics "0"

setenv condev "console=ttyS0,115200n8 console=tty0 consoleblank=0"
setenv condev "console=ttyAML0,115200n8 consoleblank=0"
setenv condev "console=ttyAML0,115200n8 console=tty0 console=ttyS0,115200n8 consoleblank=0"

setenv verbosity "255"

setenv bootargs "root=LABEL=ROOTFS rootflags=data=writeback rw ${condev} no_console_suspend hdmimode=${m} m_bpp=${m_bpp} fsck.repair=yes net.ifnames=0"

setenv bootargs "root=LABEL=ROOTFS rootflags=data=writeback rw ${condev} no_console_suspend fsck.repair=yes net.ifnames=0"

setenv bootargs "root=LABEL=EMMCROOT rootwait rootflags=data=writeback rw ${condev} no_console_suspend fsck.repair=yes net.ifnames=0 break=y"

setenv kernel_loadaddr "0x11000000"
setenv initrd_loadaddr "0x13000000"
ext4load mmc 1:1 ${initrd_loadaddr} /boot/uInitrd
ext4load mmc 1:1 ${kernel_loadaddr} /boot/zImage
ext4load mmc 1:1 ${dtb_mem_addr} /boot/dtb/kvim.dtb

ext4load mmc 1:1 ${initrd_loadaddr} uInitrd-partprobe
ext4load mmc 1:1 ${kernel_loadaddr} zImage
ext4load mmc 1:1 ${dtb_mem_addr} dtb/meson-gxl-s905x-khadas-vim.dtb

ext4load mmc 1:1 ${dtb_mem_addr} dtb/kvim.dtb

fdt addr ${dtb_mem_addr}
if test "${mesontimer}" = "1"; then fdt rm /timer; fi
if test "${nographics}" = "1"; then fdt rm /reserved-memory; fdt rm /aocec; fi
booti ${kernel_loadaddr} ${initrd_loadaddr} ${dtb_mem_addr}
```

# Save the dtb to eMMC of it gets lost
Use the above steps, up to the "fdt addr..." line, and then
```
store dtb write ${dtb_mem_addr}
```

# Manual installation with f2fs on eMMC
1.  Write the downloaded image to SD card
2.  Make sure you have serial access to the console port of Vim
3.  I don't know what happened here but I ended up with uboot installed on eMMC
4.  Boot from the SD card all the way into the OS.  Follow the promots to create
    a user account and set a password.
4.  Create partitions on eMMC:
      sudo parted /dev/mmcblk1
        1: 2048s to 100M - fat32 - this could also be ext2/4 next time.
        2: 100M to 2900M - ext2
5.  Copy the .img file that you wrote to the SD card into the filesystem of the
    running OS.  I just used SCP.
6.  Extract the filesystems from the .img file:
      sfdisk -l -uS Armbian_5.27_S905_Ubuntu_xenial_4.12.0-next-20170516_server.img
        Look for this:
          Armbian.....img2      198656 3928063 3729408  1.8G 83 Linux
        You need the first num-j3ber (198656) for the next command:
          sudo dd if=Armbian_5.27_S905_Ubuntu_xenial_4.12.0-next-20170516_server.img of=/dev/mmcblk1p2 skip=198656 bs=512
7.  Mount the new filesystem nad make sure there are files and it looks like a
    linux root:
      sudo mkdir /mnt/target
      sudo mount /dev/mmcblk1p2 /mnt/target
8.  Give the cloned ext4 filesystem it's own unique label so we don't confuse it
    with the one on the SD card:
      sudo e2label /dev/mmcblk1p2 EMMCROOTFS
9.  I only extracted the ext4 root filesystem from the image.  I manually
    formatted and populated the smaller boot partition:
      sudo mkfs.vfat -n EMMCBOOTFS /dev/mmcblk1p1
      sudo mount /dev/mmcblk1p2 /mnt/target/boot
      cd /mnt/target/boot
      shlopt -s dotglob
      sudo cp -ra /boot/* .
      sudo cp dtb/meson-gxl-s905x-khadas-vim.dtb dtb.img
10. rm /etc/init.d/resize2fs

TODO:
1.  On first boot, the / filesystem will be expanded to consume all available
    space on eMMC


## Configure u-boot to boot the OS on the eMMC
Dunno yet.

Manual steps:

setenv bootargs "root=LABEL=EMMCROOTFS rootflags=data=writeback rw ${condev} no_console_suspend hdmimode=${m} m_bpp=${m_bpp} fsck.repair=yes net.ifnames=0"
setenv kernel_loadaddr "0x11000000"
setenv initrd_loadaddr "0x13000000"
fatload mmc 1:1 ${initrd_loadaddr} uInitrd
fatload mmc 1:1 ${kernel_loadaddr} zImage
fatload mmc 1:1 ${dtb_mem_addr} dtb/meson-gxl-s905x-khadas-vim.dtb
fdt addr ${dtb_mem_addr}
booti ${kernel_loadaddr} ${initrd_loadaddr} ${dtb_mem_addr}

We probably need to modify aml_autoscript or s905_autoscript and then use a
command like this to make the signed(?) version:
mkimage -A arm -O linux -T script -C none -d s905_autoscript.cmd s905_autoscript

Append this to /boot/s905_autoscript.cmd
```
setenv bootargs "root=LABEL=EMMCROOTFS rootflags=data=writeback rw ${condev} fsck.repair=yes net.ifnames=0 mac=${mac}"
if fatload mmc 1:1 ${initrd_loadaddr} uInitrd; then if fatload mmc 1:1 ${kernel_loadaddr} zImage; then if fatload mmc 1:1 ${dtb_mem_addr} dtb.img; then run boot_start; else store dtb read ${dtb_mem_addr}; run boot_start;fi;fi;fi;
```

Run this:
```
cd /boot
mkimage -A arm -O linux -T script -C none -d s905_autoscript.cmd s905_autoscript
```

Now you can boot with only this:
```
setenv s905_autoscript "0x10000000"
fatload mmc 1:1 "0x10000000" s905_autoscript
autoscr ${s905_autoscript}
```

I still don't know how to make this run on it's own though.

# Other crap
## Armbian testing images for Vim:
https://yadi.sk/d/pHxaRAs-tZiei/Test


imgread kernel ${bootdisk} ${loadaddr}; bootm ${loadaddr}


store read name addr off|partition size
    read 'size' bytes starting at offset 'off'
    to/from memory address 'addr', skipping bad blocks.


store read ramdisk ${loadaddr} 800000 2000000
                               800000

fatload mmc 0:1 ${dtb_mem_addr} meson1.dtb
setenv bootargs "root=LABEL=ROOTFS rootflags=data=writeback rw console=ttyS0,115200n8 console=tty0 no_console_suspend consoleblank=0 fsck.repair=yes net.ifnames=0"
store read ramdisk ${loadaddr} 800000 2000000
bootm ${loadaddr}

## Filesystem info
/dev/mmcblk1p1: UUID="ab1a67e3-e83d-46bf-bfb4-86d61832ec62" TYPE="ext4" PARTUUID="602c2d31-01"
EMMCROOTFS







-rwxr-xr-x 1 root root      533 May 16 13:04 aml_autoscript
-rwxr-xr-x 1 root root      395 May 16 13:04 aml_autoscript.zip
-rwxr-xr-x 1 root root     1623 May 16 13:04 amlogics905x_init.sh
-rwxr-xr-x 1 root root     6944 May 16 13:04 boot.bmp
-rwxr-xr-x 1 root root   161155 May 16 12:54 config-4.12.0-next-20170516+
drwxr-xr-x 2 root root     1024 May 24 13:37 dtb
drwxr-xr-x 2 root root     1024 May 16 13:03 dtb-4.12.0-next-20170516+
-rwxr-xr-x 1 root root  7034500 May 16 13:04 initrd.img-4.12.0-next-20170516+
-rwxr-xr-x 1 root root 23197198 May 16 13:04 linux.img
drwx------ 2 root root    12288 May 16 13:03 lost+found
-rwxr-xr-x 1 root root      873 May 16 13:04 s905_autoscript
-rwxr-xr-x 1 root root      801 May 16 13:04 s905_autoscript.cmd
-rwxr-xr-x 1 root root  3689934 May 16 12:54 System.map-4.12.0-next-20170516+
-rwxr-xr-x 1 root root  5230678 May 24 13:39 uInitrd
-rwxr-xr-x 1 root root  5230678 May 24 13:39 uInitrd-3.14.29
-rwxr-xr-x 1 root root  7034564 May 16 13:04 uInitrd-4.12.0-next-20170516+
-rwxr-xr-x 1 root root        0 May 16 13:04 .verbose
-rwxr-xr-x 1 root root 16409088 May 16 12:54 vmlinuz-4.12.0-next-20170516+
-rwxr-xr-x 1 root root 16932800 May 24 13:38 zImage
-rwxr-xr-x 1 root root 16932800 May 24 13:38 zImage-3.14.29
-rwxr-xr-x 1 root root 16409088 May 16 12:54 zImage-4.12.0-next-20170516+





Partition table get from SPL is :
        name                        offset              size              flag
================================================================================
===
   0: bootloader                         0            400000                  0
   1: reserved                     2400000           4000000                  0
   2: env                          6c00000            800000                  0
   3: logo                         7c00000           2000000                  1
   4: ramdisk                      a400000           2000000                  1
   5: rootfs                       cc00000         1c5400000                  4

Partition table get from SPL is :
       name                        offset              size              flag
================================================================================
===
  0: bootloader                         0            400000                  0
  1: reserved                     2400000           4000000                  0
  2: env                          6c00000            800000                  0
  3: logo                         7c00000           2000000                  1
  4: ramdisk                      a400000           2000000                  1
  5: rootfs                       cc00000         1c5400000                  4

================================================================================
===
   0: bootloader                         0            400000                  0
   1: reserved                     2400000           4000000                  0
   2: cache                        6c00000                 0                  0
   3: env                          7400000            800000                  0
   4: logo                         8400000           2000000                  1
   5: ramdisk                      ac00000           2000000                  1
   6: rootfs                       d400000         1c4c00000                  4


2400000
4000000





partitions {
		parts = <0x3>;
		part-0 = <0x18>;
		part-1 = <0x19>;
		part-2 = <0x1a>;

		logo {
			pname = "logo";
			size = <0x0 0x2000000>;
			mask = <0x1>;
			linux,phandle = <0x18>;
			phandle = <0x18>;
		};

		ramdisk {
			pname = "ramdisk";
			size = <0x0 0x2000000>;
			mask = <0x1>;
			linux,phandle = <0x19>;
			phandle = <0x19>;
		};

		rootfs {
			pname = "rootfs";
			size = <0xffffffff 0xffffffff>;
			mask = <0x4>;
			linux,phandle = <0x1a>;
			phandle = <0x1a>;
		};
	};

## Modify the .dtb/dts file
### Reduce the size of the rootfs partition
find this section:
```
    rootfs {
			pname = "rootfs";
			size = <0xffffffff 0xffffffff>;
			mask = <0x4>;
			linux,phandle = <0x1a>;
			phandle = <0x1a>;
		};
```

All 'f's seems to mean fill the rest of the storage.  If you want the rootfs
partition to be less than 4GiB, replace the first block with just 0x0.  Leaving
the other block as 0xffffffff will give you 4GiB

I wanted to be closer to 2GiB.  I stuck ffffffff into calculator in hex mode and
saw that that number in base 10 is 4GiB in bytes.  I just fiddled with the
number until I got close enough to 2GiB.  My result was:
```
rootfs {
        pname = "rootfs";
        size = <0x0 0x78ffffff>;
        mask = <0x4>;
        linux,phandle = <0x29>;
        phandle = <0x29>;
};
```

In hindsight, 2GiB is probably 0x88888888.  Ah well.  Near enough.

### Adding a partition
My data partition will simply take up the rest of the eMMC, hense the
`size = <0xffffffff 0xffffffff>`.  You need to find a unique phandle.  I just
searched until I found one that wasn't in use, 0x70.
```
datafs {
        pname = "datafs";
        size = <0xffffffff 0xffffffff>;
        mask = <0x4>;
        linux,phandle = <0x70>;
        phandle = <0x70>;
};
```

## Installing the Khadas image with modifications



## Remove these packages:
autotools-dev binutils build-essential bzip2 cpp cpp-5 dpkg-dev fakeroot g++ g++-5 gcc gcc-5 libalgorithm-diff-perl
  libalgorithm-diff-xs-perl libalgorithm-merge-perl libasan2 libatomic1 libcc1-0 libdpkg-perl libfakeroot libfile-fcntllock-perl
  libgcc-5-dev libgomp1 libisl15 libitm1 libltdl-dev libltdl7 libmpc3 libmpfr4 libstdc++-5-dev libtool libubsan0 make
