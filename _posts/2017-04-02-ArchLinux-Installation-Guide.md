---
layout: post
title: ArchLinux Installation Guide
categories: ArchLinux
---

> Arch Linux 安装记录。

<!-- more -->

1. 刻录启动盘。  
在[archlinux官网](https://www.archlinux.org/download/)寻找国内合适的源下载镜像（即.iso文件）。  
插入U盘，查看U盘分区：  
`$ lsblk`  
我的显示： 
```  
NAME            MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT  
......  
sdc               8:32   1  14.4G  0 disk  
└─sdc1            8:33   1  14.4G  0 part /run/media/nian/76379  
......  
```  
其中sdc就指的是我的U盘，sdc1则表示U盘上的一个分区，后面的路径是当前该分区挂载的路径，如果为空则表示未挂载。  
取消挂载并格式化：  
`$ umount /dev/sdc`  
`$ sudo wipefs --all /dev/sdc`  
再次查看：  
`$ lsblk`  
```  
NAME            MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT  
......  
sdc               8:32   1  14.4G  0 disk  
......  
```  
取消挂载之后开始刻录（注意修改镜像路径以及U盘名称）：  
`$ sudo dd bs=4M if=/home/nian/Downloads/archlinux-2017.02.01-dual.iso of=/dev/sdc status=progress >> sync`  
注意这个过程可能需要持续几分钟，而且中间不会显示写入的过程，请等待提示写入完成之后再拔下U盘。  

2. 未完待续。。。  
