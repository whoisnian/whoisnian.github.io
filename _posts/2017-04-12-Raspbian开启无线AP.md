---
layout: post
title: Raspbian开启无线AP
categories: Raspberry
---

> 开启无线AP后，树莓派就成为了一个无线路由器，而且以后只需要通过WIFI连接热点，就可以SSH连接树莓派了。

<!-- more -->

### 安装软件  
* 更新软件仓库：  
  `$ sudo apt-get update`  
  `$ sudo apt-get upgrade`  
* 安装需要的软件：  
  `$ sudo apt-get install hostapd dnsmasq`  
* 删除默认无线客户端：  
  `$ sudo apt-get remove wpasupplicant`  
* 暂时关闭服务：  
  `$ sudo systemctl stop hostapd`  
  `$ sudo systemctl stop dnsmasq`  


### 设置转发
* 允许IPv4转发：  
  `$ sudo vim /etc/sysctl.conf`  
  取消`net.ipv4.ip_forward=1`的注释。  
* 开启IPv4转发：  
  `$ sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE`  
  `$ sudo iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT`  
  `$ sudo iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT`  
* 保存转发配置：  
  `$ sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"`  
* 设置开机时自动加载该配置：  
  `$ sudo nano /etc/rc.local`  
  在`exit 0`前添加：  
  `iptables-restore < /etc/iptables.ipv4.nat`  

### 配置静态IP
* 禁止DHCP服务给wlan0分配IP：  
  `$ sudo vim /etc/dhcpcd.conf`  
  在文件末尾添加以下内容，但要保证在其它interface行之前：  
  `denyinterfaces wlan0`  
* 设置wlan0静态IP：  
  `$ sudo vim /etc/network/interfaces`  
  将其中wlan0部分修改为：  
  ```
  allow-hotplug wlan0
  iface wlan0 inet static
      address 4.3.2.1
      netmask 255.255.255.0
      network 4.3.2.0
  ```
* 重启dhcpcd服务并刷新wlan0配置：  
  `$ sudo systemctl restart dhcpcd`  
  `$ sudo ifdown wlan0`  
  `$ sudo ifup wlan0`  

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
  wmm_enabled=1
  macaddr_acl=0
  auth_algs=1
  ignore_broadcast_ssid=0
  wpa=2
  wpa_passphrase=12345678
  wpa_key_mgmt=WPA-PSK
  rsn_pairwise=CCMP
  ```
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
  `$ sudo vim /etc/dnsmasq.d/dns`  
  修改为：  
  ```
  expand-hosts  
  neg-ttl=60  
  max-ttl=3600  
  max-cache-ttl=3600  
  localise-queries  
  bogus-priv  
  stop-dns-rebind  
  rebind-localhost-ok  
  domain-needed  
  cache-size=4096  
  domain=local,4.3.2.1/24,local  
  ```
* 配置本地DHCP：  
  `$ sudo vim /etc/dnsmasq.d/dhcp`  
  修改为：  
  ```
  no-dhcp-interface=eth0  
  dhcp-range=lan,4.3.2.50,4.3.2.100  
  dhcp-option=tag:lan,option:router,4.3.2.1  
  dhcp-option=tag:lan,option:dns-server,4.3.2.1  
  dhcp-broadcast=tag:needs-broadcast  
  dhcp-authoritative  
  dhcp-leasefile=/var/run/dnsmasq/dhcp.lease  
  ```
* 重启服务：  
  `$ sudo systemctl restart dnsmasq`  
