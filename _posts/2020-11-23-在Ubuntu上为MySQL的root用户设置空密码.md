---
layout: post
title: 在Ubuntu上为MySQL的root用户设置空密码
categories: Server
---

> 自己有一台操作系统为`Ubuntu 18.04.5 LTS`的测试服务器，使用`ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '';`为 MySQL 的root用户设置了空密码。  
> 但一直正常运行的测试应用偶尔会出现连接失败的情况，到服务器上查看后发现数据库root用户的`authentication plugin`变回了`auth_socket`，再次手动改成`mysql_native_password`就好了。  
> 由于问题出现的频率不高，修复也并不麻烦，所以前几次都是手动处理了。最近又遇到了这个问题，决定找到具体原因彻底解决掉。  

<!-- more -->

## 问题分析
为了与生产环境保持一致，测试服务器上运行的 MySQL 版本是 5.7，默认的`authentication plugin`还是`mysql_native_password`，跟 MySQL 8.0 对默认`authentication plugin`的调整没什么关系，还是要从服务器日志开始下手。  

执行`sudo systemctl status mysql`直接显示的内容比较少，所以使用`sudo journalctl -ru mysql`查看`systemd-journald`记录的所有日志，`-r`指定按时间逆序排列，`-u mysql`指定服务是`mysql.service`。内容大致如下：  
```log
Oct 28 06:33:28 ubuntu systemd[1]: Started MySQL Community Server.
Oct 28 06:33:27 ubuntu systemd[1]: Starting MySQL Community Server...
Oct 28 06:33:16 ubuntu systemd[1]: Stopped MySQL Community Server.
Oct 28 06:33:15 ubuntu systemd[1]: Stopping MySQL Community Server...
Oct 20 07:49:10 ubuntu systemd[1]: Started MySQL Community Server.
Oct 20 07:49:09 ubuntu systemd[1]: Starting MySQL Community Server...
-- Reboot --
Oct 20 07:48:43 ubuntu systemd[1]: Stopped MySQL Community Server.
Oct 20 07:48:41 ubuntu systemd[1]: Stopping MySQL Community Server...
Oct 17 04:12:23 ubuntu systemd[1]: Started MySQL Community Server.
Oct 17 04:12:21 ubuntu systemd[1]: Starting MySQL Community Server...
```

日志显示在下午14:33的时候 MySQL 被重启了一次。自己用的是 zsh，所以可以通过`history -i`查看带有时间戳的命令历史，但在历史记录中并未发现涉及 MySQL 的命令，`uptime`查看服务器的运行时间也十分正常，近期没有被重启过。  
将root用户改成空密码后手动重启`mysql.service`，root用户的`authentication plugin`没有发生改变，所以推测是某些特殊原因导致了 MySQL 重启并重置了root用户的认证方式，接下来就去寻找这个“特殊原因”究竟是什么。

除了`systemd-journald`记录的日志外，MySQL 本身在`/var/log/mysql/`目录下也存放有`error.log`，但是在里面只有正常 shutdown 和 start 的记录，下午14:33的重启和刚刚的手动重启在日志里并没有明显区别。


### To Be Continued...
