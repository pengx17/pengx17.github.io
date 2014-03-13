---
layout: post
keywords: blog
description: blog
title: "gdb/Eclipse调试C++STL库容器的美化方法"
categories: [cpp]
tags: [cpp]
group: cpp
icon: file-o
---

最近一段时间需要在Ubuntu上做项目。为了方便开发，使用了Eclipse的C++插件来帮助调试。可是日常使用时经常遇到一个很麻烦的问题，Eclipse的调试器(也就是gdb）对C++的STL库的支持很差。比如我想查看一个`std::vector`的内容，用Visual Studio的调试器可以很方便的看到这个容器的大小和每个元素的值，微软甚至提供给[用户自定义调试器显示容器内容的方法](http://msdn.microsoft.com/en-us/library/vstudio/jj620914.aspx)；不过，默认情况下，Eclipse/gdb就会显示下面这一陀对调试用处不大的东西：

```cpp
bar {...}
    std::_Vector_base<TSample<MyTraits>, std::allocator<TSample<MyTraits> > >
        _M_impl {...}   
            std::allocator<TSample<MyTraits> >  {...}   
            _M_start    0x00007ffff7fb5010  
            _M_finish   0x00007ffff7fd4410  
            _M_end_of_storage   0x00007ffff7fd5010
```

------------

于是乎xp在SO上找到了个[解决方案][1]。这里要借助一个叫做_Python libstdc++ printers_的插件来实现美化功能。

#### 1. 安装python2.7和python-gdb

    $> sudo apt-get install python2.7
    $> sudo apt-get install gdb python2.7-dbg

#### 2. 下载Python libstdc++ printers代码。

    $> mkdir ~/python_printer
    $> cd ~/python_printer
    $> svn co svn://gcc.gnu.org/svn/gcc/trunk/libstdc++-v3/python

#### 3. 修改并添加以下脚本gdb配置文件`~/.gdbinit`，如果没有就创建一个。这个以我的为例：

```python
python
import sys
sys.path.insert(0, '/home/pengx17/python_printer/python')
from libstdcxx.v6.printers import register_libstdcxx_printers
register_libstdcxx_printers (None)
end
```

#### 4. 修改Eclipse的gdb配置文件路径。
> 修改`Run->Debug Configurations...->Debugger`的`GDB command file`为`/home/pengx17/.gdbinit`


### 完成！\o/

[1]: http://stackoverflow.com/questions/11320822/why-does-calling-method-through-null-pointer-work-in-c

