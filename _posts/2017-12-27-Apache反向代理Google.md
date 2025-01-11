---
layout: post
title: Apache反向代理Google
categories: server
last_modified_at: 2017-12-30T16:17:27+08:00
---

> 偶然间发现了一个反向代理 Google 的网站，感觉用起来很方便，于是就想自己也搭一个玩玩。  

<!-- more -->

### 服务器
服务器就用的自己 Vultr 上的 Arch Linux，环境是 LAMP。  
额。。。貌似设置反向代理就与 Apache 有关，网上有好多使用 nginx 进行反向代理的教程，然而我并不想为此再装一个 nginx，所以我这里就用服务器上已有的 Apache/2.4.29 了。  

### 设置反向代理
* 开启 Apache 的代理模块：  
  `$ sudo vim /etc/httpd/conf/httpd.conf`  
  取消`proxy_module`，`proxy_connect_module`，`proxy_http_module`，`ssl_module`四个模块的注释。  
  proxy_module：提供了基本的代理功能。  
  proxy_connect_module：用于代理时的SSL。  
  proxy_http_module：用于http协议，即HTTP/0.9，HTTP/1.0和HTTP/1.1。  
  ssl_module：为服务器提供SSL支持。  
  （各代理模块具体功能可参考[官方文档](https://httpd.apache.org/docs/2.4/mod/mod_proxy.html)）  

* 设置用于反向代理的虚拟主机，绑定自己的域名：  
  ```
  <VirtualHost *:80>
      ServerAdmin webmaster@example.com
      ServerName google.example.com
      ErrorLog "/var/log/httpd/google.example.com-error_log"
      CustomLog "/var/log/httpd/google.example.com-error_log" common

      # Proxy
      SSLProxyEngine On
      ProxyRequests Off
      ProxyPass / https://www.google.com/
      ProxyPassReverse / https://www.google.com/
  </VirtualHost>
  ```
  重启 Apache 并设置好DNS解析之后就可以通过你自己设置的`google.example.com`来访问 Google 了。  
  
**注：开启SSL是由于Google的https，不开启的话访问`http://google.example.com`会被跳转到`https://www.google.com`，被坑了好久，最终在stackoverflow上偶然看到一个[问题](https://stackoverflow.com/questions/16130303/how-to-proxy-http-to-https-using-apache-httpd-v2-2)之后才解决。** 

**警告：仅通过http进行反向代理应该是比较容易被识别到的，所在服务器存在被封掉IP的风险。**  
