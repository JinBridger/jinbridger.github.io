---
title: "如何给校园网内地址做 DDNS"
weight: 1
date: 2024-05-12
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 如何在校园网内做 DDNS

{{< hint info >}}
**背景**  
校园网内会给设备分配内网 IPv4 地址 (10.x.x.x)，并且内网可以相互访问。但是内网分配的 IPv4 地址可能会变动，正好年初在 Porkbun 购买了一个域名，于是尝试将内网某设备的 IP 跟域名动态绑定，实现内网通过域名可以访问设备。
{{< /hint >}}

## 什么是 DDNS？

DDNS（Dynamic Domain Name Server，动态域名服务）是将用户的动态 IP 地址映射到一个固定的域名解析服务上，用户每次连接网络的时候客户端程序就会通过信息传递把该主机的动态 IP 地址传送给位于服务商主机上的服务器程序，服务器程序负责提供DNS服务并实现动态域名解析。

## 以 Porkbun 提供的 API 为例进行 DDNS 绑定

Porkbun 是一家通过 ICANN 认证的域名商，成立于 2016 年，总部位于美国。
Porkbun 创始人 Raymond 同时也是 SnapNAMES 的创始人，域名界的名人。
另外，Porkbun 有很多的优点：免费送 WHOIS 隐私保护，后台操作简单，没有隐藏费用，客服回复快， URL 发（301，302，屏蔽转发），账号登陆二次验证和账号登陆邮件通知。

年初在 Porkbun 购买了一个自己的域名 (jinbridge.dev)，下面就以这个域名为例，尝试将 `example.jinbridge.dev` 这个域名绑定到某个设备上去。

{{< hint warning >}}
**关于 .dev 域名可能要注意的一点**  
.dev 域名是强制启用 https 的域名，因此如果部署网站或者服务的话可能要注意一下，否则在浏览器中可能无法访问。（不过类似远程桌面的应用并不会受到什么影响）
{{< /hint >}}

在使用对应的 API 之前，需要生成对应的 Key. 具体可以看 [Porkbun 官方给出的文档.](https://kb.porkbun.com/article/190-getting-started-with-the-porkbun-api)

在设置完以后，可以获得对应的 API Key 跟 Secret Key. 接下来就是将 DDNS 服务部署到机器上去。这里我基于 Porkbun 官方的 [porkbun-dynamic-dns-java](https://github.com/porkbundomains/porkbun-dynamic-dns-java) 进行了修改。

### 获取内网 IPv4 地址

我采用的方法是 curl 校园网登录网站的方法。执行 `curl w.seu.edu.cn` 响应中就会带有 IPv4 地址：

```bash
...
pwm=1;v6af=0;v6df=0;uid='213212533';pwd='';v46m=0;v4ip='10.208.100.251';v6ip='::';// 0123456';v46m=001;v4ip='192.168.100.100';v6ip='0000:0000:0000:0000:0000:0000:0000:0000';////
...
```

其中的 `v4ip='10.208.100.251'` 就是校园网内的 IPv4 地址。简单写个正则就可以匹配出来。
Java 代码如下：

``` java
private static String getInternalIPv4Address() throws IOException {
    String url = "w.seu.edu.cn";
    String command = "curl " + url;
    String curlResult = "";
    try {
        Process process = Runtime.getRuntime().exec(command);
        BufferedReader reader = new BufferedReader(
                new InputStreamReader(process.getInputStream()));
        String line;
        while ((line = reader.readLine()) != null) {
            curlResult += line;
        }
        reader.close();
    } catch (Exception e) {
        e.printStackTrace();
    }
    String ipv4Pattern = "v4ip='(\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3})'";
    Pattern pattern = Pattern.compile(ipv4Pattern);
    Matcher matcher = pattern.matcher(curlResult);
    while (matcher.find()) {
        return matcher.group(1);
    }
    return "";
}
```

### 将内网的 IPv4 地址绑定到 Porkbun 上面

按照 [porkbun-dynamic-dns-java](https://github.com/porkbundomains/porkbun-dynamic-dns-java) 的 README 执行即可，以绑定 `example.jinbridge.dev` 为例，指令如下：

```bash
java -jar PATH/TO/porkbun-ddns.jar jinbridge.dev "example" A
```

返回信息如下：

```bash
jinbridge@jinbridge-vm:~/porkbun-dynamic-dns-java$ java -jar /home/jinbridge/porkbun-dynamic-dns-java/porkbun-ddns.jar jinbridge.dev "example" A
API endpoint: https://porkbun.com/api/json/v3
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
200 OK
Detected current IPv4 as xxxxxxxxxxxxxx.
200 OK
A record for example.jinbridge.dev is currently xxxxxxxxxxxxxx.
```

说明成功绑定了。

### 将绑定指令添加到定时任务中

把指令添加到 crontab 中即可，以每十分钟绑定一次为例，输入 `crontab -e` 编辑 crontab 文件，将以下内容添加进去：

```
*/10 * * * * java -jar PATH/TO/porkbun-ddns.jar jinbridge.dev "example" A
```

{{< hint info >}}
**crontab 格式**
```
f1 f2 f3 f4 f5 program
```
- 其中 `f1` 是表示分钟，`f2` 表示小时，`f3` 表示一个月份中的第几日，`f4` 表示月份，`f5` 表示一个星期中的第几天。`program` 表示要执行的程序。
- 当 `f1` 为 `*` 时表示每分钟都要执行 `program`，`f2` 为 `*` 时表示每小时都要执行程序，其此类推
- 当 `f1` 为 `a-b` 时表示从第 `a` 分钟到第 `b` 分钟这段时间内要执行，`f2` 为 `a-b` 时表示从第 `a` 到第 `b` 小时都要执行，其此类推
- 当 `f1` 为 `*/n` 时表示每 `n` 分钟个时间间隔执行一次，`f2` 为 `*/n` 表示每 `n` 小时个时间间隔执行一次，其此类推
- 当 `f1` 为 a, b, c,... 时表示第 a, b, c,... 分钟要执行，`f2` 为 a, b, c,... 时表示第 a, b, c...个小时要执行，其此类推
```
*    *    *    *    *
-    -    -    -    -
|    |    |    |    |
|    |    |    |    +----- 星期中星期几 (0 - 6) (星期天 为0)
|    |    |    +---------- 月份 (1 - 12) 
|    |    +--------------- 一个月中的第几天 (1 - 31)
|    +-------------------- 小时 (0 - 23)
+------------------------- 分钟 (0 - 59)
```
{{< /hint >}}

最后通过 `grep CRON /var/log/syslog` 确认是否执行了定时任务，输出如下：

```
......
May 11 14:17:01 jinbridge CRON[2866]: (root) CMD (   cd / && run-parts --report /etc/cron.hourly)
May 11 14:30:01 jinbridge CRON[3019]: (jinbridge) CMD (java -jar /home/jinbridge/porkbun-dynamic-dns-java/porkbun-ddns.jar jinbridge.dev "example" A)
May 11 14:30:14 jinbridge CRON[3018]: (CRON) info (No MTA installed, discarding output)
```

可以看到确实正常执行了。