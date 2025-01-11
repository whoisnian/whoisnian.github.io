---
layout: post
title: Raspbian Installation Guide
categories: raspberry
last_modified_at: 2020-03-06T23:03:34+08:00
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
  `$ sudo touch boot/ssh`  
* 取消挂载：  
  `$ sudo umount boot`  

### SSH连接
* 安装 dnsmasq 后在 NetworkManager 中新建一个连接，选择“有线以太网（共享）”，然后用一根网线连接树莓派和笔记本，接通树莓派电源。  
* 查看以太网口IP地址：  
  `$ ip addr`  
  {% highlight default %}
  ......  
  2: enp6s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000  
    link/ether c8:5b:76:3a:6d:37 brd ff:ff:ff:ff:ff:ff  
    inet 10.42.0.1/24 brd 10.42.0.255 scope global enp6s0  
       valid_lft forever preferred_lft forever  
    inet6 fe80::3150:5745:898e:3350/64 scope link  
       valid_lft forever preferred_lft forever  
  ......  
  {% endhighlight %}
* 搜索树莓派IP地址（注意换成你自己的以太网口IP）：  
  `$ nmap 10.42.0.1/24`  
  {% highlight default %}
  Starting Nmap 7.40 ( https://nmap.org ) at 2017-04-11 13:31 CST  
  Nmap scan report for 10.42.0.1  
  Host is up (0.000045s latency).  
  Not shown: 999 closed ports  
  PORT   STATE SERVICE  
  53/tcp open  domain  
  
  Nmap scan report for 10.42.0.131  
  Host is up (0.0042s latency).  
  Not shown: 999 closed ports  
  PORT   STATE SERVICE  
  22/tcp open  ssh  
  
  Nmap done: 256 IP addresses (2 hosts up) scanned in 2.53 seconds  
  {% endhighlight %}
* 连接树莓派：  
  `$ ssh pi@10.42.0.131`  
  默认密码：raspberry  

### 基础配置
* 进入配置界面：  
  `$ sudo raspi-config`  
  ![raspi-config](/public/image/raspi-config.webp)
* 选择1，修改默认密码。  
* 选择4-I1，修改语言环境。  
* 选择4-I2，修改时区。  
* 选择7-A1，扩展SD卡。  
* 重启：  
  `$ sudo reboot`  

### 启用root账户
* 设置root密码：  
  `$ sudo passwd root`  
* 解锁root账户：  
  `$ sudo passwd --unlock root`  

### 更换镜像源
* 换用清华源：  
  `$ sudo nano /etc/apt/sources.list`  
  注释掉原有的官方源，再加入以下内容：  
  ```
  deb http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ jessie main non-free contrib  
  deb-src http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ jessie main non-free contrib  
  ```
* 更新软件源列表：  
  `$ sudo apt-get update`  
* 更新系统：  
  `$ sudo apt-get upgrade`
