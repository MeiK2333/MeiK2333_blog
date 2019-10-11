---
title: 如何写一个评测姬 - 第四篇
date: 2019-05-08 15:05:04
tags: [OnlineJudge, 评测姬]
---

从零开始实现一个 OJ 评测姬。

<!--more-->

本篇讲一下如何防止评测姬因用户程序的异常而崩溃。首先，我们来看看用户程序都会出现什么异常，然后再来考虑如何解决这些异常。

## 解决过多资源占用

用户可能因为失误而提交了占用过多资源的代码，比如死循环、占用过多内存、输出过多等。为了解决这些问题，我们需要限制用户程序的时间、内存以及输出数据的大小。

Linux 有系统函数可以解决这个问题：[`setrlimit`](https://linux.die.net/man/3/setrlimit)。这个函数可以限制程序的 CPU 时间、内存占用、输出数据等，用法大致如下：

```c
#include <sys/resource.h>
int main() {
    struct rlimit rl;
    rl.rlim_cur = rl.rlim_max = 1;
    setrlimit(RLIMIT_CPU, &rl);
    while (1) {
    }
}
```

执行：

```bash
$ gcc main.c
$ time ./a.out 
[2]    3057 cpu limit exceeded  ./a.out
./a.out  0.98s user 0.02s system 96% cpu 1.034 total
```

可以看到，虽然我们在代码中写了个死循环，但是这个程序只运行一秒就结束了。`setrlimit` 可以对进程添加资源限制，并在超出限制时触发信号，父进程可以捕获这个信号。同时 `exec` 会继承这些限制，因此可以使用与流重定向类似的方法来添加资源限制。

有关限制的实际代码，可以查看 <https://github.com/MeiK2333/ZeroJudger/blob/master/limit/main.c>。

## 解决恶意攻击

不同于用户不小心提交的占用资源过多的代码，有些同学会故意提交一些恶意攻击评测姬的代码，俗称“卡评测”。对于这些恶意攻击，我们也需要做好防备。

简单起见，我们的评测姬不对恶意攻击做防范，仅对用户无意间造成的资源过度占用做限制。

常用的恶意攻击的方式我之前写过博客，后来我注销了 CSDN 账号，博客也没了，这里就重新写一遍吧。

### include 攻击

利用 OJ 会返回编译错误的特点，使用 C/C++ 语言使用 `#include` 来导入一个敏感的文件，然后通过编译错误的信息来获得其中敏感的数据。

```bash
$ echo 'password' > password.txt
$ echo '#include <password.txt>' > main.c 
$ gcc main.c 
main.c:1:10: error: 'password.txt' file not found with <angled> include; use "quotes" instead
#include <password.txt>
         ^~~~~~~~~~~~~~
         "password.txt"
In file included from main.c:1:
./password.txt:1:1: error: unknown type name 'password'
password
^
main.c:1:24: error: expected identifier or '('
#include <password.txt>
                       ^
3 errors generated.
```

解决方案为配置好评测用户权限，使用低权限的用户进行评测，以防止此类攻击。同时可以考虑使用 Docker 等进行评测，Docker 环境内不放任何敏感的数据。

### 编译超时 && 编译产生巨大文件 && 编译产生巨量错误信息

- 编译超时  
```c
#include </dev/random>或者#include </dev/tty> //(Linux)
#include <CON> //(Windows)
```
- 编译产生巨大文件  
```c
main[-1u]={1};
```
- 编译产生巨量错误信息

这几种攻击方式并不能让攻击者获得什么数据或权限，只是单纯的会卡住评测。但有人就是这么无聊的……

解决方案也很简单，就是像限制用户程序资源占用一样限制编译时的资源占用。

### shell 攻击 && fork 炸弹 && 其他系统调用攻击 && 网络模块攻击

shell 攻击：

```c
system("reboot");
```

`fork` 炸弹：

```c
while (1) {
    fork();
}
```

网络模块攻击：如果评测姬没有限制联网的话，攻击的程序可以通过网络做到很多的攻击方式，程序可能会收集很多运行时评测姬和服务器的信息，然后提交到某处。

这几种攻击方式，如果只是简单的禁用 `system` 和 `fork` 等这些函数是没有用的，因为 C/C++ 的宏可以轻易的绕过这些限制。可以考虑在系统调用的层面进行限制，大多开源的评测机都是通过限制系统调用来保证评测安全的。

限制系统调用，可以使用 `ptrace` 来自行限制，也可以使用开源库 [seccomp](https://github.com/seccomp/libseccomp) 来进行限制。

## 总结

本篇我们介绍了常见的攻击手段与防范手段，这样我们的评测姬就会有一定的安全性保障。下一篇我们将介绍一下如何获取用户程序运行占用的资源。
