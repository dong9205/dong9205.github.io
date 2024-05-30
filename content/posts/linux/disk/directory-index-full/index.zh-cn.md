---
weight: 0
title: "单个目录下文件爆满"
subtitle: ""
date: 2024-05-30T20:34:24+08:00
lastmod: 2024-05-30T20:34:24+08:00
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

# 报错

## 创建文件报错

```bash
No space left on device
```

## syslog日志错误

```bash
Directory (ino: 4230742) index full, reach max htree level :2
Large directory feature is not enabled on this filesystem
```


# 背景

1. 在data目录下无法创建file-1, 但是可以创建file-2, 通过`df -hT` 和 `df -hi`确认不是磁盘不足、不是inode不足导致
2. 经过排查是因为目录下的文件超过了目录文件限制.  但是删除一些文件后，创建file-1还是报错
3. 话说重启解决99%的问题，所以我重启发现这是剩余的1%的问题.

## 出现问题的原因
* ext4自身没有做相关方面的限制
* Linux 2.6.23以后引入的HTree索引做的，2-level HTree索引限定了约10-12 million（百万）
* Linux 4.12后largedir特性开启的话3-level限定了约6 billion（十亿）

https://en.wikipedia.org/wiki/Ext4
```bash
**Unlimited number of subdirectories**
ext4 does not limit the number of subdirectories in a single directory, except by the inherent size limit of the directory itself. (In ext3 a directory can have at most 32,000 subdirectories.)[17][obsolete source] To allow for larger directories and continued performance, ext4 in Linux 2.6.23 and later turns on HTree indices (a specialized version of a B-tree) by default, which allows directories up to approximately 10–12 million entries to be stored in the 2-level HTree index and 2 GB directory size limit for 4 KiB block size, depending on the filename length. In Linux 4.12 and later the large_dir feature enabled a 3-level HTree and directory sizes over 2 GB, allowing approximately 6 billion entries in a single directory.
```

# 解决方案

## tune2fs启用磁盘large_dir 
> 我测试过一次这种方案，当时出现了系统异常
```bash
tune2fs -O large_dir /dev/mapper/vg_test-lv_home
```

## 重建目录
```bash
mv data data-bak
mkdir data
find data-bak/ -type f   | xargs -I {} -n 1  mv {} data/
```

# 参考连接
http://blog.wafcloud.cn/linux/linux-ext4-reach-max-htree-level2.html

