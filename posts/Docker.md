---
title: Docker
description: Docker容器技术的一些细节
date: 2025-11-22
tags:
  - 虚拟化技术
---

## Linux命名空间（Namespaces）

Namespace是Linux内核提供的一种轻量级虚拟化技术，用于隔离进程的资源视图。默认情况下，整个Linux系统只有一个全局Namespace，所有进程共享同一资源视图：文件系统挂载点、网络设备列表、进程ID、主机名、用户ID空间等。
Namespcae类型：

- Mount(mnt)：文件系统挂载点
- PID(pid)：进程ID
- Network(net)：网络接口、IP地址、路由表
- User命名空间(user)：用户和组ID
- IPC：进程间通信资源
- UTS：主机名和域名

Namespace提供**视图隔离**，Cgroups提供**资源限制**。

## Docekr镜像层次

Docker里镜像不是一个单一整体文件，而是由多个只读的层叠加而成。

```Dockerfile
FROM ubuntu:22.04                               # Layer 1: 基础操作系统
RUN apt-get update && apt-get install -y curl   # Layer 2: 安装 curl
COPY app.py /app/                               # Layer 3: 复制应用代码
CMD ["python3", "/app/app.py"]                  # Layer 4: 启动命令（元数据层）
```

最终镜像=Layer 1 + Layer 2 + Layer 3 + Layer 4，相同的Layer的ID相同，节省空间。

```bash
docker history <image_name>
```

## 容器技术&虚拟机技术的可移植性

Docker镜像不能跨架构直接运行，Docker容器共享宿主机内核，因此要求宿主机和容器使用相同架构的CPU指令集。虚拟机可利用QEMU等模拟器实现跨架构运行，但性能较差。
当前真正解决Docker跨架构运行的方案是**多架构镜像（Multi-Arch Image）**，通过在同一镜像标签下打包多个架构的镜像版本，实现自动选择合适版本运行。通过`docker buildx`命令加以构建。
