---
title: go-zero
description: 点滴拾遗
date: 2026-01-22
tags:
  - Golang Web Framework
---

## goctl model

`goctl`生成的Model代码只为主键查询和唯一索引查询自动添加缓存，实现了标准的`Cache Aside`模式, 类似于Write-Invalidate策略。

### 查询模式

辅以唯一索引查询优化：
> 为了避免数据存两份（一份Key是ID，一份Key是 Phone），go-zero做了巧妙的优化：
> cache:user:phone:13800000000这个Key里面存的不是用户数据，而是主键ID。
> 查询流程:

1) 先查 cache:user:phone:... 拿到ID。
2) 再用这个ID查cache:user:id:... 拿到真正的用户数据。
3) 如果第一次查 Phone，它会先查DB拿到完整行，然后同时设置Phone->ID 的映射缓存和ID->Data的数据缓存。

### 删除/更新模式

先DB后cache：事务提交后，再删除对应cache。这里与延迟双删策略不同，go-zero选择了更简单直接的写后删策略。

### 缓存雪崩/穿透

默认过期时间是7天，

1) 为避免雪崩，在此基础上随机增加一个偏差值。
2) 为避免穿透，查询不到数据时缓存空结果，过期时间很短。
