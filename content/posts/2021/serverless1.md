---
title: 腾讯云serverless部署hugo博客
date: 2021-02-08T15:47:00+08:00
lastmod: 2021-02-09T13:14:00+08:00
author: sasaba
cover: /img/腾讯云serverless部署hugo博客.jpg
images:
  - /img/腾讯云serverless部署hugo博客.jpg
categories:
  - 云计算
tags:
  - serverless
---

怎么使用腾讯云serverless部署hugo博客。

<!--more-->

## 前提&原因&目的

### 缘由

很早之前有想法去做一个个人博客，但是如果是传统的博客，部署上会有些麻烦。如果服务器到期了，或者不常去维护，再或者没有一个好的文本编辑系统。总之就是非常的鸡肋，所以想去换一个方式达成目的。

最近刚好腾讯云再推serverless，操作简单方便，配合coding更是简单易用（其实也有不少的波折），于是就打算尝试下静态博客，也体验下serverless的生态。

### 前提

需要去了解一下hugo搭建博客的过程，其他静态博客系统如hexo也可以，它们的原理都是把markdown直接编译成html的静态文件，然后通过静态服务器代理就可以了。本文的前提至少需要一个编译后的文件夹/public或者/static。

### 目的

做每一件事都必须有一个明确的目标，否则做的都只是用处不太大的尝试。本文所要求的目标或者是要到达的目的是可以使用git版本管理，自动构建部署。也就是我们只需要去提交一个markdown文件，然后剩下的事都放在流水线上做。同时只要每隔一段时间去申请一个免费证书，保证腾讯云账户里有少量的钱就行了。

## 代码仓库

代码仓库选择的是腾讯的[coding](https://www.coding.net/)，选择的原因有很多，我主要列出来几条：

1. coding拥有很大的仓库空间，不用担心仓库文件太多比较大的问题，可以肆意折腾。
2. coding的项目管理做得比较完善，操作比较简单。
3. coding自己的CI兼容了jenkinsfile的语法，并且可以用可视化编辑，每个月有1000分钟的免费构建时间。

当然也可以尝试github+action或者gitee的CI，甚至是gitlab、jenkins等等。这里我只对coding的方式介绍。

### 注册和创建项目

**注册比较简单，一步一步来即可，不多介绍。**

创建项目可以选择任意风格的项目，最后只要保证持续集成的功能打开即可。如果没有打开，可以按照下文所示操作：

项目详情-左下角项目设置-功能开关-持续集成

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210209114641.png" style="zoom:80%;" />

创建一个代码仓库然后上传项目

![](https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210209114916.png)

其实coding持续部署里面是有一个静态网站的部署的，但是它不支持hugo的在线编译，所以放弃了。也可以去了解一下。

因为hugo是需要一个可执行文件的[hugo release](https://github.com/gohugoio/hugo/releases)，coding的免费构建云主机是ubuntu因此下载linux版本的即可，这里我为了减少构建时间和方便管理直接把文件放在代码仓库里推上去了。

### 配置构建计划

选择持续集成-构建计划-新建构建计划

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210209124116.png" style="zoom:80%;" />

选择最下面的自定义构建过程

![](https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210209124209.png)

设置好名称，代码仓库和配置的来源

<img src="https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210209124256.png" style="zoom:80%;" />

然后通过编辑界面编辑jenkinsfile

![](https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210209124419.png)

以下附上jinkinsfile文本

```
pipeline {
  agent any
  stages {
    stage('检出') {
      steps {
        checkout([$class: 'GitSCM',
        branches: [[name: GIT_BUILD_REF]],
        userRemoteConfigs: [[
          url: GIT_REPO_URL,
          credentialsId: CREDENTIALS_ID
        ]]])
      }
    }
    stage('初始化 env') {
      steps {
        sh 'env'
        sh 'date'
        sh 'echo TENCENT_SECRET_ID=$TENCENT_SECRET_ID >> .env'
        sh 'echo TENCENT_SECRET_KEY=$TENCENT_SECRET_KEY >> .env'
      }
    }
    stage('安装 Severless 环境') {
      steps {
        sh 'pnpm install -g serverless'
        sh 'sls -v'
      }
    }
    stage('构建') {
      steps {
        sh '''chmod +x ./cli/hugo
./cli/hugo -D'''
      }
    }
    stage('部署应用') {
      steps {
        sh 'serverless deploy --debug'
      }
    }
  }
}
```

其中需要解释的地方如下：

1. 检出代码，是coding的默认配置。
2. 初始化env，使用了下图所示的变量，sercret和key获取[api密钥获取](https://console.cloud.tencent.com/cam/capi)。

![](https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210209124830.png)

3. 构建使用了代码仓库cli目录下的hugo可执行文件。
4. serverless deploy --debug是为了部署serverless framework，下文会详细讲。

## serverless framework

首先附上[组件仓库](https://github.com/serverless-components/tencent-website)和[文档](https://cloud.tencent.com/document/product/1154/38787)

**我的理解**

腾讯云serverless framework其实就是通过yml配置方式去调用腾讯云的各种资源，比如对象储存，内容分发，文件储存，云函数等等。这样做的好处是通过一定的配置，自由的调度云资源而不用去关心底层构架的结构，而且部署上也简化了很多，一个配置文件生成后很少需要去修改。在部署的时候只需要执行命令行就可以了，这个应该是未来的一种趋势，一种时代。

就比如本文所要达成的静态页面其实就是使用了腾讯云的COS储存桶去放这些编译好的静态文件，然后通过cdn进行分发回源。结构大概如下图所示：

![](https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic20210209130356.png)

相比于使用服务器部署，性能和简便程度都提高了不少，成本也会大大缩减。(当然单独一个服务器的话能做更多，可定制化也越强。)

总结一下就是一个yml的文件，具体可以参考上面组件仓库，我只附上经过尝试最合适的配置：

```yaml
component: website
name: hugoblog

inputs:
  src:
    src: ./public # 这是你构建完的目录
    index: index.html
    error: 404.html
  region: ap-shanghai # 区域
  bucketName: {name}-{userid}
  protocol: http
  replace: true
  disableErrorStatus: false
  autoSetupAcl: true
  autoSetupPolicy: false
  hosts:
    - host: sasaba.net
      async: true # 是否同步等待 CDN 配置。CI推荐为true
      area: mainland # 加速地区，因为我域名备案了所以选的是大陆
      autoRefresh: false # 调用刷新的api 这个必须上面async为false
      onlyRefresh: false 
      https:
        switch: on
        http2: on
        certInfo:
          certId: '{cert-id}' # 域名证书的id，可在控制台查看
      cache:
        simpleCache:
          followOrigin: on
          cacheRules:
            - cacheType: all
              cacheContents:
                - '*'
              cacheTime: 1000
      cacheKey:
        fullUrlCache: on
      forceRedirect:
        switch: on
        redirectType: https
        redirectStatusCode: 301
    - host: www.sasaba.net
      async: true
      area: mainland
      autoRefresh: false
      onlyRefresh: false
      https:
        switch: on
        http2: on
        certInfo:
          certId: '{cert-id}'
      cache:
        simpleCache:
          followOrigin: on
          cacheRules:
            - cacheType: all
              cacheContents:
                - '*'
              cacheTime: 1000
      cacheKey:
        fullUrlCache: on
      forceRedirect:
        switch: on
        redirectType: https
        redirectStatusCode: 301
```

只要修改其中一些配置搭配上文的构建过程就可以使用。