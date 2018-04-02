# Testing Device Tree and Kernel via TFTP
## TFTP Server
There are several documents online that guide you through setting up a TFTP
server on your preferred OS.  I use pfSense as my home router OS.  I simply
installed the TFTP package from it's Package Manager GUI.

## Before you begin
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

## Install U-Boot
I didn't try installing u-boot itself via TFTP.  I just installed it [the old
fashioned way](InstallOntoVim.md).

## Initial Network Setup
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

Save these settings so you don't need to enter them for every boot:
```
saveenv
```

## Download and install the Device Tree
```
tftp ${dtb_mem_addr} kvim_linux-${kver}.dtb
fdt addr ${dtb_mem_addr}
store dtb write ${dtb_mem_addr}
reset
```

## Download and launch a Boot Image
```
tftp ${kernel_loadaddr} boot-${kver}.img
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
tftp ${dtb_mem_addr} kvim_linux-${kver}.dtb; fdt addr ${dtb_mem_addr}
tftp ${kernel_loadaddr} uImage-${kver}
tftp ${initrd_loadaddr} uInitrd-4.9.40
bootm ${kernel_loadaddr} ${initrd_loadaddr} ${dtb_mem_addr}
```
