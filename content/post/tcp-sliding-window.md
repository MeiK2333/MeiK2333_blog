---
title: TCP 的滑动窗口
date: 2018-09-15 15:35:53
tags: [TCP, 网络编程]
---

作为一个可靠的连接，TCP 应该尽力使得数据送达，即当有数据报丢失的话，TCP 应该有某种机制来得知丢失的部分并重传。这个机制就是滑动窗口。

滑动窗口是 TCP 连接中重要的一个部分，TCP 连接的两端通过滑动窗口来动态的调节数据的发送与接收，以保证数据被正确接收而不会因为过多的发送而被淹没；还可以在数据报丢失的时候进行重传，以尽力使数据送达。

<!--more-->

滑动窗口分为发送窗口和接收窗口，因为 TCP 是全双工的连接，因此客户端与服务端都各有一个发送窗口与接收窗口（一般认为，主动发起 TCP 连接的一端为客户端，被动接收 TCP 连接的一端为服务端）。

## 总体流程

首先我们先复习一下 TCP 连接中数据交流的流程（详细的过程可以看我之前的博客[《TCP 连接的流程与状态转换》](/posts/tcp-status/)）：首先通过三次握手建立起一个 TCP 连接，进行数据交互，然后四次挥手结束连接。

为保证其可靠性，TCP 的接收端会对接收到的数据报进行确认（ACK），如果发送端在 RTO（RTO: retransmission timeout，即重新发送超时时间） 的时间内没有收到确认的话，就会重发数据报。具体重发哪些数据，以及一次性可以发送多少数据，都是由上一次的 ACK 附带的滑动窗口的信息来确定的。

## 接收端

在一段时间内，接收窗口可以接收的数据量是有限的。同时接收窗口的大小也是在不断变化的，因此接收端需要在每次对数据确认（ACK）时发送自己当前的窗口大小，发送端根据这个数值来动态的调节发送的数据量。初始的窗口规模在建立 TCP 连接的时候确定。

为了保证数据完整，接收端同时还需要通知发送端目前已成功接收的进度，因此在 ACK 时还会额外附带进度信息。在 TCP 中，这个信息表示为接收端想要接收的下一个字节的位置。比如接收端接收到了 1 - 1024 字节的数据，那么它向发送端发送的确认号将是 1025。而如果此时收到了 2049 - 4096 字节的数据的话，接收端的确认号依旧为 1025。因为中间的数据还没有收到，确认号是已经完整接收到的所有数据的下一位。

## 发送端

因为 TCP 是全双工的连接，因此其实每一端都同时可以为发送端与接收端，这里的发送端与接收端仅仅代表一次传输中的相对关系。

对于发送端来说，要做的就是根据接收端 ACK 中的确认号与窗口规模来确定发送的数据。如果将数据看成一条数据组成的链的话，滑动窗口就是在这条链上不停的向后移动，这也是滑动窗口这个名词的来源。（在[这个回答](https://www.zhihu.com/question/32255109/answer/68558623)里有图示，但我感觉还是《计算机网络》课本里的表述更好理解）

## 参考

- 《计算机网络》 - 电子工业出版社 谢希仁著
- [TCP协议的滑动窗口具体是怎样控制流量的？](https://www.zhihu.com/question/32255109) - 知乎问题