---
title: "DNS 域名系统"
date: 2019-03-04T23:55:47+08:00
categories:
    - Backend
tags:
    - Backend
    - tips
---

DNS （Domain Name System）的作用，就是根据域名查出 IP 地址，是把 www.example.com 等域名转换成 IP 地址。

域名系统是分层次的，一些 DNS 服务器位于顶层。当查询（域名） IP 时，路由或 ISP 提供连接 DNS 服务器的信息。较底层的 DNS 服务器缓存映射，它可能会因为 DNS 传播延时而失效。DNS 结果可以缓存在浏览器或操作系统中一段时间，时间长短取决于 [存活时间 TTL](https://en.wikipedia.org/wiki/Time_to_live)。

CloudFlare 等平台提供管理 DNS 的功能。某些 DNS 服务通过集中方式来路由流量:

* 加权轮询调度
    * 防止流量进入维护中的服务器
    * 在不同大小集群间负载均衡
* A/B 测试
* 基于延迟路由
* 基于地理位置路由

一些缺陷:

* 虽说缓存可以减轻 DNS 延迟，但连接 DNS 服务器还是带来了轻微的延迟
* 虽然它们通常由政府，网络服务提供商和大公司管理，但 DNS 服务管理仍是复杂的
* DNS 服务最近遭受 DDoS 攻击，阻止不知道 Twitter IP 地址的用户访问 Twitter

延伸阅读:

* [DNS 架构](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-R2-and-2008/dd197427(v=ws.10))
* [DNS wiki](https://en.wikipedia.org/wiki/Domain_Name_System)
* [关于 DNS 的文章](https://support.dnsimple.com/categories/dns/)


## 查询过程

虽然只需要返回一个 IP 地址，但是 DNS 的查询过程非常复杂，分成多个步骤。

工具软件 `dig` 可以显示整个查询过程

```
dig www.github.com
```

输出

```
; <<>> DiG 9.10.6 <<>> www.github.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 42658
;; flags: qr rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 8, ADDITIONAL: 10

;; QUESTION SECTION:
;www.github.com.			IN	A

;; ANSWER SECTION:
www.github.com.		3451	IN	CNAME	github.com.
github.com.		300	IN	A	52.74.223.119
github.com.		300	IN	A	13.250.177.223
github.com.		300	IN	A	13.229.188.59

;; AUTHORITY SECTION:
github.com.		191	IN	NS	ns-1707.awsdns-21.co.uk.
github.com.		191	IN	NS	ns4.p16.dynect.net.
github.com.		191	IN	NS	ns1.p16.dynect.net.
github.com.		191	IN	NS	ns2.p16.dynect.net.
github.com.		191	IN	NS	ns3.p16.dynect.net.
github.com.		191	IN	NS	ns-520.awsdns-01.net.
github.com.		191	IN	NS	ns-421.awsdns-52.com.
github.com.		191	IN	NS	ns-1283.awsdns-32.org.

;; ADDITIONAL SECTION:
ns-1283.awsdns-32.org.	130431	IN	A	205.251.197.3
ns-1283.awsdns-32.org.	130432	IN	AAAA	2600:9000:5305:300::1
ns-1707.awsdns-21.co.uk. 130431	IN	A	205.251.198.171
ns-1707.awsdns-21.co.uk. 130431	IN	AAAA	2600:9000:5306:ab00::1
ns-520.awsdns-01.net.	130423	IN	A	205.251.194.8
ns-520.awsdns-01.net.	130423	IN	AAAA	2600:9000:5302:800::1
ns3.p16.dynect.net.	44027	IN	A	208.78.71.16
ns4.p16.dynect.net.	44027	IN	A	204.13.251.16
ns1.p16.dynect.net.	44027	IN	A	208.78.70.16
ns-421.awsdns-52.com.	130431	IN	A	205.251.193.165

;; Query time: 25 msec
;; SERVER: 192.168.0.1#53(192.168.0.1)
;; WHEN: Mon Mar 04 23:21:09 CST 2019
;; MSG SIZE  rcvd: 510
```

第一段是查询参数和统计

```
; <<>> DiG 9.10.6 <<>> www.github.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 42658
;; flags: qr rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 8, ADDITIONAL: 10
```

第二段是查询内容

```
;; QUESTION SECTION:
;www.github.com.			IN	A
```

上面结果表示，查询域名 `www.github.com` 的 `A 记录 `，`A` 是 address 的缩写

第三段是 DNS 服务器的答复

```
;; ANSWER SECTION:
www.github.com.		3451	IN	CNAME	github.com.
github.com.		300	IN	A	52.74.223.119
github.com.		300	IN	A	13.250.177.223
github.com.		300	IN	A	13.229.188.59
```

上面结果显示，`www.github.com` 有 3 个 A 记录，即 3 个 IP 地址。300 是 TTL 值（Time to live 的缩写），表示缓存时间，即 300 秒之内不用重新查询

第四段显示 `www.github.com` 的 `NS 记录 `（Name Server 的缩写），即哪些服务器负责管理 `www.github.com` 的 DNS 记录

```
;; AUTHORITY SECTION:
github.com.		191	IN	NS	ns-1707.awsdns-21.co.uk.
github.com.		191	IN	NS	ns4.p16.dynect.net.
github.com.		191	IN	NS	ns1.p16.dynect.net.
github.com.		191	IN	NS	ns2.p16.dynect.net.
github.com.		191	IN	NS	ns3.p16.dynect.net.
github.com.		191	IN	NS	ns-520.awsdns-01.net.
github.com.		191	IN	NS	ns-421.awsdns-52.com.
github.com.		191	IN	NS	ns-1283.awsdns-32.org.
```

上面结果显示 `www.github.com` 共有 8 条 NS 记录，即 8 个域名服务器，向其中任一台查询就能知道 `www.github.com` 的 IP 地址是什么

第五段是上面 8 个域名服务器的 IP 地址，这是随着前一段一起返回的

```
;; ADDITIONAL SECTION:
ns-1283.awsdns-32.org.	130431	IN	A	205.251.197.3
ns-1283.awsdns-32.org.	130432	IN	AAAA	2600:9000:5305:300::1
ns-1707.awsdns-21.co.uk. 130431	IN	A	205.251.198.171
ns-1707.awsdns-21.co.uk. 130431	IN	AAAA	2600:9000:5306:ab00::1
ns-520.awsdns-01.net.	130423	IN	A	205.251.194.8
ns-520.awsdns-01.net.	130423	IN	AAAA	2600:9000:5302:800::1
ns3.p16.dynect.net.	44027	IN	A	208.78.71.16
ns4.p16.dynect.net.	44027	IN	A	204.13.251.16
ns1.p16.dynect.net.	44027	IN	A	208.78.70.16
ns-421.awsdns-52.com.	130431	IN	A	205.251.193.165
```

第六段是 DNS 服务器的一些传输信息

```
;; Query time: 25 msec
;; SERVER: 192.168.0.1#53(192.168.0.1)
;; WHEN: Mon Mar 04 23:21:09 CST 2019
;; MSG SIZE  rcvd: 510
```

本机的 DNS 服务器是 `192.168.0.1`，查询端口是 `53`（DNS 服务器的默认端口），以及回应长度是 510 字节。

如果不想看到这么多内容，可以使用 `+short` 参数

## DNS 服务器
首先，本机一定要知道 DNS 服务器的 IP 地址，否则上不了网。通过 DNS 服务器，才能知道某个域名的 IP 地址到底是什么

DNS 服务器的 IP 地址，有可能是动态的，每次上网时由网关分配，这叫做 DHCP 机制；也有可能是事先指定的固定地址。Linux 系统里面，DNS 服务器的 IP 地址保存在 ·/etc/resolv.conf· 文件。

有一些公网的 DNS 服务器，也可以使用，其中最有名的就是 Google 的 `8.8.8.8` 和 Level 3 的 `4.2.2.2`。

本机只向自己的 DNS 服务器查询，dig 命令有一个 `@` 参数，显示向其他 DNS 服务器查询的结果。

```
dig @8.8.8.8 www.github.com
```

上面命令指定向 DNS 服务器 8.8.8.8 查询

## 域名的层级
DNS 服务器怎么会知道每个域名的 IP 地址呢？答案是分级查询。

请仔细看前面的例子，每个域名的尾部都多了一个点 `.`

```
;; QUESTION SECTION:
;www.github.com.			IN	A
```

比如，域名 `www.github.com` 显示为 `www.github.com.`。这不是疏忽，而是所有域名的尾部，实际上都有一个根域名。

举例来说，`www.example.com` 真正的域名是 `www.example.com.root`，简写为 `www.example.com.`。因为，根域名 `.root` 对于所有域名都是一样的，所以平时是省略的

根域名的下一级，叫做 "顶级域名"（top-level domain，缩写为 TLD），比如 `.com`、`.net`；再下一级叫做 "次级域名"（second-level domain，缩写为 SLD），比如 `www.example.com` 里面的 `.example`，这一级域名是用户可以注册的；再下一级是主机名（host），比如 `www.example.com` 里面的 `www`，又称为 "三级域名"，这是用户在自己的域里面为服务器分配的名称，是用户可以任意分配的。

即:

```

主机名. 次级域名. 顶级域名. 根域名

# 即

host.sld.tld.root
```

## 根域名服务器

DNS 服务器根据域名的层级，进行分级查询。

需要明确的是，每一级域名都有自己的 `NS 记录 `，`NS 记录 ` 指向该级域名的域名服务器。这些服务器知道下一级域名的各种记录。

所谓 "分级查询"，就是从根域名开始，依次查询每一级域名的 `NS 记录 `，直到查到最终的 IP 地址，过程大致如下

```
从 "根域名服务器" 查到 "顶级域名服务器" 的 NS 记录和 A 记录（IP 地址）

从 "顶级域名服务器" 查到 "次级域名服务器" 的 NS 记录和 A 记录（IP 地址）

从 "次级域名服务器" 查出 "主机名" 的 IP 地址
```

仔细看上面的过程，你可能发现了，没有提到 DNS 服务器怎么知道 "根域名服务器" 的 IP 地址。回答是 "根域名服务器" 的 NS 记录和 IP 地址一般是不会变化的，所以内置在 DNS 服务器里面

目前，世界上一共有十三组根域名服务器，从 `A.ROOT-SERVERS.NET` 一直到 `M.ROOT-SERVERS.NET`

## 分级查询的实例
`dig` 命令的 `+trace` 参数可以显示 `DNS` 的整个分级查询过程。

```
dig +trace www.github.com
```

## NS 记录的查询

`dig` 命令可以单独查看每一级域名的 NS 记录

```
dig ns com
dig ns github.com
```

`+short` 参数可以显示简化的结果

```
dig +short ns com
dig +short ns github.com
```

## DNS 的记录类型

域名与 IP 之间的对应关系，称为 "记录"（record）。根据使用场景，"记录" 可以分成不同的类型（type），前面已经看到了有 A 记录和 NS 记录

常见的 DNS 记录类型如下

- NS 记录（域名服务） 指定解析域名或子域名的 DNS 服务器。
- MX 记录（邮件交换） 指定接收信息的邮件服务器。
- A 记录（地址） 指定域名对应的 IP 地址记录。
- CNAME（规范） 一个域名映射到另一个域名或 CNAME 记录（ example.com 指向 www.example.com ）或映射到一个 A 记录。
- PTR 逆向查询记录（Pointer Record）只用于从 IP 地址查询域名

一般来说，为了服务的安全可靠，至少应该有两条 NS 记录，而 A 记录和 MX 记录也可以有多条，这样就提供了服务的冗余性，防止出现单点失败

`CNAME` 记录主要用于域名的内部跳转，为服务器配置提供灵活性，用户感知不到。

PTR 记录用于从 IP 地址反查域名。`dig` 命令的 `-x` 参数用于查询 PTR 记录。
逆向查询的一个应用，是可以防止垃圾邮件，即验证发送邮件的 IP 地址，是否真的有它所声称的域名

`dig` 命令可以查看指定的记录类型

```
dig a github.com
dig ns github.com
dig mx github.com
```

## 其他 DNS 工具
host 命令:

`host` 命令可以看作 `dig` 命令的简化版本，返回当前请求域名的各种记录。

`host` 命令也可以用于逆向查询，即从 IP 地址查询域名，等同于 `dig -x <ip>`

```
host github.com
```

```
github.com has address 13.250.177.223
github.com has address 52.74.223.119
github.com has address 13.229.188.59
github.com mail is handled by 1 aspmx.l.google.com.
github.com mail is handled by 10 alt3.aspmx.l.google.com.
github.com mail is handled by 5 alt1.aspmx.l.google.com.
github.com mail is handled by 10 alt4.aspmx.l.google.com.
github.com mail is handled by 5 alt2.aspmx.l.google.com.
```

nslookup 命令:

`nslookup` 命令用于互动式地查询域名记录

```
nslookup

> facebook.github.io
Server:     192.168.1.253
Address:    192.168.1.253#53

Non-authoritative answer:
facebook.github.io  canonical name = github.map.fastly.net.
Name:   github.map.fastly.net
Address: 103.245.222.133
```

whois 命令:
`whois` 命令用来查看域名的注册情况

```
whois github.com
```

