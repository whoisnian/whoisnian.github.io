---
layout: post
title: 在Ubuntu上为MySQL的root用户设置空密码
categories: server
---

> 自己有一台操作系统为`Ubuntu 18.04.5 LTS`的测试服务器，使用`ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '';`为 MySQL 的root用户设置了空密码。  
> 但一直正常运行的测试应用偶尔会出现连接失败的情况，到服务器上查看后发现数据库root用户的`authentication plugin`变回了`auth_socket`，再次手动改成`mysql_native_password`就好了。  
> 由于问题出现的频率不高，修复也并不麻烦，所以前几次都是手动处理了。最近又遇到了这个问题，决定找到具体原因彻底解决掉。  

<!-- more -->

## 问题分析
为了与生产环境保持一致，测试服务器上运行的 MySQL 版本是 5.7，默认的`authentication plugin`还是`mysql_native_password`，跟 MySQL 8.0 对默认`authentication plugin`的调整没什么关系，还是要从服务器日志开始下手。  

执行`sudo systemctl status mysql`直接显示的内容比较少，所以使用`sudo journalctl -ru mysql`查看`systemd-journald`记录的所有日志，`-r`指定按时间逆序排列，`-u mysql`指定服务是`mysql.service`。内容大致如下：  
```apib
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
将root用户改成空密码后手动重启`mysql.service`，重启并没有导致root用户的`authentication plugin`发生改变，所以推测是某些特殊原因导致了 MySQL 重启并重置了root用户的认证方式，接下来就要找到这个“特殊原因”究竟是什么。  

除了`systemd-journald`记录的日志外，MySQL 自身在`/var/log/mysql/`目录下也存放有`error.log`，但是在里面只有正常 shutdown 和 start 的记录，下午14:33的重启和刚刚的手动重启在日志里并没有明显区别。  

之前查看 Docker 的`log-driver`时了解到 Ubuntu 上还有一个`rsyslog.service`，`/var/log/`目录下的`auth.log`、`syslog`、`kern.log`、`mail.log`等就是由该服务收集的，然后由`logrotate`定期归档整理。其中`/var/log/syslog`文件存储了`rsyslog`服务收集的所有日志，这些日志在这里记录一份后再根据相应规则写入单独的文件。  

那 Ubuntu 上同时运行`rsyslog`和`systemd-journald`两个日志服务不显得很多余吗？对比了一下感觉还行，两个服务的特征及各自负责的范围有所差异，并不会一团乱麻。`systemd-journald`收集`systemctl`启动的各项服务输出内容作为日志，服务本身不需要进行配置，直接使用标准输出即可；而`rsyslog`如果要提取某个程序的日志，不仅要修改`rsyslog`的配置文件，程序本身也要使用对应的日志写入方式，它在 Ubuntu 上负责的主要就是非`systemd`的一些系统服务。  
除此之外，`man 5 journald.conf`还能看到一个名称为`ForwardToSyslog`的选项，而且其默认值为`true`，所以`/var/log/syslog`其实是 Ubuntu 上最全的一份日志。使用`sudo`打开该文件，翻到对应的时间点，在 MySQL 的重启记录附近看到了这样的内容：
```apib
Oct 28 06:32:53 ubuntu systemd[1]: Starting Daily apt upgrade and clean activities...
Oct 28 06:33:14 ubuntu systemd[1]: Reloading.
Oct 28 06:33:15 ubuntu systemd[1]: Stopping MySQL Community Server...
Oct 28 06:33:16 ubuntu systemd[1]: Stopped MySQL Community Server.
Oct 28 06:33:16 ubuntu systemd[1]: Reloading.
Oct 28 06:33:27 ubuntu systemd[1]: message repeated 4 times: [ Reloading.]
Oct 28 06:33:27 ubuntu systemd[1]: Starting MySQL Community Server...
Oct 28 06:33:28 ubuntu systemd[1]: Started MySQL Community Server.
Oct 28 06:33:28 ubuntu systemd[1]: Reloading.
Oct 28 06:33:31 ubuntu systemd[1]: Started Daily apt upgrade and clean activities.
```
🤔 服务器还会每天自动`apt upgrade`？于是去查看了`apt`的日志文件`/var/log/apt/history.log`，在里面看到了以下内容：
```apib
Start-Date: 2020-10-28  06:33:14
Commandline: /usr/bin/unattended-upgrade
Upgrade: mysql-server-5.7:amd64 (5.7.31-0ubuntu0.18.04.1, 5.7.32-0ubuntu0.18.04.1), mysql-server-core-5.7:amd64 (5.7.31-0ubuntu0.18.04.1, 5.7.32-0ubuntu0.18.04.1)
End-Date: 2020-10-28  06:33:28
```
搜了一下`unattended-upgrade`，还真是 Ubuntu 从 Debian 继承过来的自动更新程序。`man unattended-upgrade`可以查看程序简要信息，它对自己的介绍是`automatic installation of security (and other) upgrades`，默认配置文件位于`/etc/apt/apt.conf.d/50unattended-upgrades`，过滤掉其中的注释后还剩下：
```config
Unattended-Upgrade::Allowed-Origins {
        "${distro_id}:${distro_codename}";
        "${distro_id}:${distro_codename}-security";
        "${distro_id}ESMApps:${distro_codename}-apps-security";
        "${distro_id}ESM:${distro_codename}-infra-security";
};
Unattended-Upgrade::Package-Blacklist {
};
Unattended-Upgrade::DevRelease "false";
```
查找`unattended-upgrade`的配置说明，在 Debian Wiki 上的[这个页面](https://wiki.debian.org/UnattendedUpgrades)只找到了一些简单介绍，但在`See Also`部分额外提到了`/usr/share/doc/unattended-upgrades/README.md.gz`，执行`zless /usr/share/doc/unattended-upgrades/README.md.gz`直接查看文件内容，发现这个`README.md`里对相关配置项的介绍还是比较详细的，进一步搜索后在 Github 上也找到了对应的仓库，地址是 [mvo5/unattended-upgrades](https://github.com/mvo5/unattended-upgrades) 。  

`README.md`中提到可以使用`apt-cache policy`查看相关 repository 的`o`and`a`两项，配置文件中`Unattended-Upgrade::Allowed-Origins`块就是一个`origin:archive`的列表，表示对应的 repository 启用了自动更新；`Unattended-Upgrade::Package-Blacklist`块是一个正则匹配的列表，表示要跳过更新的软件包；最后一个`Unattended-Upgrade::DevRelease`则表示在尚未发布正式 release 的 Ubuntu 上是否启用自动更新。  

关于`尚未发布正式 release 的 Ubuntu`，在`unattended-upgrades`的[源码](https://github.com/mvo5/unattended-upgrades/blob/master/unattended-upgrade#L2065)里可以看到依据的是`distro_info.UbuntuDistroInfo().devel(result="object")`，`distro_info`是一个 Python 软件包，从 Ubuntu 软件仓库中下载得到[源码](http://security.ubuntu.com/ubuntu/pool/universe/d/distro-info/python-distro-info_0.18ubuntu0.18.04.1_all.deb)后发现其依据的是`/usr/share/distro-info/ubuntu.csv`。最新版 Ubuntu 21.04 的 release date 是 2021-04-22，`Unattended-Upgrade::DevRelease`如果设为`auto`的话在 2021-04-01 才会开始自动更新。  

执行`cat /etc/lsb-release`可以看到当前`Ubuntu 18.04.5 LTS`对应的`distro_id`是`Ubuntu`，对应的`distro_codename`是`bionic`，所以`apt-cache policy`显示的所有 repository 中除了`bionic-backports`，`bionic-updates`和自己添加的第三方 repository 外，都在每天自动更新的范围之内，不同更新类型的说明在 Ubuntu Wiki 上的[这个页面](https://help.ubuntu.com/community/UbuntuUpdates)有部分介绍。  

回到 Mysql 这边，执行`apt policy mysql-server-5.7`得到：
```
mysql-server-5.7:
  Installed: 5.7.32-0ubuntu0.18.04.1
  Candidate: 5.7.32-0ubuntu0.18.04.1
  Version table:
 *** 5.7.32-0ubuntu0.18.04.1 500
        500 https://opentuna.cn/ubuntu bionic-updates/main amd64 Packages
        500 https://opentuna.cn/ubuntu bionic-security/main amd64 Packages
        100 /var/lib/dpkg/status
     5.7.21-1ubuntu1 500
        500 https://opentuna.cn/ubuntu bionic/main amd64 Packages
```
可以看到其最新版`5.7.32-0ubuntu0.18.04.1`是同时属于`bionic-updates`和`bionic-security`的，而`bionic-security`被包含于`unattended-upgrade`的`Allowed-Origins`，所以每天的自动更新任务会更新到该版本。`sudo apt update && apt list --upgradable`查看当前需手动升级的软件包，也就是没有随着自动更新被一起升级的软件包，有`docker-ce`和`nodejs`，都是自己添加的第三方 repository；还有`grub-common`，执行`apt policy grub-common`得到：
```
grub-common:
  Installed: 2.02-2ubuntu8.17
  Candidate: 2.02-2ubuntu8.20
  Version table:
     2.02-2ubuntu8.20 500
        500 https://opentuna.cn/ubuntu bionic-updates/main amd64 Packages
 *** 2.02-2ubuntu8.17 500
        500 https://opentuna.cn/ubuntu bionic-security/main amd64 Packages
        100 /var/lib/dpkg/status
     2.02-2ubuntu8 500
        500 https://opentuna.cn/ubuntu bionic/main amd64 Packages
```
所以自动更新只把它更新到了`bionic-security`的最新版本`2.02-2ubuntu8.17`，`bionic-updates`由于不在`Allowed-Origins`内所以不会被升级到对应版本。手动降级后再主动执行`unattended-upgrade`进行测试：
```bash
sudo apt install grub-common=2.02-2ubuntu8      # downgrad to 2.02-2ubuntu8
apt policy grub-common
sudo unattended-upgrade --dry-run               # show what unattended-upgrade will do
sudo unattended-upgrade                         # unattended-upgrade to 2.02-2ubuntu8.17
apt policy grub-common
sudo apt upgrade                                # upgrade to 2.02-2ubuntu8.20
apt policy grub-common
```
结果与预期相符，自动更新只会更新到`bionic-security`对应的`2.02-2ubuntu8.17`，手动更新才能更新到`bionic-updates`对应的`2.02-2ubuntu8.20`。  

手动降级`mysql-server-5.7`和`mysql-server-core-5.7`后再升回来，root用户的`authentication plugin`果然被重置回了`auth_socket`，搜索相关问题后在 Ubuntu 的 [Bug tracking](https://bugs.launchpad.net/ubuntu/+source/mysql-5.7/+bug/1571668) 找到了解答，关键性内容可以执行`zless /usr/share/doc/mysql-server-5.7/NEWS.Debian.gz`查看：
> mysql-5.7 (5.7.13-1~exp1) experimental; urgency=medium
> 
> Password behaviour when the MySQL root password is empty has changed. Packaging now enables socket authentication when the MySQL root password is empty. This means that a non-root user can't log in as the MySQL root user with an empty password. The new logic is as follows:  
> - The auth_socket plugin will be installed automatically only if it is to be activated for the root user.
> - The auth_socket plugin will be activated for the root user:
>   + If you had a database before with an empty root password.
>   + If you create a new database with an empty root password.
> - The auth_socket plugin will NOT be activated for the root user:
>   + If you had a database before with a root password set.
>   + If you create a new database with a root password set.
> - The auth_socket plugin will NOT be activated for any other user.
> - If you do not want the new behaviour, set the MySQL root password to be non-empty.

问题很清晰了，Ubuntu 每日自动更新升级`mysql-server`时，Mysql 检查到当前 root 用户的密码为空，就改回了`auth_socket`。

## 解决办法
最开始发现问题是 Ubuntu 的每日自动更新导致的后，想直接把测试服务器上的自动更新禁用掉，毕竟在服务器上每天自动 upgrade 总是不太安心。但查完`unattended-upgrades`相关资料后就觉得这个自动更新还是很有必要的，而且配置文件里对更新类型有所限制，并不是简单地`apt upgrade`，为了保障服务器的安全还是选择把它留了下来。  

那么问题该怎么解决呢？最安全的方法当然是给 root 用户指定非空密码，但那样就要去多个仓库里改涉及到的测试环境配置，为了方便我就选了另一条歪门邪道：首先创建`/etc/mysql/conf.d/mysql_empty_password.cnf`文件，在里面加上`init_file`项，然后指向一个 sql 文件，在 sql 文件中为 root 用户设置空密码。
```bash
sudo vim /etc/mysql/conf.d/mysql_empty_password.cnf
# [mysqld]
# init_file=/etc/mysql/conf.d/mysql_empty_password.sql

sudo vim /etc/mysql/conf.d/mysql_empty_password.sql
# ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '';
```
这样 Mysql 在每次启动时都会执行一遍`mysql_empty_password.sql`里的命令，升级后在下次启动时也会被改回空密码，仅用于测试服务器还是没什么问题的。

## To Be Continued...
对于服务器来说，Ubuntu 继承自 Debian 的`unattended-upgrades`还是十分有用的，配合包管理中区分出来的不同更新类型，可以实现自动升级安全补丁，关键更新等待人工干预。如果是 Arch Linux 的 pacman 的话，就没办法实现部分更新，自己的两台 Arch Linux 服务器大概两周左右会上去手动整体更新一遍，跟`unattended-upgrades`比还是低了一个档次。那么同样主要用于服务器的 CentOS 有没有类似的自动更新机制呢？和 Debian 比起来又孰强孰弱呢？
