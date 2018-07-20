---
layout: post
title: ArchLinux使用记录
categories: ArchLinux
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
  Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch  
  ```
  然后导入 GPG key：  
  `$ sudo pacman -Syy && sudo pacman -S archlinuxcn-keyring`  

### 常用软件
* 网页浏览器：  
  `$ sudo pacman -S google-chrome`  
  `$ sudo pacman -S firefox firefox-i18n-zh-cn`  
* 科学上网：  
  `$ sudo pacman -S shadowsocks`  
  然后复制默认配置文件并修改：  
  `$ cd /etc/shadowsocks`  
  `$ sudo cp example.json myconfig.json`  
  `$ sudo vim myconfig.json`  
  启动时执行`sudo systemctl start shadowsocks@myconfig`即可。  
* 音乐播放器：  
  `$ sudo pacman -S netease-cloud-music`  
* 视频播放器：  
  `$ sudo pacman -S vlc`  
* 图像查看器：  
  `$ sudo pacman -S gwenview`  
* PDF阅读器：  
  `$ sudo pacman -S okular poppler-data`  
* 压缩包管理器：  
  `$ sudo pacman -S ark`  
* 截屏工具：  
  `$ sudo pacman -S spectacle`  
* Visual Studio Code:    
  `$ sudo pacman -S visual-studio-code-bin`  
* TG桌面客户端：  
  `$ sudo pacman -S telegram-desktop`  
* 相机：  
  `$ sudo pacman -S kamoso`  
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
* 连接 5 GHz WiFi 热点：  
  `$ sudo pacman -S crda`
* 输入法：  
  `$ sudo pacman -S fcitx kcm-fcitx fcitx-rime fcitx-im`  
  要想正常使用中文输入法，还需要在~/.xprofile中添加：  
  ```
  export GTK_IM_MODULE=fcitx  
  export QT_IM_MODULE=fcitx  
  export XMODIFIERS=@im=fcitx  
  ```
* office套装：  
  `$ sudo pacman -S wps-office ttf-wps-fonts`  
  [这里](/2018/06/13/WPS-Office使用记录/)是我在ArchLinux上使用WPS Office时遇到的几个问题及解决方法。  

### 配置
* vim  
* fcitx-rime  
* bash  
* zsh  

Repo: [https://github.com/whoisnian/nian](https://github.com/whoisnian/nian)
