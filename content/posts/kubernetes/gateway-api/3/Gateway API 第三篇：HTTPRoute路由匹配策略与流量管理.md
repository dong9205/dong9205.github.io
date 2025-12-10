---
title: Gateway API 第三篇：HTTPRoute路由匹配策略与流量管理
subtitle:
date: 2025-12-04T00:47:17+08:00
slug: gateway-api-03
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
在前两章中，我们探索了 Gateway API 的起源，深入了解了 `GatewayClass` 和 `Gateway` 的核心机制，并成功为 Gateway 绑定了可被外部访问的 LoadBalancer IP。

现在，让我们进入 Gateway API 最核心的部分——**HTTPRoute**。  
HTTPRoute 是 Gateway API 中用于定义 HTTP/HTTPS 路由规则的核心资源，它决定了流量如何从 Gateway 转发到后端服务。

本篇将深入介绍：
- 🎯 **HTTPRoute 的核心字段与结构**
- 🔍 **路由匹配规则（Matches）**
- 🚦 **流量分发策略（BackendRefs）**
- ⚖️ **权重分配与负载均衡**
- 🛡️ **请求头与响应头操作**
- 🔄 **重定向与重写规则**

## 🗺️ HTTPRoute — 流量的导航图
`HTTPRoute` 是命名空间级别的资源，用于定义 HTTP/HTTPS 流量的路由规则。每个 `HTTPRoute` 必须关联一个或多个 `Gateway`，并定义如何将匹配的请求转发到后端服务。（PS：如果有熟悉Nginx的同学，可以把HTTPRoute理解为Nginx的虚拟主机VHOST）

### ⭐ 核心字段说明

#### 📌`parentRefs`（必需字段）
`parentRefs` 指定该 `HTTPRoute` 关联的 `Gateway` 列表。一个 `HTTPRoute` 可以关联多个 `Gateway`，实现跨网关的路由配置。

```YAML
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: backend-route
  namespace: default
spec:
  parentRefs:
    - name: eg
      namespace: default
      sectionName: http  # 可选，指定 Gateway 中的特定 listener
```

**字段说明**：
- `name`: Gateway 的名称（必需）
- `namespace`: Gateway 所在的命名空间（可选，默认为当前命名空间）
- `sectionName`: 指定 Gateway 中的特定 listener 名称（可选）
- `port`: 指定 Gateway 的端口号（可选）

#### 📌`hostnames`（可选字段）
`hostnames` 用于匹配请求的 Host 头，支持精确匹配和通配符匹配。这里指定的就是客户端请求的域名。

```YAML
spec:
  hostnames:
    - "www.example.com"
    - "*.example.com"
    - "api.example.com"
```

**匹配规则**：
- 精确匹配：`www.example.com` 只匹配完全相同的域名
- 通配符匹配：`*.example.com` 匹配所有 `example.com` 的子域名
- 如果未指定 `hostnames`，则匹配所有 Host 头

#### 📌`rules`（必需字段）
`rules` 是 `HTTPRoute` 的核心，定义了路由匹配规则和转发策略。每个 `rule` 包含匹配条件（`matches`）和转发目标（`backendRefs`）。

```YAML
spec:
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
          headers:
            - name: X-Version
              value: v2
      backendRefs:
        - name: backend-v2
          port: 3000
          weight: 100
```

## 🔍 路由匹配规则（Matches）

### 📌 路径匹配（Path Matching）
路径匹配是最常用的匹配规则，支持三种匹配类型：

#### 1️⃣ `Exact` - 精确匹配
精确匹配指定的路径，区分大小写。

```YAML
matches:
  - path:
      type: Exact
      value: /api/users
```

**匹配示例**：
- ✅ `/api/users` - 匹配
- ❌ `/api/users/` - 不匹配（末尾斜杠）
- ❌ `/api/users/123` - 不匹配（路径更长）
- ❌ `/API/users` - 不匹配（大小写敏感）

#### 2️⃣ `PathPrefix` - 前缀匹配
匹配以指定路径为前缀的所有请求。

```YAML
matches:
  - path:
      type: PathPrefix
      value: /api
```

**匹配示例**：
- ✅ `/api` - 匹配
- ✅ `/api/users` - 匹配
- ✅ `/api/users/123` - 匹配
- ✅ `/api/v2/products` - 匹配
- ❌ `/apis` - 不匹配（不是前缀）

#### 3️⃣ `RegularExpression` - 正则表达式匹配
使用正则表达式进行路径匹配（需要 Gateway Controller 支持）。

```YAML
matches:
  - path:
      type: RegularExpression
      value: "^/api/v[0-9]+/.*"
```

**匹配示例**：
- ✅ `/api/v1/users` - 匹配
- ✅ `/api/v2/products` - 匹配
- ❌ `/api/users` - 不匹配（缺少版本号）

### 📌 请求头匹配（Header Matching）
根据 HTTP 请求头进行匹配，支持精确匹配和正则表达式匹配。

```YAML
matches:
  - headers:
      - name: Content-Type
        value: application/json
      - name: X-Version
        type: RegularExpression
        value: "^v[0-9]+$"
```

**匹配类型**：
- `Exact`（默认）：精确匹配
- `RegularExpression`：正则表达式匹配

### 📌 查询参数匹配（Query Parameters Matching）
根据 URL 查询参数进行匹配。

```YAML
matches:
  - queryParams:
      - name: version
        type: Exact
        value: v2
      - name: env
        type: RegularExpression
        value: "^(dev|staging|prod)$"
```

### 📌 方法匹配（Method Matching）
根据 HTTP 方法进行匹配。

```YAML
matches:
  - method: GET
  - method: POST
```

**支持的 HTTP 方法**：
- `GET`、`POST`、`PUT`、`DELETE`、`PATCH`、`HEAD`、`OPTIONS`

### 📌 组合匹配
多个匹配条件可以组合使用，所有条件必须同时满足（AND 逻辑）。

```YAML
matches:
  - path:
      type: PathPrefix
      value: /api
    headers:
      - name: X-Version
        value: v2
    method: POST
    queryParams:
      - name: env
        value: prod
```

## 🚦 流量分发策略（BackendRefs）

### ⭐ 基本转发配置
`backendRefs` 定义了请求转发到的后端服务列表。

```YAML
rules:
  - matches:
      - path:
          type: PathPrefix
          value: /
    backendRefs:
      - name: backend
        group: ""  # 空字符串表示 core API group
        kind: Service
        port: 3000
        weight: 100
```

**字段说明**：
- `name`: 后端服务的名称（必需）
- `group`: API 组，空字符串表示 Kubernetes 核心 API（默认）
- `kind`: 资源类型，通常是 `Service`（默认）
- `port`: 后端服务的端口号（必需）
- `weight`: 权重，用于流量分配（默认 100）

### ⚖️ 权重分配与负载均衡
当有多个 `backendRefs` 时，可以通过 `weight` 字段实现流量分配。

```YAML
rules:
  - matches:
      - path:
          type: PathPrefix
          value: /
    backendRefs:
      - name: backend-v1
        port: 3000
        weight: 80  # 80% 的流量
      - name: backend-v2
        port: 3000
        weight: 20  # 20% 的流量
```

**权重计算**：
- 权重是相对值，按比例分配流量
- 例如：`weight: 80` 和 `weight: 20` 表示 80% 和 20% 的流量分配
- 如果所有后端权重相同，则平均分配

### 🔄 流量镜像（Traffic Mirroring，v1.3+）
流量镜像允许将请求的副本发送到另一个后端，用于测试或监控，不影响主流量。

```YAML
rules:
  - matches:
      - path:
          type: PathPrefix
          value: /
    backendRefs:
      - name: backend-primary
        port: 3000
        weight: 100
    filters:
      - type: RequestMirror
        requestMirror:
          backendRef:
            name: backend-mirror
            port: 3000
          percentage: 10  # 镜像 10% 的流量
```

## 🛡️ 请求头与响应头操作（Filters）

### 📌 请求头修改
可以在转发请求前添加、修改或删除请求头。

```YAML
rules:
  - matches:
      - path:
          type: PathPrefix
          value: /
    filters:
      - type: RequestHeaderModifier
        requestHeaderModifier:
          add:
            - name: X-Forwarded-For
              value: "192.168.1.1"
          set:
            - name: X-Version
              value: "v2"
          remove:
            - "X-Old-Header"
    backendRefs:
      - name: backend
        port: 3000
```

**操作类型**：
- `add`: 添加请求头（如果已存在则追加）
- `set`: 设置请求头（如果已存在则覆盖）
- `remove`: 删除请求头

### 📌 响应头修改
可以在返回响应前修改响应头。

```YAML
filters:
  - type: ResponseHeaderModifier
    responseHeaderModifier:
      add:
        - name: X-Response-Time
          value: "100ms"
      set:
        - name: Cache-Control
          value: "no-cache"
      remove:
        - "X-Debug-Info"
```

### 📌 URL 重写（URL Rewrite）
可以重写请求的路径和主机名。

```YAML
filters:
  - type: URLRewrite
    urlRewrite:
      path:
        type: ReplacePrefixMatch
        replacePrefixMatch: /login
      hostname: new.example.com
```

**路径重写类型**：
- `ReplaceFullPath`: 完全替换路径
- `ReplacePrefixMatch`: 替换路径前缀

### 🔄 重定向（Redirect）
可以将请求重定向到另一个 URL。

```YAML
filters:
  - type: RequestRedirect
    requestRedirect:
      scheme: https
      hostname: secure.example.com
      port: 443
      statusCode: 301  # 301 永久重定向，302 临时重定向
      path:
        type: ReplaceFullPath
        replaceFullPath: /new-path
```

## 🧪 实战示例
### 初始化
* 创建`GatewayClass`和`Gateway`
* 创建bar服务（Deployment，Service）
* 创建bar-canary服务（Deployment，Service）
📄 配置
```YAML
kind: GatewayClass
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: example-gateway-class
  labels:
    example: http-routing
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-gateway
  labels:
    example: http-routing
spec:
  gatewayClassName: example-gateway-class
  listeners:
    - name: http
      protocol: HTTP
      port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: bar-svc
  labels:
    example: http-routing
spec:
  ports:
    - name: http
      port: 8080
      targetPort: 3000
  selector:
    app: bar-backend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bar-backend
  labels:
    app: bar-backend
    example: http-routing
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bar-backend
  template:
    metadata:
      labels:
        app: bar-backend
    spec:
      containers:
        - name: bar-backend
          image: gcr.io/k8s-staging-gateway-api/echo-basic:v20231214-v1.0.0-140-gf544a46e
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          resources:
            requests:
              cpu: 10m
---
apiVersion: v1
kind: Service
metadata:
  name: bar-canary-svc
  labels:
    example: http-routing
spec:
  ports:
    - name: http
      port: 8080
      targetPort: 3000
  selector:
    app: bar-canary-backend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bar-canary-backend
  labels:
    app: bar-canary-backend
    example: http-routing
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bar-canary-backend
  template:
    metadata:
      labels:
        app: bar-canary-backend
    spec:
      containers:
        - name: bar-canary-backend
          image: gcr.io/k8s-staging-gateway-api/echo-basic:v20231214-v1.0.0-140-gf544a46e
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          resources:
            requests:
              cpu: 10m
---
```
确认网关地址
```shell
kubectl get gateways --selector=example=http-routing
NAME              CLASS                   ADDRESS          PROGRAMMED   AGE
example-gateway   example-gateway-class   192.168.50.101   True         28s
```
修改`/etc/hosts`增加一条解析
```shell
echo "$(kubectl get gateway/example-gateway -o jsonpath='{.status.addresses[0].value}') bar.example.com" >> /etc/hosts

tail -n 1 /etc/hosts
192.168.50.101 bar.example.com
```
### ⭐ 示例一：基于路径的路由分发
将不同路径的请求转发到不同的后端服务。

📄 配置示例
```YAML
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bar-route
  labels:
    example: http-routing
spec:
  parentRefs:
    - name: example-gateway
  hostnames:
    - "bar.example.com"
  rules:
# 基于路径的路由分发
    - matches:
        - path:
            type: PathPrefix
            value: /canary
      backendRefs:
        - name: bar-canary-svc
          port: 8080
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: bar-svc
          port: 8080
```
验证
```shell
> curl -s http://bar.example.com | jq .pod
"bar-backend-75959c65c-x89q6"
> curl -s http://bar.example.com/canary | jq .pod
"bar-canary-backend-5b76958f4f-dbbtx"
```
### ⭐ 示例二：基于版本头的灰度发布
根据请求头中的版本信息，将流量分发到不同版本的后端服务。

📄 配置示例
```YAML
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bar-route
  labels:
    example: http-routing
spec:
  parentRefs:
    - name: example-gateway
  hostnames:
    - "bar.example.com"
  rules:
# 基于版本头的灰度发布
    - matches:
        - headers:
            - type: RegularExpression
              name: env
              value: (test|canary|dev)  # Header中env的值为test、canary、dev时，将流量发到bar-canary服务
      backendRefs:
        - name: bar-canary-svc
          port: 8080
    - backendRefs:
        - name: bar-svc
          port: 8080
```
验证
```shell
> curl -s -H "env: canary" http://bar.example.com | jq .pod
"bar-canary-backend-5b76958f4f-dbbtx"
> curl -s -H "env: dev" http://bar.example.com | jq .pod
"bar-canary-backend-5b76958f4f-dbbtx"
> curl -s -H "env: test" http://bar.example.com | jq .pod
"bar-canary-backend-5b76958f4f-dbbtx"
# 当Header中env为非test、canary、dev时或为空时，正常将流量发到bar服务
> curl -s -H "env: prod" http://bar.example.com | jq .pod
"bar-backend-75959c65c-x89q6"
> curl -s http://bar.example.com | jq .pod
"bar-backend-75959c65c-x89q6"
```
### ⭐ 示例三：金丝雀发布
逐步将流量从旧版本迁移到新版本。

📄 配置示例
```YAML
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bar-route
  labels:
    example: http-routing
spec:
  parentRefs:
    - name: example-gateway
  hostnames:
    - "bar.example.com"
  rules:
    - backendRefs:
        # 90% 流量到稳定版本
        - name: bar-svc
          port: 8080
          weight: 90
        # 10% 流量到新版本
        - name: bar-canary-svc
          port: 8080
          weight: 10
```
**验证**：发送了十次请求，只有一次请求到了`bar-canary`
```shell
> curl -s http://bar.example.com | jq .pod
"bar-backend-75959c65c-x89q6"
> curl -s http://bar.example.com | jq .pod
"bar-backend-75959c65c-x89q6"
> curl -s http://bar.example.com | jq .pod
"bar-backend-75959c65c-x89q6"
> curl -s http://bar.example.com | jq .pod
"bar-backend-75959c65c-x89q6"
> curl -s http://bar.example.com | jq .pod
"bar-canary-backend-5b76958f4f-dbbtx"
> curl -s http://bar.example.com | jq .pod
"bar-backend-75959c65c-x89q6"
> curl -s http://bar.example.com | jq .pod
"bar-backend-75959c65c-x89q6"
> curl -s http://bar.example.com | jq .pod
"bar-backend-75959c65c-x89q6"
> curl -s http://bar.example.com | jq .pod
"bar-backend-75959c65c-x89q6"
> curl -s http://bar.example.com | jq .pod
"bar-backend-75959c65c-x89q6"
```
### ⭐ 示例四：请求头注入、响应头注入、路径重写、重定向
📄 配置示例
```YAML
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bar-route
  labels:
    example: http-routing
spec:
  parentRefs:
    - name: example-gateway
  hostnames:
    - "bar.example.com"
  rules:
# 请求头注入、响应头注入、路径重写、重定向
    - matches:
        - path:
            type: PathPrefix
            value: /v1
      filters:
        - type: RequestHeaderModifier  # 请求头注入
          requestHeaderModifier:
            set:
              - name: version
                value: "v1-v2"
        - type: ResponseHeaderModifier  # 响应头注入
          responseHeaderModifier:
            set:
              - name: version
                value: "v2"
        - type: URLRewrite  # 路径重写
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v2
            hostname: v2.example.com
      backendRefs:
        - name: bar-svc
          port: 8080
    # 重定向
    - matches:
        - path:
            type: Exact
            value: /ip
      filters:
        - type: RequestRedirect
          requestRedirect:
            scheme: http
            hostname: cip.cc
            port: 80
            statusCode: 301  # 301 永久重定向，302 临时重定向
            path:
              type: ReplaceFullPath
              replaceFullPath: /
    - backendRefs:
        - name: bar-svc
          port: 8080
```
**验证**
* 返回的Header中，增加了`version: v2`的数据
* 在请求的数据中
	* `path`从`/v1/test`修改为`/v2/test`
	* `host`从`bar.example.com`修改为`v2.example.com`
	* `headers`中增加了`Version`字段
```shell
> curl -i http://bar.example.com/v1/test
HTTP/1.1 200 OK
content-type: application/json
version: v2
{
 "path": "/v2/test",
 "host": "v2.example.com",
 "headers": {
  "Version": [
   "v1-v2"
  ],
 },
 "pod": "bar-backend-75959c65c-x89q6"
}
```
* 请求`/ip`，会返回一个重定向，重定向的地址是`http://cip.cc/`
* 然后curl命令会请求重定向的地址，并返回结果
```shell
> curl -L -i http://bar.example.com/ip
HTTP/1.1 301 Moved Permanently
location: http://cip.cc/

HTTP/1.1 200 OK
Server: openresty
Content-Type: text/plain; charset=utf-8

IP	: x.x.x.x
地址	: 中国 上海 上海
运营商	: 联通

数据二	: 中国上海上海 | 联通

数据三	: 中国上海上海市 | 联通

URL	: http://www.cip.cc/x.x.x.x

```
### ⭐ 示例五：通过Gateway实现JWT认证（JWT = JSON Web Token）
通过Gateway对用户进行验证，HTTPRoute无需修改，SecurityPolicy中指定HTTPRoute名称即可

📄 配置示例
```YAML
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bar-route
  labels:
    example: http-routing
spec:
  parentRefs:
    - name: example-gateway
  hostnames:
    - "bar.example.com"
  rules:
    - backendRefs:
        - name: bar-svc
          port: 8080
---
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: SecurityPolicy
metadata:
  name: jwt-example
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: bar-route  # 这里指定HTTPRoute的名称
  jwt:
    providers:
      - name: example
        recomputeRoute: true
        claimToHeaders:
          - claim: sub
            header: x-sub
          - claim: admin
            header: x-admin
          - claim: name
            header: x-name
        remoteJWKS:
          uri: https://raw.githubusercontent.com/envoyproxy/gateway/main/examples/kubernetes/jwt/jwks.json
```
测试
* 直接请求返回401的报错
```shell
> curl -i http://bar.example.com
HTTP/1.1 401 Unauthorized

Jwt is missing
```
* 请求时指定Token，则可请求成功，并且在Headers中增加了JWT的用户信息
```shell
> TOKEN=$(curl https://raw.githubusercontent.com/envoyproxy/gateway/main/examples/kubernetes/jwt/test.jwt -s) && echo "$TOKEN" | cut -d '.' -f2 - | base64 --decode
{"sub":"1234567890","name":"John Doe","admin":true,"iat":1516239022}

> curl -H "Authorization: Bearer $TOKEN" http://bar.example.com
{
 "path": "/",
 "host": "bar.example.com",
 "method": "GET",
 "headers": {
  "Authorization": [
   "Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWUsImlhdCI6MTUxNjIzOTAyMn0.NHVaYe26MbtOYhSKkoKYdFVomg4i8ZJd8_-RU8VNbftc4TSMb4bXP3l3YlNWACwyXPGffz5aXHc6lty1Y2t4SWRqGteragsVdZufDn5BlnJl9pdR_kdVFUsra2rWKEofkZeIC4yWytE58sMIihvo9H1ScmmVwBcQP6XETqYd0aSHp1gOa9RdUPDvoXQ5oqygTqVtxaDr6wUFKrKItgBMzWIdNZ6y7O9E0DhEPTbE9rfBo6KTFsHAZnMg4k68CDp2woYIaXbmYTWcvbzIuHO7_37GT79XdIwkm95QJ7hYC9RiwrV7mesbY4PAahERJawntho0my942XheVLmGwLMBkQ"
  ],
  "X-Admin": [
   "true"
  ],
  "X-Name": [
   "John Doe"
  ],
  "X-Sub": [
   "1234567890"
  ]
 },
 "pod": "bar-backend-75959c65c-x89q6"
}
```
## 🎯 总结
- `HTTPRoute` 是 Gateway API 中定义 HTTP/HTTPS 路由规则的核心资源
- `parentRefs` 关联 Gateway，`hostnames` 匹配域名，`rules` 定义路由规则
- 支持多种匹配方式：路径、请求头、查询参数、HTTP 方法
- `backendRefs` 定义转发目标，通过 `weight` 实现流量分配
- `filters` 提供强大的流量处理能力：请求/响应头修改、URL 重写、重定向、流量镜像
- 灵活的路由规则支持复杂的流量管理场景：路径分发、灰度发布、金丝雀发布等

HTTPRoute 是流量的导航系统，  
它指引每一份请求到达正确的目的地。