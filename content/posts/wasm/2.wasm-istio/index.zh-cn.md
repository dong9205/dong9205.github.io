---
weight: 0
title: "使用 WebAssembly 对 Istio 进行扩展"
subtitle: ""
date: 2024-01-26T16:27:41+08:00
lastmod: 2024-01-26T16:27:41+08:00
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


lightgallery: false
---

# 使用 WebAssembly 对 Istio 进行扩展

## Proxy-Wasm


编写 WASM 的工具有 Solo.io 团队的 `wasme`、`tinygo`等，目前应用比较多是 `tinygo`，`tinygo` 支持的包可以查看 https://tinygo.org/docs/reference/lang-support/stdlib/ 进行了解。

> TinyGo 是 Go 编程语言规范的一个编译器实现，为什么不使用官方的 Go 编译器？目前官方编译器无法生成可以在浏览器外部运行的 WASM 二进制文件，因此也无法生成与 Proxy-Wasm 兼容的二进制文件。

### Proxy-Wasm

Proxy-Wasm是开源社区针对「网络代理场景」设计的一套 ABI 规范，定义了网络代理和运行在网络代理内部的 Wasm 虚拟机之间的接口，属于当前的事实规范。当前支持该规范的网络代理软件包括 Envoy、MOSN 和 ATS(Apache Traffic Server)，支持该规范的 Wasm 扩展 SDK 包括 C++、Rust 和 Go。采用该规范的好处在于能让 Wasm 扩展程序在不同的网络代理产品上运行，比如 MOSN 的 Wasm 扩展程序可以运行在 Envoy 上，而 Envoy 的 Wasm 扩展程序也可以运行在 MOSN 上。


`Proxy-Wasm` 规范定义了宿主机与 Wasm 扩展程序之间的交互细节，包括 API 列表、函数调用规范以及数据传输规范这几个方面。其中，API 列表包含了 L4/L7、property、metrics、日志等方面的扩展点，涵盖了网络代理场景下所需的大部分交互点。目前实现该规范的 Wasm 扩展 SDK 包括 AssemblyScript、C++、Rust 和 Go：

* AssemblyScript SDK
* C++ SDK
* Go (TinyGo) SDK
* Rust SDK

为了方便，我们也直接选择已有的 `proxy-wasm-go-sdk` 这个 SDK 进行开发。这个 Proxy-Wasm Go SDK 是用于使用 Go 编程语言在 Proxy-Wasm ABI 规范之上扩展网络代理（例如 Envoyproxy）的 SDK，有了这个 SDK，每个人都可以轻松地生成与 Proxy-Wasm 规范兼容的 Wasm 二进制文件，而无需了解低级且对于没有专业知识的人来说难以理解的 Proxy-Wasm ABI 规范。

## 参考文档
[使用 WebAssembly 对 Istio 进行扩展](https://mp.weixin.qq.com/s/MEUeKZ6Bdnecy41SuAw0LA)