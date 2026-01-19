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

## 一些坑点

### slice

- slice扩容机制导致的“浅拷贝”问题

```go
a := []int{1, 2, 3, 4, 5}
b := a[1:2]
```

由于slice由len、cap和底层ptr组成，没有扩容时，修改b中元素，a也会跟着一起变化，append操作扩容后底层数组会被重新分配，导致原有slice和新slice指向不同的底层数组，从而互不影响。

- slice作为函数参数传递是值传递

```go
func modifySlice(s []int) {
    s[0] = 100   // 影响原slice
    s = []int{1} // 不影响原slice
} 
```

传递slice时传递的是slice结构体的副本，修改元素会影响原slice，但重新赋值不会影响原slice。此外，在函数中append操作会导致底层数组重新分配，但并不影响原slice。
如希望函数能够扩展外部slice，一般返回新的slice或传递slice指针。

### map

- 并发读写导致panic

一般map在并发场景下需要加锁保护，或者使用sync.Map。

- map遍历顺序不确定

map的遍历顺序是随机的，每次遍历的顺序可能不同。如果非要按某种规则遍历map，需要先将key提取到slice中排序，然后按排序后的顺序访问map。

- map的value无法取地址

```go
type Person struct{ Age int }
m := map[string]Person{"alice": {Age: 20}}
// m["alice"].Age = 21 // ❌ cannot assign to struct field in map
```

Golang中只有可寻址的值才能取地址，而map的value不可寻址的，在编译器层面禁止取地址操作。根本原因是map的内部实现是动态的，不稳定的。map元素增加导致bucket桶重新分配，同一个key的value地址可能会变化，原来的地址可能指向无效或错误的内存。

相较之下，slice的底层数组在扩容前是稳定的，因此slice元素可寻址的，可以取地址。扩容后，原有slice元素地址依然有效，不会立即被GC回收。
