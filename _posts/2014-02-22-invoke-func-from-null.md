---
layout: post
keywords: blog
description: blog
title: "调用空指针对象函数"
categories: [cpp]
tags: [cpp]
group: cpp
icon: file-o
---

前两天遇到一个挺有意思的问题：

> 已知有一个`class A`的实例，`A`有一个函数`func`，但不知道`A`的具体声明和定义。
> 如果我们有一个`A`的`NULL`指针`A *a = NULL`，如果调用`a->func()`的话，可能会出现什么情况呢？

先不管调用空指针是否是**未定义行为**。我们从C++语言本身角度去考虑，这样调用是有可能不抛出异常的。

我总结了几个不同的情况，如下(Visual Studio 2012):


```cpp
#include <iostream>
#include <Windows.h>
#include <exception>

using namespace std;

class A
{
public:
    void func()
    {
        cout << "wtf?" << endl;
    }
    void func_this()
    {
        cout << "wtf: " << this->data << endl;
    }
    static void func_static()
    {
        cout << "static wtf?" << endl;
    }
    virtual void func_virtual()
    {
        cout << "virtual wtf?" << endl;
    }
    A():data(0){}
    int data;
};

int main()
{
    A *a = NULL;
    a->func();
    __try
    {
        a->func_this();
    }
    __except(EXCEPTION_EXECUTE_HANDLER)
    {
        cout << "cannot invoke func_this" << endl;
    }
    a->func_static();
    __try
    {
        a->func_virtual();
    }
    __except(EXCEPTION_EXECUTE_HANDLER)
    {
        cout << "cannot invoke func_virtual" << endl;
    }
    return 0;
}
```

命令行输出结果为:

```
wtf?
cannot invoke func_this
static wtf?
cannot invoke func_virtual
```


####分析

我们来依次分析一下能正常运行的`func()`和`func_static()`:

*  调用`func()`函数时`A`指针不是必须的。在编译时，`A`类型已知，`func()`函数指针已经可以确认了。
*  同理，调用静态函数`func_static()`也不需要实际的实例对象。

对于抛出异常的`func_this()`和`func_virtual()`:

*  `func_this()`用到了`this`指针，而`this`在这样的情况下是`NULL`，所以会抛出异常。
*  而调用虚函数`func_virtual()`时，我们需要一个可用的虚函数表(`vtable`)指针，但显然这个指针是拿不到的，因此抛出异常。

不过，实际开发中要尽量避免这种情况哟。

####参考

[Why does calling method through null pointer “work” in C++?](http://stackoverflow.com/questions/11320822/why-does-calling-method-through-null-pointer-work-in-c)

[C++, __try and try/catch/finally](http://stackoverflow.com/questions/7049502/c-try-and-try-catch-finally)
