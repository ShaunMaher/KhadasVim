## update-initramfs always fails with an error referring to flash-kernel
```
sudo vim /etc/initramfs/post-update.d/flash-kernel
```
Add the following line near the top of the script, after the #!
```
exit 0
```
