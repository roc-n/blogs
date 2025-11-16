---
title: Prometheus
description: 相关概念理解与细节
date: 2025-11-16
tags:
  - 可观测技术
---

## 四种基本类型

### Counter

计数器，一个累积量，只能递增或重置为0，类似里程计
典型应用：
- http请求数
- 任务完成数
- 错误总数
PromQL用法：`rate(http_requests_total[5m])`表示过去5分钟内每秒的http请求数

### Gauge

仪表盘，一个能够任意上升或下降的数值，类似速度表
典型应用：
- 当前活跃的连接数
- 当前内存使用量
- CPU温度
PromQL用法：配合`delta()/predict_linear()`表示一段时间内的变化/预测未来值

###  Histogram

直方图，采样观测值，通过可配置的`bucket`来进行计数，桶里的数值代表区间的上界
典型应用：
- http请求延迟
PromQL用法：
- `http_request_duration_seconds_bucket{le="0.005"}`，请求延迟小于等于0.005s的总数
- `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))`，过去5分钟内请求延迟的95%分位数（p95延迟），代表过去5分钟95%的请求的延迟小于等于计算出的值。

### Summary

待补充

## prometheus/client_golang

### Vector

Vec在client_golang的上下文中指的是某一基本类型的容器/集合，里面通过自定义的`label`将同一基本类型划分为多个维度，方便整合从不同维度监测同一种对象。

通过`.WithLabelValues(...).Inc()`改变监测值时，内部代码通过`懒加载`+`缓存机制`高效运作。