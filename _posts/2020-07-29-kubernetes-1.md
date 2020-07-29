---
layout:     post                    # 使用的布局（不需要改）
title:      二进制部署 Kubernetes （Ubuntu 18.04 Server）        # 标题 
subtitle:   第一节：初始化系统和全局变量     #副标题
date:       2020-07-29              # 时间
author:     ZYT                     # 作者
header-img: img/Kubernetes.png   #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:
    - Kubernetes
---

## 一、集群规划

- master: 192.168.31.193
- node1: 192.168.31.148
- node2: 192.168.31.121

每台机器的 `/etc/hosts` 文件中添加主机名和 IP 的对应关系：

```
cat >> /etc/hosts <<EOF
192.168.31.193 master
192.168.31.148 node1
192.168.31.121 node2
EOF
```

## 二、添加节点信任关系

本操作只需要在 master 节点上进行，设置 root 账户可以无密码登录所有节点：


```
ubuntu 服务开启 ssh 登录权限：
vim /etc/ssh/sshd_config
PermitRootLogin yes

重启 ssh 服务：
systemctl restart sshd

配置无密码登录方式：
ssh-keygen -t rsa 
ssh-copy-id root@master
ssh-copy-id root@node1
ssh-copy-id root@node2
```

## 三、更新 PATH 变量

```
echo 'PATH=/opt/k8s/bin:$PATH' >>/root/.bashrc; source /root/.bashrc
```

- `/opt/k8s/bin` 目录保存安装的程序

## 四、安装依赖包

```
apt install -y chrony conntrack ipvsadm ipset jq iptables curl sysstat wget socat git libseccomp-dev selinux-utils
```

- kube-proxy 使用 ipvs 模式，ipvsadm 为 ipvs 的管理工具；
- chrony 用于系统时间同步；

## 五、关闭防火墙

关闭防火墙，清理防火墙规则，设置默认转发策略：

```
ufw disable

# --flush   -F [chain]		Delete all rules in  chain or all chains
# --delete-chain -X [chain]		Delete a user-defined chain
iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat
iptables -P FORWARD ACCEPT
```

## 六、关闭 swap 分区

kubelet 启动检查 swap 是否开启(可以设置 kubelet 启动参数 --fail-swap-on 为 false 关闭 swap 检查)：

```
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## 七、关闭 SELinux

关闭 SELinux，否则 kubelet 挂载目录时可能报错（Permission denied）

```
getenforce
setenforce 0
```

## 八、优化内核参数

```
cat > kubernetes.conf <<EOF
net.ipv4.ip_forward=1
net.ipv4.neigh.default.gc_thresh1=4096
vm.swappiness=0
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF

cp kubernetes.conf /etc/sysctl.d/kubernetes.conf; sysctl -p /etc/sysctl.d/kubernetes.conf
```

## 九、设置系统时区

```
timedatectl set-timezone Asia/Shanghai

# 重启依赖于系统时间的服务
systemctl restart rsyslog
```

## 十、创建相关目录

```
mkdir -p /opt/k8s/{bin,work} /etc/{kubernetes,etcd}/cert
```

## 十一、分发集群配置参数脚本

environment.sh 内容，根据自己机器情况进行修改：

```
#!/usr/bin/bash

# 生成 EncryptionConfig 所需的加密 key
export ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

# 集群各机器 IP 数组
export NODE_IPS=(192.168.31.193 192.168.31.148 192.168.31.121)

# 集群各 IP 对应的主机名数组
export NODE_NAMES=(master node1 node2)

# etcd 集群服务地址列表
export ETCD_ENDPOINTS="https://192.168.31.193:2379,https://192.168.31.148:2379,https://192.168.31.121:2379"

# etcd 集群间通信的 IP 和端口
export ETCD_NODES="master=https://192.168.31.193:2380,node1=https://192.168.31.148:2380,node2=https://192.168.31.121:2380"

# kube-apiserver 的反向代理(kube-nginx)地址端口
export KUBE_APISERVER="https://127.0.0.1:8443"

# 节点间互联网络接口名称
export IFACE="enp0s3"

# etcd 数据目录
export ETCD_DATA_DIR="/data/k8s/etcd/data"

# etcd WAL 目录，建议是 SSD 磁盘分区，或者和 ETCD_DATA_DIR 不同的磁盘分区
export ETCD_WAL_DIR="/data/k8s/etcd/wal"

# k8s 各组件数据目录
export K8S_DIR="/data/k8s/k8s"

## DOCKER_DIR 和 CONTAINERD_DIR 二选一
# docker 数据目录
export DOCKER_DIR="/data/k8s/docker"

# containerd 数据目录
export CONTAINERD_DIR="/data/k8s/containerd"

## 以下参数一般不需要修改

# TLS Bootstrapping 使用的 Token，可以使用命令 head -c 16 /dev/urandom | od -An -t x | tr -d ' ' 生成
BOOTSTRAP_TOKEN="42814b4543bdd8705b7c403c5d7ee9cc"

# 最好使用 当前未用的网段 来定义服务网段和 Pod 网段

# 服务网段，部署前路由不可达，部署后集群内路由可达(kube-proxy 保证)
SERVICE_CIDR="10.254.0.0/16"

# Pod 网段，建议 /16 段地址，部署前路由不可达，部署后集群内路由可达(flanneld 保证)
CLUSTER_CIDR="172.30.0.0/16"

# 服务端口范围 (NodePort Range)
export NODE_PORT_RANGE="30000-32767"

# kubernetes 服务 IP (一般是 SERVICE_CIDR 中第一个IP)
export CLUSTER_KUBERNETES_SVC_IP="10.254.0.1"

# 集群 DNS 服务 IP (从 SERVICE_CIDR 中预分配)
export CLUSTER_DNS_SVC_IP="10.254.0.2"

# 集群 DNS 域名（末尾不带点号）
export CLUSTER_DNS_DOMAIN="cluster.local"

# 将二进制目录 /opt/k8s/bin 加到 PATH 中
export PATH=/opt/k8s/bin:$PATH
```

将 environment.sh 拷贝到所有节点：

```
source environment.sh # 先修改
for node_ip in ${NODE_IPS[@]}
    do
        echo ">>> ${node_ip}"
        scp environment.sh root@${node_ip}:/opt/k8s/bin/
        ssh root@${node_ip} "chmod +x /opt/k8s/bin/*"
    done
```
