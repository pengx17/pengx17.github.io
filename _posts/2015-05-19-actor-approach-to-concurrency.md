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

经过前面几篇文章的介绍，你学习到了Scala的类型系统(type system)的使用时的灵活性与编译时的安全性，
我们接下来转移到在系列中已经提到过的一个话题：Scala的并发策略。

在更早的几篇文章中，你学习过通过 `Future` 类型的组合做到非同步并发的方法。

这种方法已经可以非常适合地解决不同的问题。但Scala还提供了另一种方式，也就是Scala并发策略的第二个基石：**行动者模式** (Actor Model)。
这是一种完全基于进程间消息传递机制的并发策略。

行动者（以下有时候称为Actor）模式并不是Scala所提出的一种新概念 - 你还可以从Erlang语言中找到Actor并发模型最好实现之一。
Scala 核心库曾经在很长一段时间内保有自己的Actor实现，但 [Akka](http://akka.io) 工具箱包含了一个更好的行动者模式的实现，
并在很长一段时间被社区看做Scala的Actor模式实质的标准。
随后在 Scala 的 2.11 版本中Actor正式被移除出 Scala 核心库，Scala Actor也就彻底的被Akka工具包的Actor实现替代掉了。

此篇文章会向你介绍Akka的Actor模式；你还会学到基本的Akka工具包编程的范例。
需要注意的是，这篇文章与此系列之前的文章不同，我并不会深入讨论你所需要了解的Akka Actor细节；
其目的是提供给你Akka解决问题的思路，并作为吸引你真正去使用它的契机。

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

另外，性能调优一个以底层并发控件搭建的代码工程，是一个十分具有挑战的工作。

----

## 行动者模式

行动者编程模式旨在在避免上述问题的同时，让你写出可推导的、高性能的并发代码。
与目前大规模使用的共享可变状态方法不同的是，行动者模式要求你在从开始编写代码的时候就在脑中考虑到程序设计中的并发问题 - 它并不允许你在之后再把并发支持加进来。

按照Actor的思路，你的应用应由许多个轻量的实体，也就是 `Actor` 构成。每一个actor都负责一个小任务，因此它可以很容易的被描述。
对于更复杂的业务逻辑来说，多个actor之间会产生交互，比如任务委派、或者将消息传递给其他的协作者。

## 行动者系统

行动者是一种可怜的生物：他们不能靠自己存活很久。更确切地说，Akka中的每一个actor都寄生于另一个actor之中，并且每一个都是由所谓的actor系统创造的。
`ActorSystem` 使你可以创造和搜寻actor的同时，还提供了一大堆*我们目前并不需要了解的功能*。

为了允许下面代码的运行，首先把下面的库依赖以及依赖解析器添加到你的基于SBT的Scala 2.10版本的项目中：

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

我们在上面实例化了一个新的 `ActorSystem`，并给它起了个 "Barista"（意为"咖啡师"）的名字 - 如果你之前看过我们的那篇关于[制作咖啡的文章](http://danielwestheide.com/blog/2013/01/09/the-neophytes-guide-to-scala-part-8-welcome-to-the-future.html)，
那你应该熟悉了如何组合多个 `Future` 对象。

最后，作为优秀市民，我们最终把不需要使用的行动者系统关闭掉。

## 定义一个行动者

你的应用中有十几个还是几百万个actor，完全取决于你的使用案例，而且对于Akka来说，几百万个是可以做的到的。
也许你会以为我们在用大数字忽悠你。关于Akka很很重要一点，一个actor与一个线程之间**并没有**一一对应的关系 - 假设说如果有的话，我们会很快的消耗光内存。
更恰当地说，由于actor天生的非阻塞的特性，一个线程可以执行许多个actor，而线程到底需要切换到哪一个执行是由其中哪个有消息需要处理来决定的。

为了理解一个行动者到底会做什么，最好还是先创建第一个简单的actor。
在此，我们定义一个只会接受订单并打印消息到控制台之外，别的事都不会做的 `Barista`：

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

首先，我们定义了几种我们的行动者可以理解的消息 - `CappuccinoRequest` 和 `EspressoRequest`。
通常来讲，如需传递参数，`case class` 会被用作actor之间消息传递的类型。
如果你不需要传递有参数的消息的话，就可以跟我们现在所做的方式一样，以 `case object` 来代表消息。

在任何情况下，请保证你的消息是不可更改的。不然很可怕的事情会发生。不然很可怕的事情会发生。

接下来，我们看一下 `Barista` 类 - 一个继承自 `Actor` 特征 (trait)的行动者。
`Actor` 在实例化时，需要实现它一个返回值为 `Receive`、名为 `receive` 的方法 （也就是 `def receive: Receive`）。
`Receive` 是 `PartialFunction[Any, Unit]` 的一种类型别名。

## 消息处理

然而 `receive` 方法的意义是什么？它的返回类型 `PartialFunction[Any, Unit]` 又是什么？

简而言之，一个由 `receive` 方法返回的部分函数(partial function)会负责处理你的消息。
当你的软件的任何部分 - 不管是不是当前的行动者 - 给你的行动者发了一条消息，Akka 总会尝试让这个行动者处理那条消息：
调用这个行动者的 `receive` 方法，并将消息以参数的形式传入。

#### 产生副作用

当处理一条消息时，一个行动者能做到你任何想做的事，除了让它返回一个值。

> “你说啥!?”

根据 `receive` 返回值部分函数的 `Unit` 类型可以推断，你的部分函数是有副作用的。
也许这对你的世界观会产生动摇，因为我们一直在强调以纯函数范式编程。
但对并发运算来说，很多事情在有副作用的时候才说的通：
行动者们储存着你程序的状态，他们有一些被严格控制的副作用行为自然是可以的；
行动者们所收到的每条信息都是一条一条隔离处理的，因此你不必在他们中间引入同步或锁。

#### 未定类型

这里使用的部分函数不仅有副作用，它的参数还是一个 `Any` 类型的未定类型的值。
为什么我们没有在此利用我们强大的类型系统呢？

这与Akka的几个重要的设计决定相关：Akka允许你把消息转发给其他的actor、实现负载平衡、或是在不知情的情况下将任务代理给其他的actor。

实际使用中，`receive` 的返回值为未定类型通常不会导致问题出现。你所需要做的就如同上面的例子一样：对消息本身进行强类型化，然后对所需的不同消息类型进行模式匹配。

有时候，弱类型的行动者是可能会导致编译器无法检查出的恶心bug。
如果你已经有了强类型强迫症，而且想在你程序的每个角落使用强类型，
那你可以去看看Akka新提供的[Typed Channels](http://doc.akka.io/docs/akka/2.2.0/scala/typed-channels.html)特性。

#### 非同步与非阻塞

我在前面写道，Akka中的actor**总会**处理你发给他的消息。这一条应牢记于心：发送消息和处理消息的过程是非同步并且非阻塞的。
消息的发送者不会在消息被处理完成之前被阻塞；相反，他们会在消息发送后立刻进行自己其他的工作。
至于接受者在稍后是否返回信息 - 消息的发送者也许期待、也许完全不关心。

当行动者从应用的某个部件收到消息时，消息首先会被置于行动者自身的**邮箱**（类似于一个队列）中。
把消息置入行动者的邮箱是一个非阻塞操作。也就是说，发送者不必确认并等待消息是否真的进入了接受者的邮箱队列里。

**调度员**（dispatcher）在新的消息进入行动者邮箱时，会以非同步的方式通知相应组件。
如果行动者没有处理过相同消息的话，这个行动者就会被分配到执行上下文中的某个可用线程中；
当行动者已经处理了某个消息，调度员就把邮箱中下一个所需要处理的消息委派给这个行动者。

行动者在消息处理的时候会阻塞分配给自己的线程。虽然这样做不会阻塞消息发送者，但这也意味着过长的操作也会降低整体性能。
这是因为其他的行动者分配线程进入消息处理阶段时，被阻塞的线程是不可用的。

因此，设计 `Receive` 部分函数的核心原则是**尽量减少每个消息的处理时长**。
最重要的，尽量不要再消息处理的代码里面调用任何阻塞代码。

当然，严格避免阻塞代码，可能会导致你有些行为无法实现 -
比如说目前大部分的数据库驱动仍然是会阻塞的，但你想在你基于行动者模型开发的应用中访问或者保存数据。
对于此类难题已经有相关解决方案，但我们并不会在这篇介绍性文章涉及到。

## 创建行动者

定义一个actor已经在上面顺利完成，但如何在我们的应用里实际使用咖啡师 `Barista` actor呢？
为了做到这点，我们必须先实例化一个新的 `Barista` 行动者。
你也许会像往常一样，像下面一样调用它的构造函数：

```scala
val barista = new Barista // 会抛出异常
```

这样做会导致运行失败！Akka 发给你了一张写着 `ActorInitializationException` 的谢谢卡。
为了整个行动者模式运转正常，actor们必须由 `ActorSystem` 和它的组件来进行管理。
因此，你必须请求行动者系统来初始化一个新的actor：

```scala
import akka.actor.{ActorRef, Props}
val barista: ActorRef = system.actorOf(Props[Barista], "Barista")
```

定义在 `ActorSystem` 的 `actorOf` 方法需要一个 `Props` 实例，而它能提供配置新创建行动者的方法。视需要，你还可以给你实例化的那个actor起个名字。

请注意，`actorOf` 返回的对象类型不是 `Barista`，而是 `ActorRef`。
行动者们从不会直接的访问其他的行动者，因此我们不必访问actor实例。
然而，行动者们或是其他组件在发送消息给其他行动者时，会取得他们的引用对象，而不是他们本身。

所以，`ActorRef` 就像是actor的一种代理人（proxy）。
这样会给我们带来一些方便，比如一个 `ActorRef` 可以被序列化，然后将它作为一个非本机的远程行动者的代理。
对获取到 `ActorRef` 的组件来说，actor的物理位置 - 到底是存在于同一个JVM还是远程电脑上 - 是透明的。
我们将其称之为**位置透明性**。

请注意，`ActorRef` 没有类型化参数。一个 `ActorRef` 可以被替换成任意一个其他的 `ActorRef`，这就允许我们把消息发送给任意的 `ActorRef` 引用对象。
就像上面所提到的，这是Akka的特别设计 - 允许了你在改变行动者系统拓扑结构的同时不必对发送者进行任何修改。

## 发送消息

现在我们已经实例化了一个 `Barista` actor和引用到它的 `ActorRef`，然后我们就可以发消息了。
调用 `ActorRef` 的 `!` 方法：

```scala
barista ! CappuccinoRequest
barista ! EspressoRequest
println("I ordered a cappuccino and an espresso")
```

调用 `!` 是一个放射后不管(fire-and-forget)的操作：你*告诉* `Barista` 你要点一杯卡布奇诺，然而并不等待咖啡师的回应，
这就是Akka中actor之间交互的最常见模式。调用此方法实际上的行为是，你让Akka把你的消息放置于接受者的邮箱队列里。
上面介绍过，消息发送不是阻塞行为，消息的接受者最终会在将来的某时刻处理你发送的消息。

Due to the asynchronous nature, the result of the above code is not deterministic. It might look like this:
由于消息机制的不同步的性质，上面代码的结果是非决定性的。
看起来有可能是这样：

```
I have to prepare a cappuccino!
I ordered a cappuccino and an espresso
Let's prepare an espresso.
```

尽管我们最初发送两条消息给 `Barista` 的邮箱，在上面的示例中，我们自己的 `println` 输出插在了处理两条消息之间。

## 答复消息

只是把消息发送给别人是不够的。你有时候会想要答复消息的发送者，当然，仍然按非同步的方式。

为了直接让你知道如何答复发送者，我们略过一些内容直接告诉你，actor有一个能返回最后一条（也就是当前正在处理的）消息的发送者的方法：`sender`。

但为什么actor能知道是谁发送的消息呢？答案就在 `!` 方法的第二个参数，一个隐含的参数列表类型：

```
def !(message: Any)(implicit sender: ActorRef = Actor.noSender): Unit
```

当 `ActorRef` 的 `!` 方法在一个行动者内被调用时，行动者会把自己的 `ActorRef` 隐式的传入此方法。

我们把 `Barista` 的代码稍作更改，在打印到控制台之前立刻回复一个 `Bill` 消息给 `CoffeeRequest` 消息的发送者：

```scala
case class Bill(cents: Int)
case object ClosingTime
class Barista extends Actor {
  def receive = {
    case CappuccinoRequest =>
      sender ! Bill(250)
      println("I have to prepare a cappuccino!")
    case EspressoRequest =>
      sender ! Bill(200)
      println("Let's prepare an espresso.")
    case ClosingTime => context.system.shutdown()
  }
}
```

我们在上一段代码中加入了一条新的消息 `ClosingTime`。这个消息会使得 `Barista` 通过访问 `ActorContext` 来 关闭整个行动者系统。

现在，我们介绍第二个行动者，其代表了一个客户 `customer`：

```scala
case object CaffeineWithdrawalWarning
class Customer(caffeineSource: ActorRef) extends Actor {
  def receive = {
    case CaffeineWithdrawalWarning => caffeineSource ! EspressoRequest
    case Bill(cents) => println(s"I have to pay $cents cents, or else!")
  }
}
```
这个行动者是一个咖啡成瘾者，因此他所能做的就是点咖啡。
我们传递一个 `ActorRef` 到他的构造函数中。
对这个顾客来说，他不知道这个 `ActorRef` 是指向了一个 `Barista` 还是什么，只知道这个行动者引用是他的咖啡因饮料的来源。
他只关心是否能发送 `CoffeeRequest` 给这个引用。

最后，为了让所有东西运转起来，我们需要创建两个行动者，并将一个 `CaffeineWithdrawalWarning` 消息发送给我们的顾客：

```scala
val barista = system.actorOf(Props[Barista], "Barista")
val customer = system.actorOf(Props(classOf[Customer], barista), "Customer")
customer ! CaffeineWithdrawalWarning
barista ! ClosingTime
```

对于 `Customer`，我们使用一种不同的创建 `Prop` 的工厂方法：
需要实例化的行动者的类型和实例化它所需要的参数一起传入到工厂方法里。
这样我们的咖啡师的 `ActorRef` 就可以传入到顾客的构造函数里了。

发送一条 `CaffeineWithdrawalWarning` 消息给顾客，会使得它发送一个 `EspressoRequest` 消息给咖啡师；
咖啡师在接收后，再反过来返回给顾客一个 `Bill` 消息。
输出会像是下面这样：

```
Let's prepare an espresso.
I have to pay 200 cents, or else!
```

首先，当咖啡师处理 `EspressoRequest` 消息时，它会给顾客发送一条新消息；
它在发送新消息给客户时，并不会阻塞 `EspressoRequest` 消息的处理（也就是往控制台打印一段字符串）。
稍后，顾客开始处理 `Bill` 账单信息，并把它打印到控制台。

## 问问题

Sometimes, sending an actor a message and expecting a message in return at some later time isn’t an option – the most common place where this is the case is in components that need to interface with actors, but are not actors themselves. Living outside of the actor world, they cannot receive messages.

有时候，仅仅发送消息给行动者并期待将来某个时间的回复是不够的。
最常见的情况是，我们需要在不同的组件中与行动者互动，而不是仅仅在行动者之间互动。
在行动者的世界外，其他组件是无法接收消息的。

For situations such as these, there is Akka’s ask support, which provides some sort of bridge between actor-based and future-based concurrency. From the client perspective, it works like this:

为了对应这种情况，Akka 加入了对于 `ask`（询问）的支持，它提供了一个基于actor和基于future的并行实现之间进行交互的一架桥梁。

```scala
import akka.pattern.ask
import akka.util.Timeout
import scala.concurrent.duration._
implicit val timeout = Timeout(2.second)
implicit val ec = system.dispatcher
val f: Future[Any] = barista2 ? CappuccinoRequest
f.onSuccess {
  case Bill(cents) => println(s"Will pay $cents cents for a cappuccino")
}
```

首先，你需要导入一些包以获得`ask` 语法支持，并隐性的为 `?` 方法返回的 `Future` 对象添加一个超时规则。
并且，你需要一个 `ExecutionContext`。这里，我们简单地使用了 `ActorSystem` 的默认调度器 - 它同时还是一个方便获取的一个 `ExecutionContext`。

就像你看到的，返回的`Future` 对象的内含类型是 `Any`。这应该不会让你感到惊讶，毕竟它就是一个行动者发送过来的任意一条消息而已。

对于被询问的行动者来说，`ask` 行为上跟返回给一个消息发送者一条消息时一回事。
这就是为什么我们不必更改任何代码，就可以询问一个 `Barista`。

被询问的行动者返回消息给询问者时，`Promise` 对象所属的返回的 `Future` 就完成了。

一般来讲，在可以使用 `告知` 的情况下就不要使用 `询问`，因为后者会消耗更多资源。
Akka 不是跟懂礼貌的人用的！
但是，总有情况是你必须使用询问的，在这种情况下请自由使用。

## 有状态的行动者

一个行动者也许会有自己的内部状态，但并不是一定需要。
有时，应用的一大半状态是由行动者之间传递的不可变消息组成的。

一个行动者在同一时刻只会处理一条信息时。由于做到了这一点，行动者理论上是可以修改内部状态的。
这意味着行动者内部可能会有可变状态，但由于每条消息是在隔离开的情况下处理，同一个行动者的内部状态并不会因为并行问题而搞砸。

为了演示，我们把没有状态的 `Barista` 改造成携带状态的行动者。简单的让它记录订单数量：

```scala
class Barista extends Actor {
  var cappuccinoCount = 0
  var espressoCount = 0
  def receive = {
    case CappuccinoRequest =>
      sender ! Bill(250)
      cappuccinoCount += 1
      println(s"I have to prepare cappuccino #$cappuccinoCount")
    case EspressoRequest =>
      sender ! Bill(200)
      espressoCount += 1
      println(s"Let's prepare espresso #$espressoCount.")
    case ClosingTime => context.system.shutdown()
  }
}
```

我们引入了两个变量，`cappuccinoCount` 和 `espressoCount`，分别记录每种咖啡的订单数。
事实上这是我们在整个系列教程里第一次使用变量 `var`。
尽管我们在函数式编程中尽量避免使用它，但这是唯一一种允许行动者携带状态的方式。
因为每条消息是在被隔离开的情况下执行，上面的代码执行起来就像是在非行动者环境下使用 `AtomicInteger` 值。

## 总结

到此为止就是我们关于行动者编程模型的介绍，还有如何在Akka中使用它。
虽然我们只是粗略的体验了Akka一些表面的内容，也略过了不少重要的概念，
但我仍希望你已经有了足够多关于使用行动者模型的并行策略的领悟，并使你继续学习更多的内容。

在接下来的文章中，我会丰富我们的小例子，给它加一些有意义的行为，并向你讲解Akka更多的理念，还有向你介绍在行动者系统是如何处理错误的。

Posted by Daniel Westheide Feb 27th, 2013
