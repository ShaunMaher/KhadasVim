## Resizing an f2fs root filesystem
You cannot use the resize.f2fs tool on a mounted filesystem.  This makes things
a little tricky.

One option is to simply boot from a USB stick or SD card.  You're possibly going
to have to deal with partition name conflicts ("ROOTFS" on both eMMC and
SD/USB).

Another option is to break the boot process on purpose and then use resize.f2fs
from the initramfs environment.  The first hurdle is getting resize.f2fs into
the initramfs image.

Create `/usr/share/initramfs-tools/hooks/resize.f2fs` with the following
content:
```
#!/bin/sh

PREREQ=""

prereqs() {
	echo "$PREREQ"
}

case $1 in
# get pre-requisites
	prereqs)
		prereqs
		exit 0
		;;
esac

. /usr/share/initramfs-tools/hook-functions
if [ -e /sbin/resize.f2fs ]; then
cp -pnL /sbin/resize.f2fs ${DESTDIR}/sbin/resize.f2fs
chmod 755 ${DESTDIR}/sbin/resize.f2fs
```
and then make it executable:
```
sudo chmod +c resize.f2fs
```

And then update the initramfs image:
```
sudo update-initramfs -c -k 4.16.13
```

TODO
