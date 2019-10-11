---
title: 如何写一个爬虫 - 第三篇
date: 2019-08-16 14:00:57
tags: [爬虫, Python]
---

从零开始实现一个爬虫。

<!--more-->

> "But man is not made for defeat," he said. "A man can be destroyed but not defeated."
> “不过我得记住一点，人不是为失败而生的，”他说，“一个人可以被毁灭，但不能给打败。”------《老人与海》 - 海明威

我们已经有了一个可用的爬虫框架——虽然它只有一个函数 `fetch`。它的用法并不够明确，且直接返回 HTTP 响应的所有内容未免过于简单粗暴，这对我们用户的使用带来了很多困扰。因此，我们还需要对其进行改造，使我们的爬虫框架更加清晰与易用。

## Request 与 Response

如果你曾经用某种语言写过 Web 应用的话，那应该对 `Request` 和 `Response` 并不陌生。一般我们将用户发送给服务器的请求称为 `Request` ，而服务器对用户返回的响应即为 `Response` 。

我们也使用 `Request` 与 `Response` 对框架进行封装，用户需要构造一个 `Request`来发送请求，并获得一个 `Response` 的实例。

`Request` 沿用之前 `fetch` 的参数，而对于 `Response` ，我们先简单的做一下响应的解析。因此，两个类的定义分别如下：

**models.py/Request**

```python
class Request(object):
    def __init__(
        self,
        method: str = None,
        url: str = None,
        headers: dict = None,
        data: str = None,
        params: dict = None,
    ):
        data = "" if data is None else data
        headers = {} if headers is None else headers
        params = {} if params is None else params

        self.method = method
        self.url = url
        self.headers = headers
        self.data = data
        self.params = params
```

**models.py/Response**

```python
class Response(object):
    def __init__(self, request, raw_content):
        self.raw_content = raw_content  # 原始的响应内容
        self.headers = {}  # 响应的 headers
        self.content = None  # 响应的正文内容
        self.status_code = None  # 响应的 HTTP 状态码
        self.request = request  # 对应的 Request
```

`Request` 与 `Response` 分别对应请求与响应的内容，我们认为一次（或同一站点的多次）与网站的交互为一次会话，请求与响应通过会话联系起来。因此为了将两个类联系起来，我们引入一个新的类 `Session` 。一个 `Session` 可能关联多个 `Request` 与 `Response` 。

**sessions.py/Session**

```python
class Session(object):
    def send(self, request: Request) -> Response:
        # 利用 fetch 函数，发送 Request 并返回 Response
```

从这个版本开始，我们的代码量已经比较大了，因此在博客中将只介绍部分代码，具体代码需要读者去 [GitHub](https://github.com/MeiK2333/ZeroCrawler) 上自行查看。我也推荐读者能自己敲一下完整的代码，从而对我们的爬虫框架有更深刻的理解。

实现了以上三个类的内容后，用户使用我们的框架的画风将是这样的：

```python
>>> from ZeroCrawler.models import Request, Response
>>> from ZeroCrawler.sessions import Session
>>> session = Session()  # 实例化 Session
>>> req = Request(method='get', url='http://httpbin.org/get')  # 实例 Request
>>> resp = session.send(req)  # 发起请求
>>> print(resp)
<Response [200]>
>>> resp.status_code
200
>>> resp.headers
{'Access-Control-Allow-Credentials': 'true', 'Access-Control-Allow-Origin': '*', 'Content-Type': 'application/json', 'Date': 'Fri, 16 Aug 2019 07:40:38 GMT', 'Referrer-Policy': 'no-referrer-when-downgrade', 'Server': 'nginx', 'X-Content-Type-Options': 'nosniff', 'X-Frame-Options': 'DENY', 'X-XSS-Protection': '1; mode=block', 'Content-Length': '146', 'Connection': 'Close'}
>>> resp.content
'{\n  "args": {}, \n  "headers": {\n    "Host": "httpbin.org"\n  }, \n  "origin": "36.110.78.251, 36.110.78.251", \n  "url": "https://httpbin.org/get"\n}\n'
```

除了 `Response` 已经帮我们把数据解析出来了以外，看起来好像并没有比以前简洁多少。别慌，让我们再加点东西。

## 人类可用的 API

> **警告**：非专业使用其他 HTTP 库会导致危险的副作用，包括：安全缺陷症、冗余代码症、重新发明轮子症、啃文档症、抑郁、头疼、甚至死亡。------ [Requests](https://2.python-requests.org//zh_CN/latest/)

首先我们为 `Session` 添加两个函数，让我们可以在 `Session` 中使用 `with` ：

```python
class Session(object):
    def __enter__(self):
        return self

    def __exit__(self, *args):
        pass
```

这让我们可以这样使用：

```python
with sessions.Session() as session:
    resp = session.send(req)
```

代码的缩进关系可以让我们更容易看出类之间的包含关系，当然这并不是我们修改的主要原因，下一章将具体介绍这段代码的作用，具体的优化在后面。

创建文件 `api.py` ，并写入如下内容：

```python
from . import sessions

def request(method: str, url: str, **kwargs):
    with sessions.Session() as session:
        return session.request(method=method, url=url, **kwargs)
```

现在我们使用它的方法变成了这样：

```python
>>> from ZeroCrawler.api import request
>>> resp = request('get', 'http://httpbin.org/get')
>>> resp.status_code
200
```

是不是很简单？我们还能让它更简单一点。在上一章中我们介绍了 HTTP 的请求方法，目前共有九种，我们可以为这九种（其实是七种，因为有两种 `CONNECT` 和 `TRACE` 是用于 HTTP 调试的）来创建“快捷方式”：

**api.py**

```python
def get(url: str, params: dict = None, **kwargs):
    return request("get", url, params=params, **kwargs)

def post(url: str, data: str = None, **kwargs):
    return request("post", url, data=data, **kwargs)

def options(url: str, **kwargs):  # ...
def head(url: str, **kwargs):  # ...
def put(url: str, data: str = None, **kwargs):  # ...
def patch(url: str, data: str = None, **kwargs):  # ...
def delete(url: str, **kwargs):  # ...
```

最终，我们的用法变成了这样：

```python
>>> from ZeroCrawler.api import get
>>> resp = get('http://httpbin.org/get')
>>> resp.status_code
200
```

简单明了，符合我们的期望。

## 总结

如果你用过 Python 中的 `requests` 库的话，会发现我们的 API 与它的 API 极其相似——因为我是照着它的 API 来写的。

ZeroCrawler 的整个设计都是借鉴的早期版本的 `requests` ，我们就是在循序渐进的造一个迷你版的 `requests` 。同样是从基础开始学习，比起跟着质量良莠不齐的各种教程，我更希望读者能一开始就跟着业内最高质量的开源项目来学习。过程中可能有很多精华在我转换的过程中丢失了，如果有遗漏之处，请读者告知。

照例，本版本的项目源码在 GitHub 上：[ZeroCrawler Version 0.1.0](https://github.com/MeiK2333/ZeroCrawler/tree/46f8803797095db0ebaad12f44a5227b09f2de5a) ，从这个版本开始，我们的 API 将基本定型，之后的更新也不会破坏之前的 API 。为了表示我们 API 已经稳定了，我们将版本从 0.0.2 直接提升到了 0.1.0 。

至此，我们的爬虫框架已经支持基础的爬虫功能了。但我们对 1.0 及之后的特性、 HTTP header 的具体行为还没有支持，之后的章节我们会依次介绍这些特性并支持它们，还请多多关注。
