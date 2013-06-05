---
layout: post
keywords: me
description: atomic add for floating point in OpenCL
title: 在OpenCL中实现浮点数的原子加法运算
categories: [HPC, OpenCL]
tags: [OpenCL, CUDA, 翻译]
group: archive
icon: code
---
今天在OpenCL的开发过程中遇到了对浮点数的原子运算(atomic operations)的问题。OpenCL spec中只提供了对于32位或64位整数的原子运算；对于浮点数，我们就得另辟蹊径了。

<!-- more -->

因为OpenCL在语法上跟CUDA非常类似，我们可以参考一下[CUDA C Programming Guide](http://developer.nvidia.com/cuda/nvidia-gpu-computing-documentation)上面关于浮点数原子加法的例子，如下：

{% highlight c++ %}
__device__ double atomicAdd(double* address, double val)  
{  
  unsigned long long int* address_as_ull = (unsigned long long int*)address;  
  unsigned long long int old = *address_as_ull, assumed;  
  do {  
    assumed = old;  
    old = atomicCAS(address_as_ull, assumed,   
                    __double_as_longlong(val + __longlong_as_double(assumed)));  
  } while (assumed != old);  
  return __longlong_as_double(old);  
}
{% endhighlight %}

atomicCAS是cuda中一个把compare和swap组合起来的函数。对应的OpenCL函数是atom_cmpxchg。

有经验的读者会注意到cmpxchg也存在于Intel的汇编指令集，而这条指令常常用来实现琐无关的线程等待机制。
具体可参考：[锁无关的(Lock-Free)数据结构——在避免死锁的同时确保线程继续](http://blog.csdn.net/pongba/article/details/588638)。

转化成OpenCL中的内联函数，float版本：

{% highlight c++ %}
inline void AtomicAdd(volatile __global float *source, const float operand) {  
    union {  
        unsigned int intVal;  
        float floatVal;  
    } newVal;  
    union {  
        unsigned int intVal;  
        float floatVal;  
    } prevVal;  
    do {  
        prevVal.floatVal = *source;  
        newVal.floatVal = prevVal.floatVal + operand;  
    } while (atomic_cmpxchg((volatile __global unsigned int *)source,   
                             prevVal.intVal, newVal.intVal)   
                             != prevVal.intVal);  
}  
{% endhighlight %}

对于乘法和除法，可以把其中关键运算的那一行替换

{% highlight c++ %}
newVal.floatVal = prevVal.floatVal + operand;
{% endhighlight %}

替换为

{% highlight c++ %}
AtomicMul(): newVal.floatVal = prevVal.floatVal * operand; //乘法  
AtomicMad(source,operand1,operand2): newVal.floatVal = mad(operand1,operand2,prevVal.floatVal); //乘后相加  
AtomicDiv(): newVal.floatVal = prevVal.floatVal / operand;  //除法  
{% endhighlight %}

不过，浮点数的原子运算效率非常低，所以实际应用中应尽量避免。
来源

1. [OpenCL 1.1: Atomic operations on floating point values](http://suhorukov.blogspot.com/2011/12/opencl-11-atomic-operations-on-floating.html)

2. [OpenCLで浮動小数のatomic addをしたい](http://d.hatena.ne.jp/aont/20110627/1309192561)
