---
title: Jumpserver Celery 容器 OOM 排查纪实
subtitle:
date: 2025-12-10T22:56:17+08:00
slug: jumpserver-celery-oom
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
  - JumpServer
  - 问题记录
categories:
  - JumpServer
  - 问题记录
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRelated: false
hiddenFromFeed: false
summary:
resources:
  - name: jumpserver
    src: jumpserver.png
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
## 问题描述

环境信息：Jumpserver v2.14.2

某日正在跳板机上执行操作时，系统突然卡顿无响应。检查后发现服务器内存已耗尽，但系统未触发 OOM killer，导致内存持续居高不下。由于无法通过 SSH 正常登录，只能通过阿里云控制台强制重启服务器。重启后系统恢复正常，但几分钟后内存再次被占满。经过排查，确认问题由 Jumpserver 的 Celery 容器引起。再次重启后，临时关闭 Celery 容器，系统恢复正常运行。

## 🕵️问题排查

经过排查分析，发现内存激增的根本原因在于 **/opt/jumpserver/apps/terminal/tasks.py** 文件中 `clean_expired_session_period` 函数的实现。原本的写法 `expired_sessions.delete()` 会将所有符合条件的过期会话一次性加载到内存并删除，容易导致容器瞬时占用大量内存，从而引发内存问题。

## 🔍技术原理说明

Django ORM 的 QuerySet `.delete()` 方法在执行删除操作时，会先将所有匹配的对象完整实例加载到内存中（包括关联对象），然后再执行删除。这意味着：

1. **内存占用高**：每个对象实例都会占用内存，如果对象包含外键关联，还会加载关联对象
2. **级联删除**：Django 会处理外键的级联删除，进一步增加内存占用
3. **一次性加载**：所有数据一次性加载，没有分批处理机制

当过期数据量很大（比如几十万甚至上百万条记录）时，会导致内存占用急剧上升。

## ✨优化方案

采用分批删除策略，核心思路是**只获取 ID，分批处理**：

1. **只获取 ID**：使用 `values_list('id', flat=True)` 只获取主键 ID，不加载完整对象实例，大幅减少内存占用
2. **分批处理**：每次处理 1000 条记录（`batch_size = 1000`），循环删除，避免一次性加载大量数据
3. **进度监控**：添加详细的进度日志，便于监控删除进度和排查问题

批次大小选择 1000 是一个平衡点：太小会增加数据库查询次数，太大仍可能导致内存压力。

### 💡原始代码

```python
@shared_task
@register_as_period_task(interval=3600*24)
@after_app_ready_start
@after_app_shutdown_clean_periodic
def clean_expired_session_period():
    logger.info("Start clean expired session record, commands and replay")
    days = get_log_keep_day('TERMINAL_SESSION_KEEP_DURATION')
    expire_date = timezone.now() - timezone.timedelta(days=days)
    expired_sessions = Session.objects.filter(date_start__lt=expire_date)
    timestamp = expire_date.timestamp()
    expired_commands = Command.objects.filter(timestamp__lt=timestamp)
    replay_dir = os.path.join(default_storage.base_location, 'replay')

    expired_sessions.delete()
    logger.info("Clean session item done")
    expired_commands.delete()
    logger.info("Clean session command done")
    command = "find %s -mtime +%s -name '*.gz' -exec rm -f {} \\;" % (
        replay_dir, days
    )
    subprocess.call(command, shell=True)
    command = "find %s -type d -empty -delete;" % replay_dir
    subprocess.call(command, shell=True)
    logger.info("Clean session replay done")
```

### 🌱优化后代码

```python
@shared_task
@register_as_period_task(interval=3600*24)
@after_app_ready_start
@after_app_shutdown_clean_periodic
def clean_expired_session_period():
    logger.info("Start clean expired session record, commands and replay")
    days = get_log_keep_day('TERMINAL_SESSION_KEEP_DURATION')
    expire_date = timezone.now() - timezone.timedelta(days=days)
    timestamp = expire_date.timestamp()
    # 优化1: 分批删除 Session，避免一次性加载大量数据到内存
    batch_size = 1000
    deleted_sessions = 0

    while True:
        # 只获取ID，不加载完整对象
        expired_session_ids = list(
            Session.objects.filter(date_start__lt=expire_date)
            .values_list('id', flat=True)[:batch_size]
        )

        if not expired_session_ids:
            break

        # 批量删除
        Session.objects.filter(id__in=expired_session_ids).delete()
        deleted_sessions += len(expired_session_ids)
        logger.info(f"Deleted {deleted_sessions} sessions so far")
    logger.info(f"Clean session item done, total deleted: {deleted_sessions}")
    # 优化2: 分批删除 Command
    deleted_commands = 0

    while True:
        expired_command_ids = list(
            Command.objects.filter(timestamp__lt=timestamp)
            .values_list('id', flat=True)[:batch_size]
        )

        if not expired_command_ids:
            break

        Command.objects.filter(id__in=expired_command_ids).delete()
        deleted_commands += len(expired_command_ids)
        logger.info(f"Deleted {deleted_commands} commands so far")

    logger.info(f"Clean session command done, total deleted: {deleted_commands}")
    replay_dir = os.path.join(default_storage.base_location, 'replay')

    command = "find %s -mtime +%s -name '*.gz' -exec rm -f {} \\;" % (
        replay_dir, days
    )
    subprocess.call(command, shell=True)
    command = "find %s -type d -empty -delete;" % replay_dir
    subprocess.call(command, shell=True)
    logger.info("Clean session replay done")
```

## 🌈优化效果

优化后的代码在实际运行中取得了显著效果：
* **内存占用大幅降低**：从一次性加载全部数据（可能占用数 GB 内存）降低到每次仅处理 1000 条记录（仅占用几 MB 内存）
* **可监控性提升**：通过详细的进度日志，可以实时监控删除进度，便于问题排查
* **系统稳定性提升**：Celery 容器不再因内存问题导致系统卡顿

经过优化后，该清理任务已稳定运行，未再出现内存问题。