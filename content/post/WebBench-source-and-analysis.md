---
title: 'WebBench 源码分析 [不推荐阅读]'
date: 2018-07-19 21:39:23
tags: [C 语言, 网络编程]
---

在学习的过程中, 总是要阅读很多别人的代码的. 很多优秀的开源代码能够让我们受益匪浅, 也有一些代码, 看了之后只会让我们感觉浪费了时间......

Webbench是Radim Kolar在1997年写的一个在linux下使用的非常简单的网站压测工具。它使用fork()模拟多个客户端同时访问我们设定的URL，测试网站在压力下工作的性能，最多可以模拟3万个并发连接去测试网站的负载能力。
<!--more-->

## 问题
看完这个项目之后, 我注意到了这个项目的一些问题

- 滥用全局变量
- 一个函数将一个全局变量作为函数参数传递给了另一个函数
- 代码风格不一致, 同样的功能(获取指定字符串位置)用了好几种不同的办法, 还混写在了一起
- `strcat(str1 + strlen(str1), str2)` 这种神奇的操作
- 可能会创建多个子进程(文档描述中说明可以上万个), 却只判断了最后一个是否成功
- 使用 `sleep` 来调度父子进程的先后关系
- 管道没有关闭不需要的一端
- 奇怪的变量复用, 比如在子进程中用 `pid` 又去表示了 `fscanf` 的返回值
- ......

## 建议
不要看这个项目! 不要看这个项目! 不要看这个项目!

除非你想要从中获得什么反面的教学用例

如果你想要通过一个代码并不长、并不复杂的项目来学习一个 HTTP 服务相关的话, 那么我推荐 [Tinyhttpd 源码及分析](/2018/06/29/Tinyhttpd-源码及分析/). 虽然那个项目也有一些问题, 只能作为一个玩具使用, 但是对于学习来说是有一定意义的.

## 源代码
我居然用了一个多小时的时间看完了他的源代码并且加上了注释......

### socket.c
```C
/* $Id: socket.c 1.1 1995/01/01 07:11:14 cthuang Exp $
 *
 * This module has been modified by Radim Kolar for OS/2 emx
 */

/***********************************************************************
  module:       socket.c
  program:      popclient
  SCCS ID:      @(#)socket.c    1.5  4/1/94
  programmer:   Virginia Tech Computing Center
  compiler:     DEC RISC C compiler (Ultrix 4.1)
  environment:  DEC Ultrix 4.3
  description:  UNIX sockets code.
 ***********************************************************************/

#include <arpa/inet.h>
#include <fcntl.h>
#include <netdb.h>
#include <netinet/in.h>
#include <stdarg.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

int Socket(const char *host, int clientPort) {
    int sock;
    unsigned long inaddr;
    struct sockaddr_in ad;
    struct hostent *hp;

    memset(&ad, 0, sizeof(ad));
    ad.sin_family = AF_INET;

    /* 尝试以 IP 形式解析 */
    inaddr = inet_addr(host);
    if (inaddr != INADDR_NONE)
        memcpy(&ad.sin_addr, &inaddr, sizeof(inaddr));
    else { /* 如果解析失败, 则尝试通过 DNS 进行转换 */
        /* gethostbyname 是过时的函数, 应当使用 getaddrinfo 替换掉它 */
        hp = gethostbyname(host);
        if (hp == NULL) return -1;
        memcpy(&ad.sin_addr, hp->h_addr, hp->h_length);
    }
    ad.sin_port = htons(clientPort);

    /* 创建 socket 连接 */
    sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) return sock;
    if (connect(sock, (struct sockaddr *)&ad, sizeof(ad)) < 0) return -1;
    return sock;
}

```

### webbench.c
```C
/*
 * (C) Radim Kolar 1997-2004
 * This is free software, see GNU Public License version 2 for
 * details.
 *
 * Simple forking WWW Server benchmark:
 *
 * Usage:
 *   webbench --help
 *
 * Return codes:
 *    0 - sucess
 *    1 - benchmark failed (server is not on-line)
 *    2 - bad param
 *    3 - internal error, fork failed
 *
 */

#include <getopt.h>
#include <rpc/types.h>
#include <signal.h>
#include <strings.h>
#include <sys/param.h>
#include <time.h>
#include <unistd.h>
#include "socket.c"

/* values */
volatile int timerexpired = 0;
int speed = 0;
int failed = 0;
int bytes = 0;

/* globals */
int http10 = 1; /* 0 - http/0.9, 1 - http/1.0, 2 - http/1.1 */
/* Allow: GET, HEAD, OPTIONS, TRACE */
#define METHOD_GET 0
#define METHOD_HEAD 1
#define METHOD_OPTIONS 2
#define METHOD_TRACE 3
#define PROGRAM_VERSION "1.5"
int method = METHOD_GET;
int clients = 1;
int force = 0;
int force_reload = 0;
int proxyport = 80;
char *proxyhost = NULL;
int benchtime = 30;

/* internal */
int mypipe[2];
char host[MAXHOSTNAMELEN];
#define REQUEST_SIZE 2048
char request[REQUEST_SIZE];

static const struct option long_options[] = {
    {"force", no_argument, &force, 1},
    {"reload", no_argument, &force_reload, 1},
    {"time", required_argument, NULL, 't'},
    {"help", no_argument, NULL, '?'},
    {"http09", no_argument, NULL, '9'},
    {"http10", no_argument, NULL, '1'},
    {"http11", no_argument, NULL, '2'},
    /* 绑定命令行参数与变量 */
    {"get", no_argument, &method, METHOD_GET},
    {"head", no_argument, &method, METHOD_HEAD},
    {"options", no_argument, &method, METHOD_OPTIONS},
    {"trace", no_argument, &method, METHOD_TRACE},
    {"version", no_argument, NULL, 'V'},
    {"proxy", required_argument, NULL, 'p'},
    {"clients", required_argument, NULL, 'c'},
    {NULL, 0, NULL, 0}};

/* prototypes */
static void benchcore(const char *host, const int port, const char *request);
static int bench(void);
static void build_request(const char *url);

static void alarm_handler(int signal) { timerexpired = 1; }

static void usage(void) {
    fprintf(
        stderr,
        "webbench [option]... URL\n"
        "  -f|--force               Don't wait for reply from server.\n"
        "  -r|--reload              Send reload request - Pragma: no-cache.\n"
        "  -t|--time <sec>          Run benchmark for <sec> seconds. Default "
        "30.\n"
        "  -p|--proxy <server:port> Use proxy server for request.\n"
        "  -c|--clients <n>         Run <n> HTTP clients at once. Default "
        "one.\n"
        "  -9|--http09              Use HTTP/0.9 style requests.\n"
        "  -1|--http10              Use HTTP/1.0 protocol.\n"
        "  -2|--http11              Use HTTP/1.1 protocol.\n"
        "  --get                    Use GET request method.\n"
        "  --head                   Use HEAD request method.\n"
        "  --options                Use OPTIONS request method.\n"
        "  --trace                  Use TRACE request method.\n"
        "  -?|-h|--help             This information.\n"
        "  -V|--version             Display program version.\n");
}

int main(int argc, char *argv[]) {
    /* 读取命令行参数 */
    int opt = 0;
    int options_index = 0;
    char *tmp = NULL;

    if (argc == 1) {
        usage();
        return 2;
    }

    while ((opt = getopt_long(argc, argv, "912Vfrt:p:c:?h", long_options,
                              &options_index)) != EOF) {
        switch (opt) {
            case 0:
                break;
            case 'f':
                /* 设置 force 模式 */
                force = 1;
                break;
            case 'r':
                /* 设置 reload 模式 */
                force_reload = 1;
                break;
            case '9':
                /* HTTP/0.9 */
                http10 = 0;
                break;
            case '1':
                /* HTTP/1.0 */
                http10 = 1;
                break;
            case '2':
                /* HTTP/1.1 */
                http10 = 2;
                break;
            case 'V':
                /* 打印版本信息并退出 */
                printf(PROGRAM_VERSION "\n");
                exit(0);
            case 't':
                /* 设置运行时间 */
                benchtime = atoi(optarg);
                break;
            case 'p':
                /* proxy server parsing server:port */
                /**
                 * 解析代理设置, 使用 ":" 分割代理字符串
                 * eg: -p 127.0.0.1:1080
                 * proxyhost(char *): "127.0.0.1"
                 * proxyport(int): 1080
                 * */
                tmp = strrchr(optarg, ':');
                proxyhost = optarg;
                if (tmp == NULL) {
                    break;
                }
                /* 判断是否提供 hostname */
                if (tmp == optarg) {
                    fprintf(stderr,
                            "Error in option --proxy %s: Missing hostname.\n",
                            optarg);
                    return 2;
                }
                /* 判断是否提供 port */
                if (tmp == optarg + strlen(optarg) - 1) {
                    fprintf(
                        stderr,
                        "Error in option --proxy %s Port number is missing.\n",
                        optarg);
                    return 2;
                }
                *tmp = '\0';
                proxyport = atoi(tmp + 1);
                break;
            case ':':
            case 'h':
            case '?':
                /* 打印帮助 */
                usage();
                return 2;
                break;
            case 'c':
                /* 设置客户端个数 */
                clients = atoi(optarg);
                break;
        }
    }

    /**
     * 当所有带选项的参数都已读取结束, optind
     * 应当指向第一个不包含选项的命令行参数 若 optind == argc,
     * 则代表没有不包含选项的命令行参数
     * */
    if (optind == argc) {
        fprintf(stderr, "webbench: Missing URL!\n");
        usage();
        return 2;
    }

    /* 设置 clients 和 benchtime 的默认值 */
    if (clients == 0) clients = 1;
    if (benchtime == 0) benchtime = 30;

    /* Copyright */
    /* 打印 Copyright */
    fprintf(stderr,
            "Webbench - Simple Web Benchmark " PROGRAM_VERSION
            "\n"
            "Copyright (c) Radim Kolar 1997-2004, GPL Open Source Software.\n");

    build_request(argv[optind]);

    // print request info ,do it in function build_request
    /*printf("Benchmarking: ");

    switch(method)
    {
        case METHOD_GET:
        default:
        printf("GET");break;
        case METHOD_OPTIONS:
        printf("OPTIONS");break;
        case METHOD_HEAD:
        printf("HEAD");break;
        case METHOD_TRACE:
        printf("TRACE");break;
    }

    printf(" %s",argv[optind]);

    switch(http10)
    {
        case 0: printf(" (using HTTP/0.9)");break;
        case 2: printf(" (using HTTP/1.1)");break;
    }

    printf("\n");
    */

    /* 打印配置 */
    printf("Runing info: ");

    if (clients == 1)
        printf("1 client");
    else
        printf("%d clients", clients);

    printf(", running %d sec", benchtime);

    if (force) printf(", early socket close");
    if (proxyhost != NULL)
        printf(", via proxy server %s:%d", proxyhost, proxyport);
    if (force_reload) printf(", forcing reload");

    printf(".\n");

    /* 开始执行 */
    return bench();
}

void build_request(const char *url) {
    char tmp[10];
    int i;

    // bzero(host,MAXHOSTNAMELEN);
    // bzero(request,REQUEST_SIZE);
    memset(host, 0, MAXHOSTNAMELEN);
    memset(request, 0, REQUEST_SIZE);

    /* 根据 reload 和 method 等参数调整 HTTP */
    if (force_reload && proxyhost != NULL && http10 < 1) http10 = 1;
    if (method == METHOD_HEAD && http10 < 1) http10 = 1;
    if (method == METHOD_OPTIONS && http10 < 2) http10 = 2;
    if (method == METHOD_TRACE && http10 < 2) http10 = 2;

    switch (method) {
        default:
        case METHOD_GET:
            strcpy(request, "GET");
            break;
        case METHOD_HEAD:
            strcpy(request, "HEAD");
            break;
        case METHOD_OPTIONS:
            strcpy(request, "OPTIONS");
            break;
        case METHOD_TRACE:
            strcpy(request, "TRACE");
            break;
    }

    strcat(request, " ");

    /* 检查 URL 是否合法 */
    if (NULL == strstr(url, "://")) {
        fprintf(stderr, "\n%s: is not a valid URL.\n", url);
        exit(2);
    }
    /* 检查 URL 长度 */
    if (strlen(url) > 1500) {
        fprintf(stderr, "URL is too long.\n");
        exit(2);
    }
    /* 忽略大小写对比两个字符串的前 n 个字符 */
    /* 仅支持 HTTP 协议 */
    if (0 != strncasecmp("http://", url, 7)) {
        fprintf(stderr,
                "\nOnly HTTP protocol is directly supported, set --proxy for "
                "others.\n");
        exit(2);
    }

    /* protocol/host delimiter */
    /* 获得去掉协议后的 URL */
    i = strstr(url, "://") - url + 3;

    /* 判断 '/' 是否在 URL 中出现过 */
    if (strchr(url + i, '/') == NULL) {
        fprintf(stderr,
                "\nInvalid URL syntax - hostname don't ends with '/'.\n");
        exit(2);
    }

    /* url + i 指向 URL 去掉协议后的开头 */
    if (proxyhost == NULL) {
        /* get port from hostname */
        /**
         * 如果没有使用代理, 则解析 URL 到 host 和 proxyport
         * eg: http://127.0.0.1:8080/
         * host(char *): 127.0.0.1
         * proxyport(int): 8080
         * */
        if (index(url + i, ':') != NULL &&
            index(url + i, ':') < index(url + i, '/')) { /* 如果指定了端口 */
            strncpy(host, url + i, strchr(url + i, ':') - url - i);
            // bzero(tmp,10);
            memset(tmp, 0, 10);
            strncpy(tmp, index(url + i, ':') + 1,
                    strchr(url + i, '/') - index(url + i, ':') - 1);
            /* printf("tmp=%s\n",tmp); */
            proxyport = atoi(tmp);
            if (proxyport == 0) proxyport = 80;
        } else { /* 没有指定端口 */
            strncpy(host, url + i, strcspn(url + i, "/"));
        }
        // printf("Host=%s\n",host);
        /**
         * 这个 request + strlen(request) 我实在是没看懂
         * strlen 获得 request 后第一个 '\0' 的位置
         * strcat 替换 request 后第一个 '\0' 并追加字符串, 在末尾添加 '\0'
         * 那为什么要先找到 '\0' 的位置再去 strcat......
         * */
        strcat(request + strlen(request), url + i + strcspn(url + i, "/"));
    } else {
        // printf("ProxyHost=%s\nProxyPort=%d\n",proxyhost,proxyport);
        strcat(request, url);
    }

    /* 拼接 HTTP 协议 */
    if (http10 == 1)
        strcat(request, " HTTP/1.0");
    else if (http10 == 2)
        strcat(request, " HTTP/1.1");

    strcat(request, "\r\n");

    /* 拼接 UA */
    if (http10 > 0)
        strcat(request, "User-Agent: WebBench " PROGRAM_VERSION "\r\n");
    /* 拼接 Host */
    if (proxyhost == NULL && http10 > 0) {
        strcat(request, "Host: ");
        strcat(request, host);
        strcat(request, "\r\n");
    }

    /* 拼接其余字段 */
    if (force_reload && proxyhost != NULL) {
        strcat(request, "Pragma: no-cache\r\n");
    }

    if (http10 > 1) strcat(request, "Connection: close\r\n");

    /* add empty line at end */
    if (http10 > 0) strcat(request, "\r\n");

    printf("\nRequest:\n%s\n", request);
}

/* vraci system rc error kod */
static int bench(void) {
    int i, j, k;
    pid_t pid = 0;
    FILE *f;

    /* check avaibility of target server */
    i = Socket(proxyhost == NULL ? host : proxyhost, proxyport);
    if (i < 0) {
        fprintf(stderr, "\nConnect to server failed. Aborting benchmark.\n");
        return 1;
    }
    close(i);

    /* create pipe */
    /* 创建管道 */
    if (pipe(mypipe)) {
        perror("pipe failed.");
        return 3;
    }

    /* not needed, since we have alarm() in childrens */
    /* wait 4 next system clock tick */
    /*
    cas=time(NULL);
    while(time(NULL)==cas)
    sched_yield();
    */

    /* fork childs */
    /* 创建处理子进程 */
    for (i = 0; i < clients; i++) {
        pid = fork();
        /* 不推荐将 fork 成功和失败的情况放在一起处理 */
        if (pid <= (pid_t)0) {
            /* child process or error*/
            /* 你不能确定进程将于何时中断 */
            /* 因此这种方法并不推荐使用 */
            sleep(1); /* make childs faster */
            break;
        }
    }

    /* 这是? 创建了多个子进程, 然后判断最后一个 fork 是否成功? */
    if (pid < (pid_t)0) {
        fprintf(stderr, "problems forking worker no. %d\n", i);
        perror("fork failed.");
        return 3;
    }

    if (pid == (pid_t)0) {
        /* I am a child */
        /* 开始发送请求 */
        /* 取全局变量中的 request 作为参数传给本文件内的另一个函数 */
        /* 什么写法...... */
        if (proxyhost == NULL)
            benchcore(host, proxyport, request);
        else
            benchcore(proxyhost, proxyport, request);

        /* write results to pipe */
        /* 将结果写入 pipe */
        /* 此处没有关闭 mypipe[0] */
        /* 不知是何用意 */
        f = fdopen(mypipe[1], "w");
        if (f == NULL) {
            perror("open pipe for writing failed.");
            return 3;
        }
        /* fprintf(stderr,"Child - %d %d\n",speed,failed); */
        fprintf(f, "%d %d %d\n", speed, failed, bytes);
        fclose(f);

        return 0;
    } else {
        /* 同样的, 此处没有关闭 mypipe[1] */
        f = fdopen(mypipe[0], "r");
        if (f == NULL) {
            perror("open pipe for reading failed.");
            return 3;
        }

        /* 设置无缓冲 */
        setvbuf(f, NULL, _IONBF, 0);

        speed = 0;
        failed = 0;
        bytes = 0;

        while (1) {
            pid = fscanf(f, "%d %d %d", &i, &j, &k);
            if (pid < 2) {
                fprintf(stderr, "Some of our childrens died.\n");
                break;
            }

            speed += i;
            failed += j;
            bytes += k;

            /* fprintf(stderr,"*Knock* %d %d read=%d\n",speed,failed,pid); */
            if (--clients == 0) break;
        }

        fclose(f);

        printf(
            "\nSpeed=%d pages/min, %d bytes/sec.\nRequests: %d susceed, %d "
            "failed.\n",
            (int)((speed + failed) / (benchtime / 60.0f)),
            (int)(bytes / (float)benchtime), speed, failed);
    }

    return i;
}

void benchcore(const char *host, const int port, const char *req) {
    int rlen;
    char buf[1500];
    int s, i;
    struct sigaction sa;

    /* setup alarm signal handler */
    /* 设置 SIGALRM 信号的处理程序 */
    sa.sa_handler = alarm_handler;
    sa.sa_flags = 0;
    if (sigaction(SIGALRM, &sa, NULL)) exit(3);

    /* 设置定时器, 超时产生 SIGALRM 信号 */
    alarm(benchtime);  // after benchtime,then exit

    rlen = strlen(req);
nexttry:
    while (1) {
        /**
         * 一直重试(重新发送一遍)
         * 直到超时
         * 如果超时的话, 把 failed--(是说被计时器中断的那个不算错误吗?)
         * */
        if (timerexpired) {
            if (failed > 0) {
                /* fprintf(stderr,"Correcting failed by signal\n"); */
                failed--;
            }
            return;
        }

        s = Socket(host, port);
        if (s < 0) {
            failed++;
            continue;
        }
        if (rlen != write(s, req, rlen)) {
            failed++;
            close(s);
            continue;
        }
        if (http10 == 0)
            if (shutdown(s, 1)) {
                failed++;
                close(s);
                continue;
            }
        if (force == 0) {
            /* read all available data from socket */
            /* 读取服务器响应 */
            while (1) {
                if (timerexpired) break;
                i = read(s, buf, 1500);
                /* fprintf(stderr,"%d\n",i); */
                if (i < 0) {
                    failed++;
                    close(s);
                    goto nexttry;
                } else if (i == 0)
                    break;
                else
                    bytes += i;
            }
        }
        if (close(s)) {
            failed++;
            continue;
        }
        speed++;
    }
}

```
