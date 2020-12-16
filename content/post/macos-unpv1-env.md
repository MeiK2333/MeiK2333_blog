---
title: MacOS 搭建 UNPv1 编译环境
date: 2018-08-21 20:49:13
tags: [UNPv1, 网络编程, C 语言]
---

为了能够编译运行 《UNIX 网络编程》（UNPv1）上的示例代码，需要编译 `libunp.a` 库文件，然后才可以正常编译书中的代码。

关于如何编译《UNIX 环境高级编程》（APUE）的示例代码，可以参照[这里](https://github.com/MeiK2333/apue/tree/master/apue.3e)。

<!--more-->

## 准备工作

首先需要去[官网](http://www.unpbook.com/src.html)下载所需的头文件，下载之后将其解压。

切换文件夹

```shell
cd unpv13e
vim README
```

其中有 `README` 文件指示了如何编译这些文件，但是在 MacOS 下直接按照其步骤会出现一些问题。

## 开始编译

```shell
./configure
cd lib
make

cd ../libfree
make
```

此处出现报错

```shell
inet_ntop.c:56:1: error: conflicting types for 'inet_ntop'
inet_ntop(af, src, dst, size)
^
/usr/include/arpa/inet.h:77:13: note: previous declaration is here
const char      *inet_ntop(int, const void *, char *, socklen_t);
                 ^
1 error generated.
```

找到 `inet_ntop.c` 文件，把 `#include <arpa/inet.h>` 这行注释掉，再次执行 `make`。

## 测试

```shell
cd ../intro    
make daytimetcpcli
```

然后执行编译好的 `daytimetcpcli` 程序，如果你之前没有完成书上 1.5 的“一个简单的时间获取服务器程序”，那么你这里按照 `README` 中的方法是不行的。（事实上你肯定没有这个程序，因为这个程序需要 `unp.h` 头文件，而你现在正在编译它...）

```shell
./daytimetcpcli 127.0.0.1
connect error: Connection refused
```

这时候我们可以在 [NIST Internet Time Servers](https://tf.nist.gov/tf-cgi/servers.cgi) 来找一个可用的时间服务器地址进行测试

```shell
./daytimetcpcli 129.6.15.28

58351 18-08-21 12:38:39 50 0 0 724.4 UTC(NIST) *
```

## 使用

虽然 `libunp.a` 已经成功编译出来了，但是如果每次编写程序的时候都要将这个文件添加到查找目录里未免过于繁琐。我们可以将静态链接库添加到系统目录中。

回到主目录，首先修改 `./lib/unp.h`，将其中的 `#include "../config.h"` 修改为 `#include "config.h"`，然后拷贝文件。

```shell
sudo cp ./config.h  /usr/local/include
sudo cp ./lib/unp.h  /usr/local/include

sudo cp ./libunp.a  /usr/local/lib
```

我们用书中 3.4 的确定主机字节序的程序来测试一下。

```C
// byteorder.c
#include "unp.h"

int main(int argc, char *argv[]) {
    union {
        short s;
        char c[sizeof(short)];
    } un;
    un.s = 0x0102;
    printf("%s: ", CPU_VENDOR_OS);
    if (sizeof(short) == 2) {
        if (un.c[0] == 1 && un.c[1] == 2) {
            printf("big-endian\n");
        } else if (un.c[0] == 2 && un.c[1] == 1) {
            printf("little-endian\n");
        } else {
            printf("unknown\n");
        }
    } else {
        printf("sizeof(short) = %d\n", sizeof(short));
    }
    exit(0);
}
```

编译时指定库文件链接 `-lunp`

```shell
gcc byteorder.c -lunp -o byteorder
./byteorder 
i386-apple-darwin17.7.0: little-endian
```

运行成功！

另附一份 CLion 的 CMakeList.txt：

```CMake
cmake_minimum_required(VERSION 3.9)
project(UNPv1 C)

include_directories(/usr/local/include)
set(CMAKE_C_STANDARD 11)

add_executable(UNPv1 main.c)

target_link_libraries(UNPv1 unp)
```

## 参考

- [UNIX网络编程（第3版）环境搭建——使用MAC OSX10.10 - 简书](https://www.jianshu.com/p/7e395e4f8515)
- [《UNIX网络编程(第3版)》unp.h等源码文件的编译安装 - 博客园](https://www.cnblogs.com/52php/p/5684487.html)
- [MeiK 的 GitHub](https://github.com/MeiK2333/apue/tree/master/apue.3e)