---
title: 关于 mmap 映射的内存的释放问题
date: 2018-08-16 20:28:25
tags: [C 语言]
---

```C
#include <sys/mman.h>
void *mmap(void *addr, size_t len, int prot, int flag, int fd, off_t off);
// 若执行成功， 则返回映射区的起始地址；若出错，则返回 `MAP_FAILED` 。
```

<!--more-->

其参数含义与可选选项可以参照[这里](http://man7.org/linux/man-pages/man2/mmap.2.html)。

`mmap` 映射的空间应该使用 [`munmap`](http://man7.org/linux/man-pages/man2/munmap.2.html) 释放， 而非 `free` ，使用 `munmap` 后其空间将被自动释放。而如果使用 `free` 释放的话，将会导致程序卡死。