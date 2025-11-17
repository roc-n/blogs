---
title: CORS&CSRF
description: 相关概念理解与细节
date: 2025-11-17
tags:
  - Web开发
---

## CORS

### 定义

跨源资源共享（Cross-Origin Resource Sharing，CORS）是一种机制，使用额外的HTTP头来告诉浏览器，允许哪些源（域）访问资源，重点在于谁能读取资源。

三要素:

- 服务器：标识允许访问资源的源
- HTTP头：`Access-Control-Allow-Origin:*` 及 `Origin: http://example.com等`
- 浏览器：实现CORS相关规范，是否载入资源由浏览器控制

### 工作机制

- 简单请求：直接发送请求，根据响应头中的Access-Control-Allow-Origin决定是否允许访问
- 复杂请求：浏览器会先发送一个OPTIONS预检请求，询问服务器是否允许实际请求

一般而言，简单请求定义为`GET,HEAD,POST`，且请求头属于`Accept,Accept-Language,Content-Language,Content-Type,Range`且Content-Type值为`application/x-www-form-urlencoded,multipart/form-data,text/plain`的请求。

后端服务器原则上将轻量请求归为简单请求，复杂请求则需要进行预检，降低负载。

### 为什么需要CORS

1. **节省资源**，有效减轻服务器负载
2. **保护隐私**，限制敏感数据访问
3. **安全考虑**，防止盗取用户数据及部分CSRF攻击，准确而言是防范*借助用户登录态的第三方JS发起的跨源数据窃取*，这里特别指出，CORS并不是为了防止CSRF攻击而设计的，只是风险站点和目标站点非同源，适用于CORS机制，但是简单POST请求发出后响应被浏览器拦截没用，真正的后端逻辑已经执行了。参考[CSRF&CORS](https://juejin.cn/post/7504926961628282899)

### 如何解决CORS问题

1. 跨域有问题,那就不要跨域,在同一个域名下发起请求，具体而言，可以通过反向代理（如Nginx）将前端请求转发到后端服务。
2. 按照CORS规范，在服务器端设置适当的HTTP头，允许特定源访问资源。例如，设置`Access-Control-Allow-Origin`头应对简单请求，复杂请求还需设置`Access-Control-Allow-Methods`和`Access-Control-Allow-Headers`等。
3. JSONP（JSON with Padding）：通过动态创建`<script>`标签来实现跨域请求，适用于GET请求，但存在安全隐患。

## CSRF

待补充
