---
title: 一、Kubernetes Gateway API介绍和简单使用
subtitle:
date: 2025-11-28T01:05:17+08:00
slug: 1e2294ddd
draft: false
author:
  name:
  link:
  email:
  avatar:
description:
keywords:
license:
comment: false
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
toc: false
math: false
lightgallery: false
password:
message:
repost:
  enable: true
  url:

# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---

<!--more-->
## 概述
Gateway API 是一个Kubernetes官方项目，用于替代`Ingress`实现L4 和 L7 路由
### 有了Ingress，为什么出现了Gateway API
Ingress是一个成功的项目，但是在它诞生五年后开发者发现，在实际的使用中，为了支持ingress的灵活性，出现了大量自定义资源(CRD)和大量的注释(Annotations)，这严重的限制了Ingress的发展，在2019年的Kubecon大会上，充满热情的贡献者聚集在一起，根据以下假设，诞生了Gateway API
1. 路由匹配、流量管理和服务暴露的底层 API 标准已非常基础和普遍，将其作为自定义 API 实现，对于实现者和用户而言价值微乎其微。
2. 通过通用的核心 API 资源来表达 **L4/L7 路由**和流量管理是完全可行的。
3. 可以在不牺牲核心 API 用户体验的前提下，为更复杂的功能提供**扩展性**。

### Gateway API发展历史
* [2019-5](https://kubernetes.io/blog/2021/04/22/evolving-kubernetes-networking-with-the-gateway-api/) 提出了 **"Service APIs"** 的概念原型
* [2020-11-19](https://github.com/kubernetes-sigs/gateway-api/releases/tag/v0.1.0) 正式对外发布 "Service APIs" (v1alpha1)
* [2021-02-17](https://github.com/kubernetes-sigs/gateway-api/releases/tag/v0.2.0) 将`Service APIs`更名为 `Gateway API`
* [2022-07-14](https://kubernetes.io/blog/2022/07/13/gateway-api-graduates-to-beta/) Gateway API发布beta版，逐渐走向稳定,并且在本次发布中，提出了`GAMMA计划`，目的是让`Gateway API`支持服务网格，并且大量头部的服务网格社区已达成初步共识
* [2023-08-29](https://kubernetes.io/blog/2023/08/29/gateway-api-v0-8/) `Gateway API ` v0.8.0发版，宣布服务网格已进入实验状态，可以使用该版本进行测试，并且Kuma 2.3+、Linkerd 2.14+和Istio 1.16+已完全适配Gateway API
* [2023-11-11](https://kubernetes.io/blog/2023/10/31/gateway-api-ga/) `Gateway API ` 正式发布GA (generally available)版本，可以开始在生产中使用
* [2024-05-06](https://kubernetes.io/blog/2024/05/09/gateway-api-v1-1) `Gateway API ` 发布v1.1，更多的实验功能到GA阶段GRPCRoute、ParentReference Port ，服务网格优化
* [2024-10-04](https://kubernetes.io/blog/2024/11/21/gateway-api-v1-2) `Gateway API`  发布v1.2，部分功能已稳定`GRPCRoute` 和 `ReferenceGrant`从`v1alpha2`中移除；配置超时时间；后端协议兼容Service的`appProtocol`字段；支持在Gateway中配置基础设施`labels`或`annotations`实现对底层设施的控制
* [2025-04-24](https://kubernetes.io/blog/2025/06/02/gateway-api-v1-3/) `Gateway API`  发布v1.3，支持基于百分比的流量镜像
* [2025-10-06](https://kubernetes.io/blog/2025/11/06/gateway-api-v1-4/) `Gateway API`  发布v1.4，支持`Backend TLS policy(后端 TLS 策略)`，解决Gateway到下游服务请求明文的问题；GatewayClass Status支持，可以明确知道当前的 Gateway 控制器到底支持哪些功能，简化了自动化工具逻辑；Named Rules for Routes，给路由中的具体规则（Rules）起个名字，便于运维和调试

### 为什么要使用Gateway API
* 我认为推动使用Gateway API最大的原因动力是，Kubernetes团队已经在2025-11-11日发布声明，在2026年3月，停止对`Ingress`的维护，官方建议迁移到`Gateway API`
* `Gateway API`是由Kubernetes官方团队发起并维护的下一代API网关
* Istio、Linkerd、Traefik、Cilium这些常用的服务网格、网关完全兼容并支持`Gateway API`

## 部署Gateway API
### 启动一个Kubernetes集群
这里使用k3d启动一个k3s集群，k3s集群和k8s集群有相同的接口，消耗更低
```shell
 wget -c https://github.com/k3d-io/k3d/releases/download/v5.8.3/k3d-linux-amd64
 mv k3d-linux-amd64 /usr/local/bin/k3d
 chmod +x /usr/local/bin/k3d
 k3d cluster create k3s-cluster01 -a 1
```
### 部署Envoy Gateway
Gateway API只是定义了规范，实际的流量代理还是通过Gateway Controller进行的，我这里使用Envoy Gateway实现代理功能，支持的Gateway Controller列表可以参考https://gateway-api.sigs.k8s.io/implementations/#gateway-controller-implementation-status
#### 安装
```shell
wget https://github.com/envoyproxy/gateway/releases/download/v1.6.0/install.yaml
kubectl apply --server-side -f install.yaml
```
#### 等待Gateway服务就绪
```shell
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```
#### 部署官方示例应用
📄 quickstart.yaml
```YAML
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
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
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  labels:
    app: backend
    service: backend
spec:
  ports:
    - name: http
      port: 3000
      targetPort: 3000
  selector:
    app: backend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
      version: v1
  template:
    metadata:
      labels:
        app: backend
        version: v1
    spec:
      serviceAccountName: backend
      containers:
        - image: gcr.io/k8s-staging-gateway-api/echo-basic:v20231214-v1.0.0-140-gf544a46e
          imagePullPolicy: IfNotPresent
          name: backend
          ports:
            - containerPort: 3000
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: backend
spec:
  parentRefs:
    - name: eg
  hostnames:
    - "www.example.com"
  rules:
    - backendRefs:
        - group: ""
          kind: Service
          name: backend
          port: 3000
          weight: 1
      matches:
        - path:
            type: PathPrefix
            value: /

```

然后应用这些配置，等待服务启动完成
```shell
kubectl -n default apply -f quickstart.yaml
```
#### 测试访问
##### 获取Gateway controller的POD名称
```shell
export ENVOY_SERVICE=$(kubectl get svc -n envoy-gateway-system --selector=gateway.envoyproxy.io/owning-gateway-namespace=default,gateway.envoyproxy.io/owning-gateway-name=eg -o jsonpath='{.items[0].metadata.name}')
```
##### 使用port-forward对流量进行代理
```shell
kubectl -n envoy-gateway-system port-forward service/${ENVOY_SERVICE} 8888:80 &
```
##### 使用curl访问
```shell
curl --verbose --header "Host: www.example.com" http://localhost:8888/get
```
## 相关概念
### GatewayClass
`GatewayClass`代表一类可以实例化的网关（Gateway Controller）,通过`controllerName`字段和不同的`Gateway controller`进行关联
### Gateway
`Gateway`可以理解为是一个负载均衡，每创建一个`Gateway`，都会关联一个`GatewayClass`，并且使用`GatewayClass`关联的这个`Gateway Controller`创建一个对应`deployment`和一个LoadBalance类型的`Service`
### HTTPRoute、GRPCRoute
`HTTPRoute`、`GRPCRoute`会关联一个或者多个`Gateway`,按照不同的域名、请求路径、header将请求转发个对应的Service服务