---
layout:     post                    # 使用的布局（不需要改）
title:      二进制部署 Kubernetes （Ubuntu 18.04 Server）        # 标题 
subtitle:   第七节：验证集群功能     #副标题
date:       2020-08-03              # 时间
author:     ZYT                     # 作者
header-img: img/kubernetes.png   #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:
    - Kubernetes
---

# 1. 检查节点状态

```
$ kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    <none>   88m   v1.18.7-rc.0.1+73d9f50c2ba112-dirty
node1    Ready    <none>   88m   v1.18.7-rc.0.1+73d9f50c2ba112-dirty
node2    Ready    <none>   88m   v1.18.7-rc.0.1+73d9f50c2ba112-dirty
```

# 2. 创建测试文件

```
cat > nginx-ds.yml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-ds
  labels:
    app: nginx-ds
spec:
  type: NodePort
  selector:
    app: nginx-ds
  ports:
  - name: http
    port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ds
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      app: nginx-ds
  template:
    metadata:
      labels:
        app: nginx-ds
    spec:
      containers:
      - name: my-nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
EOF
```

# 3. 执行测试

```
$ kubectl create -f nginx-ds.yml
```

# 4. 检查各节点的 Pod IP 连通性

```
$ kubectl get pods -o wide -l app=nginx-ds
NAME             READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
nginx-ds-f9c6m   1/1     Running   0          64s   172.30.104.1     node2    <none>           <none>
nginx-ds-jgcgw   1/1     Running   0          64s   172.30.166.129   node1    <none>           <none>
nginx-ds-m58ln   1/1     Running   0          64s   172.30.219.66    master   <none>           <none>

# 在所有 Node 上分别 ping 上面三个 Pod IP，看是否连通：
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    ssh ${node_ip} "ping -c 1 172.30.104.1"
    ssh ${node_ip} "ping -c 1 172.30.166.129"
    ssh ${node_ip} "ping -c 1 172.30.219.66"
  done

# 输出
>>> 192.168.31.44
PING 172.30.104.1 (172.30.104.1) 56(84) bytes of data.
64 bytes from 172.30.104.1: icmp_seq=1 ttl=63 time=0.447 ms

--- 172.30.104.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.447/0.447/0.447/0.000 ms
PING 172.30.166.129 (172.30.166.129) 56(84) bytes of data.
64 bytes from 172.30.166.129: icmp_seq=1 ttl=63 time=1.29 ms

--- 172.30.166.129 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.296/1.296/1.296/0.000 ms
PING 172.30.219.66 (172.30.219.66) 56(84) bytes of data.
64 bytes from 172.30.219.66: icmp_seq=1 ttl=64 time=0.057 ms

--- 172.30.219.66 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.057/0.057/0.057/0.000 ms
>>> 192.168.31.136
PING 172.30.104.1 (172.30.104.1) 56(84) bytes of data.
64 bytes from 172.30.104.1: icmp_seq=1 ttl=63 time=0.371 ms

--- 172.30.104.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.371/0.371/0.371/0.000 ms
PING 172.30.166.129 (172.30.166.129) 56(84) bytes of data.
64 bytes from 172.30.166.129: icmp_seq=1 ttl=64 time=0.094 ms

--- 172.30.166.129 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.094/0.094/0.094/0.000 ms
PING 172.30.219.66 (172.30.219.66) 56(84) bytes of data.
64 bytes from 172.30.219.66: icmp_seq=1 ttl=63 time=0.356 ms

--- 172.30.219.66 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.356/0.356/0.356/0.000 ms
>>> 192.168.31.90
PING 172.30.104.1 (172.30.104.1) 56(84) bytes of data.
64 bytes from 172.30.104.1: icmp_seq=1 ttl=64 time=0.097 ms

--- 172.30.104.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.097/0.097/0.097/0.000 ms
PING 172.30.166.129 (172.30.166.129) 56(84) bytes of data.
64 bytes from 172.30.166.129: icmp_seq=1 ttl=63 time=1.13 ms

--- 172.30.166.129 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.134/1.134/1.134/0.000 ms
PING 172.30.219.66 (172.30.219.66) 56(84) bytes of data.
64 bytes from 172.30.219.66: icmp_seq=1 ttl=63 time=0.502 ms

--- 172.30.219.66 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.502/0.502/0.502/0.000 ms
```

# 5. 检查服务 IP 和端口可达性

```
$ kubectl get svc -l app=nginx-ds
NAME       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx-ds   NodePort   10.254.135.98   <none>        80:30708/TCP   5m42s
```

- Service Cluster IP：10.254.135.98
- 服务端口：80
- NodePort 端口：30708

在所有 Node 上 curl Service IP：

```
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    ssh ${node_ip} "curl -s 10.254.135.98"
  done

# 输出
>>> 192.168.31.44
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
>>> 192.168.31.136
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
>>> 192.168.31.90
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

# 6. 检查服务的 NodePort 可达性

```
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    ssh ${node_ip} "curl -s ${node_ip}:30708"
  done

# 输出
>>> 192.168.31.44
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
>>> 192.168.31.136
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
>>> 192.168.31.90
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```