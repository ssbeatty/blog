---
title: oms使用插件安装vnc
date: 2023-04-17T15:25:05+08:00
lastmod: 2023-04-17T15:25:05+08:00
author: sasaba
cover: /img/oms使用插件安装vnc.jpg
images:
- /img/oms使用插件安装vnc.jpg
categories:
  - 工具 
tags:
  - OMS
draft: true
---

oms使用插件安装vnc。

<!--more-->

## OMS简介

[OMS](https://github.com/ssbeatty/oms)是一个开源运维管理系统，它的使用场景跟接近于本地的、简易的、一定规模的运维管理系统。

## 如何安装和下载

[release](https://github.com/ssbeatty/oms/releases)里面下载最新的版本，然后放到本地一个干净的文件夹内并重命名为oms，然后执行`./oms`即可。

## 如何使用

1. **添加主机和相关配置**

因为本文主要讲插件的使用，所以这里就不详细讲解如何添加主机了，可以参考[这里](https://wang918562230.gitbook.io/ssbeattyoms-wen-dang/guides/zi-chan-guan-li)。

2. **安装插件**

在[这里](https://github.com/ssbeatty/oms_plugins)可以下载到所有的插件。

![download.png](/context_img/oms_plugin/download.png)

下载并解压缩后，选择需要的插件目录这里以vnc_install为例。

![zip.png](/context_img/oms_plugin/zip.png)

打开oms页面的资产-剧本-导入插件 并选择vnc_install.zip文件和点击**导入文件**。
![import_page.png](/context_img/oms_plugin/import_page.png)


**或者直接将vnc_install目录拷贝到oms同级的data/plugin/src目录下。**


3. **重启oms**

4. **使用插件**
![add_plugin.png](/context_img/oms_plugin/add_plugin.png)
打开oms页面的资产-剧本-添加剧本 并选择vnc_install插件，填写相关参数并点击**确定**。

![exec.png](/context_img/oms_plugin/exec.png)
然后在oms页面的资产-运维模式-执行命令 并选择刚才添加的剧本，点击**执行**。

等待返回日志即可。

5. **使用vnc**

![vnc_use.png](/context_img/oms_plugin/vnc_use.png)

![vnc.png](/context_img/oms_plugin/vnc.png)

## 其他的问题

### 开机自动登录root
因为x11vnc使用的是lightdm，所以如果需要使用vnc的话，需要在主机上安装lightdm。

这个是插件自行安装切换的，因此需要修改开机进入root的操作时，需要修改如下配置文件：

> /usr/share/lightdm/lightdm.conf.d/50-ubuntu.conf

```text
autologin-user=root
greeter-show-manual-login=true
```