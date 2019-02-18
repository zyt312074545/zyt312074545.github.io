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

