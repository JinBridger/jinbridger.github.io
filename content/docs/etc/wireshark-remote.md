---
title: "如何用 Wireshark 远程抓包"
weight: 1
date: 2024-08-04
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 如何用 Wireshark 远程抓包

{{< hint info >}}
**原理**

1. 利用 ssh 登录到远程主机上启动 tcpdump
2. 将 tcpdump 的输出通过 ssh 发送到本地再用 Wireshark 进行分析
{{< /hint>}}

## 配置免密启动 tcpdump

如果远程的机器执行 tcpdump 命令需要 sudo 输入密码，可以通过以下操作实现免密输入：

编辑 `sudo` 的配置文件
```bash
sudo visudo
```

在文件的**末尾**添加下面一行:
```
username ALL=(ALL) NOPASSWD: /usr/bin/tcpdump
```
将上面的 `username` 替换为你的用户名, `/usr/bin/tcpdump` 替换为你的 tcpdump 的位置 (可以通过 `which tcpdump` 查看位置)

{{< hint info >}}
**这一行命令的意思**

```
username ALL=(ALL) NOPASSWD: /usr/bin/tcpdump
   |      |    |      |             |
   |      |    |      |             +---------- 可以执行 tcpdump 命令
   |      |    |      +------------------------ 不需要密码
   |      |    +------------------------------- 可以以所有用户/用户组的身份执行
   |      +------------------------------------ 可以在所有主机上执行
   +------------------------------------------- 用户名
```
{{< /hint>}}

配置完成后新建一个终端，可以发现执行 `sudo tcpdump` 指令时不需要输入密码。

## 远程连接

使用如下命令将远程 tcpdump 的输出通过 ssh 发送到本地再用 Wireshark 进行分析:

```bash
# eth0 更换成你的机器 interface 名称, 通过 ip a 查看
ssh username@host "sudo tcpdump -i eth0 -l -w - not port 22" | wireshark -k -i -
```

- 对于 Windows 用户，在 cmd 中执行即可 (在 Wireshark 安装目录下)
- 对于其他平台用户，在终端中执行即可

{{< hint warning >}}
建议设置 `not port 22` 来过滤掉 ssh 连接

<div align="center">
<img src="/image/etc/wireshark-remote/nossh.png" width="40%">
    <br>
    <div style="border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">不过滤 ssh 连接的后果</div>
	<!-- <img src="" >
    msdos -->
</div>

{{< /hint>}}


## 参考资料

- sudoers文件说明 - sudo免密码 - 限制sudo执行特殊命令 https://www.cnblogs.com/liulianzhen99/articles/17629534.html
- 使用 tcpdump 和 Wireshark 进行远程实时抓包分析 https://thiscute.world/posts/tcpdump-and-wireshark
- using wireshark/tshark in command line to ignore ssh connections https://serverfault.com/questions/463820/using-wireshark-tshark-in-command-line-to-ignore-ssh-connections