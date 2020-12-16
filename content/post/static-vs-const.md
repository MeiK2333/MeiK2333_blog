---
title: C/C++ 中 static 与 const 的用法与区别
date: 2018-09-24 17:15:59
tags: [C 语言, C++]
---

const 关键词指定对象或变量不可修改，static 关键词指定对象或变量为静态的。两者可以分别使用也可以结合使用，这篇博客就来探讨一些这两个关键词的用法。

<!--more-->

## const 的用法

const 关键字指定变量的值是常量，并通知编译器防止程序员修改它。比起 `#define` 来说，const 有类型检查。

在 C语言中，const 修饰的常量值默认为外部链接，因此它们只能出现在源文件中。 在 C++ 中，常量值默认为内部链接，这使它们可以出现在头文件中。

### 修饰变量与函数

当修饰变量时，这个变量无法进行修改。而当修饰函数时，代表这个函数的返回值无法进行修改。

要额外注意一下，在修饰指针变量时的用法：

```C++
const char *string1 = "Hello World!";
// *string1 = 'h';  错误，string1 指针所指向的内容无法修改
string1 = "other string";

char const *string2 = "Hello World!";
// *string2 = 'h';  错误，这种写法和第一种相同
string2 = "other string";

char *const string3 = "Hello World!";
string3[0] = 'h';
// string3 = "other string";  错误，string3 指针的指向无法修改

const char * const string4 = "Hello World!";
// 这种写法既不能修改指针指向，也不能通过这个指针修改指向的内容
```

### 修饰类的成员函数

const 可以修饰基本类型如 char 等，也可以修饰自己创建的类的对象。在 const 修饰类的对象时，必须保证被修饰的对象无法对类中的内容作出修改。C++ 中规定，被 const 修饰的对象只能调用类里的 const 成员函数。

声明类的 const 成员函数与声明 const 变量不一样（如果一样放在前面的话，就会变成修饰返回值了），在修饰类的成员函数时，const 应当在函数名的后面，如：

```C++
class Date {
public:
    Date(int mn, int dy, int yr) {
    }

    int getMonth() const;     // A read-only function
    void setMonth(int mn);   // A write function; can't be const
private:
    int month = 0;
};

int Date::getMonth() const {
    return month;        // Doesn't modify anything
}

void Date::setMonth(int mn) {
    month = mn;          // Modifies data member
}

int main() {
    Date MyDate(7, 4, 1998);
    const Date BirthDate(1, 18, 1953);
    MyDate.setMonth(4);    // Okay
    BirthDate.getMonth();    // Okay
    BirthDate.setMonth(4); // C2662 Error
}
```

示例代码来自 MSDN，能够看出来，被 const 修饰的对象无法调用非 const 的成员函数，而在 const 修饰的成员函数中，也不能调用非 const 的成员函数，或者修改任何非静态的数据成员。

## static 的用法

static 用于创建一个静态持续时间的变量，被 static 修饰的变量会在静态区中（全局变量也在静态区中），在整个程序的生存时间内都存在。

如果是在函数内定义的 static 变量，那么即使在函数外，这个变量依旧存在（但只能在其作用域内使用）。

未赋值的 static 变量会被被初始化为 0，static 修饰的变量具有内部链接性，无法在其他文件使用。如果用 static 修饰一个全局变量的话，那么它的作用就是限制这个变量只能在当前文件（模块）使用。

```C++
#include <iostream>

using namespace std;

void showstat(int curr) {
    static int nStatic;    // Value of nStatic is retained
    // between each function call
    nStatic += curr;
    cout << "nStatic is " << nStatic << endl;
}

int main() {
    for (int i = 0; i < 5; i++)
        showstat(i);
}
```

```bash
nStatic is 0
nStatic is 1
nStatic is 3
nStatic is 6
nStatic is 10
```

依旧是 MSDN 的示例代码，这个代码就已经很好的表明了 static 的一些特性。

在类中使用 static 变量：

```C++
#include <iostream>

using namespace std;
class CMyClass {
public:
   static int m_i;
};

int CMyClass::m_i = 0;
CMyClass myObject1;
CMyClass myObject2;

int main() {
   cout << myObject1.m_i << endl;
   cout << myObject2.m_i << endl;

   myObject1.m_i = 1;
   cout << myObject1.m_i << endl;
   cout << myObject2.m_i << endl;

   myObject2.m_i = 2;
   cout << myObject1.m_i << endl;
   cout << myObject2.m_i << endl;

   CMyClass::m_i = 3;
   cout << myObject1.m_i << endl;
   cout << myObject2.m_i << endl;
}
```

```bash
0
0
1
1
2
2
3
3
```

在类的成员函数中使用 static 变量：

```C++
#include <iostream>
using namespace std;
struct C {
   void Test(int value) {
      static int var = 0;
      if (var == value)
         cout << "var == value" << endl;
      else
         cout << "var != value" << endl;

      var = value;
   }
};

int main() {
   C c1;
   C c2;
   c1.Test(100);
   c2.Test(100);
}
```

```bash
var != value
var == value
```

## 参考

- [Visual C++文档](https://msdn.microsoft.com) - MSDN