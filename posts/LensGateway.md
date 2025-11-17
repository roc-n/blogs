---
title: LensGateway
description: API网关一些思考
date: 2025-11-4
tags:
  - API网关
---

## 请求日志

### 请求日志ID选型

考虑API网关中“请求日志ID”的核心需求：

- 唯一：能区分每次请求
- 高性能：生成ID的性能高，不会成为系统瓶颈
- 易于实现：不引入额外服务或复杂配置管理节点ID

采用v4版本的uuid作为请求日志的唯一id，这里不选择snowflake的原因是，snowflake的“趋势递增”特性非必需，日志本身带有时间戳，同时引入节点ID会增加复杂度。

### 为什么请求日志中间件中请求ID要在c.Next()前置入http header？

首先需要明确一点，“下游”需要知道请求ID，这里包括下游中间件（请求日志中间件处于中间件链第一个位置）与后端服务节点。原因主要在于**链路追踪**，后端所有服务节点知晓了请求ID后，可以将该请求ID附加在自己的日志中，从而实现跨服务的日志关联：

> - [LensGateway]  reqID="abc-123", 收到请求 /orders
> - [LensGateway]  reqID="abc-123", 鉴权成功
> - [订单服务]      reqID="abc-123", 开始创建订单
> - [订单服务]      reqID="abc-123", 调用用户服务
> - [用户服务]      reqID="abc-123", 查询用户信息
> - [用户服务]      reqID="abc-123", 查询成功
> - [订单服务]      reqID="abc-123", 调用库存服务失败: timeout
> - [LensGateway]  reqID="abc-123", 收到上游 504 错误, 返回客户端

基于reqID，可以将上述日志串联起来，快速定位问题。

### 系统资源瓶颈时，主动丢弃日志

logChan已满的情况下，一个次要功能（日志）的性能问题，导致核心功能（流量转发）的完全瘫痪。这是典型的“雪崩效应”。为此，设计上需要考虑在系统资源瓶颈时，主动丢弃日志，保证核心功能的可用。具体而言，有若干方案：

#### 监控“丢弃”本身

记录“丢弃”事件，暴露Prometheus指标，方便运维监控。

#### 优先级别队列与采样

- 错误日志优先：设计俩Channel，一个errorLogChan，一个infoLogChan，缓冲区告急时，优先从infoLogChan开始丢弃，尽最大努力保留error日志。
- 动态采样：logChan 的容量超过80%时，系统自动进入“降级模式”，开始对info级日志进行采样（比如只记录10%），但仍然 100% 记录error日志。

#### 磁盘缓冲

当缓冲区满了后，不直接丢弃，而是将其暂存到本地磁盘上的临时文件，一个独立的线程会稍后尝试从这个磁盘缓冲区读取数据，并重新发送。但是这会引入磁盘 I/O，增加网关本身复杂性和潜在故障点（比如磁盘满了）。
针对这点更好的思路是让网关只负责快速地把日志吐到标准输出 (stdout)，然后由一个专门的**Log Shipper**进程去负责收集、磁盘缓冲和可靠发送。

为了精简，考虑采用监控“丢弃”本身，后续引入Prometheus指标监控日志丢弃率。

## 负载均衡

### 后端服务节点健康检查

在验证p2c负载均衡算法的健康检查时，出现并发问题：

```go
2025/11/07 14:43:50 Site unreachable, remove localhost:8082 from load balancer.
2025/11/07 14:43:50 Site unreachable, remove localhost:8081 from load balancer.
2025/11/07 14:43:50 Site unreachable, remove localhost:8081 from load balancer.
2025/11/07 14:43:50 Site unreachable, remove localhost:9091 from load balancer.
```

检查发现nodes列表出现**重复节点**，而执行检查的代码片段本身并没有做**并发控制**：

```go
prev := as.ReadAlive(host)
if !alive && prev {
  log.Printf("Site unreachable, remove %s from load balancer.", host)
  as.SetAlive(host, false)
  bb.Remove(n)
}
```

这里俩goroutine读取同一节点存活状态，在其中一个goroutine打印信息后SetAlive()之前，另一个goroutine的if语句判定为true从而执行打印，重复删除同一节点。

## CORS

目前主流的前后端分离架构下，前端应用（SPA）通常部署在与后端API服务不同的域名或端口下，引发跨域请求问题。为了保障安全性，浏览器默认会阻止跨域请求，除非服务器明确允许，因此在API网关中设计CORS中间件进行处理。

## 多实例若干问题

API网关假如也是多个实例部署在多台物理机上，就意味着需要在API网关的前面还需要加一层负载均衡器。

### L4LB/L7LB

L4LB作为入口负载均衡器，工作在传输层（第四层），基于*IP地址*和*端口*进行流量分发。L4LB的优点是性能高，延迟低，适合处理大量并发连接，但缺乏对HTTP协议的深入理解，无法进行基于内容的路由。

L7LB工作在应用层（第七层），能够理解HTTP协议，支持基于URL路径、HTTP头等内容进行路由。L7LB可以实现更复杂的业务路由规则和流量管理，但相对来说性能较低，延迟较高。当前的LensGateway实例，可以作为L7LB使用，部署在L4LB之后。

L4+L7架构下，L4LB处理南北流量，L7LB处理东西流量。
L7+L7的架构也比较普遍，前者可以粗粒度地处理安全、全局路由、SSL卸载等，后者可以做细粒度的认证、鉴权、限流、服务发现等。

> 本地高可用：集群化负载均衡器，主备模式（Active-Passive）或主主模式（Active-Active），前者共享VIP地址，后者基于复杂的路由技术分发到集群内所有活动节点
> 跨数据中心高可用：DNS负载均衡，基于地理位置、延迟等因素将流量分发到不同数据中心的负载均衡器
