---
title: "防火墙与 iptables"
weight: 1
date: 2026-01-04
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 防火墙与 iptables

当内核接收到来自外部网络的数据包时, 需要根据规则来决定是丢弃还是继续处理数据包.
这个机制就是内核的防火墙.

内核中实现这个机制的框架叫做 Netfilter.

## Netfilter

Netfilter 是 Linux 内核中的一个软件框架, 用于管理网络数据包.
不仅具有网络地址转换 (NAT) 的功能. 也具备数据包内容修改, 以及数据包过滤等防火墙功能。

Netfilter 制定了五个数据包的 Hook 点, 分别是 PREROUTING, INPUT, OUTPUT, FORWARD与POSTROUTING. 用户可以在这五个 Hook 点上定义自己的函数来决定对数据包进行对应的操作.
具体的 Hook 点位置参见下图:

<div align="center">
<img id="mem-class_auto_svg" src="/image/sys/linux-firewall/netfilter.svg" width="90%">
    <br>
    <div style="display: inline-block;
    color: #999;
    padding: 2px;">Netfilter 示意图.</div>
	<!-- <img src="" >
    msdos -->
</div>

有了这五个 Hook 点就可以实现内核层面的防火墙. 例如, 可以在 INPUT 上挂载一个函数, 如果数据包来自某个 IP 就直接丢弃掉, 从而防止来自某个 IP 的攻击.

## iptables

但是直接操作 Netfilter 往 Hook 点上挂函数太麻烦了. 于是就有了 iptables.
它允许通过命令行的方式配置规则.

iptables 包括三个核心概念: Table（表）, Chain（链） 与 Rule（规则）.
- Table（表）：规则的“分类”，决定**做什么类型的处理**
- Chain（链）：规则的“执行点”，决定**在数据包的哪个阶段处理**
- Rule（规则）：具体的“条件 + 动作”，决定**匹配什么、怎么处理**

一个 Table 包含多个 Chain, 一条 Chain 上可以有多个 Rule.

### Table 与 Chain

具体来说, iptables 有多个 table, 这些 table 是 netfilter 设置的, 无法修改.
在每个 table 里面多个 Chain. 这些 Chain 可以分为两种: 内建的 Chain 与用户自定义的 Chain.
- 内建 Chain 不可修改, 默认挂载在 netfiler 的 Hook 上.
- 用户自定义的 Chain 只能从其他 Chain 跳转进来, 不能直接挂载在 Hook 上.


iptables 里面具体的 table 如下:
| Table  | 内建 Chain                                                 | 作用                      |
| ------ | --------------------------------------------------------- | ----------------------- |
| filter | `INPUT`<br/>`OUTPUT`<br/>`FORWARD` | 默认表，用于 **允许 / 拒绝** 流量 |
| nat    | `PREROUTING`<br/>`OUTPUT`<br/>`POSTROUTING`  | 网络地址转换（SNAT / DNAT）     |
| mangle | `PREROUTING`<br/>`INPUT`<br/>`FORWARD`<br/>`OUTPUT`<br/>`POSTROUTING` | 修改数据包（TTL、标记等）          |
| raw    | `PREROUTING`<br/>`OUTPUT` | 绕过连接跟踪                  |

其中最常用的是 filter 表. 用于实现防火墙逻辑.


### Rule

iptables 的 rule =「匹配条件（matches） + 动作（target）」

当数据包进入某个 Chain 后：
1. 按顺序检查每一条 rule
2. 所有 match 都成立 → 命中
3. 执行 target
4. 可能结束，也可能继续

iptables 定义 Rule 的命令为:
```
iptables [-t table] -A chain  [matches...]  -j target  [target-opts...]
```
例如:
```
iptables -A INPUT -p tcp --dport 22 -s 192.168.1.0/24 -j ACCEPT
```
拆解如下:
- `-t filter`          表（默认 filter）
- `-A INPUT`           链           
- `-p tcp`             协议匹配        
- `--dport 22`         目标端口        
- `-s 192.168.1.0/24`  源地址         
- `-j ACCEPT`          动作          


iptables 默认提供一些基础的 matches:
- 协议   : `-p tcp`                      
- 源地址  : `-s 1.2.3.4`                  
- 目的地址 : `-d 5.6.7.8`                  
- 接口   : `-i eth0`                     
- 状态   : `-m state --state ESTABLISHED`

iptables 的 Target 可以是内置的 Target, 也可以是其他的 Chain 用于跳转.

内置的 Target 分为两种:
- 终止型: 命中后执行操作, 然后停止匹配:
  - `ACCEPT`        放行
  - `DROP`          丢弃
  - `REJECT`        拒绝
  - `DNAT` / `SNAT` 地址转换
- 非终止型: 命中后执行操作, 然后继续匹配:
  - `LOG`    记录日志
  - `MARK`   打标
  - `QUEUE`  送用户态
  - `RETURN` 返回上一级 chain 

用 `-j` 来指定跳转的 Target.
例如, `-j MYCHAIN` 会跳转到 `MYCHAIN` 上继续处理, `-j ACCEPT` 则会直接放行.

### 其它常用命令

```
iptables -L           # 查看规则

iptables -L -n        # 不解析域名和端口（推荐）
iptables -L -v        # 显示详细信息（命中次数、流量）
iptables -L -nv       # 最常用组合
iptables -L INPUT     # 查看指定链

iptables -F                 # 清空当前表的所有规则
iptables -t nat -F          # 清空 nat 表
iptables -X                 # 删除自定义链
iptables -Z                 # 清空计数器

iptables -P INPUT DROP      # 设置 INPUT 默认策略为 DROP
```

## nftables

nftables 是新一代防火墙框架，用来取代 iptables / ip6tables / arptables / ebtables.

目前大多数主流发行版中 iptables 只是一个前端命令.
实际规则被转换后交给 nftables 内核子系统.
例如下面的 `(nf_tables)` 就说明底层实现是 nftables.
```
$ iptables --version
iptables v1.8.7 (nf_tables)
```

nftables 的命令为 `nft`, 仍然包括三个核心概念: Table（表）, Chain（链） 与 Rule（规则）.
只不过语法与 iptables 略有不同.

在执行上, nftables 也不像 iptables 要一条一条匹配, 而是用一个解释执行引擎以近乎 O(1) 的复杂度进行匹配. 效率大大提升.
