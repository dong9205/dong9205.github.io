---
weight: 0
title: "Kubernetes 1.27升级至1.28"
subtitle: ""
date: 2024-08-23T21:15:18+08:00
lastmod: 2024-08-23T21:15:18+08:00
draft: false
author: "Derrick"
authorLink: "https://www.p-pp.cn/"
summary: ""
license: ""
tags: [""]
resources:
- name: "featured-image"
  src: ""
categories: 
- "documentation"

featuredImage: ""
featuredImagePreview: ""

hiddenFromHomePage: false
hiddenFromSearch: false

toc:
  enable: true
  auto: true

mapbox:
share:
  enable: true
comment:
  enable: true

lightgallery: true
---

## 介绍

之前升级都是直接使用阿里云的控制台，只知道升级了控制面，然后升级了节点，但是不知道其中的更多细节，所以自己在测试环境升级学习下, 升级的话直接按照官方文档升级即可: https://v1-28.docs.kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrading-control-plane-nodes, 我这里主要记录下命令和遇到的问题

## 命令
> 我的是本地测试环境，所以直接重启kubelet，如果是生产环境，请参考文档驱逐pod、配置节点不可调度
```bash
apt-get install -y kubeadm='1.28.13-1.1'
kubeadm upgrade plan
kubeadm upgrade apply v1.28.13 -y
systemctl restart kubelet
```

## 执行upgrade apply时卡在etcd

### 报错

#### kubeadm升级的报错

```golang
> etcd endpoints read from pods
……
I0823 11:38:26.483010   12990 etcd.go:150] etcd endpoints read from pods: https://172.18.0.2:2379
context deadline exceeded
error syncing endpoints with etcd
k8s.io/kubernetes/cmd/kubeadm/app/util/etcd.NewFromCluster
        k8s.io/kubernetes/cmd/kubeadm/app/util/etcd/etcd.go:166
k8s.io/kubernetes/cmd/kubeadm/app/phases/upgrade.StaticPodControlPlane
        k8s.io/kubernetes/cmd/kubeadm/app/phases/upgrade/staticpods.go:443
k8s.io/kubernetes/cmd/kubeadm/app/phases/upgrade.PerformStaticPodUpgrade
        k8s.io/kubernetes/cmd/kubeadm/app/phases/upgrade/staticpods.go:617
k8s.io/kubernetes/cmd/kubeadm/app/cmd/upgrade.PerformControlPlaneUpgrade
        k8s.io/kubernetes/cmd/kubeadm/app/cmd/upgrade/apply.go:216
k8s.io/kubernetes/cmd/kubeadm/app/cmd/upgrade.runApply
        k8s.io/kubernetes/cmd/kubeadm/app/cmd/upgrade/apply.go:156
k8s.io/kubernetes/cmd/kubeadm/app/cmd/upgrade.newCmdApply.func1
        k8s.io/kubernetes/cmd/kubeadm/app/cmd/upgrade/apply.go:74
github.com/spf13/cobra.(*Command).execute
        github.com/spf13/cobra@v1.6.0/command.go:916
github.com/spf13/cobra.(*Command).ExecuteC
        github.com/spf13/cobra@v1.6.0/command.go:1040
github.com/spf13/cobra.(*Command).Execute
        github.com/spf13/cobra@v1.6.0/command.go:968
k8s.io/kubernetes/cmd/kubeadm/app.Run
        k8s.io/kubernetes/cmd/kubeadm/app/kubeadm.go:50
main.main
        k8s.io/kubernetes/cmd/kubeadm/kubeadm.go:25
runtime.main
        runtime/proc.go:271
runtime.goexit
        runtime/asm_amd64.s:1695
failed to create etcd client
k8s.io/kubernetes/cmd/kubeadm/app/phases/upgrade.StaticPodControlPlane
        k8s.io/kubernetes/cmd/kubeadm/app/phases/upgrade/staticpods.go:445
k8s.io/kubernetes/cmd/kubeadm/app/phases/upgrade.PerformStaticPodUpgrade
        k8s.io/kubernetes/cmd/kubeadm/app/phases/upgrade/staticpods.go:617
k8s.io/kubernetes/cmd/kubeadm/app/cmd/upgrade.PerformControlPlaneUpgrade
        k8s.io/kubernetes/cmd/kubeadm/app/cmd/upgrade/apply.go:216
k8s.io/kubernetes/cmd/kubeadm/app/cmd/upgrade.runApply
        k8s.io/kubernetes/cmd/kubeadm/app/cmd/upgrade/apply.go:156
k8s.io/kubernetes/cmd/kubeadm/app/cmd/upgrade.newCmdApply.func1
        k8s.io/kubernetes/cmd/kubeadm/app/cmd/upgrade/apply.go:74
github.com/spf13/cobra.(*Command).execute
        github.com/spf13/cobra@v1.6.0/command.go:916
github.com/spf13/cobra.(*Command).ExecuteC
        github.com/spf13/cobra@v1.6.0/command.go:1040
github.com/spf13/cobra.(*Command).Execute
        github.com/spf13/cobra@v1.6.0/command.go:968
k8s.io/kubernetes/cmd/kubeadm/app.Run
        k8s.io/kubernetes/cmd/kubeadm/app/kubeadm.go:50
main.main
        k8s.io/kubernetes/cmd/kubeadm/kubeadm.go:25
runtime.main
        runtime/proc.go:271
runtime.goexit
        runtime/asm_amd64.s:1695
[upgrade/apply] FATAL
k8s.io/kubernetes/cmd/kubeadm/app/cmd/upgrade.runApply
        k8s.io/kubernetes/cmd/kubeadm/app/cmd/upgrade/apply.go:157
k8s.io/kubernetes/cmd/kubeadm/app/cmd/upgrade.newCmdApply.func1
        k8s.io/kubernetes/cmd/kubeadm/app/cmd/upgrade/apply.go:74
github.com/spf13/cobra.(*Command).execute
        github.com/spf13/cobra@v1.6.0/command.go:916
github.com/spf13/cobra.(*Command).ExecuteC
        github.com/spf13/cobra@v1.6.0/command.go:1040
github.com/spf13/cobra.(*Command).Execute
        github.com/spf13/cobra@v1.6.0/command.go:968
k8s.io/kubernetes/cmd/kubeadm/app.Run
        k8s.io/kubernetes/cmd/kubeadm/app/kubeadm.go:50
main.main
        k8s.io/kubernetes/cmd/kubeadm/kubeadm.go:25
runtime.main
        runtime/proc.go:271
runtime.goexit
        runtime/asm_amd64.s:1695
```

#### etcd的logs

`remote error: tls: bad certificate`

```bash
{"level":"warn","ts":"2024-08-23T13:51:24.330935Z","caller":"embed/config_logging.go:169","msg":"rejected connection","remote-addr":"172.18.0.2:41408","server-name":"172.18.0.2","error":"remote error: tls: bad certificate"}
```
查看了相关isuee, 感觉像是etcd证书的问题: 
https://github.com/kubernetes/kubeadm/issues/910
https://github.com/kubernetes/kubernetes/pull/64988
https://github.com/kubernetes/kubeadm/issues/2560

### 排查步骤

#### 小版本升级

最开始看到有的isuee中说可能是跨的版本太大，所以尝试了小版本升级，结果还是报错.

#### 在机器上安装etcdctl，然后查看etcd信息
> 安装etcdctl https://github.com/etcd-io/etcd/releases/tag/v3.5.7
```bash
cp /tmp/etcd-download-test/etcd /tmp/etcd-download-test/etcdctl /usr/local/bin/
etcdctl --endpoints https://127.0.0.1:2379  --cert="/etc/kubernetes/pki/etcd/server.crt"  --key="/etc/kubernetes/pki/etcd/server.key"  --cacert="/etc/kubernetes/pki/etcd/ca.crt" member list -w table
+------------------+---------+--------------------------+-------------------------+-------------------------+------------+
|        ID        | STATUS  |           NAME           |       PEER ADDRS        |      CLIENT ADDRS       | IS LEARNER |
+------------------+---------+--------------------------+-------------------------+-------------------------+------------+
| c9d4f4d293c08711 | started | k8s-test01-control-plane | https://172.18.0.3:2380 | https://172.18.0.2:2379 |      false |
+------------------+---------+--------------------------+-------------------------+-------------------------+------------+
etcdctl --endpoints https://127.0.0.1:2379  --cert="/etc/kubernetes/pki/etcd/server.crt"  --key="/etc/kubernetes/pki/etcd/server.key"  --cacert="/etc/kubernetes/pki/etcd/ca.crt" endpoint health -w table
+------------------------+--------+-------------+-------+
|        ENDPOINT        | HEALTH |    TOOK     | ERROR |
+------------------------+--------+-------------+-------+
| https://127.0.0.1:2379 |   true | 11.957984ms |       |
+------------------------+--------+-------------+-------+
```

#### 确认etcd异常点

从上一步的结果，看到2个异常点
1. PEER ADDRS是172.18.0.3 (原因是因为我使用的kind安装的，重启docker后，容器的ip会变化)
2. IS LEARNER的状态是false (这个好像是因为我是1节点的集群，所以是false)

然后当我尝试使用`172.18.0.2`查看etcd状态时，发现了如下报错

```bash
root@k8s-test01-control-plane:/etc/kubernetes/pki/etcd# etcdctl --endpoints https://172.18.0.2:2379  --cert="/etc/kubernetes/pki/etcd/server.crt"  --key="/etc/kubernetes/pki/etcd/server.key"  --cacert="/etc/kubernetes/pki/etcd/ca.crt" m
ember list -w table
{"level":"warn","ts":"2024-08-23T13:32:36.628Z","logger":"etcd-client","caller":"v3@v3.5.7/retry_interceptor.go:62","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0xc0003de000/172.18.0.2:2379","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = latest balancer error: last connection error: connection error: desc = \"transport: authentication handshake failed: x509: certificate is valid for 172.18.0.3, 127.0.0.1, ::1, not 172.18.0.2\""} 
Error: context deadline exceeded
```

#### 确认etcd证书配置的DNS

```bash
openssl x509 -in /etc/kubernetes/pki/etcd/server.crt -noout -text
……
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Authority Key Identifier:
                keyid:9F:3B:70:71:EF:A2:A1:DA:F4:0F:87:80:D1:A0:BA:31:A5:EF:FB:3B

            X509v3 Subject Alternative Name:
                DNS:k8s-test01-control-plane, DNS:localhost, IP Address:172.18.0.3, IP Address:127.0.0.1, IP Address:0:0:0:0:0:0:0:1
……
```

所以可以确认是证书的问题，因为最开始容器的IP是172.18.0.3，集群在etcd的证书中只允许了 172.18.0.3, 127.0.0.1, ::1这三个地址，所以访问172.18.0.2的时候，会报错。

### 解决方法

#### 使用cfssl重新生成支持172.18.0.2的证书即可
```bash
# 安装cfssl
curl -s -L -o /usr/local/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
curl -s -L -o /usr/local/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x /usr/local/bin/{cfssl,cfssljson}
# 生成证书
cd /etc/kubernetes/pki
mkdir cfssl
cfssl print-defaults config > cfssl/ca-config.json

cd /etc/kubernetes/pki/etcd
echo '{"CN":"k8s-test01-control-plane","hosts":[""],"key":{"algo":"rsa","size":2048}}' | cfssl gencert -ca=ca.crt -ca-key=ca.key  -profile=etcd-server -hostname="k8s-test01-control-plane,localhost,172.18.0.3,172.18.0.2,127.0.0.1,0:0:0:0:0:0:0:1" - | cfssljson -bare etcd-server
```

#### 替换集群证书
```bash
cd /etc/kubernetes/pki/etcd
mkdir server/
mv server.crt server.crt.txt server.key server/

mv etcd-server-key.pem server.key
mv etcd-server.pem server.crt
```

#### 重启ETCD
```bash
mv /etc/kubernetes/manifests/etcd.yaml ~/etcd.yaml && sleep 3 && mv ~/etcd.yaml /etc/kubernetes/manifests/etcd.yaml
```

然后再次升级集群正常

## 参考文档

etcdctl安装: https://github.com/etcd-io/etcd/releases/tag/v3.5.7

etcd TLS: https://etcd.io/docs/v3.5/op-guide/security/

cfssl生成证书: https://github.com/coreos/docs/blob/master/os/generate-self-signed-certificates.md

cfssl仓库地址: https://github.com/cloudflare/cfssl.git

节点更换IP: https://www.qikqiak.com/post/how-to-change-k8s-node-ip/

