---
title: Golang基础
description: 一些工程实践上的知识
date: 2025-11-4
tags:
  - Golang
---

## 匿名导包

在LensGateway项目中，使用了匿名导包的方式来注册请求日志中间件：

```go
import _ "LensGateway.com/internal/logging"
```

为了触发`logging`包中的`init()`函数，从而完成中间件注册工作。
