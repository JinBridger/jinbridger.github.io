---
title: "解决阿里云 OSS 下载速度问题"
weight: 1
date: 2024-08-25
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 解决阿里云 OSS 下载速度问题

## 遇到的问题

最近把业务尝试迁移到阿里云的函数计算上面，发现用 OSS Python SDK 走内网下载 OSS 上东西的速度不太对。一般在 ECS 上能跑到 100M+ 但是在函数计算上面却只能跑到 4M+

<div align="center">
	<img src="/image/etc/aliyun-oss-crcmod-debug/fc-download-slow.png" width="70%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    函数计算走内网下载 OSS 文件速度，只有 4M+
    </div>
</div>

<div align="center">
	<img src="/image/etc/aliyun-oss-crcmod-debug/ecs-download-fast.png" width="100%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    ECS 走内网下载 OSS 文件速度，可以达到 100M+
    </div>
</div>

## 排查过程

首先尝试排查函数计算实例的网络问题。发起请求后点击实例 ID 尝试登入实例中进行排查，在简单安装了一下 wget 以后尝试下载 OSS 的文件，发现速度似乎是对的

<div align="center">
	<img src="/image/etc/aliyun-oss-crcmod-debug/fc-wget.png" width="100%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    函数计算使用 wget 下载 OSS 文件速度，可以达到 30M+
    </div>
</div>

排除掉函数计算本身的网络的问题以后，大概率是代码的问题。但原来的业务代码是可以跑到 100M+ 的。思考了一下，**唯一的区别就是函数计算的业务是跑在容器里面的。**

翻了一下阿里云的 Python OSS SDK 安装指南，发现了这样一句话:

> [!NOTE]
> **安装 `python-devel`**
> 
> 完成环境准备后，您需要先要安装 python-devel 包。
> 
> **说明**
> 
> OSS Python SDK 需要 crcmod 计算 CRC 校验码，而 crcmod 依赖 python-devel 包中的 Python.h 文件。如果系统缺少 Python.h 文件，虽然之后安装 OSS Python SDK 不会失败，但 crcmod 的 C 扩展模式安装会失败。如果 crcmod 的 C 扩展模式安装失败，在上传、下载计算 CRC 校验码时会使用纯 Python 模式进行 CRC 数据校验。纯 Python 模式的性能远差于 C 扩展模式，从而导致上传、下载等操作效率非常低下。

感觉大概率是这个 crcmod 的问题。注意到 Python 官方提供的容器里面并没有 gcc 这种编译器，自然也无法编译生成 C 扩展。

## 解决方法

思考了一下怎么改动 Dockerfile, 一个可能的解决方案是用 multi-stage 构建一个带有 C 扩展的 wheel 包出来，然后替换掉安装的 crcmod.

为了保证构建出来的 wheel 包能用，使用了一致的镜像 `python:3.10-slim`


```Dockerfile
# 构建 crcmod 的 C extension wheel
# 如果不构建会导致 oss2 下载速度比较慢
# 参考: https://help.aliyun.com/zh/oss/developer-reference/installation-14
FROM python:3.10-slim

RUN apt update -y \
 && apt install -y build-essential \
 && mkdir build_wheel \
 && pip wheel -w ./build_wheel/ crcmod

# 构建最终镜像
FROM python:3.10-slim

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

COPY . /root/CODE/
COPY --from=0 ./build_wheel /root/build_wheel/

ADD https://install.python-poetry.org /root/poetry_install.py

WORKDIR /root/CODE

RUN python3 /root/poetry_install.py
RUN export PATH="/root/.local/bin:$PATH" \
 && poetry install \
 && poetry cache clear --all . \
 && poetry run pip install --upgrade --force-reinstall --find-links /root/build_wheel/ crcmod

 
CMD [ "/root/.local/bin/poetry", "run", "python", "YOUR_CODE" ]
```

实验了一下，速度果然正常了

<div align="center">
	<img src="/image/etc/aliyun-oss-crcmod-debug/fc-download-fast.png" width="100%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    使用带有 C 扩展的 crcmod 以后，速度正常了
    </div>
</div>

## 参考资料

- OSS Python SDK 安装指南 https://help.aliyun.com/zh/oss/developer-reference/installation-14
- Multi-stage builds https://docs.docker.com/build/building/multi-stage/