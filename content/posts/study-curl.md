---
title: curl 学习
date: 2018-07-22 14:17:54
tags: [网络编程, cUrl]
---

curl 不仅仅是一个编程时可以包含的函数库，也是一个功能强大的网站开发工具。仅仅使用其原生命令，就可以实现大部分网络爬虫的功能。

<!--more-->
## curl 的使用

### url
将要请求的 url, 必须填写的参数. 

默认的请求类型为 GET, 默认输出到标准输出

```shell
curl http://httpbin.org/get
```

### -C, --continue-at
使下载支持断点续传, 需要指定偏移量或者 `-` 自动计算

```shell
curl https://httpbin.org/download.zip -O -C -
```

如果下载中途中断了, 则可以再次执行此命令来进行续传

### -b, --cookie
使用指定的 cookie 请求页面

- 从 `-c` 导出的文件(cookie.txt)中读取
```shell
curl http://httpbin.org/cookies -b @cookie.txt
```

- 使用 cookie 字符串
```shell
curl http://httpbin.org/cookies -b 'key=value'
```

### -c, --cookie-jar
将 cookie 保存到指定的文件

```shell
curl http://httpbin.org/cookies/set/key/value -c cookie.txt
```

### -d, --data
请求中的 POST 数据, 添加此参数后默认请求类型变为 POST

```shell
curl http://httpbin.org/post -d 'key1=value1&key2=value2&key2=value3'
{
    "key1": "value1",
    "key2": ["value2", "value2"]
}
curl http://httpbin.org/post -d 'key1=value1' -d 'key2=value2' -d 'key2=value3'
{
    "key1": "value1",
    "key2": ["value2", "value2"]
}
```

### --data-ascii
类似于 -d, 暂时没有发现区别

### --data-urlencode
类似于 -d, 参数中的 '&' 会被转义, 不会切断参数

```shell
curl http://httpbin.org/post --data-urlencode 'key1=value1&key2=value2&key2=value3'
{
    "key1": "value1&key2=value2&key2=value3"
}
curl http://httpbin.org/post --data-urlencode 'key1=value1' --data-urlencode 'key2=value2' --data-urlencode 'key2=value3'
{
    "key1": "value1",
    "key2": ["value2", "value2"]
}
```

### -F, --form
指定 multipart MIME 数据, 可以用来上传文件

```shell
curl http://httpbin.org/post -F file=@filename
```

### --form-string
以 multipart MIME 格式上传数据

```shell
curl http://httpbin.org/post --form-string 'key=value'
```

### -G, --get
将请求的数据放在 URL 中, 以 `GET` 形式请求

```shell
curl meik$ curl http://httpbin.org/get -d 'key=value' -G
```

### -g, --globoff
忽略表示范围的 `[]` 和 `{}`

```shell
curl http://httpbin.org/cookies/set/key/value[1-5] -g
```

关于 `[]` 和 `{}` 表示范围的用法, 可以查看下面的"拓展用法"部分

### -I, --head
只请求头部信息, 使用的是 `head` 请求

```shell
curl http://httpbin.org/get -I
```

可以通过 `-v` 选项来验证其使用的是 `head` 请求

```shell
curl http://httpbin.org/get -I -v
...
> HEAD /get HTTP/1.1
...
```

### -H, --header
添加自定义的 `header` 信息

```shell
curl http://httpbin.org/get -H 'Key: value'
curl http://httpbin.org/get -H @header.txt
```

### -0, --http1.0
使用 HTTP/1.0

```shell
curl http://httpbin.org/get -0
```

### --http1.1
使用 HTTP/1.1

```shell
curl http://httpbin.org/get --http1.1
```

### -i, --include
同时在 `stdout` 中显示响应的头部信息

```shell
curl http://httpbin.org/get -i
```

### -L, --location
自动进行页面跳转

```shell
curl http://httpbin.org/status/302 -L
```

添加 `-v` 参数查看具体请求过程

```shell
curl http://httpbin.org/status/302 -L -v
> GET /status/302 HTTP/1.1
> Host: httpbin.org
>
< HTTP/1.1 302 FOUND
< Location: /redirect/1
<
> GET /redirect/1 HTTP/1.1
> Host: httpbin.org
>
< HTTP/1.1 302 FOUND
< Location: /get
<
> GET /get HTTP/1.1
> Host: httpbin.org
>
< HTTP/1.1 200 OK
```

可以看出, 如果返回的仍然是一个重定向的话, 那么 curl 会继续请求

如果跳转的时候设置了 `set-cookie`, 那么此选项也不会在后续的请求中添加 `cookie` 信息

```shell
curl http://httpbin.org/cookies/set/key/value -L
{"cookies":{}}
```

为了在跳转的页面中可以使用 `cookie`, 可以搭配使用 `-c` 和 `-b` 选项

```shell
curl http://httpbin.org/cookies/set/key/value -L -c cookie.txt -b @cookie.txt
{"cookies":{"key":"value"}}
```

### -o, --output
将请求到的数据保存为指定的文件

```shell
curl http://httpbin.org/get -o get.html
```

### -#, --progress-bar
下载时显示进度条

```shell
curl http://httpbin.org/download.zip -O -C - -#
#####                                    13.7%
```

会隐藏其他信息, 显示一个 `#` 组成的进度条, 以及文件百分比

### -x, --proxy
使用代理

```shell
curl https://www.google.com
(无响应)
^C
curl https://www.google.com -x http://127.0.0.1:6666
...(请求的结果)
```

### -e, --referer
设置 HTTP 头信息的 `referer`

```shell
curl http://httpbin.org/get -e 'http://httpbin.org'
```

有些网站会验证 `referer` 字段


### -O, --remote-name
将请求到的数据以 URL 中的路径为名保存到本地

```shell
curl http://httpbin.org/get -O
```

请求到的数据会被保存到名为 `get` 的文件内

### -X, --request
指定请求的类型

```shell
curl http://httpbin.org/post -X POST
```

请求的类型可以为自定义的类型, 比如

```shell
curl http://httpbin.org/post -X POST666 -v
...
> POST666 /post HTTP/1.1
...
```

### -S, --show-error
在静默模式下也输出错误信息

### -s, --silent
静默模式

### trace
输出 debug 信息到文件

```shell
curl http://httpbin.org/get --trace trace.txt
```

### trace-ascii
输出 debug 信息到文件, 不包含 hex 数据

```shell
curl http://httpbin.org/get --trace-ascii trace.txt
```

### trace-time
为 `trace` 的输出时间

```shell
curl http://httpbin.org/get --trace-ascii trace.txt --trace-time
```

### --url
要请求的 URL

```shelll
curl --url http://httpbin.org/get
```

### -A, --user-agent
修改请求的 UA

```shell
curl meik$ curl http://httpbin.org/get -A 'MeiK'
"User-Agent": "MeiK"
```

### -v, --verbose
让输出更加详细, 会额外输出请求 ip 、请求头、响应头等信息

```shell
curl http://httpbin.org/get -v
```

### -V, --version
输出版本信息和版本特性等信息

```shell
curl -V
```

## 拓展用法

### `[]` 与 `{}` 表示范围

- 用 `{}` 表示多个 URL
```shell
curl http://httpbin.org/cookies/set/key/value{1,2,3}
```
会访问三个 URL

- 用 `[]` 表示范围
```shell
curl http://httpbin.org/cookies/set/key/value[1-3]
curl http://httpbin.org/cookies/set/key/value[a-z]
```
如果你用过 Python 的话, 那应该对这种用法很熟悉
```shell
curl http://httpbin.org/cookies/set/key/value[1-10:2]
```
冒号后边的部分是步长, 表示等差排列的阶跃数

- 混合使用 `{}` 与 `[]`
```shell
curl http://httpbin.org/cookies/set/key{1,2,3}/value[1-3]
```
会请求其每种组合(3*3=9)
