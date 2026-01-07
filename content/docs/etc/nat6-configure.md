---
title: "OpenWrt 校园网 ipv6 NAT 配置"
weight: 1
date: 2026-01-06
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# OpenWrt 校园网 ipv6 NAT 配置

记录一下在校园网内配置 NAT6 的过程.

## 获取 ipv6 地址

如果你的 ISP 提供了 ipv6 但 ImmortalWrt 没有获取到 ipv6 地址, 一个可能的原因是只下发了一个单一的 ipv6 地址. (比如在校园网就有可能出现这种情况).

对于这种情况进行如下操作:

- 在 网络 - 接口 - wan6 - 编辑 中:
  - 将 `请求 IPv6 前缀` 设置为已禁用.

不出意外的话, 首页 - 网络部分会多出来一个 IPv6 上游. 地址段前缀为 128. (地址段里只有一个地址)

## 配置 NAT6

这里很多网上的教程都有点过时, 盲目的使用命令行进行配置有可能导致变砖.

这里采用 LuCI 进行配置.
参考 https://www.cnblogs.com/MAENESA/p/18781636

### 先前准备

- 请确保你的 路由器设备已经具有 IPv6, 并可以正常访问 IPv6 网站
- 请确保你的 OpenWRT 具有以下功能
  - 网络 - 防火墙 - NAT规则
  - 网络 - 接口 - LAN - DHCP服务器 - IPv6 设置
  - 网络 - 接口 - WAN - DHCP服务器 - IPv6 设置
- 本文章默认读者打开了 dns 的 v6 解析
- 本文章默认:
  - 广域网 IPv4 的接口是 wan
  - 广域网 IPv6 的接口是 wan6
  - 局域网 的接口是 LAN

1. 在 网络 - 接口 - wan6 - 编辑 - 高级设置 中:
   - 取消勾选 `IPv6 源路由`
   > [!INFO]
   > 如果不关闭，OpenWRT会自动设置一个 过于精准的IPV6路由 ，例如 `default from 2409:xxxx::/64 via fe80::... dev WAN6` ,而我们需要的是 `default via fe80::... dev WAN6` ,这会导致 NATv6 没有默认路由 而无法使用v6

### 设置内网 IPv6 地址段
1. 在 网络 - 接口 - LAN - 编辑 - 高级设置 中:
   - `委托 IPv6 前缀` 设置为 `禁用`
   - `IPv6 分配长度` 设置为 `已禁用`
2. 在 网络 - 接口 - LAN - 编辑 - 常规设置 中:
   - `IPv6 地址` 设置为 内网 IPv6 地址 (比如 `fd91:417b:3eff::/48` )
3. 在 网络 - 接口 - LAN - 编辑 - DHCP 服务器 - IPv6 设置 中:
   - 取消勾选 `指定的主接口`
   - `RA 服务` 设置为 `服务器模式`
   - `DHCPv6 服务` 设置为 `服务器模式`
   - `NDP 代理` 设置为 `已禁用`
4. 在 网络 - 接口 - LAN - 编辑 - DHCP 服务器 - IPv6 RA 设置 中:
   - `默认路由器` 设置为 `强制的`
   - 勾选 `启用 SLAAC`
  
### 设置 NAT 规则
1. 在 网络 - 防火墙 - WAN - 编辑 - 高级设置 中:
   - 勾选 `IPv6 伪装`
2. 在 网络 - 防火墙 - WAN - 编辑 - 常规设置 中:
   - 勾选 `MSS 钳制`


## 参考资料
- [OpenWRT在UI里配置NATv6和中继IPV6](https://www.cnblogs.com/MAENESA/p/18781636)
