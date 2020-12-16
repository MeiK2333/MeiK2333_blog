---
title: 如何写一个评测姬 - 第五篇
date: 2019-07-08 20:44:09
tags: [OnlineJudge, 评测姬]
---

从零开始实现一个 OJ 评测姬。

<!--more-->

获取指定子进程的资源占用，是一个常见的需求，Linux 也提供了相应的实现方法。OJ 也需要使用这些方法来评判用户程序的时间与空间复杂度。

## 获取程序运行时间

获取程序运行了多久，其实很简单：在程序运行前后分别计时，然后相减即可。

```c
struct timeval start, end;
pid_t pid;
int time_used;  // 运行耗时

gettimeofday(&start, NULL);  // 运行前计时
if ((pid = fork()) == 0) {  // child
    // exec 执行子进程
}
// wait 等待子进程结束
gettimeofday(&end, NULL);  // 运行后计时
time_used =
    (int)(end.tv_sec * 1000 + end.tv_usec / 1000 - start.tv_sec * 1000 -
            start.tv_usec / 1000);
```

## 获取 CPU 时间

上面的方法乍一想好像没什么问题，但仔细一想，Linux 是分时处理系统，我们并不能保证耗时相同的两个程序进行了同样多的计算。如果一个时间复杂度较高的程序在系统负载低的时候运行，而时间复杂度较低的程序在系统满载的时候运行，那么有可能时间复杂度高的程序运行耗时反而更少，这样肯定是不公平的。

这就引出了另一个概念：[CPU time](https://en.wikipedia.org/wiki/CPU_time)。CPU time 是程序运行实际占用 CPU 的时间，如果是多线程的程序的话，则是多个线程的累加耗时。如果使用 CPU 时间作为程序运行耗时的话，就可以公平的评比用户程序了。

在 Linux 中可以通过 `rusage` 结构体与 `wait4` 系统调用来获取子进程的 CPU 时间。

```c
int status, time_used;  // CPU 时间
struct rusage ru;

wait4(pid, &status, 0, &ru);
time_used =
    ru.ru_utime.tv_sec * 1000 + ru.ru_utime.tv_usec / 1000 +
    ru.ru_stime.tv_sec * 1000 + ru.ru_stime.tv_usec / 1000;
```

## 获取内存占用

同样的， `rusage` 结构体中也包含了进程运行占用的内存，我们可以使用与 CPU 时间一样的方法来获得这个值。

```c
int status, memory_used;  // 运行过程中的最大内存占用
struct rusage ru;

wait4(pid, &status, 0, &ru);
memory_used = ru.ru_maxrss;
```

不过，如果你使用这种方法来测量进程内存的话，可能会出现这种情况：一个输出 `Hello World!` 的简单 C 语言程序与一个申请了长度几千的数组的 C++ 程序，测量出来的内存是一样多的。

这种情况并非 BUG，而是预期中的行为。在 <https://linux.die.net/man/2/getrusage> 中明确地指出： `Resource usage metrics are preserved across an execve(2)`。如果你要测量的程序的内存占用还没有你 `exec` 之前的子进程占用的内存大的话， `ru_maxrss` 实际是你 `exec` 之前的进程内存占用。

虽然这不是 BUG，但是对于较少内存占用的程序来说，确实会造成测量不精准的问题。针对这个问题，有三个解决方案：

1. 置之不理，对于较少内存占用的程序不做精确测量
2. 尽量减少 `exec` 之前的内存占用，比如 [QDUOJ](https://github.com/QingdaoU/Judger)，他们使用 C 语言写了一个很小的评测核心，其功能基本只有执行子进程，因此其内存占用很低，可以降低测量不精准的上限。虽然这个问题依旧存在，可对于 ACM 来说，精度已经足够了，如果有题目在这种尺度进行卡内存的话，就容易出现“卡常毒瘤题”
3. 读取 `/proc/<pid>/status` 的内容，这个文件中有进程当前的内存占用，只要循环读取这个文件，解析出内存占用即可。缺点也很明显：因为要循环读取文件，效率较低

其实还有一种方案：[评测鸭](https://duck.ac/)，他们自己造了个操作系统……这就不在讨论范围内了。我们在下面稍微讨论一下如何使用方法三测量较为精确的内存占用。

## 获取精确的内存占用

循环读取 `/proc/<pid>/status` 文件内容，实现内存占用测量。那么问题来了，我们以什么频率去读取呢？

Linux 还有个系统调用 `ptrace`，可以捕获被调试进程的每个系统调用，考虑到进程申请与释放内存都会使用系统调用，因此，我们仅在进程进行系统调用时进行读取即可。此处仅给出思路，就不给出具体代码了。

## 获取子进程退出状态

依旧是我们的 `wait4`(`waitpid`)，文档 <https://linux.die.net/man/2/waitpid> 中有对它的明确定义。配合使用 `WEXITSTATUS` 等宏，就可以获得子进程的退出状态。

## 总结

通过这章内容可以看出来，想要获取子进程的资源占用，其实基本只使用 `wait4` 就够了……本章的示例代码可以在 <https://github.com/MeiK2333/ZeroJudger/tree/master/getusage> 中找到，这个代码简单的实现了获取子进程运行占用资源与退出状态的功能。

加上这篇文章，如何写一个评测姬系列已经五篇了，虽然中间隔了近三个月……下一篇里，我们将运用我们这在这五篇中介绍过的内容，实现一个功能完整的简单评测姬。
