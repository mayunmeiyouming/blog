---
layout: post
title:  "Prometheus 和 Golang"
date:   2020-08-26 18:49:01 +0800
categories: Prometheus
tag: Prometheus
---

* content
{:toc}

## 安装 Prometheus

```
docker run --name=prometheus -d -p 9090:9090 --network host -v /home/huangw/桌面/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml -v /home/huangw/桌面/prometheus/rules.yml:/etc/prometheus/rules.yml prom/prometheus --config.file=/etc/prometheus/prometheus.yml --web.enable-lifecycle
```

--network host 是指定网络模式为 host

## Golang 嵌入 Prometheus metrics

```
package main

import (
	"flag"
	"log"
	"net/http"

	"github.com/go-chi/chi"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
	weavework_middleware "github.com/weaveworks/common/middleware"  // v0.0.0-20200820123129-280614068c5e
)

var (
	addr = flag.String("listen-address", ":8080", "The address to listen on for HTTP requests.")
)

var (
	// RequestDuration ...
	RequestDuration = prometheus.NewHistogramVec(prometheus.HistogramOpts{
		Name:    "request_duration_seconds",
		Help:    "The HTTP request latencies in seconds.",
		Buckets: nil,
	}, []string{"Method", "route", "code", "isWS"})
	// HTTPReqTotal ...
	HTTPReqTotal = prometheus.NewGaugeVec(prometheus.GaugeOpts{
		Name: "requests_total",
		Help: "Total number of HTTP requests made.",
	}, []string{"Method", "route"})
	// RequestBodySize ...
	RequestBodySize = prometheus.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:    "request_body_size",
			Help:    "The HTTP request latencies in seconds.",
			Buckets: nil,
		}, []string{"Method", "route"})
	// ResponseBodySize ...
	ResponseBodySize = prometheus.NewHistogramVec(prometheus.HistogramOpts{
		Name:    "response_body_size",
		Help:    "The HTTP request latencies in seconds.",
		Buckets: nil,
	}, []string{"Method", "route"})
)

func init() {
	prometheus.MustRegister(
		RequestDuration,
		HTTPReqTotal,
		RequestBodySize,
		ResponseBodySize,
	)
}

func main() {
	flag.Parse()

	router := chi.NewRouter()

	// Expose the registered metrics via HTTP.
	router.Handle("/metrics", promhttp.HandlerFor(
		prometheus.DefaultGatherer,
		promhttp.HandlerOpts{
			// Opt into OpenMetrics to support exemplars.
			EnableOpenMetrics: true,
		},
	))

	instrument := weavework_middleware.Instrument{
		Duration:         RequestDuration,
		InflightRequests: HTTPReqTotal,
		ResponseBodySize: ResponseBodySize,
		RequestBodySize:  RequestBodySize,
	}

	log.Fatal(http.ListenAndServe(*addr, instrument.Wrap(router)))
}
```

[Prometheus 中文文档](https://www.prometheus.wang/)
[玩转PROMETHEUS(1)–第1个例子](http://vearne.cc/archives/11085)

