---
weight: 0
title: "部署Headscale"
subtitle: ""
date: 2024-09-10T10:29:30+08:00
lastmod: 2024-09-10T10:29:30+08:00
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

toc:
  enable: true
  auto: true

mapbox:
share:
  enable: true
comment:
  enable: true

lightgallery: true
---
# 服务端
## 部署

```bash
mkdir /opt/go/src/github.com/juanfont
cd /opt/go/src/github.com/juanfont
git clone https://github.com/juanfont/headscale.git
cd headscale
mkdir config
touch config/db.sqlite
cp config-example.yaml config/config.yaml
vim config/config.yaml 
docker run --name=headscale --user=0 --volume=/opt/go/src/github.com/juanfont/headscale/config:/etc/headscale/ --workdir=/ -p 8080:8080 -p 9090:9090 --restart=always --runtime=runc --detach=true headscale/headscale:v0.23.0-beta.4 serve
echo "alias headscale='docker container exec headscale headscale'" >> ~/.bashrc && . ~/.bashrc
```

## 配置用户

```bash
headscale user create test01
```
# 客户端

## 安装客户端

> https://tailscale.com/download

## 配置客户端

```bash
tailscale login --login-server https://headscale.xxx.com
```

## 服务端认证客户端
```bash
headscale nodes register --user test01 --key mkey:xxxxx
```

# 参考文档
https://icloudnative.io/posts/how-to-set-up-or-migrate-headscale/
https://github.com/juanfont/headscale
