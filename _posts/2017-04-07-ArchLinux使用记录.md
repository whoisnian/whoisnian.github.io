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
  SigLevel = Optional TrustAll  
  Server = http://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch  
  ```
  然后导入 GPG key：  
  `sudo pacman -Syy && sudo pacman -S archlinuxcn-keyring`  

### 常用软件
* 网页浏览器：  
  `$ sudo pacman -S google-chrome`  
* 科学上网：  
  `$ sudo pacman -S shadowsocks-qt5`  
* 音乐播放器：  
  `$ sudo pacman -S netease-cloud-music`  
* 视频播放器：  
  `$ sudo pacman -S vlc`  
* 图像查看器：  
  `$ sudo pacman -S gwenview`  
* PDF阅读器：  
  `$ sudo pacman -S okular`  
* 压缩包管理器：  
  `$ sudo pacman -S ark`  
* 截屏工具：  
  `$ sudo pacman -S spectacle`  
* Visual Studio Code:    
  `$ sudo pacman -S visual-studio-code`  
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
  由于wps-office中包含宋体，而大多数应用程序将宋体作为默认字体，然后这些应用程序的字体就会变得特别不舒服。  
  只在KDE的系统设置中修改字体选项不起作用，修改用户的字体配置文件可以解决：  
  `$ vim ~/.config/fontconfig/fonts.conf`  
  ```
  <?xml version='1.0'?>
  <!DOCTYPE fontconfig SYSTEM 'fonts.dtd'>
  <fontconfig>

      <!--设置默认字体，英文用Hack字体，中文用文泉驿字体-->
      <alias>
          <family>serif</family>
          <prefer>
              <family>Hack</family>
              <family>WenQuanYi Micro Hei</family>
          </prefer>
      </alias>
      <alias>
          <family>sans-serif</family>
          <prefer>
              <family>Hack</family>
              <family>WenQuanYi Micro Hei</family>
          </prefer>
      </alias>
      <alias>
          <family>monospace</family>
          <prefer>
              <family>Hack</family>
              <family>WenQuanYi Micro Hei Mono</family>
          </prefer>
      </alias>

      <!-- 将宋体用文泉驿字体替换-->
 
      <alias binding="same">
          <family>宋体</family>
          <family>新宋体</family>
          <family>SimSun</family>
          <accept>
              <family>WenQuanYi Micro Hei</family>
          </accept>
      </alias>

  </fontconfig>
  ```

### 配置
* vim  
* fcitx-rime  
* bash  
* zsh  

Repo: [https://github.com/whoisnian/nian](https://github.com/whoisnian/nian)
