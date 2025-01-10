---
layout: post
title: otel-trace中的spans队列
categories: server
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
### OpenTelemetry SDK
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

### Jaeger v1
旧版 `jaeger-collector:1.64.0` 的程序入口在 [cmd/collector/main.go](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/main.go)，主要功能由 jaeger 自行实现，重点关注的部分为：
* metrics
  * `jaeger_collector_spans_dropped_total` 初始化于 [NewSpanProcessorMetrics](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/metrics.go#L125)，实际使用是在 [NewSpanProcessor](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L58) 创建 [BoundedQueue](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L91) 时，作为 [droppedItemHandler](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L85) 传入
  * `jaeger_collector_spans_received_total` 初始化于 [NewSpanProcessorMetrics](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/metrics.go#L117)，实际使用是在 [ProcessSpans](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L167) 中执行了 [enqueueSpan](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L219)，但尚未提交至 [BoundedQueue](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L236) 之前
  * `jaeger_collector_spans_saved_by_svc_total` 初始化于 [NewSpanProcessorMetrics](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/metrics.go#L130)，实际使用是在 [BoundedQueue](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L65) 的异步消费 [processItemFromQueue](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L191)，进一步调用到 [processSpan](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L127) 中的 [saveSpan](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L138) 时
  * [startOTLPReceiver](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/handler/otlp_receiver.go#L53) 中通过配置 [TelemetrySettings.MeterProvider](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/handler/otlp_receiver.go#L79) 使得 otlpReceiver 自身 [ObsReport](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/receiver/receiverhelper/obsreport.go#L53) 的 metrics 失效
* spans 接收
  * `collector.otlp.enabled` 配置项的[默认值](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/flags/flags.go#L185)为 true，因此会使用 [StartOTLPReceiver](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/collector.go#L149) 创建 [otlpReceiver](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/receiver/otlpreceiver/factory.go#L29) 来接收 OpenTelemetry SDK 发送的 spans，当调用 [otlpReceiver.Start](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/handler/otlp_receiver.go#L98) 时会执行到 [otlpReceiver.startHTTPServer](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/receiver/otlpreceiver/otlp.go#L192) 的内部逻辑，在默认的 [TracesURLPath](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/receiver/otlpreceiver/factory.go#L22C2-L22C22) 路由上注册 [handleTraces](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/receiver/otlpreceiver/otlp.go#L138)
  * [handleTraces](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/receiver/otlpreceiver/otlphttp.go#L28) 会将收到的客户端请求解析后交给 [tracesReceiver](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/receiver/otlpreceiver/otlphttp.go#L45)，初始化自外层的 [nextTraces](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/receiver/otlpreceiver/otlp.go#L137)，而 nextTraces 则是 jaeger 在执行 [startOTLPReceiver](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/handler/otlp_receiver.go#L39) 时通过 [otlpFactory.CreateTraces](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/receiver/otlpreceiver/factory.go#L65) 封装的 [nextConsumer](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/receiver/otlpreceiver/factory.go#L82)，即 jaeger 提前初始化的 [spanProcessor](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/collector.go#L102)
  * spanProcessor 的 [ProcessSpans](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L167) 只包含 [enqueueSpan](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L182)，即提交至 [BoundedQueue](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L236)，并不包含实际的 spans 保存
* spans 保存
  * [NewSpanProcessor](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L65) 在初始化的时候会启动多个消费者异步执行 [processItemFromQueue](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L191)，实际的 [processSpan](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L127) 里至少包含了 `preSave`，`saveSpan` 和 `countSpan` 几个步骤
  * [saveSpan](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L138) 会调用到 [spanWriter.WriteSpan](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L150)，spanWriter 来自外层的 [storageFactory.CreateSpanWriter](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/main.go#L75)，对于 elasticsearch 来说就是 [esFactory.createSpanWriter](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/plugin/storage/es/factory.go#L262)，最终到 [writeSpan](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/plugin/storage/es/spanstore/writer.go#L152)

因此 `jaeger-collector:1.64.0` 接收 spans 的 otlpReceiver 来自 [opentelemetry-collector](https://github.com/open-telemetry/opentelemetry-collector)，接收到数据后再依次交由自行实现的 SpanProcessor 和 storage plugin 进行后续处理。  
SpanProcessor 内维护了一个队列 BoundedQueue，在消费速度慢导致队列排满时就会主动丢弃 spans，从 client 端只能看到请求被正常响应，client 端感知不到服务端的主动丢弃。  

### Jaeger v2
新版 `jaeger:2.1.0` 的程序入口在 [cmd/jaeger/main.go](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/jaeger/main.go)，主要功能均来自 [opentelemetry-collector](https://github.com/open-telemetry/opentelemetry-collector)，重点关注的部分为：
* metrics
  * `otelcol_receiver_accepted_spans` 由 [mdatagen](https://github.com/open-telemetry/opentelemetry-collector/tree/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/cmd/mdatagen) 生成为 [ReceiverAcceptedSpans](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/receiver/receiverhelper/internal/metadata/generated_telemetry.go#L68)，在 [receiverhelper](https://github.com/open-telemetry/opentelemetry-collector/tree/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/receiver/receiverhelper) 的 [ObsReport.EndTracesOp](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/receiver/receiverhelper/obsreport.go#L80) 中通过 [recordMetrics](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/receiver/receiverhelper/obsreport.go#L196) 更新，实际使用是在 [httpTracesReceiver](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/receiver/otlpreceiver/otlp.go#L137) 的 [Export](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/receiver/otlpreceiver/internal/trace/otlp.go#L33) 中
  * `otelcol_exporter_sent_spans` 由 [mdatagen](https://github.com/open-telemetry/opentelemetry-collector/tree/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/cmd/mdatagen) 生成为 [ExporterSentSpans](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/exporter/exporterhelper/internal/metadata/generated_telemetry.go#L146)，在 [exporterhelper](https://github.com/open-telemetry/opentelemetry-collector/tree/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/exporter/exporterhelper) 的 [ObsReport.EndTracesOp](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/exporter/exporterhelper/internal/obsexporter.go#L60) 中通过 [recordMetrics](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/exporter/exporterhelper/internal/obsexporter.go#L118) 更新，实际使用是在 [BaseExporter](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/exporter/exporterhelper/internal/base_exporter.go#L146) 链式调用到 [ObsrepSender.Send](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/exporter/exporterhelper/traces.go#L157) 时
* spans 接收
  * jaeger 只是提供了部分[配置项](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/jaeger/internal/command.go#L36)，然后初始化 [otelcol.Collector](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/otelcol/command.go#L32) 并执行 [Collector.Run](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/otelcol/command.go#L36)。主体逻辑在 [setupConfigurationComponents](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/otelcol/collector.go#L159) 中，初始化 [col.service](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/otelcol/collector.go#L183) 并执行 [Service.Start](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/otelcol/collector.go#L228)
  * 初始化 [Service](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/service/service.go#L102) 时会使用传入的 Configs 和 Factories 创建 [srv.host.Receivers](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/service/service.go#L106C72-L106C81)/[srv.host.Processors](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/service/service.go#L107)/[srv.host.Exporters](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/service/service.go#L108)，然后执行 [initGraph](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/service/service.go#L195) 调用 graph.Build 创建 [srv.host.Pipelines](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/service/service.go#L342C5-L342C23)
  * [graph.Build](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/service/internal/graph/graph.go#L73) 中先通过 [createNodes](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/service/internal/graph/graph.go#L95) 依次创建 receiverNode/processorNode/exporterNode，再通过 [createEdges](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/service/internal/graph/graph.go#L251) 按顺序连接 `receiverNode -> capabilitiesNode -> processorNode -> fanOutNode -> exporterNode`，最后通过 [buildComponent](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/service/internal/graph/graph.go#L280) 调用 [Factory.CreateTraces](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/service/internal/builders/receiver.go#L48) 创建实例并保存为 [Node.Component](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/service/internal/graph/receiver.go#L55)
  * 执行 [Service.Start](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/otelcol/collector.go#L228) 时会执行 [srv.host.Pipelines.StartAll](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/service/service.go#L253)，依次调用前面创建保存的各个 [Component.Start](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/service/internal/graph/graph.go#L419)，例如 [receiver.Factory.createTraces](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/receiver/otlpreceiver/factory.go#L65) 得到的 [otlpReceiver.Start](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/receiver/otlpreceiver/otlp.go#L188)
  * 配置文件中使用的 processors 只有 [batch](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/processor/batchprocessor/batch_processor.go#L113)，在默认未设置 `metadata_keys` 时会使用 [singleShardBatcher](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/processor/batchprocessor/batch_processor.go#L130) 创建 [batcher](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/processor/batchprocessor/batch_processor.go#L152)，其中 newItem 使用 `make(chan T, runtime.NumCPU())` 和一个 [goroutine](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/processor/batchprocessor/batch_processor.go#L183) 做 batch，无主动丢弃逻辑。将 buffered channel size 从 1 改为 `runtime.NumCPU` 的 commit 在 [29cd959](https://github.com/open-telemetry/opentelemetry-collector/commit/29cd959efcf89cd947a5838e3678cf6166a124c8)，无详细原因说明，推测是 receiver 会并行处理请求，为了减少数据写入 channel 时阻塞导致的 goroutine 切换。但对应到并行请求数量的话感觉 `runtime.GOMAXPROCS` 更为合适
* spans 保存
  * jaeger 同样是只提供部分[配置项](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/jaeger/internal/command.go#L36)，然后初始化 [otelcol.Collector](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/otelcol/command.go#L32) 并执行 [Collector.Run](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/otelcol/command.go#L36)。主体逻辑在 [setupConfigurationComponents](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/otelcol/collector.go#L159) 中，初始化 [col.service](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/otelcol/collector.go#L183) 并执行 [Service.Start](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/otelcol/collector.go#L228)
  * 初始化 [Service](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/service/service.go#L102) 时会使用传入的 Configs 和 Factories 创建 [srv.host.Receivers](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/service/service.go#L106C72-L106C81)/[srv.host.Processors](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/service/service.go#L107)/[srv.host.Exporters](https://github.com/open-telemetry/opentelemetry-collector/blob/4ed80bbc4d9a6753ba6b959f5625a6f75fa1229c/service/service.go#L108)，spans 保存主要涉及 srv.host.Exporters，jaeger 在提供 [factories.Exporters](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/jaeger/internal/components.go#L90) 时添加了自定义的 [storageexporter.NewFactory](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/jaeger/internal/exporters/storageexporter/factory.go#L23)，依赖自定义的 [jaeger_storage extension](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/jaeger/internal/exporters/storageexporter/exporter.go#L37) 来初始化 [traceWriter](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/jaeger/internal/exporters/storageexporter/exporter.go#L42C9-L42C20)
  * 配置文件中使用的 jaeger_storage extension 只有 [elasticsearch](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/jaeger/internal/extension/jaegerstorage/extension.go#L157)，初始化 traceWriter 时调用的 [CreateTraceWriter](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/storage_v2/factoryadapter/factory.go#L47) 实际是 storage_v1 版本 CreateSpanWriter 的封装，对于 elasticsearch 来说就是 [esFactory.createSpanWriter](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/plugin/storage/es/factory.go#L262)，最终与 Jaeger v1 的 spans 保存实现相同，都会到 [writeSpan](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/plugin/storage/es/spanstore/writer.go#L152)

因此 `jaeger:2.1.0` 接收处理 spans 的 receivers 和 processors 都来自 [opentelemetry-collector](https://github.com/open-telemetry/opentelemetry-collector)，但保存数据的 jaeger_storage_exporter 是对自身已有 storage_v1 的封装。  
BatchProcessor 内维护了一个队列 singleShardBatcher，但该队列不会主动丢弃 spans，在队列满时服务端会出现请求阻塞，推测 opentelemetry 更倾向把大量 spans 的场景优化放在 client 端。  

## 功能增强
### Jaeger v1
从原因分析中可知服务端使用 `jaeger-collector:1.64.0` 时是服务端主动丢弃，client 端无任何感知，因此完整数据指标只能以服务端为准。  
此外服务端 BoundedQueue 未找到行为参数设置，需要针对源码进行修改，再加上 client 端设置 BlockOnQueueFull 才可以实现禁止丢弃。  
```go
// https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/pkg/queue/bounded_queue.go#L104
func (q *BoundedQueue) ProduceSync(item any) bool {
	if q.stopped.Load() != 0 {
		q.onDroppedItem(item)
		return false
	}

	q.size.Add(1)
	*q.items <- item
	return true
}

// https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/cmd/collector/app/span_processor.go#L219
func (sp *spanProcessor) enqueueSpan(span *model.Span, originalFormat processor.SpanFormat, transport processor.InboundTransport, tenant string) bool {
	// ...
	return sp.queue.ProduceSync(item) // Produce() => ProduceSync()
}

func main() {
	// ...
	provider := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter, sdktrace.WithBlocking()),
		sdktrace.WithResource(rsc),
	)
	// ...
}
```

### Jaeger v2
从原因分析中可知服务端使用 `jaeger:2.1.0` 时是 client 端主动丢弃，因此服务端并不知道实际的 spans 数量，完整数据指标需要在 client 端进行统计。  
例如补充一个自定义 SpanProcessor 用于统计 spans 总数，封装一层 exporter 统计实际发送的 spans 数量，计算两者的差值来得到实际的丢弃数量。  
```go
type countExporter struct {
	*otlptrace.Exporter
	Sum *atomic.Int64
}

func (e *countExporter) ExportSpans(ctx context.Context, ss []sdktrace.ReadOnlySpan) error {
	e.Sum.Add(int64(len(ss)))
	return e.Exporter.ExportSpans(ctx, ss)
}
func (e *countExporter) Shutdown(ctx context.Context) error {
	log.Printf("shutdown countExporter %d", e.Sum.Load())
	return e.Exporter.Shutdown(ctx)
}

type countSpanProcessor struct {
	Sum *atomic.Int64
}

func (sp *countSpanProcessor) OnStart(parent context.Context, s sdktrace.ReadWriteSpan) {}
func (sp *countSpanProcessor) OnEnd(s sdktrace.ReadOnlySpan)                            { sp.Sum.Add(1) }
func (sp *countSpanProcessor) Shutdown(ctx context.Context) error {
	log.Printf("shutdown countSpanProcessor %d", sp.Sum.Load())
	return nil
}
func (sp *countSpanProcessor) ForceFlush(ctx context.Context) error { return nil }

func main() {
	// ...
	provider := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(&countExporter{exporter, new(atomic.Int64)}),
		sdktrace.WithSpanProcessor(&countSpanProcessor{new(atomic.Int64)}),
		sdktrace.WithResource(rsc),
	)
	// ...
}
```

此外在 client 端为 BatchSpanProcessor 设置 BlockOnQueueFull 即可实现禁止丢弃。  
```go
func main() {
	// ...
	provider := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter, sdktrace.WithBlocking()),
		sdktrace.WithResource(rsc),
	)
	// ...
}
```

## 拓展
* [BoundedQueue](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/pkg/queue/bounded_queue.go#L26) 在 2017 年的提交 [simplify span processor queue](https://github.com/jaegertracing/jaeger/commit/0ef81e47cacd7a5ee76fc631dd4b7e58d6e57318) 中修改了丢弃逻辑，将 `drop oldest` 简化为了 `drop newest`，同时删除了[注释](https://github.com/jaegertracing/jaeger/blob/65cff3c30823ea20d3dc48bae39d5685ae307da5/pkg/queue/bounded_queue.go#L24)中提到的 [Reaper goroutine](https://github.com/jaegertracing/jaeger/blob/008695e6d9331b9cee159912026849eb9caf0d42/pkg/queue/drop_oldest_queue.go#L62)。
* 搜索 `make(chan T, runtime.NumCPU())` 相关原因时涉及到了 goroutines 调度规则，[部分文章](https://dev.to/girishg4t/internals-of-goroutines-and-channels-397p)提到说  
  > Goroutines are cooperatively scheduled... The switch between goroutines only happens at well defined points...  

  但实际上 2020 年的 [Go 1.14 Release Notes](https://go.dev/doc/go1.14#runtime) 明确提到了  
  > Goroutines are now asynchronously preemptible. As a result, loops without function calls no longer potentially deadlock the scheduler or significantly delay garbage collection.  

  因此最新最准确的 goroutine scheduler 说明可能要去翻 runtime 相关源码了，例如 [src/runtime/HACKING.md](https://github.com/golang/go/blob/194de8fbfaf4c3ed54e1a3c1b14fc67a830b8d95/src/runtime/HACKING.md) 和 [src/runtime/proc.go](https://github.com/golang/go/blob/194de8fbfaf4c3ed54e1a3c1b14fc67a830b8d95/src/runtime/proc.go)。
