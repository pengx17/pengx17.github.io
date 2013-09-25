---
layout: post
keywords: blog
description: blog
title: "AMD的显卡架构与OpenCL性能之间的一点思考"
categories: [OpenCL]
tags: [Archive]
group: archive
icon: file-alt
---
##Intro

对于简单OpenCL程序来说，开发者只要知道大概的OpenCL C的语法和Runtime API调用，
把`for`循环替换成kernel线程，就能使用OpenCL获得明显的加速比。
例如，矩阵的加减法运算，模版匹配算法等。这从另一方面体现出了OpenCL的强大之处：
**我们不需要详细了解运行目标设备的参数和架构，就能用统一的方法跨平台多、超线程编程，实现算法的加速。**

不过对于复杂的算法，我们发现为它分配了大量线程，做到了高度的并发，
但由于内核程序撰写质量的问题，比如GPU占用率并不高，使得离预期的加速比相差甚远，很可能是由于CPU设备上的程序(C++)写法并不适合GPU架构；
另一方面，有时候我们要为特殊的OpenCL设备进行优化，例如移动平台、AMD HD79xx系列的GCN架构等，就得充分利用目标设备的特性。

笔者的OpenCL知识主要来源于AMD APP Programming Guide [[1]]，目标硬件设备是AMD的独立显卡。
跟大部分的OpenCL教程不同，它从开始就简明的分析AMD硬件的架构，
让读者去了解OpenCL的线程是如何分配到各个GPU的核心上去的，来挖掘显卡设备潜在的强大性能。
不过由于笔者知识储备的不足，在OpenCL学习之初并没有充分理解这一部分
的内容。更惭愧的是，经过了1年的OpenCL相关工作实践，笔者并没有加深AMD显卡架构的理解，
尤其是在OpenCL核函数优化方面<del>步步惊心</del>步履维艰。

于是乎，就有了这一篇 *入门* 级别的笔记博文。同时感谢让我感受到自己不足的**南千秋**（？）老师。
***************

##非GCN架构，Evergreen and Northen Islands devices
我们先来了解一下出现在GCN之前的AMD传统显卡架构，如Evergreen和Northern Islands设备。

[To be continued ...]

##GCN架构，Southn Islands devices
Graphics Cores Next，GCN [[2]] 架构是AMD专门为通用计算设计的显卡架构，始于AMD HD79xx系列显卡，以
HD7970显卡为代表。

[To be continued ...]

***************
##参考资料
[[1]] AMD APP Programming Guide

[[2]] AMD GRAPHICS CORES NEXT (GCN) ARCHITECTURE

[1]: http://developer.amd.com/download/AMD_Accelerated_Parallel_Processing_OpenCL_Programming_Guide.pdf
[2]: http://www.amd.com/us/Documents/GCN_Architecture_whitepaper.pdf




