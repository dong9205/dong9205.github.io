---
title: K8S准入控制Mutating Webhook介绍与开发实战
subtitle:
date: 2025-12-15T01:21:04+08:00
slug: ffc8ac0
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
  - Go
  - Mutatingwebhook
categories:
  - kubernetes
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRelated: false
hiddenFromFeed: false
summary:
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
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
## 1. 什么是 Mutating Webhook 以及使用场景

### 1.1 Mutating Webhook 简介

Mutating Webhook 是 Kubernetes 提供的一种准入控制器（Admission Controller）机制，它允许在资源对象被持久化到 etcd 之前，对资源进行修改。与 Validating Webhook（验证型 Webhook）不同，Mutating Webhook 可以修改资源对象的规格，而不仅仅是验证。

### 1.2 典型使用场景

在实际生产环境中，Mutating Webhook 有以下几个典型的使用场景：
* 自动注入 Sidecar 容器
* 自动添加注解和标签，配置环境变量，添加资源限制
* 自动配置 Pod 亲和性、反亲和性
* 安全策略自动应用

### 1.3 为什么需要 Mutating Webhook

在传统的 Kubernetes 使用方式中，用户需要在每个 Deployment、StatefulSet 等资源中手动配置这些通用设置，这会导致：

- **配置重复**：相同的配置需要在多个资源中重复编写
- **容易遗漏**：开发者可能忘记添加某些必要的配置
- **维护困难**：当需要修改通用配置时，需要修改所有相关资源
- **不符合最佳实践**：无法统一管理和审计

通过 Mutating Webhook，我们可以实现"配置即代码"的理念，让 Kubernetes 自动为资源添加必要的配置，减少人工干预，提高一致性和可维护性。

**本文结构如下**：
- 第 1–2 章：概念 + 原理
- 第 3 章：完整实战代码案例
- 第 4–5 章：本地 & 生产部署
- 第 6 章：完整验证闭环
## 2. Mutating Webhook 实现原理

### 2.1 Kubernetes 准入控制流程

Kubernetes 的准入控制发生在 API Server 处理请求的过程中，具体流程如下：

```
用户请求 → API Server → 认证（Authentication）→ 授权（Authorization）→ 准入控制（Admission Control）→ 持久化到 etcd
```

准入控制分为两个阶段：
1. **Mutating 阶段**：可以修改资源对象
2. **Validating 阶段**：只能验证资源对象，不能修改

### 2.2 Mutating Webhook 工作流程

当 API Server 收到创建或更新资源的请求时，会按照以下流程处理：

1. **接收请求**：API Server 接收到创建/更新资源的请求
2. **匹配规则**：根据 MutatingWebhookConfiguration 中定义的规则，判断是否需要调用 Webhook
3. **序列化资源**：将资源对象序列化为 JSON 格式
4. **发送请求**：向 Webhook 服务发送 HTTPS POST 请求，请求体包含 AdmissionReview 对象
5. **处理响应**：Webhook 服务处理请求，返回修改后的资源对象（如果需要修改）
6. **应用修改**：API Server 应用 Webhook 返回的修改
7. **继续验证**：进入 Validating 阶段进行验证
8. **持久化**：验证通过后，将资源持久化到 etcd

### 2.3 AdmissionReview 对象结构

Mutating Webhook 接收和返回的数据格式遵循 Kubernetes 的 AdmissionReview API：

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "request": {
    "uid": "请求唯一标识",
    "kind": {
      "group": "",
      "version": "v1",
      "kind": "Pod"
    },
    "resource": {
      "group": "",
      "version": "v1",
      "resource": "Pod"
    },
    "object": {
      // 完整的资源对象（JSON格式）
    },
    "oldObject": {
      // 更新操作时的旧对象
    },
    "namespace": "default",
    "operation": "CREATE" | "UPDATE" | "DELETE",
    "userInfo": {
      // 请求用户信息
    }
  }
}
```

### 2.4 JSON Patch 格式

Mutating Webhook 通过返回 JSON Patch 来修改资源对象。JSON Patch 是一种描述 JSON 文档修改的格式，例如：

```json
[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/-",
    "value": {
      "name": "sidecar",
      "image": "sidecar:latest"
    }
  }
]
```

`op` 代表操作类型，常见的操作类型：
- `add`：添加字段或数组元素
- `remove`：删除字段或数组元素
- `replace`：替换字段值
- `move`：移动字段
- `copy`：复制字段
- `test`：测试字段值

### 2.5 MutatingWebhookConfiguration 配置

MutatingWebhookConfiguration 是 Kubernetes 的资源对象，用于定义哪些资源需要调用哪些 Webhook：

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: mutating-webhook
  labels:
    app: mutating-webhook
webhooks:
  - name: mutating-webhook.k8s.io
    clientConfig:
      url: https://xxx.mutating.com/path
      caBundle: "CA证书的base64编码"
      # service:
      #   name: webhook
      #   namespace: webhook
      #   path: /mutating
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Fail
```

关键字段说明：
- `clientConfig`：Webhook 服务的地址和证书
- `rules`：匹配规则，定义哪些操作和资源会触发 Webhook
- `admissionReviewVersions`：支持的 AdmissionReview API 版本
- `sideEffects`：Webhook 是否有副作用（None/NoneOnDryRun）
- `failurePolicy`：Webhook 调用失败时的策略（Ignore/Fail）
	⚠️ 注意：生产环境使用 `Fail` 意味着  
	Webhook 不可用 = Pod 创建失败  
	初期测试或非核心链路可考虑 `Ignore`
## 3. 实战案例：基于注解的日志收集自动注入

### 3.1 需求分析

在实际生产环境中，我们经常需要为应用收集日志。传统方式需要手动在每个 Deployment 中添加日志收集的 Sidecar 容器，这种方式存在以下问题：

- 配置重复且容易遗漏
- 不同应用的日志配置可能不一致
- 维护成本高

通过 Mutating Webhook，我们可以实现：
- 当 Deployment 包含特定注解（如 `filebeat.enabled=true`）时，自动注入日志收集 Sidecar
- 根据注解中的配置（如日志路径、收集方式）动态配置 Sidecar
- 统一管理日志收集策略

### 3.2 设计方案

#### 3.2.1 注解设计

| 注解名称                             | 注解介绍                                                                                                                 | 示例值                                     | 默认值                                     |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------ | ------------------------------------------ |
| `filebeat.enabled`                   | 控制是否启用日志收集功能                                                                                                 | `"true"`                                   | 无（必须显式设置为 `"true"` 才会启用）     |
| `filebeat.log-path`                  | 指定日志收集的路径，支持多个路径（用逗号分隔）                                                                           | `"/var/log"` 或 `"/var/log,/app/logs"`     | `"/var/log/*.log"`                         |
| `filebeat.configmap`                 | 指定包含 Filebeat 配置的 ConfigMap 名称，ConfigMap 中的 `filebeat.yml` 文件会被挂载到 `/usr/share/filebeat/filebeat.yml` | `"filebeat-config"`                        | `"filebeat-config"`                        |
| `filebeat.image`                     | 指定 Filebeat 容器使用的镜像                                                                                             | `"docker.elastic.co/beats/filebeat:9.2.2"` | `"docker.elastic.co/beats/filebeat:9.2.2"` |
| `filebeat.worker`                    | 设置 `WORKER` 环境变量，用于配置 Filebeat 工作线程数                                                                     | `"2"`                                      | `"1"`                                      |
| `filebeat.bulk-max-size`             | 设置 `BulkMaxSize` 环境变量，用于配置批量处理大小                                                                        | `"4096"`                                   | `"2048"`                                   |
| `filebeat.resources.requests.cpu`    | 设置 Filebeat 容器的 CPU 请求值                                                                                          | `"100m"`                                   | `"100m"`                                   |
| `filebeat.resources.requests.memory` | 设置 Filebeat 容器的内存请求值                                                                                           | `"128Mi"`                                  | `"128Mi"`                                  |
| `filebeat.resources.limits.cpu`      | 设置 Filebeat 容器的 CPU 限制值                                                                                          | `"500m"`                                   | `"100m"`                                   |
| `filebeat.resources.limits.memory`   | 设置 Filebeat 容器的内存限制值                                                                                           | `"512Mi"`                                  | `"128Mi"`                                  |
| `filebeat.lifecycle.type`            | 指定 preStop 钩子类型，支持 `exec` 或 `httpget`                                                                          | `"exec"` 或 `"httpget"`                    | `"exec"`                                   |
| `filebeat.lifecycle.exec.command`    | 当 `lifecycle.type` 为 `exec` 时，指定执行的命令（使用逗号分隔的命令数组）                                               | `"sh,-c,sleep 15"`                         | `"sh,-c,sleep 15"`                         |
| `filebeat.lifecycle.httpget.path`    | 当 `lifecycle.type` 为 `httpget` 时，指定 HTTP 路径                                                                      | `"/stop"`                                  | `"/"`                                      |
| `filebeat.lifecycle.httpget.port`    | 当 `lifecycle.type` 为 `httpget` 时，指定端口号或命名端口                                                                | `"8080"`                                   | `"8080"`                                   |
| `filebeat.lifecycle.httpget.scheme`  | 当 `lifecycle.type` 为 `httpget` 时，指定协议类型                                                                        | `"HTTP"` 或 `"HTTPS"`                      | `"HTTP"`                                   |

#### 3.2.2 Sidecar 容器设计

根据注解配置，自动注入相应的日志收集 Sidecar 容器，并配置：
- 日志收集容器的镜像
- 挂载应用容器的日志目录
- 环境变量配置
- 资源限制

### 3.3 代码实现

代码实现分为两部分：
1. **main函数**：引入 gin 包启动HTTP服务器，注册路由将请求路由到mutating函数处理
2. **mutating函数**：分析 Pod 的注解，生成JSON Patch返回给Kubernetes
#### 3.3.1 Webhook 服务依赖包和 main 函数

```go
import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"path/filepath"
	"strconv"
	"strings"

	"github.com/gin-gonic/gin"
	"github.com/wI2L/jsondiff"
	admission "k8s.io/api/admission/v1"
	v1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/resource"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/serializer"
	"k8s.io/apimachinery/pkg/util/intstr"
)

func main() {
	// use gin
	r := gin.Default()
	r.POST("/mutating", mutating)
	fmt.Println(r.Run(":9200"))
}
```

#### 3.3.2 处理 AdmissionReview 请求，生成 JSON Patch

```go
// 全局变量
var (
	scheme                = runtime.NewScheme()
	_                     = v1.AddToScheme(scheme)
	codecFactory          = serializer.NewCodecFactory(scheme)
	universalDeserializer = codecFactory.UniversalDeserializer()
)

// FilebeatConfig 配置结构
type FilebeatConfig struct {
	APP            string
	Namespace      string
	LogDir         []string
	LogPath        string
	FilebeatConfig string
	Resources      *v1.ResourceRequirements
	Lifecycle      *v1.Lifecycle
	Worker         string
	BulkMaxSize    string
	FilebeatImage  string
}

// parseFilebeatConfig 解析注解配置
func parseFilebeatConfig(annotations map[string]string) *FilebeatConfig {
	config := &FilebeatConfig{
		APP:            "filebeat",
		LogDir:         []string{"/var/log"},                     // 默认目录
		LogPath:        "/var/log/*.log",                         // 默认路径
		FilebeatConfig: "filebeat-config",                        // 默认配置
		Worker:         "1",                                      // 默认值
		BulkMaxSize:    "2048",                                   // 默认值
		FilebeatImage:  "docker.elastic.co/beats/filebeat:9.2.2", // 默认镜像
	}

	// 解析日志路径
	if logPath, ok := annotations["filebeat.log-path"]; ok && logPath != "" {
		config.LogPath = logPath
		config.LogDir = []string{}
		for _, dir := range strings.Split(logPath, ",") {
			config.LogDir = append(config.LogDir, filepath.Dir(dir))
		}
	}

	// 解析worker
	if worker, ok := annotations["filebeat.worker"]; ok && worker != "" {
		config.Worker = worker
	}

	// 解析filebeat配置
	if filebeatConfig, ok := annotations["filebeat.configmap"]; ok && filebeatConfig != "" {
		config.FilebeatConfig = filebeatConfig
	}

	// 解析bulk_max_size
	if bulkMaxSize, ok := annotations["filebeat.bulk-max-size"]; ok && bulkMaxSize != "" {
		config.BulkMaxSize = bulkMaxSize
	}

	// 解析filebeat镜像
	if image, ok := annotations["filebeat.image"]; ok && image != "" {
		config.FilebeatImage = image
	}

	// 解析resources
	config.Resources = parseResources(annotations)

	// 解析lifecycle
	config.Lifecycle = parseLifecycle(annotations)

	return config
}

// parseResources 解析资源限制配置
func parseResources(annotations map[string]string) *v1.ResourceRequirements {
	resources := &v1.ResourceRequirements{
		Requests: make(v1.ResourceList),
		Limits:   make(v1.ResourceList),
	}

	// 解析requests
	if cpu, ok := annotations["filebeat.resources.requests.cpu"]; ok && cpu != "" {
		resources.Requests[v1.ResourceCPU] = resource.MustParse(cpu)
	} else {
		resources.Requests[v1.ResourceCPU] = resource.MustParse("100m")
	}
	if memory, ok := annotations["filebeat.resources.requests.memory"]; ok && memory != "" {
		resources.Requests[v1.ResourceMemory] = resource.MustParse(memory)
	} else {
		resources.Requests[v1.ResourceMemory] = resource.MustParse("128Mi")
	}

	// 解析limits
	if cpu, ok := annotations["filebeat.resources.limits.cpu"]; ok && cpu != "" {
		resources.Limits[v1.ResourceCPU] = resource.MustParse(cpu)
	} else {
		resources.Limits[v1.ResourceCPU] = resource.MustParse("100m")
	}
	if memory, ok := annotations["filebeat.resources.limits.memory"]; ok && memory != "" {
		resources.Limits[v1.ResourceMemory] = resource.MustParse(memory)
	} else {
		resources.Limits[v1.ResourceMemory] = resource.MustParse("128Mi")
	}

	return resources
}

// parseLifecycle 解析lifecycle配置
func parseLifecycle(annotations map[string]string) *v1.Lifecycle {
	lifecycleType := annotations["filebeat.lifecycle.type"]
	if lifecycleType == "" {
		lifecycleType = "exec" // 默认使用exec
	}

	lifecycle := &v1.Lifecycle{}

	switch strings.ToLower(lifecycleType) {
	case "exec":
		// 默认exec命令：sleep 15
		command := []string{"sh", "-c", "sleep 15"}
		if cmd, ok := annotations["filebeat.lifecycle.exec.command"]; ok && cmd != "" {
			// 支持自定义命令，格式：command1,command2,command3
			command = strings.Split(cmd, ",")
		}
		lifecycle.PreStop = &v1.LifecycleHandler{
			Exec: &v1.ExecAction{
				Command: command,
			},
		}
	case "httpget":
		// HTTP GET配置
		path := "/"
		if p, ok := annotations["filebeat.lifecycle.httpget.path"]; ok && p != "" {
			path = p
		}
		port := intstrFromString(annotations["filebeat.lifecycle.httpget.port"], 8080)
		scheme := v1.URISchemeHTTP
		if s, ok := annotations["filebeat.lifecycle.httpget.scheme"]; ok && s != "" {
			scheme = v1.URIScheme(strings.ToUpper(s))
		}
		lifecycle.PreStop = &v1.LifecycleHandler{
			HTTPGet: &v1.HTTPGetAction{
				Path:   path,
				Port:   port,
				Scheme: scheme,
			},
		}
	}

	return lifecycle
}

// intstrFromString 从字符串创建IntOrString
func intstrFromString(s string, defaultVal int) intstr.IntOrString {
	if s == "" {
		return intstr.FromInt32(int32(defaultVal))
	}
	// 尝试解析为数字
	if val, err := strconv.Atoi(s); err == nil {
		return intstr.FromInt32(int32(val))
	}
	// 否则作为命名端口
	return intstr.FromString(s)
}

// createFilebeatContainer 创建filebeat容器
func createFilebeatContainer(config *FilebeatConfig) v1.Container {
	container := v1.Container{
		Name:  "filebeat",
		Image: config.FilebeatImage,
		Env: []v1.EnvVar{
			{
				Name:  "WORKER",
				Value: config.Worker,
			},
			{
				Name:  "BulkMaxSize",
				Value: config.BulkMaxSize,
			},
			{
				Name:  "LOG_PATH",
				Value: config.LogPath,
			},
			{
				Name:  "APP",
				Value: config.APP,
			},
			{
				Name:  "NAMESPACE",
				Value: config.Namespace,
			},
		},
		VolumeMounts: []v1.VolumeMount{
			{
				Name:      config.FilebeatConfig,
				MountPath: "/usr/share/filebeat/filebeat.yml",
				SubPath:   "filebeat.yml",
				ReadOnly:  true,
			},
		},
	}

	// 设置resources
	if config.Resources != nil {
		container.Resources = *config.Resources
	}

	// 设置lifecycle
	if config.Lifecycle != nil {
		container.Lifecycle = config.Lifecycle
	}

	return container
}

func mutating(c *gin.Context) {
	// 读取请求 body
	bodyBytes, err := io.ReadAll(c.Request.Body)
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": fmt.Sprintf("读取请求体失败: %v", err)})
		return
	}

	// 打印 body 内容
	log.Printf("收到请求 body:\n%s\n", string(bodyBytes))

	// 重新设置 body，以便后续解析
	c.Request.Body = io.NopCloser(bytes.NewReader(bodyBytes))

	// 解析请求
	var admissionReviewReq admission.AdmissionReview
	if err := c.ShouldBindJSON(&admissionReviewReq); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	// 创建响应
	admissionReviewResponse := admission.AdmissionReview{
		TypeMeta: metav1.TypeMeta{
			Kind:       "AdmissionReview",
			APIVersion: "admission.k8s.io/v1",
		},
		Response: &admission.AdmissionResponse{
			UID:     admissionReviewReq.Request.UID,
			Allowed: true,
		},
	}

	// 如果命名空间是 kube-system、kube-public 等系统命名空间，则直接返回，不进行任何处理
	switch admissionReviewReq.Request.Namespace {
	case metav1.NamespacePublic, metav1.NamespaceSystem, "kube-flannel", metav1.NamespaceAll:
		c.JSON(200, admissionReviewResponse)
		return
	}
	// 解析请求中的 Pod 对象
	var oldPod v1.Pod
	oldPodRaw := admissionReviewReq.Request.Object.Raw
	if _, _, err := universalDeserializer.Decode(oldPodRaw, nil, &oldPod); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	// 打印接收到的 Pod 信息
	log.Printf("收到Pod信息:\n%+v\n", oldPod)
	// 检查是否启用日志收集
	annotations := oldPod.Annotations
	if annotations == nil {
		annotations = make(map[string]string)
	}

	// 检查是否启用 filebeat
	enabled := annotations["filebeat.enabled"]
	if enabled != "true" {
		c.JSON(200, admissionReviewResponse)
		return
	}

	// 创建修改后的Pod副本
	newPod := oldPod.DeepCopy()

	// 解析配置
	config := parseFilebeatConfig(annotations)
	if _, ok := oldPod.ObjectMeta.Labels["app"]; ok {
		config.APP = oldPod.ObjectMeta.Labels["app"]
	}
	config.Namespace = oldPod.ObjectMeta.Namespace
	// 添加 filebeat sidecar 容器
	filebeatContainer := createFilebeatContainer(config)
	newPod.Spec.Containers = append(newPod.Spec.Containers, filebeatContainer)
	newPod.Spec.Volumes = append(newPod.Spec.Volumes, v1.Volume{
		Name: config.FilebeatConfig,
		VolumeSource: v1.VolumeSource{
			ConfigMap: &v1.ConfigMapVolumeSource{
				LocalObjectReference: v1.LocalObjectReference{
					Name: config.FilebeatConfig,
				},
			},
		},
	})
	volumeMounts := []v1.VolumeMount{}
	for _, dir := range config.LogDir {
		volumeMounts = append(volumeMounts, v1.VolumeMount{
			Name:      fmt.Sprintf("log-volume-%s", strings.ReplaceAll(dir, "/", "-")),
			MountPath: dir,
			ReadOnly:  false,
		})
		newPod.Spec.Volumes = append(newPod.Spec.Volumes, v1.Volume{
			Name: fmt.Sprintf("log-volume-%s", strings.ReplaceAll(dir, "/", "-")),
			VolumeSource: v1.VolumeSource{
				EmptyDir: &v1.EmptyDirVolumeSource{},
			},
		})
	}
	// 为所有容器（包括filebeat）挂载日志目录
	for i := range newPod.Spec.Containers {
		newPod.Spec.Containers[i].VolumeMounts = append(newPod.Spec.Containers[i].VolumeMounts, volumeMounts...)
	}

	patch, err := jsondiff.Compare(oldPod, newPod)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": fmt.Sprintf("generate patch: %v", err)})
		return
	}

	patchBytes, err := json.MarshalIndent(patch, "", "    ")
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": fmt.Sprintf("marshal patch: %v", err)})
		return
	}

	// 设置patch响应
	admissionReviewResponse.Response.Patch = patchBytes
	patchType := admission.PatchTypeJSONPatch
	admissionReviewResponse.Response.PatchType = &patchType
	// 打印响应
	log.Printf("响应:\n%+s\n", patchBytes)
	c.JSON(200, admissionReviewResponse)
}

```
## 后续几部分介绍
- 第四部分：在本地开发阶段，Kubernetes 直接连接到本地的k8s-webhook服务，介绍了两种方式
- 第五部分：用于线上部署，介绍了两种部署方式
- 第六部分：功能验证，确保k8s-webhook根据注解信息完成了Pod的配置

**说明**：第四部分和第五部分共介绍了四种部署方式，完成其中任意一种方式后，即可进行第六部分的验证。
## 4. 开发阶段怎么玩：Webhook 本地调试的两种姿势

**Cloudflare Tunnel 的价值在于**
在开发阶段，为了更方便地调试和测试，我们可以先在本地启动 Webhook 服务，这样代码的变动能快速验证。
### 4.1 本地使用HTTP方式启动，借助 Cloudflare Tunnel 实现 HTTPS 访问

> **不用改集群、不用签证书，就能让 APIServer 访问你的本地 Webhook**
#### 4.1.1 Cloudflare Tunnel 简介

Cloudflare Tunnel 是 Cloudflare 提供的一种安全隧道服务，可以将本地服务暴露到公网，并提供 HTTPS 支持，无需配置域名和证书。
#### 4.1.2 配置自定义域名的Cloudflare Tunnel 
> 这里推荐一个 Cloudflare Tunnel 视频教程: https://www.bilibili.com/video/BV1H4421X7Wg

1. 首先需要在Cloudflare上添加你的域名，按照Cloudflare的步骤操作即可
2. 打开Cloudflare的ZeroTrust服务，或者直接打开网址 https://one.dash.cloudflare.com/
3. 在`网络`-`连接器`中添加一个`Cloudflare Tunnel`，按照Cloudflare的步骤操作即可
4. 在最后一步配置发布应用的页面，类型使用`HTTP`，URL使用`localhost:9200`，下图是我的配置
![CloudFlare](/kubernetes/20251214001229.png)
#### 4.1.3 配置 MutatingWebhookConfiguration

将 Cloudflare Tunnel 配置的 HTTPS URL 配置到 MutatingWebhookConfiguration 中：

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: k8s-webhook
  labels:
    app: k8s-webhook
webhooks:
  - name: k8s-webhook.k8s.io
    clientConfig:
      url: https://k8s-hook.p-pp.xyz/mutating
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Fail
```
### 4.2 本地使用TLS启动

本地生成自签证书，然后将CA证书通过base64编码写在MutatingWebhookConfiguration中
#### 4.2.1 生成证书
* 如果使用IP地址作为webhook的地址，需要将`IP.1`的值改为你的实际IP地址
* 证书有效期为100年
```shell
mkdir certs && cd certs
cat > csr.conf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = *
IP.1 = 172.17.0.1
EOF
openssl genrsa -out tls.key 2048
openssl req -new -key tls.key -out tls.csr -subj  "/CN=k8s-webhook" -config csr.conf
openssl x509 -req -in tls.csr -signkey tls.key -out tls.crt -days 36500 -extensions v3_req -extfile csr.conf
```
#### 4.2.2 生成CA_BUNDLE
- CA_BUNDLE将配置在MutatingWebhookConfiguration的`caBundle`字段中
```shell
cat tls.crt | base64 | tr -d '\n'
LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURBekNDQWV1Z0F3SUJBZ0lVSG5nVjFJSExOdFNtbjZTMmgxbFZML0xoMno0d0RRWUpLb1pJaHZjTkFRRUwKQlFBd0ZqRVVNQklHQTFVRUF3d0xhemh6TFhkbFltaHZiMnN3SUJjTk1qVXhNakUwTVRNeE9ERTJXaGdQTWpFeQpOVEV4TWpBeE16RTRNVFphTUJZeEZEQVNCZ05WQkFNTUMyczRjeTEzWldKb2IyOXJNSUlCSWpBTkJna3Foa2lHCjl3MEJBUUVGQUFPQ0FROEFNSUlCQ2dLQ0FRRUFrLzFNMFQ3VVpoZitmQU9LaG9vM2dQRE8wblMvZWQ4cWRoZmwKQTBoamtEcEFCd0pPVndxT21JRS8rZjhWT2gwZVphVzc0UE5PTGJMbDl0aGdtM29mcmZxSUFqS2hONWQyTDFtaQpnSS9WeVNSTmxkUm8zZ05oYU1mNFRXell3OVY3VUFYaXNZMi9aZUVSeThSb3UzbWwxNjdjYnNrWFNVU3lOU3RrCnVBTzlDYlBYOVBOMzAzaGZvWGFhNHhxUHBoZE5SNmU5YmZOcm5qU0NhQ1plcWdpeXUzT29EekNHWXhXaVY0ei8KQ3pHbndOcXp5VzJRMXlSYWxDWEVURjZrSExQYU9uSGJYcC9jeDJxaVRWMzVObXlDMFRiZDNRK3haZXh5ZGdzbwpPa1N6bUJnMXU1VFNobkFiOWJPd3l0Z0d0NnpqU2krYWJrRWZMV0JqbFE3Q3ZpellBd0lEQVFBQm8wY3dSVEFKCkJnTlZIUk1FQWpBQU1Bc0dBMVVkRHdRRUF3SUY0REFNQmdOVkhSRUVCVEFEZ2dFcU1CMEdBMVVkRGdRV0JCUmgKa08yUUlTRC95Um92VVUzREwzRkhGWERGMGpBTkJna3Foa2lHOXcwQkFRc0ZBQU9DQVFFQU1TcVJydTBvT1lLWgorSERyWG1DOUViR1FOdzFNMnNwY3Aza3NJQnNTMWxOcG9ENU1rSUJ3eEhDVXNCTTcxVWRnVlVEN1NqQUJ3V3N2ClpQNTVsSVlNOWptZnZuajJMcmNoYWlYVEJVVnFabHFOd1NPUGxXRGE3WUQxRlRMY3FxUW5mazQrV2R5Z2lHZVYKK0lOSys2MmlUK0psdnovZ1NWOVdnT1ROcldBa1hNVWlSTGdVcms0L3dVemZvM0NBVkN4b2t1NDhqTThjY3QrQwpSV1N4RDhhSnI2V01xL2ZkbThjbjJwQWJFK2xJZkIxdm9DaTRHcm44NHdsSmkwVFU2Q3JyQVp4Q21RNEJSNGJjCmxvdEZEMis1VWh4OVpYVTBLZG1FUlVEejNJa200S2pDUkpFS2w0TEdkUGM4aWNVVnIzb0ZabFUvSmVRUUJQUTIKcHdQTnFvMUFnUT09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
```
#### 4.2.3 配置MutatingWebhookConfiguration

- MutatingWebhookConfiguration的YAML
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: k8s-webhook
  labels:
    app: k8s-webhook
webhooks:
  - name: k8s-webhook.k8s.io
    clientConfig:
      url: https://172.17.0.1:9200/mutating
      caBundle: "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURDVENDQWZHZ0F3SUJBZ0lVSThjVllUb1ZBMHFCYm5wWnN0SFY5ZlMxY0o4d0RRWUpLb1pJaHZjTkFRRUwKQlFBd0ZqRVVNQklHQTFVRUF3d0xhemh6TFhkbFltaHZiMnN3SUJjTk1qVXhNakUwTVRNek1qSXpXaGdQTWpFeQpOVEV4TWpBeE16TXlNak5hTUJZeEZEQVNCZ05WQkFNTUMyczRjeTEzWldKb2IyOXJNSUlCSWpBTkJna3Foa2lHCjl3MEJBUUVGQUFPQ0FROEFNSUlCQ2dLQ0FRRUF3OFVOaUdueTM2N2V0Nlo0c3BpVEFPUDlod3pULzBadWhrdE8KbnVpWGkxajFEVnppL3pRZXdCWjVGNjloM0dUeGo2UEZJK3J3YzZqNFJkRTZyUzMrOGVDSmVFNmtWeDRleUxhdQpLbFVPSDcyQUZtQlZKUXoxaVNGVW1wY1hQNVdTN2ZyUlk5bVIxL0lXR1kxaGwrcFV2UkhXbStFTDVCa1B0bjdWClVOYU1EdE5mN3FFeUNOclZhRzZBZzJ3ZzNPY2JYNURqRHJvM1ZvL0RwNE9xK0NhK1pXMXhFT0Y5UFFKOU02VzMKOHhLaU9hUlVtVU9nOEdZcmVEK1R5SHNkV21MOWxTM2FWSnQwTnVVTjNNNGVJcGNBYXM5ck1oZ08wMkJka3hhYQpPZVhBRFkzbTVXaFhiMW81UFpmWTFUK2ozOSt5enFtQm1EUk5UeE9iOUF6K1FoeWVFd0lEQVFBQm8wMHdTekFKCkJnTlZIUk1FQWpBQU1Bc0dBMVVkRHdRRUF3SUY0REFTQmdOVkhSRUVDekFKZ2dFcWh3U3NFUUFCTUIwR0ExVWQKRGdRV0JCUzlXUUxUZ2M4K3pqVk9KaWJoVDJKYjloMmNuekFOQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBSDNhUQpJZC9mZ21rQWhENWNvQVhWc05xYnJyc3FkZ1MwMHp5eXMvWWZ5SncybGRuNStGZGNRQ3JtMkJJdGhZL2VIdkc3CjlIYTZZSC9pTWRPSGNwTHpMTE9yU2g4RlBNYzl0b0xJSWtRRlM5QmtYL01Jczh1QmJZOVJVS3Zyd3B4a2JQY0YKaWtMRVYrTkJGUE1PdGdMSnlmL0F3N2JWRHJpYTJIcnk3ak9tZUtvQk9YY1JORVQvODM1SlBueXFuOTVMeEtJSApUZ3Vqd0pBYlE4NldnRXY3L2xhZGZqQlgrMWhOb1UvSDI0Y3VuTlIzYkwydVNTbFJyNkltbjJIU09TZEduUzRPCnpVN1BEYUdyVHJQL0l2cmZNV0VSR2Rxams4YkVQVHRvMTB3cVdKTkw4elN3L2poeU1NK0Y3Nkg4WDFXc001NEUKWWZUeXRpQWk1VjFLaTIxM1JRPT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo="
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Fail
```
#### 4.2.4 修改代码，使用TLS模式启动
- 只需要修改main函数中的启动方式即可
```go
func main() {
	r := gin.Default()
	r.POST("/mutating", mutating)
	r.RunTLS(":9200", "certs/tls.crt", "certs/tls.key")
}
```
## 5. 生产环境部署
在生产环境中，Webhook 服务需要部署在 Kubernetes 集群内部，并配置正确的 TLS 证书。
### 5.1 将代码打包为镜像
使用支持TLS的代码版本进行打包
- Dockerfile文件
```Dockerfile
FROM golang:1.25.4-alpine AS builder

WORKDIR /app
ENV GOPROXY=https://goproxy.cn,direct \
    CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64
# 复制 go mod 文件
COPY go.mod go.sum ./
RUN go mod download

# 复制源代码
COPY main.go ./

# 构建应用
RUN go build -o webhook main.go

# 运行阶段
FROM alpine:latest
RUN set -eux && sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories 
RUN apk --no-cache add ca-certificates tzdata

WORKDIR /root/

# 从构建阶段复制二进制文件
COPY --from=builder /app/webhook .

# 暴露端口
EXPOSE 9200

# 运行应用
CMD ["./webhook"]
```
- 打包为镜像并上传到镜像仓库。这里使用的是阿里云的镜像仓库，`registry.cn-hangzhou.aliyuncs.com/dong9205/k8s-webhook-log` 是示例镜像仓库地址，需要根据实际情况修改为你的镜像仓库地址
```shell
docker build -t registry.cn-hangzhou.aliyuncs.com/dong9205/k8s-webhook-log:20251214 -f deploy/Dockerfile .
docker push registry.cn-hangzhou.aliyuncs.com/dong9205/k8s-webhook-log:20251214
```
### 5.2 自建TLS并将服务部署在Kubernetes集群

生成证书和CA_BUNDLE的方式可以参考前面4.2节中TLS启动的生成方式
#### 5.2.1 使用生成的证书创建secret
```shell
kubectl -n kube-system create secret tls k8s-webhook-tls --cert=certs/tls.crt --key=certs/tls.key
```
#### 5.2.2 将k8s-webhook服务部署在集群中
```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-webhook
  namespace: kube-system
  labels:
    app: k8s-webhook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k8s-webhook
  template:
    metadata:
      labels:
        app: k8s-webhook
    spec:
      containers:
        - name: webhook
          image: registry.cn-hangzhou.aliyuncs.com/dong9205/k8s-webhook-log:20251214
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 9200
              name: webhook
              protocol: TCP
          volumeMounts:
            - name: tls-certs
              mountPath: /app/certs
              readOnly: true
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
      volumes:
        - name: tls-certs
          secret:
            secretName: k8s-webhook-tls
---
apiVersion: v1
kind: Service
metadata:
  name: k8s-webhook
  namespace: kube-system
  labels:
    app: k8s-webhook
spec:
  ports:
    - port: 443
      targetPort: 9200
      protocol: TCP
      name: webhook
  selector:
    app: k8s-webhook
  type: ClusterIP
```
#### 5.2.3 配置MutatingWebhookConfiguration
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: k8s-webhook
  labels:
    app: k8s-webhook
webhooks:
  - name: k8s-webhook.k8s.io
    clientConfig:
      service:
        name: k8s-webhook
        namespace: kube-system
        path: /mutating
      caBundle: "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURDVENDQWZHZ0F3SUJBZ0lVSThjVllUb1ZBMHFCYm5wWnN0SFY5ZlMxY0o4d0RRWUpLb1pJaHZjTkFRRUwKQlFBd0ZqRVVNQklHQTFVRUF3d0xhemh6TFhkbFltaHZiMnN3SUJjTk1qVXhNakUwTVRNek1qSXpXaGdQTWpFeQpOVEV4TWpBeE16TXlNak5hTUJZeEZEQVNCZ05WQkFNTUMyczRjeTEzWldKb2IyOXJNSUlCSWpBTkJna3Foa2lHCjl3MEJBUUVGQUFPQ0FROEFNSUlCQ2dLQ0FRRUF3OFVOaUdueTM2N2V0Nlo0c3BpVEFPUDlod3pULzBadWhrdE8KbnVpWGkxajFEVnppL3pRZXdCWjVGNjloM0dUeGo2UEZJK3J3YzZqNFJkRTZyUzMrOGVDSmVFNmtWeDRleUxhdQpLbFVPSDcyQUZtQlZKUXoxaVNGVW1wY1hQNVdTN2ZyUlk5bVIxL0lXR1kxaGwrcFV2UkhXbStFTDVCa1B0bjdWClVOYU1EdE5mN3FFeUNOclZhRzZBZzJ3ZzNPY2JYNURqRHJvM1ZvL0RwNE9xK0NhK1pXMXhFT0Y5UFFKOU02VzMKOHhLaU9hUlVtVU9nOEdZcmVEK1R5SHNkV21MOWxTM2FWSnQwTnVVTjNNNGVJcGNBYXM5ck1oZ08wMkJka3hhYQpPZVhBRFkzbTVXaFhiMW81UFpmWTFUK2ozOSt5enFtQm1EUk5UeE9iOUF6K1FoeWVFd0lEQVFBQm8wMHdTekFKCkJnTlZIUk1FQWpBQU1Bc0dBMVVkRHdRRUF3SUY0REFTQmdOVkhSRUVDekFKZ2dFcWh3U3NFUUFCTUIwR0ExVWQKRGdRV0JCUzlXUUxUZ2M4K3pqVk9KaWJoVDJKYjloMmNuekFOQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBSDNhUQpJZC9mZ21rQWhENWNvQVhWc05xYnJyc3FkZ1MwMHp5eXMvWWZ5SncybGRuNStGZGNRQ3JtMkJJdGhZL2VIdkc3CjlIYTZZSC9pTWRPSGNwTHpMTE9yU2g4RlBNYzl0b0xJSWtRRlM5QmtYL01Jczh1QmJZOVJVS3Zyd3B4a2JQY0YKaWtMRVYrTkJGUE1PdGdMSnlmL0F3N2JWRHJpYTJIcnk3ak9tZUtvQk9YY1JORVQvODM1SlBueXFuOTVMeEtJSApUZ3Vqd0pBYlE4NldnRXY3L2xhZGZqQlgrMWhOb1UvSDI0Y3VuTlIzYkwydVNTbFJyNkltbjJIU09TZEduUzRPCnpVN1BEYUdyVHJQL0l2cmZNV0VSR2Rxams4YkVQVHRvMTB3cVdKTkw4elN3L2poeU1NK0Y3Nkg4WDFXc001NEUKWWZUeXRpQWk1VjFLaTIxM1JRPT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo="
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Fail

```
### 5.3 使用 cert-manager 管理证书

cert-manager 是 Kubernetes 中广泛使用的证书管理工具，可以自动申请、续期和管理 TLS 证书，并且可以自动为MutatingWebhookConfiguration配置CA Bundle

#### 5.3.1 安装 cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.19.2/cert-manager.yaml
```
#### 5.3.2 创建 Issuer 或 ClusterIssuer

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
```

#### 5.3.3 创建 Certificate 资源

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: k8s-webhook-cert
  namespace: kube-system
spec:
  secretName: k8s-webhook-tls
  duration: 2160h # 证书90天有效期
  renewBefore: 360h # 证书提前15天续签
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  dnsNames:
    - k8s-webhook.kube-system.svc
    - k8s-webhook.kube-system.svc.cluster.local
```
#### 5.3.4 配置 Webhook 服务使用证书

在 Webhook 服务的 Deployment 中，挂载证书 Secret：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-webhook
  namespace: kube-system
  labels:
    app: k8s-webhook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k8s-webhook
  template:
    metadata:
      labels:
        app: k8s-webhook
    spec:
      containers:
        - name: webhook
          image: registry.cn-hangzhou.aliyuncs.com/dong9205/k8s-webhook-log:20251214
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 9200
              name: webhook
              protocol: TCP
          volumeMounts:
            - name: tls-certs
              mountPath: /app/certs
              readOnly: true
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
      volumes:
        - name: tls-certs
          secret:
            secretName: k8s-webhook-tls
---
apiVersion: v1
kind: Service
metadata:
  name: k8s-webhook
  namespace: kube-system
  labels:
    app: k8s-webhook
spec:
  ports:
    - port: 443
      targetPort: 9200
      protocol: TCP
      name: webhook
  selector:
    app: k8s-webhook
  type: ClusterIP
```
#### 5.3.5 更新 MutatingWebhookConfiguration
- 通过添加注解 `cert-manager.io/inject-ca-from`，cert-manager会自动为MutatingWebhookConfiguration添加caBundle
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: k8s-webhook
  annotations:
    cert-manager.io/inject-ca-from: kube-system/k8s-webhook-cert
  labels:
    app: k8s-webhook
webhooks:
  - name: k8s-webhook.k8s.io
    clientConfig:
      service:
        name: k8s-webhook
        namespace: kube-system
        path: /mutating
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Fail

```
## 6. 功能验证
### 6.1 启动并配置Elasticsearch和Kibana
#### 启动服务
```shell
docker run -d --name es01 -p 19200:9200 -it -m 1GB docker.elastic.co/elasticsearch/elasticsearch:9.2.2
docker run -d --name kib01 -p 5601:5601 docker.elastic.co/kibana/kibana:9.2.2
```
#### 配置 Elasticsearch
* 配置 Elasticsearch 用户的密码，我这里生成的密码是`q=vuEQS=cw*444OM3BeZ`，后续登录 Kibana 和配置 Filebeat 会使用
```shell
docker exec -it es01 /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```
* 初始化 Kibana 连接的 token，这里的Token配置kibana会使用
```shell
docker exec -it es01 /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
```
#### 配置Kibana
打开Kibana页面，输入前面步骤中生成的Kibana enrollment token，之后使用elastic用户和密码登录即可
### 6.2 配置 Filebeat 的 ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
data:
  filebeat.yml: |
    filebeat.inputs:
    - type: filestream
      id: input-${NAMESPACE}-${APP}
      paths: ${LOG_PATH:/var/log/*.log}
      fields:
        appid: ${APP}
        namespace: ${NAMESPACE}
      prospector.scanner.fingerprint:
        length: 64
        offset: 0
    
    processors:
    - add_cloud_metadata: ~
    
    output.elasticsearch:
      hosts: '${ELASTICSEARCH_HOSTS:https://172.17.0.1:19200}'
      ssl.verification_mode: none
      username: '${ELASTICSEARCH_USERNAME:elastic}'
      password: '${ELASTICSEARCH_PASSWORD:q=vuEQS=cw*444OM3BeZ}'
      index: "app-%{[fields][appid]}"
      worker: ${WORKER:"1"}
      bulk_max_size: ${BulkMaxSize:"50"}
    setup.ilm.enabled: false
    setup.template.name: "log"
    setup.template.pattern: "log-*"
```
### 6.3 启动一个 Nginx 的 Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      annotations:
        filebeat.enabled: "true"
        filebeat.log-path: /var/log/nginx/*.log
        filebeat.resources.limits.cpu: 200m
        filebeat.resources.limits.memory: 256Mi
        filebeat.resources.requests.cpu: 200m
        filebeat.resources.requests.memory: 256Mi
        filebeat.lifecycle.exec.command: /bin/sh,-c,sleep 60
      labels:
        app: nginx
    spec:
      containers:
      - image: docker.io/library/nginx:latest
        imagePullPolicy: IfNotPresent
        name: nginx
```
### 6.4 验证Pod配置

```shell
kubectl get pods  -l app=nginx -o custom-columns=name:.metadata.name,containers_name:.spec.containers[*].name,resources:.spec.containers[1].resources,lifecycle:.spec.containers[1].lifecycle.preStop
name                    containers_name   resources                                                                    lifecycle
nginx-867f8b5cc-nwxm9   nginx,filebeat    map[limits:map[cpu:200m memory:256Mi] requests:map[cpu:200m memory:256Mi]]   map[exec:map[command:[/bin/sh -c sleep 60]]]
```
* 自动增加了filebeat容器：可以看到结果中`containers_name`字段包含nginx和filebeat
* resources已按照注解配置
* lifecycle.preStop已按照注解配置
### 6.5 查看Kibana日志是否收集
![Kibana](/kubernetes/20251215001274.png)
## 7. 总结

本文介绍了 Kubernetes Mutating Webhook 的开发实战，包括：

1. **使用场景**：了解了 Mutating Webhook 在自动注入 Sidecar、配置环境变量等场景中的应用
2. **实现原理**：深入理解了 Kubernetes 准入控制流程和 Mutating Webhook 的工作机制
3. **实战案例**：通过日志收集自动注入的案例，学习了如何开发一个实用的 Webhook
4. **本地开发**：使用 Cloudflare Tunnel 实现本地 HTTPS 代理，方便开发调试
5. **生产部署**：使用 cert-manager 管理证书，实现生产环境的自动化部署

通过 Mutating Webhook，我们可以实现 Kubernetes 资源的自动化管理，提高运维效率和系统一致性。

## 8. 参考资料

- [Kubernetes准入控制](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/)
- [cert-manager文档](https://cert-manager.io/docs/)
- [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/private-net/cloudflared/connect-private-hostname/)