---
title: Linux
description: Linux相关
date: 2025-11-10
tags:
  - Linux
---

## 常用命令

### 查找进程pid

```bash
pgrep -f  <full process name>        
```

如 pgrep -f "./dist/lensgateway"

### 查找端口被哪个进程占用

```bash
lsof -i :<port>
```

### kill

```bash
kill <pid>
```

kill默认发送SIGTERM信号，还可以`-HUP`或`-INT`来向指定进程发送SIGHUP或SIGINT信号

## 机制

### 包装器命令

`minikube kubectl -- <kubectl command>`这里的--作为选项结束的标志，后续所有参数都传递给kubectl命令。
