---
layout: post
title: ArchLinux安装配置WineQQ
categories: ArchLinux
---

> 体验了一段时间通过Virtualbox的无缝模式使用QQ。  
> 开个共享文件夹互相拷文件；然后打开共享剪切版，可以在Linux下截图并选择复制到剪切版，到QQ对话框里直接粘贴就行了。而且Win Thin PC用起来也没有一般Win7的那种反应总是慢了一点点的感觉。整体感觉还不错。  
> 然而虚拟机开关机真的好慢啊QAQ，所以我又尝试了一下WineQQ。  

<!-- more -->

### 安装Wine
* 由于我的Arch是64位，所以需要先启用Multilib仓库：  
  `$ sudo vim /etc/pacman.conf`  
  取消下面两项的注释：  
  `[multilib]`  
  `Include = /etc/pacman.d/mirrorlist`  
  更新仓库：  
  `$ sudo pacman -Syy`  
* 安装Wine：  
  `$ sudo pacman -S wine wine_gecko wine-mono`  
  (wine_gecko和wine-mono分别用于运行依赖于Internet Explorer和.NET的程序)  
* 工具：  
  * winecfg：Wine的图形界面配置程序，`winecfg`命令启动。  
  * control.exe：Windows控制面板的Wine实现，`wine control`命令启动。  
  * regedit：Wine的注册表编辑器，最主要的配置工具，`regedit`命令启动。  

**注：Wine的各种工具都不需要root权限，也最好不要使用root用户来运行Wine。**

### 字体问题
* TIM需要用到windows字体，因此我就直接从一台Win10上直接拷来了`C:\windows\Fonts`下的所有字体，并放到了`/usr/share/fonts/WindowsFonts/`下。  
  （如果拷过来这些字体后导致你的浏览器字体变得特别丑，请参考[ArchLinux使用记录](https://whoisnian.com/2017/04/07/ArchLinux%E4%BD%BF%E7%94%A8%E8%AE%B0%E5%BD%95/)中的字体配置文件用其他字体来替换宋体）  
* 字体大小请在winecfg中以下页面修改DPI进行调节：  
  ![Wine-TIM-Font](/public/image/wine_font.png)
* 如果Wine中的字母和数字字体边缘锯齿化严重，需要导入注册表进行修改：
  新建文本文件：  
  ```
  REGEDIT4
  [HKEY_CURRENT_USER\Software\Wine\X11 Driver]
  "ClientSideAntiAliasWithCore"="Y"
  "ClientSideAntiAliasWithRender"="Y"
  "ClientSideWithRender"="Y"
  ```
  和  
  ```
  REGEDIT4
  [HKEY_CURRENT_USER\Control Panel\Desktop]
  "FontSmoothing"="2"
  "FontSmoothingType"=dword:00000002
  "FontSmoothingGamma"=dword:00000578
  "FontSmoothingOrientation"=dword:00000001
  ```
  分别在regedit中导入。  
* 字体乱码，例如汉字显示为空心方块，可能是语言设置问题。我在Archlinux的KDE中文环境下没有遇到。

### 安装TIM
* QQ的企业版，当成轻聊版来用也是不错的，直接到[TIM官网](http://tim.qq.com)下载Windows安装包,再通过Wine来安装：  
  `$ wine ~/Downloads/TIM1.2.0.exe`  
* 运行TIM（wine + .exe文件路径）：  
  `$ wine ~/.wine/drive_c/Program\ Files\ \(x86\)/Tencent/TIM/Bin/TIM.exe`  
* 登录框出现后可能会无法输入帐号，但可以扫码登录。输入帐号解决办法如下：  
  `$ winecfg`  
  为TIM增加程序设置：  
  ![Wine-TIM](/public/image/wine_tim_1.png)  
  在函数库中为TIM新增函数库顶替msvcp60，riched20，riched32三项，均设置为原装先于新建：  
  ![Wine-TIM](/public/image/wine_tim_2.png)  
  此时TIM即可输入帐号密码进行登录。  
* 效果图：  
  * 登录界面：  
  ![log-in](/public/image/wine_tim_show1.png)
  * 消息界面：  
  ![message](/public/image/wine_tim_show2.png)
  * 托盘按钮：  
  ![button](/public/image/wine_tim_bar.png)

### 已知问题
* 登录框的密码输入框可能需要多次点击才能激活。  
* 记住密码与自动登录功能无法使用。  
* 切换聊天对象时TIM窗口闪屏，而且可能不会立即刷新窗口，可以手动点击其他地方引起刷新。  
* 选择表情页面切换有时不流畅。  
* 在聊天框中直接点击图片查看原图时，TIM打开的图片不会自动刷新，可以手动缩放引起刷新。  
* QQ聊天中的网页链接TIM使用Wine内置IE浏览器打开，效率低下。（[已解决](/2017/08/14/设置Wine调用Linux浏览器代替内置IE/)）  
* 运行过程中TIM崩溃，并提示与msls31.dll有关的错误。我在打开某个QQ群时有很大机率会出现。（用winetricks安装msls31.ddl后未再次出现）  
  ![Wine-TIM-Error](/public/image/wine_tim_error.png)
