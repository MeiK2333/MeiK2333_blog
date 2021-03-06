---
title: 面经答案整理二
date: 2018-09-21 23:53:18
tags: [面经, C++]
---

转自牛客网[《C++后台腾讯WXG实习面经》](https://www.nowcoder.com/discuss/77507)，同样按我自己的理解进行了作答。

<!--more-->

## 一颗二叉搜索树，找出树中的第 k 大节点

对于二叉搜索树来说，其中序遍历将按大小顺序依次输出其每个节点。因此只需要中序遍历，然后输出第 k 个即可。

关于具体实现，可以递归，也可以用栈来存储中间节点。

## fork 的过程

首先父进程调用 `fork()`，内核根据父进程的信息复制出一个子进程，子进程与父进程的 PCB 相同，用户态代码和数据也相同，因此此时子进程与父进程看起来状态相同。

然后根据内核调度方式的不同，两个进程先后返回。对于父进程来说，这个函数返回创建的子进程的 PID。而在子进程中，这个函数返回 0。因此可以用返回值来判断当前是父进程还是子进程。（如果 `fork` 执行出现错误，则返回一个负值）

另外，因为 Linux 中执行一个新的进程需要 `fork` + `exec`，这种组合也被大量使用。如果在 `fork` 后紧接着执行 `exec`，那么子进程由父进程复制来的数据几乎毫无作用。对此内核进行了优化：如果在 `fork` 之后没有对数据进行过修改，直接进行 `exec` 的话，则不会复制一份数据给子进程，也就是 Copy-on-write。

## fork、vfork 以及 clone 的区别

这三个函数最终调用的都是 `do_fork()`

### fork 与 vfork

`fork` 与 `vfork` 都是创建一个新的进程，主要区别有以下几点：

- `fork` 子进程拷贝父进程的数据段、代码段；`vfork` 子进程与父进程共享数据段。
- `fork` 父子进程的执行次序不确定；`vfork` 保证子进程先运行。

因为 `vfork` 父子进程共享数据段，而且保证子进程先执行，如果子进程中进行了 `return`，则会扰乱父进程的栈空间，而如果子进程中修改了其他如全局变量的数据，也很可能导致父进程崩溃。

在 `fork` 提供 Copy-on-write 之前，`vfork` 是更好的创建新进程的方法，因为它不需要复制父进程的数据。但是现在基本已经没有继续使用 `vfork` 的必要了。

### clone

`clone` 的功能很强大，可以通过指定参数让它达到 `fork` 或者 `vfork` 的效果，这里不过多说明，有兴趣可以自行搜索。

## 僵尸进程

### 什么是僵尸进程

在 Linux 中，每个进程退出后都有一些资源尚未释放，这是为了让这些进程的退出的信息可以被获取。

但是，这些资源如果迟迟不被释放的话，就会占用系统资源，变成僵尸进程。

### 如何避免僵尸进程

当某个进程还在运行的时候，它的父进程退出了，那这个进程就会变成孤儿进程，然后被 init 进程收养。当这个进程退出之后，init 进程会通过 `wait` 来释放它遗留的资源。

因此，可以通过 `fork` 两次的方式，将要执行的进程变为孙子进程，然后其父进程退出，那么要执行的进程就会变成孤儿进程，由 init 进程来处理它的善后工作。

如果想要自行处理的话，对于某个进程，其子进程退出的时候会给它发送 `SIGCHLD` 信号，这个进程可以监听这个信号，为这个信号安装处理函数，在函数里通过 `wait` 释放子进程的资源。

## 从汇编层去解释一下引用

汇编我一窍不通...这个问题先 pass，日后学了再回来补充。

## 惊群效应

### 什么是惊群效应

多个进程或线程正在阻塞等待同一个事件时，如果被等待的这个事件发生，那么这些进程（线程）会被全部唤醒。如果这些进程（线程）是为了处理这个事件被唤醒的话，那么最后一般只会有一个进程（线程）得到这个事件，其他的进程（线程）将会重新进入休眠状态。

就像是用一粒米来喂一群鸽子，整群鸽子都被惊动跑来争夺这一粒米，最终却只会有一只鸽子获得这粒米。如果发生了惊群效应，那么就会白白浪费了很多的系统资源。

### 如何解决惊群问题

一个典型的惊群问题，一个 web 服务器的多个进程等待 accept 同一个 sockfd，当一个客户端的连接请求到来时，所有的进程都会被唤醒，除了一个能够抢夺到连接之外，其他的都会失败。（较新的 Linux 内核已经解决了这个问题，在这个场景下，会优先指定一个进程来获得连接）

如果我们想要自行解决这个惊群问题的话，可以自己设定一个锁，多个进程同时尝试请求这个锁，只有请求到锁的进程可以休眠等待连接（`accept`），其他的进程要等待这个进程释放掉锁后重新抢夺锁。

另一个场景是使用 `epoll_wait` 时，这个问题已经被解决了。可以通过添加 `EPOLLEXCLUSIVE` 标志来指定在事件发生时只唤醒一个进程。

## 中断的作用

原作者对此有很深的理解，可以在原文中查看。

对我来说，我认知的中断的作用主要就是让单个处理核心能够并行处理多个任务，不会因为单个任务的阻塞影响后续所有任务。

## tcp 的连接关闭

高频问题，我之前写过相关的博客。

- [《TCP 的三次握手与四次挥手》](/posts/tcp-connection/)
- [《TCP 连接的流程与状态转换》](/posts/tcp-status/)

## TIME_WAIT 的作用

依然是我之前的博客[《TCP 的三次握手与四次挥手》](/posts/tcp-connection/#TIME-WAIT-状态)

关于 TIME_WAIT 过多时的解决方案，有几种解决方案：

- TIME_WAIT 重用
- 编译内核，缩短 TIME_WAIT 的时间
- 结束连接时发送 RST 而不是 FIN，以越过 TIME_WAIT

但是这几种方案都违反了设立 TIME_WAIT 的初衷。

其实在我看来，如果是 web 服务器的话，那么升级到较新的协议是有用的。比如 http 0.9 是由服务器关闭连接，那么将由服务器处理这个 TIME_WAIT。而 1.0 以后，服务器可以发送 `Content-Length` 字段，由客户端关闭连接，因此将改由客户端处理关闭连接。还有 http 1.1 中的 `keepalive`，可以使多个 http 请求复用同一个连接，也可以有效的缓解这个问题。

## 如果网络延迟很高，但是又没有发生丢包，利用tcp会使吞吐量下降，该如何解决呢

### TCP 的滑动窗口

参照[《TCP 的滑动窗口》](/posts/tcp-sliding-window/)

### TCP 的拥塞控制

参照[《TCP 的拥塞控制》](/posts-congestion-control/)

### 如果网络延迟很高，但是又没有发生丢包，利用tcp会使吞吐量下降，该如何解决呢

这个问题见仁见智吧，原作者的回答：从应用层封装udp协议或者使用raw socket直接封装ip数据报

我认为也可以采用其他拥塞控制算法，比如 BBR 等。

## DDos 攻击原理

分布式拒绝服务(DDoS: Distributed Denial of Service) 有很多类型，其中最常用的可能就是 SYN Flood 攻击了。

正巧关于这种攻击方式我之前也有过一些测试，这里就直接放博客[《SYN Flood 攻击原理与 C 语言实现》](/2018/08/29/SYN-Flood-攻击原理与-C-语言实现/)。

## 数据库引擎

这里的篇幅肯定是不够的...自行搜索吧。

## 数据库三范式

1. 数据库表中不能出现重复记录，每个字段是原子性的不能再分
2. 第二范式是建立在第一范式基础上的，另外要求所有非主键字段完全依赖主键，不能产生部分依赖
3. 建立在第二范式基础上的，非主键字段不能传递依赖于主键字段（不要产生传递依赖）

## 网络层、数据链路层、传输层的设备有哪些

OSI 七层模型从下到上：

1. 物理层：网线、集线器等
2. 数据链路层：网卡、网桥、二层交换机等
3. 网络层：路由器、多层交换机、防火墙等
4. 传输层：没有硬件设备，TCP 与 UDP 等都是这层的协议
5. 会话层：没有硬件设备与具体协议
6. 表示层：没有硬件设备与具体协议
7. 应用层：没有硬件设备，HTTP、FTP、DNS 等等协议都属于这一层

## 网络层、传输层协议有哪些

1. 物理层：ISO2110、IEEE802、IEEE802.2
2. 数据链路层：SLIP、CSLIP、PPP、ARP、RARP、MTU
3. 网络层：IP、ICMP、RIP、OSPF、BGP、IGMP
4. 传输层：TCP、UDP
5. 会话层：没有具体协议
6. 表示层：没有具体协议
7. 应用层：TFTP、HTTP、SNMP、FTP、SMTP、DNS、Telnet

## 网络层、数据链路层、传输层使用的寻址地址分别是什么

数据链路层使用 mac 寻址，网络层使用 IP，传输层使用 IP + port。

## 内存分配原理

这个问题 CSAPP 中有详细的解答，我将在看过这本书后回来解答这个问题（也许吧）。

## AVL 树、B＋树、红黑树

B+ 树上篇博客有过提及，是一种每层节点要多于两个的排序树，可以以较少的层数存储更多的数据。一般来说可以作为数据库的索引，因为它能够更好的利用磁盘的性质。

AVL 树是自平衡二叉树，红黑树也是大致平衡的二叉树，两者的区别在于 AVL 是严格平衡的，因此在插入一个节点时平均需要更多的旋转次数，但是可以保证 O(log(n)) 的查询复杂度；而红黑树只保证最长的一条路不会比最短的一条长一倍，因此插入时需要的旋转次数少于 AVL 树，但是查询复杂度会略高一点。

AVL 树与红黑树在极端数据的情况下也能保证不错的复杂度，而普通的排序二叉树在插入有序数据的情况下，查找速度可能会劣化为 O(n)。

## poll 和 epoll 的区别

一般都是把 select、poll 和 epoll 放在一起比较。

select 和 poll 很接近，首先将一组要监听的描述符与事件加入监听列表中，当这一组描述符中有任何一个读写事件就绪的时候，就会返回。然后用户需要遍历整组描述符，找到真正就绪的那一个，然后对其进行处理。

epoll 是对 select 和 poll 的改进，它不像 select 一样只能监听 1024 个描述符，它所支持的 FD 上限是最大可以打开文件的数目。

epoll 也不像 select 和 poll 一样，需要遍历整组描述符，epoll 只会返回已就绪的描述符，可以通过返回的 `events` 判断是什么类型的就绪（可读、可写、关闭或错误等）。

## epoll 中 ET 模式与 LT 模式的区别

ET：边缘触发，当有事件发生时，只会通知一次。比如通过 `epoll_wait` 返回了一个就绪的描述符，没有处理完其事件，那么此时再次调用 `epoll_wait`，将不能再获得这个描述符。
LT：水平触发，当有事件发生时，会持续通知。如果某个返回的描述符事件还没有被处理完，那么下一次调用 `epoll_wait` 时，这个描述符依旧会被返回。

---

其他还有一些题目，但那些是原作者专精的方向，与我学习的方向不太相同，因此这里也不对其进行整理了。

## 总结

说实话，我和大佬们差距还是很大的。这篇面经是原作者在大三的下学期写的，我现在大四上学期了，水平还差得远呢...

之前我的学习都太过肤浅了，很多问题，别说原理了，我连过程都不知道。还是要静下心来踏实学习才是啊。

## 参考

- [《UNIX环境高级编程（第3版）》](https://book.douban.com/subject/25900403/) - 人民邮电出版社
- [《Linux C编程--fork()详解》](https://blog.csdn.net/dlutbrucezhang/article/details/8692227) - CSDN
- [《Linux中fork，vfork和clone详解（区别与联系）》](https://blog.csdn.net/gatieme/article/details/51417488) - CSDN
- [《nginx如何解决惊群效应》](https://www.jianshu.com/p/21c3e5b99f4a) - 简书