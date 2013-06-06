---
layout: post
keywords: 学习Chromium
description: 
title: Chrome学习整理
categories: [learning, chromium]
tags: [chromium]
group: archive
icon: book
tldr: true
---

由于工作需要我需要分析Chrome的图片解码流程。众所周知，Chrome是建立在开源的Chromium项目上的。最近的一个多月时间里，我基本把所有心思都放在Chromium上了，而且不得不说，学习并分析开源项目的代码对一个程序员的提高确实蛮大的。这篇博文我会记录一下学习过程中我遇到的一些问题，并分享学习中我所参考的几篇优秀的Chromium代码分析文章。

<!-- more -->

## 构建的一点吐槽和官方的方法文档摘录

不看不知道，编译好的Chromium可执行文件虽然只有40+M，但打包好的源码文件包竟然高达3G之多（也许是我孤陋寡闻了，但是这个反差确实让我吓我一跳）。
Chromium的构建方法就不多说了，推荐 [官方安装帮助文档](http://www.chromium.org/developers/how-tos/build-instructions-windows/index.html)。

值得一提的是，一定要按照他所提供的办法严格的执行，不然可能会遇到各种编译错误。编译过程很可能会长达数小时之久 (在我的笔记本上构建了一整晚才完成)。为了加速编译过程，推荐使用多核CPU，64位windows，至少8G内存和60G硬盘空间，这些要求应该都不难达到。还有其他的几点：

- 尽量使用Visual Studio 2010 SP1 版本，不然precompiled headers模式可能会出现很多问题。由于Chromium中有大量的头文件，使用此选项大概能加速25%。
- 关闭杀毒软件，或者把.ilk, .pdb, .cc, .h文件加入白名单。再或者，把Chromium源代码目录加入杀毒软件白名单。
- 在一个没有虚拟内存交换的硬盘上构建Chromium。或者固态硬盘，如果可能的话。
- 使用ninja构建Chromium。这是Google专门为Chromium项目开发的构建工具链，笔者没有尝试，文档称使用ninja会让incremental linking过程快很多。
- 修改Visual Studio选项，减少同时链接的项目数量，以减少Chromium链接时所需要的内存。因为Chromium链接需要大量的内存，如果内存不足，大量的分页操作会大大降低系统的反应速度。"maximum number of parallel project builds"可以在Tools/Options/Projects and Solutions/Build and Run页面找到。这个选项可以减少Visual Studio启动的VCBuildHelper.exe实例数目。
- 修改include.gypi文件，减少同时执行的cl.exe的数目。默认情况下，Visual Studio会开启/MP选项，动用系统一切资源生成多个编译进程加速编译。但是由于Chromium项目过大，最差情况下一个cl.exe（链接器）可能会占用大约1G内存。

## Visual Studio中调试Chromium浅谈

分析Chromium源码，为了快速的在代码海洋中找到自己所需要的部分，不可避免的要调试的Chromium项目。比如说，我们想知道，Chromium是如何从页面上下载一张图片，经过渲染，最后显示在页面上的呢？这个流程实际上的要比看起来复杂不少：一个页面内的WebKit分析出DOM树，发现一个`<img>`元素里的图片需要显示；WebKit发送一个URL下载的IPC消息给主进程Browser Process，然后WebKit根据返回数据的MIME类型标签进行解析，发现这个资源是个图片，再根据编码方式调用相应的WebCore::ImageDecoder类进行解码，最后还要进行一系列的渲染，把需要绘制的东西交给RenderWidget，这样我们才能看到图片。这个过程如果单纯的去浏览代码找到对应的类和方法调用关系，效率低下不说，还很容易出错，这时候我们就需要在Chromium中进行代码调试和追踪了。

有个关键问题，Chromium不同于普通的单进程程序，默认情况下它是多进程模式的。比如，新打开的只有一个首页的Chrome程序，至少有3个相关进程，分别是Browser进程（管理所有的UI框架，全局消息传递，资源下载等等），GPU进程（提供WebGL渲染，绕过沙盒机制调用3D API），和渲染进程Renderer（包括首页响应用户操作的content和负责渲染的WebKit实例等等），甚至还可能还有其他的进程外运行的插件进程，等等。具体的Chrome进程模型还有不同进程之间如何用IPC消息进行交互的，请参考本文推荐的阅读源。如果直接通过visual studio进行调试，只有主进程（Browser）被调试，而负责渲染的renderer进程则像是其他程序一样，表示一切跟我没有关系。最直观的表现就是，使用Chrome浏览器时，Windows任务管理器中会出现多个chrome.exe进程实例。

**调试多进程的Chromium**，其实利用visual studio自带的工具就完全可以进行: 调试开始后，只要把已经运行起来的子进程附加到debugger上就可以了。具体操作是，开始调试，选择`Tools > Attach to Process`， 然后按住ctrl选择多个你想要调试的chrome.exe进程，附加到debugger上。另外，最好用debug构建的Chromium进行调试，不然release的编译器优化问题会产生一些很诡异的情况（比如取不到特定变量的值，跳过了一些被优化过的函数什么的。。。）
附加进程到debugger的界面如图所示：

![processes-chromium.PNG](/image/post/processes-chromium.PNG)

附加到debugger后，你可以在process窗口看到正在调试的进程:

![debugger.PNG](/image/post/debugger.PNG)

另外，刚才说过，Chromium默认情况下是多进程模式（Process-per-site-instance，一个网站打开的所有页面属于同一进程）。你还可以提供运行时参数--single-process，强制让Chromium只用一个进程运行。这种模式可以通过传统的方式进行调试，但是这种模式属于实验性质的，可能会有这样和那样的bug出现，导致页面崩溃。

Chromium project官网还提供了多个调试办法，不过直接利用visual studio的附加到进程方式最简便。http://dev.chromium.org/developers/how-tos/debugging

不过实际开发调试中，开发者并不需要每次都要打开庞大的Chromium项目。Chromium源码目录还包含了2个供测试开发用的项目，分别是content_shell和test_shell，默认情况下这两个项目可以从src\content\content.sln和src\webkit\webkit.sln中打开。test_shell主要展示了chromium webkit API，包含了大部分的webkit渲染引擎核心的chrome移植接口（不包括HTML5，GPU加速，沙箱模型），注意，它是单进程的。对于content_shell，这个程序主要用来展示content shell API，它几乎包含了所有的浏览器的功能，像是一个没有华丽Chrome外壳的浏览器。跟test_shell不同，content_shell是多进程的，进程模型与chromium相同。如果你愿意，完全可以在content_shell上加一个外壳变成自己自主开发的浏览器，就像是360那样。。。开发者可以根据自己的需要（比如，单独测试Webkit部分，或是要研究不同浏览器部件之间如何进行交互）选择不同的程序调试或开发。

下图是一个content_shell在windows下运行的效果。注意，这个东西处理HTTP请求很慢（content API并不包括完整的资源下载功能，Chrome的所有资源下载都是通过Browser进程负责的），最好只用来测试本地的页面。


![content-shell.PNG](/image/post/content-shell.PNG)

## 推荐阅读和引用来源

从零开始分析源码来了解Chromium项目是十分困难的。好在Google为我们提供了不少有用的文章，具体可以从 [The Chromium Project->For Developers->How-Tos](http://www.chromium.org/developers/how-tos/) 搜索。不过这些文章有些晦涩难懂，二来是英语的，阅读起来会对我等英语渣渣造成困难；这里推荐以下几个中文博客：

- [Chrome源码剖析 1~5](http://www.cnblogs.com/duguguiyu/archive/2008/10/02/1303095.html) by duguguiyu。这位大神在08年Chromium刚公布源码后就进行了系统化并且通俗易懂的源码分析（说实话帮了我大忙~谢谢大神）。文章没有从代码上进行详细说明，而是从更高一层的设计层面分析了Chrome的不同组件，这是更难能可贵的。具体内容包括了Chromium的多线程任务模型、多进程浏览页模型，Chrome内的IPC消息传递机制，Chrome的图形渲染绘制流程。
- [Chrome源码阅读](http://blog.csdn.net/zero_lee/article/category/1212479) by zero_lee。这位作者主要从代码层面分析了多个Chrome内部类的作用和实现技巧，对于代码的理解很有帮助。
- [Chromium研究](http://mogoweb.net/categories/chromium-research) by chen.zhengyong。作者把眼光放在了Linux系统和嵌入式设备上的Chrome的开发和实现上。里面有很多绘制精美的UML图，值得一看。
- [Chrome扩展开发文档中文版](http://open.chrome.360.cn/extension_dev/overview.html) by 360。360公司翻译的Chrome扩展开发文档，如果要写Chrome扩展（不是插件，扩展和插件在Chrome中是两种东西）大概会有用。

先写这些，如果遇到比较好的文章我会进行补充。