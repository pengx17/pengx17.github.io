---
layout: post
keywords: me
description:
title: New blog, new start, memories off
tagline: 博客迁移！
categories: [life]
tags: [about me]
group: archive
icon: rocket
reason: 彻底斩断一年多的单相思后
---

{% comment %}
谢谢你，我不会忘记。
{% endcomment %}
_"红花当然配绿叶. 这一辈子谁来陪. 渺渺茫茫来又回. 往日情景再浮现. 藕虽断了丝还连. 轻叹世间事多变迁"_

* * *
{% if 0 == 1 %} {{ page.reason }}
{% else %}
<!--{{ page.reason }}--> 
从最后一次写博客到现在
{% endif %}，我停写了大概五个月的博客。从那之后我的休息时间比之前空余出来不少，可总提不起兴致来写博客。

不过，五个月来我也没有闲着。因为时间多了，就有空停下来放慢节奏： 翻翻已经积了很久灰的工具书，逛逛网上牛人们的博客，同时也补充积累了不少非计算机专业的知识。

我捉摸着，是到了继续写博客的时候了。
为了督促自己行动起来，我决定把博客迁移到 <del>牛逼烘烘的</del> [GitHub Pages](http://pages.github.com/, "GitHub Pages")上。

<!-- more -->

之前的博客是用的[CSDN在线博客](http://blog.csdn.net/pengx17)服务。我是最近才知道了静态博客系统这种看起来很Geek的玩意儿。
[网上](http://www.cnblogs.com/cuxnil/archive/2013/01/08/2850458.html)提到了几种其他的流行的博客系统，
发现GitHub Pages上面原生支持的是Jekyll，我一拍脑门，就用它得了。

经过一个周末的时间，我浏览了一些关于用Jekyll在GitHub Pages上搭建静态博客的文章。
了解了一些常用操作后后，就把之前在CSDN的几篇博文转移到了这里。

因为笔者主要在Windows平台上开发，而Jekyll在Windows上的支持性并不好，导致迁移博客时出现了非常多的文件编译错误问题，让人焦头烂额。

短暂的使用过后，我觉得静态博客系统最厉害的地方在于它抛弃了MySQL的使用。
在线的动态内容，比如帖子的留言和评论，被云端的评论服务系统所替代托管。
例如本博客使用的[DISQUS](http://disqus.com/)。可能是我孤陋寡闻好久没关注业内消息了，不过看到这个想法的时候确实让我眼前一亮。跟这个类似的还有著名的
[Gravatar头像托管](http://en.gravatar.com/)，用户在评论时填写了邮箱，博客程序会自动查找在Gravatar上是否有与之绑定的头像，一劳永逸。

虽然Jekyll提供给用户很高的自由度和高度定制的自动化机制，但用Jekyll写博客还是不如所见即所得的在线博客写起来更方便更直观一些，并且作为普通的博客用户来说学习曲线过陡了点。
这就像是学校里大家写论文时对于文档编辑器的选择了：普通青年用Word，文艺青年用LaTex，二逼青年用写字板。

最后，感谢博客的主题原作者[codepiano](http://codepiano.github.io/about.html)提供的中文本地化的博客模板。很合我胃口, <del>呵呵</del> 。



