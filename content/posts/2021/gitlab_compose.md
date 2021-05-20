---
title: docker搭建gitlab
date: 2021-05-20T20:43:05+08:00
lastmod: 2021-05-20T20:43:05+08:00
author: sasaba
cover: /img/docker搭建gitlab.jpg
images:
  - /img/docker搭建gitlab.jpg
categories:
  - 工具
tags:
  - 技术
  - git
  - docker
draft: true
---

使用Docker部署gitlab并开启邮件和容器仓库功能。

<!--more-->

## 安装gitlab

### 安装gitlab web程序
本次测试选用docker安装gitlab，选用rc版本的镜像。

#### 最简单部署
> 简单的yaml部署文件
```yaml
version: '3'
services:
  web:
    image: "gitlab/gitlab-ce:rc"
    restart: always
    ports:
      - '80:80'
      - '443:443'
      - '10022:22'
    volumes:
      - './data/gitlab/config:/etc/gitlab'
      - './data/gitlab/logs:/var/log/gitlab'
      - './data/gitlab/data:/var/opt/gitlab'
```

这样可以生成一个简单的实例，具备大部分功能。

#### 部署SSL
1. 生成证书
简单点，使用腾讯云或者阿里云的SSL。支持一键生成证书。
2. 配置SSL
```text
        nginx['redirect_http_to_https'] = true
        nginx['ssl_certificate'] = '/etc/gitlab/ssl/gitlab.sasaba.cn.crt'
        nginx['ssl_certificate_key'] = '/etc/gitlab/ssl/gitlab.sasaba.cn.key'
```

#### 部署邮件功能
下面的例子是腾讯云企业邮箱的，其他的例子可以看其邮箱SMTP的配置。
```text
        gitlab_rails['smtp_enable'] = true
        gitlab_rails['smtp_address'] = "smtp.exmail.qq.com"
        gitlab_rails['smtp_port'] = 465
        gitlab_rails['smtp_user_name'] = "wgx@sasaba.net"
        gitlab_rails['smtp_password'] = ""
        gitlab_rails['smtp_authentication'] = "login"
        gitlab_rails['smtp_enable_starttls_auto'] = true
        gitlab_rails['smtp_tls'] = true
        gitlab_rails['gitlab_email_from'] = 'wgx@sasaba.net'
        gitlab_rails['smtp_domain'] = "exmail.qq.com"
```

#### 部署容器仓库
上文中部署SSL的功能就是为了这里可以简单点，如果仓库不是SSL需要docker daemon配置信任，比较麻烦。

```text
        registry_external_url "https://gitlab.sasaba.cn:4567"
        registry_nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.sasaba.cn.crt"
        registry_nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.sasaba.cn.key"
        gitlab_rails['registry_host'] = "gitlab.sasaba.cn"
        gitlab_rails['registry_port'] = "4567"
        gitlab_rails['registry_api_url'] = "http://localhost:5000"
```

#### 完整的web yaml文件

```yaml
version: '3'
services:
  web:
    image: "gitlab/gitlab-ce:rc"
    restart: always
    hostname: 'gitlab.sasaba.cn'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://gitlab.sasaba.cn'
        gitlab_rails['gitlab_shell_ssh_port'] = 10022
        gitlab_rails['time_zone'] = 'Asia/Shanghai'
        nginx['redirect_http_to_https'] = true
        nginx['ssl_certificate'] = '/etc/gitlab/ssl/gitlab.sasaba.cn.crt'
        nginx['ssl_certificate_key'] = '/etc/gitlab/ssl/gitlab.sasaba.cn.key'
        gitlab_rails['lfs_enabled'] = true

        gitlab_rails['smtp_enable'] = true
        gitlab_rails['smtp_address'] = "smtp.exmail.qq.com"
        gitlab_rails['smtp_port'] = 465
        gitlab_rails['smtp_user_name'] = "wgx@sasaba.net"
        gitlab_rails['smtp_password'] = ""
        gitlab_rails['smtp_authentication'] = "login"
        gitlab_rails['smtp_enable_starttls_auto'] = true
        gitlab_rails['smtp_tls'] = true
        gitlab_rails['gitlab_email_from'] = 'wgx@sasaba.net'
        gitlab_rails['smtp_domain'] = "exmail.qq.com"

        registry_external_url "https://gitlab.sasaba.cn:4567"
        registry_nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.sasaba.cn.crt"
        registry_nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.sasaba.cn.key"
        gitlab_rails['registry_host'] = "gitlab.sasaba.cn"
        gitlab_rails['registry_port'] = "4567"
        gitlab_rails['registry_api_url'] = "http://localhost:5000"
    ports:
      - '80:80'
      - '443:443'
      - '10022:22'
      - '4567:4567'
    volumes:
      - './data/gitlab/config:/etc/gitlab'
      - './certs:/etc/gitlab/ssl'
      - './data/gitlab/logs:/var/log/gitlab'
      - './data/gitlab/data:/var/opt/gitlab'
    networks:
      - gitlab

networks:
  gitlab:
    name: gitlab
```

### 安装gitlab runner
安装runner比较简单，直接附上yaml文件

```yaml
version: '3'
services:
  runner:
    image: "gitlab/gitlab-runner:latest"
    restart: always
    privileged: true
    volumes:
      - './data/gitlab-runner/config:/etc/gitlab-runner'
      - '/var/run/docker.sock:/var/run/docker.sock'
    networks:
      - gitlab

networks:
  gitlab:
    name: gitlab
```

### 注册runner
1. 在gitlab界面上边栏的设置中找到runner的配置，找到注册的key。如下图所示:
![](https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com//uPic20210520210737.png)
2. 执行shell
```shell script
docker run --rm -it --network gitlab -v /gitlab/data/gitlab-runner/config:/etc/gitlab-runner gitlab/gitlab-runner:latest register \
 --non-interactive \
 --executor "docker" \
 --docker-image alpine:latest \
 --url "https://gitlab.sasaba.cn" \
 --registration-token "" \
 --description "my-first-runner" \
 --tag-list "dockercicd,test,dev,prod" \
 --run-untagged="true" \
 --locked="false" \
 --access-level="not_protected" \
 --docker-extra-hosts "gitlab.sasaba.cn:172.17.0.1" \
 --docker-pull-policy "if-not-present" \
 --docker-volumes "/gitlab/data/gitlab-runner/cache:/cache" \
 --docker-volumes "/var/run/docker.sock:/var/run/docker.sock" \
 --docker-privileged
```
其中路径跟上面runner的文件一样，修改registration-token和docker-extra-hosts和url即可。

### 完整的yaml

```yaml
version: '3'
services:
  web:
    image: "gitlab/gitlab-ce:rc"
    restart: always
    hostname: 'gitlab.sasaba.cn'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://gitlab.sasaba.cn'
        gitlab_rails['gitlab_shell_ssh_port'] = 10022
        gitlab_rails['time_zone'] = 'Asia/Shanghai'
        nginx['redirect_http_to_https'] = true
        nginx['ssl_certificate'] = '/etc/gitlab/ssl/gitlab.sasaba.cn.crt'
        nginx['ssl_certificate_key'] = '/etc/gitlab/ssl/gitlab.sasaba.cn.key'
        gitlab_rails['lfs_enabled'] = true

        gitlab_rails['smtp_enable'] = true
        gitlab_rails['smtp_address'] = "smtp.exmail.qq.com"
        gitlab_rails['smtp_port'] = 465
        gitlab_rails['smtp_user_name'] = "wgx@sasaba.net"
        gitlab_rails['smtp_password'] = ""
        gitlab_rails['smtp_authentication'] = "login"
        gitlab_rails['smtp_enable_starttls_auto'] = true
        gitlab_rails['smtp_tls'] = true
        gitlab_rails['gitlab_email_from'] = 'wgx@sasaba.net'
        gitlab_rails['smtp_domain'] = "exmail.qq.com"

        registry_external_url "https://gitlab.sasaba.cn:4567"
        registry_nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.sasaba.cn.crt"
        registry_nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.sasaba.cn.key"
        gitlab_rails['registry_host'] = "gitlab.sasaba.cn"
        gitlab_rails['registry_port'] = "4567"
        gitlab_rails['registry_api_url'] = "http://localhost:5000"
    ports:
      - '80:80'
      - '443:443'
      - '10022:22'
      - '4567:4567'
    volumes:
      - './data/gitlab/config:/etc/gitlab'
      - './certs:/etc/gitlab/ssl'
      - './data/gitlab/logs:/var/log/gitlab'
      - './data/gitlab/data:/var/opt/gitlab'
    networks:
      - gitlab

  runner:
    image: "gitlab/gitlab-runner:latest"
    restart: always
    privileged: true
    volumes:
      - './data/gitlab-runner/config:/etc/gitlab-runner'
      - '/var/run/docker.sock:/var/run/docker.sock'
    networks:
      - gitlab

networks:
  gitlab:
    name: gitlab
```