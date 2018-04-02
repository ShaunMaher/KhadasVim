# Testing Device Tree and Kernel via TFTP
## TFTP Server
There are several documents online that guide you through setting up a TFTP
server on your preferred OS.  I use pfSense as my home router OS.  I simply
installed the TFTP package from it's Package Manager GUI.

## Before you begin
You will need to find an unused IP address on your network and your network's
netmask (e.g. 255.255.255.0).  As far as I know, u-boot (or at least the Khadas
fork of u-boot) does not support DHCP.  If your TFTP server is in another
subnet, you need to know your gateway's IP.

## Install U-Boot
I didn't try installing u-boot itself via TFTP.  I just installed it [the old
fashioned way](InstallOntoVim.md).

## Initial Setup
You need to give u-boot the network information you gathered earlier.  For my
network this looks like:
```
setenv gatewayip 172.30.0.2
setenv netmask 255.255.0.0
setenv ipaddr 172.30.0.111
setenv serverip 172.30.0.2
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
TODO: don't know yet
