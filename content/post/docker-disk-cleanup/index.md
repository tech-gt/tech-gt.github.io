---
title: "记一次 Docker overlay2 目录导致磁盘爆满的排查与解决"
date: 2024-01-10T15:30:00+08:00
categories: ["运维", "Docker"]
tags: ["Docker", "磁盘清理", "overlay2", "故障排查"]
description: "详细记录一次因 Docker overlay2 目录膨胀导致服务器磁盘空间耗尽的故障排查过程，并提供一套行之有效的清理方案与最佳实践。"
---

## 一、问题背景

近日，收到开发环境一台服务器的磁盘空间告警，提示根分区使用率已达 96%。该服务器主要用于运行各类应用的 Docker 容器，日常数据量和日志增长相对稳定，因此初步判断问题可能出在 Docker 本身。本文旨在记录完整的排查思路与最终的解决方案。

## 二、排查过程

1.  **确认磁盘使用情况**

    首先，登录服务器，使用 `df -h` 命令检查磁盘各分区的详细使用情况。

    ```bash
    $ df -h
    Filesystem      Size  Used Avail Use% Mounted on
    /dev/vda1        100G   96G  4.0G  96% /
    ...
    ```

    输出证实了根分区 `/` 确实即将耗尽。

2.  **定位大文件目录**

    接下来，需要找出是哪个目录占用了大量空间。我们从根目录开始，使用 `du -sh /*` 命令逐层分析。

    ```bash
    # 为了避免权限问题，建议使用 sudo
    $ sudo du -sh /* | sort -rh
    78G     /var
    15G     /usr
    ...
    ```
    很快定位到 `/var` 目录异常庞大。继续深入排查 `/var` 目录。

    ```bash
    $ sudo du -sh /var/* | sort -rh
    75G     /var/lib
    ...
    ```

    最终，我们将目标锁定在 `/var/lib/docker` 目录，它占用了超过 70G 的空间。

3.  **分析 Docker 磁盘占用**

    既然确定是 Docker 的问题，就可以使用 Docker 自带的命令进行分析。`docker system df` 是一个非常有用的命令，可以清晰地列出 Docker 各类资源的占用情况。

    ```bash
    $ docker system df
    TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
    Images          58        12        25.8GB    22.1GB (85%)
    Containers      15        10        1.2GB     250MB (20%)
    Local Volumes   22        5         8.9GB     6.5GB (73%)
    Build Cache     541       0         35.6GB    35.6GB (100%)
    ```

    结果一目了然：
    - **Images**：存在大量未使用的镜像，可回收空间巨大。
    - **Local Volumes**：存在许多无主（dangling）的数据卷。
    - **Build Cache**：构建缓存占用了惊人的 35.6GB，且全部可以回收。

    这些可回收空间（RECLAIMABLE）的总和，正是导致我们磁盘爆满的元凶。

## 三、原因剖析

`overlay2` 是 Docker 使用的默认存储驱动，它负责管理镜像层和容器的读写层。我们拉取的每一个镜像、构建的每一个中间层、运行的每一个容器，都会在 `/var/lib/docker/overlay2` 目录下留下数据。随着时间推移，以下几类资源会不断累积：
- **悬空镜像（Dangling Images）**：镜像的新版本覆盖了旧版本后，没有标签（tag）的旧镜像层并未被删除。
- **停止运行的容器**：执行 `docker stop` 后的容器虽然不占用计算资源，但其文件系统层依然占用磁盘。
- **无主数据卷（Dangling Volumes）**：删除了容器但未删除其关联的数据卷，这些卷会一直存在。
- **构建缓存（Build Cache）**：`docker build` 过程中会产生大量的中间镜像层作为缓存，以加速后续构建，这是最容易被忽视的“空间大户”。

## 四、解决方案

针对以上问题，Docker 提供了强大的 `prune` 系列命令用于清理。

### 方案一：一键“瘦身”（适用于清理开发环境）

对于开发或测试环境，可以使用以下命令进行一次彻底的清理，它会删除所有未被容器使用的资源。

```bash
# -a: 清理所有未被容器引用的资源，而不仅仅是悬空的
# --volumes: 同时清理无主的数据卷
# -f: 强制执行，无需交互式确认
docker system prune -a -f --volumes
```

**警告**：`--volumes` 参数会删除所有未被任何现有容器引用的数据卷。在生产环境执行前，请务必确认这些数据卷不再需要，否则可能导致数据永久丢失。

### 方案二：精细化清理（推荐）

更安全、更可控的方式是分类进行清理。

1.  **清理已停止的容器**
    ```bash
    docker container prune -f
    ```

2.  **清理无用镜像**
    ```bash
    # 只清理悬空镜像
    docker image prune -f
    
    # 清理所有未被使用的镜像（慎用）
    docker image prune -a -f
    ```

3.  **清理无用数据卷**
    ```bash
    docker volume prune -f
    ```

4.  **清理构建缓存**
    ```bash
    docker builder prune -f
    ```

执行完上述清理操作后，再次运行 `df -h`，可以看到磁盘空间被成功释放。

## 五、建立长效机制

为了避免问题复现，建议建立自动化的清理机制。例如，可以设置一个 `cron` 定时任务，在业务低峰期（如每日凌晨）自动执行清理。

```bash
# 编辑 crontab
crontab -e

# 添加以下任务，每天凌晨3点执行清理
0 3 * * * /usr/bin/docker system prune -f --volumes > /dev/null 2>&1
```

同时，在编写 `Dockerfile` 时，应遵循最佳实践，例如合并多个 `RUN` 命令、使用 `.dockerignore` 文件，以减少镜像体积和构建缓存。
