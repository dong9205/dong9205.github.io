---
weight: 0
title: "单点登录报错记录"
subtitle: ""
date: 2024-10-11T20:52:22+08:00
lastmod: 2024-10-11T20:52:22+08:00
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

# SAML登录异常错误记录

## Missing signature referencing the top-level element

### 原因

使用的Google的Saml协议接入单点登录, 后端配置了验证，需要勾选"**已签署响应复选框**"

### 排查细节

1. 首先通过报错发现了具体的错误函数，发现是通过调用了goxmldsig.NewDefaultValidationContext创建的ValidationContext结构体ctx,然后ctx的Validate函数的报错
2. 继续Debug，发现是findSignature下sig的结果为空，导致返回报错"Missing signature referencing the top-level element"
3. sig为空的原因是"idAttr"和"ref.URI"的不相同.
4. 然后找到其他正常的请求发现XML里边有ds:Signature字段，但是Google的saml返回结果里边没有这个字段
5. 然后在平台上找到"**已签署响应复选框**"选项
