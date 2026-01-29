---
title: "什么是 IPVS"
weight: 1
date: 2026-01-20
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 什么是 IPVS

## IPVS 是什么

IPVS (IP Virtual Server) 是 Linux 内核里的一个 L4 负载均衡子系统,
最早由章文嵩提出, 是 LVS (Linux Virtual Server) 项目的核心组件.
它主要用于高性能、可扩展的服务负载均衡场景.

> [!INFO]
> **nginx 也可以做 L4 负载均衡, 为什么需要 IPVS?**
> 
> nginx 运行在用户态, 而 IPVS 运行在内核.
> 这导致两者性能有数量级的差距.
> 同时 IPVS 与 nginx 的工作方式也不同.

Linux 中使用 `ipvsadm` 来进行管理 IPVS 服务.

相关的术语:
- VIP: Virtual IP, 服务对外暴露的 IP.
- Director: 负载均衡部署的机器.
- RS: Real Server, 真实部署服务的机器.


## IPVS 的工作模式

IPVS 支持三种工作模式: NAT, DR, TUN:
- NAT:
  - Director 做 DNAT + SNAT. 处理所有的流量.
  - 特点:
    - RS 不需要特殊网络配置.
    - 进出流量都经过 Director.
    - 性能一般.
  - 完整路径: `Client -> Director -> RS -> Director -> Client`
- DR:
  - Director 只处理入流量. IPVS 将入站数据包的 MAC 地址修改为 RS 的 MAC 地址然后发出.
  - 特点:
    - RS 需要绑定 VIP 并关闭 ARP 响应.
    - 性能最好.
  - 完整路径: `Client -> Director -> RS -> Client`
- TUN:
  - Director 用 IPIP 隧道把包发给 RS. 适用于 RS 与 Director 异地的场景. 性能接近 DR.
  - 完整路径: `Client -> Director --(IPIP 隧道)-> RS -> Client`

## IPVS 的调度算法

- **rr:** Round Robin. 轮询.
  - 请求按顺序一个个发给后端服务器.
  - 适用于服务器硬件配置基本一致且请求处理时间差不多的场景
- **wrr:** Weighted Round Robin. 加权轮询.
  - 请求按权重发给后端服务器.
  - 适用于服务器性能不同且请求处理时间差不多的场景
- **dh:** Destination Hash. 目标地址哈希.
  - 根据目标 IP 选服务器.
  - 同一个目标 IP 永远打到同一台 RS
- **sh:** Source Hash. 源地址哈希.
  - 根据客户端 IP 选服务器.
  - 同一个客户端 IP 总是访问同一台 RS
  - 适用于需要会话粘性的场景
- **lc:** Least Connection. 最少连接.
  - 选当前连接数最少的服务器
  - 适用于请求处理时间差异大, 有长连接 (如 API、数据库代理) 的场景
- **wlc:** Weighted Least Connections. 加权最少连接.
  - 性能强的服务器即使连接多，也可能继续分配到
  - 评价标准为: `当前连接数 / 权重`
- **sed:** Shortest Expected Delay. 最短期望延迟.
  - 选「预测响应最快」的服务器. 比 wlc 更激进地倾向于空闲服务器.
  - 评价标准为: `(当前连接数 + 1) / 权重`
- **nq:** Never Queue. 永不排队.
  - 在 sed 的基础上: 如果有空闲服务器 (0 连接), 立刻用它.
  - 适合低延迟系统 (RPC, 实时业务)

最常用的调度策略是 `wlc`

## 一个简单的示例

下面举一个简单的 DR 模式的负载均衡部署方案:

假设有一个 Director, 两个 RS. 对应的真实 IP 如下:
- Director: `192.168.1.213`
- RS1: `192.168.1.220`
- RS1: `192.168.1.193`

假设服务的 VIP 为 `192.168.1.10`.
调度算法使用 `rr`.

### 配置 Director

安装 ipvsadm:
```bash
sudo apt install -y ipvsadm
```

将 VIP 绑定到网卡上. 在这里网卡名称为 `ens18`.
```bash
sudo ip addr add 192.168.1.10/32 dev ens18
sudo ip link set ens18 up
```

配置 IPVS. 这里采用轮询调度算法 `rr`.
```bash
# 清空现有规则
sudo ipvsadm -C

# 建立一个虚拟服务器 192.168.1.10:80, 采用轮询算法
sudo ipvsadm -A -t 192.168.1.10:80 -s rr

# 把 RS1 与 RS2 加入 VIP 的后端, 权重均为 1, 使用 DR 模式
sudo ipvsadm -a -t 192.168.1.10:80 -r 192.168.1.220:80 -g -w 1
sudo ipvsadm -a -t 192.168.1.10:80 -r 192.168.1.193:80 -g -w 1

# 查看 LVS 状态
sudo ipvsadm -Ln
```

输出如下:
```bash
$ sudo ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.1.10:80 rr
  -> 192.168.1.193:80             Route   1      0          0         
  -> 192.168.1.220:80             Route   1      0          0
```

### 配置 RS

分别在两台 RS 上执行下面的操作:

绑定 VIP 到 loopback:
```bash
sudo ip addr add 192.168.1.10/32 dev lo
sudo ip link set lo up
```

配置 ARP 抑制:
```bash
# 只有当目标 IP 是本接口上的地址才响应 ARP
sudo sysctl -w net.ipv4.conf.all.arp_ignore=1
sudo sysctl -w net.ipv4.conf.lo.arp_ignore=1

# 对外宣告 ARP 时用更合适的源地址
sudo sysctl -w net.ipv4.conf.all.arp_announce=2
sudo sysctl -w net.ipv4.conf.lo.arp_announce=2
```

配置一个简单的 nginx 服务:
```bash
sudo apt-get -y install nginx

# RS2 写 RS2
sudo bash -c 'echo "RS1" > /var/www/html/index.html'

sudo systemctl enable --now nginx
```

### 测试

在局域网内任意一台机器上运行 `curl` 获取内容:
```bash
$ curl http://192.168.1.10/
RS1
$ curl http://192.168.1.10/
RS2
$ curl http://192.168.1.10/
RS1
$ curl http://192.168.1.10/
RS2
```
可以看到 RS1 与 RS2 交替返回.
