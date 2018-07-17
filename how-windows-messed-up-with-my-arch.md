I have updated my windows to version 1803 and after the update, when i booted into Arch, I was greeted with an error message and landed into a preboot emergency shell.

The error message was

```
mount: mount(2) failed: /new_root: unknown filesystem type ntfs  
```

My immediate reaction was to check if windows has formatted my Linux partitions, so i booted back into windows and used [ext2fsd](http://www.ext2fsd.com/) to check if my Linux partitions are formatted, luckily they are still intact. Just remember, as long as your root and home partitions are intact, you can always fix boot errors with minimal effort. 
 

Now back to booting Arch, from the error message i can see that arch is trying to mount /dev/sda6 as new_root but it is failing because /dev/sda6 is a NTFS partition. I had a look at the /etc/fstab ( you can't view this file from emergency shell because / is not mounted yet, I have copied this file when i was in windows using ex2fsd) and the entry was 
 

```
# 
# /etc/fstab: static file system information
#
# <file system>	<dir>	<type>	<options>	<dump>	<pass>
UUID=6b0dd95b-db3e-4d4a-933c-dbd7a9cc8e4c	/         	ext4      	rw,relatime,data=ordered	0 1

UUID=06B5-1AEA      	/boot     	vfat      	rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro	0 2

UUID=239533d1-e40f-490f-9638-10de52f69aae	/home     	ext4      	rw,relatime,data=ordered	0 2
```

Surprisingly my /etc/fstab doesn't even contain a entry for /dev/sda6 but instead uses a UUID, so why are we even trying to mount /dev/sda6?? 

Now that i know the UUID of my root partition, i can temporarily fix this issue by mouting the root partition and then exiting the shell. But mount command need the partition label as argument ( actually you can mount a partition by it's UUID but i didn't know this ) , so how can we find the partition label from UUID?, we can use the below command to figure that out.

```
#ls -l /dev/disk/by-uuid 

total 0
lrwxrwxrwx 1 root root 10 Jul  8 14:53 06B5-1AEA -> ../../sda2
lrwxrwxrwx 1 root root 10 Jul  8 14:53 239533d1-e40f-490f-9638-10de52f69aae -> ../../sda8
lrwxrwxrwx 1 root root 10 Jul  8 14:53 64DEC9F7DEC9C192 -> ../../sda6
lrwxrwxrwx 1 root root 10 Jul  8 14:53 68686a99-4cd7-4c71-8345-eb0ca1ff8abe -> ../../sda9
lrwxrwxrwx 1 root root 10 Jul  8 14:53 6b0dd95b-db3e-4d4a-933c-dbd7a9cc8e4c -> ../../sda7
lrwxrwxrwx 1 root root 10 Jul  8 14:53 86DAB9C9DAB9B5B1 -> ../../sda4
lrwxrwxrwx 1 root root 10 Jul  8 14:53 AC3CB1583CB11DF0 -> ../../sda1
lrwxrwxrwx 1 root root 10 Jul  8 14:53 EC9C30A29C3068EA -> ../../sda5

```
 
I mounted the /dev/sda7 as new_root and I was able to boot into a proper shell
```
#mount /dev/sda7 new_root
#exit
```

Now that we have a proper shell, we need to figure out why we are trying to mount /dev/sda6 as new_root. From dmesg logs i figured out that initrd is using /dev/sda6 as root 


```
[    0.000000] Command line: initrd=\intel-ucode.img initrd=\initramfs-linux.img root=/dev/sda6 rw resume=/dev/sda8 
```
 

This values come from the bootloader entries , so i opend my boot config and found out that i have specifed /dev/sda6 as root in my config

```
title Ahiknsrs Arch  

linux /vmlinuz-linux 

initrd  /intel-ucode.img 

initrd /initramfs-linux.img 

options root=/dev/sda6 rw  

options resume=/dev/sda8 
```
 
so earlier /dev/sda6 used to be my root partition, but after the windows update the /dev/sda6 became /dev/sda7 (the UUID is still the same) I have specifically used UUID's in /etc/fstab because [partition names may change](https://unix.stackexchange.com/a/137868) but UUID's will always be the same. 

After changing my root to /dev/sda7 in boot config, my issue was fixed. 
