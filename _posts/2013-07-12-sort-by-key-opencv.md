---
layout: post
keywords: blog
description: blog
title: "在OpenCV中实现sort_by_key"
categories: [OpenCV]
tags: [OpenCV, sort]
group: archive
icon: code
tldr: true
---
最近遇到一个问题需要在OpenCV中实现C++版本的按键值排序，以验证OpenCL实现的正确性。

按值排序的例子:

```
input:
keys   = {2,    3,   1}   (CV_8UC1)
values = {10,5, 4,3, 6,2} (CV_8UC2)
```

通过调用`sort_by_key(keys,values)`把输出变为:

```
output:
keys   = {1,    2,   3}   (CV_8UC1)
values = {6,2, 10,5, 4,3} (CV_8UC2)
```

通过搜索，网上并没有提供很好的办法。有的需要额外的[Boost库依赖][boost sort]，这就有点小题大做了。

考虑到排序的性能并没有很高要求，我就用STL的`std::sort`实现了一下。麻烦的是`cv::Mat`并不是基于模板的，所以我们无法获得如`int`, `float`这样的数据类型，而必须动态的通过调用`int cv::Mat::type()`来取得矩阵数据类型，再正确调用预先实例化的函数模板。这里，为了减少`if/else`代码量，我引入了实例化模板函数指针的数组。

代码如下：

<!-- more -->

```cpp
#include <map>
#include <functional>
#include "opencv2/core/core.hpp"

template<class key_type, class val_type>
bool kvgreater(pair<key_type, val_type> p1, pair<key_type, val_type> p2)
{
    return p1.first > p2.first;
}

template<class key_type, class val_type>
bool kvless(pair<key_type, val_type> p1, pair<key_type, val_type> p2)
{
    return p1.first < p2.first;
}

template<class key_type, class val_type>
void toKVPair(
    MatConstIterator_<key_type> kit,
    MatConstIterator_<val_type> vit,
    int vecSize,
    vector<pair<key_type, val_type> >& kvres
    )
{
    kvres.clear();
    for(int i = 0; i < vecSize; i ++)
    {
        kvres.push_back(make_pair(*kit, *vit));
        ++kit;
        ++vit;
    }
}

template<class key_type, class val_type>
void kvquicksort(Mat& keys, Mat& vals, bool isGreater = false)
{
    vector<pair<key_type, val_type> > kvres;
    toKVPair(keys.begin<key_type>(), vals.begin<val_type>(), keys.cols, kvres);
    
    if(isGreater)
    {
        std::sort(kvres.begin(), kvres.end(), kvgreater<key_type, val_type>);
    }
    else
    {
        std::sort(kvres.begin(), kvres.end(), kvless<key_type, val_type>);
    }
    
    //write back results
    key_type * kptr = keys.ptr<key_type>();
    val_type * vptr = vals.ptr<val_type>();
    for(int i = 0; i < keys.cols; i ++)
    {
        kptr[i] = kvres[i].first;
        vptr[i] = kvres[i].second;
    }
}

// this class is to statically hold specialized template function pointers
class SortByKey_STL
{
public:
    static void sort(cv::Mat&, cv::Mat&, bool is_gt);
private:
    typedef void (*quick_sorter)(cv::Mat&, cv::Mat&, bool);
    SortByKey_STL();
    quick_sorter quick_sorters[CV_64FC4][CV_64FC4];
    static SortByKey_STL instance;
};

SortByKey_STL SortByKey_STL::instance = SortByKey_STL();

SortByKey_STL::SortByKey_STL()
{
    memset(instance.quick_sorters, 0, sizeof(quick_sorters));
    //there should be total of [CV_64FC4 * CV_64FC4] number of specializations
    //but for convinience and demonstration purpose we only list the following
    instance.quick_sorters[CV_32SC1][CV_32SC1] = kvquicksort<int, int>;
    instance.quick_sorters[CV_32SC1][CV_32SC2] = kvquicksort<int, Vec2i>;

    instance.quick_sorters[CV_32SC1][CV_32FC1] = kvquicksort<int, float>;
    instance.quick_sorters[CV_32SC1][CV_32FC2] = kvquicksort<int, Vec2f>;

    instance.quick_sorters[CV_32FC1][CV_32FC1] = kvquicksort<float, float>;
    instance.quick_sorters[CV_32FC1][CV_32FC2] = kvquicksort<float, Vec2f>;

    instance.quick_sorters[CV_32FC1][CV_32SC1] = kvquicksort<float, int>;
    instance.quick_sorters[CV_32FC1][CV_32SC2] = kvquicksort<float, Vec2i>;
}

void SortByKey_STL::sort(cv::Mat& keys, cv::Mat& vals, bool is_gt)
{
    instance.quick_sorters[keys.type()][vals.type()](keys, vals, is_gt);
}

```
[boost sort]: http://stackoverflow.com/questions/9343846/boost-zip-iterator-and-stdsort


