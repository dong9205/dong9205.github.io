---
title: Gateway API 实战（二）：GatewayClass 与 Gateway 全解析 · 绑定公网 IP 完整指南
subtitle:
date: 2025-11-29T01:37:17+08:00
slug: 1e2294222
draft: false
author:
  name: Derrick
  link: https://www.p-pp.cn/
  email: 920506213@qq.com
  avatar:
description:
keywords:
license:
comment: true
weight: 0
tags:
  - kubernetes
  - Gateway API
categories:
  - kubernetes
  - Gateway API
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRelated: false
hiddenFromFeed: false
summary:
resources:
  - name: resource-model
    src: resource-model.png
toc: true
math: false
lightgallery: false
password:
message:
repost:
  enable: false
  url:

# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---

<!--more-->
## 📌概述
在第一章中，我们追溯了 Gateway API 的缘起与使命。  而现在——我们要真正走进它的“心脏”。

本篇将深入介绍 **GatewayClass 与 Gateway 核心字段**  
并带你完成一件真正落地的事情：  🎯 **为 Gateway 绑定可被外部访问的 LoadBalancer IP**
## 🔍 GatewayClass — 标准的诞生地
`GatewayClass` 是集群级别的资源，类似于 Kubernetes 的 `StorageClass`，它定义了一类具有共同配置和行为的网关控制器。每个 `GatewayClass` 由特定的 Gateway Controller 管理。

### ⭐ 核心字段说明

#### 📌`controllerName`（必需字段）
`controllerName` 用于指定负责管理该 `GatewayClass` 的控制器名称。格式通常为 `域名/控制器名称`。

```YAML
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

**说明**：
- 每个 `GatewayClass` 必须有一个唯一的 `controllerName`
- 控制器通过这个字段识别自己需要管理的 `GatewayClass`
- 常见的控制器名称：
  - Envoy Gateway: `gateway.envoyproxy.io/gatewayclass-controller`
  - Istio: `istio.io/gateway-controller`
  - Traefik: `traefik.io/gateway-controller`
  - Nginx: `gateway.nginx.org/nginx-gateway-controller`

#### 📌parametersRef（可选字段）
`parametersRef` 用于引用包含特定配置参数的资源，允许为 `GatewayClass` 提供额外的配置，在实际生产环境中非常有用。引用的资源必须由控制器支持，否则不会生效。

```YAML
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  parametersRef:
    group: gateway.envoyproxy.io
    kind: EnvoyProxy
    name: custom-proxy-config
    namespace: default
```

**说明**：
- `group`: 参数资源的 API 组
- `kind`: 参数资源的类型
- `name`: 参数资源的名称
- `namespace`: 参数资源所在的命名空间（可选）

**实际用途**：可以用于大多数自定义场景，如自定义 Deployment 副本数、镜像、注解、资源限制、目录挂载，Service 注解、Pod 注解等
#### 💬`description`（可选字段）
`description` 用于描述该 `GatewayClass` 的用途和特性。

```YAML
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  description: "Envoy Gateway 控制器，用于生产环境"
```

### 📊 GatewayClass Status
`GatewayClass` 的状态信息，由控制器自动更新：

```YAML
status:
  conditions:
  - type: Accepted
    status: "True"
    reason: Accepted
    message: Valid GatewayClass
    lastTransitionTime: "2025-11-28T15:57:00Z"
```

**状态说明**：
- `Accepted`: 表示控制器是否接受并管理该 `GatewayClass`
- `status`: `True` 表示已接受，`False` 表示未接受
- `reason`: 状态的原因（如 `Accepted`、`InvalidParameters` 等）

## 🚪 Gateway — 连接外界的现实入口
`Gateway` 是命名空间级别的资源，代表一个实际的网关实例。每个 `Gateway` 都会关联一个 `GatewayClass`，并由对应的控制器创建实际的负载均衡器和代理服务。

### ⭐ 核心字段说明

#### 📌`gatewayClassName`（必需字段）
`gatewayClassName` 指定该 `Gateway` 使用的 `GatewayClass` 名称。

```YAML
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: eg
  namespace: default
spec:
  gatewayClassName: eg
```

**说明**：
- 必须引用一个已存在的 `GatewayClass`
- `GatewayClass` 是集群级别的资源（不需要指定 namespace）

#### 📌`listeners`（必需字段）
`listeners` 定义网关监听的端口和协议列表。

```YAML
spec:
  gatewayClassName: eg
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            group: ""
            name: example-cert
            namespace: envoy-gateway-system
      allowedRoutes:
        namespaces:
          from: All
```

**Listener 字段详解**：
- `name`: 监听器名称，在同一 `Gateway` 中必须唯一
- `protocol`: 协议类型（`HTTP`、`HTTPS`、`TLS`、`TCP`、`UDP`）
- `port`: 监听端口号（1-65535）
- `hostname`: 可选，限制该监听器只接受特定主机名的请求
- `tls`: TLS 配置（仅用于 HTTPS/TLS 协议）
  - `mode`: TLS 模式（`Terminate` 终止 TLS、`Passthrough` 透传 TLS）
  - `certificateRefs`: 引用的 TLS 证书资源，这里引用了 `envoy-gateway-system` 命名空间下的 `Secret`
- `allowedRoutes`: 定义了哪些路由可以附加到此监听器。
  - `namespaces`: 命名空间选择器，默认情况下，只允许当前命名空间的路由进行关联
    - `from: All` - 允许所有命名空间
    - `from: Same` - 仅允许相同命名空间
    - `from: Selector` - 通过标签选择器指定命名空间
  - `kinds`: 除了可以指定 `namespaces`，还可以指定允许的路由资源类型（如 `HTTPRoute`、`GRPCRoute`）

#### 🌐`addresses`（可选字段）
`addresses` 用于指定 `Gateway` 的 IP 地址列表，这是绑定 `LoadBalancer IP` 的关键字段。

```YAML
spec:
  gatewayClassName: eg
  listeners:
    - name: http
      protocol: HTTP
      port: 80
  addresses:
    - type: IPAddress
      value: 192.168.50.150
```

**Address 类型**：
- `IPAddress`: 指定具体的 `IP` 地址
- `Hostname`: 基于 `DNS` 的入口点，这个概念可能用于云负载均衡器中，`Hostname` 目前多云厂商实践中常见，如 AWS ELB 对应 `DNS`

#### ⚙`infrastructure`（可选字段，v1.2+）
* `infrastructure` 允许在 `Gateway` 中配置底层基础设施的 `labels` 或 `annotations`，实现对底层资源的控制，配置的内容会同时应用到 `Service`、`Deployment`、`Pod` 的 YAML 中。
* `infrastructure` 中可以配置 `parametersRef`，与 `GatewayClass` 中 `parametersRef` 的含义相同

```YAML
spec:
  gatewayClassName: eg
  listeners:
    - name: http
      protocol: HTTP
      port: 80
  infrastructure:
    labels:
      environment: production
      team: platform
    annotations:
      cloud-provider.io/load-balancer-type: "internal"
    parametersRef:
      group: gateway.envoyproxy.io
      kind: EnvoyProxy
      name: custom-proxy-config
      namespace: default

```

## 🧪实战示例

### ⭐示例一：自定义 Deployment 副本数
📄 配置示例
```YAML
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  parametersRef:
    group: gateway.envoyproxy.io
    kind: EnvoyProxy
    name: custom-proxy-config
    namespace: default
---
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: custom-proxy-config
  namespace: default
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyDeployment:
        replicas: 2
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: eg
spec:
  gatewayClassName: eg
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```
结果如下，可以看到对应的 Deployment 副本数为 2
```shell
kubectl -n envoy-gateway-system get deployments.apps envoy-default-eg-e41e7b31 
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
envoy-default-eg-e41e7b31   2/2     2            2           23h
```
### ⭐示例二：为 Gateway 绑定 LoadBalancer IP
#### ❌ 使用Gateway直接配置IPAddress的弊端
在 `Gateway` 的配置中，可以直接绑定 `IPAddress`。绑定之后，K8S 集群内的主机可以通过该 IP 访问 `Gateway`，但是集群外部的机器无法与 `Gateway` 进行通信，因为 `IPAddress` 中配置的 IP 地址并没有被实际绑定在网卡设备上，而是通过 iptables 的方式对流量做了代理。
📄 配置示例
```YAML
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: eg
spec:
  gatewayClassName: eg
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
  addresses:
    - type: IPAddress
      value: 192.168.50.150
```
**验证**：
* 在容器内部请求正常返回 404
```shell
docker container exec -it k3d-k3s-cluster01-agent-0 wget -O - 192.168.50.150
Connecting to 192.168.50.150 (192.168.50.150:80)
wget: server returned error: HTTP/1.1 404 Not Found
```
* 在 K8S 集群外的机器访问失败
```shell
wget -O - 192.168.50.150
--2025-11-29 00:45:54--  http://192.168.50.150/
Connecting to 192.168.50.150:80... failed: No route to host.
```
#### ✔ 使用[MetalLB](https://metallb.io/installation/) 实现真正的 Service LoadBalancer
**部署**
```shell
wget https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
kubectl apply -f metallb-native.yaml
```
**配置**：
📄 创建一个 `MetalLB` 的地址池 `metallb-ippool.yaml`
```YAML
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.50.100-192.168.50.200
---
# 若在虚拟化环境，需要启用 L2 模式最佳
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool

```
📄 在 Gateway 中通过注解指定 MetalLB 的 IP 地址
```YAML
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: eg
spec:
  gatewayClassName: eg
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
  infrastructure:
    annotations:
      metallb.io/loadBalancerIPs: 192.168.50.150
```
**验证**：再次通过宿主机的 wget 进行访问，成功返回 404
```shell
wget -O - 192.168.50.150
--2025-11-29 01:00:45--  http://192.168.50.150/
Connecting to 192.168.50.150:80... connected.
HTTP request sent, awaiting response... 404 Not Found
2025-11-29 01:00:45 ERROR 404: Not Found.
```


## 🎯 总结
- `GatewayClass` 定义了网关控制器的类型和配置
- `Gateway` 是实际的网关实例，通过 `gatewayClassName` 关联 `GatewayClass`
- `listeners` 定义了网关监听的端口和协议
- `addresses` 字段用于绑定 LoadBalancer IP 地址
- `infrastructure` 字段（v1.2+）允许配置底层资源的 `labels` 和 `annotations`
- 使用 `MetalLB` 可以让集群外部机器通过 LoadBalancer IP 访问 `Gateway`

Gateway 是现代云网关的航海灯塔，  
而你，便是为集群绘制疆域的人。

下一章  
我们将走向真实流量的路径——  
👉 **HTTPRoute 与路由匹配策略**