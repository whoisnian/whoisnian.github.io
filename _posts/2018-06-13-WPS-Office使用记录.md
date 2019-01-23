---
layout: post
title: WPS Office使用记录
categories: ArchLinux
---

> 在 ArchLinux 上使用 WPS Office 的一些记录。

<!-- more -->

### 安装
* 在添加了archlinuxcn源后可以直接从源中安装wps及部分所需字体：  
  `$ sudo pacman -S wps-office ttf-wps-fonts`  

### 文件关联
* 如果电脑上的`.docx`，`.xlsx`，`.pptx`等文件被识别为`.zip`文件导致无法直接用wps打开，则可以先删除`/usr/share/mime/packages`下的几个`wps-office*.xml`文件：  
  `$ sudo rm /usr/share/mime/packages/wps-office-*`  
  然后更新mime数据：  
  `$ sudo update-mime-database /usr/share/mime`  
  接着打开office文件时只需要在第一次打开时选择打开方式，之后再次遇到同类文件就会直接使用wps打开了。 

### 字体显示
* 虽然`ttf-wps-fonts`中已经包含了部分wps所需字体，但在直接打开微软office文件时依然会遇到各种字体问题，例如刚装完wps时打开包含宋体的word：  
  ![wps_no_simsun](/public/image/wps_no_simsun.png)  
  不仅字体不正常，而且word文件的格式也不正常。  
  因此我手动从Windows上复制过来了系统目录下的Fonts文件夹，并将其内容放入`/usr/share/fonts/WindowsFonts`目录中，最后将该目录权限设置为与其它目录相同，再次打开同一个word文件看起来就正常多了：  
  ![wps_with_simsun](/public/image/wps_with_simsun.png)  

### 字体影响
* 由于在系统中加入了宋体，而部分应用程序又将宋体作为默认字体，因此这些应用程序的字体就会变得特别不舒服。  
  只在KDE的系统设置中修改字体选项不起作用，手动修改字体配置文件可以解决：  
  `$ sudo vim /etc/fonts/local.conf`  
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
  </fontconfig>
  ```
  我当前使用的字体配置文件为[local.conf](https://github.com/whoisnian/nian/blob/master/local.conf)。  

### 主题影响
* 使用过程中遇到问题，在KDE系统设置中选择暗色微风主题后，影响到了wps表格默认的背景色以及字体颜色。  
  例如新创建一个表格后，配色为：  
  ![wps_et_no_style](/public/image/wps_et_no_style.png)  
  尝试发现KDE选择亮色主题就不会影响wps表格配色，但我还是更喜欢暗色微风。  
  偶然在wiki上看到wps条目下关于[使用 GTK+ UI](https://wiki.archlinux.org/index.php/WPS_Office_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#.E4.BD.BF.E7.94.A8_GTK.2B_UI)的说明。在系统设置中将GTK主题设置为亮色微风，然后修改`/usr/share/applications/wps-office-et.desktop`中的内容，将`Exec=/usr/bin/et %f`修改为`Exec=/usr/bin/et -style gtk+ %f`，wps在启动时就会使用设定的GTK主题，表格配色不受影响。而系统中大部分程序都还是使用的QT的暗色微风主题，可以接受。  
  修改完成后效果如下：  
  ![wps_et_with_style](/public/image/wps_et_with_style.png)  
  wps最近更新后遇到doc文档中的宋体变灰色，也是同样的原因，加上`-style gtk+`即可：  
  ![wps_wps_no_style](/public/image/wps_wps_no_style.png)  
  ![wps_wps_with_style](/public/image/wps_wps_with_style.png)  
  KDE桌面环境下，修改完毕后如果直接点击开始菜单项打开的WPS程序配色正常，但是点击关联文件打开的WPS程序配色不正常，则可能是受到了`系统设置 -> 色彩 -> 将颜色应用到非Qt应用程序`的影响，关闭此选项即可。

