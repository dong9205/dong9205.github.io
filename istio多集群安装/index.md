# istio多集群安装


## 一、istio多集群模型介绍
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

## 二、准备工作

### 创建两个集群

```
kind create cluster --name cluster01
# kind默认在kubeconfig中生成的地址是https://127.0.0.1:xxxxx，需要把地址改为容器的IP地址，否则两个集群无法访问
kubectl config set-cluster kind-cluster01 --server=https://$(docker inspect -f '{{.NetworkSettings.Networks.kind.IPAddress}}' cluster01-control-plane):6443
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
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: empty
  namespace: metallb-system
EOF

kind create cluster --name cluster02
kubectl config set-cluster kind-cluster02 --server=https://$(docker inspect -f '{{.NetworkSettings.Networks.kind.IPAddress}}' cluster02-control-plane):6443
cat <<EOF | kubectl create -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: custom-172.168.12.0-255
  namespace: metallb-system
spec:
  addresses:
  - 172.18.12.0-172.18.12.255
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: empty
  namespace: metallb-system
EOF
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
```

### 配置信任关系

> 默认情况下，**Istio CA 会生成一个自签名的根证书和密钥，并使用它们来签署工作负载证书。** 为了保护根 CA 密钥，您应该使用在安全机器上离线运行的根 CA，并**使用根 CA 向运行在每个集群上的 Istio CA 签发中间证书**。Istio CA 可以使用管理员指定的证书和密钥来签署工作负载证书， 并将管理员指定的根证书作为信任根分配给工作负载。

下图展示了在包含两个集群的网格中推荐的 CA 层次结构。

{{< figure src="CA层次结构.svg" title="CA层次结构" >}}

* Makefile.k8s.mk：基于 k8s 集群中的 root-ca 创建证书。默认 kubeconfig 中的当前上下文用于访问集群。
* Makefile.selfsigned.mk：基于生成的自签名根创建证书。

下表描述了两个 Makefile 支持的目标
 | <div style="width: 106pt">Make Target | <div style="width: 146pt">Makefile | Description | 
 | :------ | :-------- | :----------- | 
 | `root-ca` | `Makefile.selfsigned.mk` | 生成自签名根 CA 密钥和证书. | 
 | `fetch-root-ca` | `Makefile.k8s.mk` | 使用默认 `kubeconfig` 中的当前上下文从 Kubernetes 集群获取 Istio CA. | 
 | `$NAME-cacerts` | Both | 为具有 `$NAME` 的集群或虚拟机（例如 `us-east`、`cluster01` 等）生成由根 CA 签名的中间证书。它们存储在 `$NAME` 目录下。为了区分集群，我们在证书`主题`字段中包含`位置` (`L`) 名称以及集群名称。| 
 | `$NAMESPACE-certs` | Both | 使用根证书为使用 serviceAccount `$SERVICE_ACCOUNT` 连接到命名空间 `$NAMESPACE` 的虚拟机生成中间证书和签名证书，并将它们存储在 `$NAMESPACE` 目录下。 | 
 | `clean` | Both | 删除任何生成的根证书、密钥和中间文件。. | 

#### 创建一个目录来存放证书和密钥
```bash
 mkdir -p certs
 pushd certs
```

#### 生成根CA证书

```bash
 make -f ../tools/certs/Makefile.selfsigned.mk root-ca
```
将会生成以下文件：

* root-cert.pem：生成的根证书
* root-key.pem：生成的根密钥
* root-ca.conf：生成根证书的 openssl 配置
* root-cert.csr：为根证书生成的 CSR

#### 为cluster生成证书
```bash
 make -f ../tools/certs/Makefile.selfsigned.mk cluster01-cacerts
 make -f ../tools/certs/Makefile.selfsigned.mk cluster02-cacerts
```
运行以上命令，将会在名为 cluster01、cluster02 的目录下生成以下文件：

* ca-cert.pem：生成的中间证书
* ca-key.pem：生成的中间密钥
* cert-chain.pem：istiod 使用的生成的证书链
* root-cert.pem：根证书

#### 创建cacerts Secret 
```bash
 kubectl --context kind-cluster01 create namespace istio-system
 kubectl --context kind-cluster01 create secret generic cacerts -n istio-system \
    --from-file=cluster01/ca-cert.pem \
    --from-file=cluster01/ca-key.pem \
    --from-file=cluster01/root-cert.pem \
    --from-file=cluster01/cert-chain.pem

kubectl --context kind-cluster02 create namespace istio-system
kubectl --context kind-cluster02 create secret generic cacerts -n istio-system \
     --from-file=cluster02/ca-cert.pem \
     --from-file=cluster02/ca-key.pem \
     --from-file=cluster02/root-cert.pem \
     --from-file=cluster02/cert-chain.pem
```

## 三、istio多集群部署

### 扁平网络单控制面(主从架构)
该模型下只需要将 Istio 控制面组件部署在主集群中，然后可以通过这个控制面来管理所有集群的 Service 和 Endpoint，其他的 Istio 相关的 API 比如 VirtualService、DestinationRule 等也只需要在主集群中配置即可，其他集群不需要部署 Istio 控制面组件。

控制平面的 Istiod 核心组件负责连接所有集群的 kube-apiserver，获取每个集群的 Service、Endpoint、Pod 等信息，所有集群的 Sidecar 均连接到这个中心控制面，由这个中心控制面负责所有的 Envoy Sidecar 的配置生成和分发。

{{< figure src="主从架构的安装.svg" title="主从架构的安装" >}}

<!-- {{< figure src="扁平网络单控制面.png" title="扁平网络单控制面" >}} -->

多集群扁平网络模型和单一集群的服务网格在访问方式上几乎没什么区别，但是需要注意不同集群的 Service IP 和 Pod 的 IP 不能重叠，否则会导致集群之间的服务发现出现问题，这也是扁平网络模型的一个缺点，需要提前规划好集群的网段。



### 扁平网络多控制面(多主架构)



### 非扁平网络单控制面(跨网络主从架构)

### 非扁平网络多控制面(跨网络多主架构)

{{< figure src="跨网络的多主集群.svg" title="跨网络的多主集群" >}}

#### 为 cluster01 设置缺省网络
```bash
kubectl --context="kind-cluster01" label namespace istio-system topology.istio.io/network=network1
```

#### 将 cluster01 设为主集群
```bash
cat <<EOF > cluster01.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    accessLogFile: /dev/stdout
    accessLogEncoding: JSON
    accessLogFormat: '{"authority":"%REQ(:AUTHORITY)%","bytes_received":"%BYTES_RECEIVED%","bytes_sent":"%BYTES_SENT%","downstream_local_address":"%DOWNSTREAM_LOCAL_ADDRESS%","downstream_remote_address":"%DOWNSTREAM_REMOTE_ADDRESS%","duration":"%DURATION%","istio_policy_status":"%DYNAMIC_METADATA(istio.mixer:status)%","method":"%REQ(:METHOD)%","path":"%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%","protocol":"%PROTOCOL%","request_id":"%REQ(X-REQUEST-ID)%","requested_server_name":"%REQUESTED_SERVER_NAME%","response_code":"%RESPONSE_CODE%","response_flags":"%RESPONSE_FLAGS%","route_name":"%ROUTE_NAME%","start_time":"%START_TIME%","trace_id":"%REQ(X-B3-TRACEID)%","upstream_cluster":"%UPSTREAM_CLUSTER%","upstream_host":"%UPSTREAM_HOST%","upstream_local_address":"%UPSTREAM_LOCAL_ADDRESS%","upstream_service_time":"%RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)%","upstream_transport_failure_reason":"%UPSTREAM_TRANSPORT_FAILURE_REASON%","user_agent":"%REQ(USER-AGENT)%","x_forwarded_for":"%REQ(X-FORWARDED-FOR)%"}'
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster01
      network: network1
      logAsJson: true
EOF
```
将配置文件应用到 cluster01：

`istioctl install --context="kind-cluster01" -f cluster01.yaml -y`

#### 在 cluster01 安装东西向网关
在 cluster01 安装专用的 [东西向](https://en.wikipedia.org/wiki/East-west_traffic)网关。 默认情况下，此网关将被公开到互联网上。 生产系统可能需要添加额外的访问限制（即：通过防火墙规则）来防止外部攻击。 咨询您的云服务商，了解可用的选择。

```bash
/root/istio-1.18.2/samples/multicluster/gen-eastwest-gateway.sh --mesh mesh1 --cluster cluster01 --network network1 | istioctl --context=kind-cluster01 install -y -f -
```

> 如果控制面已经安装了一个修订版，可以在 gen-eastwest-gateway.sh 命令中添加 --revision rev 标志。


等待东西向网关被分配外部 IP 地址：
```bash
kubectl --context="kind-cluster01" get svc istio-eastwestgateway -n istio-system
NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                           AGE
istio-eastwestgateway   LoadBalancer   10.96.135.178   172.18.11.1   15021:31259/TCP,15443:30115/TCP,15012:30290/TCP,15017:30995/TCP   61s
```

#### 开放 cluster01 中的服务
因为集群位于不同的网络中，所以我们需要在两个集群东西向网关上开放所有服务（*.local）。 虽然此网关在互联网上是公开的，但它背后的服务只能被拥有可信 mTLS 证书、工作负载 ID 的服务访问， 就像它们处于同一网络一样。

`kubectl --context="kind-cluster01" apply -n istio-system -f /root/istio-1.18.2/samples/multicluster/expose-services.yaml`

#### 为 cluster02 设置缺省网络
```bash
kubectl --context="kind-cluster02" label namespace istio-system topology.istio.io/network=network2
```

#### 将 cluster02 设为主集群
```bash
cat <<EOF > cluster02.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    accessLogFile: /dev/stdout
    accessLogEncoding: JSON
    accessLogFormat: '{"authority":"%REQ(:AUTHORITY)%","bytes_received":"%BYTES_RECEIVED%","bytes_sent":"%BYTES_SENT%","downstream_local_address":"%DOWNSTREAM_LOCAL_ADDRESS%","downstream_remote_address":"%DOWNSTREAM_REMOTE_ADDRESS%","duration":"%DURATION%","istio_policy_status":"%DYNAMIC_METADATA(istio.mixer:status)%","method":"%REQ(:METHOD)%","path":"%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%","protocol":"%PROTOCOL%","request_id":"%REQ(X-REQUEST-ID)%","requested_server_name":"%REQUESTED_SERVER_NAME%","response_code":"%RESPONSE_CODE%","response_flags":"%RESPONSE_FLAGS%","route_name":"%ROUTE_NAME%","start_time":"%START_TIME%","trace_id":"%REQ(X-B3-TRACEID)%","upstream_cluster":"%UPSTREAM_CLUSTER%","upstream_host":"%UPSTREAM_HOST%","upstream_local_address":"%UPSTREAM_LOCAL_ADDRESS%","upstream_service_time":"%RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)%","upstream_transport_failure_reason":"%UPSTREAM_TRANSPORT_FAILURE_REASON%","user_agent":"%REQ(USER-AGENT)%","x_forwarded_for":"%REQ(X-FORWARDED-FOR)%"}'
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster02
      network: network2
      logAsJson: true
EOF
```
将配置文件应用到 cluster02：

`istioctl install --context="kind-cluster02" -f cluster02.yaml -y`

#### 在 cluster02 安装东西向网关
在 cluster02 安装专用的 [东西向](https://en.wikipedia.org/wiki/East-west_traffic)网关。 默认情况下，此网关将被公开到互联网上。 生产系统可能需要添加额外的访问限制（即：通过防火墙规则）来防止外部攻击。 咨询您的云服务商，了解可用的选择。

```bash
/root/istio-1.18.2/samples/multicluster/gen-eastwest-gateway.sh --mesh mesh1 --cluster cluster02 --network network2 | istioctl --context=kind-cluster02 install -y -f -
```

等待东西向网关被分配外部 IP 地址：
```bash
kubectl --context="kind-cluster02" get svc istio-eastwestgateway -n istio-system
NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                           AGE
istio-eastwestgateway   LoadBalancer   10.96.139.160   172.18.12.1   15021:32493/TCP,15443:30897/TCP,15012:31854/TCP,15017:30771/TCP   8s
```

#### 开放 cluster02 中的服务
因为集群位于不同的网络中，所以我们需要在两个集群东西向网关上开放所有服务（*.local）。 虽然此网关在互联网上是公开的，但它背后的服务只能被拥有可信 mTLS 证书、工作负载 ID 的服务访问， 就像它们处于同一网络一样。

```bash
kubectl --context="kind-cluster02" apply -n istio-system -f /root/istio-1.18.2/samples/multicluster/expose-services.yaml
```

#### 启用端点发现

在 cluster02 中安装一个提供 cluster01 API Server 访问权限的远程 Secret。

```bash
istioctl create-remote-secret \
  --context="kind-cluster01" \
  --name=cluster01 | \
  kubectl apply -f - --context="kind-cluster02"
```

在 cluster01 中安装一个提供 cluster02 API Server 访问权限的远程 Secret。

```bash
istioctl create-remote-secret \
  --context="kind-cluster02" \
  --name=cluster02 | \
  kubectl apply -f - --context="kind-cluster01"

```

### 验证安装结果

我们将在 cluster01 安装 V1 版的 HelloWorld 应用程序， 在 cluster02 安装 V2 版的 HelloWorld 应用程序。 当处理一个请求时，HelloWorld 会在响应消息中包含它自身的版本号。

我们也会在两个集群中均部署 Sleep 容器。 这些 Pod 将被用作客户端（source），发送请求给 HelloWorld。 最后，通过收集这些流量数据，我们将能观测并识别出是那个集群处理了请求。

#### 部署服务 `HelloWorld`
为了支持从任意集群中调用 `HelloWorld` 服务，每个集群的 DNS 解析必须可用 （详细信息，参见[部署模型](https://istio.io/latest/zh/docs/ops/deployment/deployment-models#dns-with-multiple-clusters) 7）。 我们通过在网格的每一个集群中部署 `HelloWorld` 服务，来解决这个问题，

首先，在每个集群中创建命名空间 `sample`：

```bash
kubectl create --context="kind-cluster01" namespace sample
kubectl create --context="kind-cluster02" namespace sample

```
为命名空间 `sample` 开启 sidecar 自动注入：

```bash
kubectl label --context="kind-cluster01" namespace sample \
    istio-injection=enabled
kubectl label --context="kind-cluster02" namespace sample \
    istio-injection=enabled

```
在每个集群中创建 `HelloWorld` 服务：

```bash
kubectl apply --context="kind-cluster01" \
    -f /root/istio-1.18.2/samples/helloworld/helloworld.yaml \
    -l service=helloworld -n sample
kubectl apply --context="kind-cluster02" \
    -f /root/istio-1.18.2/samples/helloworld/helloworld.yaml \
    -l service=helloworld -n sample

```

#### 部署 V1 版的 HelloWorld

```bash
kubectl apply --context="kind-cluster01" \
    -f  /root/istio-1.18.2/samples/helloworld/helloworld.yaml \
    -l version=v1 -n sample

```

确认 helloworld-v1 pod 的状态：

```bash
kubectl get pod --context="kind-cluster01" -n sample -l app=helloworld
```

等待 helloworld-v1 的状态最终变为 Running 状态：



#### 部署 V2 版的 HelloWorld

把应用 helloworld-v2 部署到 cluster02：

```bash
kubectl apply --context="kind-cluster02" \
    -f /root/istio-1.18.2/samples/helloworld/helloworld.yaml \
    -l version=v2 -n sample

```

确认 helloworld-v2 pod 的状态：

```bash
kubectl get pod --context="kind-cluster02" -n sample -l app=helloworld

```

等待 helloworld-v2 的状态最终变为 Running 状态：

#### 部署 Sleep

把应用 Sleep 部署到每个集群：

```bash
kubectl apply --context="kind-cluster01" \
    -f /root/istio-1.18.2/samples/sleep/sleep.yaml -n sample
kubectl apply --context="kind-cluster02" \
    -f /root/istio-1.18.2/samples/sleep/sleep.yaml -n sample

```
确认 cluster01 上 Sleep 的状态：

`kubectl get pod --context="kind-cluster01" -n sample -l app=sleep`

等待 Sleep 的状态最终变为 Running 状态：

确认 cluster02 上 Sleep 的状态：

`kubectl get pod --context="kind-cluster02" -n sample -l app=sleep`

等待 Sleep 的状态最终变为 Running 状态：

#### 验证跨集群流量

要验证跨集群负载均衡是否按预期工作，需要用 `Sleep` **pod** 重复调用服务 `HelloWorld`。 为了确认负载均衡按预期工作，需要从所有集群调用服务 `HelloWorld`。

从 `cluster01` 中的 `Sleep` **pod** 发送请求给服务 `HelloWorld`, 重复几次这个请求，验证 HelloWorld 的版本在 v1 和 v2 之间切换：

```bash
for i in $(seq 10);do kubectl exec --context="kind-cluster01" -n sample -c sleep "$(kubectl get pod --context="kind-cluster01" -n sample -l app=sleep -o jsonpath='{.items[0].metadata.name}')" -- curl -s helloworld.sample:5000/hello;done
Hello version: v1, instance: helloworld-v1-94b6f7986-twskx
Hello version: v2, instance: helloworld-v2-f976ddc5b-c6ptz
Hello version: v2, instance: helloworld-v2-f976ddc5b-c6ptz
Hello version: v2, instance: helloworld-v2-f976ddc5b-c6ptz
Hello version: v2, instance: helloworld-v2-f976ddc5b-c6ptz
Hello version: v2, instance: helloworld-v2-f976ddc5b-c6ptz
Hello version: v1, instance: helloworld-v1-94b6f7986-twskx
Hello version: v1, instance: helloworld-v1-94b6f7986-twskx
Hello version: v1, instance: helloworld-v1-94b6f7986-twskx
Hello version: v2, instance: helloworld-v2-f976ddc5b-c6ptz
```

现在，用 cluster02 中的 Sleep pod 重复此过程，重复几次这个请求，验证 HelloWorld 的版本在 v1 和 v2 之间切换：

```bash
for i in $(seq 10);do kubectl exec --context="kind-cluster02" -n sample -c sleep "$(kubectl get pod --context="kind-cluster02" -n sample -l app=sleep -o jsonpath='{.items[0].metadata.name}')" -- curl -s helloworld.sample:5000/hello;done
Hello version: v2, instance: helloworld-v2-f976ddc5b-c6ptz
Hello version: v2, instance: helloworld-v2-f976ddc5b-c6ptz
Hello version: v2, instance: helloworld-v2-f976ddc5b-c6ptz
Hello version: v1, instance: helloworld-v1-94b6f7986-twskx
Hello version: v1, instance: helloworld-v1-94b6f7986-twskx
Hello version: v1, instance: helloworld-v1-94b6f7986-twskx
Hello version: v2, instance: helloworld-v2-f976ddc5b-c6ptz
Hello version: v1, instance: helloworld-v1-94b6f7986-twskx
Hello version: v2, instance: helloworld-v2-f976ddc5b-c6ptz
Hello version: v1, instance: helloworld-v1-94b6f7986-twskx
```





## 参考文档

官方文档: [多集群安装](https://istio.io/latest/zh/docs/setup/install/multicluster/)

k8s技术圈：[Istio多集群实践](https://mp.weixin.qq.com/s/tPxw48np5bAaTeumUET1fA)

metallb: [Installation By Manifest](https://metallb.universe.tf/installation/#installation-by-manifest)
