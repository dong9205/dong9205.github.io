---
weight: 0
title: "将Mysql查询内容转换为JSON"
subtitle: ""
date: 2024-05-30T20:25:28+08:00
lastmod: 2024-05-30T20:25:28+08:00
draft: false
summary: ""
license: ""
tags: ["mysql","json"]
resources:
- name: "featured-image"
  src: ""
categories: 
- "MySQL"

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

# 安装命令
```bash
pip install csvkit mycli
```

# 打印JSON格式

## 在Mycli终端中打印

```bash
 $ mycli -P 3306 -proot mysql
MySQL root@localhost:mysql> \T csv
MySQL root@localhost:mysql> pager csvjson | jq '.'

```

## 在Linux命令行打印

```bash
mycli -P 3306 -proot mysql --csv -e "select * from user;" | csvjson | jq .

```

# 参考连接
https://github.com/dbcli/mycli/issues/753#issuecomment-504279817