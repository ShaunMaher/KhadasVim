# Testing Device Tree and Kernel via TFTP
## TFTP Server
There are several documents online that guide you through setting up a TFTP
server on your preferred OS.  I use pfSense as my home router OS.  I simply
installed the TFTP package from it's Package Manager GUI.

## Before you begin
**Reference**: https://www.emcraft.com/som/using-dhcp<br/>
There are two ways to setup u-boot's network stack.  First is manually
specifying network parameters (IP, netmask, gateway, TFTP server) or using DHCP.

To use DHCP, simply replace any of the "tftp" commands in the below u-boot
commands with "dhcp".

If you are not going to use DHCP, you will need to find an unused IP address on
your network, your network's netmask (e.g. 255.255.255.0) and, if your TFTP
server is in another subnet, you're network's default gateway.

## ROOTFS
You can follow the following steps without having a ROOTFS for the kernel to
mount and init from, but you'll obviously reach a point where the kernel stops.

By default, the kernel is passed command line options from u-boot and the u-boot
we build tells the kernel for mount any filesystem it can find that has the
label "ROOTFS".  This means that you could have such a partition on an SD card,
USB drive or on the eMMC itself.

## Manual Network Setup
If you are not using DHCP, you need to give u-boot the network information you
gathered earlier.  For my network this looks like:
```
setenv gatewayip 172.30.0.2
setenv netmask 255.255.0.0
setenv ipaddr 172.30.0.111
setenv serverip 172.30.0.2
saveenv
```

## Initial Setup
For some reason, these are not set:
```
setenv kernel_loadaddr "0x11000000"
setenv initrd_loadaddr "0x13000000"
```

I'm also going to setup a kernel version variable to save typing:
```
setenv kver 4.9.40
```

I'm going to create another environment variable that will be set to either
"dhcp" or "tftp".  This way all the following commands can be the same
regardless of you're using DHCP or a manual network setup.
For DHCP
```
setenv tftpcmd "dhcp"
```

For TFTP with manual network settings:
```
setenv tftpcmd "tftp"
```

Save these settings so you don't need to enter them for every boot:
```
saveenv
```

## Test u-boot without installing
You can chainload U-Boot, which is to say, you can use the already installed
u-boot to load and execute your new u-boot to give it a trial run without
changing the u-boot stored on the MMC.

To do this, you need to know where in memory U-Boot expects to find itself.  In
the root of your U-Boot build tree there should be a file called simply
"u-boot".  This is the u-boot executable before it's stripped and wrapped up.
```
objdump -f u-boot
```

From the following output we can see that the u-boot for the Vim seems to expect
to be at 0x1000000
```
u-boot.elf:     file format elf64-little
architecture: UNKNOWN!, flags 0x00000112:
EXEC_P, HAS_SYMS, D_PAGED
start address 0x0000000001000000
```

Copy the u-boot.bin from the u-boot build root directory (not the one in the fip
directory) to your TFTP server.  Boot the Vim to a u-boot prompt (repeatedly tap
space while it's going through the very first stages of boot) and try the
following:
```
${tftpcmd} 0x1000000 u-boot.bin
go 0x1000000
```

You should see the U-Boot startup text flash by:
```
kvim#go 0x1000000
## Starting application at 0x01000000 ...


U-Boot 2018.01-rc3 (Apr 30 2018 - 19:41:50 +1000) khadas-vim
```

If, instead, the system seems to have rebooted itself, something went wrong.

## Install U-Boot
```
${tftpcmd} ${initrd_loadaddr} u-boot.bin
store rom_write ${initrd_loadaddr} 0 100000
reset
```
After the device restarts:
```
defenv
saveenv
```
Repeat the "Initial Network Setup" and "Initial Setup" steps above as your saved
environment variables have been lost.

## Download and install the Device Tree
```
${tftpcmd} ${dtb_mem_addr} kvim_linux-${kver}.dtb
fdt addr ${dtb_mem_addr}
store dtb write ${dtb_mem_addr}
reset
```

## Download and launch a Boot Image
```
${tftpcmd} ${kernel_loadaddr} boot-${kver}.img
bootm
```

Or, before booting it ("bootm"), save it to eMMC:
```
TODO
```

## Download and launch a Kernel and Initrd
This method has the advantage of not needing the mkbootimg tool on your build
machine.  Having a dependence on an x86 machine in order to get your Arm machine
running is silly.
```
${tftpcmd} ${dtb_mem_addr} kvim_linux-${kver}.dtb; fdt addr ${dtb_mem_addr}
${tftpcmd} ${kernel_loadaddr} uImage-${kver}
${tftpcmd} ${initrd_loadaddr} uInitrd-${kver}
bootm ${kernel_loadaddr} ${initrd_loadaddr} ${dtb_mem_addr}
```

The above merged into a single line:
```
setenv serverip 172.30.0.2; setenv kver 4.14.11; setenv tftpcmd "dhcp";setenv kernel_loadaddr "0x11000000"; setenv initrd_loadaddr "0x13000000"



${tftpcmd} ${dtb_mem_addr} kvim_linux-${kver}.dtb; fdt addr ${dtb_mem_addr};${tftpcmd} ${kernel_loadaddr} uImage-${kver};${tftpcmd} ${initrd_loadaddr} uInitrd-${kver};bootm ${kernel_loadaddr} ${initrd_loadaddr} ${dtb_mem_addr}
```
