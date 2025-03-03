---
layout: post
title: ArchLinux使用记录
categories: archlinux
last_modified_at: 2022-10-08T00:54:58+08:00
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
* socks转http代理：  
  `$ sudo pacman -S privoxy`  
* web缓存代理：  
  `$ sudo pacman -S squid`  
* 音乐播放器：  
  `$ sudo pacman -S netease-cloud-music`  
  参考[这里](https://forum.ubuntu.org.cn/viewtopic.php?f=74&t=484624)，在启动命令前加上`env XDG_CURRENT_DESKTOP=DDE`假装自己是Deepin，可解决KDE上网易云音乐托盘图标右键菜单无效的问题。
* 视频播放器：  
  `$ sudo pacman -S vlc`  
* 图像查看器：  
  `$ sudo pacman -S gwenview`  
* 图像编辑器：  
  `$ sudo pacman -S gimp`
* PDF阅读器：  
  `$ sudo pacman -S okular poppler-data`  
* 压缩包管理器：  
  `$ sudo pacman -S ark`  
* 截屏工具：  
  `$ sudo pacman -S spectacle`  
* 屏幕录制工具：  
  `$ sudo pacman -S peek`  
* 按键显示工具：  
  `$ sudo pacman -S screenkey`  
* Visual Studio Code:    
  `$ pikaur -S visual-studio-code-bin`  
* 中英文1:2字体：  
  `$ sudo pacman -S ttf-ubuntu-font-family ttf-mplus`
* API测试工具：  
  `$ sudo pacman -S postman-bin`
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
* 连接 5 GHz WiFi 热点：  
  `$ sudo pacman -S crda`  
* 网络共享：  
  `$ sudo pacman -S dnsmasq`  
* 多线程下载：  
  `$ sudo pacman -S axel`  
* 录屏：  
  `$ sudo pacman -S obs-studio`  
* 有道词典命令行翻译工具：  
  `$ sudo pacman -S ydcv-rs-git`  
* 输入法：  
  `$ sudo pacman -S fcitx5-im fcitx5-chinese-addons fcitx5-pinyin-zhwiki`  
  要想正常使用中文输入法，还需要在~/.xprofile中添加：  
  ```
  export GTK_IM_MODULE=fcitx  
  export QT_IM_MODULE=fcitx  
  export XMODIFIERS=@im=fcitx  
  ```
* KDE应用程序风格（GTK）设置：  
  `$ sudo pacman -S kde-gtk-config`  
* KDE图标查看器（cuttlefish）：  
  `$ sudo pacman -S plasma-sdk`  
* 磁盘速度测试：  
  `$ sudo pacman -S gnome-disk-utility`  
* Windows启动盘制作工具：  
  `$ sudo pacman -S woeusb-git`
* C语言变量定义解释：  
  `$ sudo pacman -S cdecl`  
* 密码管理器：  
  `$ sudo pacman -S keepassxc`  
* 带有编码转换的unzip：  
  `$ sudo pacman -S unzip-iconv`  
* IP扫描工具：  
  `$ sudo pacman -S nmap`  
* KDE全局菜单：  
  `$ sudo pacman -S appmenu-gtk-module libdbusmenu-glib`  
  参考[这里](https://forum.manjaro.org/t/gtk-global-menu-in-plasma-5-13/42112)，全局菜单在KDE自带的小部件中就可以找到，但对于一些应用程序需要额外操作才可以正常显示。  
* office套装：  
  `$ sudo pacman -S wps-office ttf-wps-fonts`  
  [这里](/2018/06/13/WPS-Office使用记录/)是我在ArchLinux上使用WPS Office时遇到的几个问题及解决方法。  
* 串口调试工具：  
  `$ sudo pacman -S comtool`  
* 音乐/视频刮削器：  
  `$ sudo pacman -S picard mediaelch`  
