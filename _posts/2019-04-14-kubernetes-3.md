---
layout:     post                    # 使用的布局（不需要改）
title:      Kubernetes [3]         # 标题 
subtitle:   Networking   #副标题
date:       2019-04-14              # 时间
author:     ZYT                     # 作者
header-img: img/kubernetes.png   #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:
    - Kubernetes                   #标签
    - 基础架构
---

# 一、Kubernetes Networking Model

`Kubernetes` 网络模型有一个非常重要的设计哲学：每一个 `Pod` 都有唯一的一个 `IP` 地址。`Pod` 中的所有容器都共享这个 `IP` 地址。由 `Pod` 中运行的 `Pause` 容器提供。每个容器的启动和销毁都会由 `Pause` 提供网络。

`Node` 内部通信：

![Node Networking](/img/dockerNetwork.gif)

多个 `Kubernetes` 节点之间的通信：

![Docker Networking](/img/kubernetesNet.gif)
