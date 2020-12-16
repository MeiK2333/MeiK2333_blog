---
title: 《APUE》习题：通过 XSI 信号量函数实现父子进程的交替执行
date: 2018-07-17 15:55:06
tags: [C 语言, APUE]
---

通过 XSI 信号量函数实现父子进程交替执行.
<!--more-->

XSI 信号量函数可以指定添加或者消耗的资源个数, 通过一个 XSI 信号量如何实现父子进程交替执行?

因为添加与减少的个数都可以自由指定, 初始有 1 个资源, 可以父进程消耗 1 个资源, 释放 2 个资源, 然后再申请 3 个, 这样父进程无法通过现有资源执行; 而子进程初始申请消耗 2 个资源, 释放 3 个资源, 这样子进程初始无法执行, 但是父进程释放 2 个资源后就可以执行, 而子进程释放的 3 个进程将用于父进程执行. 以此类推, 可以实现父子进程交替执行.

代码如下:

```C
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/sem.h>
#include "apue.h"

#define NLOOPS 1000
#define SIZE sizeof(long)

static int update(long *ptr) { return ((*ptr)++); }

int main() {
    int i, counter;
    pid_t pid;
    void *area;
    int semid;
    struct sembuf sbf[1];
    semid = semget(0, 1, IPC_CREAT | 0666);

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

            sbf[0].sem_op = i + 1;
            semop(semid, sbf, 1);
            sbf[0].sem_op = -i - 2;
            semop(semid, sbf, 1);
        }
    } else {
        for (i = 1; i < NLOOPS + 1; i += 2) {
            sbf[0].sem_op = -i;
            semop(semid, sbf, 1);

            if ((counter = update((long *)area)) != i) {
                err_quit("child: expected %d, got %d", i, counter);
            }
            printf("update %d from child\n", i);

            sbf[0].sem_op = i + 1;
            semop(semid, sbf, 1);
        }
    }
    exit(0);
}

```