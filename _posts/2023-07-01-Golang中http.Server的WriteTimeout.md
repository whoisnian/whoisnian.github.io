---
layout: post
title: Golang中http.Server的WriteTimeout
categories: Programming
---

> 新功能上线后 Nginx 的日志监控频繁上报某类请求 `502 Bad Gateway`，但在对应的 Golang 后端服务日志中却没有找到任何异常。  
> 抓取请求参数在测试环境复现后，发现异常请求在 Golang 后端服务日志中会正常打印为 `200 OK`，与其他正常请求无区别。  
> 根据 Nginx 日志中的请求时间推测，是 Golang 后端服务在请求超过 60 秒后主动关闭了连接，但并没有处理相关超时异常。  

<!-- more -->

## 问题分析
由于是 Nginx 日志中出现的 502，所以直接根据请求路由查找上游服务，期望对应的后端服务记录了详细错误。但在对应的服务日志中并没有找到任何异常，只能回到 Nginx 日志继续分析。  
收集了对应接口的同类请求日志后，发现异常请求的响应时间都比较接近，例如 63.349s，66.804s，60.167s，而正常请求的响应时间则分散在 60 秒内，如 32.090s，18.052s，42.306s。  
因此推测是 Golang 后端服务内自行配置了 60 秒的超时时间，在请求超过 60 秒后会主动关闭连接，但并没有处理相关超时异常，导致没有留下相关日志。  
在后端服务代码仓库内全局搜索，确实找到了 `http.Server` 初始化时 `WriteTimeout: 60 * time.Second` 的配置，看起来该配置与框架无关，使用 Golang 标准库就可以模拟复现。  

## 模拟复现
查看服务器上的 Nginx 配置文件，提取其中主要配置如下：  
```nginx
http {
    proxy_http_version        1.1;
    proxy_request_buffering   off;
    proxy_read_timeout        180;
    proxy_buffer_size         16k;
    proxy_buffers             8 16k;
    proxy_busy_buffers_size   32k;

    # proxy_connect_timeout     default 60s
    # proxy_send_timeout        default 60s
    # send_timeout              default 60s

    server {
        listen          8080;
        server_name     _;

        location /go/v1/ {
            proxy_pass  http://127.0.0.1:9500;
            proxy_set_header Host $http_host;
            proxy_set_header X-Original-ADDR  $http_host;
            proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
            add_header Cache-Control  "no-cache, no-store";
        }
    }
}
```

对应的 Golang 后端服务使用的是 [Gin](https://github.com/gin-gonic/gin) 框架，通过 middleware 的形式使用 [zap](https://github.com/uber-go/zap) 格式化请求日志，使用标准库可以简化为：  
```go
package main

import (
	"log"
	"net/http"
	"time"
)

type loggingResponseWriter struct {
	http.ResponseWriter
	status int
}

func (lw *loggingResponseWriter) WriteHeader(code int) {
	lw.status = code
	lw.ResponseWriter.WriteHeader(code)
}

func withLogging(handler http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		st := time.Now()
		lw := &loggingResponseWriter{w, http.StatusOK}

		handler.ServeHTTP(lw, r)

		log.Print("[R] ",
			r.RemoteAddr, " [",
			lw.status, "] ",
			r.Method, " ",
			r.RequestURI, " ",
			r.UserAgent(), " ",
			time.Since(st).Milliseconds(),
		)
	})
}

func testHandler(w http.ResponseWriter, _ *http.Request) {
	// time.Sleep(65 * time.Second)
	w.Write([]byte("ok\n"))
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/go/v1/test", testHandler)

	server := &http.Server{
		Addr:         "127.0.0.1:9500",
		Handler:      withLogging(mux),
		WriteTimeout: 60 * time.Second,
	}
	if err := server.ListenAndServe(); err != nil {
		log.Fatal(err)
	}
}
```
在 testHandler 中使用 `time.Sleep(65 * time.Second)` 代替一批耗时较长的第三方接口调用，在此期间没有对 ResponseWriter 的任何写入。使用 curl 请求时现象为：  
* 2023-07-02 16:39:15
  * 使用 `curl 127.0.0.1:8080/go/v1/test` 发送请求到 Nginx，监视各服务日志内容
* 2023-07-02 16:40:20
  * curl 收到 Nginx 的 `502 Bad Gateway` 响应，总耗时 65s
  * Nginx 的 access.log 内打印请求日志 `"GET /go/v1/test HTTP/1.1" 502 150 "-" "curl/8.1.2"`
  * Nginx 的 error.log 内打印报错信息 `upstream prematurely closed connection while reading response header from upstream`
  * Goalng 后端服务打印请求日志 `[200] GET /go/v1/test curl/8.1.2 65004`

结合 http.ResponseWriter 的具体实现 [http.response](https://github.com/golang/go/blob/5b72f45dd17314af39627c2fcac0fbc099b67603/src/net/http/server.go#L423) 可以看到，对应的 struct 内部存在多层缓冲区，因此在写入少量数据时只是写入到了缓冲区，并未实际从网络发送出去，所以 testHandler 中的 `w.Write()` 调用尚未触发实际的数据发送，甚至在 withLogging 打印出请求日志时也尚未实际发送数据，需要运行到 [http.conn.serve()](https://github.com/golang/go/blob/master/src/net/http/server.go#L2015) 里的 `w.finishRequest()` 时才会有 `Flush()` 去触发实际的数据发送。  

根据 [http.conn.readRequest()](https://github.com/golang/go/blob/5b72f45dd17314af39627c2fcac0fbc099b67603/src/net/http/server.go#L989) 中的 `c.rwc.SetWriteDeadline()` 判断，为 http.Server 设置的 WriteTimeout 会被设置到 net.Conn 上，只有调用到 net.Conn 的 `Write()` 操作时才会检查是否超时，因此并不是在达到超时限制时断开连接，而是在超时后的下次 `Write()` 调用时断开连接。  

## 测试验证
尝试修改 testHandler 的具体写入逻辑，再为 http.Server 设置不同的 WriteTimeout 来验证相关推测：  
```go
func testHandler(w http.ResponseWriter, _ *http.Request) {
	data := bytes.Repeat([]byte{'0'}, 1024) // 1 KiB
	for i := 0; i < 65; i++ {
		time.Sleep(time.Second)
		n, err := w.Write(data)
		log.Printf("[I] w.Write(data) = (%v, %v)\n", n, err)
		if err != nil {
			log.Panic(err)
		}
	}
}
```

__当设置 WriteTimeout 为 1s 时，应当在缓冲区填满，触发 net.Conn 实际发送时断开连接：__  
* 2023-07-02 20:42:29
  * 使用 `curl 127.0.0.1:8080/go/v1/test > /dev/null` 发送请求到 Nginx，监视各服务日志内容
* 2023-07-02 20:42:30
  * Goalng 后端服务开始打印 `[I] w.Write(data) = (1024, <nil>)`，每秒一条，持续到 20:42:33
* 2023-07-02 20:42:34
  * curl 收到 Nginx 的 `502 Bad Gateway` 响应，总耗时 5s
  * Nginx 的 access.log 内打印请求日志 `"GET /go/v1/test HTTP/1.1" 502 150 "-" "curl/8.1.2"`
  * Nginx 的 error.log 内打印报错信息 `upstream prematurely closed connection while reading response header from upstream`
  * Goalng 后端服务触发 panic，打印报错信息 `write tcp 127.0.0.1:9500->127.0.0.1:53364: i/o timeout`

__当设置 WriteTimeout 为 10s 时，应当在缓冲区首次填满时成功发送 header 和部分响应内容，在超时后的下一次缓冲区填满触发实际发送时断开连接：__  
* 2023-07-02 20:55:16
  * 使用 `curl 127.0.0.1:8080/go/v1/test > /dev/null` 发送请求到 Nginx，监视各服务日志内容
* 2023-07-02 20:55:17
  * Goalng 后端服务开始打印 `[I] w.Write(data) = (1024, <nil>)`，每秒一条，持续到 20:55:28
* 2023-07-02 20:55:21
  * curl 收到部分响应内容，Received 从 0 跳到 3940
* 2023-07-02 20:55:25
  * curl 收到部分响应内容，Received 从 3940 跳到 8022
* 2023-07-02 20:55:29
  * curl 打印错误信息 `curl: (18) transfer closed with outstanding read data remaining`
  * Nginx 的 access.log 内打印请求日志 `"GET /go/v1/test HTTP/1.1" 200 8036 "-" "curl/8.1.2"`
  * Nginx 的 error.log 内打印报错信息 `upstream prematurely closed connection while reading upstream`
  * Goalng 后端服务触发 panic，打印报错信息 `write tcp 127.0.0.1:9500->127.0.0.1:34262: i/o timeout`

## 解决办法
由于是在实际发送数据时才会检测是否超时，所以 http.Server 的 WriteTimeout 适用场景应该是持续响应数据的接口，在每次 http.ResponseWriter 调用 `Write()` 操作后调用者需要自行处理超时异常，提前结束 Handler，跳过后续步骤。  
对于数据处理完成后单次写入最终响应的接口，WriteTimeout 并不会在达到超时限制时主动断开连接，只会在 Handler 执行结束后发送数据时检测是否已超时，判断是否需要断开连接，所以这种场景下更适合不设置 http.Server 的 WriteTimeout。  
如果出于某些原因必须要设置 WriteTimeout 的话，则需要自行捕获超时异常并记录到服务端日志中。  

手动调用 [http.response](https://github.com/golang/go/blob/5b72f45dd17314af39627c2fcac0fbc099b67603/src/net/http/server.go#L1710) 的 `Flush()` 可以触发 header 的发送，但标准库内的 http.Flusher 屏蔽了返回值，需要调用到下一层的 `FlushError()` 来获取超时异常：  
```go
type ErrorFlusher interface {
	FlushError() error
}

func withLogging(handler http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		st := time.Now()
		lw := &loggingResponseWriter{w, http.StatusOK}

		handler.ServeHTTP(lw, r)

		timeoutMsg := ""
		if ef, ok := lw.ResponseWriter.(ErrorFlusher); ok {
			if err := ef.FlushError(); errors.Is(err, os.ErrDeadlineExceeded) {
				timeoutMsg = "(timeout)"
			} else if err != nil {
				log.Panic(err)
			}
		}

		log.Print("[R] ",
			r.RemoteAddr, " [",
			lw.status, "] ",
			r.Method, " ",
			r.RequestURI, " ",
			r.UserAgent(), " ",
			time.Since(st).Milliseconds(),
			timeoutMsg,
		)
	})
}
```
