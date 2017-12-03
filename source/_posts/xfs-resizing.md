---
title: XFS filesystem resizing
date: 2017-12-03 14:50:14
categories: DevOps
---
## Shrink

[XFS](http://xfs.org/)由硅谷图形开发，支持在线碎片整理和扩容，截至到2017年8月，暂不支持在线收缩，只能dump, mkfs and restore。

* Backup home
  `$ yum install xfsdump`
  `$ xfsdump -f /tmp/home.dump /home`

* umount home
  `$ umount /home`
  
* Resize
  `lvreduce -L 40G /dev/cl/home`
   
* Recreate xfs filesystem
  `$ mkfs.xfs -f /dev/cl/home`
  
* mount home
  `$ mount /dev/mapper/cl-home /home`

* Restore home
  `$ xfsrestore -f /tmp/home.dump /home`

<!-- more -->

* 检查/etc/fstab文件和/home

## Extension

``` bash　　　　　　　　　　　　　　　    
$ pvcreate /dev/sda3
$ pvs -v
$ vgextend centos /dev/sda3
$ vgs -v
$ lvextend -L +100G /dev/mapper/centos-root　　
$ lvs -v
$ xfs_growfs /dev/mapper/centos-root
```
