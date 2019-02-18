---
layout:     post                    # 使用的布局（不需要改）
title:      Kubernetes [1]          # 标题 
subtitle:   Kubernetes 架构设计与实现原理    #副标题
date:       2019-02-18              # 时间
author:     ZYT                     # 作者
header-img: img/kubernetes.png   #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:
    - Kubernetes                   #标签
    - 基础架构
---

# 一、背景

`Kubernetes` 的依赖于目前的两大趋势：

- 容器技术 —— 资源隔离
- 微服务架构

# 二、架构设计

![Kubernetes Architecture](/img/KubernetesArch.png)

- Master
    - `API Server` : 处理来自用户的请求
    - `Controller Manager` : 管理控制器进程
    - `Scheduler` : 调度器，为 `Pod` 选择 `Node` 节点
- Node
    - `Kubelet` : 与 `API Server` 进行通信，管理 `Pod`
    - `Kube Proxy`: 管理宿主机的子网管理
    - `Pause`: 在 `Pod` 中作为 `Linux 命名空间` 共享的基础，启用 `PID命名空间` 共享，并收集僵尸进程

# 三、详解基本对象

`Kubernetes` 中四大基本对象：

- `Pod`
- `Service`
- `Volume`
- `Namespace`

## 1、Pod

`Pod` 是 `Kubernetes` 集群中的基本单元，其中有以下基本概念：

### (1) 容器

每一个 `Kubernetes` 的 `Pod` 中存在两种不同的容器：

- `InitContainer`: 初始化配置
- `Container`: 应用容器

### (2) 卷

每一个 `Pod` 中的容器可以通过卷共享文件目录，卷能够持久化存储数据，且当 `Pod` 出现故障或者滚动更新时，卷中的数据不会清除。

### (3) 网络

同一个 `Pod` 中的多个容器共享网络栈，即多个容器可以通过 `localhost` 互相访问到彼此的端口和服务。同一个 `Pod` 中的所有容器连接到同一个网络设备，即 `pause` 容器。

### (4) 生命周期

```mermaid
    graph TD;
        A(Create) --> B(Probe(健康检查));
        B(Probe(健康检查)) --> C(Running);
        C(Running) --> D(Shutdown);
        D(Shutdown) --> E(Restart);
```

## 2、Service

由于 `Pod` 存在服务漂移的可能性，如果用 `PodIP` 对外提供服务，就很有可能随着服务漂移而无法提供服务，所以建立了 `Service` 的概念，为提供同一服务的 `Pod` 建立一个抽象。
