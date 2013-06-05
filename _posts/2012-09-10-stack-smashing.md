---
layout: post
description:
keywords: me
title: strcmp引申的堆栈攻击问题
categories: [learning, cpp]
tags: [cpp]
group: archive
icon: code
---
我们先来看下面的代码 （[来源于"12个有趣的C语言问答"](http://www.oschina.net/question/213217_69069)）:

<!-- more -->

{% highlight c++ %}
#include <stdio.h>  
  
int main(int argc, char *argv[])  
{  
    int flag = 0;  
    char passwd[10];  
  
    memset(passwd,0,sizeof(passwd));  
  
    strcpy(passwd, argv[1]);  
  
    if(0 == strcmp("LinuxGeek", passwd))  
    {  
        flag = 1;  
    }  
  
    if(flag)  
    {  
        printf("\n Password cracked \n");  
    }  
    else  
    {  
        printf("\n Incorrect passwd \n");  
  
    }  
    return 0;  
}  
{% endhighlight %}


这段关于密码验证的代码简单明了，除却argv\[1\]为空（用户没有给出参数）可能导致的程序崩溃问题，还有一个致命缺陷。

读者可以尝试编译以上代码并运行。当然，在参数为LinuxGeek时，控制台会正常输出显示密码正确；但是，问题出在参数的位数大于10的情况。

xp对此疑惑不解（直接说不懂C语言就好了嘛！），于是浏览此贴评论，才恍然大悟：

以下来自oschina用户vingzhang
> "flag和passwd变量是存在栈上的，栈的扩展是从高地址到低地址。压栈的时候，先压int型的flag占四个字节，然后压passwd，占十个字节。举个例子，如果0x60fe0102表示passwd开始存放的地址，那么flag开始存放的地址是0x60fe010c，高10个字节。在intel x86的平台上，字节序是小端，也就是0x60fe010c标记的那个字节的存放是flag的最低位的一个字节数据，比如，flag的值为0xffffff01，那么01就存放在0x60fe010c标记的那个字节上。

> 攻击的目的，是让flag的值为非零，就可以绕过密码的验证，让打印为“ Passwd cracked ”。所以，只要输入的字节超过10之后的那四个字节任意一个非零就可以了。"

简单来说，用户使用strcpy函数时，如果输入的数据量大于了缓冲buffer数据的额定大小，就会覆写掉末端之后的数据。在上面的情况来说，覆盖掉了flag本来的值，而这个值不为0，于是如果输入长度大于10的时候，程序总会提示"密码已破解"。

通过命令行输出，我们可以更清楚的看出passwd和flag在栈上的地址：

{% highlight c++ %}
printf("flag addr: 0x%x passwd[0] addr: 0x%x passwd[10] passwd: 0x%x \n", &flag, passwd, passwd + 10);  
//flag addr: 0x28ff0c passwd[0] addr: 0x28ff02 passwd[10] passwd: 0x28ff0c`
{% endhighlight %}

**延伸阅读** [StackSmashing](http://c2.com/cgi/wiki?StackSmashing)

栈攻击，是攻击者有意的利用溢出栈缓存的手段达到访问隐藏的计算机内存的一种恶意攻击方法。这种手段非常恶劣。栈攻击通常基于C或者C++以下的特征来达到目的：

基于栈的语言将数据和返回值地址放在同样的栈上;

语言本身允许程序越过数组边缘读取储存在栈上的数据;

语言允许储存在栈上的数据被执行;

被攻击的程序有攻击者所需要的东西：例如，root权限;

攻击者将数据加载到把数据放到栈数组的程序。数据量过大，超过了程序开始设置的额定长度大小，但是程序本身没有检测输入数据的大小，于是多余的数据会覆盖掉数组结尾之后的数据，直到覆盖掉函数返回地址的值。这样攻击者就覆写了真实的返回地址，以自己的攻击程序代替。当这个函数返回时，函数并没有返回给它的发起者，而是直接返回到了储存在栈上的攻击代码位置，这样攻击者就能让这段程序执行这个程序所能允许的任何功能了;

栈攻击的威胁可以通过写一个有一个后台运行的权限受限的daemon程序来动态检查数组长度的语言来避免。像是C语言这种不检查数组越界问题的语言，你可以通过system call来显示的分配和释放页面文件作为缓存内存区来使用，这样对于这个缓存区所有的越界操作都会被硬件捕捉到。

