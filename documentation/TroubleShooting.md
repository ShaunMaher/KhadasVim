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
I don't know.  I haven't worked this one out yet.
