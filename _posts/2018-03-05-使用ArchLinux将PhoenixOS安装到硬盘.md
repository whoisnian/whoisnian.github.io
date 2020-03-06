---
layout: post
title: 使用ArchLinux将PhoenixOS安装到硬盘
categories: ArchLinux
---

> 想在PC上体验《绝地求生：刺激战场》，而虚拟机的性能又太低，就想到了之前体验过的 PhoenixOS 。  
> 之前在 Windows 上直接从[PhoenixOS官网](http://www.phoenixos.com/download_x86)下载安装程序，简单选择几个选项就可以将其安装到硬盘或刻录到U盘。  
> 但现在电脑上只有一个 ArchLinux，按照 PhoenixOS 官网上的教程（[Linux下如何安装凤凰系统](http://www.phoenixos.com/help/installation/iso-Linux)）使用 dd 刻录启动盘也失败了，在 BIOS 里找不到启动项，最终花了半天时间总算是成功安装上了。  

<!-- more -->

### 相关环境
* Arch Linux x86_64，使用 GRUB 进行 UEFI 引导  
* PhoenixOSInstaller_v2.5.9.375_x86_x64.iso 镜像  

### 安装步骤
* 使用分区工具在硬盘上为 PhoenixOS 分出一个区，并将其格式化为 ext4 格式，然后挂载该分区。  
  假设我想要使用在`/dev/sda`上新分出`/dev/sda4`分区，则需要：  
  * 使用`sudo fdisk /dev/sda`对`/dev/sda`进行分区；  
  * 使用`sudo mkfs.ext4 /dev/sda4`对`/dev/sda4`进行格式化；  
  * 使用`sudo mount /dev/sda4 /mnt`将`/dev/sda4`挂载到`/mnt`。  
* 在[PhoenixOS官网](http://www.phoenixos.com/download_x86)下载所需的iso镜像，然后解压并将其中的`initrd.img`，`kernel`，`ramdisk.img`，`system.sfs`拷贝到挂载的目录上。  
  再使用`squashfs-tools`包中的`unsquashfs system.sfs`命令将`system.sfs`转换为`system.img`，然后复制出`system.img`，并删除不再需要的`system.sfs`。  
  最后`mkdir /mnt/data`创建数据文件夹。  
* 编辑`/etc/grub.d/40_custom`给`PhoenixOS`添加 GRUB 启动项，因为我的安卓分区在`/dev/sda4`，即第一块硬盘的第四分区，所以内容如下：  
  ```
  menuentry 'Phoenix OS' --class android-x86 {
      set root=(hd1,4)
      search --set=root --file /kernel
      linux /kernel quiet root=/dev/ram0 SRC=/ vga=auto
      initrd /initrd.img
  }
  ```
  最后更新 GRUB 配置文件：  
  `sudo grub-mkconfig -o /boot/grub/grub.cfg`  
* 此时重启电脑，在 GRUB 选择页面就会出现`Phoenix OS`的条目了。  

### 玩游戏
* 我安装 PhoenixOS 系统的目的可是为了玩游戏，系统装好了，游戏也装好了，打开游戏却是黑屏。  
  不过还好论坛上已经给出了比较详细的解决办法：[整合帖：解决刺激战场的各种问题](http://bbs.phoenixstudio.org/cn/read.php?tid=14554&fid=12)。  
黑屏貌似是我笔记本 Nvidia 显卡的锅，需要 Root 后手动修改 GPU 参数，论坛上过程描述得比较详细，这里就不再进行多余的记录了。  
大致就是使用官网提供的`SuperSU.zip`运行里面的一键脚本进行 Root，然后利用 GLtools 为《绝地求生：刺激战场》修改 GPU 参数即可。  

### 截图
* 下面就扔几张截图作为结尾吧！  
  **=====  一个没什么用的多图预警  =====**  
  （反正你看到这儿的时候图已经加载完了）  

![PhoenixOS_1](/public/image/PhoenixOS_1.webp)  
![PhoenixOS_2](/public/image/PhoenixOS_2.webp)  
![PhoenixOS_3](/public/image/PhoenixOS_3.webp)  
![PhoenixOS_4](/public/image/PhoenixOS_4.webp)  
![PhoenixOS_5](/public/image/PhoenixOS_5.webp)  

**注：安装步骤主要是受到了XDA论坛上[Remix OS - Installation, ROOTing & more!](https://forum.xda-developers.com/remix/remix-os/remix-os-installation-rooting-t3293769)这篇教程的启示，然后踩了几个坑，又搜了一些东西才成功的**  

**PS：整体上 PhoenixOS 系统看起来还是蛮不错的，但在我的电脑上经常出现“XXX 程序未响应”的问题，不知道是不是个例，因此并不推荐日常使用，装上只是单纯打个游戏还是挺舒服的。感觉以后稳定下来作为日常使用也还可以，希望 PhoenixOS 能越来越棒吧！**
