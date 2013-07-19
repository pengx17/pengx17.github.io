---
layout: post
keywords: 学习Chromium
description: 
title: 用Prolog完成Greplin challange
categories: [learning, prolog]
tags: [prolog]
group: archive
icon: code
tldr: true
---
去年在Imperial College学习中我选修了非常有趣的Prolog课程。从名字（**Pro**gramming in **Log**ic）不难看出这是一个专门为人工智能设计的语言；而且跟传统的过程式语言（如C++）和函数式编程语言（如Haskell）思考的方式完全不同，Prolog是一种声明式的逻辑编程语言。

如Wiki中所说，"*有别于一般的过编程语言，prolog的程式是基于谓词逻辑的理论。最基本的写法是定立物件与物件之间的关系，之后可以用询问目标的方式来查询各种物件之间的关系。系统会自动进行匹配及回溯，找出所询问的答案。*" 这句话很好，我直接复制过来了。 Prolog解决问题的方式非常巧妙。鉴于篇幅问题，读者有兴趣可以去查找其他更详尽的学习资料查看Prolog具体的编程写法。

<!-- more -->

这里举一个很简单的例子，修改自wiki:

我预先定义5个事实（fact）
{% highlight prolog %}
human( kate ).  
human( bill ).  
human( xp ).  
likes( kate, bill ).  
likes( bill, kate ).  
likes( xp,   kate ).  
{% endhighlight %}
这几个事实告诉了当前上下文环境：kate, bill 和 xp 是 human；kate 喜欢 bill，bill 也喜欢 kate，而 xp 单恋 kate。

我们再添加一条规则（rule）
{% highlight prolog %}
friend(X,Y):-   
  X \= Y,   
  likes(X,Y),   
  likes(Y,X).   
{% endhighlight %}

这个规则的意思是如果给定的X和Y满足三个条件（这三个条件分别是，X和Y不同，X和Y相互喜欢），那么X和Y满足friend这个条件。

这样的话，如果我在解释器中输入`friend(A, B)`的话，Prolog会自动列出所有的满足这个规则的A和B的组合。

比如根据这个事实，我会得到

{% highlight prolog %}
A = kate, B = bill ?  
A = bill, A = kate .  
{% endhighlight %}
这里没有顺序差别，所以我们会得到2个组合。

* * *
言归正传。greplin challange是greplin公司设计的几个编程挑战，共计3关。

去年看到这个的时候，我正好在学习Prolog，而且我发现每个题目用Prolog来做都很简单，所以就心血来潮的实现了一下^^，同时也为Prolog语言抛砖引玉。

**问题一**：

给定一段连续的字符，找出其中最长的对称子字符串。（如 I like racecars that go fast 这句话中的racecar）


{% highlight prolog %}
:- use_module(library(lists)).  
%%%% Question 1  
ws("你的字符串").                  % 把问题给的长字符串贴到这里  
  
get_all( Lists ):-  
    ws(WS),                        % WS是题目给的字符串  
    reverse(WS, RWS),              % RWS是WS的反转字符串  
    findall( Sublist,              % 找出所有满足以下条件的子字符串，放入Lists中  
        (segment( WS, Sublist ),   % Sublist是WS的子字符串，  
        length(Sublist, L),        % Sublist的长度是L  
        L < 15, L > 2,             % L长度小于15，但大于2（减少计算量）  
        reverse(Sublist, Sublist), % Sublist的反转是Sublist  
        segment( RWS, Sublist )),  % Sublist是RWS的子字符串  
    Lists ).  
      
run( String ):-  
    get_all( Lists ),              % 得到所有的对称的子字符串  
    member( S, Lists ),            % S一个对称的子字符串  
    \+ (                           % 对于所有以下条件的组合，必须至少有一条为假  
         member( OS, Lists ),      % OS是不同于S的另一个字符串  
         S \= OS,                    
         length( S, L1 ),          % 得到OS和S的长度  
         length( OS, L2 ),  
         L1 =< L2 ),               % OS的长度大于或等于S  
    name( String, S ).             % 把S保存成可识别的字符串  
{% endhighlight %}


**问题二**：

i.  给定一个数字，找到第一个比它大的斐波那契质数 X (prime fibonacci number)
ii. 找出所有 X + 1 的质数约数。

{% highlight prolog %}
% i， 求出X  
prime_fibo( Target, Fib ):-       % 给定T，找出比第一比T大的斐波那契质数  
    fibo( Target, Fib ),          % Fib是比T大的一个斐波那契数字  
    prime( Fib ).                 % Fib是质数  
prime_fibo( Target, PFib ):-        
    fibo( Target, Fib ),            
    \+ prime( Fib ),              % Fib不是质数  
    prime_fibo( Fib, PFib ).      % 从Fib开始找结果  
  
  
% ii，给定X + 1，求出所有 X + 1 的质数约数。  
all_div( Target, List ):-         % 找出所有能整除Target的质数，放到List中  
    findall(D,   
    prime_div(Target, D),   
    List ).  
  
% utility rules  
fibo( T, Fib ):-                  % Fib是一个比T大的第一个斐波那契数字  
    fibo( 1, 1, T, Fib ).         % 从1，1开始算斐波那契数列  
fibo( Fib, _, T, Fib ):-           
    Fib > T.                       
fibo( Fib, LastFib, T, FFib ):-   % Fib是当前的Fib数，LastFib是上次的Fib数  
    Fib =< T,                       
    NFib is Fib + LastFib,          
    fibo( NFib, Fib, T, FFib ).   % 计算新的斐波那契数  
          
prime( N ):-                      % 判定N是否为质数  
    prime( N, 2 ).                % 从不能被2整除开始累加  
prime( N, D ):-                   % N不能被D整除  
    0 =\= N mod D,                % N不被D整除  
    D =< sqrt( N ),               % D小于等于N的平方根（缩小D的取值范围）  
    ND is D + 1,                  % ND为D + 1  
    prime( N, ND ).               % ND也不能被ND整除  
prime( N, D ):-                   % 如果D大于N的平方根，则直接判定N不能被D整除  
    D > sqrt( N ).  
  
gen( [N|List], N ):-              % 生成一个 N 到 2 的数组  
    N > 2,                   
    NN is N - 1,  
    gen( List, NN ).  
gen( [2], 2 ).   
  
prime_div( Target, Diviser ):-      
    gen( List, Target ),  
    member( Diviser, List ),        
    is_div( Target, Diviser ).    % 只保留gen生成的数组中的能整除Target的  
  
is_div( T, D ):-                  % T能被质数D整除  
    prime( D ),                     
    0 =:= T mod D. 
{% endhighlight %}

**问题三**：

给定一个数列，找出所有子数列，使得子数列里最大的值是其余数字之和。
这个问题用Prolog很直接，直接按要求枚举出所有结果就好了~


{% highlight prolog %}
% 手动复制给定的数列  
l( [3,4,9,14,15,19,28,37,47,50,54,56,59,61,70,73,78,81,92,95,97,99]).   
fin( Lists ):-  
    l( W ),                        
    findall(                        
      List, (                                 
        subseq0( W, List ),         
        last( Fore, Last, List ),   
        sumlist( Fore, Last)        
      ), Lists ).       
{% endhighlight %}

我非常喜欢Prolog的编程方式：思考直接、代码精简。但是对于此类语言的初学者来说可能理解起来有些吃力。
相信我，如果你明白了Prolog的好处后，你会像我一样喜欢上这个语言的~

