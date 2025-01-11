---
layout: post
title: SSH连接VirtualBox中的archlinux虚拟机
categories: archlinux
last_modified_at: 2017-11-19T01:46:23+08:00
---

> 玩Socket时想要使用VirtualBox来模拟两台电脑的效果，于是就往里面装了一个没有图形界面的archlinux，却在SSH连接时遇到了问题。

<!-- more -->

### 问题
开始时使用“网络地址转换（NAT）”模式，直接`ssh nian@10.0.2.15`，然后就没提示了；  
后来改为“桥接网卡”模式（校园网会分配给虚拟机里面的arch一个独立的IP，很棒哦），再次ssh，依然没有反应。  

### 解决
猜测可能是虚拟机的端口在宿主机之下，所以无法直接连接到虚拟机的对应端口。  

Google之后发现给VirtualBox设置一个端口转发就好了：  
* 虚拟机网卡设置：  
  ```
  连接方式：  
    网络地址转换（NAT）
  端口转发：
    名称：Rule1
    协议：TCP
    主机端口: 2222
    子系统端口：22
  ```
* 之后就可以通过主机的2222端口访问虚拟机的22端口了  
  `$ ssh nian@127.0.0.1 -p 2222`  
  输入密码，搞定。
