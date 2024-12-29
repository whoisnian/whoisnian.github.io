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
<details><summary markdown="span" style="cursor:pointer;background-color:#f9f9f9;">compose.yaml</summary>

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

<details><summary markdown="span" style="cursor:pointer;background-color:#f9f9f9;">jaeger-config.yaml</summary>

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

<details><summary markdown="span" style="cursor:pointer;background-color:#f9f9f9;">main.go</summary>

```go
package main

import (
	"context"
	"flag"
	"log"
	"net/http"
	"sync"
	"time"

	"github.com/go-logr/stdr"
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
	Debug    bool
	Workers  int
	Traces   int
	Service  string
	Endpoint string
}

func init() {
	flag.BoolVar(&CFG.Debug, "debug", false, "Enable debug output")
	flag.IntVar(&CFG.Workers, "workers", 1, "Number of workers (goroutines)")
	flag.IntVar(&CFG.Traces, "traces", 1, "Number of traces for each worker")
	flag.StringVar(&CFG.Service, "service", "oteltrace", "Service name")
	flag.StringVar(&CFG.Endpoint, "endpoint", "http://127.0.0.1:4318", "OTLP http trace exporter endpoint")
	flag.Parse()
}

func main() {
	if CFG.Debug {
		stdr.SetVerbosity(8)
	}

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
#### OpenTelemetry SDK
根据文档 [Migration to OpenTelemetry SDK](https://www.jaegertracing.io/sdk-migration/)，Jaeger client 在 2022 年就已经被停用，并推荐迁移到 OpenTelemetry SDK。在 1.64.0 版本的文档中也已经[清理了](https://github.com/jaegertracing/documentation/pull/788) Jaeger client 的相关说明。  
模拟环境的 OpenTelemetry SDK 接入代码主要参考 [OpenTelemetry-Go: Getting Started](https://opentelemetry.io/docs/languages/go/getting-started/)，整体可以分为 **初始化 SDK** 和 **提交 span** 两部分，具体实现为：  
* **初始化 SDK：**
  * 初始化 exporter 负责向服务端 HTTP 接口发送 [protobuf](https://github.com/open-telemetry/opentelemetry-go/blob/bc2fe88756962b76eb43ea2fd92ed3f5b6491cc0/exporters/otlp/otlptrace/otlptracehttp/client.go#L127) 格式的 spans
  * 初始化 resource 记录自身作为上报者的[识别信息](https://github.com/open-telemetry/opentelemetry-go/blob/bc2fe88756962b76eb43ea2fd92ed3f5b6491cc0/sdk/resource/resource.go#L222)
  * 使用 exporter 初始化 BatchSpanProcessor，再使用 BatchSpanProcessor 和 resource 初始化 TracerProvider，并保存至全局
* **提交 span：**
  * 从全局获取 TracerProvider 创建 Tracer，再使用 Tracer 创建 span，在 span 上记录信息并提交。span 的创建和提交操作会调用 BatchSpanProcessor 的 `OnStart()` 和 `OnEnd()`
  * BatchSpanProcessor 的 [OnStart](https://github.com/open-telemetry/opentelemetry-go/blob/bc2fe88756962b76eb43ea2fd92ed3f5b6491cc0/sdk/trace/batch_span_processor.go#L125) 不包含任何处理逻辑，而 [OnEnd](https://github.com/open-telemetry/opentelemetry-go/blob/bc2fe88756962b76eb43ea2fd92ed3f5b6491cc0/sdk/trace/batch_span_processor.go#L128) 则会调用 [enqueue](https://github.com/open-telemetry/opentelemetry-go/blob/bc2fe88756962b76eb43ea2fd92ed3f5b6491cc0/sdk/trace/batch_span_processor.go#L366) 函数，根据配置项 [BlockOnQueueFull](https://github.com/open-telemetry/opentelemetry-go/blob/bc2fe88756962b76eb43ea2fd92ed3f5b6491cc0/sdk/trace/batch_span_processor.go#L57) 的不同，队列满时可以阻塞等待直至队列出现空位，也可以直接丢弃并将 [dropped](https://github.com/open-telemetry/opentelemetry-go/blob/bc2fe88756962b76eb43ea2fd92ed3f5b6491cc0/sdk/trace/batch_span_processor.go#L67) 计数器加一
  * BatchSpanProcessor 在创建时就会启动一个单独的 goroutine 来执行 [processQueue](https://github.com/open-telemetry/opentelemetry-go/blob/bc2fe88756962b76eb43ea2fd92ed3f5b6491cc0/sdk/trace/batch_span_processor.go#L117) 监听队列，并按照设定的 [BatchTimeout](https://github.com/open-telemetry/opentelemetry-go/blob/bc2fe88756962b76eb43ea2fd92ed3f5b6491cc0/sdk/trace/batch_span_processor.go#L40) 或 [MaxExportBatchSize](https://github.com/open-telemetry/opentelemetry-go/blob/bc2fe88756962b76eb43ea2fd92ed3f5b6491cc0/sdk/trace/batch_span_processor.go#L51) 来触发 [exportSpans](https://github.com/open-telemetry/opentelemetry-go/blob/bc2fe88756962b76eb43ea2fd92ed3f5b6491cc0/sdk/trace/batch_span_processor.go#L263)，再往下会调用到 exporter 的 [ExportSpans](https://github.com/open-telemetry/opentelemetry-go/blob/bc2fe88756962b76eb43ea2fd92ed3f5b6491cc0/sdk/trace/batch_span_processor.go#L277)，将 spans 实际发送出去

在 [batchSpanProcessor.exportSpans](https://github.com/open-telemetry/opentelemetry-go/blob/bc2fe88756962b76eb43ea2fd92ed3f5b6491cc0/sdk/trace/batch_span_processor.go#L263) 的源码中看到有额外的 Debug 日志，对应的 logger 实现使用了 [stdr.New](https://github.com/open-telemetry/opentelemetry-go/blob/bc2fe88756962b76eb43ea2fd92ed3f5b6491cc0/internal/global/internal_logging.go#L21)，查找[相关文档](https://pkg.go.dev/github.com/go-logr/stdr)后在代码中提前设置 `stdr.SetVerbosity(8)` 即可正常输出日志。  
开启 Debug 日志后再次执行测试，发现服务端使用 1.64.0 版本时 client 端确实未出现 spans 丢弃，而服务端切换为 2.1.0 版本后则在 client 端发生了 spans 丢弃。  

#### Jaeger v1
旧版 `jaeger-collector:1.64.0` 的程序入口在 [cmd/collector/main.go](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/main.go)，主要功能由 jaeger 自行实现，重点关注的部分为：
* metrics
  * `jaeger_collector_spans_dropped_total` 初始化于 [NewSpanProcessorMetrics](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/metrics.go#L125)，实际使用是在 [NewSpanProcessor](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L58) 创建 [BoundedQueue](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L91) 时，作为 [droppedItemHandler](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L85) 传入
  * `jaeger_collector_spans_received_total` 初始化于 [NewSpanProcessorMetrics](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/metrics.go#L117)，实际使用是在 [ProcessSpans](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L167) 中执行了 [enqueueSpan](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L219)，但尚未提交至 [BoundedQueue](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L236) 之前
  * `jaeger_collector_spans_saved_by_svc_total` 初始化于 [NewSpanProcessorMetrics](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/metrics.go#L130)，实际使用是在 [BoundedQueue](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L65) 的异步消费 [processItemFromQueue](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L191)，进一步调用到 [processSpan](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L127) 中的 [saveSpan](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L138) 时
* spans 接收
* spans 发送

#### Jaeger v2
新版 `jaeger:2.1.0` 的程序入口在 [cmd/jaeger/main.go](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/jaeger/main.go)，主要功能来自 [go.opentelemetry.io/collector](https://pkg.go.dev/go.opentelemetry.io/collector)，重点关注的部分为：
* metrics
  * `otelcol_receiver_accepted_spans`
  * `otelcol_exporter_sent_spans`
* spans 接收
* spans 发送

### 功能设置
TODO:   
* 从前面的源码分析中得知服务端使用 2.1.0 版本时是 client 丢弃，因此 server 端不知道实际的 spans 数量，完整数据需要在 client 端进行统计  
* 例如补充一个自定义 SpanProcessor 用于统计 spans 总数，封装一层 exporter 统计实际发送的 spans 数量，计算两者差值可以得到实际的丢弃数量  
* v2 版本直接在 client 端设置 BlockOnQueueFull 可以实现禁止丢弃  
* v1 版本未找到队列行为参数设置，还需要修改服务端源码才可以实现禁止丢弃  

## 拓展
TODO: [BoundedQueue](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/pkg/queue/bounded_queue.go)
