---
layout:     post
title:      二进制方式安装kubernetes v1.12.6高可用集群
subtitle:   
date:       2019-03-12
author:     Kaerser
header-img: img/tag-bg.jpg
catalog: true
tags:
    - kubernetes
---


[TOC]

------



####  一、集群节点规划

```shell
master01/etcd01:        10.199.139.101
master02/etcd02:        10.199.139.102
master03/etcd03:        10.199.139.103
node01:                 10.199.139.104
node02:                 10.199.139.105
```
***master服务映射关系：***

```shell
前端：10.199.136.220:443
后端：10.199.139.101:6443    
     10.199.139.102:6443    
     10.199.139.103:6443

前端：10.199.136.220:80
后端：10.199.139.101:8080
     10.199.139.102:8080
     10.199.139.103:8080
```


#### 二、hosts文件配置，互信配置
```
10.199.139.101         hz-k8s-master-199-139-101
10.199.139.102         hz-k8s-master-199-139-102
10.199.139.103         hz-k8s-master-199-139-103
10.199.139.104         hz-k8s-node-199-139-104
10.199.139.105         hz-k8s-node-199-139-105
```

#### 三、版本
```shell
master:  v1.12.6   https://dl.k8s.io/v1.12.6/kubernetes-server-linux-amd64.tar.gz
node:    v1.12.6   https://dl.k8s.io/v1.12.6/kubernetes-node-linux-amd64.tar.gz
client:  v1.12.6   https://dl.k8s.io/v1.12.6/kubernetes-client-linux-amd64.tar.gz
etcd:    v3.3.12   https://github.com/etcd-io/etcd/releases/download/v3.3.12/etcd-v3.3.12-linux-amd64.tar.gz
cfssl:             https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
cfssljson          https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
cfssl-certinfo     https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
```


#### 四、配置安装docker
```shell
yum-config-manager  --add-repo  https://download.docker.com/linux/centos/docker-ce.repo
yum list docker-ce --showduplicates | sort -r
yum install -y docker-ce-18.06.1.ce-3.el7
```

#### 五、创建所需目录以及ca证书和密钥
```shell
for i in {101..103}
do
    ssh 10.199.139.$i 'mkdir -p /usr/local/k8s/bin/env;mkdir -p /etc/kubernetes/ssl/;mkdir -p /etc/etcd/ssl/;mkdir -p /var/lib/etcd/;mkdir -p /etc/flanneld/ssl/'
done
```

***生成默认config.json和csr.json***

```shell
./cfssl print-defaults config > config.json
./cfssl print-defaults csr > csr.json
```

***config.json配置文件：***

```yaml
{
    "signing": {
        "default": {
            "expiry": "168h"
        },
        "profiles": {
            "www": {
                "expiry": "8760h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry": "8760h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            }
        }
    }
}
```

***csr.json配置文件：***

```yaml
{
    "CN": "example.net",
    "hosts": [
        "example.net",
        "www.example.net"
    ],
    "key": {
        "algo": "ecdsa",
        "size": 256
    },
    "names": [
        {
            "C": "US",
            "L": "CA",
            "ST": "San Francisco"
        }
    ]
}
```

***创建CA config.json文件ca-config.json：***

```yaml
{
    "signing": {
        "default": {
            "expiry": "87600h"
        },
        "profiles": {
            "kubernetes": {
                "expiry": "87600h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
```

***创建ca csr.json文件ca-csr.jaon：***

```yaml
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
```

***生成CA证书和私钥：***

```shell
/root/k8s-v1.12.6/cfssl/cfssl gencert -initca ca-csr.json | /root/k8s-v1.12.6/cfssl/cfssljson -bare ca
```

***分发ca证书：***

```shell
for i in {101..103}
do
    scp /root/k8s-v1.12.6/cfssl/ca/ca* 10.199.139.$i:'/etc/kubernetes/ssl/'
done
```

#### 六、定义全局环境变量：
```shell
# TLS Bootstrapping 使用的Token，可以使用命令 head -c 16 /dev/urandom | od -An -t x | tr -d ' ' 生成
export BOOTSTRAP_TOKEN="8981b594122ebed7596f1d3b69c78223"
# 建议使用未用的网段来定义服务网段和Pod 网段
# 服务网段(Service CIDR)，部署前路由不可达，部署后集群内部使用IP:Port可达
export SERVICE_CIDR="10.254.0.0/16"
# Pod 网段(Cluster CIDR)，部署前路由不可达，部署后路由可达(flanneld 保证)
export CLUSTER_CIDR="172.30.0.0/16"
# 服务端口范围(NodePort Range)
export NODE_PORT_RANGE="30000-32766"
# etcd集群服务地址列表
export ETCD_ENDPOINTS="https://10.199.139.101:2379,https://10.199.139.102:2379,https://10.199.139.103:2379"
# flanneld 网络配置前缀
export FLANNEL_ETCD_PREFIX="/kubernetes/network"
# kubernetes 服务IP(预先分配，一般为SERVICE_CIDR中的第一个IP)
export CLUSTER_KUBERNETES_SVC_IP="10.254.0.1"
# 集群 DNS 服务IP(从SERVICE_CIDR 中预先分配)
export CLUSTER_DNS_SVC_IP="10.254.0.2"
# 集群 DNS 域名
export CLUSTER_DNS_DOMAIN="cluster.local."
# MASTER API Server 地址
export MASTER_URL="10.199.136.220"
```
#### 七、部署etcd集群
```shell
etcd集群信息：
etcd01:   10.199.139.101
etcd02:   10.199.139.102
etcd03:   10.199.139.103
```
##### 1.定义环境变量：
```shell
# 当前部署的机器名称(随便定义，只要能区分不同机器即可)
export NODE_NAME=master01
# 当前部署的机器IP
export NODE_IP=10.199.139.101
# etcd 集群所有机器 IP
export NODE_IPS="10.199.139.101 10.199.139.102 10.199.139.103"
# etcd 集群间通信的IP和端口
export ETCD_NODES=master01=https://10.199.139.101:2380,master02=https://10.199.139.102:2380,master03=https://10.199.139.103:2380
```

```shell
#导入用到的其它全局变量：ETCD_ENDPOINTS、FLANNEL_ETCD_PREFIX、CLUSTER_CIDR
source /usr/local/k8s/bin/env/env.sh
```

##### 2.分发etcd文件：
```shell
for i in {101..103}
do
    scp /root/k8s-v1.12.6/etcd-v3.3.12/etcd-v3.3.12-linux-amd64/etcd* 10.199.139.$i:/usr/local/k8s/bin/
done
```

##### 3.生成证书请求文件：

```shell
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "${NODE_IP}"
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
      "OU": "System"
    }
  ]
}
EOF
```

*生成etcd证书和私钥：*

```shell
/root/k8s-v1.12.6/cfssl/cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes etcd-csr.json | /root/k8s-v1.12.6/cfssl/cfssljson -bare etcd
```

*分发etcd证书和私钥至对应主机：*

```shell
scp etcd.pem etcd-key.pem 10.199.139.101:/etc/etcd/ssl/
```

##### 4.创建etcd systemd unit文件：
```shell
cat > etcd.service <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/local/k8s/bin/etcd \\
  --name=${NODE_NAME} \\
  --cert-file=/etc/etcd/ssl/etcd.pem \\
  --key-file=/etc/etcd/ssl/etcd-key.pem \\
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \\
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \\
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --initial-advertise-peer-urls=https://${NODE_IP}:2380 \\
  --listen-peer-urls=https://${NODE_IP}:2380 \\
  --listen-client-urls=https://${NODE_IP}:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls=https://${NODE_IP}:2379 \\
  --initial-cluster-token=etcd-cluster-0 \\
  --initial-cluster=${ETCD_NODES} \\
  --initial-cluster-state=new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
 EOF
```
*分发etcd.service文件*

##### 5.启动etcd集群：
```shell
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
systemctl status etcd
```

##### 6.验证etcd集群服务状态：
```shell
for ip in ${NODE_IPS}; do
  ETCDCTL_API=3 /usr/local/k8s/bin/etcdctl \
  --endpoints=https://${ip}:2379  \
  --cacert=/etc/kubernetes/ssl/ca.pem \
  --cert=/etc/etcd/ssl/etcd.pem \
  --key=/etc/etcd/ssl/etcd-key.pem \
  endpoint health; done
```

*输出以下内容及表示etcd集群状态正常：*

```shell
https://10.199.139.101:2379 is healthy: successfully committed proposal: took = 1.754538ms
https://10.199.139.102:2379 is healthy: successfully committed proposal: took = 1.716494ms
https://10.199.139.103:2379 is healthy: successfully committed proposal: took = 1.770061ms
```
#### 八、部署flanneld网络
##### 1.创建flannled证书签名请求：
```shell
cat > flanneld-csr.json <<EOF
{
  "CN": "flanneld",
  "hosts": [],
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
      "OU": "System"
    }
  ]
}
EOF
```

*生成flanneld证书和私钥：*

```shell
/root/k8s-v1.12.6/cfssl/cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes flanneld-csr.json | /root/k8s-v1.12.6/cfssl/cfssljson -bare flanneld
```

*分发flanneld证书和私钥至对应主机：*

```shell
for i in {101..103}
do
scp flanneld.pem flanneld-key.pem 10.199.139.$i:/etc/flanneld/ssl/
done
```

##### 2.向etcd集群中写入pod网络信息
```shell
etcdctl \
  --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/flanneld/ssl/flanneld.pem \
  --key-file=/etc/flanneld/ssl/flanneld-key.pem \
  set ${FLANNEL_ETCD_PREFIX}/config '{"Network":"'${CLUSTER_CIDR}'", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}'
```

*得到以下信息：*

```shell
{"Network":"172.30.0.0/16", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}
```

##### 3.安装配置flanneld
*分发flanneld文件至指定主机：*

```shell
for i in {101..103}
do
scp /root/k8s-v1.12.6/flannel-v0.11.0/{flanneld,mk-docker-opts.sh} 10.199.139.$i:/usr/local/k8s/bin/
done
```

##### 4.创建flanneld的systemd unit文件：
```shell
cat > /etc/systemd/system/flanneld.service << EOF
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
ExecStart=/usr/local/k8s/bin/flanneld \\
  -etcd-cafile=/etc/kubernetes/ssl/ca.pem \\
  -etcd-certfile=/etc/flanneld/ssl/flanneld.pem \\
  -etcd-keyfile=/etc/flanneld/ssl/flanneld-key.pem \\
  -etcd-endpoints=${ETCD_ENDPOINTS} \\
  -etcd-prefix=${FLANNEL_ETCD_PREFIX}
ExecStartPost=/usr/local/k8s/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
EOF
```
#####  5.启动flanneld服务：
```shell
systemctl daemon-reload
systemctl enable flanneld
systemctl start flanneld
systemctl status flanneld
```

##### 6.检查分配给flanneld的pod网段信息：
*查看集群pod网段*

```shell
etcdctl \
  --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/flanneld/ssl/flanneld.pem \
  --key-file=/etc/flanneld/ssl/flanneld-key.pem \
  get ${FLANNEL_ETCD_PREFIX}/config
```

*查看已分配的pod子网列表*

```shell
etcdctl \
  --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/flanneld/ssl/flanneld.pem \
  --key-file=/etc/flanneld/ssl/flanneld-key.pem \
  ls ${FLANNEL_ETCD_PREFIX}/subnets
```

```shell
/kubernetes/network/subnets/172.30.32.0-24
/kubernetes/network/subnets/172.30.97.0-24
/kubernetes/network/subnets/172.30.71.0-24
```
*查看某一 Pod 网段对应的 flanneld 进程监听的 IP 和网络参数*

```shell
etcdctl \
  --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/flanneld/ssl/flanneld.pem \
  --key-file=/etc/flanneld/ssl/flanneld-key.pem \
  get ${FLANNEL_ETCD_PREFIX}/subnets/172.30.32.0-24
```
```shell
{"PublicIP":"10.199.139.101","BackendType":"vxlan","BackendData":{"VtepMAC":"8a:0d:69:63:81:48"}}
```

#### 九、部署master节点

##### 1.分发master二进制文件：
```
for i in {101..103}
do
scp /root/k8s-v1.12.6/server/kubernetes/server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler} 10.199.139.$i:/usr/local/k8s/bin/
done
```

##### 2.创建master证书签名请求：
```shell
cat > /root/k8s-v1.12.6/cfssl/server/master01/kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "${NODE_IP}",
    "${MASTER_URL}",
    "${CLUSTER_KUBERNETES_SVC_IP}",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
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
      "OU": "System"
    }
  ]
}
EOF
```
*生成master证书和私钥：*

```shell
/root/k8s-v1.12.6/cfssl/cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes kubernetes-csr.json | /root/k8s-v1.12.6/cfssl/cfssljson -bare kubernetes
```

*分发证书和私钥至指定主机：*

```shell
scp /root/k8s-v1.12.6/cfssl/server/master01/{kubernetes-key.pem,kubernetes.pem} 10.199.139.101:/etc/kubernetes/ssl/
scp /root/k8s-v1.12.6/cfssl/server/master02/{kubernetes-key.pem,kubernetes.pem} 10.199.139.102:/etc/kubernetes/ssl/
scp /root/k8s-v1.12.6/cfssl/server/master03/{kubernetes-key.pem,kubernetes.pem} 10.199.139.103:/etc/kubernetes/ssl/
```

##### 3.配置和启动kube-apiserver
*创建kube-apiserver使用的客户端token文件*

```shell
cat > /root/k8s-v1.12.6/cfssl/server/token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```

*分发token文件*

```shell
for i in {101..103}
do
scp /root/k8s-v1.12.6/cfssl/server/token.csv 10.199.139.$i:/etc/kubernetes/
done
```

*创建kube-apiserver的systemd unit文件：*

```shell
cat  > /etc/systemd/system/kube-apiserver.service <<EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/k8s/bin/kube-apiserver \\
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --advertise-address=${NODE_IP} \\
  --bind-address=0.0.0.0 \\
  --insecure-bind-address=${NODE_IP} \\
  --authorization-mode=Node,RBAC \\
  --runtime-config=rbac.authorization.k8s.io/v1alpha1 \\
  --kubelet-https=true \\
  --enable-bootstrap-token-auth \\
  --token-auth-file=/etc/kubernetes/token.csv \\
  --service-cluster-ip-range=${SERVICE_CIDR} \\
  --service-node-port-range=${NODE_PORT_RANGE} \\
  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \\
  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \\
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \\
  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \\
  --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem \\
  --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem \\
  --etcd-servers=${ETCD_ENDPOINTS} \\
  --enable-swagger-ui=true \\
  --allow-privileged=true \\
  --endpoint-reconciler-type=lease \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/lib/audit.log \\
  --audit-policy-file=/etc/kubernetes/audit-policy.yaml \\
  --event-ttl=1h \\
  --logtostderr=true \\
  --v=6
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```
*创建审计日志策略文件：*

```shell
cat  > /root/k8s-v1.12.6/server/audit-policy.yaml <<EOF
apiVersion: audit.k8s.io/v1beta1 # This is required.
kind: Policy
# Don't generate audit events for all requests in RequestReceived stage.
omitStages:
  - "RequestReceived"
rules:
  # Log pod changes at RequestResponse level
  - level: RequestResponse
    resources:
    - group: ""
      # Resource "pods" doesn't match requests to any subresource of pods,
      # which is consistent with the RBAC policy.
      resources: ["pods"]
  # Log "pods/log", "pods/status" at Metadata level
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods/log", "pods/status"]

  # Don't log requests to a configmap called "controller-leader"
  - level: None
    resources:
    - group: ""
      resources: ["configmaps"]
      resourceNames: ["controller-leader"]

  # Don't log watch requests by the "system:kube-proxy" on endpoints or services
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: "" # core API group
      resources: ["endpoints", "services"]

  # Don't log authenticated requests to certain non-resource URL paths.
  - level: None
    userGroups: ["system:authenticated"]
    nonResourceURLs:
    - "/api*" # Wildcard matching.
    - "/version"

  # Log the request body of configmap changes in kube-system.
  - level: Request
    resources:
    - group: "" # core API group
      resources: ["configmaps"]
    # This rule only applies to resources in the "kube-system" namespace.
    # The empty string "" can be used to select non-namespaced resources.
    namespaces: ["kube-system"]

  # Log configmap and secret changes in all other namespaces at the Metadata level.
  - level: Metadata
    resources:
    - group: "" # core API group
      resources: ["secrets", "configmaps"]

  # Log all other resources in core and extensions at the Request level.
  - level: Request
    resources:
    - group: "" # core API group
    - group: "extensions" # Version of group should NOT be included.

  # A catch-all rule to log all other requests at the Metadata level.
  - level: Metadata
    # Long-running requests like watches that fall under this rule will not
    # generate an audit event in RequestReceived.
    omitStages:
      - "RequestReceived"
EOF
```

*分发audit-policy.yaml文件：*

```shell
for i in {101..103}
do
scp /root/k8s-v1.12.6/server/audit-policy.yaml 10.199.139.$i:/etc/kubernetes/
done
```

*启动kube-apiserver服务：*

```shell
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver
```

##### 4.配置和启动kube-controller-manager
*创建kube-controller-manager的systemd unit文件*

```shell
cat > /etc/systemd/system/kube-controller-manager.service <<EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/k8s/bin/kube-controller-manager \\
  --address=127.0.0.1 \\
  --master=http://${MASTER_URL} \\
  --allocate-node-cidrs=true \\
  --service-cluster-ip-range=${SERVICE_CIDR} \\
  --cluster-cidr=${CLUSTER_CIDR} \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \\
  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \\
  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \\
  --root-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --leader-elect=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
*启动kube-controller-manager*

```shell
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
systemctl status kube-controller-manager
```

##### 配置和启动kube-scheduler
*创建kube-scheduler的systemd unit文件*

```shell
cat > /etc/systemd/system/kube-scheduler.service <<EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/k8s/bin/kube-scheduler \\
  --address=127.0.0.1 \\
  --master=http://${MASTER_URL} \\
  --leader-elect=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
*启动kube-scheduler*

```shell
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
systemctl status kube-scheduler
```

***至此master集群配置完成；***

#### 十、配置kubectl命令行工具查看集群状态

```shell
export KUBE_APISERVER="https://${MASTER_URL}"
```

**分发kubectl**

```
scp /root/k8s-v1.12.6/client/kubernetes/client/bin/kubectl 10.199.139.101:/usr/local/k8s/bin/
```

**创建admin证书**

>kubectl与kube-apiserver的安全端口通信，需要为安全通信提供TLS证书和密钥。创建admin证书签名请求：

```shell
cat > /root/k8s-v1.12.6/cfssl/client/admin/admin-csr.json <<EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF
```
**生成admin证书和密钥：**

```shell
/root/k8s-v1.12.6/cfssl/cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes admin-csr.json | /root/k8s-v1.12.6/cfssl/cfssljson -bare admin
```

**分发admin证书至所有master节点：**

```shell
for i in {101..103}
do
scp /root/k8s-v1.12.6/cfssl/client/admin/{admin-key.pem,admin.pem} 10.199.139.$i:/etc/kubernetes/ssl/
done
```

**创建kubectl kubeconfig文件**

```shell
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER}
# 设置客户端认证参数
kubectl config set-credentials admin \
  --client-certificate=/etc/kubernetes/ssl/admin.pem \
  --embed-certs=true \
  --client-key=/etc/kubernetes/ssl/admin-key.pem \
  --token=${BOOTSTRAP_TOKEN}
# 设置上下文参数
kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin
# 设置默认上下文
kubectl config use-context kubernetes
```
**验证master节点**

```shell
kubectl get componentstatuses
```

**查看cluster模式下的kube-scheduler、kube-controller-manager leader节点信息：**

```shell
kubectl get endpoints kube-scheduler --namespace=kube-system  -o yaml
kubectl get endpoints kube-controller-manager --namespace=kube-system  -o yaml
```

#### 十一、部署node节点

##### 1.创建所需目录及分发证书文件
```shell
for i in {104..105}
do
    ssh 10.199.139.$i 'mkdir -p /etc/kubernetes/ssl/;mkdir -p /etc/flanneld/ssl/;mkdir -p /usr/local/k8s/bin/；mkdir  -p /var/lib/kubelet;mkdir -p /var/lib/kube-proxy'
done

for i in {104..105}
do
scp /etc/kubernetes/ssl/ca.pem 10.199.139.$i:/etc/kubernetes/ssl/
scp /root/k8s-v1.12.6/flannel-v0.11.0/{flanneld,mk-docker-opts.sh} 10.199.139.$i:/usr/local/k8s/bin/
done
```
##### 2.node节点安装flanneld
*此处参照上面的步骤*

##### 3.开启路由转发

```shell
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
```

##### 4.修改docker.service文件配置
```shell
systmectl cat docker
```
```shell
EnvironmentFile=-/run/flannel/docker
ExecStart=/usr/bin/dockerd --log-level=info $DOCKER_NETWORK_OPTIONS
```

##### 5.安装和配置kubelet
> **kubelet启动时向kube-apiserver发送TLS bootstrapping请求，需要先将bootstrap token文件中的kubelet-bootstrap用户赋予system:node-bootstrapper角色，然后kubelet才有权限创建认证请求(certificatesigningrequests)：**

```shell
kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```

*--user=kubelet-bootstrap 是文件 /etc/kubernetes/token.csv 中指定的用户名，同时也写入了文件 /etc/kubernetes/bootstrap.kubeconfig*

**为Node请求创建一个RBAC授权规则：**

```shell
kubectl create clusterrolebinding kubelet-nodes --clusterrole=system:node --group=system:nodes
```

**分发kubelet和kube-proxy文件：**

```shell
for i in {104..105}
do
scp /root/k8s-v1.12.6/node/kubernetes/node/bin/{kubelet,kube-proxy} 10.199.139.$i:/usr/local/k8s/bin/
done
```

**创建kubelet bootstapping kubeconfig文件**

```shell
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig
# 设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig
# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig
# 设置默认上下文
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
```
```shell
mv bootstrap.kubeconfig /etc/kubernetes/
```

**分发bootstrap.kubeconfig文件至node节点**

```shell
for i in {104..105}
do
scp bootstrap.kubeconfig 10.199.139.$i:/etc/kubernetes/
done
```

##### 6.创建kubelet的systemd unit文件
```shell
cat > kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/k8s/bin/kubelet \\
  --fail-swap-on=false \\
  --cgroup-driver=cgroupfs \\
  --address=${NODE_IP} \\
  --hostname-override=${NODE_IP} \\
  --bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \\
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \\
  --cert-dir=/etc/kubernetes/ssl \\
  --cluster-dns=${CLUSTER_DNS_SVC_IP} \\
  --cluster-domain=${CLUSTER_DNS_DOMAIN} \\
  --hairpin-mode promiscuous-bridge \\
  --allow-privileged=true \\
  --serialize-image-pulls=false \\
  --logtostderr=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
*分发kubelet服务文件*

```shell
scp /root/k8s-v1.12.6/node/kubernetes/node/node01/kubelet.service 10.199.139.104:/etc/systemd/system/
scp /root/k8s-v1.12.6/node/kubernetes/node/node02/kubelet.service 10.199.139.105:/etc/systemd/system/
```

```shell
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet
```
*查看csr信息*

```shell
kubectl get csr
```
```shell
NAME                                                   AGE     REQUESTOR           CONDITION
node-csr-YCHXi4rRM-n1MzhnW6OUr-My5zdQSfbaJkVhv2BjEzU   2m56s   kubelet-bootstrap   Pending
node-csr-wE9C2rH3IktEu735ELr--Q42RLXA1-KqkBuFW3s9JFI   3m30s   kubelet-bootstrap   Pending
```
*查看csr详细信息*

```shell
kubectl describe csr node-csr-YCHXi4rRM-n1MzhnW6OUr-My5zdQSfbaJkVhv2BjEzU
```
```shell
Name:               node-csr-YCHXi4rRM-n1MzhnW6OUr-My5zdQSfbaJkVhv2BjEzU
Labels:             <none>
Annotations:        <none>
CreationTimestamp:  Thu, 07 Mar 2019 23:02:57 +0800
Requesting User:    kubelet-bootstrap
Status:             Pending
Subject:
         Common Name:    system:node:10.199.139.105
         Serial Number:  
         Organization:   system:nodes
Events:  <none>
```
*通过csr请求：*

```shell
kubectl certificate approve node-csr-YCHXi4rRM-n1MzhnW6OUr-My5zdQSfbaJkVhv2BjEzU
```

##### 7.配置kube-proxy
*创建kube-proxy证书签名请求*

```shell
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "hosts": [],
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
      "OU": "System"
    }
  ]
}
EOF
```
*生成kube-proxy客户端证书和私钥*

```shell
/root/k8s-v1.12.6/cfssl/cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes kube-proxy-csr.json | /root/k8s-v1.12.6/cfssl/cfssljson -bare kube-proxy
```

*分发kube-proxy客户端证书和私钥*

```shell
for i in {104..105}
do
scp /root/k8s-v1.12.6/cfssl/node/{kube-proxy-key.pem,kube-proxy.pem} 10.199.139.$i:/etc/kubernetes/ssl/
done
```

*创建kube-proxy kubeconfig文件*

```shell
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig
# 设置客户端认证参数
kubectl config set-credentials kube-proxy \
  --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \
  --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
# 设置默认上下文
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```
*分发kube-proxy kubeconfig文件*

```shell
for i in {104..105}
do
scp /root/k8s-v1.12.6/cfssl/node/kube-proxy.kubeconfig 10.199.139.$i:/etc/kubernetes/
done
```

*创建kube-proxy的systemd unit文件*

```shell
cat > kube-proxy.service <<EOF
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/k8s/bin/kube-proxy \\
  --bind-address=${NODE_IP} \\
  --hostname-override=${NODE_IP} \\
  --cluster-cidr=${SERVICE_CIDR} \\
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \\
  --logtostderr=true \\
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

*分发kube-proxy的systemd unit文件*

```shell
scp /root/k8s-v1.12.6/node/kubernetes/node/node01/kube-proxy.service 10.199.139.104:/etc/systemd/system/
scp /root/k8s-v1.12.6/node/kubernetes/node/node02/kube-proxy.service 10.199.139.105:/etc/systemd/system/
```

```shell
systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy
```