---
title: "什么是 Kubernetes"
weight: 1
date: 2025-10-14
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 什么是 Kubernetes

Kubernetes 是一个可移植, 可扩展的开源平台, 用于管理容器化的工作负载和服务, 方便进行声明式配置和自动化.
可以简单的理解为是用于管理容器的自动化工具.

Kubernetes 常常被简称为 k8s.

<div align="center">
<img src="/image/dev/k8s-intro/kubernetes-cluster-architecture.svg" width="100%">
    <br>
    <div style="
    display: inline-block;
    color: #999;
    padding: 2px;">k8s 的架构</div>
</div>

k8s 由以下部分组成:
- 一个控制平面, 用来为集群做出全局决策, 比如资源的调度, 检测和响应集群事件
- 一个或多个 Node. 用于运行实际的计算任务.

## 运行方式

k8s 项目维护的是一个完整的、可部署在任何环境的「容器调度与编排系统」的实现.
包含 API Server、Controller、Scheduler、Kubelet、Proxy 等核心组件.
提供标准接口 (CRI/CNI/CSI) 以支持多厂商生态.

但如何运行这些核心组件由具体的运行方式来决定.
常见的运行方式有 GKE, EKS, AKS, minikube, k3s, kind 等.

## 命令行工具: kubectl

k8s 的命令行工具是 kubectl.
它通过发送 HTTP 请求到 k8s 的 kube-apiserver 来实现控制.
kubectl 与 k8s 集群的关系类似于 DBeaver/Navicat 与数据库的关系.

kubectl 常用的命令包括以下几个:
- `kubectl get ...`: 用于获取某类资源的信息.
  - 例如, 获取所有 Pod 的信息就是 `kubectl get pods`
- `kubectl describe ...`: 用于获取某个资源的信息.
  - 例如, 获取某个 Pod 的信息就是 `kubectl describe pod <name>`
- `kubectl logs <name>`: 用于查看某个 Pod 的日志.
- `kubectl exec ...`: 用于调试 Pod. 类似 Docker 的 exec.
- `kubectl create -f <yaml>`: 用于创建一类资源, 例如创建 Job, Deployment 等.
- `kubectl apply -f <yaml>`: 用于创建/更新一类资源, 是幂等操作.
- `kubectl delete ...`: 用于删除某类资源.
- `kubectl port-forward ...`: 用于转发 Pod/Service 的端口, 用于调试.
  - 比如可以用于在本地浏览器访问集群内应用, 调试 REST 接口
- `kubectl top ...`: 用于查看某类资源的系统占用率

## 类比到数据库

| 概念                | 类比到数据库                          |
| ----------------- | ------------------------------------ |
| `kubectl`         | 客户端 e.g. Navicat                 |
| `kube-apiserver`  | 服务端核心 e.g. `postgres`             |
| minikube        | 本地一键安装版 PostgreSQL              |
| k3s             | 精简嵌入式版 PostgreSQL                |
| kind            | 容器里跑的测试版 PostgreSQL             |
| GKE / EKS / AKS | 云托管的 PostgreSQL                   |

## 关键概念

### Pod

Pod 直译过来是荚果. 是 Kubernetes 中创建和管理的、最小的可部署的计算单元.

- Pod 是一组容器, 这些容器共享存储、网络、以及怎样运行这些容器的规约.
- Pod 中的内容总是一同调度, 在共享的上下文中运行.
- Pod 所建模的是特定于应用的「逻辑主机」, 其中包含一个或多个应用容器.

### ReplicaSet 与 Deployment

- ReplicaSet 的作用是维持在任何给定时间运行的一组稳定的副本 Pod. 
- Deployment 是 ReplicaSet 的上层封装, 用于管理运行一个应用负载的一组 Pod, 通常适用于无状态的负载.

通常会定义一个 Deployment, 并用这个 Deployment 自动管理 ReplicaSet.

### Service

提供 Pod 的稳定访问入口, 用于将在集群中运行的应用通过同一个面向外界的端点公开出去.

### Volume

Volume 为 Pod 中的容器提供了一种通过文件系统访问和共享数据的方式.
Volume 的主要作用有两个: 数据持久性与共享存储.
- 数据持久性: 当容器崩溃或被停止时, 容器的状态不会被保存, 因此在容器生命期内创建或修改的所有文件都将丢失. 
- 共享存储: 当多个容器在一个 Pod 中运行并需要共享文件时, 在所有容器之间设置和访问共享文件系统可能会很有难度。
通过使用 Volume 就可以解决上面的两个问题.

### Namespace

类似 C++ 的 Namespace, k8s 的 Namespace 提供一种机制, 将同一集群中的资源划分为相互隔离的组.
同一名字空间内的资源名称要唯一, 但跨名字空间时没有这个要求.

### Job

Job 表示一次性任务, 运行完成后就会停止.

- Job 会创建一个或者多个 Pod, 并将继续重试 Pod 的执行, 直到指定数量的 Pod 成功终止.
- 随着 Pod 成功结束, Job 跟踪记录成功完成的 Pod 个数.
  - 当数量达到指定的成功个数阈值时, Job 即结束.
- 删除 Job 的操作会清除所创建的全部 Pod.
- 挂起 Job 的操作会删除 Job 的所有活跃 Pod, 直到 Job 被再次恢复执行.

