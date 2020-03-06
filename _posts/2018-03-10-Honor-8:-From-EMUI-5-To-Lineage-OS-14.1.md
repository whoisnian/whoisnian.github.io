---
layout: post
title: 'Honor 8: From EMUI 5 To Lineage OS 14.1'
categories: Others
---

> 一直想在自己手机上体验一下原生安卓，很久之前就看上了 [Lineage OS](https://lineageos.org/)。  
> 然而手中的 Honor 8 并没有 Lineage OS 的[官方支持](https://wiki.lineageos.org/devices/)，在[XDA论坛](https://forum.xda-developers.com/)上找到的 [Unofficial 版本](https://forum.xda-developers.com/honor-8/development/rom-lineageos-14-1-honor-8-t3615506)据说是用于海外版本的 EMUI，国内的 Honor 8 由于某些底层问题而不能直接刷，所以就一直没动手。  
> 趁着刚开学的空闲时间比较多，就尝试着刷了一下 Lineage OS 14.1。  
> **注：华为对所有 2018 年 5 月 24 日后新上市产品停止解锁码服务，已经上市产品自 2018 年 5 月 24 日起 60 天后停止解锁码服务。**

<!-- more -->

### 手机型号
* Honor 8
* FRD-AL10
* FRD-AL10C00B396
* 4GB + 64GB
* EMUI 5.0.1
* Android 7.0
* Hisilicon Kirin 950

### 解锁 bootloader
Honor 8 解锁 bootloader 后才可以烧写第三方固件，先要去 EMUI 官网[解锁页面](https://emui.com/cn/plugin/unlock/index)申请解锁码。  
> 申请解锁需要满足以下条件：  
> 1.用户必须申请开通华为云帐号;  
> 2.用户必须在申请解锁的设备上登录华为云帐号并使用超过两周;  
> 3.每个华为云帐号半年内只能申请不超过5个设备解锁码;

然后使用 adb 工具进行解锁：  
* 在`设置 -> 关于手机`中连点版本号7次，开启开发者模式。
* 在`设置 -> 开发者选项`中打开 USB 调试，通过数据线将手机连接至电脑，`adb devices`可以查看已激活的设备，注意要在手机上同意 USB 调试。
* `adb reboot bootloader`重启进入 bootloader，应该可以看到绿色的`PHONE Locked`，此时手机尚未解锁。
* `fastboot oem unlock XXXXXXXXXXXXXXXX`，其中使用自己申请到的解锁码，然后在手机上音量键调整选项，电源键确认解锁，手机会自动重启并恢复出厂设置。
* 再次开启开发者模式，并同意 USB 调试，然后`adb reboot bootloader`重启进入 bootloader，此时应该可以看到红色的`PHONE Unlocked`，手机已经解锁。

### TWRP
* 华为自带的 Recovery 是不支持刷机的，这里需要刷入第三方的 TWRP。
* 刷入第三方 Recovery 需要在 bootloader 下进行，上一步`adb reboot bootloader`后手机正处于 bootloader 下，此时就可以刷入 TWRP。  
* 在 XDA 论坛上可以找到 [Honor 8 专用的 TWRP](https://forum.xda-developers.com/honor-8/development/twrp-t3566563)，下载 [twrp-3.1.1-1-frd.img](https://github.com/OpenKirin/android_device_honor_frd/releases/download/3.1.1-1/twrp-3.1.1-1-frd.img) 到电脑上。
* 进入 twrp-3.1.1-1-frd.img 所在文件夹，`fastboot flash recovery twrp-3.1.1-1-frd.img`刷入 TWRP，然后`fastboot reboot`重启。

### 海外版 EMUI
* XDA 论坛上已经有人提供了各个地区的 [EMUI 5 ROM 官网下载链接](https://forum.xda-developers.com/honor-8/how-to/to-emui5-custom-roms-tested-openkirin-t3638445)，直接下载即可。我下载的是 EU models 的 FRD-L19C432。
  > FRD-L19C432  
  > B360  
  > [http://update.hicloud.com:8180/TDS/d...ull/update.zip](http://update.hicloud.com:8180/TDS/data/files/p3/s15/G753/g104/v75675/f1/full/update.zip)  
  > [http://update.hicloud.com:8180/TDS/d...full_hw_eu.zip](http://update.hicloud.com:8180/TDS/data/files/p3/s15/G753/g104/v75675/f1/full/hw/eu/update_data_full_hw_eu.zip)  
* `adb reboot recovery`进入 TWRP，通过`adb push update.zip /sdcard/`和`adb push update_data_full_hw_eu.zip /sdcard/`两个命令将下载好的海外版 EMUI ROM 传到手机 SD 卡根目录下。
* 在 TWRP 的 Install 中选择`update.zip`刷入，然后再选择`update_data_full_hw_eu.zip`刷入。
* 两个 zip 都刷入之后重启手机，重启后在`设置 -> 关于手机`中可以看到 EMUI 版本由 5.0.1 变成了 5.0，版本号由 FRD-AL10C00B396 变成了 NRD90M test-keys，然后桌面上也多出了 Google 全家桶。根据版本号，此时还不是完整版本的海外 EMUI，但已经可以作为底包开始刷 Lineage OS 了。
* 此时手机的 Recovery 变回了原来华为的，进入 bootloader 会发现 bootloader 也被重新锁定，所以需要再次解锁 bootloader 和刷入 TWRP：
  * `adb reboot bootloader`
  * `fastboot oem unlock XXXXXXXXXXXXXXXX`
  * `adb reboot bootloader`
  * `fastboot flash recovery twrp-3.1.1-1-frd.img`
  * `fastboot reboot`  

**注：刷入`update.zip`过程结束后系统可能自动重启，若自动重启，需要在再次解锁 bootloader 和刷入 TWRP 后刷入`update_data_full_hw_eu.zip`。**

### Lineage OS
* 刚开始尝试直接刷入 [lineage-14.1-20170629](https://forum.xda-developers.com/honor-8/development/rom-lineageos-14-1-honor-8-t3615506)，结果开机后读不出 SIM 卡，检查可能是基带版本有问题。在论坛的[回复](https://forum.xda-developers.com/honor-8/development/rom-t3521731/post72130396#post72130396)里看到有说先刷低版本，再往上刷的，尝试之后成功了。
* `adb reboot recovery`进入 TWRP，选择 Wipe 中的 Advanced Wipe，清理`Dalvik/ART Cache`和`Cache`两个分区，清理完毕后返回，选择 Wipe 中的 Format Data，输入 yes 后格式化 Data 分区。若出现红色报错信息，则在 Reboot Recovery 后再次尝试 Format Data，格式化成功后再次 Reboot Recovery，准备开始刷入 Lineage OS。
* `adb push lineage-14.1-20170304-UNOFFICIAL-frd.zip /sdcard/`传入 lineage-14.1-20170304，在 Install 中选中，进行安装，安装完成后 Reboot 到 System。如果开机界面显示 Encryption unsuccessful，提示需要 reset phone，那么不要 reset phone，而是在连接 USB 的状态下同时长按关机键和音量+键，进入 Huawei eRecovery，这里会提示检测到 Data 分区损坏，选择 Format data partition，然后 Reboot，就能成功进入 Lineage OS 了。20170304 版本是单 SIM 卡，但此时应该可以正常使用 SIM 卡 1，接下来继续往上刷。
* `adb reboot recovery`进入 TWRP，不需要 Wipe，然后`adb push lineage-14.1-20170411-UNOFFICIAL-frd.zip /sdcard/`，在 Install 中刷入 lineage-14.1-20170411，完成后 Reboot 到 System，依旧单 SIM。
* `adb reboot recovery`进入 TWRP，不需要 Wipe，然后`adb push lineage-14.1-20170629-UNOFFICIAL-frd.zip /sdcard/`，在 Install 中刷入 lineage-14.1-20170629，完成后 Reboot 到 System，此时手机支持双 SIM，并且已经是当前的最新版本了。  
  ![lineageos](/public/image/lineageos.webp)  

### GApps
* 如果想要使用谷歌框架，就需要刷入 [Open GApps](http://opengapps.org/)。ARM64，Android 7.1，然后选择一个喜欢的 Variant，在 Wiki 上有不同 Variant 之间的[区别](https://github.com/opengapps/opengapps/wiki/Package-Comparison)。我选择的是 pico 版，只有最基础的内容。
* `adb reboot recovery`进入 TWRP，然后`adb push open_gapps-arm64-7.1-pico-20180311.zip /sdcard/`将下载好的 zip 包发送到手机，在 Install 中选择安装，完成后重启系统，就可以在 Play Store 中下载需要的应用了。

### Weather
* 尝试在桌面小部件的 cLock 里显示天气时，在天气来源中找不到天气提供商。Lineage OS 默认是没有内置的，但是在其官网的 [Extras 下载页](https://download.lineageos.org/extras)可以找到 Weather Providers 栏，提供了三个 apk，下载其中一个安装到手机，cLock 就可以在你设置完地区之后显示天气了。我选择的是雅虎天气，即 YahooWeatherProvider.apk。

### 无网络连接
* 开启数据连接或 WiFi 时，状态栏图标旁边会显示一个 x 表示无网络连接，但是实际上可以正常上网:  
  ![network_x](/public/image/network_x.webp)  
  这是由于原生安卓在开启数据连接或连接 WiFi 后，会访问 [www.google.com/generate_204](www.google.cn/generate_204) 根据 HTTP 响应码判断是否联网，同时判断是否需要登录，而由于“中国特色”，手机获取不到响应码，就会误以为是无网络连接，图标旁边显示一个 x，实际并不影响使用。  
* 如果想要去掉这个 x 的话，可以使用 adb 工具把 www.google.com/generate_204 修改为 www.google.cn/generate_204，图标旁就不会出现 x 了。  
  `adb shell "settings put global captive_portal_http_url http://www.google.cn/generate_204"`  
  `adb shell "settings put global captive_portal_https_url https://www.google.cn/generate_204"`  
  修改完的效果：  
  ![network_ok](/public/image/network_ok.webp)  

### Root  
* ~~如果想要获取 Root 权限，可以使用 [SuperSU](http://www.supersu.com/) 进行，在[下载页面](http://www.supersu.com/download)下载最新版 V2.82 到电脑，然后就是熟悉的过程了，`adb reboot recovery`进入 TWRP，`adb push SuperSU-v2.82-201705271822.zip /sdcard/`发送 zip 到手机，在 Install 里选择安装，重启系统。手机上应该已经多出了 SuperSU 的应用。~~
* 如果想要获取 Root 权限，可以使用 [Magisk](https://github.com/topjohnwu/Magisk) 进行，在[下载页面](https://github.com/topjohnwu/Magisk/releases)下载最新版到电脑，然后就是熟悉的过程了，`adb reboot recovery`进入 TWRP，`adb push Magisk-v17.1.zip /sdcard/`发送 zip 到手机，在 Install 里选择安装，重启系统。手机上开机后等待几分钟，应该已经多出了 Magisk Manager 的应用。

### 参考
* [\[ROM\]\[New\]\[Unofficial\] OpenKirin's LineageOS 14.1 for Honor 8](https://forum.xda-developers.com/honor-8/development/rom-lineageos-14-1-honor-8-t3615506)  
* [\[ROM\]\[7.1.x\]\[UNOFFICIALY\]\[LineageOS 14.1 For HONOR 8 \]\[Update 11/04/2017\]](https://forum.xda-developers.com/honor-8/development/rom-t3521731)  
* [\[GUIDE\] Back to EMUI5 from custom roms](https://forum.xda-developers.com/honor-8/how-to/to-emui5-custom-roms-tested-openkirin-t3638445)  
* [How to Install Magisk](https://www.xda-developers.com/how-to-install-magisk/)  
* [【ROM】荣耀8LineageOS14稳定版来袭～～](http://tieba.baidu.com/p/5297266072)  
* [【ROM】荣耀8国外版固件下载合集～](http://tieba.baidu.com/p/5264414806)  
* [关于 V2EX 提供的 Android Captive Portal Server 地址的更新](https://www.v2ex.com/t/303889)  
