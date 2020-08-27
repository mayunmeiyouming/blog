---
layout: post
title:  "Prometheus 检测性能"
date:   2020-08-26 18:49:01 +0800
categories: Prometheus
tag: Prometheus
---

* content
{:toc}

## 安装 Prometheus

```
docker run --name=prometheus -d -p 9090:9090 -v /home/chenqionghe/promethues/server/prometheus.yml:/etc/prometheus/prometheus.yml -v /home/chenqionghe/promethues/server/rules.yml:/etc/prometheus/rules.yml prom/prometheus --config.file=/etc/prometheus/prometheus.yml --web.enable-lifecycle
```

[Prometheus 中文文档](https://www.prometheus.wang/)
[玩转PROMETHEUS(1)–第1个例子](http://vearne.cc/archives/11085)

