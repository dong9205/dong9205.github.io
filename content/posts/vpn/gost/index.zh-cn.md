---
weight: 0
title: "gost配置反向代理隧道"
subtitle: ""
date: 2024-09-11T19:47:08+08:00
lastmod: 2024-09-11T19:47:08+08:00
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

# gost配置反向代理隧道

## 初始化

```bash
mkdir -p /opt/go/src/github.com/go-gost
cd /opt/go/src/github.com/go-gost
git@github.com:go-gost/gost.git
mkdir gost-plus
cd gost-plus/
```

## 配置gost.yaml

```yaml
services:
- name: service-0
  addr: :8080
  handler:
    type: tunnel
    metadata:
      entrypoint: :8000
      ingress: ingress-0
      #scheme: https
  listener:
    type: ws

ingresses:
- name: ingress-0
  plugin:
    type: grpc
    addr: gost-plugins:8000

log:
  level: info
```
## 配置docker-compose.yaml

```yaml
version: '3'

services:
  gost-tunnel: 
    image: gogost/gost
    restart: always
    ports:
      - "8081:8081"
      - "8082:8082"
    volumes:
      - /opt/go/src/github.com/go-gost/gost-plus/gost-plus:/etc/gost/

  gost-plugins: 
    image: ginuerzh/gost-plugins
    restart: always
    command: "ingress --addr=:8082 --redis.addr=redis:6379 --redis.db=2 --redis.expiration=24h --domain=gost.p-pp.cn --log.level=debug"

  redis: 
    image: redis:7.2.1-alpine
    restart: always
    command: "redis-server --save 60 1 --loglevel warning"
    volumes:
      - redis:/data

volumes:
  redis:
    driver: local
```

## 启动
```bash
docker compose up -d
```
## 配置nginx

```bash
cat /etc/nginx/sites-enabled/tunnel.gost.p-pp.cn.conf 
upstream internal.infra.gost {
        server localhost:8081;
}

server {
        listen 80;
        server_name tunnel.gost.p-pp.cn;
        return 308 https://$host$request_uri;
}

server {
        listen 443 ssl;
        server_name tunnel.gost.p-pp.cn;
        ssl_certificate gost-cert/tunnel.gost.p-pp.cn.pem;
        ssl_certificate_key gost-cert/tunnel.gost.p-pp.cn.key;
        ssl_protocols TLSv1.2;
        ssl_ciphers ECDHE-RSA-AES256-SHA384:AES256-SHA256:RC4:HIGH:!MD5:!aNULL:!eNULL:!NULL:!DH:!EDH:!AESGCM;
        ssl_session_cache shared:SSL_WS2:500m;
        ssl_session_timeout 10m;
        ssl_prefer_server_ciphers on;
        access_log /var/log/nginx/tunnel.gost.p-pp.cn.access.log body_main;
        error_log /var/log/nginx/tunnel.gost.p-pp.cn.access.log;

        location / {
                proxy_pass_header server;
                proxy_set_header host $http_host;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header x-real-ip $remote_addr;
                proxy_set_header x-scheme $scheme;
                proxy_http_version 1.1;
                proxy_set_header Connection keep-alive;
                proxy_set_header Keep-Alive 3600;
                keepalive_timeout 3600;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
                proxy_pass http://internal.infra.gost;
        }
}

cat /etc/nginx/sites-enabled/gost.p-pp.cn.conf 
upstream internal.infra.gost-plugin {
        server localhost:8082;
}

server {
        listen 80;
        server_name *.gost.p-pp.cn;
        return 308 https://$host$request_uri;
}

server {
        listen 443 ssl;
        server_name *.gost.p-pp.cn;
        ssl_certificate gost-cert/tunnel.gost.p-pp.cn.pem;
        ssl_certificate_key gost-cert/tunnel.gost.p-pp.cn.key;
        #ssl_certificate gost-cert/$ssl_server_name.pem;
        #ssl_certificate_key gost-cert/$ssl_server_name.key;
        ssl_protocols TLSv1.2;
        ssl_ciphers ECDHE-RSA-AES256-SHA384:AES256-SHA256:RC4:HIGH:!MD5:!aNULL:!eNULL:!NULL:!DH:!EDH:!AESGCM;
        ssl_session_cache shared:SSL_WS2:500m;
        ssl_session_timeout 10m;
        ssl_prefer_server_ciphers on;
        access_log /var/log/nginx/gost.p-pp.cn.access.log main;
        error_log /var/log/nginx/gost.p-pp.cn.access.log;

        location / {
                proxy_pass_header server;
                proxy_set_header host $http_host;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header x-real-ip $remote_addr;
                proxy_set_header x-scheme $scheme;
                proxy_http_version 1.1;
                proxy_set_header Connection keep-alive;
                proxy_set_header Keep-Alive 3600;
                keepalive_timeout 3600;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
                proxy_pass http://internal.infra.gost-plugin;
        }
}
```

## 配置域名解析

| 域名           | 类型  | 地址           |
| -------------- | ----- | -------------- |
| gost.p-pp.cn   | CNAME | host02.p-pp.cn |
| *.gost.p-pp.cn | CNAME | host02.p-pp.cn |

## 客户端下载
> https://github.com/go-gost/gost/releases

## 客户端连接

```bash
./gost -L rtcp://:/127.0.0.1:3000 -F "tunnel+wss://tunnel.gost.p-pp.cn:443"
{"handler":"rtcp","kind":"service","level":"info","listener":"rtcp","msg":"listening on :0/tcp","service":"service-0","time":"2024-09-11T21:30:01.587+08:00"}
{"connector":"tunnel","dialer":"wss","endpoint":"065d67d07b0a820f","hop":"hop-0","kind":"connector","level":"info","msg":"create tunnel on 065d67d07b0a820f:0/tcp OK, tunnel=d101599e-7040-11ef-a7f4-c4c6e6b12218, connector=03a48873-fdbd-4b49-a034-f517d2e69c29, weight=0","node":"node-0","time":"2024-09-11T21:30:01.656+08:00","tunnel":"d101599e-7040-11ef-a7f4-c4c6e6b12218"}
```

## 访问测试

065d67d07b0a820f.gost.p-pp.cn

## 参考文档

https://gost.run/blog/2023/gost-plus/
https://gost.run/tutorials/reverse-proxy-tunnel/