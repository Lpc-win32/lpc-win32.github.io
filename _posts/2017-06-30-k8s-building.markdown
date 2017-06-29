---
layout:     post
title:      "kubernetes集群搭建手册--未完待续..."
subtitle:   " \"手把手教学kubernetes集群搭建\""
date:       2017-06-30 1:06
header-img: "img/google-picture.jpg" 
tags:
    - kubernetes
    - etcd
    - master
    - minion
    - calico
    - lvs
---

### 1. kubernetes-master节点部署

> master节点ip：172.16.189.22

master是中央控制点,它提供了一个统一的集群视图。有一个主节点,控制多个下属slave

master节点需要的组件如下：

- etcd
- kubelet
- kube-apiserver
- kube-controller-manager
- kube-scheduler

#### 1.1 etcd部署

- etcd version：2.2.1
- etcdctl version：2.2.1

#### 1.2 etcd进程启动

##### 1.2.1 etcd配置文件管理

etcd进程采用Systemd管理。在/lib/systemd/system路径下创建etcd.service文件，内容如下：

```
[Unit]
Description=Etcd Server
After=network.target

[Service]
Type=notify
WorkingDirectory=/
EnvironmentFile=-/etc/k8s/cfg/etcd.conf
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/local/bin/etcd $NAME \
                                                                $DATADIR \
                                                                $LISTEN_PEER_URLS \
                                                                $LISTEN_CLIENT_URLS \
                                                                $ADVERTISE_CLIENT_URLS \
                                                                "

[Install]
WantedBy=multi-user.target
```

创建配置文件/etc/k8s/cfg/etcd.conf存放etcd配置文件，内容如下：

```
NAME="--name etcd-test"
DATA_DIR="--data-dir /var/etcd"
LISTEN_CLIENT_URLS="--listen-client-urls http://172.16.189.22:2379,http://172.16.189.22:4001,http://127.0.0.1:2379"
ADVERTISE_CLIENT_URLS="--advertise-client-urls http://172.16.189.22:2379,http://172.16.189.22:4001"
```

- NAME： 当前etcd名称，etcd集群内需唯一
- DATA_DIR： 指定节点的数据存储目录，这些数据包括节点ID，集群ID，集群初始化配置，Snapshot文件，若未指定-wal-dir，还会存储WAL文件；如果不指定会用缺省目录。
- LISTEN\_CLIENT\_URLS： 用于监听当前etcd节点etcd客户发送信息的地址
- ADVERTISE\_CLIENT\_URLS: etcd客户使用，客户通过该地址与本member交互信息

##### 1.2.2 etcd进程启动

1. 更新Systemd状态：systemctl daemon-reload
2. 设置etcd开启启动：systemctl enable etcd
3. 启动etcd服务：systemctl start etcd

如果没有什么报错，etcd进程就能正常启动了，登录到其他节点上可以通过curl进行简单的验证一下etcd服务是否正常运行：

```
[root@master_189_23 ~]# curl 172.16.189.22:4001/version
{"etcdserver":"2.2.1","etcdcluster":"2.2.0"}
# 看到有这样的提示，表明etcd启动成功
```

#### 1.3 签署证书

集群通讯间可采用TLS(https)双向认证机制，需要自建CA签发证书

##### 1.3.1 自签 CA

```
# 创建证书存放目录
[root@master_189_22 net.d]# mkdir -p /etc/kubernetes/ssl && cd /etc/kubernetes/ssl
# 创建CA私钥
[root@master_189_22 net.d]# openssl genrsa -out ca-key.pem 2048
# 自签CA
[root@master_189_22 net.d]# openssl req -x509 -new -nodes -key ca-key.pem -days 10000 -out ca.pem -subj "/CN=kube-ca"
```

##### 1.3.2 签署apiserver证书

用于与apiserver组件的https交互，单台master节点大致分为如下3个步骤：

1. 复制openssl配置文件
- [root@master_189_22 net.d]# cp /etc/pki/tls/openssl.cnf .
2. 编辑openssl.cnf配置使其支持 IP 认证
- [root@master_189_22 net.d]# vi openssl.cnf

```
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.c.meizu.com
DNS.5 = kube-master
DNS.6 = kubernetes-master
IP.1 = 192.168.3.1      # Cluster IP
IP.2 = 172.16.189.22
```

3. 开始签署apiserver相关的证书

```
# 生成apiserver私钥
[root@master_189_22 net.d]# openssl genrsa -out apiserver-key.pem 2048
# 生成签署请求
[root@master_189_22 net.d]# openssl req -new -key apiserver-key.pem -out apiserver.csr -subj "/CN=kube-apiserver" -config openssl.cnf
# 使用自建CA签署
[root@master_189_22 net.d]# openssl x509 -req -in apiserver.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out apiserver.pem -days 365 -extensions v3_req -extfile openssl.cnf
```

4. 签署node证书

配置worker-openssl.cnf

```
[req]
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
IP.1 = 172.16.189.22
```

签署node证书

```
# 先声明WORKER_FQDN变量方便引用
WORKER_FQDN=worker-1          # node 昵称
# 生成node私钥
openssl genrsa -out ${WORKER_FQDN}-worker-key.pem 2048
# 生成签署请求
openssl req -new -key ${WORKER_FQDN}-worker-key.pem -out ${WORKER_FQDN}-worker.csr -subj "/CN=${WORKER_FQDN}" -config worker-openssl.cnf
# 使用自建 CA 签署
openssl x509 -req -in ${WORKER_FQDN}-worker.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out ${WORKER_FQDN}-worker.pem -days 365 -extensions v3_req -extfile worker-openssl.cnf
```

5. worker-kubeconfig配置文件

```
apiVersion: v1
kind: Config
clusters:
- name: local
  cluster:
    certificate-authority: /etc/kubernetes/ssl/ca.pem
users:
- name: kubelet
  user:
    client-certificate: /etc/kubernetes/ssl/worker-1-worker.pem
    client-key: /etc/kubernetes/ssl/worker-1-worker-key.pem
contexts:
- context:
    cluster: local
    user: kubelet
  name: kubelet-context
current-context: kubelet-context
```

#### 1.4 kubelet部署

##### 1.4.1 kubelet简介

kubernetes是一个分布式的集群管理系统，在每个节点（node）上都要运行一个worker对容器进行生命周期的管理，这个worker程序就是kubelet

kubelet主要功能有：

1. Pod管理
2. 容器健康检查
3. 容器监控

##### 1.4.2 kubelet配置文件管理

将kubelet组件交由Systemd管理，在/lib/systemd/system新建service文件：kubelet.service 内容如下：

```
[root@master_189_22 system]# vi kubelet.service 
[Unit]
Description=Kubernetes Kubelet
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/local/bin/hyperkube kubelet --network-plugin=cni --network-plugin-dir=/etc/cni/net.d --logtostderr=true \
                                           --v=2 --allow-privileged=true --address=172.16.189.22  --port=10250 --hostname-override=172.16.189.22 \
                                           --api-servers=https://172.16.189.22:443  --log-dir=/var/log/k8s/kubelet --config=/etc/kubernetes/manifests \
                                           --pod-infra-container-image=reg.local:5000/pause:3.0
                                           --kubeconfig=/etc/kubernetes/ssl/worker-kubeconfig.yaml
Restart=always
StartLimitInterval=3600
StartLimitBurst=1000

[Install]
WantedBy=multi-user.target
```

- network-plugin=cni： CNI以插件方式支持多种网络（包括我们所使用的Calico）
- allow-privileged=true：允许特权模式
- api-servers=...： kube-apiserver暴露的http/https地址
- pod-infra-container-image：每个Pod启动时都会启动的基础容器pause，pause容器做为Pod的网络接入点，Pod中其他的容器会使用容器映射模式启动并接入到这个pause容器。属于同一个Pod的所有容器共享网络的namespace
- config： 本地manifest文件的路径或者目录

##### 1.4.3 kubelet组件启动

1. 更新Systemd状态：systemctl daemon-reload
2. 设置kubelet开启启动：systemctl enable kubelet
3. 启动kubelet服务：systemctl start kubelet

> 注意：执行start后，往往kubelet并不能成功启动。使用命令 journalctl -xe 查看报错信息

#### 1.5 kube-apiserver部署

##### 1.5.1 kube-apiserver简介

API Server负责和etcd交互（其他组件不会直接操作etcd，只有API Server这么做），是整个kubernetes集群的数据中心，所有的交互都是以API Server为核心的。简单来说，API Server提供了以下功能：

- 整个集群管理的API接口
- 集群内部各个模块之间通信的枢纽：所有模块之前并不会之间互相调用，而是通过和API Server打交道来完成自己那部分的工作
- 集群安全控制：API Server提供的验证和授权保证了整个集群的安全

##### 1.5.2 kube-apiserver配置

kube-apiserver组件以容器的方式运行于k8s集群中的master上，因此需要将其配置文件至于manifests路径下，manifests在kubelet中配置，在上文中有介绍。配置内容如下所示：

```
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
spec:
  hostNetwork: true
  containers:
  - name: kube-apiserver
    image: reg.local:5000/google_containers/hyperkube-amd64:vs1.5.2
    command:
    - /bin/sh
    - -c
    - /hyperkube apiserver --insecure-bind-address=0.0.0.0
      --insecure-port=8080
      --etcd-servers=http://172.16.189.22:4001
      --advertise-address=172.16.189.22
      --admission-control=NamespaceLifecycle,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota
      --service-cluster-ip-range=192.168.3.0/24 --client-ca-file=/etc/kubernetes/ssl/ca.pem
      --tls-cert-file=/etc/kubernetes/ssl/apiserver.pem --tls-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem
      --secure-port=443  --v=4 --apiserver-count=2
      --allow-privileged=True
      --logtostderr=true
      --log-dir=/var/log/
    ports:
    - containerPort: 443
      hostPort: 443
      name: https
    - containerPort: 7080
      hostPort: 7080
      name: http
    - containerPort: 8080
      hostPort: 8080
      name: local
    volumeMounts:
    - mountPath: /etc/kubernetes
      name: etckube
    - mountPath: /var/log
      name: logfile
    - mountPath: /etc/ssl
      name: etcssl
      readOnly: true
    - mountPath: /etc/pki/tls
      name: etcpkitls
      readOnly: true
  volumes:
  - hostPath:
      path: /etc/kubernetes
    name: etckube
  - hostPath:
      path: /var/log/k8s
    name: logfile
  - hostPath:
      path: /etc/ssl
    name: etcssl
  - hostPath:
      path: /etc/pki/tls
    name: etcpkitls
```

- insecure-bind-address： HTTP 访问的地址
- insecure-port： HTTP访问的端口
- service-cluster-ip-range： service要使用的网段，使用CIDR格式
- admission-control： 准入控制

将完成的kube-apiserver.yaml放入/etc/kubernetes/manifests路径下，kube-apiserver将会自动以容器的方式启动

##### 1.5.3 kube-apiserver验证

```
[root@master_189_22 manifests]# docker ps 
CONTAINER ID        IMAGE                                                      COMMAND                  CREATED             STATUS              PORTS               NAMES
abcf5b450229        reg.local:5000/google_containers/hyperkube-amd64:vs1.5.2   "/bin/sh -c '/hyperku"   2 hours ago         Up 2 hours                              k8s_kube-apiserver.ccbfa766_kube-apiserver-172.16.189.22_default_a1b53b0981653b1ebff3a63c40d35ff5_5613d2ba
85a60b158c7f        reg.local:5000/pause:3.0                                   "/pause"                 2 hours ago         Up 2 hours                              k8s_POD.a47cda96_kube-apiserver-172.16.189.22_default_a1b53b0981653b1ebff3a63c40d35ff5_0503c135
```

使用kubectl命令验证apiserver是否启动并服务

```
[root@master_189_22 manifests]# kubectl get pod
No resources found.
# 出现这种情况，说明kube-apiserver已成功运行，并提供了http服务
```

#### 1.6 kube-controller-manager部署

##### 1.6.1 kube-controller-manager简介

controller manager主要的工作就是和apiserver通信，获取集群的特定信息，然后做出响应的反馈动作

##### 1.6.2 kube-controller-manager配置文件

将kube-controller-manager.yaml文件放入manifests中启动

```
apiVersion: v1
kind: Pod
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-controller-manager
    image: reg.local:5000/google_containers/hyperkube-amd64:vs1.5.2
    command:
    - /bin/sh
    - -c
    - /hyperkube controller-manager --master=https://kubernetes:443
      --v=2
      --leader-elect=true
      --service-account-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem
      --root-ca-file=/etc/kubernetes/ssl/ca.pem
      --kubeconfig=/etc/kubernetes/ssl/worker-kubeconfig.yaml
      --logtostderr=true
      --log-dir=/var/log/
    livenessProbe:
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10252
      initialDelaySeconds: 15
      timeoutSeconds: 1
    volumeMounts:
    - mountPath: /etc/kubernetes/ssl
      name: ssl-certs-kubernetes
      readOnly: true
    - mountPath: /etc/ssl/certs
      name: ssl-certs-host
      readOnly: true
    - mountPath: /var/log/
      name: logfile
  volumes:
  - hostPath:
      path: /etc/kubernetes/ssl
    name: ssl-certs-kubernetes
  - hostPath:
      path: /usr/share/ca-certificates
    name: ssl-certs-host
  - hostPath:
      path: /var/log/k8s-test
    name: logfile
```

#### 1.7 kube-scheduler部署

##### 1.7.1 kube-scheduler简介

为新建立的Pod进行节点(node)选择(即分配机器)，负责集群的资源调度

##### 1.7.2 kube-scheduler配置文件

```
apiVersion: v1
kind: Pod
metadata:
  name: kube-scheduler
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-scheduler
    image: reg.local:5000/google_containers/hyperkube-amd64:vs1.5.2
    command:
    - /bin/sh
    - -c
    - /hyperkube scheduler  --master=https://kubernetes:443
      --leader-elect=true
      --logtostderr=false
      --log-dir=/var/log/
      --kubeconfig=/etc/kubernetes/ssl/worker-kubeconfig.yaml
      --v=3
    livenessProbe:
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10251
      initialDelaySeconds: 15
      timeoutSeconds: 1
    volumeMounts:
    - mountPath: /etc/kubernetes/ssl
      name: etckube
    - mountPath: /var/log/
      name: logfile
  volumes:
  - hostPath:
      path: /var/log/k8s
    name: logfile
  - hostPath:
      path: /etc/kubernetes/ssl
    name: etckube
```

#### 1.8 kubernetes集群master节点部署验证

经过上述步骤，master节点部署完毕，通过kubectl工具验证

```
[root@master_189_22 manifests]# kubectl get pod -o wide
NAME                           READY     STATUS    RESTARTS   AGE       IP              NODE
kube-apiserver-172.16.189.22   1/1       Running   0          1h        172.16.189.22   172.16.189.22
```

可以看到kube-apiserver pod已经成功Running，至此，master节点部署完成

### 2. kubernetes集群minion节点部署

> minion节点ip：172.16.189.30

minion是一个由主节点委托运行任务的worker。它能运行一个或多个pod。它在容器环境中提供了一个特定于应用程序的"虚拟主机"

minion需要的组件如下：

- kubelet
- kube-proxy
- calico

#### 2.1 获取master节点上的worker证书

由于master节点上启用了TLS双向认证管理。因此我们需要获取master节点上的worker证书与kubeconfig配置文件到/etc/kubernetes/ssl路径下，文件列表如下：

- ca.pem
- worker-1-worker-key.pem
- worker-1-worker.pem
- worker-kubeconfig.yaml

#### 2.2 kubelet部署

同master节点一样，kubelet组件运行在宿主机上，kubelet服务通过Systemd管理。kubelet.service文件置于/lib/systemd/system路径下，配置内容如下：

```
[Unit]
Description=Kubernetes Kubelet
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/local/bin/hyperkube kubelet --kubeconfig=/etc/kubernetes/ssl/worker-kubeconfig.yaml --logtostderr=true \
                                           --pod-infra-container-image=reg.local:5000/pause:3.0  --network-plugin-dir=/etc/cni/net.d \
                                           --network-plugin=cni --config=/etc/kubernetes/manifests  --v=2  --address=172.16.189.30 \
                                           --port=10250  --hostname-override=172.16.189.30  --api-servers=https://172.16.189.22:443 \
                                           --allow-privileged=true  --log-dir=/var/log/k8s \
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

#### 2.3 kube-proxy部署

##### 2.3.1 kube-proxy简介

kube-proxy网络代理运行在每个minion节点上。它的功能反映了定义在每个节点上kubernetes api中的services信息，并且可以做简单的TCP流转发或在一组服务后端做round robin的流转发。用于选择一个端口暴露服务

##### 2.3.2 kube-proxy配置

```
apiVersion: v1
kind: Pod
metadata:
  name: kube-proxy
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-proxy
    image: reg.local:5000/google_containers/hyperkube-amd64:v1.5.2
    command:
    - /bin/sh
    - -c
    - /hyperkube proxy --master=https://172.16.189.22:443 --hostname-override=172.16.189.30
      --proxy-mode=iptables
      --logtostderr=true
      --log-dir=/var/log/
      --v=2
      --kubeconfig=/etc/kubernetes/ssl/worker-kubeconfig.yaml
    securityContext:
       privileged: true
    volumeMounts:
    - mountPath: /etc/kubernetes/ssl
      name: etckube
    - mountPath: /var/log/
      name: logfile
  volumes:
  - hostPath:
      path: /var/log/k8s/kube-proxy
    name: logfile
  - hostPath:
      path: /etc/kubernetes/ssl
    name: etckube
```

将kube-proxy.yaml置于/etc/kubernetes/manifests路径下，kubelet将为我们启动kube-proxy组件

```
[root@master_189_29 kubernetes]# docker ps
CONTAINER ID        IMAGE                                                     COMMAND                  CREATED             STATUS              PORTS               NAMES
230031a56556        reg.local:5000/google_containers/hyperkube-amd64:v1.5.2   "/bin/sh -c '/hyperku"   3 seconds ago       Up 2 seconds                            k8s_kube-proxy.fc56beed_kube-proxy-172.16.189.30_kube-system_f36d6665bd44281c7d65da89cd8c62ce_2656a4b1
056dda523e61        reg.local:5000/pause:3.0                                  "/pause"                 5 minutes ago       Up 5 minutes                            k8s_POD.a47cda96_kube-proxy-172.16.189.30_kube-system_f36d6665bd44281c7d65da89cd8c62ce_d2169619
```

1. 更新Systemd状态：systemctl daemon-reload
2. 设置kubelet开启启动：systemctl enable kubelet
3. 启动kubelet服务：systemctl start kubelet

登录到master(172.16.189.22)节点上查看集群node：

```
[root@master_189_22 manifests]# kubectl get node
NAME            STATUS    AGE
172.16.189.22   Ready     2h
172.16.189.30   Ready     1m
# 可以看到新部署的minion节点已被k8s集群接管
```

#### 2.4 Calico部署

### 3. K8s集群LVS部署

> LVS机器地址： 172.16.189.198  
LVSL互联IP： 172.16.189.64/26 Gateway: 172.16.189.65  
LVSL VIP： 172.16.189.128/26 Gateway: 172.16.189.129  
LVSL Local IP： 172.16.189.192 Gateway: 172.16.189.193

- keepalived(Support full-nat)
- keepalived-vip
- quagga(ospf)

#### 3.1 keepalived部署

1. 编译keepalived为可执行程序，置于/usr/sbin/路径下
2. 创建/etc/keepalived/keepalived.conf文件

```
global_defs {
  vrrp_version 3
  vrrp_iptables KUBE-KEEPALIVED-VIP
}

local_address_group laddr_g1 {  
  172.16.189.201-230
}
# 新增lvs转发规则后，keepalived-vip会更改次配置文件，并通知keepalived reload
```

3. /usr/sbin/keepalived --dont-fork -S 0 -C -d -D & 后台启动keepalived

#### 3.2 keepalived-vip部署

1. 拷贝worker证书

```
[root@master_189_198 log]# ll /etc/kubernetes/ssl/
total 16
-rw-r--r-- 1 root root 1090 Jun 29 14:13 ca.pem
-rw-r--r-- 1 root root 1675 Jun 29 14:13 worker-1-worker-key.pem
-rw-r--r-- 1 root root 1042 Jun 29 14:13 worker-1-worker.pem
```

2. 启动keepalived-vip
 
```
kube-keepalived-vip --services-configmap=default/vip-configmap --server=https://172.16.189.22 \
                            --use-kubernetes-cluster-service=false --use-local-addresses=172.16.189.201-230 \
                            --use-service-port=false --use-link-address=172.16.189.65 \
                            --certificate-authority=/etc/kubernetes/ssl/ca.pem --client-certificate=/etc/kubernetes/ssl/worker-1-worker.pem \
                            --client-key=/etc/kubernetes/ssl/worker-1-worker-key.pem --metrics-time-period=10
```

- services-configmap=default/vip-configmap: ConfigMap名字
- local-addresses: 与real server互联的地址
- link-address: 与switch互联的地址
- use-service-port: true - 暴露service port; false - 暴露nodeport

#### 3.3 quagga(ospfd)部署

##### 3.3.1 quagga简介

Quagga软件原名是Zebra是由一个日本开发团队编写的一个以GNU版权方式发布的软件。可以使用Quagga将Linux机器打造成一台功能完备的路由器。  
Quagga能够同时支持RIPv1、RIPv2、RIPng、OSPFv2、OSPFv3、BGP-4和 BGP-4+等诸多TCP/IP协议。

我们使用其的目的是通过ospf协议宣告LVS full-nat模式中的VIP网段，以供外部路由

##### 3.3.2 quagga运行前准备

quagga需要以quagga用户启动，且需要/var/run/quagga、/data/log/quagga两个路径的支持。前者用用户存储进程运行时PID，后者用于存放日志文件。执行步骤如下：

```
[root@master_189_198 quagga]# useradd quagga
[root@master_189_198 quagga]# mkdir /var/run/quagga
[root@master_189_198 quagga]# chown quagga /var/run/quagga
[root@master_189_198 quagga]# mkidr /data/log/quagga
[root@master_189_198 quagga]# chown quagga /data/log/quagga
```

##### 3.3.3 配置ospf路由协议

```
[root@master_189_198 quagga]# cat /data/quagga/etc/ospfd.conf 
!
! Zebra configuration saved from vty
!   2014/07/13 16:32:25
!
hostname FLY-dockerCS-32.-130
password 8 WHQQH6POr04LQ
enable password 8 WHQQH6POr04LQ
log file /data/log/quagga/ospfd.log
log stdout
log syslog
service password-encryption
!
!
interface eth0
 ip ospf network point-to-point
 ip ospf hello-interval 1
 ip ospf dead-interval 4
!
interface lo
!
router ospf
 ospf router-id 172.16.189.198
 log-adjacency-changes
! Important: ensure reference bandwidth is consistent across all routers
  auto-cost reference-bandwidth 10000

  network 172.16.189.128/26 area 0.0.0.1

line vty
!
```

##### 3.3.4 启动ospfd

```
[root@master_189_198 quagga]# /data/quagga/sbin/ospfd -A 127.0.0.1 -f /data/quagga/etc/ospfd.conf -d
```

#### 3.4 验证LVS部署情况

1. 登录到k8s集群master节点，创建configMap

```
apiVersion: v1
data:
kind: ConfigMap
metadata:
  name: vip-configmap
  namespace: default
--------------------------
# k8s集群中创建configMap
[root@master_189_22 ~]# kubectl create configmap -f vip-configmap.yaml
```

2. 修改现有service为nodePort模式(use-service-port=false)
3. 增加lvs转发规则

```
[root@master_189_22 ~]# kubectl edit configmap vip-configmap
------------------------------------
apiVersion: v1
data:
  # 增加vip到
  172.16.189.130: default/kubernetes
kind: ConfigMap
metadata:
  creationTimestamp: 2017-06-29T08:14:32Z
  name: vip-configmap
  namespace: default
  resourceVersion: "173657"
  selfLink: /api/v1/namespaces/default/configmaps/vip-configmap
  uid: facad9c8-5ca2-11e7-9305-6c92bf3426ba
```

4. 登录lvs机器查看

```
[root@master_189_198 quagga]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  172.16.189.130:30176 wlc
```

5. 查看keepalived.conf是否新增转发规则

```
[root@master_189_198 quagga]# cat /etc/keepalived/keepalived.conf 


global_defs {
  vrrp_version 3
  vrrp_iptables KUBE-KEEPALIVED-VIP
}

local_address_group laddr_g1 {  
  172.16.189.201-230
}


# Service: default/kubernetes
virtual_server 172.16.189.130 30176 {
  delay_loop 10
  lvs_sched wlc
  lvs_method FNAT
#  persistence_timeout 1800
  protocol TCP
  syn_proxy
  laddr_group_name laddr_g1
  alpha
  omega
  quorum 1
  hysteresis 0
  quorum_up "/sbin/ip addr add 172.16.189.130/32 dev lo;"
  quorum_down "echo hello;"
#  quorum_down "ip addr del 172.16.189.130/32 dev lo;"

  
  real_server 172.16.189.22 443 {
    weight 1
    MISC_CHECK {
        misc_path "/sbin/rs_check.sh 172.16.189.22 443 5 default/kubernetes 10.3.239.30:9091 lbe2"
        misc_timeout 6 
    }
  }

}
```
