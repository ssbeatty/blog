---
title: 交叉编译libjpeg-turbo
date: 2023-08-31T9:25:05+08:00
lastmod: 2023-08-31T9:25:05+08:00
author: sasaba
cover: /img/交叉编译libjpeg-turbo.jpeg
images:
- /img/交叉编译libjpeg-turbo.jpeg
categories:
  - 工具 
tags:
  - c/c++
draft: true
---

交叉编译libjpeg-turbo到armv7l。

<!--more-->

## 起因
使用golang的标准库的jpeg库解码和编码图片都比较慢，所以想使用libjpeg-turbo来加速，但是libjpeg-turbo没有提供armv7l的二进制包，所以需要自己交叉编译。

可以看这个[issue](https://github.com/golang/go/issues/24499)，不懂为什么2020年的issue还没有解决。。

## 交叉编译
使用docker编译
```text
FROM ubuntu:20.04

RUN apt-get update \
    && apt-get install -y gcc-multilib-arm-linux-gnueabihf binutils-arm-linux-gnueabihf g++-multilib-arm-linux-gnueabihf
RUN DEBIAN_FRONTEND=noninteractive TZ=Etc/UTC apt-get install -y git cmake

RUN mkdir /workspace && cd /workspace \
    && git clone https://github.com/libjpeg-turbo/libjpeg-turbo.git \
    && mkdir libjpeg-turbo/build
COPY linux-arm-toolchain.cmake /workspace/libjpeg-turbo/build

RUN cd /workspace/libjpeg-turbo/build \
    && cmake -G"Unix Makefiles" -DCMAKE_TOOLCHAIN_FILE=linux-arm-toolchain.cmake -DREQUIRE_SIMD=1 .. ${1+"$@"} \
    && make -j 2 \
    && make install


WORKDIR /opt
```

linux-arm-toolchain.cmake
```text
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR arm)
set(CMAKE_C_FLAGS "-mfloat-abi=hard -march=armv7-a -mfpu=neon -mthumb")
set(CMAKE_C_COMPILER /usr/bin/arm-linux-gnueabihf-gcc)
```

编译完成后把libjpeg.a和include文件夹copy出来即可。

## CGO使用
[go-libjpeg](https://github.com/ssbeatty/go-libjpeg)

一个简单的例子，跟std的接口基本差不多：
```go
package main

import (
	"bytes"
	"github.com/disintegration/imaging"
	log "github.com/sirupsen/logrus"
	"github.com/ssbeatty/go-libjpeg/jpeg"
	"os"
)

func main() {
	var buf bytes.Buffer
	defer buf.Reset()

	reader, err := os.OpenFile("snapshot.jpg", os.O_RDWR, 0644)
	if err != nil {
		log.Error(err)
		return
	}

	defer reader.Close()

	img, err := jpeg.Decode(reader, &jpeg.DecoderOptions{})
	if err != nil {
		log.Error(err)
		return
	}

	dist := imaging.Resize(img, 800, 0, imaging.Linear)

	err = jpeg.Encode(&buf, dist, &jpeg.EncoderOptions{Quality: 100})
	if err != nil {
		log.Error(err)
		return
	}

}
```

如何编译：
```shell
CGO_CFLAGS="-I/opt/libjpeg-turbo/include" CGO_LDFLAGS="-L/opt/libjpeg-turbo/lib32" CGO_ENABLED=1 GOOS=linux GOARCH=arm CC=arm-linux-gnueabihf-gcc go build -ldflags="-s -w"
```

## 结论
速度大概快2到3倍。因为使用了simd，cpu消耗也相对较低，但是增加了编译的复杂度，希望未来go能优化std吧。