# istio多集群安装


# 一、istio多集群模型介绍
Istio 多集群网格有多种模型，在网络拓扑上分为扁平网络和非扁平网络，在控制面上分为单一控制平面和多控制平面。

Istio 多集群网格有多种模型，在网络拓扑上分为扁平网络和非扁平网络，在控制面上分为单一控制平面和多控制平面。

*   扁平网络：所有集群都在同一个网络中，可以直接访问到其他集群的服务，不需要通过网关。
    *   优点： 跨集群访问不经过东西向网关，延迟低
    *   缺点：组网较为复杂，Service、Pod 网段不能重叠，借助 VPN 等技术将所有集群的 Pod 网络打通，需要考虑网络安全问题

*   非扁平网络：集群之间的网络是隔离的，需要通过网关访问其他集群的服务。
    *   优点：不同集群的网络是互相隔离的，安全性更高，不需要打通不同集群的容器网络，不用提前规划集群的网段
    *   缺点：跨集群访问依赖东西向网关，延迟高。东西向网关工作模式是 TLS AUTO_PASSTHROUGH，不支持 HTTP 路由策略。
    

*   单控制面：所有集群共用一个控制平面，所有集群的配置都在同一个控制平面中。
    *   优点：所有集群的配置都在同一个控制平面中，集群之间的配置可以共享，部署运维更简单
    *   缺点：控制平面的性能和可用性会受到影响，不适合大规模集群
    

*   多控制面：每个集群都有一个独立的控制平面，集群之间的配置不共享。
    *   优点：控制平面的性能和可用性不会受到影响，适合大规模集群
    *   缺点：集群之间的配置不共享，部署运维较为复杂
    
总体来说 Istio 目前支持 4 种多集群模型：**扁平网络单控制面(主从架构)、扁平网络多控制面(多主架构)、非扁平网络单控制面(跨网络主从架构)、非扁平网络多控制面(跨网络多主架构)**。其中扁平网络单控制面是最简单的模型，非扁平网络多控制面是最复杂的模型。

# 二、准备工作

## 创建两个集群

```
kind create cluster --name cluster01
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
cat <<EOF | kubectl create -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: custom-172.168.11.0-255
  namespace: metallb-system
spec:
  addresses:
  - 172.18.11.0-172.18.11.255
EOF

kind create cluster --name cluster02
cat <<EOF | kubectl create -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: custom-172.168.12.0-255
  namespace: metallb-system
spec:
  addresses:
  - 172.18.12.0-172.18.12.255
EOF
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
```

## 配置信任关系

> 默认情况下，**Istio CA 会生成一个自签名的根证书和密钥，并使用它们来签署工作负载证书。** 为了保护根 CA 密钥，您应该使用在安全机器上离线运行的根 CA，并**使用根 CA 向运行在每个集群上的 Istio CA 签发中间证书**。Istio CA 可以使用管理员指定的证书和密钥来签署工作负载证书， 并将管理员指定的根证书作为信任根分配给工作负载。

下图展示了在包含两个集群的网格中推荐的 CA 层次结构。

{{< figure src="CA层次结构.svg" title="CA层次结构" >}}

```bash
[ 12:52AM ]  [ root@derrick:~/istio-1.18.2 ]
 $ mkdir -p certs
[ 12:52AM ]  [ root@derrick:~/istio-1.18.2 ]
 $ mkdir -p certs
[ 12:52AM ]  [ root@derrick:~/istio-1.18.2 ]
 $ pushd certs
[ 12:52AM ]  [ root@derrick:~/istio-1.18.2/certs ]
 $ make -f ../tools/certs/Makefile.selfsigned.mk root-ca
generating root-key.pem
generating root-cert.csr
generating root-cert.pem
Certificate request self-signature ok
subject=O = Istio, CN = Root CA
[ 12:53AM ]  [ root@derrick:~/istio-1.18.2/certs ]
 $ ls
root-ca.conf  root-cert.csr  root-cert.pem  root-key.pem

```

* Makefile.k8s.mk：基于 k8s 集群中的 root-ca 创建证书。默认 kubeconfig 中的当前上下文用于访问集群。
* Makefile.selfsigned.mk：基于生成的自签名根创建证书。

 | Make Target | <div style="width: 100pt">Makefile | Description | 
 | :------ | :-------- | :----------- | 
 | `root-ca` | `Makefile.selfsigned.mk` | 生成自签名根 CA 密钥和证书. | 
 | `fetch-root-ca` | `Makefile.k8s.mk` | 使用默认 `kubeconfig` 中的当前上下文从 Kubernetes 集群获取 Istio CA. | 
 | `$NAME-cacerts` | Both | 为具有 `$NAME` 的集群或虚拟机（例如 `us-east`、`cluster01` 等）生成由根 CA 签名的中间证书。它们存储在 `$NAME` 目录下。为了区分集群，我们在证书`主题`字段中包含`位置` (`L`) 名称以及集群名称。| 
 | `$NAMESPACE-certs` | Both | 使用根证书为使用 serviceAccount `$SERVICE_ACCOUNT` 连接到命名空间 `$NAMESPACE` 的虚拟机生成中间证书和签名证书，并将它们存储在 `$NAMESPACE` 目录下。 | 
 | `clean` | Both | 删除任何生成的根证书、密钥和中间文件。. | 







# 三、istio多集群部署

## 扁平网络单控制面(主从架构)
该模型下只需要将 Istio 控制面组件部署在主集群中，然后可以通过这个控制面来管理所有集群的 Service 和 Endpoint，其他的 Istio 相关的 API 比如 VirtualService、DestinationRule 等也只需要在主集群中配置即可，其他集群不需要部署 Istio 控制面组件。

控制平面的 Istiod 核心组件负责连接所有集群的 kube-apiserver，获取每个集群的 Service、Endpoint、Pod 等信息，所有集群的 Sidecar 均连接到这个中心控制面，由这个中心控制面负责所有的 Envoy Sidecar 的配置生成和分发。

{{< figure src="主从架构的安装.svg" title="主从架构的安装" >}}

<!-- {{< figure src="扁平网络单控制面.png" title="扁平网络单控制面" >}} -->

多集群扁平网络模型和单一集群的服务网格在访问方式上几乎没什么区别，但是需要注意不同集群的 Service IP 和 Pod 的 IP 不能重叠，否则会导致集群之间的服务发现出现问题，这也是扁平网络模型的一个缺点，需要提前规划好集群的网段。



## 扁平网络多控制面(多主架构)

## 非扁平网络单控制面(跨网络主从架构)

## 非扁平网络多控制面(跨网络多主架构)

{{< figure src="跨网络的多主集群.svg" title="跨网络的多主集群" >}}


# 参考文档

官方文档: [多集群安装](https://istio.io/latest/zh/docs/setup/install/multicluster/)

k8s技术圈：[Istio多集群实践](https://mp.weixin.qq.com/s/tPxw48np5bAaTeumUET1fA)

metallb: [Installation By Manifest](https://metallb.universe.tf/installation/#installation-by-manifest)
