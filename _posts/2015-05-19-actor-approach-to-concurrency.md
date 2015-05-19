---
layout: post
keywords: scala, actor, akka
description: blog
title: "Actor的并发策略"
categories: [Translation]
tags: [Scala, Translation]
group: archive
icon: file-o
featured: true
---

> 原文链接 [The Neophyte's Guide to Scala Part 14: The Actor Approach to Concurrency](http://danielwestheide.com/blog/2013/02/27/the-neophytes-guide-to-scala-part-14-the-actor-approach-to-concurrency.html)

经过前面几篇文章的介绍，你学习到了Scala的类型系统(type system)强大的弹性和如何可以利用它实现编译时的安全性，
我们可以转移到下一个我们在系列中已经提到过的一个话题：Scala的并发策略。

在更早的几篇文章中，你学习过一种可以通过 `Future` 类型的组合做到非同步的方法。

这种方法已经可以非常适合地解决不同的问题。但Scala还提供了另一种方式，也就是Scala并发策略的第二个基石：**行动者模式** (Actor Model)。
这是一种完全基于进程间消息传递机制的并发策略。

行动者（以下有时候称为Actor）模式并不是Scala所提出的一种新概念 - 你还可以从Erlang语言中找到Actor并发模型最好实现之一。
Scala 核心库曾经在很长一段时间内保有自己的Actor实现，但 [Akka](http://akka.io) 工具箱的Actor实现由于更好的支持，
在很长一段时间已经作为Scala的Actor开发模式实质的标准了，随后在 Scala 的 2.11 版本中Actor正式被移除，彻底的被 Akka 工具包替代掉了。

此篇文章会向你介绍Akka的Actor模式；你还会学到基本的Akka工具包编程的范例。
需要注意的是，这篇文章与此系列之前的文章不同，我并不会非常深入讨论你所需要了解的Akka Actor细节；
其目的更是让你了解Akka解决问题的思路，并作为吸引你使用它的契机。

----

## 共享可变状态（Shared Mutable State）所带来的问题

目前，实现并发机制的方案的主要思路是共享可变状态 - 一个应用靠大量的表示状态的对象和线程组成，每个表示对象会被应用中不同部分的不同线程修改它们的状态。
一般来说，为了保证应用的正确运行，某个状态不会被应用不同部分的线程以错误的方式同时修改，代码中会散布者着各式各样的读写锁。
在此同时，我们又得尽量保证不会在大段代码外设锁，因为这样会让程序的运行速度大幅度的降低。

实际上更常见的是，程序员经常会在最开始编写的时候完全没考虑过并发的问题 - 他们总是在多线程需求来临之时才把代码重写成多线程结构。
这样所导致的结果是，人们写的没有考虑过并发需求的代码会非常直白，但如果把这样的代码移植成并发化后，代码会变得极难读懂。

上面的问题是由于以底层同步锁和线程所构成的代码不容易描述引发的。这样会导致人们很难以优雅的方式把问题解决：
如果你不能清晰的解释清代码到底在做什么，你可以大胆猜测代码里已经充斥着各式各样龌龊的bug，
比如竞态条件(race condition)，死锁(deadlock)、或是一些捉摸不透的行为 - 甚至有一些只有在你的代码部署到了生产环境后的几个月才能注意到。

另外，性能调优一个以底层控件搭建的代码工程是一个十分具有挑战性的工作。

----

## 行动者模式

行动者编程模式旨在在避免上述问题的同时，让你写出可推导的、高性能的并发代码。
与目前大规模使用的共享可变状态方法不同的是，Actor 模式要求你在从开始编写代码的时候就在脑中考虑到程序设计中的并发问题 - 它并不允许你在之后再把并发支持加进来。

按照Actor的思路，你的应用应由许多个轻量的实体 - 也就是 `Actor` - 构成。每一个actor都负责一个小任务，因此它可以很容易的被描述。
对于更复杂的业务逻辑来说，多个actor之间会产生交互，比如把任务委派、或者把消息传递给其他的协作者。

## 行动者系统

行动者是一种可怜的生物：他们不能靠自己存活很久。相反，Akka中的每一个actor都寄生于另一个actor之中，并且每一个都是由所谓的actor系统创造的。
`ActorSystem` 使你可以创造和搜寻actor的同时，还提供了一大堆*我们目前并不需要了解的功能*。

为了允许下面代码的运行，首先把下面的依赖以及解析器添加到你的基于SBT的Scala 2.10版本的项目中：

```scala
resolvers += "Typesafe Releases" at "http://repo.typesafe.com/typesafe/releases"
libraryDependencies += "com.typesafe.akka" %% "akka-actor" % "2.2.3"
```
然后，我们创建一个 `ActorSystem` 实例。我们需要它作为actor的运行环境：

```scala
import akka.actor.ActorSystem
object Barista extends App {
  val system = ActorSystem("Barista")
  system.shutdown()
}
```

我们在上面实例化了一个新的 `ActorSystem`，并给它起了个 "Barista" 的名字 - 如果你之前看过我们的那篇关于[制作咖啡的文章](http://danielwestheide.com/blog/2013/01/09/the-neophytes-guide-to-scala-part-8-welcome-to-the-future.html)，那你应该熟悉了如何组合多个 `Future` 对象。

最后，作为优秀市民，我们最终把不需要使用的行动者系统关闭掉。

-----

## 定义一个行动者


你的应用中有十几个还是几百万个actor，完全取决于你的使用案例，而且对于Akka来说，几百万个完全可以做的到。
我们并没有用大数字忽悠你！关于Akka很很重要一点，一个actor与一个线程之间**并没有**一一对应的关系 - 假设说如果有的话，我们会很快的消耗光内存。
更恰当地说，由于actor天生的非阻塞的特性，一个线程可以执行许多个actor，而线程到底需要切换到哪一个执行是由其中哪个有消息需要处理来决定的。

为了理解到底发生了什么，最好还是先创建第一个简单的actor - 一个只会接受订单并打印消息到控制台之外别的事都不会做的 `Barista` (咖啡师)：

```scala
sealed trait CoffeeRequest
case object CappuccinoRequest extends CoffeeRequest
case object EspressoRequest extends CoffeeRequest

import akka.actor.Actor
class Barista extends Actor {
  def receive = {
    case CappuccinoRequest => println("I have to prepare a cappuccino!")
    case EspressoRequest => println("Let's prepare an espresso.")
  }
}
```

首先，我们定义了几种我们的行动者可以理解的消息。通常来讲，如需传递参数，`case class` 会被用作actor之间消息传递的类型。
如果你不需要传递有参数的消息的话，就可以跟我们现在所做的方式一样，以 `case object` 来代表消息。

在任何情况下，请保证你的消息是不可更改的。不然很可怕的事情会发生。不然很可怕的事情会发生。

接下来，我们看一下 `Barista` 类 - 一个继承自 `Actor` 特征 (trait)的行动者。
`Actor` 在实例化时，需要实现它一个返回值为 `Receive`、名为 `receive` 的方法 （也就是 `def receive: Receive`）。
`Receive` 是 `PartialFunction[Any, Unit]` 的一种类型别名。

## 传递消息

然而 `receive` 方法的意义是什么？它的返回类型 `PartialFunction[Any, Unit]` 又是什么？

简而言之，一个由 `receive` 方法返回的部分函数(partial function)会负责处理你的消息。
当你的软件的任何部分 - 不管是不是当前的行动者 - 给你的行动者发了一条消息，Akka 总会让这个行动者处理那条消息，
通过调用这个行动者的 `receive` 方法，并将消息以参数的形式传入。

## 产生副作用

当处理一条消息时，一个行动者能做到你任何想做的事，除了让它返回一个值。

> “你说啥!?”

根据 `receive` 返回值部分函数的 `Unit` 类型可以推断，你的部分函数是有副作用的。
也许这对你的世界观会产生动摇，因为我们一直在强调以纯函数式编程。
但对并发运算来说，有副作用才说的通很多事情。
行动者们储存着你程序的状态，如果他们有一些被严格控制的副作用行为是完全可以的；
你的行动者所收到的每条信息都是一条一条单独处理的，因此你不必在他们中间引入同步或锁。

-----

秃笔肯忒牛...
