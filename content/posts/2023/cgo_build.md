---
title: 记录cgo交叉编译ffmpeg
date: 2023-08-16T15:25:05+08:00
lastmod: 2023-08-16T15:25:05+08:00
author: sasaba
cover: /img/记录cgo编译ffmpeg.jpeg
images:
- /img/记录cgo编译ffmpeg.jpeg
categories:
  - 工具 
tags:
  - go
  - c/c++
draft: true
---

记录cgo编译ffmpeg。

<!--more-->

## 起因

工作需要使用[gortsplib](https://github.com/bluenviron/gortsplib)，有个需求是使用go读取rtsp流并截图，摄像头是海康威视的低端摄像头。
因为需要解码，所以根据gortsplib的例子看到了如下[代码](https://github.com/bluenviron/gortsplib/blob/main/examples/client-read-format-h264-convert-to-jpeg/h264decoder.go)：
```go
package main

import (
	"fmt"
	"image"
	"unsafe"
)

// #cgo pkg-config: libavcodec libavutil libswscale
// #include <libavcodec/avcodec.h>
// #include <libavutil/imgutils.h>
// #include <libswscale/swscale.h>
import "C"

func frameData(frame *C.AVFrame) **C.uint8_t {
	return (**C.uint8_t)(unsafe.Pointer(&frame.data[0]))
}

func frameLineSize(frame *C.AVFrame) *C.int {
	return (*C.int)(unsafe.Pointer(&frame.linesize[0]))
}

// h264Decoder is a wrapper around ffmpeg's H264 decoder.
type h264Decoder struct {
	codecCtx    *C.AVCodecContext
	srcFrame    *C.AVFrame
	swsCtx      *C.struct_SwsContext
	dstFrame    *C.AVFrame
	dstFramePtr []uint8
}

// newH264Decoder allocates a new h264Decoder.
func newH264Decoder() (*h264Decoder, error) {
	codec := C.avcodec_find_decoder(C.AV_CODEC_ID_H264)
	if codec == nil {
		return nil, fmt.Errorf("avcodec_find_decoder() failed")
	}

	codecCtx := C.avcodec_alloc_context3(codec)
	if codecCtx == nil {
		return nil, fmt.Errorf("avcodec_alloc_context3() failed")
	}

	res := C.avcodec_open2(codecCtx, codec, nil)
	if res < 0 {
		C.avcodec_close(codecCtx)
		return nil, fmt.Errorf("avcodec_open2() failed")
	}

	srcFrame := C.av_frame_alloc()
	if srcFrame == nil {
		C.avcodec_close(codecCtx)
		return nil, fmt.Errorf("av_frame_alloc() failed")
	}

	return &h264Decoder{
		codecCtx: codecCtx,
		srcFrame: srcFrame,
	}, nil
}

// close closes the decoder.
func (d *h264Decoder) close() {
	if d.dstFrame != nil {
		C.av_frame_free(&d.dstFrame)
	}

	if d.swsCtx != nil {
		C.sws_freeContext(d.swsCtx)
	}

	C.av_frame_free(&d.srcFrame)
	C.avcodec_close(d.codecCtx)
}

func (d *h264Decoder) decode(nalu []byte) (image.Image, error) {
	nalu = append([]uint8{0x00, 0x00, 0x00, 0x01}, []uint8(nalu)...)

	// send frame to decoder
	var avPacket C.AVPacket
	avPacket.data = (*C.uint8_t)(C.CBytes(nalu))
	defer C.free(unsafe.Pointer(avPacket.data))
	avPacket.size = C.int(len(nalu))
	res := C.avcodec_send_packet(d.codecCtx, &avPacket)
	if res < 0 {
		return nil, nil
	}

	// receive frame if available
	res = C.avcodec_receive_frame(d.codecCtx, d.srcFrame)
	if res < 0 {
		return nil, nil
	}

	// if frame size has changed, allocate needed objects
	if d.dstFrame == nil || d.dstFrame.width != d.srcFrame.width || d.dstFrame.height != d.srcFrame.height {
		if d.dstFrame != nil {
			C.av_frame_free(&d.dstFrame)
		}

		if d.swsCtx != nil {
			C.sws_freeContext(d.swsCtx)
		}

		d.dstFrame = C.av_frame_alloc()
		d.dstFrame.format = C.AV_PIX_FMT_RGBA
		d.dstFrame.width = d.srcFrame.width
		d.dstFrame.height = d.srcFrame.height
		d.dstFrame.color_range = C.AVCOL_RANGE_JPEG
		res = C.av_frame_get_buffer(d.dstFrame, 1)
		if res < 0 {
			return nil, fmt.Errorf("av_frame_get_buffer() err")
		}

		d.swsCtx = C.sws_getContext(d.srcFrame.width, d.srcFrame.height, C.AV_PIX_FMT_YUV420P,
			d.dstFrame.width, d.dstFrame.height, (int32)(d.dstFrame.format), C.SWS_BILINEAR, nil, nil, nil)
		if d.swsCtx == nil {
			return nil, fmt.Errorf("sws_getContext() err")
		}

		dstFrameSize := C.av_image_get_buffer_size((int32)(d.dstFrame.format), d.dstFrame.width, d.dstFrame.height, 1)
		d.dstFramePtr = (*[1 << 30]uint8)(unsafe.Pointer(d.dstFrame.data[0]))[:dstFrameSize:dstFrameSize]
	}

	// convert frame from YUV420 to RGB
	res = C.sws_scale(d.swsCtx, frameData(d.srcFrame), frameLineSize(d.srcFrame),
		0, d.srcFrame.height, frameData(d.dstFrame), frameLineSize(d.dstFrame))
	if res < 0 {
		return nil, fmt.Errorf("sws_scale() err")
	}

	// embed frame into an image.Image
	return &image.RGBA{
		Pix:    d.dstFramePtr,
		Stride: 4 * (int)(d.dstFrame.width),
		Rect: image.Rectangle{
			Max: image.Point{(int)(d.dstFrame.width), (int)(d.dstFrame.height)},
		},
	}, nil
}
```

上面的代码是解析H264编码的例子。其实别人早就写好了，那我要做的就是编译和使用。

```shell
apt update -y
apt install -y libavcodec-dev libavutil-dev libswscale-dev

go build
```

上面的步骤是linux amd64编译的过程，相对也比较简单。但是因为需要编译armv7l的版本，才有了后文。

## 交叉编译ffmpeg

```shell
git clone https://github.com/FFmpeg/FFmpeg.git

# 因为我的目标是编译armv7l 所以安装这个
sudo apt-get install gcc-arm-linux-gnueabihf
sudo apt-get install g++-arm-linux-gnueabihf

./configure --prefix=/you/path/ffmpeg --target-os=linux --arch=armv7a --enable-static --cross-prefix=arm-linux-gnueabihf- --enable-pic --disable-asm
make && make install
```

> ps:可能存在缺少什么包的情况，缺什么装什么

## cgo交叉编译

偷懒直接用goreleaser

```yaml
before:
  hooks:
    - go mod tidy
builds:
  - binary: main
    id: main-amd64
    main: main.go
    env:
      - CGO_ENABLED=1
    goos:
      - linux
    ldflags:
      - -s -w
    goarch:
      - amd64

  - binary: main-arm
    id: main-arm
    main: main.go
    env:
      - CGO_ENABLED=1
      - CC=arm-linux-gnueabihf-gcc
      - CXX=arm-linux-gnueabihf-g++
      - PKG_CONFIG_PATH=/path/you/ffmpeg/lib/pkgconfig  # 这个是上一步编译的产物
    ldflags:
      - -s -w
    goos:
      - linux
    goarch:
      - arm
    goarm:
      - "7"

archives:
  -
    name_template: "{{.Os}}-{{.Arch}}{{if .Arm}}v{{.Arm}}{{end}}-{{ .ProjectName }}"
    format: tar.gz
    format_overrides:
      - goos: windows
        format: zip
```

要注意的是要配置PKG_CONFIG_PATH 因为`#cgo pkg-config: libavcodec libavutil libswscale`其实也会寻找对应的头文件，这里以前确实是不懂。

