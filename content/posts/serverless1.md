---
title: 腾讯云serverless部署hugo博客
date: 2021-02-08T15:47:00+08:00
lastmod: 2021-02-08T16:14:00+08:00
author: sasaba
cover: /img/腾讯云serverless部署hugo博客.jpg
images:
  - /img/腾讯云serverless部署hugo博客.jpg
categories:
  - 云计算
tags:
  - 技术
---

怎么使用腾讯云serverless部署hugo博客。

<!--more-->

## 前提&原因&目的

### 缘由

很早之前有想法去做一个个人博客，但是如果是传统的博客，部署上会有些麻烦。如果服务器到期了，或者不常去维护，再或者没有一个好的文本编辑系统。总之就是非常的鸡肋，所以想去换一个方式达成目的。

最近刚好腾讯云再推serverless，操作简单方便，配合coding更是简单易用（其实也有不少的波折），于是就打算尝试下静态博客，也体验下serverless的生态。

### 前提

需要去了解一下hugo搭建博客的过程，其他静态博客系统如hexo也可以，它们的原理都是把markdown直接编译成html的静态文件，然后通过静态服务器代理就可以了。本文的前提至少需要一个编译后的文件夹/public或者/static。



