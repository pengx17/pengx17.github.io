---
layout: post
keywords: blog
description: blog
title: "简易facedetect库"
categories: [OpenCV]
tags: [Archive]
group: archive
icon: file-o
---

这两天做了一个Cascade Classifier人脸检测的项目，放到了[Github][1]上。

主要功能有：

1.  预先载入进内存的cascade文件
2.  读取I420图片和视频的一些小工具
3.  API设计成用户直接给定的图片数据的原始指针/图片颜色格式/图片大小
4.  OpenCL支持


实现起来很直接，不过有几点挺有意思的，比较值得注意一下：

####xml2header.cmake

cascade文件预读进内存的思想是用项目里的`xml2header.cmake`脚本处理cascade的`xml`文件，生成一个含有长字符串的`.h`头文件，然后`.cpp`文件引用它。
这里有个问题就是，有的xml文件很大，比如常用的`haarcascade_frontalface_alt.xml`。这个文件如果直接编译成一个静态的长字符串，编译器很可能会出错。因此，我在cmake脚本里对这个文件切割成几个小的`std::string`，然后在程序初始化时用`std::accumulate`函数再组成完整的cascade字符串。另外要注意，读取成字符串的时候要把文件中的`\`和`\\`转化成`\\`和`\\\\`，每一行结尾要再加一个`\n`。

####读取视频源

测试中我使用了两种I420视频源，一种是有header的`.y4m`格式，一种是没有header的`.yuv`格式文件。对于`.y4m`，我们可以参考[网上对于y4m格式的介绍][3]来逐帧读取。

####从内存中读取cascade字符串
处理cascade字符串时，我们可以用`FileStorage`创建一个流，然后给OpenCV的`cv::CascadeClassifier`类的`read`使用。不过实现过程中我发现`read`函数只支持新的Cascade文件 - 通过`traincascade`训练而来的，参考[OpenCV API的文档][4]- 为了绕过这一点，我重写了`load`函数的其中一小部分，这样老的cascade文件也能从内存里读取了。


[1]: http://pengx17.github.io/facedetect
[2]: https://github.com/pengx17/facedetect/blob/master/script/xml2header.cmake
[3]: http://wiki.multimedia.cx/index.php?title=YUV4MPEG2
[4]: http://docs.opencv.org/modules/objdetect/doc/cascade_classification.html#CascadeClassifier