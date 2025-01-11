---
layout: post
title: Raspberry图形界面下鼠标移动缓慢
categories: raspberry
last_modified_at: 2017-10-26T22:24:57+08:00
---

> 树莓派使用图形界面时发现鼠标移动极为缓慢，即使调高DPI或者换用鼠标也没有改善。

<!-- more -->

### 问题  
把一个显示屏外接到了树莓派上，想试一试树莓派的图形界面能否满足我的日常使用需要。  
刚开始是`archlinuxarm + awesome wm`，当打开`Chromium`时卡顿明显，于是换成了官方的`raspbian + lxde`，系统流畅性上感觉变好了一点，但使用浏览器过程中还是会卡。  
两者共同遇到的问题就是插上去的usb鼠标移动速度慢到无法忍受，仿佛鼠标的DPI极低一样。后来终于找到了解决方法。

### 解决
* 手动修改鼠标的轮询间隔：  
  `$ sudo nano /boot/cmdline.txt`  
  然后在里面一行最后添加`usbhid.mousepoll=0`。  
* 重启树莓派，鼠标终于可以正常使用了。  
  `$ sudo reboot`

__注：__ 通过图形界面日常使用树莓派还是别想了，让他静静地做一个微型Linux主机吧，有事时通过SSH来访问才是正确的姿势。
