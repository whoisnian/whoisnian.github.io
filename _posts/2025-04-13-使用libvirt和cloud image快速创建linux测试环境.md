---
layout: post
title: 使用libvirt和cloud image快速创建linux测试环境
categories: server
last_modified_at: 2025-04-22T03:37:55+08:00
---

> 前段时间新开了 [static-binaries](https://github.com/whoisnian/static-binaries) 仓库基于 [Alpine Linux aports repository](https://gitlab.alpinelinux.org/alpine/aports) 来静态编译日常开发调试过程中常用的命令行工具，计划在各个发行版的常见版本上测试编译结果能否正常运行，于是使用 libvirt 从各发行版官方提供的 cloud image 快速创建虚拟机作为测试环境。  
> 使用 Debian 和 CentOS 的 cloud image 创建虚拟机的过程中遇到了多个问题，例如创建 debian12-amd64 虚拟机时的 cloud-init 参数不生效，创建 debian12-arm64 虚拟机时报错 `/usr/share/libvirt/cpu_map/arm_Ampere-1.xml` 文件不存在，还有创建的 centos7-arm64 虚拟机在启动时报错 `Synchronous Exception`，尝试寻找解决方案。  

<!-- more -->

## 关于 cloud image
最初折腾虚拟化时尝试了 [OpenStack](https://www.openstack.org) 和 [Proxmox VE](https://pve.proxmox.com)，在 [OpenStack Virtual Machine Image Guide](https://docs.openstack.org/image-guide/obtain-images.html) 文档的相关章节发现了一些常见发行版的 cloud image 列表，实际体验上从 cloud image 创建虚拟机确实方便快捷。  
但由于是单台家用主机，OpenStack 一堆组件带来的维护复杂度和日常开关机偶发的启动问题无法忽视，因此将主机重置为了 Proxmox VE。结果先是嫌弃 Proxmox VE 没有使用 cloud image 创建虚拟机的现成功能，需要自行构建 VM Template 才能方便地多次复用，后又嫌弃 Proxmox VE 对底层 Debian 系统的大幅定制，不方便将其当作常规系统进一步折腾，于是最终将主机重置为了 Arch Linux，使用 [libvirt](https://wiki.archlinux.org/title/Libvirt) 搭配 [virt-manager](https://virt-manager.org) 对虚拟机进行管理，通过 Shell 脚本下载并初始化各发行版的 cloud image。  

根据 [Debian Official Cloud Images](https://cloud.debian.org/images/cloud/) 页面上的解释说明，文件名中指定了 `azure` 和 `ec2` 的镜像包含对云服务商的定制优化，指定 `generic` 的镜像适用于支持 cloud-init 的虚拟机和裸机环境，而 `genericcloud` 是在 `generic` 的基础上去掉了物理硬件驱动，最后的 `nocloud` 则是测试用途，不包含 cloud-init 且默认使用 root 无密码登录。其他发行版的 cloud image 也都有类似区分，但细节上会稍有不同，对于 libvirt 场景选择用于虚拟机且支持 cloud-init 的镜像即可，例如：  
* Debian 12: `azure` `ec2` `generic` **`[genericcloud]`** `nocloud`
* CentOS 9-stream: `Container-Base` `Container-Minimal` **`[GenericCloud]`** `Vagrant` `ec2`
* Ubuntu 24.04 LTS: `azure` `lxd` `ova` `vmdk` **`[img]`**
* Alpine Linux 3.21: `aws` `azure` `gcp` `oci` `nocloud` **`[generic]`**
* Arch Linux: `basic` **`[cloudimg]`**

## 正常创建 centos7-amd64 虚拟机
```sh
# 准备镜像
wget https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud-2211.qcow2
sudo cp ./CentOS-7-x86_64-GenericCloud-2211.qcow2 /var/lib/libvirt/images/centos7-amd64.qcow2
sudo qemu-img resize /var/lib/libvirt/images/centos7-amd64.qcow2 20G

# 创建虚拟机
virt-install --connect qemu:///system \
  --name centos7-amd64 \
  --osinfo centos7.0 \
  --arch x86_64 \
  --vcpus 1 \
  --memory 1024 \
  --graphics none \
  --noautoconsole \
  --import \
  --disk /var/lib/libvirt/images/centos7-amd64.qcow2 \
  --cloud-init "root-password-generate=on,disable=on,root-ssh-key=$HOME/.ssh/id_rsa.pub"

# 访问方式1：通过串口连接到文本控制台，使用 root 用户和 virt-install 命令打印的初始密码登录
virsh --connect qemu:///system console --domain centos7-amd64

# 访问方式2：获取虚拟机 IP，使用 SSH 密钥登录
virsh --connect qemu:///system domifaddr --domain centos7-amd64
ssh root@192.168.122.75 -i ~/.ssh/id_rsa
```

## 异常创建 debian12-amd64 虚拟机
```sh
# 准备镜像
wget https://cdimage.debian.org/images/cloud/bookworm/20250416-2084/debian-12-genericcloud-amd64-20250416-2084.qcow2
sudo cp ./debian-12-genericcloud-amd64-20250416-2084.qcow2 /var/lib/libvirt/images/debian12-amd64.qcow2
sudo qemu-img resize /var/lib/libvirt/images/debian12-amd64.qcow2 20G

# 创建虚拟机
virt-install --connect qemu:///system \
  --name debian12-amd64 \
  --osinfo debian12 \
  --arch x86_64 \
  --vcpus 1 \
  --memory 1024 \
  --graphics none \
  --noautoconsole \
  --import \
  --disk /var/lib/libvirt/images/debian12-amd64.qcow2 \
  --cloud-init "root-password-generate=on,disable=on,root-ssh-key=$HOME/.ssh/id_rsa.pub"
```
执行以上命令，可以看到创建出的 debian12-amd64 虚拟机处于 running 状态，但通过串口连接到文本控制台后使用 root 用户登录却提示 `Login incorrect`，尝试使用 SSH 密钥登录也会提示 `Permission denied (publickey)`，感觉就像是 cloud-init 参数完全不生效。  
参考 [cloud-init 官方文档](https://github.com/canonical/cloud-init/blob/8a1d73498c505049cd281a1661c2750b93fb4a6f/doc/rtd/howto/launch_qemu.rst) 及 [virt-install 源码实现](https://github.com/virt-manager/virt-manager/blob/d17731aea1fd8f7fa926253cc40a1c777264da07/virtinst/install/installerinject.py#L52) 可知，其原理是先将 cloud-init 参数按指定格式写入文本文件，再生成 iso 并在虚拟机首次启动时挂载为 cdrom，手动复现该逻辑：  
```sh
# 准备镜像
wget https://cdimage.debian.org/images/cloud/bookworm/20250416-2084/debian-12-genericcloud-amd64-20250416-2084.qcow2
sudo cp ./debian-12-genericcloud-amd64-20250416-2084.qcow2 /var/lib/libvirt/images/debian12-amd64.qcow2
sudo qemu-img resize /var/lib/libvirt/images/debian12-amd64.qcow2 20G

# 准备 cloud-init 配置
cat >meta-data <<EOF
instance-id: debian12-amd64
local-hostname: debian12-amd64
EOF

cat >user-data <<EOF
#cloud-config
chpasswd:
  list: |
    root:u7QofZic0KrPhLcB
  expire: True
users:
  - default
  - name: root
    ssh_authorized_keys:
      - $(cat ~/.ssh/id_rsa.pub)
runcmd:
- echo "Disabled by virt-install" > /etc/cloud/cloud-init.disabled
EOF

# 生成 seed.iso
sudo xorrisofs \
  -output /var/lib/libvirt/boot/debian12-amd64-seed.iso \
  -volid cidata -rational-rock -joliet -input-charset utf8 \
  user-data meta-data

# 创建虚拟机
virt-install --connect qemu:///system \
  --name debian12-amd64 \
  --osinfo debian12 \
  --arch x86_64 \
  --vcpus 1 \
  --memory 1024 \
  --graphics none \
  --noautoconsole \
  --import \
  --disk /var/lib/libvirt/images/debian12-amd64.qcow2 \
  --cdrom /var/lib/libvirt/boot/debian12-amd64-seed.iso
```
执行以上命令创建出的 debian12-amd64 虚拟机依旧无法登录，多次调整尝试后发现问题来自 cdrom 的硬件配置，默认的 `--cloud-init` 或 `--cdrom` 参数生成的 XML 定义基本相同：  
```xml
<disk type="file" device="cdrom">
  <driver name="qemu" type="raw"/>
  <source file="/var/lib/libvirt/boot/debian12-amd64-seed.iso" index="1"/>
  <backingStore/>
  <target dev="sda" bus="sata"/>
  <readonly/>
  <alias name="sata0-0-0"/>
  <address type="drive" controller="0" bus="0" target="0" unit="0"/>
</disk>
```
其中的 `bus="sata"` 会导致 cloud-init 不生效，将创建虚拟机时的 `--cdrom` 参数替换为 `--disk /var/lib/libvirt/boot/debian12-amd64-seed.iso,device=cdrom,bus=scsi,readonly=on` 重新创建，就可以得到 cloud-init 正常生效的 debian12-amd64 虚拟机，通过串口使用密码登录和 SSH 密钥登录均正常，此时该 cdrom 对应的 XML 定义为：  
```xml
<disk type="file" device="cdrom">
  <driver name="qemu" type="raw"/>
  <source file="/var/lib/libvirt/boot/debian12-amd64-seed.iso" index="1"/>
  <backingStore/>
  <target dev="sda" bus="scsi"/>
  <readonly/>
  <alias name="scsi0-0-0-0"/>
  <address type="drive" controller="0" bus="0" target="0" unit="0"/>
</disk>
```

尝试在 Google 上搜索 `debian virt-install cloud-init`，也可以找到类似的问题讨论，其中 [mop.koeln](https://mop.koeln) 上的 [一篇博客](https://mop.koeln/blog/creating-a-local-debian-vm-using-cloud-init-and-libvirt/) 在结尾处明确提到：  
> *genericcloud: Similar to generic. Should run in any virtualised environment. Is smaller than `generic` by excluding drivers for physical hardware.*  
>
> Obviously this sounds like the right image. Unfortunately it is not. It doesn't contain the SATA AHCI drivers which are needed because of the way the cloud-init stuff is being injected into the VM (as a cdrom drive).  

因此最终的解决方案就是换用 `generic` 镜像，或者手动为 cloud-init 准备 seed.iso，并且在挂载到虚拟机时选择非 SATA 的磁盘总线类型（Disk bus type）。

## 异常创建 debian12-arm64 虚拟机
```sh
# 准备镜像
wget https://cdimage.debian.org/images/cloud/bookworm/20250416-2084/debian-12-generic-arm64-20250416-2084.qcow2
sudo cp ./debian-12-generic-arm64-20250416-2084.qcow2 /var/lib/libvirt/images/debian12-arm64.qcow2
sudo qemu-img resize /var/lib/libvirt/images/debian12-arm64.qcow2 20G

# 创建虚拟机
virt-install --connect qemu:///system \
  --name debian12-arm64 \
  --osinfo debian12 \
  --arch aarch64 \
  --vcpus 1 \
  --memory 1024 \
  --graphics none \
  --noautoconsole \
  --import \
  --disk /var/lib/libvirt/images/debian12-arm64.qcow2 \
  --cloud-init "root-password-generate=on,disable=on,root-ssh-key=$HOME/.ssh/id_rsa.pub"
```
执行以上命令，`virt-install` 出现报错 `ERROR Failed to open file '/usr/share/libvirt/cpu_map/arm_Ampere-1.xml': No such file or directory`，添加 `--debug` 参数后重新执行可以得到更加详细的堆栈信息：  
```log
[Sun, 20 Apr 2025 23:31:47 virt-install 43286] ERROR (cli:257) Failed to open file '/usr/share/libvirt/cpu_map/arm_Ampere-1.xml': No such file or directory
[Sun, 20 Apr 2025 23:31:47 virt-install 43286] DEBUG (cli:259) 
Traceback (most recent call last):
  File "/usr/share/virt-manager/virtinst/virtinstall.py", line 966, in start_install
    domain = installer.start_install(
            guest, meter=meter,
            doboot=not options.noreboot,
            transient=options.transient)
  File "/usr/share/virt-manager/virtinst/install/installer.py", line 726, in start_install
    domain = self._create_guest(
            guest, meter, initial_xml, final_xml,
            doboot, transient)
  File "/usr/share/virt-manager/virtinst/install/installer.py", line 667, in _create_guest
    domain = self.conn.createXML(initial_xml or final_xml, 0)
  File "/usr/lib/python3.13/site-packages/libvirt.py", line 4545, in createXML
    raise libvirtError('virDomainCreateXML() failed')
libvirt.libvirtError: Failed to open file '/usr/share/libvirt/cpu_map/arm_Ampere-1.xml': No such file or directory
[Sun, 20 Apr 2025 23:31:47 virt-install 43286] DEBUG (cli:272) Domain installation does not appear to have been successful.
```
从堆栈信息中可以看到错误来源并不是 `virt-install` 自身代码，而是在调用 libvirt API 时出现的，执行 `pacman -Fl libvirt | grep usr/share/libvirt/cpu_map` 确认同文件夹下其他文件均来自 libvirt 包，且当前版本 `libvirt-1:11.2.0-1` 内确实不包含 `arm_Ampere-1.xml`。  
先在 libvirt 源码仓库内查找 v11.2.0 [相关代码](https://gitlab.com/libvirt/libvirt/-/tree/v11.2.0/src/cpu_map)，发现存在 `arm_Ampere-{1,1a}.xml` 两个文件；于是又去查看 Arch Linux 官方打包脚本 [PKGBUILD](https://gitlab.archlinux.org/archlinux/packaging/packages/libvirt/-/blob/1-11.2.0-1/PKGBUILD)，未发现明显问题；最后下载打包脚本中实际使用的 [libvirt-11.2.0.tar.xz](https://libvirt.org/sources/libvirt-11.2.0.tar.xz) 再次确认，发现同文件夹下其他文件均被 `src/cpu_map/meson.build` 引用，但引用中缺少了 `arm_Ampere-{1,1a}.xml` 两个文件。  
再次回到 libvirt 源码仓库查看 `src/cpu_map/meson.build` 文件历史记录，可确认该问题被 20250404 的 [commit:701b2c0f](https://gitlab.com/libvirt/libvirt/-/commit/701b2c0fcac8b911c4d8b916ee07eddc098e8772) 修复，且在 [issue:762](https://gitlab.com/libvirt/libvirt/-/issues/762) 中仓库 Owner 回复说通常在每月的一号发布新版本，很少会在其他时间发布 bug fix 版本。因此新版本发布之前只能临时手动修复：  
```sh
sudo wget -O /usr/share/libvirt/cpu_map/arm_Ampere-1.xml https://gitlab.com/libvirt/libvirt/-/raw/v11.2.0/src/cpu_map/arm_Ampere-1.xml
sudo wget -O /usr/share/libvirt/cpu_map/arm_Ampere-1a.xml https://gitlab.com/libvirt/libvirt/-/raw/v11.2.0/src/cpu_map/arm_Ampere-1a.xml
sudo systemctl restart libvirtd
```
重新执行 `virt-install` 命令创建虚拟机，由于是跨架构的 CPU 虚拟化所以启动较慢，耗时约 80 秒，之后通过串口使用密码登录和 SSH 密钥登录均正常。

## 异常创建 centos7-arm64 虚拟机
```sh
# 准备镜像
wget https://cloud.centos.org/centos/7/images/CentOS-7-aarch64-GenericCloud-2211.qcow2
sudo cp ./CentOS-7-aarch64-GenericCloud-2211.qcow2 /var/lib/libvirt/images/centos7-arm64.qcow2
sudo qemu-img resize /var/lib/libvirt/images/centos7-arm64.qcow2 20G

# 创建虚拟机
virt-install --connect qemu:///system \
  --name centos7-arm64 \
  --osinfo centos7.0 \
  --arch aarch64 \
  --vcpus 1 \
  --memory 1024 \
  --graphics none \
  --noautoconsole \
  --import \
  --disk /var/lib/libvirt/images/centos7-arm64.qcow2 \
  --cloud-init "root-password-generate=on,disable=on,root-ssh-key=$HOME/.ssh/id_rsa.pub"
```
执行以上命令，可以看到创建出的 centos7-arm64 虚拟机处于 running 状态，但通过串口连接到文本控制台后无任何响应，也无法使用 SSH 密钥登录，感觉像是系统未成功启动。  
等待几分钟后问题依旧，执行 `virsh --connect qemu:///system shutdown centos7-arm64` 尝试关闭虚拟机，无响应后再执行 `virsh --connect qemu:///system destroy centos7-arm64` 强制关闭，随后执行 `virsh --connect qemu:///system start centos7-arm64 --console` 启动虚拟机并附加到文本控制台，可以看到 UEFI firmware 版本提示一闪而过，随后是以下报错：  
```log
BdsDxe: loading Boot0003 "Red Hat Enterprise Linux" from HD(1,GPT,5D4156FB-1017-43DC-99EF-89413986D707,0x800,0x64000)/\EFI\centos\shimaa64.efi
BdsDxe: starting Boot0003 "Red Hat Enterprise Linux" from HD(1,GPT,5D4156FB-1017-43DC-99EF-89413986D707,0x800,0x64000)/\EFI\centos\shimaa64.efi

Synchronous Exception at 0x000000007DE2E698

Synchronous Exception at 0x000000007DE2E698
```
在 Google 上搜索 `Synchronous Exception at` 可以找到类似问题的反馈，基本围绕 `qemu aarch64` 展开，例如 [UTM issue:6427](https://github.com/utmapp/UTM/issues/6427) 中有用户在 CentOS Stream 9 升级后遇到，后续讨论中提到的临时解决方案是从 QEMU 迁移到 Apple Virtualization，并不能作为 linux 环境下的参考。  
重新使用 `qemu aarch64 Synchronous Exception at` 作为关键词进行搜索，很容易搜到 [lima issue:1645](https://github.com/lima-vm/lima/issues/1645) 和 [qemu issue:1990](https://gitlab.com/qemu-project/qemu/-/issues/1990)，基本围绕 `UEFI firmware` 及 `EDK2` 展开，并且有尝试降级到旧版 edk2-aarch64 临时解决的，执行 `pacman -Si qemu-system-aarch64` 确认 Arch Linux 使用的也是 edk2-aarch64，于是尝试降级。  

在 [Arch Linux Archive](https://wiki.archlinux.org/title/Arch_Linux_Archive) 上的 packages [子目录](https://archive.archlinux.org/packages/e/edk2-aarch64/) 中找到的 edk2-aarch64 最早版本是 [202211-1](https://archive.archlinux.org/packages/e/edk2-aarch64/edk2-aarch64-202211-1-any.pkg.tar.zst)，手动下载安装包并解压：  
```sh
wget https://archive.archlinux.org/packages/e/edk2-aarch64/edk2-aarch64-202211-1-any.pkg.tar.zst
mkdir -p /tmp/edk2-aarch64-202211-1
tar -xvf ./edk2-aarch64-202211-1-any.pkg.tar.zst -C /tmp/edk2-aarch64-202211-1 --strip-components=4 usr/share/edk2/aarch64/
```
然后为 `virt-install` 命令补充 `--boot loader=/tmp/edk2-aarch64-202211-1/QEMU_CODE.fd,loader.readonly=yes,loader.type=pflash,nvram.template=/tmp/edk2-aarch64-202211-1/QEMU_VARS.fd,loader_secure=no` 参数后重新创建虚拟机，启动过程中的报错信息无明显变化。  

查看 edk2-aarch64 软件包的 Arch Linux 官方打包脚本 [PKGBUILD 提交历史](https://gitlab.archlinux.org/archlinux/packaging/packages/edk2/-/commits/202411-1/PKGBUILD)，发现从 202208-3 升级到 202211-1 时将原先的单个 edk2-armvirt 拆分为了 edk2-arm 和 edk2-aarch64，因此再到 Arch Linux Archive 上查找 edk2-armvirt 的 [历史版本](https://archive.archlinux.org/packages/e/edk2-armvirt/)，选择最后一个 [202208-3](https://archive.archlinux.org/packages/e/edk2-armvirt/edk2-armvirt-202208-3-any.pkg.tar.zst)，手动下载安装包并解压：  
```sh
wget https://archive.archlinux.org/packages/e/edk2-armvirt/edk2-armvirt-202208-3-any.pkg.tar.zst
mkdir -p /tmp/edk2-armvirt-202208-3
tar -xvf ./edk2-armvirt-202208-3-any.pkg.tar.zst -C /tmp/edk2-armvirt-202208-3 --strip-components=4 usr/share/edk2-armvirt/aarch64/
```
然后为 `virt-install` 命令补充 `--boot loader=/tmp/edk2-armvirt-202208-3/QEMU_CODE.fd,loader.readonly=yes,loader.type=pflash,nvram.template=/tmp/edk2-armvirt-202208-3/QEMU_VARS.fd,loader_secure=no` 参数后重新创建虚拟机，耗时约 90 秒后启动成功，之后再通过串口使用密码登录和 SSH 密钥登录均正常。  

因此临时解决方案就是手动下载旧版的 [edk2-armvirt-202208-3](https://archive.archlinux.org/packages/e/edk2-armvirt/edk2-armvirt-202208-3-any.pkg.tar.zst)，并在 `virt-install` 命令中添加 `--boot` 参数使用该旧版 UEFI firmware 启动。  
