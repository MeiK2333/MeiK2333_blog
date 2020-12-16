---
title: 如何写一个爬虫 - 第二篇
date: 2019-08-05 20:56:35
tags: [爬虫, Python]
---

从零开始实现一个爬虫。

<!--more-->

> 让我大吃一惊的是，我离开大学的时候就被告知，说我已经学过航海学了！——得了，我只要到港口去兜个圈，管保学到更多的航海知识。------《瓦尔登湖》 - 亨利·大卫·梭罗

上一篇中我们实现了一个 `fetch` 函数，这个函数接受一个 `url` 参数，并返回请求的响应体。在这一章里，我们将拓展这个函数的功能，并将其打包为可安装的库。

## 改造 `fetch` 函数

### 定义函数签名

上一章中我们大概介绍了 HTTP 的格式，现在我们来对其中每一部分的格式与规范进行解读。

```
[method] [path] [协议版本]
[header 字段]: [header 值]
...
[header 字段]: [header 值]

[数据]
```

在 HTTP 的请求格式中，除了协议版本外，我们有 method 、 path 、 header 字段和数据四部分可以自定义，我们就来将这四部分作为我们新的函数的参数。

我们首先来分别看一看这四部分每一部分的定义：

**`method`**

HTTP 定义的请求方法，目前有 `GET` 、 `HEAD` 、 `POST` 、 `PUT` 、 `DELETE` 、 `CONNECT` 、 `OPTIONS` 、`TRACE`  和 `PATCH` 九种，在 [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods) 中可以查看其详细定义，其中最常用的是 `GET` 与 `POST` 。格式为字符串（Python 中的 `str` ）。

**`path`**

想要知道 `path` 是什么，就要先知道 URL 的格式定义（来自[维基百科](https://en.wikipedia.org/wiki/URL)）：

```
URI = scheme:[//authority]path[?query][#fragment]
authority = [userinfo@]host[:port]
```

（关于这里为什么是 URI 而非 URL ，可以参照[这个问题](https://stackoverflow.com/questions/176264/what-is-the-difference-between-a-uri-a-url-and-a-urn)。简而言之， URL 是 URI 的子集。）

将其展开并去掉不常见的部分后，我们将其简化为下面这样：

```
URL = scheme://host[:port]path[?query][#fragment]
```

- scheme: 连接服务器的协议，在网站的请求中，一般为 http 与 https 其中一个
- host: 服务器的域，可能是域名或者 IP ，如果是域名的话，在连接时会首先将域名转化为 IP
- port: 服务器提供服务的端口， HTTP 默认为 80 ， HTTPS 默认为 443
- path: 在早期 HTTP 世界里，这个字段代表了要访问的资源的定位。现在它的具体含义由服务器端定义
- query: HTTP GET 附带的查询参数，一般格式为 `key=value` 这样的键值对，但与 `path` 类似，它具体如何解释还是要看服务器的定义
- fragment: 常用于页内资源定位

一个常见的 URL 可能类似这样： `https://httpbin.org/get?key=value#1` ，需要注意的是， `#` 后的部分被称为 hash ，常见的作用是定位页面内的资源，一般来说不会被发送到服务器，仅在浏览器端生效。我们可以使用 `curl` 进行验证：

```curl
$ curl 'https://httpbin.org/get?key=value#1' -v
......
> GET /get?key=value HTTP/1.1
> Host: httpbin.org
> User-Agent: curl/7.54.0
> Accept: */*
>
......
{
  "args": {
    "key": "value"
  },
  "headers": {
    "Accept": "*/*",
    "Host": "httpbin.org",
    "User-Agent": "curl/7.54.0"
  },
  "origin": "36.110.78.251, 36.110.78.251",
  "url": "https://httpbin.org/get?key=value"
}
* Connection #0 to host httpbin.org left intact
```

可以看到， `#` 及其之后的部分并没有被发送到服务器，而 `key=value` 被解析为了键值对。

因此， `fragment` 部分不在我们的考虑范围内，我们要发送的 `path` 实际上是这里的 `path` 加上可选的查询字符串 `[?query]` ，在上面的例子里就是 `/get?key=value` 。其格式为字符串。

**`header`**

从上一章中可以看出， `header` 的格式为键值对——对应到 Python 里面，就是字典（ `dict` ）。

一个典型的 `header` 可能长这个样：

```
User-Agent: ZeroCrawler
```

**`data`**

数据段（ `data` ）的格式为字符串，也是 HTTP 协议所规定的。根据 `header` 的不同，数据段有多种不同的解析方式，但我们先不考虑那么多，仅以字符串来表示它。

因此，为了能够最大程度的自定义我们的请求，我们的函数签名定义为下面这样：

```python
def fetch(
    method: str, 
    url: str, 
    params: dict = None, 
    headers: dict = None
    data: str = None, 
) -> str:
```

### 填充函数内容

函数签名已经确定了，要填充其内容就比较简单了。我们要做的就是将每个参数填充到其应该在的位置。填充完的函数如下：

```python
def fetch(
        method: str, url: str, params: dict = None, data: str = None, headers: dict = None
) -> str:
    if url.startswith("http://") is False:
        raise ValueError("url must start with `http://`")

    _tmp = url[7:].split("/", 1)
    if len(_tmp) > 1:
        host = _tmp[0]
        path = "/" + _tmp[1]
    else:
        host = _tmp[0]
        path = "/"

    method = method.upper()
    params = {} if params is None else params
    headers = {} if headers is None else headers

    params_str = "?" if params else ""
    param_count = 0
    for key, value in params.items():
        if param_count != 0:
            params_str += '&'
        params_str += f"{key}={value}"
        param_count += 1

    request_list = [f"{method} {path}{params_str} HTTP/1.0", f"Host: {host}"]
    for key, value in headers.items():
        request_list.append(f"{key}: {value}")

    if data is not None:
        request_list.append(f'Content-Length: {len(data)}')
        request_list.append('')  # 空行，以分割header域与data域
        request_list.append(data)
    request_list.append("\r\n")

    request = "\r\n".join(request_list)

    sock = socket.socket()
    sock.connect((host, 80))
    sock.send(request.encode("ascii"))

    response = b""
    chunk = sock.recv(4096)
    while chunk:
        response += chunk
        chunk = sock.recv(4096)

    sock.close()

    return response.decode()
```

4 - 13 行中，我们从 URL 中分割出 `host` ，因为我们的函数还不支持 https ，因此我们对非 http 的请求抛出异常。

15 - 17 行，我们为可选的参数分配默认值，如果这些参数没有提供的话，则使用默认的策略填充。

19 - 25 行，我们将 `path` 中的 `query` 拼接起来，以便后面将其添加到 `path` 后面。

27 - 29 行，我们将 `header` 键值对加入请求中。

31 - 35 行，我们将可能存在的 `data` 添加到请求里。

往后的操作与我们上一章中所做的一样：开启 TCP 连接、发送请求、接收响应直到连接关闭。

试用一下我们的程序：

```python
>>> from ZeroCrawler import fetch
>>> resp = fetch('get', 'http://httpbin.org/get')
>>> resp
'HTTP/1.1 200 OK\r\nAccess-Control-Allow-Credentials: true\r\nAccess-Control-Allow-Origin: *\r\nContent-Type: application/json\r\nDate: Tue, 30 Jul 2019 12:19:26 GMT\r\nReferrer-Policy: no-referrer-when-downgrade\r\nServer: nginx\r\nX-Content-Type-Options: nosniff\r\nX-Frame-Options: DENY\r\nX-XSS-Protection: 1; mode=block\r\nContent-Length: 146\r\nConnection: Close\r\n\r\n{\n  "args": {}, \n  "headers": {\n    "Host": "httpbin.org"\n  }, \n  "origin": "36.110.78.251, 36.110.78.251", \n  "url": "https://httpbin.org/get"\n}\n'
```

## 创建一个可安装的库

至此，我们的函数已经能够有一定的可用性与灵活性了，现在我们要让其他人来使用我们的函数。

如果我们的程序的用户需要通过复制代码到自己项目内的方式来使用，那未免过于原始了。我们可以将程序打包为一个包，用户可以很方便的安装我们的程序。

官网有打包程序的文档： [Packaging Python Projects](https://packaging.python.org/tutorials/packaging-projects/)，按照官网的指导，我们首先创建一个文件 `setup.py` ，写入如下内容：

```python
import setuptools

with open("README.md", "r") as fh:
    long_description = fh.read()

setuptools.setup(
    name="ZeroCrawler",
    version="0.0.2",
    author="MeiK2333",
    author_email="meik2333@gmail.com",
    description="A small example package",
    long_description=long_description,
    long_description_content_type="text/markdown",
    url="https://github.com/MeiK2333/ZeroCrawler",
    packages=setuptools.find_packages(),
)
```

这样就完成了，我们可以直接安装或使用 pip 安装：

```bash
$ python setup.py install  # or pip install .
Processing /Users/meik/ZeroCrawler
Installing collected packages: ZeroCrawler
  Running setup.py install for ZeroCrawler ... done
Successfully installed ZeroCrawler-0.0.2
```

我们可以验证一下是否安装成功，我们切换到其他目录下，然后输出一下 `ZeroCrawler` 的位置：

```shell
$ python -c 'import ZeroCrawler; print(ZeroCrawler.__path__)'  # 此时引入的 ZeroCrawler 是我们正在开发的版本
['/Users/meik/ZeroCrawler/ZeroCrawler']
$ cd ~
$ python -c 'import ZeroCrawler; print(ZeroCrawler.__path__)'  # 此时引入的是我们安装的版本
['/Users/meik/ZeroCrawler/venv/lib/python3.6/site-packages/ZeroCrawler']
```

可以看到， `ZeroCrawler` 库已经被安装到我们的系统里了，我们可以在其他项目中使用它。

如果想要更多的用户可以使用我们的项目（此时已经可以被称为库了），我们可以将项目提交到 GitHub 或者 [pypi](https://pypi.org/) ， pip 可以直接安装 GitHub 或者 pypi 上公开的库。

## 添加测试

添加了 `setup.py` 之后，我们的项目已经可以称之为一个库了，但距离成为一个有诚意的库还差一点，这一点就是测试。

测试的好处很多，提高代码质量、多人协作更加放心、可以作为使用指南等等，此处不做详解。我们只关心如何为我们的框架添加测试。

Python 的测试框架有很多，其中最常用的就是 unittest ，我以前写过一篇博客来介绍其基本用法：[《Python 中的单元测试（ unittest 的基础用法）》](https://meik2333.com/2018/02/23/Python-%E4%B8%AD%E7%9A%84%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%EF%BC%88-unittest-%E7%9A%84%E5%9F%BA%E7%A1%80%E7%94%A8%E6%B3%95%EF%BC%89/)。

我们要测试的 `fetch` 函数返回值是响应的全部内容，不方便进行解析测试，因此我们创建了一个获取响应 `data` 的助手函数。

创建文件 `ZeroCrawler/utils.py` ，写入如下内容：

```python
def get_body_by_response(response):
    """ 从返回的响应中获得数据正文 """
    return response.split('\r\n\r\n', 1)[1]
```

然后我们开始写测试，创建文件 `tests/test_fetch.py` ，写入如下内容：

```python
import json
import unittest

from ZeroCrawler import fetch
from ZeroCrawler.utils import get_body_by_response


class TestFetch(unittest.TestCase):
    def test_fetch(self):
        resp = fetch("get", "http://httpbin.org/get")
        self.assertTrue(resp.startswith("HTTP/1.1 200 OK"))

    def test_fetch_params(self):
        params = {"key1": "value1", "key2": "value2"}
        resp = fetch("get", "http://httpbin.org/get", params=params)
        data = get_body_by_response(resp)
        data = json.loads(data)
        args = data.get("args")
        for key, value in params.items():
            self.assertTrue(key in args.keys())
            self.assertEqual(value, args[key])

    def test_fetch_headers(self):
        headers = {"User-Agent": "ZeroCrawler"}
        resp = fetch("get", "http://httpbin.org/get", headers=headers)
        data = get_body_by_response(resp)
        data = json.loads(data)
        resp_headers = data.get("headers")
        for key, value in headers.items():
            self.assertTrue(key in resp_headers.keys())
            self.assertEqual(value, resp_headers[key])

    def test_fetch_method(self):
        methods = ["get", "post", "put", "delete", "patch"]
        for method in methods:
            resp = fetch(method, f"http://httpbin.org/{method}")
            data = get_body_by_response(resp)
            data = json.loads(data)
            url = data.get("url")
            self.assertTrue(url.endswith(f"httpbin.org/{method}"))

    def test_fetch_data(self):
        data = "Hello World!"
        resp = fetch("post", "http://httpbin.org/post", data=data)
        resp_data = get_body_by_response(resp)
        resp_data = json.loads(resp_data)
        self.assertEqual(resp_data["data"], data)


if __name__ == "__main__":
    unittest.main()
```

这五组测试分别测试了我们函数的五个参数，写明了进行的操作和应该出现的结果。运行它：

```shell
$ python -m unittest tests/test_*.py
......
----------------------------------------------------------------------
Ran 5 tests in 27.590s

OK
```

之后如果我们修改了项目，只要测试能通过，就代表我们现在已有的功能没有被破坏（当然，前提是有全面且合理的测试用例）。

至此，我们的项目结构如下所示：

```
ZeroCrawler
    |
    |------ ZeroCrawler
    |         |-------- __init__.py
    |         |-------- fetch.py
    |         |-------- utils.py
    |      
    |------ tests
    |         |-------- test_fetch.p
    |
    |------ setup.py
```

## 总结

这一章里，我们提高了我们框架的泛用性，并用我们的框架创建了一个可安装的库。我们的框架有了走向世界的基础。

本章的代码在 GitHub: [ZeroCrawler Version 0.0.2](https://github.com/MeiK2333/ZeroCrawler/tree/96f62b98bd93b3d384e846b4b5b53b1e593e3d5c) 中，可以尝试 `fork` 它并自己进行修改。

我们的框架仅仅有个雏形，还有很多不足，比如意大利面条式的代码、没有对请求与响应内容的解析、用法并不友好等。这些问题我们会在后面的章节里一一改正，如果你认为我写的有什么不足之处，请在博客评论处或者 GitHub Issues 告诉我。我们下期再见。
