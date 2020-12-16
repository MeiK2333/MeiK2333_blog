---
title: 《APUE》习题：通过 POSIX 信号量函数实现父子进程的交替执行
date: 2018-07-17 16:01:09
tags: [C 语言, APUE]
---

通过 POSIX 信号量函数实现父子进程交替执行.
<!--more-->

与 XSI 信号量函数不同, POSIX 信号量函数更加易用, 也更加安全. 但是 POSIX 信号量只能有 +1 和 -1 操作, 因此[之前使用](/2018/07/17/《APUE》习题：通过-XSI-信号量函数实现父子进程的交替执行/)的交替增加的方法也就变得不可行了.

但是, 我们有另一种更加优雅的方式来实现父子进程交替执行, 之前使用的方法其实有很大的隐患: sem_op 字段是 short 类型的, 如果交替次数过多, 很容易溢出. 这次我们使用两个 POSIX 信号量来更加优雅和安全的实现同样的功能.

原理很简单: 申请两个信号量, 初始都为 0 , 父进程执行操作后, 释放 2 , 申请 1; 子进程首先申请 2 , 执行操作后释放 1. 这样, 父进程操作后, 因为没有 2 资源, 就需要等待子进程操作, 子进程释放 1 之后, 父进程继续执行.

代码如下:

```C
#include <fcntl.h>
#include <semaphore.h>
#include <sys/mman.h>
#include "apue.h"

#define NLOOPS 1000
#define SIZE sizeof(long)

static int update(long *ptr) { return ((*ptr)++); }

int main() {
    int i, counter;
    pid_t pid;
    void *area;

    sem_t *st1, *st2;

    st1 = sem_open("/test1", O_CREAT, 0600, 0);
    st2 = sem_open("/test2", O_CREAT, 0600, 0);
    sem_unlink("/test1");
    sem_unlink("/test2");

    if ((area = mmap(0, SIZE, PROT_READ | PROT_WRITE, MAP_ANON | MAP_SHARED, -1,
                     0)) == MAP_FAILED) {
        err_sys("mmap error");
    }

    if ((pid = fork()) < 0) {
        err_sys("fork error");
    } else if (pid > 0) {
        for (i = 0; i < NLOOPS; i += 2) {
            if ((counter = update((long *)area)) != i) {
                err_quit("parent: expected %d, got %d", i, counter);
            }
            printf("update %d from parent\n", i);

            sem_post(st2);
            sem_wait(st1);
        }
    } else {
        for (i = 1; i < NLOOPS + 1; i += 2) {
            sem_wait(st2);

            if ((counter = update((long *)area)) != i) {
                err_quit("child:expected %d, got %d", i, counter);
            }
            printf("update %d from child\n", i);

            sem_post(st1);
        }
    }
    exit(0);
}
```