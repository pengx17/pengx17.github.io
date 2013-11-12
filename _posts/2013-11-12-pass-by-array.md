---
layout: post
keywords: blog
description: blog
title: "C/C++的数组参数传递"
categories: [cpp]
tags: [cpp]
group: archive
icon: code
---
今天上午遇到个很基础但一直被xp疏忽的问题：如果在C/C++中把一个数组`int array[5]`传递给函数`void f(int a [])`的话，
`array`到底是传递的是值还是指针。
xp本以为传入`f`的参数`a`是`array`的一个拷贝，但是
经过实际编码运行测试，发现对于C++来说`array`是按指针（或是引用，根据实现不同）传入到函数`f`中的。以下面的代码为例：

```c++
#include <iostream>

void f(int a[5])
{
    a[0] = 123;
    std::cout << "address of `a` in function `f`: " << a << std::endl;
    std::cout << "sizeof `a`: " << sizeof(a) << std::endl;
    int b[5];
    std::cout << "sizeof `b`: " << sizeof(b) << std::endl;
}

int main()
{
    int a[1] = {11};
    f(a);
    std::cout << "address of `a` in main: " << a << std::endl;
    std::cout << "value of `a[0]`: " << a[0] << std::endl;
    return 0;
}
```

实际输出为（在Visual Studio 2010 32bit环境下）

```
address of `a` in function `f`: 0030FE90
sizeof `a`: 4
sizeof `b`: 20
address of `a` in main: 0030FE90
value of `a[0]`: 123
Press any key to continue . . .
```
由此可见，函数`f`中，虽然`a`和`b`看似声明方式一样，但编译器对待他们的方式是不一样的。对VS来说，`a`被实现成了一个指针而不是表达为一个数组。

在Java中，函数的参数总是以值传递。但数组在Java中比较特殊，使用数组时总是作为类似C++的引用类型处理。
因此把数组传递给Java函数时，实际上结果跟C++相同，数组的内容依然可以在函数内部被修改。

其实这部分内容十分基础，甚至显得很愚蠢，毕竟这些内容都出现在各种C/C++/Java的初级教程中单独拿出来讲过（参见[1]，[2]）。更新这篇的意思也是同时提醒自己，不要忽视语言的细节问题。

[1]: http://pages.cs.wisc.edu/~hasti/cs368/CppTutorial/NOTES/PARAMS.html
[2]: http://www.cs.utoronto.ca/~dianeh/tutorials/params/
