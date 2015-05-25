---
layout: post
keywords: scala, actor, akka
description: blog
title: "译文：处理Actor系统中的故障"
categories: [Translation]
tags: [Scala, Translation]
group: archive
icon: file-o
featured: true
---

> 原文链接 [The Neophyte's Guide to Scala Part 15: Dealing With Failure in Actor Systems](http://danielwestheide.com/blog/2013/03/20/the-neophytes-guide-to-scala-part-15-dealing-with-failure-in-actor-systems.html)

系列的前一篇文章中，我向你介绍了Scala语言中处理并发的第二块基石：行动者模式，其补充了基于组合future类的并发策略。
你学习了如何定义和创造行动者，如何向行动者们发送消息，行动者如何处理所接收到的消息以至于改变行动者的内部状态，还有如何回复给发送者消息。

希望你已经对行动者模式的并发策略产生了足够的兴趣。
如果你想开发一个完整的基于行动者的应用，仅仅使用简单的产生回声的行动者是不够的。
为了这个目的，还有不少关键性的概念需要你去学习。

行动者模型意味着它能帮助你实现很高程度的错误容忍度。
此篇文章，我们会看一下在一个基于行动者模式的应用中故障是如何被处理的。
你将会看到，它与传统的层级搭建的服务器架构的错误处理方式有着根本上的不同。

处理故障的解决方式与Akka的一些核心概念紧密相连，而其中某些还是搭建起Akka的重要的元素。
因此，这篇文章还作为这些核心概念和组件的指南。

----

To be continued...
