---
title: 面经答案整理三
date: 2018-09-22 18:18:16
tags: [面经, C++]
---

转自牛客网[《腾讯面筋》](https://www.nowcoder.com/discuss/89707)。

<!--more-->

## 讲讲 priority_queue 的底层数据结构、实现过程和原理，有没有优化的方式。

priority_queue（优先队列）底层数据结构是一个大顶堆，因此默认情况下 `pop` 优先队列是由大到小。如果想要由小到大获取，则需要重载 `<` 运算符。

## 讲讲 hash 的原理，解决哈希冲突的方法，时间复杂度是多少，怎么优化。

hash 是通过某种函数将值映射到某个位置，从而能够进行快速的查找。这个函数叫散列函数，存储数据的数据结构叫散列表。

解决 hash 冲突的办法主要有：

- 开放定址法：当要插入的数据冲突的时候，通过某种方式（线行探查法、平方探查法、双散列函数探查法）去重新尝试插入。
- 链地址法：散列函数的结果位置并不直接存储数据，而是存储一条链表的地址，如果有冲突的数据，那么直接添加到链表后即可。
- 再哈希法：指定多个哈希函数，当某个函数冲突时，更换函数重新哈希。
- 建立公共溢出区：将溢出的数据统一放入公共溢出区

Java 中的 HashMap 底层实现是链地址法 + 红黑树，当某一个位置的冲突个数不超过 7 个时链表顺序存储，超过 7 个后转化为红黑树。

在不冲突的情况下，只需要通过散列函数找到对应的位置即可获得数据，因此时间复杂度最优可以达到 O(1)。

## Hash 和搜索二叉树的优缺点，哈希表的数据迁移。

hash 可以达到更好的查找和插入的复杂度，但是需要占用的空间更多。

搜索二叉树不需要多余的空间，但是查找和插入的复杂度较高。裸的搜索二叉树查找的复杂度可能会劣化到 O(n)，AVL 或者红黑树可以将查找的平均复杂度优化到 O(log(n))。

C++中的 map 就是红黑树实现的，而 unordered_map 是 hash 表实现的。

## 一个 8g 的数据文件，怎样找出积分排名前 100 的用户。

前 k 大问题可以用大顶堆来快捷的解决。

还有一个问题是第 k 大问题。

解决第 k 大问题可以借鉴快排的思路，就是先选定一个数，将大于它放在它的左边，小于它的放在右边。此时如果这个数左边的数多于 k 个，则递归去找左边的第 k 大数，如果左边的数少于 k 个，则去找右边的第 k - left 大的数。如果正好这个数是要求的数，则问题解决。

同理，求中位数的问题也可以这样解决。

## epoll 和 select 的优缺点,如何监听 tcp 丢包问题。

这两个问题为啥会在一块...但是这两个问题我在之前的博客里都已经有过解答了。

1. [poll 和 epoll 的区别](/posts/interview2/#poll-和-epoll-的区别)
2. [TCP 的滑动窗口](/posts/tcp-sliding-window/)

## TCP 和 UDP 的细节知识点（建立和断开的状态转移）。

高频问题...直接放原来的博客：[《TCP 连接的流程与状态转换》](/posts/tcp-status/)、[《TCP 的三次握手与四次挥手》](/posts/tcp-connection/)。

## 单链表找倒数第 n 个节点，说所有你能想到的方法。

最直观的方法就是先扫一遍链表，获得总的个数，然后通过总的个数来确定倒数第 n 个是哪个节点。总共需要扫两遍节点。

还有一种办法就是用两个指针，一个指针先扫描，当第一个指针扫到第 n 个时第二个指针开始出发，当第一个指针走到尾部的时候，第二个指针指向的就是倒数第 n 个节点。

## 多线程的问题及解决方法。

这问题略显笼统啊...

## 线程和进程的区别以及优缺点。

这个问题也是个高频问题，参照我原来的答案：[进程和线程的区别是什么？](/posts/interview1/#进程和线程的区别是什么？)

## 服务器建立连接过程？

先假定是个 web 服务器，首先由内核建立一个 TCP 连接，服务进程可以由内核获得这个连接（`accept`），并创建一个全双工的套接字。服务进程可以通过读写这个套接字来与客户端进行数据的交互。

服务器读取并解析客户端发来的请求报文，并根据需要返回封装的数据（HTTP -> TCP -> IP）给客户端。

## 三次握手过程？第二条包丢了会怎么办？第三条丢了会怎样？有什么现象？

过程刚刚提过了。

第二条丢失的话，主动发起的一方将会重发第一次握手的请求。

第三条丢失的话，主动发起方已经认为 TCP 连接建立完成了，因此会直接发送数据。当被动方（服务器）接收到数据时会调整到建立连接完成的状态。

## http 报头格式？

![](https://upload-images.jianshu.io/upload_images/2964446-fdfb1a8fce8de946.png)

HTTP 里的格式很简单，比较复杂的是各种头部字段及其含义。

## http 有哪些方法？返回状态码？

GET、POST、HEAD、OPTIONS、PUT、DELETE、TRACE、CONNECT 共八种。

在浏览器判断对应文件是否存在时会使用 HEAD，而在跨站请求时浏览器会用 OPTIONS 来确定对应网站的跨站规则。

HTTP 的返回状态码有很多，最常用的有：

- 200：请求正常
- 206：部分请求，常见于请求静态资源
- 301：永久重定向
- 302：临时重定向
- 304：资源未修改，需要浏览器提供缓存资源的信息（修改时间，哈希值）
- 400：服务器无法理解客户端的请求
- 403：服务器拒绝执行请求，一般是因为权限不足
- 404：赫赫有名的状态码，请求的资源不存在
- 500：服务器内部错误
- 502：网关错误，一般是充当代理的服务器从远端服务器接收到了无效的请求
- 504：超时，一般是充当代理的服务器未及时从远端服务器获得请求数据

## Linux 下如何查看端口号？

不知道具体场景，可以使用 `lsof` 命令，也可以用 `netstat`。

## C语言里面字符串，strcpy 函数和 strncpy 函数的区别？哪个函数更安全？

`strncpy` 比起 `strcpy` 来说，添加了缓冲区长度的限制，可以防止缓冲区溢出。

明显是 `strncpy` 更安全。

## 怎么把一颗二叉树原地变成一个双向链表？

在不申请新的节点的前提下变换，可以采用递归的思想。对每个节点，左边指向它左子树变换后的最右边的节点，右边指向它右子树变换后的最左边的节点。当递归到没有子节点时直接返回，递归结束后就完成了全部的变换。

---

大致的内容就是这些，去掉了一些之前已经解答过的问题和一些原作者个人的问题。

## 参考

- [《解决哈希冲突的常用方法分析》](https://www.jianshu.com/p/4d3cb99d7580) - 简书
- [《第k大问题各类变种小结》](https://segmentfault.com/a/1190000009645951) - segmentfault
- [《关于HTTP协议，一篇就够了》](https://www.jianshu.com/p/80e25cb1d81a) - 简书