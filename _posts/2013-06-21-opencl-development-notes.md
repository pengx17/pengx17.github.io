---
layout: post
keywords: blog
description: blog
title: "Cross-platform development notes on OpenCL(英文)"
categories: [opencl]
tags: [OpenCV, OpenCL]
group: archive
icon: file-o
tldr: true
---


# Introduction

During the days developing OpenCV’s OpenCL module, we the OpenCV group encountered numerous platform-specific challenges. **Cross-platform development on OpenCL** is not easy and we need to enumerate every combinations of a valid OpenCL context: 
1. the OpenCL solution providers, for example, AMD, nvidia, Intel and even Apple (Mac OS) have their own implementations for OpenCL standards;
2. different devices, for example, AMD/Intel iGPU’s, AMD/nvidia dGPU’s and Intel/AMD CPU’s;
3. different versions of display drivers and SDK’s;
4. bitness (x86 or x64) of the system;
5. different system platforms such as Linux, Windows and Mac OS;
6. the OpenCL version (1.1 or 1.2). 

It is our task to make sure all of them work properly eliminating platform specific differences while preserving good performance at the same time.

However this looks frustrating enough, the known cross-platform development issues, including “black screen” or “screen freeze”, program crash and various accuracy problems are usually caused by similar patterns of errors. This article is to summarize the practices we found in trials and errors while digging out cross-platform solutions. It also means to provide a guide and a standard for developers to avoid cross-platform OpenCL development pitfalls and at the same time to achieve best performance. 

Here, we publish the cross-platform development notes which may make life easier for people whether in OpenCV's OCL module development or other OpenCL related projects. In the meantime, please keep in mind that some contents maybe out-of-date due to bugfixes in new releases of OpenCL drivers/SDK.

<!-- more -->

***

<div class="alert alert-block">
  <strong>Warning!</strong>
  This article may be updated frequently in future OCL module development stages.
</div>


# Common issues and bugs

This section lists common issues when an OpenCL program behaves unexpectedly. It describes each representative issues which are caused by similar errors. Possible solutions and examples are also attached.

## Program crash caused by accessing invalid addresses


This is the most frequently common issue we encountered. Usually, a program crash is caused by accessing an invalid device memory address. The symptoms for GPU and CPU devices may not behave exactly the same, which are described below.

**On GPU devices**, when running erroneous kernels attempting to access bound-of-bound addresses the screen firstly freezes at a sudden (system loses response to user’s control), after a while screen then turns black and then screen backs to normal at last (on Windows 7 a notice will pop out saying “display driver is recovered” above the system tray icon area). Occasionally you may also notice severe blurred screen even after display driver is recovered.

While for **CPU devices**, the issue behaves exactly like a page fault in memory. It often causes the program to throw a segmentation fault error on Linux and forces the program to shutdown; on Windows, user may only notice the program quits unexpectedly (run without debugging mode) but the error is only visible when running in debugging mode and prompts an Access Violation Error dialog when error occurs.

To fix this kind of issue, our suggestion is to isolate the problematic kernel by tracing into the host code and then check which pointer dereference operation in this kernel section has out of bounds access. Most of the time the invalid accesses are near edges or there is error when calculating pointer offsets. See appendix for more information.


#### nvidia GPU: Strange `CL_OUT_OF_RESOURCES` error

When a kernel has out-of-bounds access on nvidia GPU we found that the kernel often works fine, but its subsequent kernels on the command queue may throw `CL_OUT_OF_RESOURCES` error upon being started. 


## OpenCL buffer object default value pitfall

In our experiments, on Intel and AMD platforms, OpenCL buffers will be initialized to a certain value as soon as
they are created via `clCreateBuffer`; for NVIDIA, the value is undefined and even can be a `NaN` value. 

We sometime happend to encounter a situation that on NVIDIA platform the output is not stable in iterations of kernel startups. It turns out to be a bug of unintialized buffer when kernel is not well written.

Let's look at the following code, that we want to sum a vector of integers multiplies a value `p` (just for demonstration purpose):

```c++
// input = [1000, 2000, 3000]
// output is expected to be 6
// local size  = [4, 1] 
// global size = [4, 1]
kernel void function(const global int * input, global int * output)
{
    const float p = 0.001f;
    atomic_add(output, p * input[get_global_id(0)]);
}
```

This kernel does not check out-of-bound access and it will read and add another value at position `input[3]`.
We assume `input[3]` is not initialized and the runtime default values for AMD and Intel SDKs is always smaller than 1000. On AMD and Intel, the kernel can return correct result; but for NVIDIA it is sometimes assigned with `NaN` and pollute the whole `ouput` value while in another runs the output is correct, which make it really hard to figure out where the kernel goes wrong in a complex circumstance.

Anyhow, we should as possible as we can to avoid out-of-bound read - although it did happen now and then :P

## Intel platform: OpenCL programs build failures

The issues can be reproduced on Intel OpenCL SDK 2.0 (OpenCL 1.1) and 3.0 (OpenCL 1.2). We came across two issues which cause the Intel OpenCL compiler fail to build OpenCL programs. 

We suggest to use Intel’s offline OpenCL compiler to debug compilation issues.


#### Intel compiler error: parameter may not be qualified with an address space


On Intel platform we found that OpenCL address space qualifiers like `__local` and `__constant` are not allowed for function parameters when they are arrays. For example, it is safe to define `void f(__local int * x) {/*...*/}` but NOT `void f(__local int x []) {/*...*/}` for Intel SDK’s OpenCL compiler. Thus for platform portability we may have to always use pointers instead of arrays. 

#### Image2d type support

Intel OpenCL SDK 2012 and 2013 beta has a bug that the OpenCL’s compiler fails to build when kernel file contains `image2d_t` type. This has been fixed in latest SDK (2013 release).


## Backwards portability for OpenCL 1.1

We are building OpenCV’s OCL module on top of OpenCL 1.2 full profile. However there are some deprecated APIs or new added APIs, we managed to enable OpenCV’s backwards executables portability, i.e., to make a OpenCV program compiled with OpenCL 1.2 library runnable on OpenCL 1.1 only environment.

[This link](http://streamcomputing.eu/blog/2011-11-19/difference-between-opencl-1-2-and-1-1/) explains the differences between OpenCL 1.1 and OpenCL 1.2. There are two cases of difference in use: In OpenCL 1.2, `clCreateImage2D` and `clCreateImage3D` has been merged into a single function `clCreateImage`; another difference is OpenCL 1.2 added `clEnqueueFillBuffer` for conveniently fill a buffer with a pattern, which is used in ocl function `setTo`.
In both cases, we separate codes in different code blocks guarded with `CL_VERSION_1_2` macro. Also we enable `CL_USE_DEPRECATED_OPENCL_1_1_APIS` to avoid build errors on some platforms, where deprecated APIs are defaultly disabled.


## Double precision floating point support

For some algorithms, double precision floating point is required for higher precision, e.g., `sum`, `integral` and `SURF`. Despite the fact that not all devices support double and AMD has its own low performance `cl_amd_fp64` extension, we added a function `setFloatPrecision` to let the user to override default double support behaviour. Also the kernel files are added with macros to determine whether or not to enable this feature.

## CPU specific accuracy problems

Most of the time we found that CPU’s accuracy problem is related to wavefront/warp size of a CPU. Here the term warp size means the number of synchronized threads to be executed without barriers. For CPU, [the theoretic wavefront size is 1](http://devgurus.amd.com/thread/145744), even when number of CPU cores is 4.

For example, a typical solution (local `scan`) for summing 32 numbers with 16 threads parallelly would be:

{% highlight cpp %}
__local int data [32] = {...};
int tid = get_local_size(0) * get_local_id(1) + get_local_id(0);
if(tid < 16) data[tid] += data[tid + 16]; barrier(CLK_LOCAL_MEM_FENCE);
if(tid < 8)  data[tid] += data[tid + 8];  barrier(CLK_LOCAL_MEM_FENCE);
if(tid < 4)  data[tid] += data[tid + 4];  barrier(CLK_LOCAL_MEM_FENCE);
if(tid < 2)  data[tid] += data[tid + 2];  barrier(CLK_LOCAL_MEM_FENCE);
if(tid < 1)  data[tid] += data[tid + 1];  barrier(CLK_LOCAL_MEM_FENCE);
{% endhighlight %}

When execution finishes, `data[0]` has the total sum of all 32 numbers in local memory data. To achieve best performance, on GPUs *some* of the barriers in the example above are not necessary and can be eliminated. This was true during the early development stages that we only had AMD and Nvidia GPUs whose wavefront size is at least 32 (32 for nvidia and 64 for AMD GPU’s); however for CPU’s the number is always 1, meaning that we need to synchronize each of the additions with a barrier; and even worse, we found that Intel GPU’s wavefront size may vary depending on target kernel. 

Although assuming wavefront size is 1 and adding barrier behind each scan operations can resolve this issue in general, <del>it is not performance efficient</del> *see the update below*. In implementation perspective, for convenience we add a function `queryDeviceInfo()` to query device wavefront size and let the developer to control these synchronization operations in kernels. At the moment, we have `SURF`, `HOG` and `pyrlk` using this feature. Nevertheless, a source pointed out that optimization relies on wavefront size is not portable; it is highly suggested to pass macros to determine the presence of barriers in compilation time.

**update**

On AMD APP Programming guide, it stated:

> The compiler automatically removes these barriers if the kernel
specifies a `reqd_work_group_size` (see section 5.8 of the OpenCL Specification) that is less than the wavefront size.
> Developers are strongly encouraged to include the barriers where appropriate,
> and rely on the compiler to remove the barriers when possible, rather than
> manually removing the barriers(). This technique results in more portable
> code, including the ability to run kernels on CPU devices.


## Passing arguments to kernels

A common practice is to use OpenCV’s OpenCL wrapper to call OpenCL API’s. One possible pitfall of passing arguments to kernels is shown below:

{% highlight cpp %}
//if double is not supported on the current device, we cast it to float
double f = 123.;
vector<pair<size_t , const void *> > args;
if(!support_double)
{
    float tf = (float)f;
    args.push_back(sizeof(float), (void*)&tf); // Don’t do this
}
else
{
    args.push_back(sizeof(double), (void*)&f);
}
//Dereference pointer ‘tf’ is undefined in the later context
//kernel execution may fails, as tf’s value may already be reclaimed by the OS
//...
{% endhighlight %}

Remember, the value passed to kernel is a pointer. So its content should be ensured to be valid by the developer until kernel execution starts.

## Using macros to simulate C++ templates

As far as we know, only AMD’s OpenCL sdk (since AMD APP SDK version 2.6) support [C++ templates for OpenCL C programs](http://developer.amd.com/wordpress/media/2012/10/CPP_kernel_language.pdf). We cannot rely on it as we need to maximum code portability. This, however, can be partially simulated by passing type defines to build options. We have written many kernels in this way to eliminate definition duplications, for example, add/sub, and/or/xor operators, brute force matchers, etc.

#### Pass constant macro defines instead of variadic arguments

At the time optimizing `BruteForceMatcher` and `stereobm` we attempted to add macro definitions into build options when building OpenCL programs on host. Surprisingly the simple change gave a very significant performance gain. The reason, we assumed, is that these parameters are originally acting as looping counts in kernels and thus OpenCL compiler is given a hint to unroll loops if the looping counts are constant numbers defined with macros. Similarly, this can also applied to `if`/`switch` conditions. User usually do not need to specify `#pragma unroll` statement right before a `for` clause as the compiler will do this for you automatically; in contrast if user wants to manually unroll a loop without specifying a unroll loop value, the performance can be worse when the loop count is too large that it uses too much registers, etc.

## Build error on Mac OS

On Mac OS, there are some rigorous OpenCL C syntax rules during compilation for Apple's OpenCL compiler. When writing the kernel in OpenCL, we should pay extra caution against the following cases:


#### Conditional expression and assignment statement (`?` operator)


You have to ensure that all expressions in an conditional assignment expression have the same type. Conditional variable needs explicit type casting if it has different type with the right value in a assignment statement, especially when we are working with vector data types, like `ushort2`, `float4` and so on.

In practice, we found that many OpenCL compilation errors are due to such syntax error, for example in the following example, 

`float4 res = 0 ? (float4)(1, 2, 3, 4) : (float4)(5, 6, 7, 8);`

We must explicitly convert the conditional variable ‘0’ to type `float4` to make it compile.

#### Bit operators `<<` or `>>` in macros

We found that on Mac OS `<<` or `>>` operator in #define statement is not allowed and would throw build errors if doing so. Substitute them with arithmetic operators, i.e., divisions or multiplications with 2’s. 

***

# Appendix

#### Common practices to avoid out-of-bounds access (found when debugging `stereobp`, `facedetect` and `Canny`)

When copying from host memory to device memory, the device memory is often padded with blank data to make sure the step size in bytes of each row is multiples of a constant number(which is 32 at the moment), while the height of the `oclMat` remains the same. This is to make sure data is aligned for best data accessing speed when read in row-major order. In this case, out-of-bounds access is fine when `x` offset is less than `step/elemSize()`, but for `y` offsets developers must make sure never step out of the range `[0, rows)`. We noticed that some crashes for `stereobp`, `facedetect` and `Canny` are all fixed by adding a simple clamping operation.

![clamping](/image/post/appendix_p1.png)

