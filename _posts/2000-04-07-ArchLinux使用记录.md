---
layout: postt
title: ArchLinux使用记录
categories: ArchLinux
comments:
  - author:
      type: full
      displayName: whoisnian
      url: 'https://github.com/whoisnian'
      picture: 'https://avatars2.githubusercontent.com/u/23057947?v=3&s=73'
    content: qwe
    date: 2017-04-20T03:50:07.081Z

---

> 使用 ArchLinux 过程中的一些记录。

<!-- more -->

### 换源  
在学校感觉[清华源](https://mirror.tuna.tsinghua.edu.cn/)的速度还是蛮不错的，因此就把我的 Arch 上的源都换成清华源了。  
* 添加 archlinux 源：  
  `$ sudo vim /etc/pacman.d/mirrorlist`  
  然后将内容改为：  
  {% highlight default %}
  ##  
  ## Arch Linux repository mirrorlist  
  ##  
    
  ## China  
  Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch  
  {% endhighlight %}
* 添加 archlinuxcn 中文社区源：  
  `$ sudo vim /etc/pacman.conf`  
  在最后面添加：  
  ```
  [archlinuxcn]  
  Server = http://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch  
  ```
  然后导入 GPG key：  
  `sudo pacman -Syy && sudo pacman -S archlinuxcn-keyring`  

### 常用软件
* 网页浏览器：  
  `$ sudo pacman -S chromium`  
* 浏览器Flash插件：  
  `$ sudo pacman -S pepper-flash`  
* 科学上网：  
  `$ sudo pacman -S shadowsocks-qt5`  
* 音乐播放器：  
  `$ sudo pacman -S netease-cloud-music`  
* 图像查看器：  
  `$ sudo pacman -S gwenview`  
* PDF阅读器：  
  `$ sudo pacman -S okular`  
* office套装：  
  `$ sudo pacman -S libreoffice-still libreoffice-still-zh-CN`  
* 截屏工具：  
  `$ sudo pacman -S spectacle`  
* TG桌面客户端：  
  `$ sudo pacman -S telegram-desktop`  
* 简单计算器：  
  `$ sudo pacman -S kcalc`  
* 文本编辑器：  
  `$ sudo pacman -S gvim`  
  安装gvim主要是为了共享系统的剪切版和终端中vim的剪切版，还需要在.vimrc中添加：  
  `set clipboard=unnamedplus`  
* 蓝牙驱动：  
  `$ sudo pacman -S bluez bluez-utils bluedevil`  
  使用时需要手动开启蓝牙服务：  
  `$ sudo systemctl start bluetooth.service`  
* 输入法：  
  `$ sudo pacman -S fcitx kcm-fcitx fcitx-rime fcitx-im`  
  要想正常使用中文输入法，还需要在~/.xprofile中添加：  
  ```
  export GTK_IM_MODULE=fcitx  
  export QT_IM_MODULE=fcitx  
  export XMODIFIERS=@im=fcitx  
  ```

### 配置
* vim  
* fcitx-rime  
* bash  
* zsh  

Repo: [https://github.com/whoisnian/nian](https://github.com/whoisnian/nian)
