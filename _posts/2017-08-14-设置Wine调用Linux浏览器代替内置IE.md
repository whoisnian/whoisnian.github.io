---
layout: post
title: 设置Wine调用Linux浏览器代替内置IE
categories: archlinux
---

> 装完WineQQ之后在QQ中点击链接，打开邮箱等，默认调用的是Wine内置IE，十分辣眼睛。  
> 为什么不用我Linux上装的Chrome浏览器呢？

<!-- more -->

先找到 IE 所在目录，我的是`~/.wine/drive_c/Program Files (x86)/Internet Explorer`，里面的`iexplore.exe`就是那个辣眼睛的 IE。  
* 备份是个好习惯：  
  `$ cd ~/.wine/drive_c/Program\ Files\ \(x86\)/Internet\ Explorer/`  
  `$ mv iexplore.exe iexplore.exe.backup.exe`  
* 新建一个 IE 的替代品：  
  `$ vim iexplore.exe`  
  内容如下：  
  ```
  #!/bin/bash
  # Allow users to override command-line options
  if [[ -f ~/.config/chrome-flags.conf ]]; then
	CHROME_USER_FLAGS="$(cat ~/.config/chrome-flags.conf)"
  fi
  # Launch
  exec /opt/google/chrome/google-chrome $CHROME_USER_FLAGS "$@"
  ```
* 添加可执行权限：  
  `$ chmod 755 iexplore.exe`

这样当Wine调用iexplore.exe的时候就会找到这个bash脚本，从而调用Linux上的google-chrome了。
