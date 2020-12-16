---
title: Tinyhttpd 源码及分析
date: 2018-06-29 17:16:20
tags: [Tinyhttpd, C 语言, 网络编程]
---

Tinyhttpd 是J. David Blackstone在1999年写的一个不到 500 行的超轻量型 Http Server，分析这个项目可以帮助我们理解服务器软件的本质。


<!--more-->


## 源码

```C
/* J. David's webserver */
/* This is a simple webserver.
 * Created November 1999 by J. David Blackstone.
 * CSE 4344 (Network concepts), Prof. Zeigler
 * University of Texas at Arlington
 */
/* This program compiles for Sparc Solaris 2.6.
 * To compile for Linux:
 *  1) Comment out the #include <pthread.h> line.
 *  2) Comment out the line that defines the variable newthread.
 *  3) Comment out the two lines that run pthread_create().
 *  4) Uncomment the line that runs accept_request().
 *  5) Remove -lsocket from the Makefile.
 */
/**
 * 较新版本的 Linux 已经内置了对线程的支持，只需要在编译时添加 -lpthread
 * 参数即可 调整 Makefile 的编译选项，使 $(LIBS) 在第 4 行的最后即可
 */
#include <arpa/inet.h>
#include <ctype.h>
#include <netinet/in.h>
#include <pthread.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <strings.h>
#include <sys/socket.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

#define ISspace(x) isspace((int)(x))

#define SERVER_STRING "Server: jdbhttpd/0.1.0\r\n"
#define STDIN 0
#define STDOUT 1
#define STDERR 2

#define u_short unsigned short int

void accept_request(void *);
void bad_request(int);
void cat(int, FILE *);
void cannot_execute(int);
void error_die(const char *);
void execute_cgi(int, const char *, const char *, const char *);
int get_line(int, char *, int);
void headers(int, const char *);
void not_found(int);
void serve_file(int, const char *);
int startup(u_short *);
void unimplemented(int);

/**********************************************************************/
/* A request has caused a call to accept() on the server port to
 * return.  Process the request appropriately.
 * Parameters: the socket connected to the client */
/**********************************************************************/
void accept_request(void *arg) {
  int client = (intptr_t)arg;
  char buf[1024];
  size_t numchars;
  char method[255];
  char url[255];
  char path[512];
  size_t i, j;
  struct stat st;
  int cgi = 0; /* becomes true if server decides this is a CGI
                * program */
  char *query_string = NULL;

  /* 读取第一行，格式类似 "GET / HTTP/1.1" */
  numchars = get_line(client, buf, sizeof(buf));
  i = 0;
  j = 0;
  /* 首先解析 method (GET / POST) */
  while (!ISspace(buf[i]) && (i < sizeof(method) - 1)) {
    method[i] = buf[i];
    i++;
  }
  j = i;
  method[i] = '\0';

  /* 不能处理 GET 和 POST 之外的请求类型 */
  if (strcasecmp(method, "GET") && strcasecmp(method, "POST")) {
    /* 直接返回 501 Method Not Implemented */
    unimplemented(client);
    return;
  }

  /* 若为 POST 请求类型，则认为送给 cgi 程序处理 */
  if (strcasecmp(method, "POST") == 0) {
    cgi = 1;
  }

  i = 0;
  while (ISspace(buf[j]) && (j < numchars)) {
    j++;
  }
  while (!ISspace(buf[j]) && (i < sizeof(url) - 1) && (j < numchars)) {
    url[i] = buf[j];
    i++;
    j++;
  }
  url[i] = '\0';

  if (strcasecmp(method, "GET") == 0) {
    query_string = url;
    while ((*query_string != '?') && (*query_string != '\0')) {
      query_string++;
    }
    if (*query_string == '?') {
      cgi = 1;
      *query_string = '\0';
      query_string++;
    }
  }

  sprintf(path, "htdocs%s", url);
  if (path[strlen(path) - 1] == '/') {
    strcat(path, "index.html");
  }
  /* 判断请求的文件是否存在 */
  if (stat(path, &st) == -1) {
    /* read & discard headers */
    while ((numchars > 0) && strcmp("\n", buf)) {
      numchars = get_line(client, buf, sizeof(buf));
    }
    not_found(client);
  } else {
    /* 如果为一个目录，则自动寻找 index.html */
    if ((st.st_mode & S_IFMT) == S_IFDIR) {
      strcat(path, "/index.html");
    }
    if ((st.st_mode & S_IXUSR) || (st.st_mode & S_IXGRP) ||
        (st.st_mode & S_IXOTH)) {
      cgi = 1;
    }
    if (!cgi) {
      serve_file(client, path);
    } else {
      execute_cgi(client, path, method, query_string);
    }
  }

  close(client);
}

/**********************************************************************/
/* Inform the client that a request it has made has a problem.
 * Parameters: client socket */
/**********************************************************************/
void bad_request(int client) {
  char buf[1024];

  sprintf(buf, "HTTP/1.0 400 BAD REQUEST\r\n");
  send(client, buf, sizeof(buf), 0);
  sprintf(buf, "Content-type: text/html\r\n");
  send(client, buf, sizeof(buf), 0);
  sprintf(buf, "\r\n");
  send(client, buf, sizeof(buf), 0);
  sprintf(buf, "<P>Your browser sent a bad request, ");
  send(client, buf, sizeof(buf), 0);
  sprintf(buf, "such as a POST without a Content-Length.\r\n");
  send(client, buf, sizeof(buf), 0);
}

/**********************************************************************/
/* Put the entire contents of a file out on a socket.  This function
 * is named after the UNIX "cat" command, because it might have been
 * easier just to do something like pipe, fork, and exec("cat").
 * Parameters: the client socket descriptor
 *             FILE pointer for the file to cat */
/**********************************************************************/
void cat(int client, FILE *resource) {
  char buf[1024];

  fgets(buf, sizeof(buf), resource);
  while (!feof(resource)) {
    send(client, buf, strlen(buf), 0);
    fgets(buf, sizeof(buf), resource);
  }
}

/**********************************************************************/
/* Inform the client that a CGI script could not be executed.
 * Parameter: the client socket descriptor. */
/**********************************************************************/
void cannot_execute(int client) {
  char buf[1024];

  sprintf(buf, "HTTP/1.0 500 Internal Server Error\r\n");
  send(client, buf, strlen(buf), 0);
  sprintf(buf, "Content-type: text/html\r\n");
  send(client, buf, strlen(buf), 0);
  sprintf(buf, "\r\n");
  send(client, buf, strlen(buf), 0);
  sprintf(buf, "<P>Error prohibited CGI execution.\r\n");
  send(client, buf, strlen(buf), 0);
}

/**********************************************************************/
/* Print out an error message with perror() (for system errors; based
 * on value of errno, which indicates system call errors) and exit the
 * program indicating an error. */
/**********************************************************************/
void error_die(const char *sc) {
  perror(sc);
  exit(1);
}

/**********************************************************************/
/* Execute a CGI script.  Will need to set environment variables as
 * appropriate.
 * Parameters: client socket descriptor
 *             path to the CGI script */
/**********************************************************************/
void execute_cgi(int client, const char *path, const char *method,
                 const char *query_string) {
  char buf[1024];
  int cgi_output[2];
  int cgi_input[2];
  pid_t pid;
  int status;
  int i;
  char c;
  int numchars = 1;
  int content_length = -1;

  buf[0] = 'A';
  buf[1] = '\0';
  if (strcasecmp(method, "GET") == 0) {
    /* read & discard headers */
    while ((numchars > 0) && strcmp("\n", buf)) {
      numchars = get_line(client, buf, sizeof(buf));
    }
  } else if (strcasecmp(method, "POST") == 0) { /* POST */
    numchars = get_line(client, buf, sizeof(buf));
    while ((numchars > 0) && strcmp("\n", buf)) {
      buf[15] = '\0';
      if (strcasecmp(buf, "Content-Length:") == 0) {
        content_length = atoi(&(buf[16]));
      }
      numchars = get_line(client, buf, sizeof(buf));
    }
    if (content_length == -1) {
      bad_request(client);
      return;
    }
  } else { /*HEAD or other*/
  }

  if (pipe(cgi_output) < 0) {
    cannot_execute(client);
    return;
  }
  if (pipe(cgi_input) < 0) {
    cannot_execute(client);
    return;
  }

  if ((pid = fork()) < 0) {
    cannot_execute(client);
    return;
  }
  sprintf(buf, "HTTP/1.0 200 OK\r\n");
  send(client, buf, strlen(buf), 0);
  /* child: CGI script */
  /**
   * 子进程去执行 cgi 程序
   * 父子进程通过 pipe 管道通信
   * */
  if (pid == 0) {
    char meth_env[255];
    char query_env[255];
    char length_env[255];

    dup2(cgi_output[1], STDOUT);
    dup2(cgi_input[0], STDIN);
    close(cgi_output[0]);
    close(cgi_input[1]);
    sprintf(meth_env, "REQUEST_METHOD=%s", method);
    putenv(meth_env);
    if (strcasecmp(method, "GET") == 0) {
      sprintf(query_env, "QUERY_STRING=%s", query_string);
      putenv(query_env);
    } else { /* POST */
      sprintf(length_env, "CONTENT_LENGTH=%d", content_length);
      putenv(length_env);
    }
    execl(path, NULL);
    exit(0);
  } else { /* parent */
    close(cgi_output[1]);
    close(cgi_input[0]);
    if (strcasecmp(method, "POST") == 0) {
      for (i = 0; i < content_length; i++) {
        recv(client, &c, 1, 0);
        write(cgi_input[1], &c, 1);
      }
    }
    /* 将 cgi 程序的输出返回到网页 */
    while (read(cgi_output[0], &c, 1) > 0) {
      send(client, &c, 1, 0);
    }

    close(cgi_output[0]);
    close(cgi_input[1]);
    waitpid(pid, &status, 0);
  }
}

/**********************************************************************/
/* Get a line from a socket, whether the line ends in a newline,
 * carriage return, or a CRLF combination.  Terminates the string read
 * with a null character.  If no newline indicator is found before the
 * end of the buffer, the string is terminated with a null.  If any of
 * the above three line terminators is read, the last character of the
 * string will be a linefeed and the string will be terminated with a
 * null character.
 * Parameters: the socket descriptor
 *             the buffer to save the data in
 *             the size of the buffer
 * Returns: the number of bytes stored (excluding null) */
/**********************************************************************/
int get_line(int sock, char *buf, int size) {
  int i = 0;
  char c = '\0';
  int n;

  while ((i < size - 1) && (c != '\n')) {
    n = recv(sock, &c, 1, 0);
    /* DEBUG printf("%02X\n", c); */
    if (n > 0) {
      if (c == '\r') {
        n = recv(sock, &c, 1, MSG_PEEK);
        /* DEBUG printf("%02X\n", c); */
        if ((n > 0) && (c == '\n')) {
          recv(sock, &c, 1, 0);
        } else {
          c = '\n';
        }
      }
      buf[i] = c;
      i++;
    } else {
      c = '\n';
    }
  }
  buf[i] = '\0';

  return (i);
}

/**********************************************************************/
/* Return the informational HTTP headers about a file. */
/* Parameters: the socket to print the headers on
 *             the name of the file */
/**********************************************************************/
void headers(int client, const char *filename) {
  char buf[1024];
  (void)filename; /* could use filename to determine file type */

  strcpy(buf, "HTTP/1.0 200 OK\r\n");
  send(client, buf, strlen(buf), 0);
  strcpy(buf, SERVER_STRING);
  send(client, buf, strlen(buf), 0);
  sprintf(buf, "Content-Type: text/html\r\n");
  send(client, buf, strlen(buf), 0);
  strcpy(buf, "\r\n");
  send(client, buf, strlen(buf), 0);
}

/**********************************************************************/
/* Give a client a 404 not found status message. */
/**********************************************************************/
void not_found(int client) {
  char buf[1024];

  sprintf(buf, "HTTP/1.0 404 NOT FOUND\r\n");
  send(client, buf, strlen(buf), 0);
  sprintf(buf, SERVER_STRING);
  send(client, buf, strlen(buf), 0);
  sprintf(buf, "Content-Type: text/html\r\n");
  send(client, buf, strlen(buf), 0);
  sprintf(buf, "\r\n");
  send(client, buf, strlen(buf), 0);
  sprintf(buf, "<HTML><TITLE>Not Found</TITLE>\r\n");
  send(client, buf, strlen(buf), 0);
  sprintf(buf, "<BODY><P>The server could not fulfill\r\n");
  send(client, buf, strlen(buf), 0);
  sprintf(buf, "your request because the resource specified\r\n");
  send(client, buf, strlen(buf), 0);
  sprintf(buf, "is unavailable or nonexistent.\r\n");
  send(client, buf, strlen(buf), 0);
  sprintf(buf, "</BODY></HTML>\r\n");
  send(client, buf, strlen(buf), 0);
}

/**********************************************************************/
/* Send a regular file to the client.  Use headers, and report
 * errors to client if they occur.
 * Parameters: a pointer to a file structure produced from the socket
 *              file descriptor
 *             the name of the file to serve */
/**********************************************************************/
void serve_file(int client, const char *filename) {
  FILE *resource = NULL;
  int numchars = 1;
  char buf[1024];

  buf[0] = 'A';
  buf[1] = '\0';
  /* read & discard headers */
  while ((numchars > 0) && strcmp("\n", buf)) {
    numchars = get_line(client, buf, sizeof(buf));
  }

  resource = fopen(filename, "r");
  if (resource == NULL) {
    not_found(client);
  } else {
    headers(client, filename);
    cat(client, resource);
  }
  fclose(resource);
}

/**********************************************************************/
/* This function starts the process of listening for web connections
 * on a specified port.  If the port is 0, then dynamically allocate a
 * port and modify the original port variable to reflect the actual
 * port.
 * Parameters: pointer to variable containing the port to connect on
 * Returns: the socket */
/**********************************************************************/
int startup(u_short *port) {
  int httpd = 0;
  int on = 1;
  struct sockaddr_in name;

  /**
   * socker 创建套接字连接
   * PF_INET \ AF_INET IPv4 因特网域
   * SOCK_STREAM 有序的、可靠的、双向的、面向连接的字节流
   * */
  httpd = socket(PF_INET, SOCK_STREAM, 0);
  if (httpd == -1) {
    error_die("socket");
  }

  memset(&name, 0, sizeof(name));
  name.sin_family = AF_INET;
  /* htons: Host to Network Short */
  name.sin_port = htons(*port);
  /* htonl: Host to Network Long */
  name.sin_addr.s_addr = htonl(INADDR_ANY);
  if ((setsockopt(httpd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on))) < 0) {
    error_die("setsockopt failed");
  }
  /* 如果没有指定端口，则系统会自动指定一个 */
  if (bind(httpd, (struct sockaddr *)&name, sizeof(name)) < 0) {
    error_die("bind");
  }
  /* 如果没有指定端口，则将系统指定的端口赋值给端口参数 */
  if (*port == 0) {
    socklen_t namelen = sizeof(name);
    if (getsockname(httpd, (struct sockaddr *)&name, &namelen) == -1) {
      error_die("getsockname");
    }
    *port = ntohs(name.sin_port);
  }
  /* listen 的第二个参数指定了可以排队的请求的个数，多余的会被拒绝 */
  if (listen(httpd, 5) < 0) {
    error_die("listen");
  }
  return (httpd);
}

/**********************************************************************/
/* Inform the client that the requested web method has not been
 * implemented.
 * Parameter: the client socket */
/**********************************************************************/
void unimplemented(int client) {
  char buf[1024];

  sprintf(buf, "HTTP/1.0 501 Method Not Implemented\r\n");
  send(client, buf, strlen(buf), 0);
  sprintf(buf, SERVER_STRING);
  send(client, buf, strlen(buf), 0);
  sprintf(buf, "Content-Type: text/html\r\n");
  send(client, buf, strlen(buf), 0);
  sprintf(buf, "\r\n");
  send(client, buf, strlen(buf), 0);
  sprintf(buf, "<HTML><HEAD><TITLE>Method Not Implemented\r\n");
  send(client, buf, strlen(buf), 0);
  sprintf(buf, "</TITLE></HEAD>\r\n");
  send(client, buf, strlen(buf), 0);
  sprintf(buf, "<BODY><P>HTTP request method not supported.\r\n");
  send(client, buf, strlen(buf), 0);
  sprintf(buf, "</BODY></HTML>\r\n");
  send(client, buf, strlen(buf), 0);
}

/**********************************************************************/

int main(void) {
  int server_sock = -1;
  u_short port = 0;
  int client_sock = -1;
  struct sockaddr_in client_name;
  socklen_t client_name_len = sizeof(client_name);
  pthread_t newthread;

  server_sock = startup(&port);
  printf("httpd running on port %d\n", port);

  while (1) {
    /* accept 函数获得连接请求并建立连接，返回套接字描述符 */
    client_sock =
        accept(server_sock, (struct sockaddr *)&client_name, &client_name_len);
    if (client_sock == -1) {
      error_die("accept");
    }

    /* 创建新线程处理此次请求 */
    if (pthread_create(&newthread, NULL, (void *)accept_request,
                       (void *)(intptr_t)client_sock) != 0) {
      perror("pthread_create");
    }
  }

  close(server_sock);

  return (0);
}

```

## 分析

### main

调用 `startup` 函数初始化连接， `while(1)` 接收请求，将接收到的信号多线程分发给 `accept_request` 函数处理。

### startup

创建套接字连接，绑定端口，返回 `socket` 连接的描述符。

### accept_request

解析 `client` 发送的数据，从中提取出 `method` 、 `url` 等信息，根据 `url` 确定对应的文件 `path` ，根据 `method` 的具体属性决定后续处理方式。

- 如果 `method` 为 `GET` ，则尝试从 `url` 中解析 `query` 参数。
  - 如果没有 `query` 参数，则判断请求的路径对应的文件属性
    - 如果对应的文件有 `x` 属性（即可执行属性），则作为 CGI 文件处理
    - 否则作为静态页面，直接返回文件内容（此处对不同的文件类型没有区分，全部是作为 `document` 返回的，因此并不能正确的返回非文本的静态资源）。
  - 如果有 `query` 参数，则作为 CGI 程序处理。（其实这样并不完全合理，因为带有 `query` 参数也有可能是请求静态资源，比如 JavaScript 脚本文件的版本控制）
- 如果 `method` 为 `POST` ，则作为 CGI 程序处理
- 其他类型的 `method` ，返回 `501 Method Not Implemented`

静态页面请求交由 `serve_file` 函数处理， CGI 程序交由 `execute_cgi` 函数处理。

### serve_file

读取文件内容， `send` 返回给请求的服务端。

此处可以优化一下，根据不同的文件类型返回不同的 `Content-Type` 信息，这样就不仅仅可以处理文本类型的静态文件了。

### execute_cgi

`fork` 创建子进程， `pipe` 创建父进程与子进程的管道，重定向子进程的标准输入输出。

- 对于 `GET` 请求，将 `GET` 请求的 Query String 存入 `env` 的 `QUERY_STRING` 字段，然后 `exec` 执行 CGI 程序
- 对于 `POST` 请求，将 `POST` 请求的 `Content-Length` 存入 `env` 的 `CONTENT_LENGTH` 字段， `exec` 执行 CGI 程序，然后父进程将 `POST` 的数据由标准输入传递给 CGI 程序

父进程将 CGI 程序的输出 `send` 给 `client` ，浏览器将返回数据渲染出来。

此函数没有处理 `POST` 请求的 Query String ，但很多 `POST` 请求也是同时带有 Query String 参数的。

## 总结

Tinyhttpd 作为一个超轻量级的 HTTP Server ，其功能并不十分完善，但其体现出了一个 HTTP Server 的整体流程与设计思路，非常值得阅读学习。