---
layout: post
keywords: 
description: 
title: 用OpenCV实现Photoshop曲线功能的"在图像中取样设置黑场"
categories: [learning, opencv]
tags: [life, cpp, opencv, photoshop]
group: archive
icon: code
tldr: true
---
某位朋友在实验中遇到一个任务，具体来说，就是给定一张含有细胞组织采样的显微图片，手动用photoshop去除这张图片的灰色背景，从而获得背景比较干净的图片方便细胞计数 。原始实验图片如下：

![cells-origin.PNG](/image/post/cells-origin.PNG)

<!-- more -->

这张图片中图片有明显的灰蒙蒙的背景噪声，我们的目的就是想把背景的灰色信息扔掉。

**方法一，分别对RGB各个图层的每个像素进行操作，统一的减少背景色像素点的值，让背景色接近0.**
这种方式有2个问题：所有的像素值都减少的话，图像的对比度会降低（因为获得的图片像素值范围小于0到255）；另一方面图片的整体亮度会降低；
方法三中我们会改进它，让像素值空间重新映射到0到255.

**方法二，对每个图层进行阈值化。**
经过这种方式处理后，非噪声点的像素值没有变化，但是因为像素值的断点导致图片颜色不再平滑，会出现非常多的噪点，结果十分不理想。
如下图，方法二抛弃。

![cells-threshold.PNG](/image/post/cells-threshold.PNG)

**方法三，使用Photoshop曲线功能的'在图像中取样设置黑场'。**
这个功能的文字描述看起来让人摸不着头脑。实际上操作可以看成方法一减背景法的改良版。通过曲线窗口的结果我们可以看出，这个功能就是帮我们将减去背景色的像素进行了一个重新映射。
具体来说就是，点击一个像素点后，记录这个像素点的RGB值，对全图进行处理。三通道分别操作，每个低于这个点相应值的，就设成0；其他的点等比例拉伸成0~255范围。
从数学上来看，默认的曲线是 `y = x`, 所以曲线是一条对角线；进行了这个操作之后，我们变化了这条曲线的斜率，但依然让值域扩展到0到255。注意，进行曲线调整后，大部分像素值还是会降低，所以获得的结果依然比输入的图片要暗一些。

![cells-curve.PNG](/image/post/cells-curve.PNG)

有了上述分析，编写代码的方法就很清晰了，代码附录。为了方便朋友使用，加入了一些用户交互操作的代码。
这些功能包括:

- 循环读取当前目录中的.tif图片
- 显示当前图片，支持手动选择背景像素点
- 在另一个窗口加入3个跟踪控制条，用来手动调节各通道背景像素的值
- 7个操作按键，分别是：
  - q: 退出程序
  - r: 重置当前图片，包括控制条的位置和背景像素值
  - 1, 2, 3, 4: 显示RGB图像，或者是单独显示单图层图像。注意，显示单图层图像时，图像是黑白的。
  - 空格: 保存当前结果到result目录，并跳转到下一张图片。

{% highlight c++ %}
#include <stdio.h>  
#include <iostream>  
#include <windows.h>  
#include <tchar.h>  
#include <stdio.h>  
   
#include "opencv2/imgproc/imgproc.hpp"  
#include "opencv2/highgui/highgui.hpp"  
   
using namespace cv;  
using namespace std;  
   
const string winName = "Image", winCtrl = "control panel", rBar = "R", gBar = "G", bBar = "B";  
   
Mat oimg, img, smat, lut;  
const int default_bpos = 100;  
int sR = default_bpos, sG = default_bpos, sB = default_bpos;  
int thresR = 0, thresG = 0, thresB = 0;  
float scale = 0.5f;  
   
// reset trackbar positions and background pixel thresholds  
void reset()  
{  
    thresR = 0;  
    thresG = 0;  
    thresB = 0;  
    setTrackbarPos(rBar, winCtrl, default_bpos);  
    setTrackbarPos(gBar, winCtrl, default_bpos);  
    setTrackbarPos(bBar, winCtrl, default_bpos);  
}  
   
// construct lookup table for input r, g, b value   
void makeLUT(Mat& lut, int r, int g, int b)  
{  
    printf("r: %d g: %d b: %d\n", r, g, b);  
    lut.create(1, 256, CV_8UC3);  
    for(int i = 0; i < 256; i ++)  
    {  
        Vec3b& bgr = lut.at<Vec3b>(i);  
        if(i < b)  
        {  
            bgr[0] = 0;  
        }  
        else  
        {  
            bgr[0] = saturate_cast<uchar>((i - b) * 255 / (255 - b));  
        }  
        if(i < g)  
        {  
            bgr[1] = 0;  
        }  
        else  
        {  
            bgr[1] = saturate_cast<uchar>((i - g) * 255 / (255 - g));  
        }  
        if(i < r)  
        {  
            bgr[2] = 0;  
        }  
        else  
        {  
            bgr[2] = saturate_cast<uchar>((i - r) * 255 / (255 - r));  
        }  
    }  
}  
static void renew_window(int value = 0, void * ptr = 0);  
   
// handle trackbar position change  
static void renew_window(int, void *)  
{  
    makeLUT(lut, thresR - sR + default_bpos, thresG - sG + default_bpos, thresB - sB + default_bpos);  
    LUT(oimg, lut, img);  
    resize(img, smat, Size(), scale, scale);  
    imshow(winName, smat);  
}  
   
static void onMouse( int event, int x, int y, int, void* )  
{  
    if( event != CV_EVENT_LBUTTONDOWN )  
        return;  
    // input image is scaled down for ease of usage, but we need to scale up to get the actual point position  
    float fx = 1.0f / scale * x;  
    float fy = 1.0f / scale * y;  
    Point point = Point((int)fx,(int)fy);  
    Vec3b bgr = oimg.at<Vec3b>(point);  
    thresR = bgr[2];  
    thresG = bgr[1];  
    thresB = bgr[0];  
    renew_window();  
}  
   
// use mixChannels to extract the interest image plane  
void show_plane(int key)  
{  
    int ch[] = {0, 0};  
    Mat plane(smat.size(), CV_8U);  
    string txt;  
    switch(key)  
    {  
    case '1':  
        smat.copyTo(plane);  
        txt = "RGB image";  
        break;  
    case '2': //r  
        ch[0] = 2;  
        mixChannels(&smat, 1, &plane, 1, ch, 1);  
        txt = "R plane";  
        break;  
    case '3': //g  
        ch[0] = 1;  
        mixChannels(&smat, 1, &plane, 1, ch, 1);  
        txt = "G plane";  
        break;  
    case '4': //b  
        ch[0] = 0;  
        mixChannels(&smat, 1, &plane, 1, ch, 1);  
        txt = "B plane";  
        break;  
    }  
    putText(plane, txt, Point(50, 50), FONT_HERSHEY_COMPLEX, 0.5, Scalar::all(255), 1, CV_AA);  
    imshow(winName, plane);  
}  
   
void help()  
{  
    printf ("==================================================\n");  
    printf ("[Background remover]\n");  
    printf ("All input images are scaled down to %2.1f%%\n", scale * 100);  
    printf ("Hint:\n");  
    printf ("  q: quit\n");  
    printf ("  r: reset current image\n");  
    printf ("  1: show orignal image\n");  
    printf ("  2: show 'R' plane\n");  
    printf ("  3: show 'G' plane\n");  
    printf ("  4: show 'B' plane\n");  
    printf ("  spacebar: save current and proceed to the next image\n");  
    printf ("==================================================\n\n");  
}  
   
void main(int , char **)  
{  
    help();  
    WIN32_FIND_DATA FindFileData;  
    HANDLE hFind;  
   
    hFind = FindFirstFile("./*.tif", &FindFileData);  
    if (hFind == INVALID_HANDLE_VALUE)   
    {  
        printf ("No tif file found!\n");  
        return;  
    }  
   
    namedWindow( winCtrl, CV_WINDOW_NORMAL );  
    namedWindow( winName, CV_GUI_NORMAL | CV_WINDOW_AUTOSIZE );  
    CreateDirectory("./result", NULL);  
   
    setMouseCallback( winName, onMouse, 0 );  
    Mat dummy_img(1, 300, CV_8U, Scalar::all(255));   
   
    createTrackbar( rBar, winCtrl, &sR, 200, renew_window);   
    createTrackbar( gBar, winCtrl, &sG, 200, renew_window);   
    createTrackbar( bBar, winCtrl, &sB, 200, renew_window);   
    imshow(winCtrl, dummy_img);  
   
    moveWindow(winName, 0, 0);  
    moveWindow(winCtrl, 0, 0);  
    int key = 0;  
    int i = 0;  
    do  
    {  
        _tprintf (TEXT("Attemp %d: %s\n"),   
            i + 1, FindFileData.cFileName);  
        oimg = imread(FindFileData.cFileName);  
        if( oimg.empty() )  
        {  
            cout << "Failed to load " << FindFileData.cFileName << endl;  
            continue;  
        }  
_reset:  
        reset();  
        oimg.copyTo(img); // we need this if the user press spacebar straightaway  
        renew_window();  
        key = 0;  
_wait:  
        switch(key)  
        {  
        case ' ':  
            break;  
        case 'r':  
            goto _reset;  
        case 'q':  
            goto _close;  
        case '1':  
        case '2':  
        case '3':  
        case '4':  
            show_plane(key);  
        default:  
            key = waitKey();  
            goto _wait;  
        }  
        string save_name = string("./result/") + string(FindFileData.cFileName) + ".jpg";  
        imwrite(save_name, img);  
        i ++;  
    } while( FindNextFile(hFind, &FindFileData) );  
_close:  
    FindClose(hFind);  
}  
{% endhighlight %}