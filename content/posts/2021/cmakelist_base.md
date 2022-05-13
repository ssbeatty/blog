---
title: CMakeList基础
date: 2021-11-12T15:47:00+08:00
lastmod: 2022-05-13T15:47:00+08:00
author: sasaba
cover: /img/cmake基础.png
images:
  - /img/cmake基础.png 
categories:
  - 工具
tags:
  - cmake
  - c/c++
---

简单介绍一下CMakeList.txt的使用。

<!--more-->
### 一个简单的示例:
```c
cmake_minimum_required(VERSION 3.17)
project(hasp_test C)


include_directories(./include)
link_directories(./lib)
#add_library(hasp_test SHARED main.c)

add_executable(hasp_test main.c)
target_link_libraries(hasp_test libhasp_windows_x64.a)
```

### 详细解析
> cmake_minimum_required

```text
# 该命令指明了对cmake的最低(高)版本的要求，...为低版本和高版本之间的连接符号，没有其他含义。
cmake_minimum_required(VERSION <min>[...<max>] [FATAL_ERROR])
```

> set

```text
# 用来设置环境变量
set(<variable> <value>... [PARENT_SCOPE])
```

> include_directories

可以将项目使用的头文件导入然后在c文件中只需要直接`include`header文件即可。
```c
// 原来是
#include "lib/xxx.h"

// 现在可以
#include "xxx.h"
```

> link_directories

引入项目的静态库或者动态库的目录。

> add_library

将项目导出为库。

```text
# 导出为静态库 比如libxxx.a
add_library(hasp_test STATIC main.c)

# 导出为动态库 比如libxxx.so 或者 libxxx.dll
add_library(hasp_test SHARED main.c)
```

> add_executable


```text
# 增加编译为可执行函数, 左边是项目名, 右边是源文件
add_executable(hasp_test "main.c")
```

> target_link_libraries

引入项目所需要的的库。

```text
# 定义
target_link_libraries(<target> ... <item>... ...)
target表示项目的名称 item是库

# 引入了lib目录下的静态库libhasp_windows_x64.a
target_link_libraries(hasp_test libhasp_windows_x64.a)

# 引入ws2_32.lib
target_link_libraries(hasp_test ws2_32)
```

### 流程控制

> 简单的if语句

```text
if(MINGW)
    target_link_libraries(hasp_test ws2_32)
endif()
```

### QA
> 怎么使用gui查看dll依赖

[Dependencies](https://github.com/lucasg/Dependencies)

> 对于某个lib mingw使用libstdc++.dll的问题

```text
target_link_libraries(${LIB_NAME} -static stdc++ -dynamic)
```