---
layout: post
title: 在 chroot jail 中运行Golang程序
categories: programming
last_modified_at: 2023-10-25T01:22:22+08:00
---

> 使用 chroot 命令可以改变进程的可见根目录，创建出一个 chroot jail，限制对应进程可访问到的文件，降低程序的部分安全风险。  
> 之前编写过一个文件分享 Web 应用 [share-Go](https://github.com/whoisnian/share-Go)，在仓库的 README.md 中给出了 `run with linux chroot` 的简单示例，以避免非预期的文件访问。  
> 偶然发现该 chroot 示例中存在几个隐藏问题，主要涉及到 Golang 程序的外部环境依赖，就顺便整理记录下来。  

<!-- more -->

## chroot 简单介绍
根据[维基百科](https://en.wikipedia.org/wiki/Chroot)上的相关说明，早在 Linux 发布之前就有了 chroot 的概念，用来实现简单的目录隔离。为了更完善的资源隔离，BSD 将 chroot 扩展成了 [jail](https://en.wikipedia.org/wiki/FreeBSD_jail) 命令，而 Linux 则是引入了 [namespaces](https://en.wikipedia.org/wiki/Linux_namespaces) 特性，后续的 LXC，Docker 等软件都是 Linux namespaces 的实际应用。

自己接触过的 chroot 实际用途有：
* Arch Linux 在系统安装过程中会使用 chroot 从 Live CD 切换到目标磁盘进行系统配置[^1]
* Arch User Repository 推荐使用 chroot 环境调试构建流程，并在官方仓库中提供了辅助脚本[^2]
* DOMjudge 评测机需要预配置 chroot 环境，在该环境下编译和执行选手提交的代码[^3]
* HAProxy 提供了 chroot 配置项限制 worker 进程的运行目录，来防御零日攻击[^4]

## Golang 程序示例
```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os/user"
	"time"
	// _ "time/tzdata"
)

func pingHandler(w http.ResponseWriter, _ *http.Request) {
	w.Write([]byte("pong\n"))
}

func wgetHandler(w http.ResponseWriter, r *http.Request) {
	resp, err := http.Get(r.FormValue("url"))
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()
	io.Copy(w, resp.Body) // ignore error
}

func timeHandler(w http.ResponseWriter, _ *http.Request) {
	loc, err := time.LoadLocation("Asia/Shanghai")
	if err != nil {
		panic(err)
	}
	s := time.Now().In(loc).Format(time.RFC3339)
	w.Write([]byte(s))
	w.Write([]byte{'\n'})
}

func userHandler(w http.ResponseWriter, _ *http.Request) {
	u, err := user.Lookup("nobody")
	if err != nil {
		panic(err)
	}
	fmt.Fprintf(w, "user: %s(%s)\n", u.Username, u.Uid)
}

func main() {
	http.HandleFunc("/ping", pingHandler)
	http.HandleFunc("/wget", wgetHandler)
	http.HandleFunc("/time", timeHandler)
	http.HandleFunc("/user", userHandler)

	fmt.Println("Service started at 127.0.0.1:8080")
	if err := http.ListenAndServe("127.0.0.1:8080", nil); err != nil {
		panic(err)
	}
}
```
## Golang 外部依赖
### 动态链接
直接使用 `go build main.go` 编译得到可执行文件 `main`，移动到单独的 `new_root` 目录下使用 `sudo chroot ./new_root /main` 运行，出现报错：  
```sh
# 目录结构示例：
# .
# ├── main.go
# └── new_root
#     └── main

sudo chroot ./new_root /main
# chroot: failed to run command ‘/main’: No such file or directory
```
报错信息 `No such file or directory` 很容易被误以为是命令中的文件路径 `/main` 有问题，但在反复测试之后排除了路径问题，开始在 Google 上搜索相关信息。  
根据 stackoverflow 上的[这个回答](https://stackoverflow.com/a/70057902)，可以直接去翻 coreutils 仓库里 chroot 命令的源码，其中错误处理部分的主要代码为：  
```c
/* Execute the given command.  */
execvp (argv[0], argv);

int exit_status = errno == ENOENT ? EXIT_ENOENT : EXIT_CANNOT_INVOKE;
error (0, errno, _("failed to run command %s"), quote (argv[0]));
return exit_status;
```
其中错误码 `ENOENT` 对应的含义就是 `No such file or directory`，根据代码推测这里的 `errno` 来自于前面的 `execvp`，再使用 `sudo strace chroot ./new_root /main` 可以确认 `ENOENT` 的来源是系统调用 `execve`，因此执行 `man 2 execve` 查找系统调用具体说明：
> If the executable is a dynamically linked ELF executable, the interpreter named in the PT_INTERP segment is used to load the needed shared objects.  
> This interpreter is typically /lib/ld-linux.so.2 for binaries linked with glibc (see ld-linux.so(8)).
>
> ENOENT The file pathname or a script or ELF interpreter does not exist.

因此编译得到的可执行文件 `main` 属于动态链接，运行时 chroot 环境中需要有对应的 `ELF interpreter`。根据 `readelf -l ./new_root/main` 命令确认使用的是 `/lib64/ld-linux-x86-64.so.2`，手动复制到 `new_root` 目录后重新运行：  
```sh
# 复制后的目录结构：
# .
# ├── main.go
# └── new_root
#     ├── lib64
#     │   └── ld-linux-x86-64.so.2
#     └── main

sudo chroot ./new_root /main
# /main: error while loading shared libraries: libresolv.so.2: cannot open shared object file: No such file or directory
```
此时的报错是说缺少共享库，可以先使用 `ldd` 命令列出所有需要的共享库，手动复制到对应位置后再次运行：  
```sh
ldd ./new_root/main
#   linux-vdso.so.1 (0x00007ffee589f000)
#   libresolv.so.2 => /usr/lib/libresolv.so.2 (0x00007fdb329aa000)
#   libc.so.6 => /usr/lib/libc.so.6 (0x00007fdb327c8000)
#   /lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007fdb329e2000)

# 模仿 Arch Linux 的目录结构：
# .
# ├── main.go
# └── new_root
#     ├── lib -> usr/lib
#     ├── lib64 -> usr/lib
#     ├── main
#     └── usr
#         ├── lib
#         │   ├── ld-linux-x86-64.so.2
#         │   ├── libc.so.6
#         │   └── libresolv.so.2
#         └── lib64 -> lib

sudo chroot ./new_root /main
# Service started at 127.0.0.1:8080

curl 127.0.0.1:8080/ping
# pong
```
因此手动复制共享库确实可以解决这类依赖问题，除此之外也可以换个思路：既然是动态链接引入的共享库依赖，那么不使用动态链接就好了，改为静态链接。

在 Golang Doc 的 [FAQ](https://go.dev/doc/faq#Why_is_my_trivial_program_such_a_large_binary) 中明确提到 `The linker in the gc toolchain creates statically-linked binaries by default`，这里的 `gc` 指的是 `go compiler`，即 Golang 在默认情况下是静态链接的。但涉及到 [cgo](https://pkg.go.dev/cmd/cgo) 时就属于默认情况之外了，例如标准库中的 net，os/user 和 runtime/cgo 包，具体细节在 [cmd/cgo/doc.go](https://cs.opensource.google/go/go/+/refs/tags/go1.21.3:src/cmd/cgo/doc.go;l=538) 的 `Implementation details` 部分，可以整理得到：
* 涉及 cgo 的情况下，cmd/link 分为 internal 和 external 模式，可以使用 `-linkmode` 参数手动指定：
  * internal：代码仅涉及标准库中的已知 cgo 包时，cmd/link 自行按动态链接处理，不依赖外部链接器
  * external：cmd/link 仅收集所需文件，主要工作交由外部链接器如 gcc 或 clang 处理，可以通过 `-extldflags` 指定参数
* 执行 `go build -ldflags='-linkmode "external" -extldflags "-static"' main.go` 命令指定外部链接器使用静态链接，可以得到不依赖共享库的二进制文件
* 执行 `CGO_ENABLED=0 go build main.go` 命令禁用 cgo 之后再编译，就属于默认走静态链接的情况，也可以得到不依赖共享库的二进制文件

纯 Go 代码可以很方便地进行交叉编译，但涉及到 cgo 就需要额外配置 C/C++ 的交叉编译工具链了，因此通常使用 `CGO_ENABLED=0` 禁用 cgo 来进行编译：  
```sh
CGO_ENABLED=0 go build main.go

ldd ./main
#   not a dynamic executable

# 目录结构示例：
# .
# ├── main.go
# └── new_root
#     └── main

sudo chroot ./new_root /main
# Service started at 127.0.0.1:8080

curl 127.0.0.1:8080/ping
# pong
```

### DNS 解析
执行 `sudo chroot ./new_root /main` 命令运行静态链接得到的二进制文件，在收到 `/wget?url=http://baidu.com` 请求时会出现 panic：  
`Get "http://baidu.com": dial tcp: lookup baidu.com on [::1]:53: read udp [::1]:53633->[::1]:53: read: connection refused`

根据 Golang net 标准库中 [Name Resolution](https://pkg.go.dev/net#hdr-Name_Resolution) 部分的说明，在禁用 cgo 后会使用内置的纯 Go 解析器，该解析器从 `/etc/resolv.conf` 读取服务器列表，然后直接向服务器发送 DNS 查询请求。由于当前的 chroot 环境中并不存在 `/etc/resolv.conf` 文件，所以默认会向 `[::1]:53` 发送查询请求，连接失败时报错。

在 `new_root` 目录下手动创建 `etc/resolv.conf` 文件，指定 `nameserver 223.5.5.5`，再次测试相同请求：  
```sh
# 目录结构示例：
# .
# ├── main.go
# └── new_root
#     ├── etc
#     │   └── resolv.conf
#     └── main

cat ./new_root/etc/resolv.conf
# nameserver 223.5.5.5

sudo chroot ./new_root /main
# Service started at 127.0.0.1:8080

curl "127.0.0.1:8080/wget?url=http://baidu.com"
# <html>
# <meta http-equiv="refresh" content="0;url=http://www.baidu.com/">
# </html>
```

### SSL 根证书
执行 `sudo chroot ./new_root /main` 命令运行静态链接得到的二进制文件，在收到 `/wget?url=https://ip.whoisnian.com` 请求时会出现 panic：  
`Get "https://ip.whoisnian.com": tls: failed to verify certificate: x509: certificate signed by unknown authority`

根据 Golang crypto/x509 标准库中 [SystemCertPool](https://pkg.go.dev/crypto/x509#SystemCertPool) 的具体实现，[loadSystemRoots](https://cs.opensource.google/go/go/+/refs/tags/go1.21.3:src/crypto/x509/root_unix.go;l=32) 会从 [crypto/x509/root_linux.go](https://cs.opensource.google/go/go/+/refs/tags/go1.21.3:src/crypto/x509/root_linux.go) 中定义的文件和目录下加载根证书。再结合 ArchWiki 上 [Transport Layer Security](https://wiki.archlinux.org/title/Transport_Layer_Security#Certificate_authorities) 的介绍，本地 Arch Linux 的根证书来源为：
* ca-certificates-mozilla[^5] 提供了来自 [Mozilla CA Certificate Store](https://wiki.mozilla.org/CA) 的单个 mozilla.trust.p11-kit 作为初始信任源
* p11-kit[^6] 提供了 `trust` 命令，支持从已配置的信任源中以指定格式提取证书
* ca-certificates-utils[^7] 提供了 `update-ca-trust` 脚本，可以调用 `trust` 命令生成 `/etc/ca-certificates/extracted` 和 `/etc/ssl/certs`，并配置了自动触发 hook

其中 `/etc/ssl/certs/ca-certificates.crt` 软链接指向 `/etc/ca-certificates/extracted/tls-ca-bundle.pem`，是所有 `server-auth` 用途的证书合集，因此直接使用该合集即可。

手动复制 `/etc/ssl/certs/ca-certificates.crt` 到 `new_root` 目录下，再次测试相同请求：  
```sh
# 目录结构示例：
# .
# ├── main.go
# └── new_root
#     ├── etc
#     │   ├── resolv.conf
#     │   └── ssl
#     │       └── certs
#     │           └── ca-certificates.crt
#     └── main

sudo chroot ./new_root /main
# Service started at 127.0.0.1:8080

curl "127.0.0.1:8080/wget?url=https://ip.whoisnian.com"
# 110.60.10.127
```

### tzdata
执行 `sudo chroot ./new_root /main` 命令运行静态链接得到的二进制文件，在收到 `/time` 请求时会出现 panic：`unknown time zone Asia/Shanghai`

根据 Golang time 标准库中 [LoadLocation](https://pkg.go.dev/time#LoadLocation) 的说明，该函数需要加载系统中的 tzdata 文件，或者加载 Golang 安装时附带的 `lib/time/zoneinfo.zip`，或者引入标准库中的 time/tzdata 包编译进二进制文件。将 time/tzdata 编译进二进制文件会导致其大小增加约 450 KB，变化并不算明显，改动成本不高。

在 `main.go` 的 import 列表中补充 `_ "time/tzdata"`，重新编译运行，再次测试相同请求：  
```sh
head -n 10 main.go
# package main
#
# import (
#         "fmt"
#         "io"
#         "net/http"
#         "os/user"
#         "time"
#         _ "time/tzdata"
# )

sudo chroot ./new_root /main
# Service started at 127.0.0.1:8080

curl 127.0.0.1:8080/time
# 2023-10-22T23:26:21+08:00
```

### os/user
执行 `sudo chroot ./new_root /main` 命令运行静态链接得到的二进制文件，在收到 `/user` 请求时会出现 panic：`open /etc/passwd: no such file or directory`

根据 Golang os/user 标准库中 [Overview](https://pkg.go.dev/os/user#pkg-overview) 的说明，在禁用 cgo 后会使用纯 Go 自行解析 `/etc/passwd` 和 `/etc/group` 文件来获取 id 与 name 的映射关系。作为本地日常使用的 Arch Linux 环境，user 和 group 中包含了大量软件相关的角色定义，因此可以参考现有条目及 `man 5 passwd` 和 `man 5 group` 的格式说明手动在 `new_root` 目录下创建新的文件，再次测试相同请求：  
```sh
cat ./new_root/etc/passwd 
# root:x:0:0::/root:/bin/bash
# nobody:x:65534:65534:Nobody:/:/usr/bin/nologin

cat ./new_root/etc/group 
# root:x:0:root
# nobody:x:65534:

# 目录结构示例：
# .
# ├── main.go
# └── new_root
#     ├── etc
#     │   ├── group
#     │   ├── passwd
#     │   ├── resolv.conf
#     │   └── ssl
#     │       └── certs
#     │           └── ca-certificates.crt
#     └── main

sudo chroot ./new_root /main
# Service started at 127.0.0.1:8080

curl 127.0.0.1:8080/user
# user: nobody(65534)
```

## 修复方案
缺少的依赖一共有 `/etc/resolv.conf`，`ca-certificates`，`tzdata`，`/etc/passwd` 和 `/etc/group` 几项，对应的修复方案就是补全依赖。

### 手动初始化环境
尝试使用 `arch-install-scripts` 包提供的 `pacstrap -c ./new_root filesystem tzdata ca-certificates` 命令初始化 `new_root` 目录，结果得到的目录大小约 330MB，主要是 `ca-certificates-utils` 依赖了 `bash` 和 `coreutils`，再往上的依赖就越翻越多了。对比了几个发行版后发现 Alpine Linux 在其[下载页面](https://alpinelinux.org/downloads/)提供了 MINI ROOT FILESYSTEM 的选项，于是可以基于该镜像构建 chroot 环境：
```sh
# 下载并校验
wget https://dl-cdn.alpinelinux.org/alpine/v3.18/releases/x86_64/alpine-minirootfs-3.18.4-x86_64.tar.gz
sha256sum alpine-minirootfs-3.18.4-x86_64.tar.gz
# c59d5203bc6b8b6ef81f3f6b63e32c28d6e47be806ba8528f8766a4ca506c7ba  alpine-minirootfs-3.18.4-x86_64.tar.gz

# 解压并清理
mkdir new_root
tar -xvf alpine-minirootfs-3.18.4-x86_64.tar.gz -C ./new_root
rm alpine-minirootfs-3.18.4-x86_64.tar.gz

# 补充 /etc/resolv.conf
echo 'nameserver 223.5.5.5' > ./new_root/etc/resolv.conf

# 修改文件系统权限
sudo chown -R root:root ./new_root

# 更新系统并补充 tzdata
sudo chroot ./new_root /sbin/apk upgrade --update-cache --no-cache
sudo chroot ./new_root /sbin/apk add tzdata --no-cache

# 在 chroot 环境中运行
sudo chroot ./new_root /main
```
最终的 `new_root` 目录大小约 13MB，使用 `curl` 请求各个接口均正常响应。

### 容器化 distroless
之前查 kubernetes 相关资料时还找到过一类叫 [distroless](https://github.com/GoogleContainerTools/distroless) 的镜像，主要目的就是针对编程语言优化运行环境，去除大量 Linux 系统通用组件，很符合当前需求。因此可以使用 distroless 作为基础镜像来构建应用镜像，顺便借助容器化实现更加完善的运行环境隔离：
```dockerfile
# syntax=docker.io/docker/dockerfile:1.6

FROM golang:1.21-alpine AS build

WORKDIR /app
COPY . .
RUN --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=cache,target=/go/pkg/mod \
  CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" main.go

FROM gcr.io/distroless/static-debian12:latest
COPY --from=build /app/main /
ENTRYPOINT ["/main"]
```
Dockerfile 主要参考了 [moby/moby](https://github.com/moby/moby/blob/master/Dockerfile) 和 [Dreamacro/clash](https://github.com/Dreamacro/clash/blob/master/Makefile) 的部分代码，其中 `RUN --mount` 指令的支持涉及到 BuildKit 的 [Dockerfile frontend](https://docs.docker.com/build/dockerfile/frontend/)，不同版本的区别可以参考 [Dockerfile release notes](https://docs.docker.com/build/dockerfile/release-notes/)。构建镜像并运行服务：
```sh
# 目录结构示例：
# .
# ├── Dockerfile
# └── main.go

DOCKER_BUILDKIT=1 docker build --tag whoisnian/distroless-test:0.0.1 .
# whoisnian/distroless-test:0.0.1  7d1dd55a78d8  7.1MB

docker run --rm --net=host docker.io/whoisnian/distroless-test:0.0.1
# Service started at 127.0.0.1:8080
```
最终的应用镜像大小约 7.1MB，其中二进制文件大小约 5.1MB，使用 `curl` 请求各个接口均正常响应。

---
### 注：

[^1]: [ArchWiki: Installation guide](https://wiki.archlinux.org/title/Installation_guide#Chroot)  
[^2]: [ArchWiki: Building in a clean chroot](https://wiki.archlinux.org/title/DeveloperWiki:Building_in_a_clean_chroot)  
[^3]: [DOMjudge Manual: Installation of the judgehosts](https://www.domjudge.org/docs/manual/main/install-judgehost.html#creating-a-chroot-environment)  
[^4]: [HAProxy Configuration Manual: chroot](https://docs.haproxy.org/2.8/configuration.html#chroot)  
[^5]: [Arch Linux Package: ca-certificates-mozilla](https://archlinux.org/packages/core/x86_64/ca-certificates-mozilla/)  
[^6]: [Arch Linux Package: p11-kit](https://archlinux.org/packages/core/x86_64/p11-kit/)  
[^7]: [Arch Linux Package: ca-certificates-utils](https://archlinux.org/packages/core/any/ca-certificates-utils/)  
