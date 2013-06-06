---
layout: post
keywords: OpenCV
description: 利用OpenCV的颜色空间转换分割图片
title: 一个OpenCV颜色空间转化的实例
categories: [learning, OpenCV]
tags: [OpenCV, image segmentation, color conversion]
group: archive
icon: code
tldr: true
---
最近在OpenCV中文论坛上解答了个问题，大概问题是这样的，把下图中绿色的八卦部分抠出来：

<!-- more -->

![Input](/image/post/bagua_input.JPG)

可以看出问题解决方案很直接：遍历图片的每个像素，然后如果像素的颜色接近于绿色，保留此像素；反之遗弃，设值为0。解决思路跟photoshop的颜色选择功能类似。

问题主要的难点在于如何去比较颜色。输入图像的RGB色彩并不适用于颜色的比较；这里就要引入HSV色彩空间，把RGB颜色转化成H色相，S饱和度，V色调（亮度）。根据饼状图，我们发现绿色范围（转化成0~255区间）大概在35到90之间。所以，遍历过程中，我们留下H通道值在35~90范围内，并且饱和度和色调足够高（保证像素足够明亮）的像素；反之，剩下的我们认为是背景和噪点。

代码如下:

{% highlight c++ %}
#include <opencv2/opencv.hpp>  
#include <opencv2/opencv.hpp>  
#include <opencv2/highgui/highgui.hpp>  
  
using namespace cv;  
void getGreenMask(Mat& src, Mat& dst)  
{  
    cvtColor(src, dst, cv::COLOR_BGR2HSV);  
    int hi = 90, lo = 35;  
    int lo_sat_v = 50;  
    for(int i = 0; i < src.rows; i ++)  
    {  
        Vec3b * row_ptr = dst.ptr<Vec3b>(i);  
        for(int j = 0; j < src.cols; j ++)  
        {  
            Vec3b hsv = row_ptr[j];  
            if((hsv[1] < lo_sat_v || hsv[2] < lo_sat_v) // 饱和度和亮度过低  
                || (hsv[0] > hi || hsv[0] < lo))        // 色相不在范围内  
            {  
                row_ptr[j] = Vec3b::all(0);             // 剔除不在范围内的像素  
            }  
        }  
    }  
    cvtColor(dst, dst, cv::COLOR_HSV2BGR);  
}  
int main(int argc, char* argv[])  
{  
    Mat in = imread("F:/test.JPG");  
    namedWindow("green channel");  
    Mat out(in.size(), CV_8UC3, Scalar::all(0));  
    getGreenMask(in, out);  
    imshow("green channel", out);  
    waitKey();  
    return 0;  
}  
{% endhighlight %}
效果如下图：
![Input]({{site:url}}/image/post/bagua_output.JPG)