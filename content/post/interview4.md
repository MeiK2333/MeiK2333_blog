---
title: 面经答案整理四
date: 2018-09-24 15:02:29
tags: [面经, C++]
---

转自牛客网[《一名渣渣C++程序员的心酸春招/秋招记》](https://www.nowcoder.com/discuss/57942)。

最近的这几篇面经里面概念性的问题比较多，这些问题其实并不适合整理下来，还有有很多问题之前的几篇里面已经解答过了，这里过滤掉这些题目。

有好多题目我直接贴了别的博客的链接，并不是我没有好好考虑过这个问题，而是我感觉我不能比我贴的博客讲的更好。

<!--more-->
# C++

## 各种排序的时间复杂度，空间复杂度，是否稳定，时间复杂度是否与初始序列有关?

稳定的排序是指：序列中相同的数字，在排序前后的相对次序不会发生改变。

| 排序 | 平均时间复杂度 | 最优时间复杂度 | 最差时间复杂度 | 空间复杂度 | 是否稳定 |
| - | - | - | - | - | - |
| 冒泡排序 | O(n²) | O(n²) | O(n²) | O(1) | 稳定 |
| 简单选择排序 | O(n²) | O(n²) | O(n²) | O(1) | 不稳定 |
| 直接插入排序 | O(n²) | O(n²) | O(n²) | O(1) | 稳定 |
| 希尔排序 |  |  |  | O(1) | 不稳定 |
| 快速排序 | O(n * log(n)) | O(n * log(n)) | O(n²) | O(1) | 不稳定 |
| 归并排序 | O(n * log(n)) | O(n * log(n)) | O(n * log(n)) | O(n) | 稳定 |
| 堆排序 | O(n * log(n)) | O(n * log(n)) | O(n * log(n)) | O(1) |  不稳定 |
| 基数排序 | O(k * n) | O(k * n) | O(k * n) | O(n + k) | 不稳定 |
| 计数排序 | O(n + k) | O(n + k) | O(n + k) | O(n + k) | 不稳定 |

## 指针和引用的区别？

- 指针是一个实体，需要分配内存空间。引用是变量的别名，不需要分配额外空间。
- 引用必须在定义的时候进行初始化，并且后续不能改变。指针定义时不需要初始化，并且指向可以改变。
- 引用只能由一级，指针可以有多级。
- 指针和引用的自增运算结果不同。
- `sizeof` 一个引用得到的是所指向的变量的空间，`sizeof` 指针的值是指针本身的大小。
- 引用访问一个变量是直接访问，指针访问一个变量是间接访问。
- 引用底层是通过指针实现的。

## 悬空指针和野指针有什么区别？

野指针指向一个已删除的对象或未申请访问受限内存区域的指针。与空指针不同，野指针无法通过简单地判断是否为 NULL避免，而只能通过养成良好的编程习惯来尽力减少。对野指针进行操作很容易造成程序错误。需对指针进行初始化，

一个指针的指向对象已被删除，那么就成了悬空指针。野指针是那些未初始化的指针。

## 什么是内存泄漏？怎么产生的？如何检测？

在计算机科学中，内存泄漏指由于疏忽或错误造成程序未能释放已经不再使用的内存。内存泄漏并非指内存在物理上的消失，而是应用程序分配某段内存后，由于设计错误，导致在释放该段内存之前就失去了对该段内存的控制，从而造成了内存的浪费。

可以 mock `malloc` 和 `free`，然后进行分析。

## static 和 const 区别？

参照我的博客[《C/C++ 中 static 与 const 的用法与区别》](/posts/static-vs-const/)。

## const 和 define 的区别？

- const 需要占用内存，define 不需要
- define 在预处理阶段展开，const 在编译阶段使用
- define 可以定义一些函数或者表达式，有很多方便的用法（比如 @、@@）

在定义常量时，const 比 define 更有优势，因为：

- const 有类型检查，define 没有
- const 便于调试，define 在预处理时被全部替换了，不利于调试

## struct 和 class 的区别？

在 C++ 中，struct 就是一个默认权限为 public 的 class，两者可以相互继承。事实上，C++ 中的 struct 也可以使用 private、public 等权限控制。

在实际编码时，使用 struct 还是 class 几乎完全没有区别，但是一般来说，如果要定义的是一种数据结构的话，那么就使用 struct，否则的话应该使用 class。

## sizeof 和 strlen 的区别？

sizeof 是运算符，值会在编译时被计算出来。在不同的情况下，其值有不同的含义：

- 当参数为数组时，值为编译时分配的数组大小
- 当参数为指针时，值为指针所占用的空间大小
- 当参数为类型时，值为该类型所占的空间大小
- 当参数为对象时，值为该对象所占用的空间大小
- 当参数为函数时，值为该函数的返回值所占用的空间大小

strlen 是函数，在执行函数时才会返回，作用是返回参数的字符串的长度。

在计算常量字符串（像 `char str[] = "Hello World!";` 这样定义的）的长度时，sizeof 的值会比 strlen 的值大 1，因为 strlen 不计最后的 '\0'。

## 32位，64位系统中，各种常用内置数据类型占用的字节数？

```C++
#include <iostream>

using namespace std;

int main() {
    cout << "bool: " << sizeof(bool) << endl;
    cout << "char: " << sizeof(char) << endl;
    cout << "unsigned char: " << sizeof(unsigned char) << endl;
    cout << "short: " << sizeof(short) << endl;
    cout << "int: " << sizeof(int) << endl;
    cout << "unsigned int: " << sizeof(unsigned int) << endl;
    cout << "long: " << sizeof(int) << endl;
    cout << "long long: " << sizeof(long long) << endl;
    cout << "float: " << sizeof(float) << endl;
    cout << "double: " << sizeof(double) << endl;
    cout << "*: " << sizeof(char *) << endl;
    return 0;
}
```

64 位系统下运行：

```bash
bool: 1
char: 1
unsigned char: 1
short: 2
int: 4
unsigned int: 4
long: 4
long long: 8
float: 4
double: 8
*: 8
```

32 位系统下，指针类型占 4 个字节。

## virtual、inline、decltype、volatile、static、const 关键字的作用？使用场景？

### virtual

声明虚函数或者虚基类。

### inline

声明函数为内联函数，inline 必须与函数定义体放在一起才能使函数成为内联，仅将 inline 放在函数声明前面不起任何作用。

inline 仅仅是个建议，如果内联函数内有类似递归等复杂操作，编译器会忽略 inline。

### decltype

decltype 类型说明符生成指定表达式的类型，常用在模板函数中，与 auto 搭配声明返回类型。

```C++
//C++11
template<typename T, typename U>
auto myFunc(T&& t, U&& u) -> decltype (forward<T>(t) + forward<U>(u))
        { return forward<T>(t) + forward<U>(u); };

//C++14
template<typename T, typename U>
decltype(auto) myFunc(T&& t, U&& u)
        { return forward<T>(t) + forward<U>(u); };
```

具体内容请参照 [MSDN](https://docs.microsoft.com/zh-cn/cpp/cpp/decltype-cpp?view=vs-2017)。

### volatile

在 C 以及 C++ 中，volatile 关键字的作用：

- 允许访问内存映射设备
- 允许在 setjmp 和 longjmp 之间使用变量
- 允许在信号处理函数中使用 sig_atomic_t 变量

## C++ 中函数指针的作用？由哪些属性唯一决定一个函数指针？

[C++中如何定义指向函数指针的指针？ - 王霄池的回答 - 知乎](https://www.zhihu.com/question/38955439/answer/79101717)

## C++ 中如何唯一确定一个重载函数？

通过参数个数与参数类型来确定一个重载函数，相同参数不同返回值的函数不能重载。

## C++ 多态的实现机制？虚函数表的内部实现机制？

C++ 中使用虚表实现了多态，虚表内使用虚表指针指向对象所属的类的虚表。

## C++ 中重载、覆盖、隐藏的区别？

[重载(overload)，覆盖(override),隐藏(hide)的区别](http://www.cppblog.com/zgysx/archive/2007/03/12/19662.html)

## 深拷贝与浅拷贝的区别？

[c++深拷贝和浅拷贝](https://blog.csdn.net/u010700335/article/details/39830425)

## 派生类中构造函数、析构函数调用顺序？

[继承中构造函数和析构函数的调用顺序](https://my.oschina.net/qihh/blog/57684)

## C++ 类中数据成员初始化顺序？

[C++类成员初始化顺序问题](https://blog.csdn.net/qq_37059483/article/details/78608375)

需要额外注意：成员变量在使用初始化列表初始化时，与构造函数中初始化成员列表的顺序无关，只与定义成员变量的顺序有关。

还需要额外注意的是：C++ 类中 `static const` 定义的变量是可以直接在声明的时候赋值的，而 C++11 之后，可以在声明普通成员变量时赋值。

## 结构体内存对齐问题？结构体/类大小的计算？

结构体的内存对齐：

1. 前面的地址必须是后面的地址整数倍，不是就补齐
2. 整个 struct 的地址必须是最大字节的整数倍

比如：

```C++
class cls1 {
    char ch1;
    char ch2;
    char ch3;
    char ch4;
    int int1;
};
cout << sizeof(cls1) << endl; // 8

class cls2 {
    char ch1;
    char ch2;
    char ch3;
    int int1;
    char ch4;
};
cout << sizeof(cls2) << endl;  // 12

class cls3 {
    char ch1;
    char ch2;
    char ch3;
    int int1;
};
cout << sizeof(cls3) << endl;  // 8
```

如果是个空类，也会占用 1 个字节。

```C++
class cls4 {
};
cout << sizeof(cls4) << endl;  // 1
```

类的普通成员函数不占用类的空间：

```C++
class cls5 {
    void func1() {}
    void func2() {}
    void func3() {}
};
cout << sizeof(cls5) << endl;  // 1
```

如果类内有虚函数的话，那么会多占用 8 个字节（32 位为 4 个字节）：

```C++
class cls6 {
    virtual void func1() {}
    void func2() {}
    void func3() {}
};
cout << sizeof(cls6) << endl;  // 8

class cls7 {
    virtual void func1() {}
    virtual void func2() {}
    virtual void func3() {}
};
cout << sizeof(cls7) << endl;  // 8
```

如果有类继承的话，那么子类的大小会加上基类的大小：

```C++
class cls1 {
    char ch;
};
class cls2: public cls1 {
    char ch;
};
cout << sizeof(cls2) << endl;  // 2
```

对于虚拟继承，除了应有的大小之外，同时还会有一个指向基类的指针：

```C++
class cls1 {
};
class cls2: virtual cls1 {
    char ch;
};
cout << sizeof(cls2) << endl;  // 8
```

## static_cast、dynamic_cast、const_cast、reinpreter_cast 的区别？

- reinterpret_cast
指针类型转换，不分青红皂白，对内容不做任何类型的检查和转换。如果转换前后的类型不能以同样的内存布局获得数据的话，那么使用将会出现问题（但是还是可以转换）。
- static_cast
可以用于子类与父类之间的互相转换，也可以用于基本类型的转换。
- dynamic_cast
只用于对象的指针和引用。当用于多态类型时，它允许任意的隐式类型转换以及相反过程。不过，与static_cast不同，在后一种情况里（注：即隐式转换的相反过程），dynamic_cast会检查操作是否有效。也就是说，它会检查转换是否会返回一个被请求的有效的完整对象。
- const_cast
这个转换类型操纵传递对象的const属性，或者是设置或者是移除。

## shared_ptr、unique_ptr、weak_ptr 的区别？auto_ptr 与 shared_ptr 的区别？weak_ptr 主要是为了解决什么问题的？shared_ptr 的内部实现？

- shared_ptr
智能指针，会通过引用计数来自动销毁。
```C++
class Cls {
public:
    Cls() {
        cout << "new cls" << endl;
    }
    ~Cls() {
        cout << "delete cls" << endl;
    }
};

void test1() {
    cout << "in test1" << endl;
    auto *cls = new Cls;
    cout << "out test1" << endl << endl;
}

void test2() {
    cout << "in test2" << endl;
    shared_ptr<Cls> cls = make_shared<Cls>();
    cout << "out test2" << endl << endl;
}

int main() {
    test1();
    test2();
    while (true);
    return 0;
}
/* 输出
in test1
new cls
out test1

in test2
new cls
out test2

delete cls
*/
```
- auto_ptr
创建一个智能指针，当出现复制构造与赋值操作时原有的指针会失效。
- unique_ptr
创建一个智能指针，行为类似 auto_ptr，但是无法进行复制构造与赋值操作。
- weak_ptr
复制和传递 shared_ptr 时使用，不会扰乱引用计数，而如果对应的智能指针被销毁，weak_ptr 也会自动变成 nullptr。

### auto_ptr 实现关键点

1. 利用特点“栈上对象在离开作用范围时会自动析构”。
2. 对于动态分配的内存，其作用范围是程序员手动控制的，这给程序员带来了方便但也不可避免疏忽造成的内存泄漏，毕竟只有编译器是最可靠的。
3. auto_ptr通过在栈上构建一个对象a，对象a中wrap了动态分配内存的指针p，所有对指针p的操作都转为对对象a的操作。而在a的析构函数中会自动释放p的空间，而该析构函数是编译器自动调用的，无需程序员操心。

## new/delete 和 malloc/free 的区别？

|特征|new/delete|malloc/free|
| - | - | - |
|分配内存的位置|自由存储区|堆|
|内存分配成功的返回值|完整类型指针|void*|
|内存分配失败的返回值|默认抛出异常|返回NULL|
|分配内存的大小|由编译器根据类型计算得出|必须显式指定字节数|
|处理数组|有处理数组的 new 版本 new[]|需要用户计算数组的大小后进行内存分配|
|已分配内存的扩充|无法直观地处理|使用 realloc 简单完成|
|是否相互调用|可以，看具体的 operator new/delete 实现|不可调用 new|
|分配内存时内存不足|客户能够指定处理函数或重新制定分配器|无法通过用户代码进行处理|
|函数重载|允许|不允许|
|构造函数与析构函数|调用|不调用|

-- 来自[《细说new与malloc的10点区别》](https://www.cnblogs.com/QG-whz/p/5140930.html)

## new operator、operator new、placement new 的区别？

[C++ 里面的new，operator new ，placement new的关系和区别](https://www.jianshu.com/p/5921001f7ad4)。

## 单例模式？懒汉式？饿汉式？

### 单例模式的定义

保证一个类仅有一个实例，并提供一个访问它的全局访问点，该实例被所有程序模块共享。

那么我们就必须保证：

1. 该类不能被复制。
2. 该类不能被公开的创造。

那么对于C++来说，它的构造函数，拷贝构造函数和赋值函数都不能被公开调用。

### 懒汉式

懒汉模式的特点是延迟加载，比如配置文件，采用懒汉式的方法，顾名思义，懒汉么，很懒的，配置文件的实例直到用到的时候才会加载，不到万不得已就不会去实例化类，也就是说在第一次用到类实例的时候才会去实例化。

### 饿汉式

饿汉式会在单例类定义的时候就进行初始化，在 main 函数之前已经完成了初始化，因此不用担心多线程的问题。

## 参考

- [Visual C++文档](https://msdn.microsoft.com) - MSDN
- [《C++多态的实现原理》](https://blog.csdn.net/tujiaw/article/details/6753498)
- [《C++的四种cast操作符的区别--类型转换》](http://www.cnblogs.com/welfare/articles/336091.html) - 博客园
- [《C++ auto_ptr智能指针的用法》](https://blog.csdn.net/MONKEY_D_MENG/article/details/5901392)
- [《细说new与malloc的10点区别》](https://www.cnblogs.com/QG-whz/p/5140930.html) - CSDN
- [《用C++实现单例模式1——懒汉模式和饿汉式》](https://blog.csdn.net/q5707802/article/details/79251148) - CSDN

# 计算机网络

## TCP粘包问题？如何解决？

TCP 是流传输协议，因此，如果想要连续发送两个包，那么这两个包可能会在一起发送，也有可能在一次传输中只传输了一部分。如果没有分开传输两个包的话，那么会对我们的接收产生影响，此时就产生了粘包问题。（UDP 有消息边界所以没有这个问题）

解决方法有：

- 每传输一次数据都重新建立一个新的 TCP 连接，发送完后立即关闭。
但是这样就和 TCP 的原则背道而驰了。
- 强行 push。
同上，效率较低
- 自行定义数据结构，比如：在每个消息头部说明消息的长度、使用特殊字符作为分割、设置定长结构等
这种方法可以利用 TCP 的效率，同时也能有效解决粘包问题，唯一的问题就是有一定技术门槛，要求操作底层数据包。

## 如何使用UDP建立可靠连接？

1. 将数据包进行编号，按包的顺序接收并存储
2. 接收端接收到数据包后，发送确认信息给发送端，发送端接收确认数据以后，再继续发送下一个包，如果接收端收到的数据编号不是期望的编号，则要求发送端重新发送

（其实就是自己实现滑动窗口 + 消息确认）

## http 协议：各个版本的区别？

1. HTTP 0.9
HTTP 协议的第一个版本，只支持 GET 请求
2. HTTP 1.0
  - 请求与响应支持头域
  - 响应对象以一个响应状态行开始
  - 响应对象不只限于超文本
  - 开始支持客户端通过POST方法向Web服务器提交数据，支持GET、HEAD、POST方法
  - （短连接）每一个请求建立一个TCP连接，请求完成后立马断开连接。这将会导致2个问题：连接无法复用，head of line blocking。连接无法复用会导致每次请求都经历三次握手和慢启动。三次握手在高延迟的场景下影响较明显，慢启动则对文件类请求影响较大。head of line blocking会导致带宽无法被充分利用，以及后续健康请求被阻塞。
3. HTTP 1.1
被广泛使用的协议版本，引入了很多关键性能优化。
  - Persistent Connection（keepalive 连接）：允许HTTP设备在事务处理结束之后将 TCP 连接保持在打开的状态，以便未来的 HTTP 请求重用现在的连接，直到客户端或服务器端决定将其关闭为止。在 HTTP1.0 中使用长连接需要添加请求头 Connection: Keep-Alive，而在 HTTP 1.1 所有的连接默认都是长连接，除非特殊声明不支持（ HTTP 请求报文首部加上 Connection: close ）。服务器端按照 FIFO 原则来处理不同的 Request。
  - chunked编码传输：该编码将实体分块传送并逐块标明长度，直到长度为 0 块表示传输结束，这在实体长度未知时特别有用(比如由数据库动态产生的数据)
  - 字节范围请求：HTTP1.1 支持传送内容的一部分。比方说，当客户端已经有内容的一部分，为了节省带宽，可以只向服务器请求一部分。该功能通过在请求消息中引入了 range 头域来实现，它允许只请求资源的某个部分。在响应消息中 Content-Range 头域声明了返回的这部分对象的偏移值和长度。如果服务器相应地返回了对象所请求范围的内容，则响应码206（Partial Content）
  - Pipelining（请求流水线）
  - 请求消息和响应消息都支持 Host 头域：在 HTTP1.0 中认为每台服务器都绑定一个唯一的 IP 地址，因此，请求消息中的 URL 并没有传递主机名（hostname）。但随着虚拟主机技术的发展，在一台物理服务器上可以存在多个虚拟主机（Multi-homed Web Servers），并且它们共享一个 IP 地址。因此，Host 头的引入就很有必要了。
  - 新增了一批 Request method：HTTP 1.1 增加了 OPTIONS、PUT、DELETE、TRACE、CONNECT 方法
  - 缓存处理：HTTP 1.1 在 1.0 的基础上加入了一些 cache 的新特性，引入了实体标签，一般被称为 e-tags，新增更为强大的 Cache-Control 头。
4. HTTP 2.0
  - 多路复用（二进制分帧）。HTTP 2.0 最大的特点：不会改动 HTTP 的语义，HTTP 方法、状态码、URI 及首部字段，等等这些核心概念上一如往常，却能致力于突破上一代标准的性能限制，改进传输性能，实现低延迟和高吞吐量。而之所以叫 2.0，是在于新增的二进制分帧层。在二进制分帧层上， HTTP 2.0 会将所有传输的信息分割为更小的消息和帧，并对它们采用二进制格式的编码 ，其中 HTTP 1.x 的首部信息会被封装到 Headers 帧，而我们的request body 则封装到 Data 帧里面。
  - HTTP 2.0 通信都在一个连接上完成，这个连接可以承载任意数量的双向数据流。相应地，每个数据流以消息的形式发送，而消息由一或多个帧组成，这些帧可以乱序发送，然后再根据每个帧首部的流标识符重新组装。
  - 头部压缩：当一个客户端向相同服务器请求许多资源时，像来自同一个网页的图像，将会有大量的请求看上去几乎同样的，这就需要压缩技术对付这种几乎相同的信息。
  - 随时复位：HTTP1.1 一个缺点是当 HTTP 信息有一定长度大小数据传输时，你不能方便地随时停止它，中断 TCP 连接的代价是昂贵的。使用 HTTP2 的 RST_STREAM 将能方便停止一个信息传输，启动新的信息，在不中断连接的情况下提高带宽利用效率。
  - 服务器端推流：Server Push。客户端请求一个资源 X，服务器端判断也许客户端还需要资源 Z，在无需事先询问客户端情况下将资源 Z 推送到客户端，客户端接受到后，可以缓存起来以备后用。
  - 优先权和依赖：每个流都有自己的优先级别，会表明哪个流是最重要的，客户端会指定哪个流是最重要的，有一些依赖参数，这样一个流可以依赖另外一个流。优先级别可以在运行时动态改变，当用户滚动页面时，可以告诉浏览器哪个图像是最重要的，你也可以在一组流中进行优先筛选，能够突然抓住重点流。

## Linux 常用网络命令的原理：ping、traceroute

ping 与 traceroute 都使用 ICMP 协议。

traceroute 的原理是：首先发送一个 TTL 为 1 的 UDP 数据报，当经过路由器时，路由器将 TTL 减一，因为 TTL 此时为 0，因此路由器丢弃这个数据报，并返回路由不可达的 ICMP 给主机。主机接收到这个 ICMP 后再发送 TTL 为 2 的 UDP，以此类推。

## DNS 的作用？什么时候使用 TCP?什么时候使用 UDP?

1. 单个 UDP 报文的最大长度为 512 字节，当单次查询的数据超过这个数值时需要使用 TCP。
2. DNS 服务器需要同步其上级服务器的数据，此时使用 TCP。
3. 其余 DNS 查询都使用 UDP。

## 参考

- [怎么解决TCP网络传输“粘包”问题？ - 知乎](https://www.zhihu.com/question/20210025)
- [《TCP粘包问题分析和解决（全）》](https://blog.csdn.net/tiandijun/article/details/41961785) - CSDN
- [《HTTP协议几个版本的比较》](https://zhuanlan.zhihu.com/p/37387316) - 知乎专栏

# 数据库

数据库这方面是我比较薄弱的地方，所以这里更多的是引用其他的博客和内容。

## MySQL 中 MyISAM 与 InnoDB 的区别？

1. InnoDB 支持事务，MyISAM 不支持，对于 InnoDB 每一条 SQL 语言都默认封装成事务，自动提交，这样会影响速度，所以最好把多条 SQL 语言放在 begin 和 commit 之间，组成一个事务
2. InnoDB 支持外键，而 MyISAM 不支持。对一个包含外键的 InnoDB 表转为 MYISAM 会失败
3. InnoDB 是聚集索引，数据文件是和索引绑在一起的，必须要有主键，通过主键索引效率很高。但是辅助索引需要两次查询，先查询到主键，然后再通过主键查询到数据。因此，主键不应该过大，因为主键太大，其他索引也都会很大。而MyISAM是非聚集索引，数据文件是分离的，索引保存的是数据文件的指针。主键索引和辅助索引是独立的
4. InnoDB 不保存表的具体行数，执行 `select count(*) from table` 时需要全表扫描。而 MyISAM 用一个变量保存了整个表的行数，执行上述语句时只需要读出该变量即可，速度很快
5. Innodb 不支持全文索引，而 MyISAM 支持全文索引，查询效率上 MyISAM 要高；

如何选择：
1. 是否要支持事务，如果要请选择innodb，如果不需要可以考虑MyISAM
2. 如果表中绝大多数都只是读查询，可以考虑MyISAM，如果既有读写也挺频繁，请使用InnoD
3. 系统奔溃后，MyISAM恢复起来更困难，能否接受
4. MySQL5.5版本开始Innodb已经成为Mysql的默认引擎(之前是MyISAM)，说明其优势是有目共睹的，如果你不知道用什么，那就用InnoDB，至少不会差

--- [Mysql 中 MyISAM 和 InnoDB 的区别有哪些？ - Oscarwin的回答 - 知乎](https://www.zhihu.com/question/20596402/answer/211492971)

## 索引？适合创建索引的条件？不适合创建索引的条件？

### 哪些情况适合建索引？

1. 主键自动建立唯一索引。
2. 频繁作为查询条件的字段应该创建索引。
3. 查询中与其他表关联的字段，外键关系建立索引。
4. 频繁更新的字段不适合创建索引，因为每次更新不单单是更新了记录还会更新索引文件。
5. where 条件里用不到的字段不创建索引。
6. 单键/组合索引的选择问题，who？（在高并发下倾向创建组合索引）。
7. 查询中排序的字段，排序字段若通过索引去访问将大大提高排序速度。（索引干两件事：检索和排序）。
8. 查询中统计或者分组字段。

### 哪些情况不适合建索引？

1. 表记录太少。
2. 经常增删改的表的字段。
3. 数据重复且分布平均的表字段，因此应该只为最经常查询和最经常排序的数据列建立索引。如果某个数据列包含许多重复的内容，为它建立索引就没太大的实际效果。

--- [索引五：哪些情况适合建索引？](https://www.jianshu.com/p/a44960298154) - 简书

## 索引优化策略？五个优化级别

[我必须得告诉大家的MySQL优化原理](https://www.jianshu.com/p/d7665192aaaf)

## 存储过程？

这个题目...啥意思...

直接放定义吧：[存储程序](https://zh.wikipedia.org/wiki/%E5%AD%98%E5%82%A8%E7%A8%8B%E5%BA%8F)

## 触发器？

[触发器](https://zh.wikipedia.org/wiki/%E8%A7%A6%E5%8F%91%E5%99%A8)

## 参考

- [Mysql 中 MyISAM 和 InnoDB 的区别有哪些？ - Oscarwin的回答 - 知乎](https://www.zhihu.com/question/20596402/answer/211492971)
- [索引五：哪些情况适合建索引？](https://www.jianshu.com/p/a44960298154) - 简书