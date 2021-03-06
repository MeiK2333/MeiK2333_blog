---
title: TCP 的拥塞控制
date: 2018-09-16 19:56:47
tags: [TCP, 网络编程]
---

TCP 的拥塞控制主要由慢启动、拥塞避免、快重传和快恢复四部分组成。

Google 开源了新的拥塞控制算法 BBR，在实际应用中体验优于传统算法，但还没有得到普及。现在互联网上大部分的设备还是传统算法，因此传统的 TCP 的拥塞控制还有很有学习的必要的。

<!--more-->

## 名词解释

开始具体介绍这些算法之前，需要先对几个名词进行一下解释

- rwnd: 接收窗口，由 TCP 的接收端确定
- cwnd: 拥塞窗口，由 TCP 的发送端视当前网络状况而定

rwnd 说明了接收端当前可以接收的最大数据量，cwnd 说明了当前网络情况允许发送的最大数据量。因此，实际可以发送的最大数据量就是 `min(rwnd, cwnd)`

- [MSS (Maximum Segment Size): 最大分段大小](https://zh.wikipedia.org/wiki/%E6%9C%80%E5%A4%A7%E5%88%86%E6%AE%B5%E5%A4%A7%E5%B0%8F)
- [RTT (Round Trip Time): 网络往返时间](https://zh.wikipedia.org/wiki/%E4%BE%86%E5%9B%9E%E9%80%9A%E8%A8%8A%E5%BB%B6%E9%81%B2)
- RTO (Retransmission TimeOut): 重传超时时间
- ssthresh: 慢启动门限，如果当前拥塞窗口小于此值则触发慢启动（就是快速增长），其初始值是由系统定义的

## 慢启动

在网络中，如果某个路由器节点没有能够及时的将数据传递出去，那么这些数据将会挤压在路由器的缓存内，等待稍后处理。而如果挤压的数据量超过路由器缓存的承载量的话，多余的数据将会被直接丢弃。

为了防止大量的数据导致网络阻塞，一个新建立的连接在开始的时候会将 cwnd 初始化为 1，即只会发送一个 MSS 的数据，然后随着报文段被确认而逐渐增长。这也是慢启动这个名称的来源。

虽然起点很低，但是 cwnd 是指数级增长的，每次报文被确认都会在原有的基础上乘 2。

1. 初始 -> cwnd = 1
2. 经过一个 RTT -> cwnd = 1 * 2 = 2
3. 经过两个 RTT -> cwnd = 2 * 2 = 4
4. 经过三个 RTT -> cwnd = 4 * 2 = 8
......

可见，虽然 cwnd 的初始值很小，但是增长极快，很快就会达到一个很大的数字。

## 拥塞避免

前面提到，慢启动的增长速度极快，因此如果一直使用慢启动的话，网络必定是承受不住的。因此需要一个限制来确保传输不会过快，这个限制就是 ssthresh，而这个限制的过程就是拥塞避免。

在当前的 cwnd 值小于 ssthresh 时，TCP 执行慢启动，以极快的速度增长。当 cwnd 值达到（即大于等于） ssthresh 时，开始执行拥塞避免。在此阶段，每经过一个 RTT，cwnd 将不再在原有基础上乘 2，而是在原有基础上加 1。从而由指数增长转为线性增长。

## 如果出现了丢包

上面的两个算法确实能让 TCP 较快的开始，同时在某个阈值之后放慢增长速度，从而不至于爆炸增长。但是，线性增长也是在增长，当到达网络的极限之后，就会出现丢包。

当 TCP 超过 RTO 还没有收到对数据的确认的话，将会调整自己的发送速率，主要做以下几件事：

- 将此次连接的 ssthresh 设置为当前 cwnd 值的一半
- 将 cwnd 值设置为 1，重新进入慢启动阶段

## 快重传

以上的两个算法基本已经可以尽可能的提高网络的利用率了，但还是有可以优化的地方。网络环境瞬息万变，如果每个包的丢失都要等一个 RTO 才能重传的话，会浪费很多时间。

从我之前的博客[《TCP 的滑动窗口》](/posts/tcp-sliding-window/)可以看出，TCP 会同时发送多个包，但对每个包的确认并不是这个包的字节编号，而是已经完整收到的最后一个包的下一个字节。

也就是说，如果发送了 1、2、3、4、5 五个包，1 到达的时候确认编号将是 2；如果此时 3 早于 2 到达，那么确认号将仍是 2 而不是 4。如果 2 丢失了的话，3、4、5 到达，将会产生三个编号为 2 的确认。

快重传就利用了这个特性，当发送端连续收到 3 个重复的 ACK 时，将会触发快重传机制。发送端立即重发丢失的包，然后进入快恢复阶段。

## 快恢复

1. 将 ssthresh 设置为当前 cwnd 值的一半
2. 将 cwnd 的值设置为 ssthresh + 3，加 3 是因为收到了三个重复的 ACK，表示有三个“老”的数据包离开了网络
3. 之后如果继续收到重复的 ACK 的话，每个重复的 ACK 将导致当前 cwnd + 1
4. 当收到一个新的 ACK 的时候，快恢复结束。将 cwnd 的值设置为 ssthresh，重新进入拥塞避免阶段

## 总结

TCP 的拥塞控制是一个自适应当前网络环境的算法，指数增长开始 + 持续线性增长也能均衡初始速度与网络压力。快重传 + 快恢复可以应对突发的丢包，保证连接不会因为少量的丢包而直接回到初始状态，在长肥管道网络环境中能提供理想的连接。

## 参考

- 《计算机网络》 - 电子工业出版社 谢希仁著
- [《TCP慢启动、拥塞避免、快速重传、快速恢复》](https://blog.csdn.net/itmacar/article/details/12278769) - CSDN
- [《TCP的拥塞控制 (二)》](https://www.cnblogs.com/zszmhd/p/3623155.html) - 博客园