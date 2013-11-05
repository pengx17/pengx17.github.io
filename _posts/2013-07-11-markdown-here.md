---
layout: post
keywords: blog
description: blog
title: "Markdown Here!"
categories: [Life]
tags: [Markdown]
group: archive
featured: true
icon: file-o
tldr: true
---
今天写邮件的时候想到，如果Gmail能提供一个插件用Markdown来写邮件，那邮件排版该多么容易！
于是xp就搜了下GMail Lab, 很遗憾的是，没有，<del>说好的谷人希呐</del>？
我心想，总归有其他程序员也想到这种问题写个插件什么的，于是就Google了一下。你别说，还真有！

_当当当~当_！下面隆重推荐**[Markdown Here][1]**插件。

盗一张官网Github README里的图：
![hehe, use markdown](https://raw.github.com/adam-p/markdown-here/master/store-assets/dos-equis-MDH.jpg)


实际使用中，用户需要在GMail的撰写界面填写好Plain Markdown代码。写完后，点击一下Chrome的`Markdown Here Toggle`按钮，就自动把Markdown代码转化成了排版好的Rich HTML邮件内容。嗯？是不是很方便？
<del>没用过Markdown的人是体会不到的啦。。</del>

***

作者在插件介绍页面提到了自己编写插件的动机（翻译）：
> ...
>
> **直到现在，为邮件排版都是一个头痛的事。** 如果你经常写长而复杂的邮件，你会发现自己总是在不停地重复
> 为排版做一些麻烦事：选择文本，点击排版按钮，然后继续为剩下的内容做类似的蛋疼体力活。这样既沉闷，
> 让人沮丧，而且还慢！
> 是不是很耳熟？
>
> **Markdown Here** 
>允许你只用键盘就能用**简单的代码撰写复杂格式的邮件**。写完后，你所要做的仅仅是**多点击一下鼠标**，就可以发送邮件了。
>
> 写邮件用的语言叫做*Markdown*, 它...实在是**太简单了**，简单到不值得去介绍。
>
> ...

效果示例：
![Markdown Here!](/image/post/markdown_here.png)

跟GitHub的Markdown形式类似，我们可以方便的在邮件里插入经过着色加亮的代码块，以如下格式:

    ```cpp
    int main(int,char**)
    {
      printf("Hello world!\n");
      return 0;
    }
    ```

会自动转换成：

```cpp
int main(int,char**)
{
  printf("Hello world!\n");
  return 0;
}
```
    
Markdown Here不仅仅只支持GMail邮件。只要是支持富内容编辑(Rich Editing)的都可以通过这个插件简化排版。
官方wiki列出了[支持的列表][2]。

另外，这个插件可以在插件选项页面修改CSS设置，来定制排版风格。比如，我想修改默认字体修改成自己常用的`Trebuchet`：

```css
.markdown-here-wrapper {
  font-family: 'trebuchet ms',sans-serif;
}
```

总之，妈妈再也不用担心邮件的排版问题了。木蛤蛤。

遗憾的是这种转换是单向的。已经发出去的邮件，我们是无法把已经转化的邮件内容再还原为为Markdown文本，这给我们将来编辑修改之前的邮件风格产生了不便，除非自己在本地保留一份Markdown格式的原本，或者使用一些把[HTML转化成Markdown文本的工具][3]。

希望作者或者社区的人能增加反向转换功能啊！<del>谜之声，这个不是开源的吗为什么你自己不去干？</del>

附上我的Markdown Here的CSS:

<!-- more -->

{% gist 5983208 %}

[1]: http://markdown-here.com/index.html
[2]: https://groups.google.com/d/msg/markdown-here/UwCTtsiWG5w/1ytDvFLz8z0J
[3]: http://html2markdown.com/
