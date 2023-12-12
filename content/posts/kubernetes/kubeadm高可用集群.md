---
title: "Kubeadm高可用集群"
subtitle: ""
date: 2023-11-30T17:56:35+08:00
lastmod: 2023-12-04T13:40:35+08:00
draft: false
author: ""
authorLink: ""
license: ""

tags: 
- "kubernetes"
- "linux"
- "haproxy"
- "keepalived"
- "高可用"
categories: 
- "kubernetes"
- "documentation"

featuredImage: ""
featuredImagePreview: ""

summary: ""

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
---
# kubeadm高可用集群

## 基础环境初始化

### 安装Docker,Containerd

#### 配置先决条件

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 应用 sysctl 参数而不重新启动
sudo sysctl --system
```

通过运行以下指令确认 br_netfilter 和 overlay 模块被加载：

```bash
lsmod | grep br_netfilter
lsmod | grep overlay
```

通过运行以下指令确认 net.bridge.bridge-nf-call-iptables、net.bridge.bridge-nf-call-ip6tables 和 net.ipv4.ip_forward 系统变量在你的 sysctl 配置中被设置为 1：

```bash
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

#### 安装docker，containerd

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
rm -f /etc/containerd/config.toml
systemctl start docker
```

#### 修改containerd配置
>报错: no master IPs were listed in storage, refusing to erase all endpoints for the kubernetes service

> 需要将SystemdCgroup修改为true: https://github.com/kubernetes/kubeadm/issues/1047

```bash
containerd config default > /etc/containerd/config.toml
sed -i 's#SystemdCgroup = false#SystemdCgroup = true#g' /etc/containerd/config.toml
```

### 安装kubeadm

#### 关闭swap

如果 kubelet 未被正确配置使用交换分区，则你必须禁用交换分区。 例如，`sudo swapoff -a` 将暂时禁用交换分区。要使此更改在重启后保持不变，请确保在如 `/etc/fstab`、`systemd.swap` 等配置文件中禁用交换分区，具体取决于你的系统如何配置。

#### 安装

```powershell
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# 将 SELinux 设置为 permissive 模式（相当于将其禁用）
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet-1.27.6 kubeadm-1.27.6 kubectl-1.27.6 --disableexcludes=kubernetes

sudo systemctl enable --now kubelet

```

## 使用内部etcd构建高可用集群



### kube-apiserver负载均衡器

> 使用haproxy+keepalived的方式做apiserver高可用

| 机器 | IP | 安装步骤 |
| :----: | :--: | :------------:|
| master1 | 192.168.25.3 | kubeadm安装 |
| master2 | 192.168.25.5 | kubeadm安装 |
| node1 | 192.168.25.4 | kubeadm安装 |
| VIP | 192.168.25.6 | kubeadm安装 |

#### 配置haproxy

1. 配置kubelet的静态pod yaml文件
```bash
cat /etc/kubernetes/manifests/haproxy.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: haproxy
  namespace: kube-system
spec:
  containers:
  #- image: haproxy:2.1.4
  - image: haproxy:2.8.4
    name: haproxy
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: localhost
        path: /healthz
        port: 30033
        scheme: HTTPS
    volumeMounts:
    - mountPath: /usr/local/etc/haproxy/haproxy.cfg
      name: haproxyconf
      readOnly: true
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/haproxy/haproxy.cfg
      type: FileOrCreate
    name: haproxyconf
```

2. 配置haproxy配置
```bash
cat /etc/haproxy/haproxy.cfg 
# /etc/haproxy/haproxy.cfg
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log stdout local0
    log stdout local1 notice
    daemon
    maxconn 75000

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 1
    timeout http-request    10s
    timeout queue           20s
    timeout connect         5s
    timeout client          20s
    timeout server          20s
    timeout http-keep-alive 10s
    timeout check           10s

#---------------------------------------------------------------------
# apiserver frontend which proxys to the control plane nodes
#---------------------------------------------------------------------
frontend apiserver
    bind *:30033
    mode tcp
    option tcplog
    default_backend apiserver

#---------------------------------------------------------------------
# round robin balancing for apiserver
#---------------------------------------------------------------------
backend apiserver
    option httpchk GET /healthz
    http-check expect status 200
    mode tcp
    option ssl-hello-chk
    balance     roundrobin
        server master1 192.168.25.3:6443 check
        server master2 192.168.25.5:6443 check
```

#### 配置keepalived

1. 配置kubelet管理静态POD文件
```bash
cat /etc/kubernetes/manifests/keepalived.yaml 
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: keepalived
  namespace: kube-system
spec:
  containers:
  - image: osixia/keepalived:stable
    name: keepalived
    resources: {}
    args:
    # Fix docker mounted file problems: https://github.com/osixia/docker-keepalived#fix-docker-mounted-file-problems
    - --copy-service
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
        - NET_BROADCAST
        - NET_RAW
    volumeMounts:
    - mountPath: /container/service/keepalived/assets/keepalived.conf
      name: config
    - mountPath: /container/service/keepalived/assets/check_apiserver.sh
      name: check
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/keepalived/keepalived.conf
    name: config
  - hostPath:
      path: /etc/keepalived/check_apiserver.sh
    name: check
```

2. 配置keepalived
```bash
cat /etc/keepalived/keepalived.conf
! /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_apiserver {
  script "/container/service/keepalived/assets/check_apiserver.sh"
  interval 3
  weight -20
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id  51
    priority 100
    authentication {
        auth_type PASS
        auth_pass  private
    }
    virtual_ipaddress {
        192.168.25.6/26 dev eth0 label eth0:1
    }
    unicast_src_ip 192.168.25.3
    unicast_peer {
        192.168.25.5
    }
    track_script {
        check_apiserver
    }
}

```

3. 配置检测脚本
```bash
cat /etc/keepalived/check_apiserver.sh 
#!/bin/sh

errorExit() {
    echo "*** $*" 1>&2
    exit 1
}

curl --silent --max-time 2 --insecure https://localhost:30033/ -o /dev/null || errorExit "Error GET https://localhost:30033/"
if ip addr | grep -q 192.168.25.6; then
    curl --silent --max-time 2 --insecure https://192.168.25.6:30033/ -o /dev/null || errorExit "Error GET https://192.168.25.6:30033/"
fi
```

#### 初始化集群

```bash
kubeadm init --image-repository "registry.aliyuncs.com/google_containers"  --control-plane-endpoint 192.168.25.6:30033 --upload-certs  --kubernetes-version=v1.27.6 --pod-network-cidr=10.20.0.0/16
[init] Using Kubernetes version: v1.27.6
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
W1201 11:29:57.646009   28996 checks.go:835] detected that the sandbox image "registry.k8s.io/pause:3.6" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.aliyuncs.com/google_containers/pause:3.9" as the CRI sandbox image.
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local master1] and IPs [10.96.0.1 192.168.25.3 192.168.25.6]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost master1] and IPs [192.168.25.3 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost master1] and IPs [192.168.25.3 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
W1201 11:30:00.360245   28996 endpoint.go:57] [endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "admin.conf" kubeconfig file
W1201 11:30:00.598039   28996 endpoint.go:57] [endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "kubelet.conf" kubeconfig file
W1201 11:30:00.742556   28996 endpoint.go:57] [endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
W1201 11:30:00.863785   28996 endpoint.go:57] [endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 7.130379 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
968910d19fbe756ab6a45f000191b31fb547e39e94255044d28e7974f5a1ed8e
[mark-control-plane] Marking the node master1 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node master1 as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: 7aaff4.za47au843za5vr8y
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
W1201 11:30:10.798592   28996 endpoint.go:57] [endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 192.168.25.6:30033 --token 7aaff4.za47au843za5vr8y \
	--discovery-token-ca-cert-hash sha256:8e833e46735247bdd1a49434a6b1dcf646d44aa94643632c524e142afb749398 \
	--control-plane --certificate-key 968910d19fbe756ab6a45f000191b31fb547e39e94255044d28e7974f5a1ed8e

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.25.6:30033 --token 7aaff4.za47au843za5vr8y \
	--discovery-token-ca-cert-hash sha256:8e833e46735247bdd1a49434a6b1dcf646d44aa94643632c524e142afb749398 
```

#### kubectl配置
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
source <(kubectl completion bash | tee /usr/share/bash-completion/completions/kubectl)
```

#### 配置CNi插件（flannel）
> https://github.com/flannel-io/flannel#deploying-flannel-with-kubectl
```bash
wget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
sed -i 's@10.244.0.0/16@10.20.0.0/16@g' kube-flannel.yml
kubectl apply -f kube-flannel.yml
```

#### 其他control-plane节点加入集群
keepalived、haproxy使用相同的配置

1. 配置keepalived
```bash
cat /etc/keepalived/keepalived.conf
! /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_apiserver {
  script "/container/service/keepalived/assets/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id  51
    priority 95
    authentication {
        auth_type PASS
        auth_pass  private
    }
    virtual_ipaddress {
        192.168.25.6/26 dev eth0 label eth0:1
    }
    unicast_src_ip 192.168.25.5
    unicast_peer {
        192.168.25.3
    }
    track_script {
        check_apiserver
    }
}

```
2. 生成join命令（前边初始化时会自动生成，但是有使用时限，过期后需要重新生成）
① 上传证书, 生成certificate key
```bash
kubeadm init phase upload-certs --upload-certs 
I1201 13:57:40.241681   87123 version.go:256] remote version is much newer: v1.28.4; falling back to: stable-1.27
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
c0e3f2c1807645736a3489074f0f3b2f53773e5242a4110ae14cf89ffdb82105
```
② 使用certificate key，生成control join命令
```bash
kubeadm token create --print-join-command --certificate-key c0e3f2c1807645736a3489074f0f3b2f53773e5242a4110ae14cf89ffdb82105
kubeadm join 192.168.25.6:30033 --token 4ctcsl.f3owcw6glcqci0d7 --discovery-token-ca-cert-hash sha256:8e833e46735247bdd1a49434a6b1dcf646d44aa94643632c524e142afb749398 --control-plane --certificate-key c0e3f2c1807645736a3489074f0f3b2f53773e5242a4110ae14cf89ffdb82105
```

3. master2加入集群
```bash
kubeadm join 192.168.25.6:30033 --token 4ctcsl.f3owcw6glcqci0d7 --discovery-token-ca-cert-hash sha256:8e833e46735247bdd1a49434a6b1dcf646d44aa94643632c524e142afb749398 --control-plane --certificate-key c0e3f2c1807645736a3489074f0f3b2f53773e5242a4110ae14cf89ffdb82105
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks before initializing the new control plane instance
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
W1201 14:14:24.499526   74685 checks.go:835] detected that the sandbox image "registry.k8s.io/pause:3.6" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.aliyuncs.com/google_containers/pause:3.9" as the CRI sandbox image.
[download-certs] Downloading the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[download-certs] Saving the certificates to the folder: "/etc/kubernetes/pki"
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost master2] and IPs [192.168.25.5 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost master2] and IPs [192.168.25.5 127.0.0.1 ::1]
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local master2] and IPs [10.96.0.1 192.168.25.5 192.168.25.6]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[certs] Using the existing "sa" key
[kubeconfig] Generating kubeconfig files
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
W1201 14:14:26.255314   74685 endpoint.go:57] [endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "admin.conf" kubeconfig file
W1201 14:14:26.376856   74685 endpoint.go:57] [endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
W1201 14:14:26.690997   74685 endpoint.go:57] [endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[check-etcd] Checking that the etcd cluster is healthy
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
[etcd] Announced new etcd member joining to the existing etcd cluster
[etcd] Creating static Pod manifest for "etcd"
[etcd] Waiting for the new etcd member to join the cluster. This can take up to 40s
The 'update-status' phase is deprecated and will be removed in a future release. Currently it performs no operation
[mark-control-plane] Marking the node master2 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node master2 as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]

This node has joined the cluster and a new control plane instance was created:

* Certificate signing request was sent to apiserver and approval was received.
* The Kubelet was informed of the new secure connection details.
* Control plane label and taint were applied to the new node.
* The Kubernetes control plane instances scaled up.
* A new etcd member was added to the local/stacked etcd cluster.

To start administering your cluster from this node, you need to run the following as a regular user:

	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster.

```

3. 查看状态
```bash
kubectl get node
NAME      STATUS   ROLES           AGE     VERSION
master1   Ready    control-plane   3h23m   v1.27.6
master2   Ready    control-plane   38m     v1.27.6

kubectl -n kube-system exec -it etcd-master1 -- etcdctl --endpoints https://127.0.0.1:2379  --cert="/etc/kubernetes/pki/etcd/server.crt"  --key="/etc/kubernetes/pki/etcd/server.key"  --cacert="/etc/kubernetes/pki/etcd/ca.crt" endpoint status --cluster  -w table
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://192.168.25.3:2379 |  2cbae31ee780d9e |   3.5.7 |  2.8 MB |      true |      false |         5 |      20066 |              20066 |        |
| https://192.168.25.5:2379 | 9c27a5e8c0b28da0 |   3.5.7 |  2.8 MB |     false |      false |         5 |      20066 |              20066 |        |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+

```

#### 其他node节点加入集群
1. 生成join命令
```bash
kubeadm token create --print-join-command 
kubeadm join 192.168.25.6:30033 --token 2fsx9v.ccid3z5hhqzfyall --discovery-token-ca-cert-hash sha256:8e833e46735247bdd1a49434a6b1dcf646d44aa94643632c524e142afb749398
```

2. node节点加入集群
```bash
kubeadm join 192.168.25.6:30033 --token 2fsx9v.ccid3z5hhqzfyall --discovery-token-ca-cert-hash sha256:8e833e46735247bdd1a49434a6b1dcf646d44aa94643632c524e142afb749398
[preflight] Running pre-flight checks
	[WARNING Hostname]: hostname "node1" could not be reached
	[WARNING Hostname]: hostname "node1": lookup node1 on 100.100.2.136:53: no such host
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

```

### 其他问题

#### kubectl使用watch操作20秒就会断开
> 原因是因为haproxy设置的20超时，但是kubectl默认的 idle timeout是30秒，将kubectl的idel timeout配置为15秒即可. issue: https://github.com/kubernetes/kubeadm/issues/2227
```bash
export HTTP2_READ_IDLE_TIMEOUT_SECONDS=15
```

#### v1.27移除--pod-eviction-timeout后怎么修改驱逐POD时长
> https://kubernetes.io/zh-cn/blog/2023/03/17/upcoming-changes-in-kubernetes-v1-27/#%E7%A7%BB%E9%99%A4-pod-eviction-timeout-%E5%91%BD%E4%BB%A4%E8%A1%8C%E5%8F%82%E6%95%B0

> 使用污点方式: https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/taint-and-toleration/#taint-based-evictions

```bash
      tolerations:
      - effect: NoExecute
        key: node.kubernetes.io/unreachable
        operator: Exists
        tolerationSeconds: 120
```

#### 下线节点
```bash
kubectl cordon node1
kubectl drain --ignore-daemonsets node1 # --ignore-daemonsets 选项为忽略DaemonSet类型的pod

```

## 参考文档
- docker安装官方文档: https://docs.docker.com/engine/install/centos/
- 容器运行时: https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/
- 安装 kubeadm: https://v1-27.docs.kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- 高可用拓扑选项: https://v1-27.docs.kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/ha-topology/ 
- 利用 kubeadm 创建高可用集群: https://v1-27.docs.kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/high-availability/
- 阿里云使用keepalived: https://developer.aliyun.com/article/438705