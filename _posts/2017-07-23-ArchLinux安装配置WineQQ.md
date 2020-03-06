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
  `$ sudo pacman -S wine wine_gecko wine-mono winetricks`  
  (wine_gecko和wine-mono分别用于运行依赖于Internet Explorer和.NET的程序)  
  (winetricks用来完善windows的一些基础组件)
* 工具：  
  * winecfg：Wine的图形界面配置程序，`winecfg`命令启动。  
  * control.exe：Windows控制面板的Wine实现，`wine control`命令启动。  
  * regedit：Wine的注册表编辑器，最主要的配置工具，`regedit`命令启动。  

**注：Wine的各种工具都不需要root权限，也最好不要使用root用户来运行Wine。**

### 字体问题
* 对于Wine自身的中文变方块，修改注册表可以解决。  
  新建一个zh.reg文件：  
  ```
  REGEDIT4

  [HKEY_LOCAL_MACHINE\Software\Microsoft\Windows NT\CurrentVersion\FontLink\SystemLink]
  "Lucida Sans Unicode"="wqy-microhei.ttc"
  "Microsoft Sans Serif"="wqy-microhei.ttc"
  "Microsoft YaHei"="SourceHanSansCN-Medium.otf"
  "MS Sans Serif"="wqy-microhei.ttc"
  "Tahoma"="wqy-microhei.ttc"
  "Tahoma Bold"="wqy-microhei.ttc"
  "SimSun"="wqy-microhei.ttc"
  "Arial"="wqy-microhei.ttc"
  "Arial Black"="wqy-microhei.ttc"
  "宋体"="SourceHanSansCN-Medium.otf"
  "新細明體"="SourceHanSansCN-Medium.otf"
  ```
  保存文件后运行`regedit zh.reg`，重新打开winecfg可以看到Wine自身的中文乱码已经消失。
* TIM需要用到windows字体，这里可以直接使用winetricks进行安装：  
  `$ winetricks corefonts cjkfonts`  
  （如果安装这些字体影响到了你的系统字体，请参考[ArchLinux使用记录](https://whoisnian.com/2017/04/07/ArchLinux%E4%BD%BF%E7%94%A8%E8%AE%B0%E5%BD%95/)中的字体配置文件用其他字体来替换宋体）  
* 字体大小可以在winecfg中以下页面修改DPI进行调节：  
  ![Wine-TIM-Font](/public/image/wine_font.webp)
* 如果Wine中字体锯齿化严重或字体模糊，需要导入注册表进行修改：  
  ```
  REGEDIT4
  [HKEY_CURRENT_USER\Software\Wine\X11 Driver]
  "ClientSideAntiAliasWithCore"="Y"
  "ClientSideAntiAliasWithRender"="Y"
  "ClientSideWithRender"="N"

  [HKEY_CURRENT_USER\Control Panel\Desktop]
  "FontSmoothing"="2"
  "FontSmoothingType"=dword:00000002
  "FontSmoothingGamma"=dword:00000578
  "FontSmoothingOrientation"=dword:00000001
  ```

### 安装TIM
* QQ的企业版，当成轻聊版来用也是不错的，直接到[TIM官网](https://office.qq.com)下载Windows安装包,再通过Wine来安装：  
  `$ wine ~/Downloads/TIM2.0.0.exe`  
* 运行TIM（wine + .exe文件路径）：  
  `$ LC_ALL=zh_CN.UTF-8 wine ~/.wine/drive_c/Program\ Files\ \(x86\)/Tencent/TIM/Bin/TIM.exe`  
* 聊天页面的消息可能无法显示，需要以下组件：  
  `$ winetricks riched20 riched30`    
* 效果图：  
  * 登录界面：  
  ![log-in](/public/image/wine_tim_show1.webp)
  * 消息界面：  
  ![message](/public/image/wine_tim_show2.webp)
  * 托盘按钮：  
  ![button](/public/image/wine_tim_bar.webp)

### 已知问题
* 记住密码与自动登录功能无法使用。  
* 选择表情页面切换有时不流畅。  
* 在聊天框中直接点击图片查看原图时，TIM打开的图片不会自动刷新，可以手动缩放引起刷新。  
* Windows版本选择Win xp时TIM界面部分文字显示不正常，而选择Win 7时则为正常显示。  
* 切换聊天对象时可能不会立即刷新窗口，可以手动点击其他地方引起刷新。（使用TIM最新版未发现该问题）  
* QQ聊天中的网页链接TIM使用Wine内置IE浏览器打开，效率低下。（[已解决](/2017/08/14/设置Wine调用Linux浏览器代替内置IE/)）  
