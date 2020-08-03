---
layout:     post                    # 使用的布局（不需要改）
title:      二进制部署 Kubernetes （Ubuntu 18.04 Server）        # 标题 
subtitle:   第八节：部署集群插件     #副标题
date:       2020-08-03              # 时间
author:     ZYT                     # 作者
header-img: img/kubernetes.png   #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:
    - Kubernetes
---

# 1. 部署 coredns 插件

## (1) 下载和配置 coredns

```
git clone https://github.com/coredns/deployment.git
mv deployment coredns-deployment
```

## (2) 创建 coredns

```
cd /opt/k8s/work/coredns-deployment/kubernetes
source /opt/k8s/bin/environment.sh
./deploy.sh -i ${CLUSTER_DNS_SVC_IP} -d ${CLUSTER_DNS_DOMAIN} | kubectl apply -f -
```

## (3) 检查 coredns 功能

```
$ kubectl get all -n kube-system -l k8s-app=kube-dns
NAME                           READY   STATUS    RESTARTS   AGE
pod/coredns-85b4878f78-62nt9   1/1     Running   0          33s

NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
service/kube-dns   ClusterIP   10.254.0.2   <none>        53/UDP,53/TCP,9153/TCP   33s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coredns   1/1     1            1           33s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/coredns-85b4878f78   1         1         1       33s
```

新建一个 Deployment：

```
cat > my-nginx.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
EOF

kubectl create -f my-nginx.yaml
```

export 该 Deployment, 生成 my-nginx 服务：

```
$ kubectl expose deploy my-nginx
service/my-nginx exposed

$ kubectl get services my-nginx -o wide
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE   SELECTOR
my-nginx   ClusterIP   10.254.39.135   <none>        80/TCP    17s   run=my-nginx
```

创建另一个 Pod，查看 /etc/resolv.conf 是否包含 kubelet 配置的 --cluster-dns 和 --cluster-domain，是否能够将服务 my-nginx 解析到上面显示的 Cluster IP 10.254.39.135

```
cat > dnsutils-ds.yml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: dnsutils-ds
  labels:
    app: dnsutils-ds
spec:
  type: NodePort
  selector:
    app: dnsutils-ds
  ports:
  - name: http
    port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: dnsutils-ds
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      app: dnsutils-ds
  template:
    metadata:
      labels:
        app: dnsutils-ds
    spec:
      containers:
      - name: my-dnsutils
        image: tutum/dnsutils:latest
        command:
          - sleep
          - "3600"
        ports:
        - containerPort: 80
EOF

kubectl create -f dnsutils-ds.yml
```

查看状态

```
$ kubectl get pods -lapp=dnsutils-ds -o wide 
NAME                READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
dnsutils-ds-h9b2f   1/1     Running   0          2m    172.30.219.68    master   <none>           <none>
dnsutils-ds-k27wc   1/1     Running   0          2m    172.30.166.132   node1    <none>           <none>
dnsutils-ds-pdpdt   1/1     Running   0          2m    172.30.104.3     node2    <none>           <none>

$ kubectl exec dnsutils-ds-h9b2f -- cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.254.0.2
options ndots:5
root@master:/opt/k8s/test# kubectl exec dnsutils-ds-h9b2f -- cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.254.0.2
options ndots:5

$ kubectl exec dnsutils-ds-h9b2f -- nslookup kubernetes
Server:		10.254.0.2
Address:	10.254.0.2#53
Name:	kubernetes.default.svc.cluster.local
Address: 10.254.0.1

$ kubectl exec dnsutils-ds-h9b2f -- nslookup www.baidu.com
Server:		10.254.0.2
Address:	10.254.0.2#53
Non-authoritative answer:
www.baidu.com	canonical name = www.a.shifen.com.
Name:	www.a.shifen.com
Address: 61.135.169.121
Name:	www.a.shifen.com
Address: 61.135.169.125

$ kubectl exec dnsutils-ds-h9b2f -- nslookup my-nginx
Server:		10.254.0.2
Address:	10.254.0.2#53
Name:	my-nginx.default.svc.cluster.local
Address: 10.254.39.135
```

# 2. 部署 dashboard 插件

## (1) 下载和修改配置文件

```
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc4/aio/deploy/recommended.yaml
mv recommended.yaml dashboard-recommended.yaml
```

## (2) 执行所有定义文件

```
kubectl apply -f dashboard-recommended.yaml
```

## (3) 查看运行状态

```
$ kubectl get pods -n kubernetes-dashboard
NAME                                         READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-779f5454cb-t85ff   1/1     Running   0          70s
kubernetes-dashboard-bc6669f7c-rmnjl         1/1     Running   0          71s
```

## (4) 访问 dashboard

从 1.7 开始，dashboard 只允许通过 https 访问，如果使用 kube proxy 则必须监听 localhost 或 127.0.0.1。对于 NodePort 没有这个限制，但是仅建议在开发环境中使用。对于不满足这些条件的登录访问，在登录成功后浏览器不跳转，始终停在登录界面。

通过 port forward 访问 dashboard

```
kubectl port-forward -n kubernetes-dashboard svc/kubernetes-dashboard 4443:443 --address 0.0.0.0

如果 chrome 不可以访问，输入 thisisunsafe
```

## (5) 创建登录 Dashboard 的 token 和 kubeconfig 配置文件

```
kubectl create sa dashboard-admin -n kube-system
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
ADMIN_SECRET=$(kubectl get secrets -n kube-system | grep dashboard-admin | awk '{print $1}')
DASHBOARD_LOGIN_TOKEN=$(kubectl describe secret -n kube-system ${ADMIN_SECRET} | grep -E '^token' | awk '{print $2}')
echo ${DASHBOARD_LOGIN_TOKEN}

# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/cert/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=dashboard.kubeconfig

# 设置客户端认证参数，使用上面创建的 Token
kubectl config set-credentials dashboard_user \
  --token=${DASHBOARD_LOGIN_TOKEN} \
  --kubeconfig=dashboard.kubeconfig

# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=dashboard_user \
  --kubeconfig=dashboard.kubeconfig

# 设置默认上下文
kubectl config use-context default --kubeconfig=dashboard.kubeconfig
```
