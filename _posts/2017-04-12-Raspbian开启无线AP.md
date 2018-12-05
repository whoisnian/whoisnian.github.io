---
layout: post
title: Raspbian开启无线AP
categories: Raspberry
---

> 开启无线AP后，树莓派就成为了一个无线路由器，而且以后只需要通过WIFI连接热点，就可以SSH连接树莓派了。

<!-- more -->


**注：本篇博客内容过于古老，推荐参考官方文档[Setting up a Raspberry Pi as an access point in a standalone network (NAT)](https://www.raspberrypi.org/documentation/configuration/wireless/access-point.md)。**

### 安装软件  
* 更新软件仓库：  
  `$ sudo apt-get update`  
  `$ sudo apt-get upgrade`  
* 安装需要的软件：  
  `$ sudo apt-get install hostapd dnsmasq`    
* 暂时关闭服务：  
  `$ sudo systemctl stop hostapd`  
  `$ sudo systemctl stop dnsmasq`  

### 设置转发
* 允许IPv4转发：  
  `$ sudo vim /etc/sysctl.conf`  
  取消`net.ipv4.ip_forward=1`的注释。  
* 开启IPv4转发：  
  `$ sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE`    
* 保存转发配置：  
  `$ sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"`  
* 设置开机时自动加载该配置：  
  `$ sudo nano /etc/rc.local`  
  在`exit 0`前添加：  
  `iptables-restore < /etc/iptables.ipv4.nat`  

### 配置静态IP  
* 设置wlan0静态IP：  
  `$ sudo vim /etc/dhcpcd.conf`  
  在文件末尾添加：  
  ```
  interface wlan0
    static ip_address=4.3.2.1/24
    nohook wpa_supplicant
  ```
* 重启dhcpcd服务：  
  `$ sudo systemctl restart dhcpcd`   

### 配置接入点参数
* 编辑hostapd配置文件：  
  `$ sudo vim /etc/hostapd/hostapd.conf`  
  添加以下内容，注意修改热点名称（ssid）以及密码（wpa_passphrase）：  
  ```
  interface=wlan0
  driver=nl80211
  ssid=Raspberry AP
  hw_mode=g
  channel=7
  wmm_enabled=0
  macaddr_acl=0
  auth_algs=1
  ignore_broadcast_ssid=0
  wpa=2
  wpa_passphrase=12345678
  wpa_key_mgmt=WPA-PSK
  wpa_pairwise=TKIP
  rsn_pairwise=CCMP
  ```
  **注：hw_mode选项可为：**
  * **a = IEEE 802.11a (5 GHz)**
  * **b = IEEE 802.11b (2.4 GHz)**
  * **g = IEEE 802.11g (2.4 GHz)**
  * **ad = IEEE 802.11ad (60 GHz)**
* 启用该配置文件：  
  `$ sudo vim /etc/default/hostapd`  
  找到 #DAEMON_CONF 一行，替换为：  
  `DAEMON_CONF="/etc/hostapd/hostapd.conf"`  
* 启动服务并设置开机启动：  
  `$ sudo systemctl start hostapd`  
  `$ sudo systemctl start dnsmasq`  
  `$ sudo systemctl enable hostapd`  
  `$ sudo systemctl enable dnsmasq`  

### 配置DHCP服务
* 配置本地DNS：  
  `$ sudo vim /etc/dnsmasq.conf`  
  修改为：  
  ```
  interface=wlan0  
  dhcp-range=4.3.2.2,4.3.2.20,255.255.255.0,24h  
  ```
* 重启服务：  
  `$ sudo systemctl restart dnsmasq`  

### 重启树莓派
* `$ sudo reboot`
