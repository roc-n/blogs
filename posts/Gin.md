---
title: Gin
description: 点滴拾遗
date: 2025-11-4
tags:
  - Gin
---

## Don’t trust all proxies

使用Gin框架时，如果应用部署在反向代理（如Nginx、Traefik）后面，需要特别注意`TrustedProxies`的配置。Gin默认信任所有代理，这可能导致安全隐患，例如客户端IP伪造。对于LensGateway，可配置受信任的代理列表，不存在的话默认本机。
