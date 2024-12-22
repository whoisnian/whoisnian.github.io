---
layout: post
title: otel-trace中的spans队列
categories: Server
---

> 还是本地搭建的 [tracing-benchmark](https://github.com/whoisnian/tracing-benchmark) 测试环境，测试过程中观察到 jaeger 在 v1 和 v2 不同版本下提供的 metrics 不同，且都包含关于 spans 数量的统计。  
> 根据测试结果可知高负载环境下不同版本均会主动丢弃 spans，好奇丢弃 spans 的具体数量，丢弃行为发生在 client 端还是 server 端，以及能否设置为禁止丢弃。  
> 因此将 otel trace 的主要逻辑提取出来并进行测试，顺便观察单个 span 从创建到保存的完整流程是什么样的。  

<!-- more -->

## 问题模拟
### 原始结果
jaeger 服务端分别使用 `jaeger-collector:1.64.0` 和 `jaeger:2.1.0`，存储后端则都是 `elasticsearch:8.15.3`，执行 `wrk -c8 -t8 -d30` 的测试结果为：  

| jaeger version | total requests | total spans | saved spans | dropped spans | dropped percent | raw metrics                                                                                                                                                                     |
| :------------: | :------------: | :---------: | :---------: | :-----------: | :-------------: | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|     1.64.0     |     435476     |   1306436   |   247340    |    1059096    |     81.07%      | [127.0.0.1:14269/metrics](https://gist.githubusercontent.com/whoisnian/6c3babac0d925667aa1eae5621d68a23/raw/f09006c90b70bb324f917f1418b4682a59272165/jaeger-metrics-1.64.0.txt) |
|     2.1.0      |     519428     |  1558292*   |   197122    |   1361170*    |     87.35%      | [127.0.0.1:8888/metrics](https://gist.githubusercontent.com/whoisnian/6c3babac0d925667aa1eae5621d68a23/raw/f09006c90b70bb324f917f1418b4682a59272165/jaeger-metrics-2.1.0.txt)   |

* 其中的 `total requests` 来自应用程序内自定义的 prometheus.CounterVec，以 [Gin Middleware](https://github.com/whoisnian/tracing-benchmark/blob/14262fab08fc44f3427c633b3685f112a6ce122e/router/router.go#L34) 的形式进行统计，可以确认是应用程序接收并处理完成的请求数量。
* 1.64.0 版本中的 `total/saved/dropped spans` 均为精确数据，分别对应 metrics 中的 `jaeger_collector_spans_received_total`、`jaeger_collector_spans_saved_by_svc_total` 和 `jaeger_collector_spans_dropped_total`。
* 2.1.0 版本中的 `saved spans` 是精确数据，与 metrics 中的 `otelcol_receiver_accepted_spans` 和 `otelcol_exporter_sent_spans` 相等，而 `total/dropped spans` 则是经计算得出。

根据 1.64.0 版本 metrics 中的数据完整性推测是 server 端发生的 spans 丢弃；但 2.1.0 版本的 metrics 中只有 `otelcol_receiver_accepted_spans` 和 `otelcol_exporter_sent_spans` 两个相等的值，类似指标 `otelcol_receiver_refused_spans` 和 `otelcol_exporter_send_failed_spans` 的值均为零，更像是在 client 端发生的 spans 丢弃，于是对问题进行简化模拟。

### 环境准备
原测试环境的 golang 应用程序是使用 Gin 框架编写的 web 应用，单次 API 请求生成的一条 trace 会包含 gin/redis/mysql 三个 span。  
模拟环境不再保留实际的请求路径，从测试环境手动复制 span 信息，使用 goroutine 创建多个 worker 并行写入。相关的 jaeger 部署配置和 golang 代码如下：  

{::options parse_block_html="true" /}
<details><summary markdown="span">compose.yaml</summary>

```yaml
name: jaeger

services:
  es:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.15.3
    restart: always
    volumes:
      - es_data:/usr/share/elasticsearch/data
    environment:
      - node.name=es
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
      - ELASTIC_PASSWORD=sHueH6Ut38ATxe4u0XvJ
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=false
    ports:
      - 9200:9200
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s http://localhost:9200 | grep -q 'missing authentication credentials'",
        ]
      interval: 10s
      timeout: 10s
      retries: 120
    cpus: '2.000'
    mem_limit: 4gb

  v1:
    image: jaegertracing/jaeger-collector:1.64.0
    restart: always
    environment:
      - GOMAXPROCS=2
      - SPAN_STORAGE_TYPE=elasticsearch
      - ES_SERVER_URLS=http://es:9200
      - ES_USERNAME=elastic
      - ES_PASSWORD=sHueH6Ut38ATxe4u0XvJ
    tmpfs:
      - /tmp
    ports:
      - 4318:4318
      - 14269:14269
    depends_on:
      es:
        condition: service_healthy
    cpus: '2.000'
    mem_limit: 4gb

  v2:
    image: jaegertracing/jaeger:2.1.0
    restart: always
    environment:
      - GOMAXPROCS=2
      - ES_SERVER_URLS=http://es:9200
      - ES_USERNAME=elastic
      - ES_PASSWORD=sHueH6Ut38ATxe4u0XvJ
    volumes:
      - ./jaeger-config.yaml:/cmd/jaeger/config.yaml
    tmpfs:
      - /tmp
    command: ["--config", "/cmd/jaeger/config.yaml"]
    ports:
      - 4319:4318
      - 8888:8888
    depends_on:
      es:
        condition: service_healthy
    cpus: '2.000'
    mem_limit: 4gb

volumes:
  es_data: {}
```
</details>

<details><summary markdown="span">jaeger-config.yaml</summary>

```yaml
# https://github.com/jaegertracing/jaeger/blob/v2.1.0/cmd/jaeger/config-badger.yaml
# https://github.com/jaegertracing/jaeger/blob/v2.1.0/cmd/jaeger/internal/all-in-one.yaml
service:
  extensions: [jaeger_storage, jaeger_query, healthcheckv2]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [jaeger_storage_exporter]
  telemetry:
    resource:
      service.name: jaeger
    metrics:
      level: basic
      address: 0.0.0.0:8888
    logs:
      level: info

extensions:
  healthcheckv2:
    use_v2: true
    http:
      endpoint: 0.0.0.0:13133

  jaeger_query:
    storage:
      traces: some_storage
    http:
      endpoint: 0.0.0.0:16686

  jaeger_storage:
    backends:
      some_storage:
        elasticsearch:
          server_urls:
            - "${env:ES_SERVER_URLS}"
          auth:
            basic:
              username: "${env:ES_USERNAME}" 
              password: "${env:ES_PASSWORD}"
          indices:
            index_prefix: "jaeger2-main"
            spans:
              date_layout: "2006-01-02"
              rollover_frequency: "day"
              shards: 5
              replicas: 1
            services:
              date_layout: "2006-01-02"
              rollover_frequency: "day"
              shards: 5
              replicas: 1
            dependencies:
              date_layout: "2006-01-02"
              rollover_frequency: "day"
              shards: 5
              replicas: 1
            sampling:
              date_layout: "2006-01-02"
              rollover_frequency: "day"
              shards: 5
              replicas: 1

receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:

exporters:
  jaeger_storage_exporter:
    trace_storage: some_storage
```
</details>

<details><summary markdown="span">main.go</summary>

```go
package main

import (
	"context"
	"flag"
	"log"
	"net/http"
	"sync"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.26.0"
	oteltrace "go.opentelemetry.io/otel/trace"
)

var CFG struct {
	Workers  int
	Traces   int
	Service  string
	Endpoint string
}

func init() {
	flag.IntVar(&CFG.Workers, "workers", 1, "Number of workers (goroutines)")
	flag.IntVar(&CFG.Traces, "traces", 1, "Number of traces for each worker")
	flag.StringVar(&CFG.Service, "service", "oteltrace", "Service name")
	flag.StringVar(&CFG.Endpoint, "endpoint", "http://127.0.0.1:4318", "OTLP http trace exporter endpoint")
	flag.Parse()
}

func main() {
	exporter, err := otlptracehttp.New(context.Background(), otlptracehttp.WithEndpointURL(CFG.Endpoint))
	panicIf(err)

	rsc, err := resource.Merge(
		resource.Default(),
		resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceName(CFG.Service),
		),
	)
	panicIf(err)

	provider := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithResource(rsc),
	)
	otel.SetTracerProvider(provider)
	otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(propagation.TraceContext{}, propagation.Baggage{}))

	wg := new(sync.WaitGroup)
	wg.Add(CFG.Workers)
	for i := 0; i < CFG.Workers; i++ {
		go handle(wg, i)
	}
	wg.Wait()

	ctx, cancel := context.WithTimeout(context.Background(), time.Second*10)
	defer cancel()
	panicIf(provider.Shutdown(ctx))
}

func handle(wg *sync.WaitGroup, index int) {
	defer wg.Done()
	log.Printf("start  worker %d", index)

	for i := 0; i < CFG.Traces; i++ {
		// parent span
		ctx, span := otel.GetTracerProvider().Tracer(
			"go.opentelemetry.io/contrib/instrumentation/github.com/gin-gonic/gin/otelgin",
			oteltrace.WithInstrumentationVersion("0.56.0"),
		).Start(context.Background(), "/ping/GRM", oteltrace.WithSpanKind(oteltrace.SpanKindServer))
		span.SetAttributes(ginAttributes...)

		// child span (redis)
		_, redisSpan := otel.GetTracerProvider().Tracer(
			"github.com/redis/go-redis/extra/redisotel",
			oteltrace.WithInstrumentationVersion("semver:9.7.0"),
		).Start(ctx, "ping", oteltrace.WithSpanKind(oteltrace.SpanKindClient))
		redisSpan.SetAttributes(redisAttributes...)
		time.Sleep(time.Microsecond * 100)
		redisSpan.End()

		// child span (mysql)
		_, mysqlSpan := otel.GetTracerProvider().Tracer(
			"gorm.io/plugin/opentelemetry",
			oteltrace.WithInstrumentationVersion("0.1.8"),
		).Start(ctx, "gorm.Raw", oteltrace.WithSpanKind(oteltrace.SpanKindClient))
		mysqlSpan.SetAttributes(mysqlAttributes...)
		time.Sleep(time.Microsecond * 100)
		mysqlSpan.End()

		span.End()
	}
	log.Printf("finish worker %d", index)
}

func panicIf(err error) {
	if err != nil {
		panic(err)
	}
}

var (
	ginAttributes = []attribute.KeyValue{
		semconv.HTTPRequestMethodGet,
		semconv.HTTPRoute("/ping/GRM"),
		attribute.String("http.scheme", "http"),          // exists in go.opentelemetry.io/otel/semconv/v1.20.0
		attribute.Int("http.status_code", http.StatusOK), // exists in go.opentelemetry.io/otel/semconv/v1.20.0
		attribute.String("http.target", "/ping/GRM"),     // exists in go.opentelemetry.io/otel/semconv/v1.20.0
		semconv.NetworkTypeIpv4,
		semconv.NetworkLocalAddress("172.18.0.4"),
		semconv.NetworkLocalPort(8080),
		semconv.NetworkProtocolName("http"),
		semconv.NetworkProtocolVersion("1.1"),
		semconv.NetworkPeerAddress("172.18.0.1"),
		semconv.NetworkPeerPort(58868),
	}
	redisAttributes = []attribute.KeyValue{
		semconv.CodeFilepath("github.com/whoisnian/tracing-benchmark/router/handler.go"),
		semconv.CodeFunction("router.pingRedis"),
		semconv.CodeLineNumber(49),
		semconv.DBSystemRedis,
		attribute.String("db.connection_string", "redis://redis:6379"), // exists in go.opentelemetry.io/otel/semconv/v1.24.0
		semconv.ServerAddress("redis"),
		semconv.ServerPort(6379),
		semconv.DBOperationName("ping"),
		semconv.DBQueryText("ping"),
	}
	mysqlAttributes = []attribute.KeyValue{
		semconv.DBSystemMySQL,
		semconv.ServerAddress("mysql"),
		semconv.ServerPort(3306),
		semconv.DBOperationName("select"),
		semconv.DBQueryText("SELECT 1"),
		attribute.Int("db.rows_affected", 0), // exists in gorm.io/plugin/opentelemetry@v0.1.8
	}
)
```
</details>
{::options parse_block_html="false" /}

### 测试复现
* 测试不同版本：
  * 1.64.0：`go run main.go -endpoint http://127.0.0.1:4318 -workers 8 -traces 10000`
  * 2.1.0：`go run main.go -endpoint http://127.0.0.1:4319 -workers 8 -traces 10000`
* 记录测试结果：
  * 1.64.0：`curl -s 127.0.0.1:14269/metrics | grep -P '(oteltrace|_dropped_total{)'`
  * 2.1.0：`curl -s 127.0.0.1:8888/metrics | grep '_spans{'`

| jaeger version | duration | total spans | saved spans | dropped spans | dropped percent |
| :------------: | :------: | :---------: | :---------: | :-----------: | :-------------: |
|     1.64.0     |   18s    |   240000    |   227611    |     12389     |      5.16%      |
|     2.1.0      |   18s    |   240000*   |   232106    |     7894*     |      3.29%      |

## 原因分析
### 源码实现
TODO：单个 span 从创建到保存的完整流程是什么样的，丢弃行为具体发生在哪里
### 功能设置
TODO：2.1.0 版本能否获取到 spans 的完整指标，spans 能否设置为禁止丢弃
## 拓展
TODO：自动丢弃队列的功能实现及状态维护
