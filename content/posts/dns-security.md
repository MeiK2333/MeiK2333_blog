---
title: "DNS 安全"
date: 2019-10-21T10:39:42+08:00
lastmod: 2019-10-24T15:39:42+08:00
---

浅谈 DNS 安全。

<!--more-->

---

> 肩膀放松，身体变轻，这我也知道。可是从你嘴里说出来，却半点用也没有啊！——《挪威的森林》 - 村上春树

---

域名系统 DNS (Domain Name System) 是互联网中最重要的协议之一，DNS 将域名转换为 IP 地址，以便浏览器加载互联网上的资源。

在一开始的时候，DNS 只是为 IP 地址提供一个好记的名字。用户只需要记住谷歌搜索的网站 `google.com`，而不需要记住谷歌的机器 IP（比如 `192.168.1.1` 或者 `2400:cb00:2048:1::c629:d7a2`），以此来方便用户使用。时至今日，DNS 已经拓展出了更多的功能、被应用到了更多的领域，比如负载均衡、主从切换、就近解析等。

作为互联网的基石协议之一，原始的 DNS 协议却是非常脆弱的。基于 UDP 的明文传输使得 DNS 极易被攻击、被记录，以及被篡改。

## DNS 攻击

对于 DNS 协议的实现，本文不做赘述，如果读者有兴趣的话可以去查阅相关资料。概括地说，虽然有多种不同的查询，总体上来看，DNS 可以看作是通过域名找到对应主机的协议。

上面我们提到了，DNS 是基于 UDP 协议的明文传输协议，对 DNS 的攻击手段也主要针对这两部分展开。

### 结果污染/抢答

与 TCP 不同，UDP 不会建立握手连接，是无状态的协议。因此，有可能会受到抢答攻击。我们用 `tcpdump` 和 `dig` 来测试一下。

首先启动 `tcpdump`，监听 53 端口上的 UDP 协议（即 DNS 所用的端口与协议）。

```
➜ sudo tcpdump udp port 53
```

然后使用 `dig` 向 cloudflare 的 DNS 服务器查询 Google 的 IP 地址。

```
➜ dig @1.1.1.1 www.google.com

; <<>> DiG 9.10.6 <<>> @1.1.1.1 www.google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 46880
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;www.google.com.			IN	A

;; ANSWER SECTION:
www.google.com.		73	IN	A	69.63.180.173

;; Query time: 19 msec
;; SERVER: 1.1.1.1#53(1.1.1.1)
;; WHEN: Thu Oct 24 10:32:36 CST 2019
;; MSG SIZE  rcvd: 48
```

与此同时，`tcpdump` 输出了我们此次查询的相关信息。

```
➜ sudo tcpdump udp port 53
tcpdump: data link type PKTAP
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on pktap, link-type PKTAP (Apple DLT_PKTAP), capture size 262144 bytes
10:32:36.569048 IP 10.2.11.54.58549 > one.one.one.one.domain: 46880+ [1au] A? www.google.com. (43)
10:32:36.588321 IP one.one.one.one.domain > 10.2.11.54.58549: 46880 1/0/0 A 69.63.180.173 (48)
10:32:36.744608 IP one.one.one.one.domain > 10.2.11.54.58549: 46880 1/0/1 A 216.58.194.164 (59)
```

可以看到，在 19ms 的时候，DNS 查询返回了结果 `69.63.180.173`。在 180ms 后，DNS 查询又返回了一个不同的结果 `216.58.194.164`。这是咋么回事呢？

我们使用 `traceroute` 和 `nmap` 对 `69.63.180.173` 进行分析，发现这台主机竟然没有开放任何一个端口，也禁止 ICMP 协议。那么这个 IP 就肯定不是 Google 的 IP 了。

在网上查询一下这个 IP 绑定的域名。

<div style="text-align:center">
<img src="/images/dns-security/ip138.png">
</div>

完整的结果可以在[此处](https://site.ip138.com/69.63.180.173/)查看。

而后返回的 IP `216.58.194.164`，我们通过访问之后确定，就是正确的 Google 的 IP。这种操作，就是 DNS 抢答攻击。

部署在网络链路上的某台机器，在收到经过自己的 DNS 查询时，对查询的域名进行判断，如果是需要被拦截的域名，则发送抢答的 UDP 数据回去。由于 DNS 查询时普遍会采用第一个响应的结果，因此这个伪造的抢答数据会被查询方认为是正确的结果，而后续返回的真正的数据则会被丢弃。

### 对明文的攻击

很多网关服务都会提供诸如网站过滤、访问控制这类的功能，这些就可以通过对 DNS 的攻击来实现。

因为 DNS 是明文协议，因此网络管理员是可以轻易的获取你正访问的网站的域名的。如果是某些禁止访问的网站，可能会拦截本次 DNS 请求，从而达到禁止访问的目的。即使没有拦截，网络管理员也可以通过这些数据来判断你都访问了什么网站。

## DNS 安全

### DNSSEC

在 [RFC4033](https://tools.ietf.org/html/rfc4033) 中定义了一种保证 DNS 查询结果不被篡改的方式 DNSSEC，但因为复杂性等原因，到目前为止，DNSSEC 依旧没有成为可行的保证 DNS 安全的方式。而且 DNSSEC 依旧有明文的缺点，易被监控管理。

### DNS over TSL（DoT）

[RFC7858](https://tools.ietf.org/html/rfc7858) 中定义的加密查询方式，通过独立的 853 端口提供服务，通过 TSL 加密，可以保证查询结果不会被污染、也不会被明文记录。

但是由于 DoT 会占用一个单独的端口，网关可以通过限制这个端口流量来限制 DoT 的查询（即使看不到也不能篡改内容）。网络管理员可以关闭这个端口，来迫使用户回退使用原始的 DNS 查询。

### DNS over HTTPS（DoH）

由 Google 率先提供的服务，之后被加入了 [RFC8484](https://tools.ietf.org/html/rfc8484) 中。基本原理是通过 HTTPS 的流量进行 DNS 查询，整个过程中与 HTTPS 的请求没有任何区别（因为它就是一个 HTTPS 的请求）。

通过 DoH，可以保证安全的 DNS 查询，网络管理员不可能关闭 HTTPS 的端口，也无法对其进行监管。是到目前为止最安全的 DNS 查询方式。

也有一些质疑的声音，因为 DoH 会将查询集中提交到 Google 的服务器中，Google 可能收集了用户的查询信息。但无论如何，DoH 看起来已经是目前最优的解决方案了。Firefox 已经率先在浏览器中支持了 DoH。到发文前几天发布的 Chrome 78 也支持了 DoH。

日后，操作系统本身也有可能引入 DoH。到目前为止，主流的系统都可以通过某些软件（比如 Simple-DNSCrypt）来设置全局 DoH。

## 未来展望

### 更多的节点

DoH 看起来很美好，但它也并非是十全十美的。首先是支持的 DNS 服务器还太少，到目前为止，我们基本还只能使用以下几个 DNS 服务器：

- `cloudflare: 1.1.1.1`
- `cloudflare: 1.0.0.1`
- `Google: 8.8.8.8`
- `Google: 8.8.4.4`

这也导致了，只要对这些 IP 进行阻断或者干扰，就可以有效的干扰 DoH 查询。实际上，目前在国内，这几个 IP 就严重的被干扰。如果后续更多的 DNS 节点支持了 DoH，这个问题也就迎刃而解了。

### 就近解析

这个问题其实和上面的问题是有关的，现在很多大型的网站会在全世界各地提供很多节点，用户可以通过连接最接近的节点来获取最好的访问体验。或者在国内，跨运营商的访问会明显变慢。

这些就可以通过 DNS 来进行控制，比如为北京联通的用户提供北京联通的机房解析，在为用户提速的同时可以降低网站的带宽费用。而如果使用了集中式的 DoH，用户就很难再获得细粒度的就近解析了。cloudflare 虽然提供了全球的就近解析，但那也主要是为了他自家的应用服务，对自建服务的 CDN 厂商并不友好。其实总体来看，这些公司踊跃的支持 DoH 等技术，也是为了能推销自家的服务😂。

补充：我问了群里做相关工作的同学，DoH 确实会对就近解析造成影响，目前有几种解决的方向，一个是通过 `edns` 携带查询者源 IP 地址等信息，根据这些信息进行解析；还有一个就是将分配提高到 HTTP 协议的层面，通过 302 重定向，为不同区域的用户设置不同的 CDN 站点。
