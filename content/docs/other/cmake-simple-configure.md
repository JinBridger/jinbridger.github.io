---
title: "CMake 简单项目配置文件"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# CMake 简单项目配置文件

配合 clangd 与 ninja 使用

<div align="center">
	<img src="/image/other/cmake-simple-configure/final-product.png" width="100%">
</div>

项目目录结构：

```
Project
 ├─CMakeLists.txt
 ├─include
 │  └─headers.h
 ├─res
 └─src
    └─sources.cpp
```

```cmake
cmake_minimum_required(VERSION 3.15)

# 项目名
project(NameOfProject)

# 添加头文件路径
include_directories(./include)

# 添加源文件路径
aux_source_directory(./src DIR_SRCS)
add_executable(main ${DIR_SRCS})

# 生成 compile_commands.json
set(CMAKE_EXPORT_COMPILE_COMMANDS on)
```