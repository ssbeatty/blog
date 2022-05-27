---
title: 记一次文件系统损坏
date: 2022-05-28T00:08:05+08:00
lastmod: 2022-05-28T00:08:05+08:00
author: sasaba
cover: /img/记一次文件系统损坏.jpg
images:
  - /img/记一次文件系统损坏.jpg
categories:
  - 文件系统
tags:
  - Linux
draft: true
---

记一次docker文件系统损坏。

<!--more-->

## 起因

某次突然发现正常运行的镜像一直在报`错exec odoo: cannot execute: Is a directory`

经过检查后发现镜像没有改变过，而且文件目录也是正常的。。

## 解决

### 通过exec bash镜像

通过exec bash进入镜像后发现`which odoo`为空，进入可执行文件的目录`/usr/local/bin`时报错

```text
cd /usr/local/bin error: Structure needs cleaning
```

经过谷歌后发现这个是因为文件系统发生了严重损坏导致的，因此把目标放在了挂载硬盘上。

### docker 路径验证

经过查看发现docker的目录`/var/lib/docker`通过软连接指向了`/dev/sda`，于是在停止了docker后通过复制目录来验证是否文件系统真的损坏。

于是在复制docker目录的过程中又报错`Structure needs cleaning`。


### 解决

发现问题后开始进行解决: 

```shell
unmount -v /dev/sda # 取消挂载文件系统
sudo fsck.ext4 /dev/sda # 修复文件系统损坏
mount /dev/sda /data # 重新挂载
```

然后重启docker并重新创建docker容器。问题解决。



















