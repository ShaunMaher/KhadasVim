# Setup an Arm64 Chroot on an X86 build machine (Ubuntu/Debian)

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
ROOTFSPATH=$(pwd)/roots/ubuntu-base-18.04
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
sudo mount -o bind /dev/pts "${ROOTFSPATH}/dev/pts"
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
