---
layout:     post                    # 使用的布局（不需要改）
title:      记一次 ubuntu 18.04 配置 flutter 环境的种种问题           # 标题 
subtitle:       #副标题
date:       2019-06-13              # 时间
author:     ZYT                     # 作者
header-img: img/flutter.png   #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:
    - Flutter                   #标签
    - Problem
---

# 1. 初始化 gradle 失败

因为墙，手动下载 gradle 包，并进行解压

# 2. gradle 安装依赖失败

还是墙，配置国内的源

# 3. 启动报错 Error connecting to the service protocol

虚拟机安装镜像选择 Pie image, 不要安装 Q image

# 其他问题

执行 `flutter doctor` 基本能够解决。
