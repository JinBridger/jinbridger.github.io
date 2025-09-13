---
title: "校园网 ImmortalWrt 配置"
weight: 1
date: 2025-09-01
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 校园网 ImmortalWrt 配置

记录一下在校园网配置 ImmortalWrt.

## 起因

实验室发了新主机, 打算装成 PVE 来用. 而且原有的 Windows 笔记本不出意外也会长期放在实验室, 通过 RDP 来连接. 为每个设备单独配 DDNS 太麻烦了, 于是打算配一个软路由, 把这两个东西放在一起, 让路由器来做 DDNS.

## 购买 & 安装

路由器选的是磊科的 N60 Pro. 有5 个网口. WAN 跟 LAN 1 是 2.5GbE, 剩下的 LAN 2-4 都是 1GbE. 支持 Wi-Fi 6. 内存 512 M, 性价比还可以的同时也有现成的 ImmortalWrt 固件, 非常方便. 某东 340 入手.

到手直接刷入 ImmortalWrt 固件. 安装教程参考 https://post.smzdm.com/p/amv75374/

## 配置

### ddns

使用 luci 配置 DDNS. 在 ImmortalWrt 里面运行

```sh
opkg update && opkg install luci-app-ddns
```

我的域名注册商是 Porkbun, 所以就参考了 [ksorio/ddns-script-porkbun](https://github.com/ksorio/ddns-script-porkbun/blob/main/README.md) 进行配置

值得一提的是, ddns 似乎只能修改已有的解析记录, 因此在配置 ddns 之前, 需要先在域名服务商那里新建对应的解析记录.

为了省事, 我直接把 `*.my.host.com` 给解析了过来.

### acme

由于我的域名结尾是 `.dev`. 而这个 TLD 启用了 HSTS. 因此必须配置 ssl.

~~此项为 TLD 强制要求, 不可能不配.~~

装一个 luci-app-acme 用来自动颁发证书. 既然前面 ddns 把 `*.my.host.com` 解析了过来, 那就直接也自动颁发一个 `*.my.host.com` 的证书. 这样之后部署服务都可以用这个证书. 

我用的是 luci-app-acme. 先简单安装一下
```sh
opkg update && opkg install luci-app-acme
```

然后去 luci 里面配置一下 acme. 对于 wildcard 证书, 只能用 dns 来验证. 在 DNS Challenge Validation 里面填好 DNS API 的密钥就可以了.

这里出了个小问题, 一直提示 `Contact emails @example.org are forbidden`. 查看 `/etc/config/acme` 发现 account_email 并没有被更新. 手动改成了自己的邮箱. 不知道是不是 luci-app-acme 的 bug.

手动 renew 一下
```sh
/etc/init.d/acme renew
```
颁发的证书会放在 `/etc/acme/*.my.host.com_ecc/` 下面


### nginx

设想是路由器对外开放 443, 通过 SNI 判断访问的服务
- 如果是 luci.example.com 就重定向到 luci
- 如果是 pve.example.com 就重定向到 pve
- ...
- 如果没有 SNI 信息直接拒绝握手, 防止被扫到内部的服务

ImmortalWrt 自带的 uhttpd 似乎不太能做到这一点. 因此打算把自带的 uhttpd 换成 nginx 来实现 SNI 判断.

先安装 luci-ssl-nginx 把 luci 切到 nginx 上.

```sh
opkg update && opkg install luci-ssl-nginx
```

`ls /etc/nginx` 发现 nginx 用的是 uci 提供的 `uci.conf`. 为了能自定义, 需要把 nginx 的配置文件换成自己编写的 `nginx.conf`

先新建一个 `/etc/nginx/nginx.conf` (大部分都是从 `uci.conf` 复制的, 只是把 server 去掉换成从 conf.d 中加载):

```nginx
worker_processes auto;

user root;

include module.d/*.module;

events {}

http {
        access_log off;
        log_format openwrt
                '$request_method $scheme://$host$request_uri => $status'
                ' (${body_bytes_sent}B in ${request_time}s) <- $http_referer';

        include mime.types;
        default_type application/octet-stream;
        sendfile on;

        client_max_body_size 128M;
        large_client_header_buffers 2 1k;
        server_names_hash_bucket_size 64;

        gzip on;
        gzip_vary on;
        gzip_proxied any;

        root /www;

        include conf.d/*.conf;
}
```

之后在 `conf.d` 下面新建一个 `all_in_one.conf` 来配置. 这里只放 luci 的配置, pve 与其他服务大同小异.

```nginx
# 重定向 http
server {
    listen 80;
    server_name *.my.host.com;
    return 301 https://$host$request_uri;
}

# luci 配置
server {
    listen 443 ssl;
    http2 on;
    server_name luci.my.host.com_ecc;
    include restrict_locally;

    ssl_certificate     /etc/acme/*.my.host.com_ecc/*.my.host.com.cer;
    ssl_certificate_key /etc/acme/*.my.host.com_ecc/*.my.host.com.key;

    ssl_session_cache shared:SSL:32k;
    ssl_session_timeout 64m;

    include conf.d/luci.locations;
    access_log off;
}

# 禁止使用 IP 访问
server {
    listen 443 ssl default_server;
    server_name _;

    ssl_reject_handshake on;
}
```

之后运行下面的命令把 `uci.conf` 换掉:

```sh
uci set nginx.global.uci_enable=false
uci commit nginx
/etc/init.d/nginx reload
```

在 luci 里面开个 443 的端口映射, 就可以从外部用 `https://luci.my.host.com` 来访问 luci 界面了. 如果被扫到, 尝试通过 IP 访问就会直接拒绝 TLS 握手, 防止被爆破:

```
jinbridge@jinbridge-macbookpro ~ % curl -v https://xxx.xxx.xxx.xxx  
* Uses proxy env variable no_proxy == '127.0.0.1,localhost'
* Uses proxy env variable https_proxy == 'http://127.0.0.1:7890'
*   Trying 127.0.0.1:7890...
* Connected to 127.0.0.1 (127.0.0.1) port 7890
* CONNECT tunnel: HTTP/1.1 negotiated
* allocate connect buffer
* Establish HTTP proxy tunnel to xxx.xxx.xxx.xxx:443
> CONNECT xxx.xxx.xxx.xxx:443 HTTP/1.1
> Host: xxx.xxx.xxx.xxx:443
> User-Agent: curl/8.7.1
> Proxy-Connection: Keep-Alive
> 
< HTTP/1.1 200 Connection established
< 
* CONNECT phase completed
* CONNECT tunnel established, response 200
* ALPN: curl offers h2,http/1.1
* (304) (OUT), TLS handshake, Client hello (1):
*  CAfile: /etc/ssl/cert.pem
*  CApath: none
* LibreSSL/3.3.6: error:1404B458:SSL routines:ST_CONNECT:tlsv1 unrecognized name
* Closing connection
curl: (35) LibreSSL/3.3.6: error:1404B458:SSL routines:ST_CONNECT:tlsv1 unrecognized name
```

### fail2ban

还要给 22 端口开个 fail2ban 防止被爆破.

先把日志保存到文件, 在 luci 界面 - 系统 - 系统 - 系统属性 - 日志 里面把写入文件开启. 为了防止日志过大占空间, 开个 logrotate 配置日志最大为 128k.

之后就是安装 fail2ban
```sh
opkg update && opkg install fail2ban
```

由于 fail2ban 自带的 dropbear 过滤器跟 wrt 自带的 dropbear 日志格式不太兼容, 还要改一下 filter: `/etc/fail2ban/filter.d/dropbear.conf`
```
failregex = ^[Ll]ogin attempt for nonexistent user ('.*' )?from <HOST>:\d+$
            ^[Bb]ad (PAM )?password attempt for .+ from <HOST>(:\d+)?$
            ^[Ee]xit before auth from <<HOST>:\d+>: \(user '.+', \d+ fails\): Max auth tries reached.*$
```

之后配一下 fail2ban 的策略 `/etc/fail2ban/jail.d/dropbear.local`. 我配的是一天内登错 3 次就封禁一个星期.
```
[dropbear]

enabled = true
filter = dropbear
action = iptables[port=22, protocol=tcp]
logpath = /tmp/system.log
maxretry = 3
bantime = 604800
findtime = 86400
```

重启一下 fail2ban:
```
fail2ban-client restart
```

查看黑名单状态:
```
fail2ban-client status dropbear
```

### Wake on LAN

最后给连接路由器的设备 (PVE 跟 Windows 笔记本) 开一个 WoL (Wake on LAN). 这样就能实现远程开机.

luci 直接装一个 WoL 面板即可
```sh
opkg update && opkg install luci-app-wol
```

Windows 机器需要在 BIOS 里面打开 WoL, 并关闭 Windows 的快速启动.

对于 PVE, 除了要在 BIOS 里面开启 WoL 以外, 还要在 PVE 里面也打开 (WoL 功能默认是关闭的).

在 PVE 上面运行 `ethtool eno1 | grep Wake-on` 查看 wol 是否开启

预期输出:
```
Supports Wake-on: pumbg
Wake-on: g
```

如果是 `Wake-on: d` 表示 WoL 未启用，需要设置为启用
```
ethtool -s eno1 wol g
```

上面的指令在重启后会失效, 因此还需要创建 systemd 服务, 让系统开机自动启用 WoL
```
# /etc/systemd/system/wol.service
[Unit]
Description=Enable Wake On Lan
[Service]
Type=oneshot
ExecStart=/sbin/ethtool --change eno1 wol g
[Install]
WantedBy=basic.target
```

## 参考资料

- 磊科N60Pro折腾记录：刷入OpenWrt安装iStore商店＆恢复官方固件 https://post.smzdm.com/p/amv75374/
- ksorio/ddns-script-porkbun https://github.com/ksorio/ddns-script-porkbun
- openwrt使用fail2ban 防止ssh暴力破解 https://www.cnblogs.com/Amelie2024/articles/17800197.html
- [Notebook/AIO] 如何启用网络唤醒功能(Wake on LAN, WOL) https://www.asus.com.cn/support/faq/1049115/
- WoL on PVE https://popolin.net/2025/03/wol-on-pve/