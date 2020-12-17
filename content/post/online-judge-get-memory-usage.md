---
title: 关于评测姬的内存测量问题
date: 2018-08-09 22:00:39
tags: [攻略, 评测姬]
---

SDUT OJ 的评测姬有时候会造成异常 MLE 的问题, 比如 [字典树](https://acm.sdut.edu.cn/onlinejudge2/index.php/Home/Index/problemdetail/pid/2828.html) 题目, 或者是 [装备合成](https://acm.sdut.edu.cn/onlinejudge2/index.php/Home/Index/problemdetail/pid/3545.html)(第八届校赛题目, 有一位同学在比赛中疯狂提交 20+ 发 MLE) .

然后, 有人会发现, 使用静态变量(栈区/静态区)的代码居然可以通过, 而使用同样多内存的动态分配(堆区)的代码却无论如何都是 MLE, 这难道是玄学问题咩?

本期的走进科学, 我们就来探究 SDUT OJ 评测姬的各种神奇之处.

<!--more-->

## 同样的代码在不同平台的表现

我们选用同样的代码, 在几个知名度较高的 OJ 平台上使用同样的编译器测试一下

```C++
#include <iostream>
using namespace std;

int main() {
    int a, b;
    cin >> a >> b;
    cout << a + b << endl;
    return 0;
}
```

选用的题目是 A + B Problem, 在选中的 OJ 上都是 1000 题目.

### POJ

| User | Problem | Result | Memory | Time | Language | Code Length |
| --- | --- | --- | --- | --- | --- | --- |
| MeiK | 1000 | Accepted | 720K | 0MS | G++ | 132B |

(因为贴图太麻烦了, 这里直接贴个表格)

### HDU OJ

| Judge Status | Pro.ID | Exe.Time | Exe.Memory | Code Len. | Language | Author |
| --- | --- | --- | --- | --- | --- | --- |
| Accepted | 1000 | 0MS | 1796K | 149 B | G++ | MeiK |

HDU OJ 上这个题目是多组输入的, 因此代码中处理了多组输入, 代码长度略长

### SDUT OJ

| Nick Name | Problem | Result | Time | Memory | Language | Code Len |
| --- | --- | --- | --- | --- | --- | --- |
| MeiK | 1000 | Accepted | 0 ms | 240 KiB | g++ | 142 B |

看出什么问题了没有? 虽然同样是 G++ 编译器, 但是在 SDUT OJ 上运行所使用的内存, 只有 POJ 的 1/3 , 只有 HDU OJ 的近 1/8 !

是因为 SDUT OJ 的评测姬有什么神奇的优化技术吗? 这里还是先讲解一下评测姬(Linux 平台下)是怎么获得程序运行的时间和内存的吧.

## 评测姬原理

### 时间

获得时间的方法各个 OJ 基本都没有区别, Linux 下有 `wait` 系列函数, 可以通过这个函数收集某个进程运行所占用的资源, 其中就有 CPU 时间.

CPU 时间是指进程实际占用 CPU 的时间, 系统中断和 I/O 阻塞的时间不计入其中, 同样的, 如果是多线程的程序的话, 那么最终统计到的也是所有线程的用时之和. 这个时间的值和墙上时钟没有什么必然的联系, 墙上时钟运行了 10S 的程序也有可能只占用了 1S 的 CPU 时间, 多线程的程序也有可能在 1S 内占用 3S CPU 时间. Linux 下有 `time` 命令, 可以轻松的计算某个程序的运行用时.

### 内存

和运行时间比起来, 获得运行占用内存是个相当不容易的事情. `wait` 函数虽然也可以获得程序占用内存, 但是有个弊端, 就是会将 `exec` 之前的内存也统计进来, 如果用户提交程序的内存占用还没有 `exec` 之前的内存大的话, 用户就会获得一个偏大的内存占用. (使用 `fork` 和 `exec`  几乎是 Linux 系统下执行新的程序的唯一方法, 所有基于 Linux 的评测姬都要通过这个方法来执行用户提交的代码)

那么有什么办法可以解决这个问题呢? HUST 开源的评测姬是这样做的: 评测姬监测用户程序, 用户程序每次执行系统调用的时候, 评测姬去 `/proc/${pid}/status` 中读取程序当前所占用的内存, 然后取一个最大值作为最终的内存占用. 

这样的好处是不会再把其他的内存给统计进去, 缺点就是: 每次用户程序执行系统调用都要中断用户程序, 导致整体运行时间变长, 并且由于过多的来回切换, CPU 时间也相应的变长了. 经统计, HUST 的评测姬比基于 `wait` 获得内存的评测姬评测出来的程序用时要大 30% 左右.

那还有没有其他的方法了? 我暂时是不知道还有什么能**比较准确**的测量程序运行占用内存的方法了. 我所知的大部分 OJ 都采用了 `wait` 的方法来获得内存占用, 但也有一些 OJ , 两种方法都没有使用, 比如 SDUTOJ ...

## 问题出现的原因

时间回到 2017 年, 大概是四五月份(也许吧), 当时评测姬被发现了一个 BUG: 任何一个程序运行超过 2S 就会超时, 当时学长们都去实习了, 我就借此机会研究了评测姬的源码. 经过一个多星期的研究, 我终于找出了 BUG 出现的原因, 可同时我也发现了一个神奇的操作 --- SDUT OJ 的评测姬, 是根据缺页中断次数判断使用内存的!

学过操作系统的同学们应该都知道, 缺页中断发生在系统缺页的时候, 也就是要访问的页还不在主存中, 这时候需要系统将数据置换到主存中才可以访问. 缺页中断的次数和程序实际使用的内存并没有必然的联系, 但是, SDUTOJ 的评测姬居然是用缺页中断的次数乘以每页内存的大小来计算内存使用的! 

在程序中用全局变量中申请的内存位于 *全局/静态区* , 这部分会在程序开始执行的时候与程序正文段一起被装载到内存中. 而使用 `new` 或者 `malloc` 申请的内存位于堆区, 在每次调用的时候都会实时申请. 这中间具体的分配过程与缺页情况我并不是很清楚, 但是, 在 SDUTOJ 评测姬的这种计算方式下, 使用同样多的内存, 如果使用全局变量的程序被计算出 1000KIB 内存占用, 那么使用 `new` 的程序将会被计算出 150000KIB 甚至更多. 没错, 是一千和十五万的、 超过 100 倍的差距.

## 为何我没有修复这个问题

我当时注意到这个问题的时候, 想过要修复它. 但是考虑到了以下几点:

- OJ 中有大量限制内存严格的题目, 比如有个题目通过内存限制只能使用某种数据结构(几百 KIB 的内存限制). 而如果修复了这个问题的话, 那么这些题目将都无法通过, 因为任何一个提交的内存占用都将超过 1000KIB.
- 当时我并不知道这个问题会导致什么样的后果, 我认为即使和其他 OJ 不同, 但只要能拿到一个相对正确的值即可. 比如两个代码, 在 POJ 上占用 2000KIB 和 4000KIB, 而在 SDUTOJ 上分别占用 300KIB 和 600KIB, 那么也是可以接受的.
- 还有一点, 就是我当时确实认为这个是学长或者原作者有意为之(原版 SDUTOJ 基于 Lo_runner, 我不知道是 Lo_runner 就做了这个设定, 还是学长在修改的时候做的). 当年写评测姬的学长已经联系不到了, 而这个评测姬已经*稳定*运行了很多年, 出于一些稳定性的原因, 我也不能去修复它.

## 现在的解决方案

如果我现在修复了这个问题, 那么后续也会有大量的问题. 之前的题目是基于错误的内存测量方式而给定的内存限制, 这些题目全部需要进行修改, 是一个不小的工程.

当然我们也并不会回避问题, 我们将在商讨后给定一个解决方案, 在修复这个 BUG 之前, 如果碰到了这种问题的话, 权宜之计就是申请全局变量来自己维持一个内存池, 这样就可以在内存方面通过限制.