---
weight: 0
title: "istio 常见问题: 返回 426 状态码"
subtitle: ""
date: 2023-12-28T10:44:47+08:00
lastmod: 2023-12-28T10:44:47+08:00
draft: false
summary: ""
license: ""
tags: [""]
resources:
- name: "featured-image"
  src: ""
categories: 
- "Istio"

featuredImage: ""
featuredImagePreview: ""

hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRelated: false
hiddenFromFeed: false
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
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/370046252)

> 本文摘自 **[istio 学习笔记](https://link.zhihu.com/?target=https%3A//imroc.cc/learning-istio/faq/426-status-code.html)**  

**背景**
------

Istio 使用 Envoy 作为数据面转发 HTTP 请求，而 Envoy 默认要求使用 HTTP/1.1 或 HTTP/2，当客户端使用 HTTP/1.0 时就会返回 `426 Upgrade Required`。

**常见的 nginx 场景**
----------------

如果用 nginx 进行 `proxy_pass` 反向代理，默认会用 HTTP/1.0，你可以显示指定 **[proxy_http_version](https://link.zhihu.com/?target=https%3A//nginx.org/en/docs/http/ngx_http_proxy_module.html%23proxy_http_version)** 为 `1.1`:

```
upstream http_backend {
    server 127.0.0.1:8080;

    keepalive 16;
}

server {
    ...

    location /http/ {
        proxy_pass http://http_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        ...
    }
}

```

**压测场景**
--------

ab 压测时会发送 HTTP/1.0 的请求，Envoy 固定返回 426 Upgrade Required，根本不会进行转发，所以压测的结果也不会准确。可以换成其它压测工具，如 **[wrk](https://link.zhihu.com/?target=https%3A//github.com/wg/wrk)** 。

**让 istio 支持 HTTP/1.0**
-----------------------

有些 SDK 或框架可能会使用 HTTP/1.0 协议，比如使用 HTTP/1.0 去资源中心 / 配置中心 拉取配置信息，在不想改动代码的情况下让服务跑在 istio 上，也可以修改 istiod 配置，加上 `PILOT_HTTP10: 1` 的环境变量来启用 HTTP/1.0。

**参考资料**
--------

*   **[Envoy won’t connect to my HTTP/1.0 service](https://link.zhihu.com/?target=https%3A//istio.io/latest/docs/ops/common-problems/network-issues/%23envoy-won-t-connect-to-my-http-1-0-service)**