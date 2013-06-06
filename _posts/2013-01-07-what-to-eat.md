---
layout: post
keywords: 
description: 
title: 用优先级随机概率解决今天吃神马问题
categories: [learning, cpp]
tags: [life, cpp]
group: archive
icon: code
tldr: true
---
众所周知，决定吃饭吃什么，不管是午休同事吃饭，亦或是朋友聚餐，可是堪比“我是谁？我从哪里来？要到哪里去？”一样的哲学难题难以解决。最催人泪下的回答“随便”更是难煞了多少少男少女，引发了无数的悲欢离合。 这几天我公司的几个同事吃饭时说起了这个问题，作为程序猿，决定用‘随便’的方法写一个随机程序来解决这个世界性难题。



最直接的方式无非是，**罗列出我们可以选择的附近所有可以去的餐馆，放入一个数组，然后随机取数组其中一个餐馆**。

这种方式显然不可取，因为每个餐馆被选择的概率相同，而作为有智慧的个体我们显然是有偏好的。同事Y提出，把更想去的餐馆复制多个，放到数组中，这样大家更想去的餐馆被选中的概率就会更大一些。这种直接的办法实现起来虽很简单而且也足够用，但是有着 <del>蛋疼</del> 崇高理想的我们显然不满足于这种图样图森破的解决方案。

我提出，扩散到一般性情况，实现成精确到小数点的去设置每个餐馆的优先级数字（数字越大越容易被选中），各个餐馆优先级数字的比值代表着它们之间被选中的可能性比值。想到大学中学过的概率方法，这种情况下，我们可以通过概率累积函数的方式实现。

例如，我们有餐馆A，B，C三个，我们分别给他们优先级为3，2，1。这样三个餐馆被选中的概率分别为`p(A) = 3/6 = 0.5,p(B) = 2/6 = 0.33, p(C) = 0.17`；概率积累函数是，`P(A) = p(A) = 0.5, P(B) = P(A) + p(B) = 0.83, P(C) = P(B) + p(C) = 1`。这样我们通过随机数生成器生成一个0到1之间的小数，我们通过这个数来决定被选择的餐馆是哪个。

今天午休时间实现了一下这个程序, 代码如下（C++）

<!-- more -->

{% highlight cpp %}
//MCW's which restaurant for today?  
// by xp  
#include <iostream>  
#include <string>  
#include <vector>  
#include <ctime>  
#include <cstdlib>  
  
using namespace std;  
  
namespace  
{  
    class REST  
    {  
    private:  
        double prob; // probability to be chosen. this is calculated automatically  
        double prior;// user specified priority for this restaurant  
        double acc_low, acc_up; // lower and upper probs (accumulator)  
        string name;   
        static double all_priors;  
  
        static vector<REST> all_rests;  
        REST(double p, string s) : prior(p), name(s)  
        {  
            all_priors += p;  
        }  
    public:  
        //allow 'add' and 'wft' to modify variables  
        friend void add(double p, string s);  
        friend void wft();  
        friend void printall();  
        static void add(double p, string s)  
        {  
            all_rests.push_back(REST(p, s));  
            double acc = 0.0;  
            for(vector<REST>::iterator it = all_rests.begin();   
                it != all_rests.end(); it ++)  
            {  
                REST& curr = (*it);  
                curr.prob = curr.prior / all_priors;  
                curr.acc_low = acc;  
                acc += curr.prob;  
                curr.acc_up  = acc;  
            }  
        }  
        static void printall()  
        {  
            for(vector<REST>::iterator it = all_rests.begin();   
                it != all_rests.end(); it ++)  
            {  
                REST& curr = (*it);  
                cout
				<< curr.name 
				<< ": " << curr.prob * 100 << "%" 
				<< " (" << curr.acc_low * 100 
				<< "% ~ " << curr.acc_up * 100 << "%)" << endl;  
            }  
        }  
        static string wft() // which one for today  
        {  
            double p = (double)(rand() % 100) / 100;  
            cout << "Dice ~: " << p * 100 << "%" << endl;  
            for(vector<REST>::iterator it = all_rests.begin();   
                it != all_rests.end(); it ++)  
            {  
                REST& curr = (*it);  
                if( p > curr.acc_low && p <= curr.acc_up )  
                {  
                    cout << "今天吃 << " << curr.name << " >> !!" << endl;  
                    return curr.name;  
                }  
            }  
              
            return "something is wrong";  
        }  
    };  
}  
double REST::all_priors = 0;  
vector<REST> REST::all_rests = vector<REST>();  
  
int main(int, char**)  
{  
    std::srand(std::time(0));  
    REST::add(1,"黄房子");  
    REST::add(1,"黄房子隔壁");  
    REST::add(2,"山西面(old)");  
    REST::add(4,"山西面(new)");  
    REST::add(2,"沙县小吃");  
    REST::add(3,"香颂拉面");  
    REST::add(3,"包子");  
    REST::add(3,"驴肉火烧");  
      
    REST::printall();  
      
    REST::wft();  
    return 0;  
} 
{% endhighlight %}