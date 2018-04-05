## update-initramfs always fails with an error referring to flash-kernel
```
sudo vim /etc/initramfs/post-update.d/flash-kernel
```
Add the following line near the top of the script, after the #!
```
exit 0
```

## bash: xmalloc: .././shell.c:1709: cannot allocate 10 bytes (0 bytes allocated)
There was a bug in the bash version included in the pre-release Ubuntu 18.04
that resulted in it crashing with the above error when being executed from
within qemu.  I overcame this by finding an older version of bash on
http://packages.debian.org for Arm64, extracting the bash executable and putting
it in /bin inside the chroot.

## When running "make modules_prepare": "scripts/kconfig/conf: Command not found"
I don't know.  I haven't worked this one out yet.  It only seems to happen with
the Khadas kernel fork version 4.9.

## Boot loop after installing Device Tree
**Don't Panic!**<br/>
I cannot get any .dtb I created from the mainline kernel to not result in a
u-boot loop.  I keep going back top the one created from the Khadas 4.9 kernel.

To break the loop you need to short a couple of pins on the Vim board.
Unfortunately these pins are not accessable while the Vim is in it's case.  Undo
the four brass nuts on the bottom of the unit, being careful to keep the screws
in their holes (makes reassembly easier), and remove the bottom acrilic layer.
At this point I put the brass nuts back onto the screws to stop the screws
falling out.

Look for a column of small components near but perpendicular to the (bottom of
the) GPIO pins.  One is marked with an "M".

With the device in it's loop, bridge the two contacts of "M" with a small
screwdriver or similar tool.  The loop will be broken and the Vim will boot to
a u-boot prompt.

Any saved environment variables you had have been lost so you will need to
reconfigure them.
