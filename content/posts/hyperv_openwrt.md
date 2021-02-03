---
title: Hyperv安装openwrt
date: 2021-01-26T11:46:09+08:00
lastmod: 2021-01-26T17:46:09+08:00
author: sasaba
cover: /img/fc1.jpg
images:
  - /img/fc1.jpg
categories:
  - 路由器
tags:
  - 技术
  - 虚拟化
draft: true
---

使用hyper-v安装openwrt路由器，实现透明代理。

<!--more-->

## 安装Hyper-v

### 打开硬件虚拟化

这个intel和amd的芯片叫法是不一样的，但是异曲同工。这里不再赘述，直接百度怎么打开intel(amd)虚拟化即可。

### 安装HyperV

hyper-v默认只支持windows10专业版和windows server。首先打开控制面板点击程序

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126130957.png" style="zoom: 50%;" />

选择启用或关闭windows功能

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126131105.png" style="zoom:50%;" />

勾选hyper-v并点击确认，重启电脑即可

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126131136.png" style="zoom: 67%;" />

如果你是家庭版请参考

[家庭版win10打开hyper-v](https://jingyan.baidu.com/article/d7130635e5678113fcf4757f.html)

## 配置网络交换机

开始菜单搜索并打开Hyper-v管理器，打开右侧虚拟交换机管理器

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126131414.png" style="zoom:50%;" />

创建一个外部虚拟交换机

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126131516.png" style="zoom:50%;" />

改名为WAN并选择你的上网网卡，同时确认允许管理操作系统共享此网络适配器处于勾选状态

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126131802.png" style="zoom:50%;" />

按照刚才的方法再创建一个内部网络

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126131722.png" style="zoom:50%;" />

选项默认并命名为LAN

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126131908.png" style="zoom:50%;" />

## 准备固件

### 自编译

请参考我的另一个文章

[openwrt编译](https://www.sasaba.net/posts/openwrt/)

> 这里要注意一个问题，因为编译出来的虚拟磁盘默认分配的空间是比较小的，如果需要装插件肯定是不太够的。因此推荐在make menuconfig后修改生成的`.config`文件

找到如下两行如此修改即可

```
CONFIG_TARGET_KERNEL_PARTSIZE=200
CONFIG_TARGET_ROOTFS_PARTSIZE=500
```

### 去恩山论坛上白嫖

附上我自己编译的固件，包含pass、plus和docker。

> 链接：https://pan.baidu.com/s/1Hj6GGMOeQetoCnrnNBPivg 
>
> 提取码：n1ik 
>
> 复制这段内容后打开百度网盘手机App，操作更方便哦--来自百度网盘超级会员V4的分享

### 转换固件

使用的软件是`StarWind Software V2V Image Converter`可以去[StarWind Software](https://www.starwindsoftware.com/starwind-v2v-converter)下载

1. 启动 StarWind Software V2V Image Converter，启动后会看到一个 StarWind 自家的广告页面，直接点击下一步即可。 「Source image location」原始镜像位置选择「local file」也就是本地文件。然后点击「Next」。 接下来是选择镜像文件，点击输入框右侧的按钮，选择我们刚才解压得到的 700 多 MB 的 img 文件，点击「Next」

   <img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126132900.png" style="zoom:67%;" />

   <img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126132920.png" style="zoom:67%;" />

2. 「Image Format」选择「Microsoft VHDX Image」，点击「Next」。 

   <img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126132945.png" style="zoom:67%;" />

3. 再下一页中的「Activate Windows Repair Mode」，注意不要勾选。 

   <img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126133016.png" style="zoom:80%;" />

4. 接下来会让你选择转换后的文件储存在哪里，默认会存在原始文件所在的目录。接下来就会开始将 img 镜像文件转换为 Hyper-V 使用的 vhdx 虚拟硬盘映像文件了。转换过程很快，等进度条跑到 100 % 以后就可以点击「Finish」退出了。

## 创建虚拟机

打开hyper-v管理器点击新建-虚拟机

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126160846.png" style="zoom: 50%;" />

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126160909.png" style="zoom:50%;" />

取消勾选动态内存

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126160934.png" style="zoom:50%;" />

选择外部网络

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126161023.png" style="zoom:50%;" />

选择刚才生成的虚拟磁盘并点击完成

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126161114.png" style="zoom:50%;" />

选择新建的虚拟机点击设置

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126162412.png" style="zoom:50%;" />

添加一个网络适配器

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126162439.png" style="zoom:50%;" />

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126162512.png" style="zoom:50%;" />

选择网卡高级功能-启用MAC地址欺诈

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126162736.png" style="zoom:50%;" />

再次配置虚拟交换机，取消允许管理操作系统共享此网络适配器

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126162927.png" style="zoom: 50%;" />

启动虚拟机并配置WAN网卡为dhcp确保虚拟机上网。这一步只要有linux基础应该都会，不再赘述。

## 配置网络

打开本地网络适配器并选择LAN网络-属性

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126163206.png" style="zoom:50%;" />

选择IPV4配置

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126163303.png" style="zoom:67%;" />



按照下图配置openwrt网关即可访问配置

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210126163401.png" style="zoom:67%;" />