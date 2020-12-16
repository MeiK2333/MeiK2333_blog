---
title: SYN Flood 攻击原理与 C 语言实现
date: 2018-08-29 15:55:32
tags: [UNPv1, 网络编程, C 语言]
---
SYN flood 或称 SYN 洪水、SYN 洪泛，是一种拒绝服务攻击，起因于攻击者发送一系列的 SYN 请求到目标系统。 —— [维基百科](https://zh.wikipedia.org/wiki/SYN_flood)

<!--more-->

## 原理

在之前的博客[《TCP-连接的流程与状态转换》](/posts/tcp-status/#TCP-连接的流程)中，我介绍了 TCP 服务器与客户端进行通信的流程。一个服务器为了能够接收并响应来自客户端的请求，需要执行 `socket`、`bind` 以及 `listen` 函数。而在 TCP 连接实际交换数据之前，需要先进行[三次握手](/posts/tcp-connection/)。

SYN flood 就是针对三次握手的机制以及 `listen` 函数进行的攻击。首先看一下 `listen` 的定义。

```C
#include <sys/socket.h>
int listen(int sockfd, int backlog);
// 若成功返回 0，若出错则为 -1
```

其中 `backlog` 的含义没有正式的定义，一般认为这个值是未完成连接队列与已完成连接队列之和。

### 连接队列

TCP 握手的过程默认会由内核自动完成，服务器的应用无需处理这些细节。内核维护两个队列：未完成连接队列与已完成连接队列。

顾名思义，未完成连接队列中是发起但还未完成三次握手的连接，而已完成连接队列中是已经完成三次握手，等待应用程序来获取的连接。进程调用 `accept` 时，已完成连接队列的队头项会被返回给进程。

内核为进程维护的实际队列一般比 `backlog` 值略大，但也是一个有限的值，因此可以针对这个有限的值进行一些攻击。

### 三次握手

关于三次握手的具体过程，我之前在[《TCP-的三次握手与四次挥手》](/posts/tcp-connect)已经介绍过了。这里针对的是其第二次握手，通过大量发送建立 TCP 连接的请求（第一次握手），迫使服务器大量的响应对此请求的确认（第二次握手）。而在连接建立完成，被进程取走之前，这些未完成的半连接都会占用连接队列。

当攻击的连接请求占据了所有的连接队列时，服务器就无法继续响应来自其他客户端的请求，从而使服务器对这些客户端拒绝服务。而攻击者一般会伪造自己的 IP，导致服务器的确认无法被响应，服务器就只能不断尝试重发，直到连接超时为止。

## 实现

实现 SYN flood 攻击需要创建一个请求连接的 TCP 报文，修改其中的源地址 IP 使得对方无法从 IP 来判断攻击者。因此，攻击者需要自己构建一个原始的 IP 报文，并且处理其中的校验和等信息。

```C
/* 创建原始套接字并可以自定义 IP 首部 */
int sockfd;
if ((sockfd = socket(AF_INET, SOCK_RAW, IPPROTO_TCP)) < 0) {
    fprintf(stderr, "socket failure!\n");
    exit(1);
}
int on = 1;
if (setsockopt(sockfd, IPPROTO_IP, IP_HDRINCL, &on, sizeof(on)) < 0) {
    fprintf(stderr, "setsockopt failure!\n");
    exit(1);
}
```

填充其中的 IP 首部与 TCP 信息后，通过 `sendto` 函数发送，就可以完成一次 SYN flood 攻击。

完整代码可以参考：[MeiK - GitHub](https://github.com/MeiK2333/ctrick/blob/master/syn_flood.c)

## 测试

使用 root 用户运行攻击的程序。分别测试了：我自己的腾讯云服务器、我自己的阿里云服务器、网上某钓鱼网站、某山寨求职网站、学校内某由我负责的服务器。攻击方式为单台主机 `while (1)` 发送攻击包（MacOS、1Mbps），访问测试使用的是 curl，每次访问创建新 TCP 连接。

### 腾讯云服务器

攻击前测试一切正常，开始攻击后测试无法请求到页面。

持续攻击大概 5-10S 之后收到腾讯云的短信与邮件，提示我的服务器正在被 DDoS 攻击，峰值流量 58Mbps。但是页面仍然无法访问。

结束攻击后腾讯云给我发送了攻击已停止的短信和邮件。

### 阿里云服务器

攻击前测试一切正常，开始攻击后大概 5S 无法请求到页面。

5S 后收到短信与邮件，阿里云自动进行流量清洗，攻击峰值流量 59Mbps。同时可以继续访问页面。

### 网上某钓鱼网站

这个网站伪造成 QQ 邮箱、QQ 空间，尝试盗取用户信息，查看其 IP 所在地为香港。

开始攻击后基本没有反应...

### 某山寨求职网站

基本没有反应...

### 学校内某服务器

效果斐然，直接无法访问。

## 防范

可以看出，云服务器服务商基本都会对此类攻击有一定的防护措施。尤其是阿里云，防护效果很好。

关于个人运维的服务器要如何防范此类攻击，我没有什么经验，就把网上看到的一些方法抄写在此处，仅供参考。

- 过滤
- 增加积压
- 减少SYN-RECEIVED定时
- 复用古老的半开通TCP
- SYN缓存
- SYN Cookie
- 混合方法
- 防火墙和代理

## 参考

- UNPv1
- [souptonuts.sourceforge.net](http://souptonuts.sourceforge.net/code/rawsockets.c.html)
- [DoS Attacks (SYN Flooding, Socket Exhaustion): tcpdump, iptables, and Rawsocket Tutorial](http://souptonuts.sourceforge.net/tcpdump_tutorial.html)