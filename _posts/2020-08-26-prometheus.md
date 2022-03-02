---
layout: post
title:  "Prometheus 和 Golang"
date:   2020-08-26 18:49:01 +0800
categories: [Tech]
tag: 
  - Prometheus
---

## 一、安装 Prometheus

```bash
docker run --name=prometheus -d -p 9090:9090 --network host \
-v /home/huangw/桌面/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
-v /home/huangw/桌面/prometheus/rules.yml:/etc/prometheus/rules.yml prom/prometheus \
--config.file=/etc/prometheus/prometheus.yml --web.enable-lifecycle

--network host 是指定网络模式为 host

--web.enable-lifecycle 开启配置热加载
```

prometheus.yml

```yml
global:
  scrape_interval:     5s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 5s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).
  scrape_timeout: 5s

# Alertmanager 的配置，如果没有配置它，可以去掉这个
alerting:
  alertmanagers:
    - static_configs:
      - targets: ['192.168.89.30:9093']

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  - /etc/prometheus/rules.yml

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  # 采集node exporter监控数据
  - job_name: 'node'
    metrics_path: /metrics
    static_configs:
      - targets: ['192.168.89.30:9100']
  - job_name: 'goTest'
    metrics_path: /metrics
    static_configs:
      - targets: ['192.168.89.30:8080']
  # 采集 consul 监控数据
  - job_name: 'consul-prometheus'
    consul_sd_configs:
    - server: '172.23.0.10:8500'
      services: []

```

rules.yml

```yml
groups:
  - name: quick_search
    rules:
      - alert: QPS
        expr: sum(irate(quick_search_request_duration_seconds_count[1m])) > 5000
        for: 1m
        labels:
          status: warning
        annotations:
          summary: "QPS过高"
          description: "QPS过高"
      - alert: 时延
        expr: quick_search_request_duration_seconds_sum / quick_search_request_duration_seconds_count > 1
        for: 1m
        labels:
          status: warning
        annotations:
          summary: "时延过高"
          description: "时延过高"
      - alert: 错误率
        expr: (quick_search_request_duration_seconds_count - quick_search_request_duration_seconds_count{code="200"}) / quick_search_request_duration_seconds_count > 0.1
        for: 1m
        labels:
          status: warning
        annotations:
          summary: "错误率过高"
          description: "错误率过高"

```

## 二、Golang 嵌入 Prometheus metrics

```go
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

## 三、安装 AlterManager

```bash
docker run -d -p 9093:9093 \
--name alertmanager \
-v /home/huangw/桌面/alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml \
prom/alertmanager
```

alertmanager.yml

```yml
global:
  resolve_timeout: 5m
  smtp_smarthost: smtp.qq.com:465
  smtp_from: email-address
  smtp_auth_username: username
  smtp_auth_identity: username
  smtp_auth_password: password
  smtp_require_tls: false

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'email'
receivers:
- name: 'email'
  email_configs:
    - to: 'email-address'
      send_resolved: true
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']

```

## 参考

1. [Prometheus 中文文档](https://www.prometheus.wang/)
2. [玩转PROMETHEUS(1)–第1个例子](http://vearne.cc/archives/11085)
3. [从零搭建Prometheus监控报警系统](https://www.cnblogs.com/chenqionghe/p/10494868.html)
