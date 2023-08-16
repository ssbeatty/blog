---
title: 记录golang接口赋值nil的坑
date: 2023-08-16T17:25:05+08:00
lastmod: 2023-08-16T17:25:05+08:00
author: sasaba
cover: /img/记录golang接口赋值nil的坑.png
images:
- /img/记录golang接口赋值nil的坑.png
categories:
  - Golang 
tags:
  - Golang
draft: true
---

记录golang接口赋值nil的坑。

<!--more-->

## 代码

```go
package main

import "fmt"

type Interface interface {
	MethodCall()
}

type Struct struct {
}

func (s *Struct) MethodCall() {

}

func NewStruct() *Struct {
	return nil
}

func main() {
	var f1, f2 Interface

	// 不判断nil
	s1 := NewStruct()
	f1 = s1

	// 判断nil
	s2 := NewStruct()
	if s2 != nil {
		f2 = s2
	}

	if f1 == nil {
		fmt.Println("f1 is nil")
	} else {
		fmt.Println("f1 is not nil")
	}

	if f2 == nil {
		fmt.Println("f2 is nil")
	} else {
		fmt.Println("f2 is not nil")
	}
}

// f1 is not nil
// f2 is nil
```

当把一个指针类型赋值给接口时，如果直接复制就会出现判断==nil时不为空的情况，这是因为f1虽然是nil但是它是有类型的，最好的方法就是遇到这种情况判断下再赋值如f2。


```go
func NewStruct() Interface {
	return nil
}
```

但是把NewStruct的返回值改成接口类型就不会有这种情况。这个写法上就仁者见仁智者见智了，但是这个坑还是要避免的。
