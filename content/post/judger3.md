---
title: 如何写一个评测姬 - 第三篇
date: 2019-05-05 15:22:54
tags: [OnlineJudge, 评测姬]
---

从零开始实现一个 OJ 评测姬。

<!--more-->

这一篇我们讲一下如何执行一个程序、重定向其输入输出。并实现一个最简单的评测姬。

## 执行一个新的进程

Linux 下可以使用 [`system`](http://man7.org/linux/man-pages/man3/system.3.html) 函数来运行一个进程，用法如下：

```c
#include <stdlib.h>
int main() {
    system("echo Hello World");
    return 0;
}
```

但 `system` 并不能满足我们的需求，我们需要在进程运行前进行一些自定义的操作，比如重定向流等。因此我们使用层级更低的 `fork` 来创建一个新的进程，使用 `exec` 函数来执行我们指定的进程（`system` 的实现也是 `fork` + `execl`）。在 `fork` 之后重定向流，然后 `exec` 来执行用户提交的程序，就可以实现我们的需求了。

使用 `fork` + `exec` 来改写上面的程序，就是这样：

```c
#include <unistd.h>
int main() {
    int pid = fork();
    if (pid == 0) {
        // 重定向流或其他操作
        execl("/bin/sh", "sh", "-c", "echo Hello World", (char *) NULL);
    }
    return 0;
}
```

虽然代码量比之前要多，但是我们可以在执行用户程序之前进行自定义的操作。同理，虽然[有研究证明](https://www.microsoft.com/en-us/research/publication/a-fork-in-the-road/) 使用 `fork` + `exec` 来执行新的进程的性能远低于 `posix_spawn`，但是我们还是只能选择这个方案。

## 重定向流

使用过 Linux 的同学可能对命令行重定向输入输出已经很熟悉了，`bash` 使用大于和小于的符号来表示流的重定向：

```bash
$ echo Hello World > hello.txt
$ cat hello.txt
Hello World
```

其中的 `>` 的用处就是将程序的标准输出重定向到文件中，本来应该被打印在终端上的内容被写入了文件。同理，`<` 的作用是可以使用文件内的内容替代标准输入来传给程序。

这样做有什么意义呢？因为我们的题目数据都是文件形式，我们的答案对比函数也是基于文件的。有一些 OJ 是要求用户自行从文件中读取输入数据，并将答案写入到文件的，但大部分 OJ 都允许用户提交直接读标准输出 `stdin`、直接写标准输出 `stdout` 的代码。因此，从标准流到文件的转换就需要我们来进行处理。

Linux 可以使用 [`dup2`](http://man7.org/linux/man-pages/man2/dup2.2.html) 函数来重定向流，使得文件流连接标准流，这样进程就可以直接从标准输入输出流中读取写入文件了。

```c
#include <unistd.h>
#include <stdio.h>
int main() {
    FILE *out_file = fopen("output.txt", "w");
    dup2(fileno(out_file), STDOUT_FILENO);
    printf("Hello World");
    return 0;
}
```

执行代码：

```bash
$ ./a.out
$ cat output.txt 
Hello World
```

可以看到，`printf` 并没有把 `Hello World` 打印到屏幕上，而是将其写入了文件。

## 结合

运行新的进程和重定向流的方法都有了，我们可以尝试结合一下这两者。为了方便起见，这里我们先不考虑编译的问题，直接使用 `system` 函数。结合完成的代码大概长这样：

```c
void BaseRun(char *code_file_name, char *input_file_name) {
    char cmd[1024];
    strcpy(cmd, "gcc ");
    strcpy(cmd + 4, code_file_name);
    system(cmd);
    int pid = fork();
    if (pid == 0) {
        FILE *input_file = fopen(input_file_name, "r");
        dup2(fileno(input_file), STDIN_FILENO);
        execl("a.out", (char *) NULL);
    }
}
```

完整代码以及对 `A + B Problem` 的测试在 GitHub 上：<https://github.com/MeiK2333/ZeroJudger/tree/master/baserun>。

## 总结

本篇讲解了实现一个评测姬最重要的核心功能：编译、执行用户代码。配合上一篇的评测函数，现在我们已经可以实现一个最基本的评测姬了。但这样还不够，评测姬会执行用户提交的代码，用户是有可能会提交恶意的代码的，我们需要某些手段来保证评测姬的安全。下一篇我们就来看看如何保证评测姬的安全。
