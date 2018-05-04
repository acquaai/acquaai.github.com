---
title: Create a bootable USB using macOS
date: 2018-05-04 15:06:39
categories: DevOps
---
```
➜  ~ ls CentOS-7*
CentOS-7-x86_64-DVD-1708.iso

➜  ~ hdiutil convert -format UDRW -o CentOS74.img CentOS-7-x86_64-DVD-1708.iso

➜  ~ diskutil list
/dev/disk0 (internal, physical):
   #:      TYPE NAME                   SIZE       IDENTIFIER
   0:      GUID_partition_scheme        *1.0 TB   disk0
   1:      EFI EFI                     209.7 MB   disk0s1
   2:      Apple_HFS Macintosh HD      999.3 GB   disk0s2
   3:      Apple_Boot Recovery HD      650.0 MB   disk0s3

/dev/disk1 (external, physical):
   #:      TYPE NAME                     SIZE       IDENTIFIER
   0:      GUID_partition_scheme         *15.5 GB   disk1
   1:      EFI EFI                       209.7 MB   disk1s1
   2:      Microsoft Basic Data KINGSTON  15.3 GB   disk1s2
   
➜  ~ diskutil unmountDisk /dev/disk1
Unmount of all volumes on disk1 was successful

➜  ~ sudo dd if=CentOS74.img.dmg of=/dev/disk1 bs=1m
```
