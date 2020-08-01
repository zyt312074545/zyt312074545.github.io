---
layout:     post                    # 使用的布局（不需要改）
title:      二进制部署 Kubernetes （Ubuntu 18.04 Server）        # 标题 
subtitle:   第四节：部署 etcd 集群     #副标题
date:       2020-07-30              # 时间
author:     ZYT                     # 作者
header-img: img/kubernetes.png   #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:
    - Kubernetes
---

本文档介绍部署一个三节点高可用 etcd 集群的步骤：

- 构建和分发 etcd 二进制文件；
- 创建 etcd 集群各节点的 x509 证书，用于加密客户端(如 etcdctl) 与 etcd 集群、etcd 集群之间的通信；
- 创建 etcd 的 systemd unit 文件，配置服务参数；
- 检查集群工作状态；

## 1. 构建 etcd/etcdctl 二进制文件

```
$ git clone https://github.com/etcd-io/etcd.git
$ git checkout origin/release-3.4
$ git checkout release-3.4

$ go mod vendor
$ make build
$ ll bin/
```

分发：

```
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
    do
        echo ">>> ${node_ip}"
        scp etcd* root@${node_ip}:/opt/k8s/bin
        ssh root@${node_ip} "chmod +x /opt/k8s/bin/*"
    done
```

## 2. 创建 etcd 证书和私钥

创建证书签名请求：
```
cd /opt/k8s/work
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "192.168.31.44",
    "192.168.31.136",
    "192.168.31.90"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "opsnull"
    }
  ]
}
EOF
```

- `hosts`：指定授权使用该证书的 etcd 节点 IP 列表，需要将 etcd 集群所有节点 IP 都列在其中；

生成证书和私钥：

```
cd /opt/k8s/work
cfssl gencert -ca=/opt/k8s/work/ca.pem \
    -ca-key=/opt/k8s/work/ca-key.pem \
    -config=/opt/k8s/work/ca-config.json \
    -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
ls etcd*pem
```

分发文件：

```
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
    do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "mkdir -p /etc/etcd/cert"
        scp etcd*.pem root@${node_ip}:/etc/etcd/cert/
    done
```

## 3. 创建 etcd 的 systemd unit 模板文件

```
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
cat > etcd.service.template <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=${ETCD_DATA_DIR}
ExecStart=/opt/k8s/bin/etcd \\
  --data-dir=${ETCD_DATA_DIR} \\
  --wal-dir=${ETCD_WAL_DIR} \\
  --name=##NODE_NAME## \\
  --cert-file=/etc/etcd/cert/etcd.pem \\
  --key-file=/etc/etcd/cert/etcd-key.pem \\
  --trusted-ca-file=/etc/kubernetes/cert/ca.pem \\
  --peer-cert-file=/etc/etcd/cert/etcd.pem \\
  --peer-key-file=/etc/etcd/cert/etcd-key.pem \\
  --peer-trusted-ca-file=/etc/kubernetes/cert/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --listen-peer-urls=https://##NODE_IP##:2380 \\
  --initial-advertise-peer-urls=https://##NODE_IP##:2380 \\
  --listen-client-urls=https://##NODE_IP##:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls=https://##NODE_IP##:2379 \\
  --initial-cluster-token=etcd-cluster-0 \\
  --initial-cluster=${ETCD_NODES} \\
  --initial-cluster-state=new \\
  --auto-compaction-mode=periodic \\
  --auto-compaction-retention=1 \\
  --max-request-bytes=33554432 \\
  --quota-backend-bytes=6442450944 \\
  --heartbeat-interval=250 \\
  --election-timeout=2000
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

- `WorkingDirectory`、`--data-dir`：指定工作目录和数据目录为 `${ETCD_DATA_DIR}`，需在启动服务前创建这个目录；
- `--wal-dir`：指定 wal 目录，为了提高性能，一般使用 SSD 或者和 `--data-dir` 不同的磁盘；
- `--name`：指定节点名称，当 `--initial-cluster-state` 值为 new 时，`--name` 的参数值必须位于 `--initial-cluster` 列表中；
- `--cert-file`、`--key-file`：etcd server 与 client 通信时使用的证书和私钥；
- `--trusted-ca-file`：签名 client 证书的 CA 证书，用于验证 client 证书；
- `--peer-cert-file`、`--peer-key-file`：etcd 与 peer 通信使用的证书和私钥；
- `--peer-trusted-ca-file`：签名 peer 证书的 CA 证书，用于验证 peer 证书；

## 4. 为各节点创建和分发 etcd systemd unit 文件

替换模板文件中的变量，为各节点创建 systemd unit 文件：

```
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for (( i=0; i < 3; i++ ))
    do
        sed -e "s/##NODE_NAME##/${NODE_NAMES[i]}/" -e "s/##NODE_IP##/${NODE_IPS[i]}/" etcd.service.template > etcd-${NODE_IPS[i]}.service 
    done
ls *.service
```

分发生成的 systemd unit 文件：

```
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
    do
        echo ">>> ${node_ip}"
        scp etcd-${node_ip}.service root@${node_ip}:/etc/systemd/system/etcd.service
    done
```

## 5. 启动 etcd 服务

```
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
    do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "mkdir -p ${ETCD_DATA_DIR} ${ETCD_WAL_DIR}"
        ssh root@${node_ip} "systemctl daemon-reload && systemctl enable etcd && systemctl restart etcd " &
    done
```

- 必须先创建 etcd 数据目录和工作目录;
- etcd 进程首次启动时会等待其它节点的 etcd 加入集群，命令 `systemctl start etcd` 会卡住一段时间，为正常现象；

## 6. 检查启动结果

```
for node_ip in ${NODE_IPS[@]}
    do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "systemctl status etcd|grep Active"
    done
```

确保状态为 `active (running)`，否则查看日志，确认原因：

```
# 查看错误日志
journalctl -u etcd

# 可能遇到的问题
cannot access data directory: directory "/data/k8s/etcd/data/","drwxr-xr-x" exist without desired file permission "-rwx------".

$ cd /data/k8s/etcd/
$ chmod 700 data
$ chmod 700 wal
```

## 7. 验证服务状态

部署完 etcd 集群后，在任一 etcd 节点上执行如下命令：

```
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
    do
        echo ">>> ${node_ip}"
        /opt/k8s/bin/etcdctl \
        --endpoints=https://${node_ip}:2379 \
        --cacert=/etc/kubernetes/cert/ca.pem \
        --cert=/etc/etcd/cert/etcd.pem \
        --key=/etc/etcd/cert/etcd-key.pem endpoint health
    done
```

预期输出：

```
>>> 192.168.31.44
https://192.168.31.44:2379 is healthy: successfully committed proposal: took = 13.562105ms
>>> 192.168.31.136
https://192.168.31.136:2379 is healthy: successfully committed proposal: took = 8.474169ms
>>> 192.168.31.90
https://192.168.31.90:2379 is healthy: successfully committed proposal: took = 9.019993ms
```

## 8. 查看当前的 leader

```
source /opt/k8s/bin/environment.sh
/opt/k8s/bin/etcdctl \
  -w table --cacert=/etc/kubernetes/cert/ca.pem \
  --cert=/etc/etcd/cert/etcd.pem \
  --key=/etc/etcd/cert/etcd-key.pem \
  --endpoints=${ETCD_ENDPOINTS} endpoint status
```

输出：

```
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|          ENDPOINT           |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://192.168.31.44:2379 | 3c0c4e2d324c0b39 |  3.4.10 |   20 kB |      true |      false |        40 |          9 |                  9 |        |
| https://192.168.31.136:2379 | 43f60f474f47b514 |  3.4.10 |   20 kB |     false |      false |        40 |          9 |                  9 |        |
| https://192.168.31.90:2379 | f9ff31e8ce518c24 |  3.4.10 |   20 kB |     false |      false |        40 |          9 |                  9 |        |
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```