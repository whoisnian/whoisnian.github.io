---
layout: post
title: Qt Creator设置项目文件关联
categories: archlinux
last_modified_at: 2017-12-19T11:53:13+08:00
---

> 学期末的数据结构课设，准备用Qt写图形界面，于是就在电脑上装了Qt Creator，用起来感觉还不错。  
> 不过有那么一点不爽，Qt Creator创建的Project会包含一个`.pro`文件，打开后应该就是整个项目，但是却被系统识别为`text/plain`，设置默认打开程序时会影响到所有`text/plain`文件，这。。。就很不友好了。  

<!-- more -->

既然`.pro`文件不能被识别出单独的MIME type，那么我们就创建一个给它呗。ArchWiki就有创建自定义类型的[例子](https://wiki.archlinux.org/index.php/Default_applications#New_MIME_types)哦，我们稍微改一下就好了。 //Arch大法吼啊！  
设置`.pro`文件关联到自定义类型`text/qt-project-file`：
* `$ mkdir -p ~/.local/share/mime/packages`  
* `$ vim ~/.local/share/mime/packages/qt-project-file.xml`  

```
<?xml version="1.0" encoding="UTF-8"?>
<mime-info xmlns="http://www.freedesktop.org/standards/shared-mime-info">
    <mime-type type="text/qt-project-file">
        <comment>Qt Project File</comment>
        <icon name="QtProject-qtcreator"/>
        <glob-deleteall/>
        <glob pattern="*.pro"/>
    </mime-type>
</mime-info>
```
就是告诉系统把`.pro`结尾的文件识别为`text/qt-project-file`，然后关联了一个`QtProject-qtcreator`的图标给它，方便~~在茫茫人海中~~一眼找到。  
接着更新MIME 数据：  
* `$ update-mime-database ~/.local/share/mime`  

现在在给它设置默认用Qt Creator打开吧，已经不会影响其他的纯文本文件了：  
* `$ vim ~/.local/share/applications/mimeapps.list`  
加上`text/qt-project-file=org.qt-project.qtcreator.desktop`。  

现在就很舒服了（手动滑稽）。
