---
layout: post
title: ArchLinux Installation Guide
categories: ArchLinux
---

> Arch Linux 安装记录。  
> **安装记录通常缺少更新，建议参考官方 [ArchWiki](https://wiki.archlinux.org/title/Installation_guide) 最新说明**

<!-- more -->

### 刻录启动盘  
* 在 archlinux [官网](https://www.archlinux.org/download/)寻找国内合适的源下载镜像（即.iso文件）。
* 插入U盘，执行 `lsblk` 查看U盘分区：  
  （例如 sdc 就是我的U盘，sdc1 则表示U盘上的一个分区，最后的路径是该分区挂载的目录，如果为空则表示未挂载）
  ```
  NAME            MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
  ......
  sdc               8:32   1  14.4G  0 disk
  └─sdc1            8:33   1  14.4G  0 part /run/media/nian/76379
  ......
  ```
* 取消挂载并格式化：  
  `$ umount /dev/sdc`  
  `$ sudo wipefs --all /dev/sdc`  
* 再次执行 `lsblk` 查看：  
  ```
  NAME            MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
  ......
  sdc               8:32   1  14.4G  0 disk
  ......
  ```
* 取消挂载之后开始刻录（注意修改镜像路径以及U盘名称）：  
  `$ sudo dd bs=4M if=/home/nian/Downloads/archlinux-2017.02.01-dual.iso of=/dev/sdc status=progress && sync`  
  这个过程可能需要持续几分钟，而且中间不会显示写入的过程，请等待提示写入完成之后再拔下U盘。  

### U盘启动  
* 将U盘插入电脑，开机，按F12（不同主板可能会有所不同），选择从USB启动：  
  这是 BIOS 启动的选择界面：  
  ![archbios](/public/image/archbios.webp)  
  这是 UEFI 启动的选择界面：  
  ![archuefi](/public/image/archuefi.webp)  
* 均选择第一项启动，加载成功后会出现：  
  `root@archiso ~ #`  

### 检查是否联网  
* 我 ping 一些常用的网站都 ping 不通：  
  `# ping -c 4 www.baidu.com`  
  百度得到，PING 是使用 ICMP 协议，有许多网站会设置 ICMP 数据包过滤来提高安全性，这可能是原因之一；另一种可能是我们学校的路由器设置了过滤。  
* 使用 Linux 下的 curl 命令进行尝试：  
  `# curl baidu.com`  
  如果已经联网，会得到：  
  ```html
  <html>  
  <meta http-equiv="refresh" content="0;url=http://www.baidu.com/">  
  </html>  
  ```

### 更新系统时间  
* `# timedatectl set-ntp true`  

### 硬盘分区  
* 查看当前硬盘分区情况：  
  `# fdisk -l`  
* 打开要安装到的硬盘，并分区：  
  `# fdisk /dev/sda`  
  （bios 可以正常分区，一个/，一个/swap）  
  （uefi 需要为 /boot 单独分区，即一个/，一个/swap，一个/boot，/boot 建议预留 1G）  
  我的分区：  
  ```
  NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
  sda      8:0    0 111.8G  0 disk
  ├─sda1   8:1    0     1G  0 part /boot
  ├─sda2   8:2    0 102.8G  0 part /
  └─sda3   8:3    0     8G  0 part [SWAP]
  ```
* 格式化根分区为 ext4 格式：  
  `# mkfs.ext4 /dev/sda2`  
  格式化交换分区：  
  `# mkswap /dev/sda3`  
* 挂载分区：  
  `# mount /dev/sda2 /mnt`  
  `# swapon /dev/sda3`  
* 如果是 UEFI 启动，需要将启动分区格式化为 vfat 格式并挂载：  
  `# mkfs.fat -F 32 /dev/sda1`  
  `# mount --mkdir /dev/sda1 /mnt/boot`  

### 安装系统  
* 编辑 /etc/pacman.d/mirrorlist ，选择合适的镜像站：  
  `# nano /etc/pacman.d/mirrorlist`  
  我仅保留了国内的清华源：  
  `Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch`  

* 安装基本系统：  
  `# pacstrap -K /mnt base base-devel linux linux-firmware nano`  
  注：2019-10-06 后仓库中的 base 软件包组被同名的软件包代替，只安装 base 软件包会比之前少很多东西，例如内核，nano 编辑器等。（[Installation guide: Install the base packages](https://wiki.archlinux.org/index.php/Installation_guide#Install_the_base_packages)）  

### 配置系统  
* 生成 fstab 文件，让系统启动的时候自动挂载分区：  
  `# genfstab -U /mnt >> /mnt/etc/fstab`  
* change root 到新安装的系统并指定 shell 为 bash：  
  `# arch-chroot /mnt /bin/bash`  
* 设置时区为上海：  
  `# ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime`  
  `# hwclock --systohc --utc`  
* 设置语言：  
  `# nano /etc/locale.gen`  
  取消下面两项前的注释：  
  ```
  en_US.UTF-8 UTF-8  
  zh_CN.UTF-8 UTF-8  
  ```
  然后 Ctrl+O 保存，Ctrl+X 退出。  
* 接着生成 locale：  
  `# locale-gen`  
* 创建 locale.conf 并保存本地设置：  
  `# echo LANG=en_US.UTF-8 > /etc/locale.conf`  
* 设置主机名：  
  `# echo yourhostname > /etc/hostname`  
* 建议添加对应的信息到 hosts：  
  `# nano /etc/hosts`  
  ```
  127.0.0.1   localhost.localdomain localhost  
  ::1         localhost.localdomain localhost  
  ```
* 设置 root 用户密码：
  `# passwd`

### 安装引导程序
* 如果是 Intel CPU，那么还需要安装`intel-ucode`来启用英特尔微码更新：  
  `# pacman -S intel-ucode`  
* 安装引导程序：  
  ```sh
  # BIOS：
  pacman -S grub
  grub-install --target=i386-pc /dev/sda
  grub-mkconfig -o /boot/grub/grub.cfg

  # UEFI：
  pacman -S efibootmgr grub
  grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
  grub-mkconfig -o /boot/grub/grub.cfg
  ```
* Win10 用户装双系统的话最好关闭快速启动，否则启动 Win10 后，下次再启动电脑时有可能找不到 grub 引导项。（说多了都是泪啊 TAT）

### 创建用户
* 创建新用户，并设置 zsh 为默认 shell：  
  `# pacman -S zsh`  
  `# useradd -m -G wheel -s /bin/zsh nian`  
* 设置密码：  
  `# passwd nian`  
* 设置 sudo 权限：  
  `# pacman -S sudo`  
  `# nano /etc/sudoers`  
  在 `root ALL=(ALL) ALL` 下面一行添加 `nian ALL=(ALL) ALL`，保存退出。  

### 安装桌面环境
* 安装桌面环境基础包：  
  `# pacman -S xorg`  
* 安装 KDE 环境：  
  较简单的 KDE 环境：  
  `# pacman -S plasma-meta dolphin kate konsole sddm`  
  或者完整的 KDE 环境（不建议）：  
  `# pacman -S plasma kde-applications sddm`  
* 安装桌面网络管理器：  
  `# pacman -S networkmanager`  
* 设置开机启动项：  
  `# systemctl enable sddm`  
  `# systemctl enable NetworkManager`  
* 安装字体（文泉驿微米黑）：  
  `# pacman -S wqy-microhei`  

### 重启。
* 输入 exit 或按 Ctrl+D 退出 chroot。然后卸载挂载的分区：  
  `# umount -R /mnt`  
  移除安装介质并重启:  
  `# reboot`  

> Thanks to:  
> 博客：[约伊兹的萌狼乡手札](https://blog.yoitsu.moe/arch-linux/installing_arch_linux_for_complete_newbies.html)  
> Wiki：[Installation guide](https://wiki.archlinux.org/index.php/Installation_guide_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))  
