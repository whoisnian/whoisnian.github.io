---
layout: post
title: Raspbian Installation Guide
categories: Raspberry
---

> Raspberry 入门配置过程。

<!-- more -->

### 刻录系统到SD卡  
* 在[raspberry官网](https://www.raspberrypi.org/downloads/raspbian/)上下载想要使用的 Raspbian 镜像。  
* 解压下载的 zip 文件：  
  `$ unzip 2017-03-02-raspbian-jessie-lite.zip`  
* 插入SD卡并格式化：  
  `$ sudo wipefs --all /dev/sdc`  
* 开始刻录（注意修改镜像路径以及SD卡名称）：  
  `$ sudo dd bs=4M if=2017-03-02-raspbian-jessie-lite.img of=/dev/sdc`  
  （注意这个过程可能需要五分钟左右，而且中间不会显示写入的过程）  
* 等待提示写入完成之后再运行 sync 确保缓存已经写入：  
  `$ sync`  

### 开启SSH
* 2016年11月之后发布的 Raspbian，默认情况下已禁用SSH服务器。使用之前需要手动启用，官方提供的方法是在SD卡的引导分区下创建一个名为`ssh`而且无扩展名的文件。  
* 挂载SD卡引导分区：  
  `$ mkdir boot`  
  `$ sudo mount /dev/sdc1 boot`  
* 创建文件：  
  `$ sudo touch ssh`  
* 取消挂载：  
  `$ sudo unmount boot`  

### 启动  

__Loading......__
