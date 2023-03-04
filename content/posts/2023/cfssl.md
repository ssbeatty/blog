---
title: 使用cfssl生成tls证书
date: 2023-03-04T22:51:05+08:00
lastmod: 2023-03-04T22:51:05+08:00
author: sasaba
cover: /img/使用cfssl生成tls证书.jpg
images:
- /img/使用cfssl生成tls证书.jpg
categories:
  - 工具
tags:
  - SSL
draft: true
---

使用cfssl生成tls证书。

<!--more-->

## cfssl安装

cfssl是CloudFlare开源的SSL/TLS瑞士军刀，使用go语言编写，可以在[官网](https://github.com/cloudflare/cfssl)看到源码。

使用go安装过程如下：
```shell
go install github.com/cloudflare/cfssl/cmd/...@latest
```

## 创建CA证书
```shell
cat > ca-csr.json <<EOF
{
	"CN": "FutureVX",
	"key": {
		"algo": "rsa",
		"size": 2048
	},
	"names": [
		{
			"C": "CN",
			"ST": "ShangHai",
			"L": "ShangHai",
			"O": "FutureVX",
			"OU": "CA"
		}
	]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "876000h"
    },
    "profiles": {
      "server": {
        "usages": [
          "signing",
          "key encipherment",
          "server auth"
        ],
        "expiry": "876000h"
      },
	  "client": {
        "usages": [
          "signing",
          "key encipherment",
          "client auth"
        ],
        "expiry": "876000h"
      },
	  "peer": {
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ],
        "expiry": "876000h"
      }
    }
  }
}
EOF
```

ca-csr.json文件中包含如下内容简要说明:

- CN: Common Name，表示业务的名称或者对外的域名。
- C: Country， 表示国家
- L: Locality，表示地区或城市
- O: Organization Name，表示组织名称或公司名称
- OU: Organizational Unit 表示组织单元名称
- ST: State，表示 州，省OU: Organization Unit Name，组织单位名称或者部门
- ca.expiry 表示证书的有效期，此处是20年
- key.algo 表示证书的签名算法，目前cfssl支持的签名算法只有rsa和ecdsa两种。
- hosts 表示要签名的域名，此处是根证书，所以空着，用于签名其他的证书。


ca-config.json文件中字段说明如下:

- signing, 表示ca.pem证书可用于签名其它证书
- profile中的peer配置的client auth 和 server auth
- profile中的client配置的client auth
- profile中的server配置的server auth
- server auth：表示 客户端client 可以用 CA证书 对 服务端server的证书进行签名验证。
- client auth：表示 服务端server 可以用 CA证书 对 客户端client 提供的证书进行签名验证。
- server auth和client auth都存在时，说明客户端和服务端双向验证。

## 编写业务域名的证书签名请求文件 demo.com.json

```json
{
  "CN": "demo.com",
  "hosts": [
    "*.demo.com"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShangHai",
      "L": "ShangHai",
      "O": "demo",
      "OU": "demo"
    }
  ]
}
```

这个业务域名签名请求文件的内容和ca-csr.json内容含义类似，关键部分是增加了hosts的配置，将需要签名认证的ip地址和域名增加到hosts列表中。

## 生成业务域名的证书和私钥
```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server demo.com.json | cfssljson -bare demo.com
```

此命令的参数说明:

- 使用ca证书：-ca=ca.pem
- 使用ca的密钥：-ca-key=ca-key.pem
- 使用ca签名证书的配置：-config=ca-config.json
- 选择ca签名证书配置的profile项：-profile=peer
- 选择业务域名的证书签名请求文件：demo.com.json
- 生成业务域名的私钥和证书文件：cfssljson -bare demo.com 会生成demo.com.pem证书文件和demo.com-key.pem的私钥文件。

## 将ca添加到windows信任的根域名

可参考[self-signed-certificate-as-trusted-root-ca-in-windows](https://cnzhx.net/blog/self-signed-certificate-as-trusted-root-ca-in-windows/)

### 写一个测试软件

```go
package main

import (
	"io"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, req *http.Request) {
		io.WriteString(w, "hello, world!\n")
	})
	if e := http.ListenAndServeTLS(":443", "demo.com.pem", "demo.com-key.pem", nil); e != nil {
		log.Fatal("ListenAndServe: ", e)
	}
}
```

### 配置hosts
```text
# hosts
...

127.0.0.1 demo.com
```

### 测试访问
可以看到浏览器显示安全的连接
![](/context_img/win_ssl_demo.png)



## 注意
此方法仅能用于内网安全环境测试！！！