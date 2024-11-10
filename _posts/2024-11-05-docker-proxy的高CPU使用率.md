---
layout: post
title: docker-proxy的高CPU使用率
categories: Server
---

> 本地使用 docker 来搭建 [tracing-benchmark](https://github.com/whoisnian/tracing-benchmark) 的测试环境，在测试过程中观察到默认的 bridge 网络下 `docker-proxy` 进程的 CPU 使用率甚至会高于应用容器本身。  
> 一方面不确定 `docker-proxy` 的高负载对应用容器的性能测试结果会有多大影响，另一方面则是觉得单纯的 TCP 端口转发功能不应该有这么高的 CPU 使用率。  
> 因此计划模拟并测试 `docker-proxy` 在高网络负载下的具体表现，以及相关替代方案的优化效果。  

<!-- more -->

## 问题模拟
### 环境准备
系统环境始终保持不变：
* 操作系统 Arch Linux，内核版本 6.11.6.arch1-1
* CPU 型号 AMD Ryzen 7 3700X 8C16T，内存 31.2Gi，硬盘 Samsung SSD 980 PRO 500GB
* 软件版本 docker:27.3.1 containerd:1.7.23 runc:1.2.1 ab:2.3 wrk:4.2.0

原测试环境的容器内运行 golang 编写的 web 应用，对外以 http 接口的形式提供服务，通过 `--publish 8080:8080` 接收宿主机 8080 端口的入站流量。  
模拟环境将 web 应用简化为使用 golang 标准库实现的最小服务，并保持相同构建流程打包容器镜像。其中 golang 服务代码如下：  
```go
package main

import "net/http"

func pingHandler(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("pong"))
}

func main() {
	http.HandleFunc("/ping", pingHandler)
	if err := http.ListenAndServe(":8080", nil); err != nil {
		panic(err)
	}
}
```
Dockerfile 中使用相同的基础镜像和多阶段构建逻辑：  
```dockerfile
# syntax=docker.io/docker/dockerfile:1.10
FROM docker.io/library/golang:1.23-alpine AS build

WORKDIR /app
COPY . .
RUN --mount=type=cache,target=/root/.cache/go-build/ \
    --mount=type=cache,target=/go/pkg/mod/ \
    CGO_ENABLED=0 go build -trimpath -ldflags='-s -w' \
    -o main main.go

FROM gcr.io/distroless/static-debian12:latest
COPY --from=build /app/main /main
ENTRYPOINT ["/main"]
```
执行构建命令 `DOCKER_BUILDKIT=1 docker build --progress=plain --tag pong:v1 .` 后即可得到本地镜像 `pong:v1`  

### 测试复现
为了避免本地资源抢占及资源耗尽导致的测试结果不稳定，可以通过容器启动参数来限制应用容器能够使用的最大 CPU 和内存，大小核架构的 CPU 最好也要求应用容器始终运行在指定核心上。  
测试过程中遇到了 [ab](https://httpd.apache.org/docs/2.4/programs/ab.html) 命令本身的性能瓶颈，于是换到了另一个常用的性能测试工具 [wrk](https://github.com/wg/wrk)，两者的效果对比如下：  

| docker run |    ab -c1    |     ab -c2     |     ab -c4     |  wrk -c1 -t1   |  wrk -c2 -t2   |   wrk -c4 -t4    |   wrk -c6 -t6    |   wrk -c8 -t8    |
| :--------: | :----------: | :------------: | :------------: | :------------: | :------------: | :--------------: | :--------------: | :--------------: |
| `--cpus=1` | 74% 38% 9324 | 100% 46% 13457 | 100% 42% 13003 | 100% 28% 26815 | 100% 44% 40840 |  100% 48% 40787  |  100% 45% 33100  |  100% 44% 27124  |
| `--cpus=2` | 74% 38% 9290 | 130% 58% 17044 | 150% 60% 18940 | 118% 33% 31009 | 150% 65% 60099 |  200% 95% 81361  |  200% 90% 66332  |  200% 88% 54463  |
| `--cpus=3` | 74% 38% 9210 | 130% 58% 17087 | 149% 60% 18925 | 119% 33% 31096 | 149% 65% 60441 | 273% 132% 117629 | 300% 136% 100612 | 300% 135% 82927  |
| `--cpus=4` | 74% 38% 9263 | 130% 58% 17037 | 150% 60% 18974 | 117% 33% 31068 | 150% 65% 60507 | 275% 131% 116223 | 400% 181% 137259 | 400% 177% 108210 |

* 应用容器启动命令示例 `docker run --rm --net=host --cpus=1 pong:v1`
* ab 性能测试命令示例 `ab -c1 -t30 -n1000000 http://127.0.0.1:8080/ping`
* wrk 性能测试命令示例 `wrk -c1 -t1 -d30 --latency http://127.0.0.1:8080/ping`
* CPU 使用率统计示例 `top -b -d10 -n3 -p $(pgrep -d, '^(main|ab|wrk)')`
* 测试结果格式为 `应用容器CPU使用率 测试工具CPU使用率 每秒完成的请求数`

测试过程中还发现 golang 程序在容器内获取到的 NumCPU 是宿主机的 CPU 总核数，对应的 GOMAXPROCS 默认值过大会造成频繁的上下文切换，对 CPU 密集型任务会有严重影响。  
可以使用 Uber 的 [automaxprocs](https://github.com/uber-go/automaxprocs) 在启动时按照规则自动计算 GOMAXPROCS，也可以通过环境变量手动指定一个合适的值，该值对测试结果的影响如下：  

|         docker run          |  wrk -c1 -t1   |  wrk -c2 -t2   |   wrk -c4 -t4    |   wrk -c6 -t6    |   wrk -c8 -t8    |
| :-------------------------: | :------------: | :------------: | :--------------: | :--------------: | :--------------: |
| `--cpus=2 -e GOMAXPROCS=1`  | 60% 39% 37983  | 79% 78% 75533  |  80% 78% 80021   |  79% 78% 80577   |  79% 78% 80126   |
| `--cpus=2 -e GOMAXPROCS=2`  | 102% 36% 34434 | 107% 70% 66081 | 160% 145% 128777 | 160% 159% 140328 | 160% 159% 141503 |
| `--cpus=2 -e GOMAXPROCS=3`  | 113% 33% 31078 | 137% 67% 62294 | 190% 139% 123172 | 200% 168% 140045 | 200% 184% 150832 |
| `--cpus=2 -e GOMAXPROCS=4`  | 114% 33% 31508 | 143% 67% 61602 | 200% 124% 107115 | 200% 146% 121266 | 200% 164% 130300 |
| `--cpus=2 -e GOMAXPROCS=16` | 116% 33% 31116 | 150% 66% 60348 |  200% 97% 82150  |  200% 91% 67595  |  200% 90% 55395  |

* 应用容器启动命令示例 `docker run --rm --net=host --cpus=2 -e GOMAXPROCS=1 pong:v1`
* wrk 性能测试命令示例 `wrk -c1 -t1 -d30 --latency http://127.0.0.1:8080/ping`
* CPU 使用率统计示例 `top -b -d10 -n3 -p $(pgrep -d, '^(main|wrk)')`
* 测试结果格式为 `应用容器CPU使用率 测试工具CPU使用率 每秒完成的请求数`

因此模拟测试最终使用 `--publish 8080:8080` 加上 `--cpus=2 -e GOMAXPROCS=3` 来启动应用容器，然后使用 `wrk` 来执行不同并发程度下的性能测试。  
测试结果如下，可以观察到除了 `docker-proxy` 会随着并发程度的增加而占用越来越多的 CPU，应用容器的最大吞吐量相比 `--net=host` 也降低了约三分之一。  

|       metrics        | wrk -c1 -t1 | wrk -c2 -t2 | wrk -c4 -t4 | wrk -c6 -t6 | wrk -c8 -t8 | wrk -c10 -t10 |
| :------------------: | :---------: | :---------: | :---------: | :---------: | :---------: | :-----------: |
| 应用容器 CPU 使用率  |     67%     |     96%     |    133%     |    167%     |    189%     |     200%      |
| 端口转发 CPU 使用率  |     41%     |     67%     |    153%     |    227%     |    297%     |     363%      |
| 测试工具 CPU 使用率  |     20%     |     38%     |     68%     |    104%     |    140%     |     163%      |
| 平均每秒完成的请求数 |    18559    |    37077    |    60250    |    81457    |    96570    |    104760     |

## 原因分析
### 源码实现
GitHub 上早期的 [docker/docker](https://github.com/docker/docker) 已经迁移到了 [moby/moby](https://github.com/moby/moby)，在仓库中可以很容易地搜索到 `docker-proxy` 所在的源码目录 [cmd/docker-proxy](https://github.com/moby/moby/tree/v27.3.1/cmd/docker-proxy)。  
从功能入口的 [main.go](https://github.com/moby/moby/blob/v27.3.1/cmd/docker-proxy/main.go#L54) 可以看到 `docker-proxy` 一共支持 tcp/udp/sctp 三种协议，默认的 `--publish` 走 tcp 协议，对应源码在 [tcp_proxy.go](https://github.com/moby/moby/blob/v27.3.1/cmd/docker-proxy/tcp_proxy.go)，其核心实现为：  
```go
var wg sync.WaitGroup
broker := func(to, from *net.TCPConn) {
	io.Copy(to, from)
	from.CloseRead()
	to.CloseWrite()
	wg.Done()
}

wg.Add(2)
go broker(client, backend)
go broker(backend, client)
```
即对于接收到的每一条 client 连接，会先建立一条到转发目标的 backend 连接，然后创建两个 goroutine 调用 `io.Copy()` 实现双向拷贝。  
看起来是很常规的流量转发实现，golang 标准库的 `io.Copy()` 在 linux 环境下本身也会对数据拷贝使用 `syscall.Splice()` 进行优化，因此这部分代码没有特别明显的优化空间。  

### 功能替代
根据 `ps aux` 进程列表中的 `/usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 8080 -container-ip 172.17.0.2 -container-port 8080` 可知，使用任意方式将请求流量转发到容器 IP 对应端口即可实现 `docker-proxy` 等价功能。服务端常见的的反向代理软件有 Nginx 和 HAProxy 等，本地使用 nginx:1.27.2 和 haproxy:3.0.6 进行测试。  
使用的 nginx 配置文件和测试结果如下，可以观察到高网络负载下相比 `docker-proxy` 节省了约一半的 CPU 使用率，且应用容器的最大吞吐量有少量恢复。  
```nginx
daemon off;
pid /tmp/nginx.pid;
worker_processes 4;
events {
  worker_connections 1024;
}

stream {
  server {
    listen 8080;
    proxy_pass 172.17.0.2:8080;
  }
}
```

|       metrics        | wrk -c1 -t1 | wrk -c2 -t2 | wrk -c4 -t4 | wrk -c6 -t6 | wrk -c8 -t8 | wrk -c10 -t10 |
| :------------------: | :---------: | :---------: | :---------: | :---------: | :---------: | :-----------: |
| 应用容器 CPU 使用率  |     67%     |    102%     |    141%     |    160%     |    195%     |     200%      |
| 反向代理 CPU 使用率  |     32%     |   32% 32%   |   48% 48%   |   65% 31%   | 63% 56% 31% |  57% 57% 42%  |
| 测试工具 CPU 使用率  |     20%     |     39%     |     77%     |     94%     |    145%     |     160%      |
| 平均每秒完成的请求数 |    18437    |    35893    |    67098    |    84093    |   116993    |    126640     |

* 应用容器启动命令示例 `docker run --rm --publish 8081:8080 --cpus=2 -e GOMAXPROCS=3 pong:v1`
* nginx 启动命令示例 `nginx -p $(pwd) -c ./nginx.conf`
* wrk 性能测试命令示例 `wrk -c1 -t1 -d30 --latency http://127.0.0.1:8080/ping`
* CPU 使用率统计示例 `top -b -d10 -n3 -p $(pgrep -d, '^(main|nginx|wrk)')`

使用的 haproxy 配置文件和测试结果如下，可以观察到高网络负载下相比 `docker-proxy` 节省了部分 CPU 使用率，且应用容器的最大吞吐量有少量恢复。  
```conf
global
    maxconn 4096
    log stderr local0 alert
    pidfile /tmp/haproxy.pid
    nbthread 4

defaults
    mode tcp
    timeout client 10s
    timeout connect 10s
    timeout server 10s

frontend main
    bind :8080
    default_backend container

backend container
    server app 172.17.0.2:8080
```

|       metrics        | wrk -c1 -t1 | wrk -c2 -t2 | wrk -c4 -t4 | wrk -c6 -t6 | wrk -c8 -t8 | wrk -c10 -t10 |
| :------------------: | :---------: | :---------: | :---------: | :---------: | :---------: | :-----------: |
| 应用容器 CPU 使用率  |     64%     |     99%     |    141%     |    170%     |    196%     |     200%      |
| 反向代理 CPU 使用率  |     38%     |     77%     |    151%     |    188%     |    223%     |     223%      |
| 测试工具 CPU 使用率  |     19%     |     37%     |     73%     |    108%     |    142%     |     157%      |
| 平均每秒完成的请求数 |    17955    |    34459    |    63186    |    88561    |   111106    |    120964     |

* 应用容器启动命令示例 `docker run --rm --publish 8081:8080 --cpus=2 -e GOMAXPROCS=3 pong:v1`
* haproxy 启动命令示例 `haproxy -f ./haproxy.cfg`
* wrk 性能测试命令示例 `wrk -c1 -t1 -d30 --latency http://127.0.0.1:8080/ping`
* CPU 使用率统计示例 `top -b -d10 -n3 -p $(pgrep -d, '^(main|haproxy|wrk)')`

## 解决方案
### 直接使用容器 IP
`docker-proxy` 的核心逻辑是监听本地端口，再将请求流量转发到容器 IP 的对应端口，因此在执行性能测试时可以直接使用容器 IP 作为目标地址，测试结果如下：  

|       metrics        | wrk -c1 -t1 | wrk -c2 -t2 | wrk -c4 -t4 | wrk -c6 -t6 | wrk -c8 -t8 | wrk -c10 -t10 |
| :------------------: | :---------: | :---------: | :---------: | :---------: | :---------: | :-----------: |
| 应用容器 CPU 使用率  |    107%     |    132%     |    183%     |    200%     |    200%     |     200%      |
| 测试工具 CPU 使用率  |     34%     |     69%     |    142%     |    185%     |    199%     |     198%      |
| 平均每秒完成的请求数 |    30166    |    60585    |   119059    |   142879    |   156343    |    156780     |

* 应用容器启动命令示例 `docker run --rm --publish 8080:8080 --cpus=2 -e GOMAXPROCS=3 pong:v1`
* wrk 性能测试命令示例 `wrk -c1 -t1 -d30 --latency http://172.17.0.2:8080/ping`
* CPU 使用率统计示例 `top -b -d10 -n3 -p $(pgrep -d, '^(main|wrk)')`

### 禁用 userland-proxy
默认配置下 docker 使用 `docker-proxy` 和 `iptables` 共同处理流量转发，同时在 `/etc/docker/daemon.json` 配置文件中也提供了 `userland-proxy` 配置项，该项设置为 false 后可以禁用 `docker-proxy`，只通过 `iptables` 转发流量，测试结果如下：  

|       metrics        | wrk -c1 -t1 | wrk -c2 -t2 | wrk -c4 -t4 | wrk -c6 -t6 | wrk -c8 -t8 | wrk -c10 -t10 |
| :------------------: | :---------: | :---------: | :---------: | :---------: | :---------: | :-----------: |
| 应用容器 CPU 使用率  |    101%     |    128%     |    176%     |    200%     |    200%     |     200%      |
| 测试工具 CPU 使用率  |     35%     |     70%     |    146%     |    200%     |    217%     |     218%      |
| 平均每秒完成的请求数 |    28558    |    56730    |   110939    |   141769    |   154384    |    153135     |

* 应用容器启动命令示例 `docker run --rm --publish 8080:8080 --cpus=2 -e GOMAXPROCS=3 pong:v1`
* wrk 性能测试命令示例 `wrk -c1 -t1 -d30 --latency http://127.0.0.1:8080/ping`
* CPU 使用率统计示例 `top -b -d10 -n3 -p $(pgrep -d, '^(main|wrk)')`

## 对比总结
将以上结果汇总到一起，忽略应用容器和测试工具的 CPU 使用率，仅保留流量转发工具的 CPU 使用率和应用容器的请求吞吐量，结果对比如下：  

|               metrics                | wrk -c1 -t1 | wrk -c2 -t2 | wrk -c4 -t4 | wrk -c6 -t6 | wrk -c8 -t8 | wrk -c10 -t10 |
| :----------------------------------: | :---------: | :---------: | :---------: | :---------: | :---------: | :-----------: |
|    host 网络，默认设置 GOMAXPROCS    |  00% 31009  |  00% 60099  |  00% 81361  |  00% 66332  |  00% 54463  |   00% 53571   |
|    host 网络，手动设置 GOMAXPROCS    |  00% 31078  |  00% 62294  | 00% 123172  | 00% 140045  | 00% 150832  |  00% 150762   |
|   端口映射，通过 docker-proxy 访问   |  41% 18559  |  67% 37077  | 153% 60250  | 227% 81457  | 297% 96570  |  363% 104760  |
|  端口映射，通过 nginx 转发到容器 IP  |  32% 18437  |  64% 35893  |  96% 67098  |  96% 84093  | 150% 116993 |  156% 126640  |
| 端口映射，通过 haproxy 转发到容器 IP |  38% 17955  |  77% 34459  | 151% 63186  | 188% 88561  | 223% 111106 |  223% 120964  |
|    端口映射，直接使用容器 IP 访问    |  00% 30166  |  00% 60585  | 00% 119059  | 00% 142879  | 00% 156343  |  00% 156780   |
|    端口映射，禁用 userland-proxy     |  00% 28558  |  00% 56730  | 00% 110939  | 00% 141769  | 00% 154384  |  00% 153135   |

* 对于处理 CPU 密集型任务的 golang 程序，在容器环境下需要配置合理的 GOMAXPROCS 来避免频繁切换上下文导致的性能下降。
* 对于宿主机监听 80/443 将请求反向代理到应用容器的场景，尽量避免通过 `docker-proxy` 监听的 `127.0.0.1` 进行转发，考虑直接使用容器 IP 或者禁用 userland-proxy。
* 在类似高频小请求的 tcp 反向代理场景下，nginx 和 haproxy 的性能差距并不明显，但 nginx 在 CPU 使用率上有着明显优势。

## 拓展
* 从 `docker-proxy` 的源码中可以了解到 golang 标准库的 `io.Copy()` 在 linux 环境下会使用零拷贝的 `splice` 系统调用来进行优化，那么其对比 nginx 的 CPU 使用率和内存占用劣势是由什么导致的？如果用 C 来实现 `docker-proxy` 的类似功能需要多少代码？能否有接近 nginx 的性能表现？
* 禁用 `userland-proxy` 时 docker 通过配置相关 `iptables` 规则，从而让 linux 内核中的 netfilter 模块执行实际的网络数据处理，替代了用户空间的 `docker-proxy`。那么能否通过 eBPF 实现 `docker-proxy` 的类似功能，以另一种形式替代用户空间的 `docker-proxy`？
* 通过 nginx 将请求转发到容器 IP 时，为 nginx 设置了 `worker_processes 4`，但在实际测试过程中发现四个 worker 并不会均匀处理请求：并发程度低时可能只有一个 worker 能看到明显的 CPU 使用率，其它 worker 的 CPU 使用率都为零；并发程度高时虽然多个 worker 都在工作，但 worker 之间的 CPU 使用率又有明显差别。nginx 的主进程是怎么调度工作进程的？能否进行人工干预？

---
### 附：
* 关于 `userland-proxy` 的一些介绍与讨论：[docker/docs/issues/17312](https://github.com/docker/docs/issues/17312)
* 禁用 `userland-proxy` 后导致的一些问题：[moby/moby/issues/14856](https://github.com/moby/moby/issues/14856)
