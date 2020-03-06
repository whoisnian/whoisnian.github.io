---
layout: post
title: Webhooks服务器端自动同步GitHub
categories: Git
---

> 我的Jekyll博客是放在阿里云上的，因此每次在本地修改文章后上传到GitHub，还需要登录服务器从GitHub同步下来，并重新构建一次。  
> 还好GitHub有Webhooks这个东西，于是就自己配置了一下。  

<!-- more -->

### Webhooks
* Webhooks的作用就是GitHub的repo触发事件时，可以post特定的数据到指定的网址。  
  这里就是让GitHub在检测到repo被更新后访问服务器上的一个PHP页面，然后PHP调用系统的shell命令对网站进行更新。

### 服务器端
* 以我的博客为例，先在网站上创建一个PHP文件，内容如下：  
  ```
  <?php
  echo shell_exec("cd /var/www/blog && git pull && jekyll build");
  ?>
  ```
  假设链接为`https://whoisnian.com/webhook.php`（当然是假的）。
* 由于PHP是以apache用户的身份运行命令，所以会出现Permission denied的情况，给网站目录加上apache用户的修改权限即可。  
  我的解决方法是将网站权限设为777，由于是静态页面，安全上应该不会有什么问题。  
  `$ sudo chmod -R 777 /var/www/blog`
* 在apache用户修改文件后，文件所有者可能会变为apache，而且权限也会更改。  
  但是git默认会追踪文件权限修改，下一次再次`git pull`的时候就会提示有文件冲突，导致从GitHub同步失败。因此需要在网站目录的git配置文件中关闭追踪文件权限修改：  
  `$ vim /var/www/blog/.git/config`  
  将`filemode = true`改为`filemode = false`

### GitHub
* 进入自己repo的Webhooks设置页面，选择Add webhook，只需要在Payload URL栏填写上自己网站上刚创建的PHP文件的链接即可，例如`https://whoisnian.com/webhook.php`，其它的选项不需要更改，然后确认Add webhook。  
  ![Webhooks](/public/image/Webhooks.webp)
* 以后再次`git push`之后GitHub就会通过这个Webhook来自动更新服务器端对应的repo了，你也可以在Webhooks设置页面查看返回内容来确定服务器端是否运行正常。  
  ![Webhooks_log](/public/image/Webhooks_log.webp)
