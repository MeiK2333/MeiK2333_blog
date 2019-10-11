---
title: Python 中的单元测试（ unittest 的基础用法）
date: 2018-02-23 00:23:00
tags: [Python, 单元测试]
---

“单元测试（unit testing），是指对软件中的最小可测试单元进行检查和验证。”—— 百度百科

<!--more-->


# 学习背景

作为一个敲代码纯靠脑补的鶸，之前在阅读某开源规范时看到，没有完备的单元测试的项目是不安全的。然而学习编程这两年多以来从来没有在自己的代码中使用过单元测试。这次学习 django ，文档中又强调了单元测试的重要性，借此机会，学习一下单元测试。

---

# 介绍

Python 中，单元测试是用来对一个模块、一个函数或者一个类来进行正确性检验的测试工作。

对于一个已有的（或者还没有的）函数或类，构造出一些输入数据，然后验证其运行的结果与我们预期的是否相同，从而判断这个模块编写是否正确。同时，如果后续修改更新了这些代码，只要简单的运行一遍单元测试，就可以知道之前的修改是否对原有功能造成了破坏。

单元测试不仅可以用于已有代码的测试，也可以用于指明开发方向。有一个著名的理念就是“测试驱动开发”，大概就是说先用单元测试描述想法，然后再去开发对应的代码。

---

# 一个小例子

比如我们现在有一个函数 `abs()` ，用于求输入的绝对值，那么我们可以构造一些测试用例：

1. 输入正数，比如 `1` 、 `1.2` 、 `0.99` ，期待返回值与输入相同；

2. 输入负数，比如 `-1` 、 `-1.2` 、 `-0.99` ，期待返回值与输入相反；

3. 输入 `0` ，期待返回 `0` ；

4. 输入非数值类型，比如 `None` 、 `[]` 、 `{}` ，期待抛出 `TypeError` 。

把上面的测试用例放到一个测试模块里，就是一个完整的单元测试。

如果单元测试通过，说明我们测试的这个函数能够正常工作。如果单元测试不通过，要么函数有bug，要么测试条件输入不正确，总之，需要修复使单元测试能够通过。

我这里直接引用廖雪峰老师的[教程](https://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/00143191629979802b566644aa84656b50cd484ec4a7838000)里面的一个例子：

---

我们来编写一个 `Dict` 类，这个类的行为和 `dict` 一致，但是可以通过属性来访问，用起来就像下面这样：

```python
>>> d = Dict(a=1, b=2)
>>> d['a']
1
>>> d.a
1
```

mydict.py 代码如下：

```python
class Dict(dict):

    def __init__(self, **kw):
        super().__init__(**kw)

    def __getattr__(self, key):
        try:
            return self[key]
        except KeyError:
            raise AttributeError(r"'Dict' object has no attribute '%s'" % key)

    def __setattr__(self, key, value):
        self[key] = value
```

为了编写单元测试，我们需要引入Python自带的 `unittest` 模块，编写 `mydict_test.py` 如下：

```python
import unittest

from mydict import Dict

class TestDict(unittest.TestCase):

    def test_init(self):
        d = Dict(a=1, b='test')
        self.assertEqual(d.a, 1)
        self.assertEqual(d.b, 'test')
        self.assertTrue(isinstance(d, dict))

    def test_key(self):
        d = Dict()
        d['key'] = 'value'
        self.assertEqual(d.key, 'value')

    def test_attr(self):
        d = Dict()
        d.key = 'value'
        self.assertTrue('key' in d)
        self.assertEqual(d['key'], 'value')

    def test_keyerror(self):
        d = Dict()
        with self.assertRaises(KeyError):
            value = d['empty']

    def test_attrerror(self):
        d = Dict()
        with self.assertRaises(AttributeError):
            value = d.empty
```

编写单元测试时，我们需要编写一个测试类，从 `unittest.TestCase` 继承。

以 `test` 开头的方法就是测试方法，不以 `test` 开头的方法不被认为是测试方法，测试的时候不会被执行。

对每一类测试都需要编写一个 `test_xxx()` 方法。由于 `unittest.TestCase` 提供了很多内置的条件判断，我们只需要调用这些方法就可以断言输出是否是我们所期望的。最常用的断言就是 `assertEqual()` ：

```python
self.assertEqual(abs(-1), 1) # 断言函数返回的结果与1相等
```

另一种重要的断言就是期待抛出指定类型的Error，比如通过 `d['empty']` 访问不存在的key时，断言会抛出 `KeyError` ：

```python
with self.assertRaises(KeyError):
    value = d['empty']
```

而通过 `d.empty` 访问不存在的key时，我们期待抛出 `AttributeError`：

```python
with self.assertRaises(AttributeError):
    value = d.empty
```

---

# 运行单元测试

通过在代码中添加 `unittest.main()` ，可以通过命令行的形式执行单元测试。也就是在写的测试文件底部添加：

```python
if __name__ == '__main__':
    unittest.main()
```
单元测试可以直接作为 Python 脚本执行：

```shell
python mydict_test.py
```

也可以在命令行通过参数 `-m unittest` 直接运行单元测试：

```shell
python -m unittest mydict_test
```

推荐使用第二种方式执行，因为通过这种方式可以同时运行多个测试文件，也可以通过添加参数获得一些其他的功能。比如添加 `-v` 选项以获得更详细的提示。

```shell
python -m unittest -v mydict_test
```

通过 `-h` 选项可以获得所有可使用的选项。 

---

# setUp与tearDown

可以在单元测试中编写两个特殊的 `setUp()` 和 `tearDown()` 方法。这两个方法会分别在每调用一个测试方法的前后分别被执行。

`setUp()` 和 `tearDown()` 方法有什么用呢？设想你的测试需要启动一个数据库，这时，就可以在 `setUp()` 方法中连接数据库，在 `tearDown()` 方法中关闭数据库，这样，不必在每个测试方法中重复相同的代码。即使测试出错， `tearDown()` 也会执行。

---

# 跳过测试以及异常测试

unittest 支持跳过单独的测试方法或者整个类，也支持条件测试或者 expected failure （期望一个错误）测试。

```python
import unittest
import sys


class TestCase(unittest.TestCase):
    # 此测试将被跳过
    @unittest.skip('demonstrating sipping')
    def test_nothing(self):
        self.fail("shouldn't happen")

    # 此测试将仅在 Windows 下执行
    @unittest.skipUnless(sys.platform.startswith("win"), 'requires Windows')
    def test_windows_support(self):
        pass

    # 此测试期望出现错误
    @unittest.expectedFailure
    def test_format(self):
        self.assertEqual(True, False)


if __name__ == '__main__':
    unittest.main()
```

被跳过的测试不会执行 `setUp()` 和 `tearDown()` 方法。

---

# 参考

本篇博客参考了以下文章：

1. 廖雪峰老师的教程：[单元测试](https://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/00143191629979802b566644aa84656b50cd484ec4a7838000)

2. 简书用户 cheneydc 的文章：[Python单元测试-unittest](https://www.jianshu.com/p/4a17a3040f07)

3. unittest [官方文档](https://docs.python.org/2/library/unittest.html)