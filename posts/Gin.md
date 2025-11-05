---
title: Gin
description: 点滴拾遗
date: 2025-11-4
tags:
  - Gin
---

## Don’t trust all proxies

使用Gin框架时，如果应用部署在反向代理（如Nginx、Traefik）后面，需要特别注意`TrustedProxies`的配置。Gin默认信任所有代理，这可能导致安全隐患，例如客户端IP伪造。对于LensGateway，可配置受信任的代理列表，不存在的话默认本机。

## 注册路由时会固定当前调用链

Gin的全局中间件在注册路由时会固定当前调用链。如果在注册路由后添加新的全局中间件，这些中间件不会应用到已经注册的路由上。对于LensGateway而言，将healthz/路由注册在最前面，确保其不受后续中间件影响。

注册路由时（router.Get(...)），Gin会将全局handlers列表拷贝到路由处理链中。
