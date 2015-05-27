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

## 行动者的层级结构

在了解当你的行动者内出现错误的时候会发生什么事情之前，你需要了解一个行动者并行系统的重要概念：
行动者系统是按照一定层级结构组织而成。

所以这个概念代表着什么意思呢？首先，它意味着你的每一个行动者都有一个parent（以下称为父行动者），每个行动者又可以创建他的子行动者。
基本上，你可以把一个行动者系统看成一个由行动者组成的金字塔。父行动者像是在实际生活中一样照看他的子女，并在子女跌倒的时候把他们扶起来。
你将会马上看到这是如何做到的。


#### 守卫行动者

前一篇文章中我们创建过两种行动者，分别为 `Barista` 咖啡师和 `Customer` 客户。
这回我们换一种方式创建，并关注一下这些不同的行动这类型是如何创建出来的：

```scala
import akka.actor.ActorSystem
val system = ActorSystem("Coffeehouse")
val barista = system.actorOf(Props[Barista], "Barista")
val customer = system.actorOf(Props(classOf[Customer], barista), "Customer")
```

如上，我们通过调用 `ActorSystem` 的 `actorOf` 方法创建出两个行动者。

所以这两个行动者的父亲是谁呢？是 `Coffeehouse` 这个行动者系统吗？嗯，这个答案虽然不是很准确，但也差不多了。
行动者系统并不是一种行动者，但是它被称为一个座位所有用户创建的根级行动者的父亲的**守卫**行动者。
所谓的用户创建的根级行动者，也就是通过调用行动者系统的 `actorOf` 创建出的行动者。

你的系统中不应该有很多由守卫行动者直接创建出的行动者。
更合理的情况是，系统内只应有少量的顶级行动者，并且他们会把大部分的任务委托给自己的子女们。

#### 行动者的路径

The hierarchical structure of an actor system becomes apparent when looking at the actor paths of the actors you create. These are basically URLs by which actors can be addressed. You can get an actor’s path by calling path on its ActorRef:

如果你实际的观察你所创造的行动者的路径，行动者系统的层级结构会变得清晰起来:
他的路径基本上可以看做是以URL的形式所代表的一条地址。
你可以通过调用 `ActorRef` 的 `path` 方法获取到行动者的路径。


```scala
barista.path // => akka.actor.ActorPath = akka://Coffeehouse/user/Barista
customer.path // => akka.actor.ActorPath = akka://Coffeehouse/user/Customer
```

一个actor的路径是由：
Akka 协议 `akka://`，
用户的守卫行动者 `Coffeehouse`，
再加上行动者的名字（也就是使用 `actorOf` 方法时用的那个）组成的。
对于运行在其他电脑上的远程的行动者来说，你也会在地址中看到远程主机名和他的端口号。

行动者的地址可以用来查询另一个行动者。
比如，我们可以在 `Customer` 行动者内调用它的 `ActorContext` 的 `actorSelection`，
传入 `Barista` 的相对路径，获取到它。

```scala
context.actorSelection("../Barista")
```

虽然看似通过查找一个行动者的路径来获取引用看起来有些用处，
但大部分时候通过构造函数的参数传入依赖的行动者的引用是更好地方式，就像我们一直以来所做的。
过于密切关注行动者依赖在系统内的位置会更容易导致bug发生，并且让你的代码变得难以重构。

#### 一个层级结构的例子

为了展示父行动者们如何监视着他们的子行动者，还有这种结构如何保证系统的容错性，我会在接下来继续关注我们的咖啡厅。
现在我们赋予 `Barista` 一个子行动者，使得咖啡师能将运营咖啡厅的一些业务委托给他的子行动者执行。

如果按实际咖啡师的工作中定义 `Barista` 模型，我们会为他的子任务创建一堆的子行动者。
但为了保证这篇文章的专注性，我们稍微为下面的例子做了点简化。

假设 `barista` 有一个收银机，这个收银机可以处理交易、打印收据、计算每天的销售额等事务。
下面是我们的第一个版本：

```scala
import akka.actor._
object Register {
  sealed trait Article
  case object Espresso extends Article
  case object Cappuccino extends Article
  case class Transaction(article: Article)
}
class Register extends Actor {
  import Register._
  import Barista._
  var revenue = 0
  val prices = Map[Article, Int](Espresso -> 150, Cappuccino -> 250)
  def receive = {
    case Transaction(article) =>
      val price = prices(article)
      sender ! createReceipt(price)
      revenue += price
  }
  def createReceipt(price: Int): Receipt = Receipt(price)
}
```

这个行动者包含了一个不可变的价目表，还有一个代表着销售额的整数类型的变量。
当他接受了一个`Transaction` 消息后，他会相应的增加销售额变量的值，并返回一个可供打印的 `Receipt` 收据消息。

就像之前提到过的，这个 `Register` 理应是作为咖啡师的一个子行动者存在的，因此我们应该在咖啡师行动者内创建它，而不是通过行动者系统。
我们第一个成为人父的行动者是这样的：

```scala
object Barista {
  case object EspressoRequest
  case object ClosingTime
  case class EspressoCup(state: EspressoCup.State)
  object EspressoCup {
    sealed trait State
    case object Clean extends State
    case object Filled extends State
    case object Dirty extends State
  }
  case class Receipt(amount: Int)
}
class Barista extends Actor {
  import Barista._
  import Register._
  import EspressoCup._
  import context.dispatcher
  import akka.util.Timeout
  import akka.pattern.ask
  import akka.pattern.pipe
  import concurrent.duration._

  implicit val timeout = Timeout(4.seconds)
  val register = context.actorOf(Props[Register], "Register")
  def receive = {
    case EspressoRequest =>
      val receipt = register ? Transaction(Espresso)
      receipt.map((EspressoCup(Filled), _)).pipeTo(sender)
    case ClosingTime => context.stop(self)
  }
}
```

首先，我们定义了 `Barista` 行动者所需要处理的消息类型。
一个意式咖啡杯 `EspressoCup` 有一个通过 `sealed trait` 实现的不可变的状态。

The more interesting part is to be found in the implementation of the Barista class. The dispatcher, ask, and pipe imports as well as the implicit timeout are required because we make use of Akka’s ask syntax and futures in our Receive partial function: When we receive an EspressoRequest, we ask the Register actor for a Receipt for our Transaction. This is then combined with a filled espresso cup and piped to the sender, which will thus receive a tuple of type (EspressoCup, Receipt). This kind of delegating subtasks to child actors and then aggregating or amending their work is typical for actor-based applications.

更令人感兴趣的部分是在 `Barista` 类中。调度器，


Also, note how we create our child actor by calling actorOf on our ActorContext instead of the ActorSystem. By doing so, the actor we create becomes a child actor of the one who called this method, instead of a top-level actor whose parent is the guardian actor.

Finally, here is our Customer actor, which, like the Barista actor, will sit at the top level, just below the guardian actor:

```scala
object Customer {
  case object CaffeineWithdrawalWarning
}
class Customer(coffeeSource: ActorRef) extends Actor with ActorLogging {
  import Customer._
  import Barista._
  import EspressoCup._
  def receive = {
    case CaffeineWithdrawalWarning => coffeeSource ! EspressoRequest
    case (EspressoCup(Filled), Receipt(amount)) =>
      log.info(s"yay, caffeine for ${self}!")
  }
}
```

It is not terribly interesting for our tutorial, which focuses more on the Barista actor hierarchy. What’s new is the use of the ActorLogging trait, which allows us to write to the log instead of printing to the console.

Now, if we create our actor system and populate it with a Barista and two Customer actors, we can happily feed our two under-caffeinated addicts with a shot of black gold:

```scala
import Customer._
val system = ActorSystem("Coffeehouse")
val barista = system.actorOf(Props[Barista], "Barista")
val customerJohnny = system.actorOf(Props(classOf[Customer], barista), "Johnny")
val customerAlina = system.actorOf(Props(classOf[Customer], barista), "Alina")
customerJohnny ! CaffeineWithdrawalWarning
customerAlina ! CaffeineWithdrawalWarning
```

If you try this out, you should see two log messages from happy customers.


----


## To crash or not to crash?

Of course, what we are really interested in, at least in this article, is not happy customers, but the question of what happens if things go wrong.

Our register is a fragile device – its printing functionality is not as reliable as it should be. Every so often, a paper jam causes it to fail. Let’s add a PaperJamException type to the Register companion object:


```scala
class PaperJamException(msg: String) extends Exception(msg)
```

Then, let’s change the createReceipt method in our Register actor accordingly:

```scala
def createReceipt(price: Int): Receipt = {
  import util.Random
  if (Random.nextBoolean())
    throw new PaperJamException("OMG, not again!")
  Receipt(price)
}
```

Now, when processing a Transaction message, our Register actor will throw a PaperJamException in about half of the cases.

What effect does this have on our actor system, or on our whole application? Luckily, Akka is very robust and not affected by exceptions in our code at all. What happens, though, is that the parent of the misbehaving child is notified – remember that parents are watching over their children, and this is the situation where they have to decide what to do.


#### Supervisor strategies

The whole act of being notified about exceptions in child actors, however, is not handled by the parent actor’s Receive partial function, as that would confound the parent actor’s own behaviour with the logic for dealing with failure in its children. Instead, the two responsibilities are clearly separated.

Each actor defines its own supervisor strategy, which tells Akka how to deal with certain types of errors occurring in your children.

There are basically two different types of supervisor strategy, the OneForOneStrategy and the AllForOneStrategy. Choosing the former means that the way you want to deal with an error in one of your children will only affect the child actor from which the error originated, whereas the latter will affect all of your child actors. Which of those strategies is best depends a lot on your individual application.

Regardless of which type of SupervisorStrategy you choose for your actor, you will have to specify a Decider, which is a PartialFunction[Throwable, Directive] – this allows you to match against certain subtypes of Throwable and decide for each of them what’s supposed to happen to your problematic child actor (or all your child actors, if you chose the all-for-one strategy).



#### Directives

Here is a list of the available directives:


```scala
sealed trait Directive
case object Resume extends Directive
case object Restart extends Directive
case object Stop extends Directive
case object Escalate extends Directive
```


- `Resume`: If you choose to Resume, this probably means that you think of your child actor as a little bit of a drama queen. You decide that the exception was not so exceptional after all – the child actor or actors will simply resume processing messages as if nothing extraordinary had happened.

- `Restart`: The Restart directive causes Akka to create a new instance of your child actor or actors. The reasoning behind this is that you assume that the internal state of the child/children is corrupted in some way so that it can no longer process any further messages. By restarting the actor, you hope to put it into a clean state again.

- `Stop`: You effectively kill the actor. It will not be restarted.

- `Escalate`: If you choose to Escalate, you probably don’t know how to deal with the failure at hand. You delegate the decision about what to do to your own parent actor, hoping they are wiser than you. If an actor escalates, they may very well be restarted themselves by their parent, as the parent will only decide about its own child actors.



#### The default strategy

You don’t have to specify your own supervisor strategy in each and every actor. In fact, we haven’t done that so far. This means that the default supervisor strategy will take effect. It looks like this:

```scala
final val defaultStrategy: SupervisorStrategy = {
  def defaultDecider: Decider = {
    case _: ActorInitializationException ⇒ Stop
    case _: ActorKilledException         ⇒ Stop
    case _: Exception                    ⇒ Restart
  }
  OneForOneStrategy()(defaultDecider)
}
```

This means that for exceptions other than ActorInitializationException or ActorKilledException, the respective child actor in which the exception was thrown will be restarted.

Hence, when a PaperJamException occurs in our Register actor, the supervisor strategy of the parent actor (the barista) will cause the Register to be restarted, because we haven’t overridden the default strategy.

If you try this out, you will likely see an exception stacktrace in the log, but nothing about the Register actor being restarted.

Let’s verify that this is really happening. To do so, however, you will need to learn about the actor lifecycle.


#### The actor lifecycle

To understand what the directives of a supervisor strategy actually do, it’s crucial to know a little bit about an actor’s lifecycle. Basically, it boils down to this: when created via actorOf, an actor is started. It can then be restarted an arbitrary number of times, in case there is a problem with it. Finally, an actor can be stopped, ultimately leading to its death.

There are numerous lifecycle hook methods that an actor implementation can override. It’s also important to know their default implementations. Let’s go through them briefly:

- **preStart**: Called when an actor is started, allowing you to do some initialization logic. The default implementation is empty.
- postStop: Empty by default, allowing you to clean up resources. Called after stop has been called for the actor.
- preRestart: Called right before a crashed actor is restarted. By default, it stops all children of that actor and then calls postStop to allow cleaning up of resources.
- postRestart: Called immediately after an actor has been restarted. Simply calls preStart by default.


Let’s see if our Register gets indeed restarted upon failure by simply adding some log output to its postRestart method. Make the Register type extend the ActorLogging trait and add the following method to it:

```scala
override def postRestart(reason: Throwable) {
  super.postRestart(reason)
  log.info(s"Restarted because of ${reason.getMessage}")
}
```

Now, if you send the two Customer actors a bunch of CaffeineWithdrawalWarning messages, you should see the one or the other of those log outputs, confirming that our Register actor has been restarted.


#### Death of an actor

Often, it doesn’t make sense to restart an actor again and again – think of an actor that talks to some other service over the network, and that service has been unreachable for a while. In such cases, it is a very good idea to tell Akka how often to restart an actor within a certain period of time. If that limit is exceeded, the actor is instead stopped and hence dies. Such a limit can be configured in the constructor of the supervisor strategy:


```scala
import scala.concurrent.duration._
import akka.actor.OneForOneStrategy
import akka.actor.SupervisorStrategy.Restart
OneForOneStrategy(10, 2.minutes) {
  case _ => Restart
}
```


#### The self-healing system?

So, is our system running smoothly, healing itself whenever this damn paper jam occurs? Let’s change our log output:

```scala
override def postRestart(reason: Throwable) {
  super.postRestart(reason)
  log.info(s"Restarted, and revenue is $revenue cents")
}
```

And while we are at it, let’s also add some more logging to our Receive partial function, making it look like this:

```scala
def receive = {
  case Transaction(article) =>
    val price = prices(article)
    sender ! createReceipt(price)
    revenue += price
    log.info(s"Revenue incremented to $revenue cents")
}
```

Ouch! Something is clearly not as it should be. In the log, you will see the revenue increasing, but as soon as there is a paper jam and the Register actor restarts, it is reset to 0. This is because restarting indeed means that the old instance is discarded and a new one created as per the Props we initially passed to actorOf.

Of course, we could change our supervisor strategy, so that it resumes in case of a PaperJamException. We would have to add this to the Barista actor:

```scala
val decider: PartialFunction[Throwable, Directive] = {
  case _: PaperJamException => Resume
}
override def supervisorStrategy: SupervisorStrategy =
  OneForOneStrategy()(decider.orElse(SupervisorStrategy.defaultStrategy.decider))
```

Now, the actor is not restarted upon a PaperJamException, so its state is not reset.


#### Error kernel

So we just found a nice solution to preserve the state of our Register actor, right?

Well, sometimes, simply resuming might be the best thing to do. But let’s assume that we really have to restart it, because otherwise the paper jam will not disappear. We can simulate this by maintaining a boolean flag that says if we are in a paper jam situation or not. Let’s change our Register like so:


```scala
class Register extends Actor with ActorLogging {
  import Register._
  import Barista._
  var revenue = 0
  val prices = Map[Article, Int](Espresso -> 150, Cappuccino -> 250)
  var paperJam = false
  override def postRestart(reason: Throwable) {
    super.postRestart(reason)
    log.info(s"Restarted, and revenue is $revenue cents")
  }
  def receive = {
    case Transaction(article) =>
      val price = prices(article)
      sender ! createReceipt(price)
      revenue += price
      log.info(s"Revenue incremented to $revenue cents")
  }
  def createReceipt(price: Int): Receipt = {
    import util.Random
    if (Random.nextBoolean()) paperJam = true
    if (paperJam) throw new PaperJamException("OMG, not again!")
    Receipt(price)
  }
}
```

Also remove the supervisor strategy we added to the Barista actor.

Now, the paper jam remains forever, until we have restarted the actor. Alas, we cannot do that without also losing important state regarding our revenue.

This is where the error kernel pattern comes in. Basically, it is just a simple guideline you should always try to follow, stating that if an actor carries important internal state, then it should delegate dangerous tasks to child actors, so as to prevent the state-carrying actor from crashing. Sometimes, it may make sense to spawn a new child actor for each such task, but that’s not a necessity.

The essence of the pattern is to keep important state as far at the top of the actor hierarchy as possible, while pushing error-prone tasks as far to the bottom of the hierarchy as possible.

Let’s apply this pattern to our Register actor. We will keep the revenue state in the Register actor, but move the error-prone behaviour of printing the receipt to a new child actor, which we appropriately enough call ReceiptPrinter. Here is the latter:

```scala
object ReceiptPrinter {
  case class PrintJob(amount: Int)
  class PaperJamException(msg: String) extends Exception(msg)
}
class ReceiptPrinter extends Actor with ActorLogging {
  var paperJam = false
  override def postRestart(reason: Throwable) {
    super.postRestart(reason)
    log.info(s"Restarted, paper jam == $paperJam")
  }
  def receive = {
    case PrintJob(amount) => sender ! createReceipt(amount)
  }
  def createReceipt(price: Int): Receipt = {
    if (Random.nextBoolean()) paperJam = true
    if (paperJam) throw new PaperJamException("OMG, not again!")
    Receipt(price)
  }
}
```

Again, we simulate the paper jam with a boolean flag and throw an exception each time someone asks us to print a receipt while in a paper jam. Other than the new message type, PrintJob, this is really just extracted from the Register type.

This is a good thing, not only because it moves away this dangerous operation from the stateful Register actor, but it also makes our code simpler and consequently easier to reason about: The ReceiptPrinter actor is responsible for exactly one thing, and the Register actor has become simpler, too, now being only responsible for managing the revenue, delegating the remaining functionality to a child actor:


```scala
class Register extends Actor with ActorLogging {
  import akka.pattern.ask
  import akka.pattern.pipe
  import context.dispatcher
  implicit val timeout = Timeout(4.seconds)
  var revenue = 0
  val prices = Map[Article, Int](Espresso -> 150, Cappuccino -> 250)
  val printer = context.actorOf(Props[ReceiptPrinter], "Printer")
  override def postRestart(reason: Throwable) {
    super.postRestart(reason)
    log.info(s"Restarted, and revenue is $revenue cents")
  }
  def receive = {
    case Transaction(article) =>
      val price = prices(article)
      val requester = sender
      (printer ? PrintJob(price)).map((requester, _)).pipeTo(self)
    case (requester: ActorRef, receipt: Receipt) =>
      revenue += receipt.amount
      log.info(s"revenue is $revenue cents")
      requester ! receipt
  }
}
```


We don’t spawn a new ReceiptPrinter for each Transaction message we get. Instead, we use the default supervisor strategy to have the printer actor restart upon failure.

One part that merits explanation is the weird way we increment our revenue: First we ask the printer for a receipt. We map the future to a tuple containing the answer as well as the requester, which is the sender of the Transaction message and pipe this to ourselves. When processing that message, we finally increment the revenue and send the receipt to the requester.

The reason for that indirection is that we want to make sure that we only increment our revenue if the receipt was successfully printed. Since it is vital to never ever modify the internal state of an actor inside of a future, we have to use this level of indirection. It helps us make sure that we only change the revenue within the confines of our actor, and not on some other thread.

Assigning the sender to a val is necessary for similar reasons: When mapping a future, we are no longer in the context of our actor either – since sender is a method, it would now likely return the reference to some other actor that has sent us a message, not the one we intended.

Now, our Register actor is safe from constantly being restarted, yay!

Of course, the very idea of having the printing of the receipt and the management of the revenue in one place is questionable. Having them together came in handy for demonstrating the error kernel pattern. Yet, it would certainly be a lot better to seperate the receipt printing from the revenue management altogether, as these are two concerns that don’t really belong together.


#### Timeouts

Another thing that we may want to improve upon is the handling of timeouts. Currently, when an exception occurs in the ReceiptPrinter, this leads to an AskTimeoutException, which, since we are using the ask syntax, comes back to the Barista actor in an unsuccessfully completed Future.

Since the Barista actor simply maps over that future (which is success-biased) and then pipes the transformed result to the customer, the customer will also receive a Failure containing an AskTimeoutException.

The Customer didn’t ask for anything, though, so it is certainly not expecting such a message, and in fact, it currently doesn’t handle these messages. Let’s be friendly and send customers a ComebackLater message – this is a message they already understand, and it makes them try to get an espresso at a later point. This is clearly better, as the current solution means they will never know that they will not get their espresso.

To achieve this, let’s recover from AskTimeoutException failures by mapping them to ComebackLater messages. The Receive partial function of our Barista actor thus now looks like this:



```scala
def receive = {
  case EspressoRequest =>
    val receipt = register ? Transaction(Espresso)
    receipt.map((EspressoCup(Filled), _)).recover {
      case _: AskTimeoutException => ComebackLater
    } pipeTo(sender)
  case ClosingTime => context.system.shutdown()
}
```


Now, the Customer actors know they can try their luck later, and after trying often enough, they should finally get their eagerly anticipated espresso.

#### Death Watch

Another principle that is important in order to keep your system fault-tolerant is to keep a watch on important dependencies – dependencies as opposed to children.

Sometimes, you have actors that depend on other actors without the latter being their children. This means that they can’t be their supervisors. Yet, it is important to keep a watch on their state and be notified if bad things happen.

Think, for instance, of an actor that is responsible for database access. You will want actors that require this actor to be alive and healthy to know when that is no longer the case. Maybe you want to switch your system to a maintenance mode in such a situation. For other use cases, simply using some kind of backup actor as a replacement for the dead actor may be a viable solution.

In any case, you will need to place a watch on an actor you depend on in order to get the sad news of its passing away. This is done by calling the watch method defined on ActorContext. To illustrate, let’s have our Customer actors watch the Barista – they are highly addicted to caffeine, so it’s fair to say they depend on the barista:


```scala
class Customer(coffeeSource: ActorRef) extends Actor with ActorLogging {
  import context.dispatcher

  context.watch(coffeeSource)

  def receive = {
    case CaffeineWithdrawalWarning => coffeeSource ! EspressoRequest
    case (EspressoCup(Filled), Receipt(amount)) =>
      log.info(s"yay, caffeine for ${self}!")
    case ComebackLater =>
      log.info("grumble, grumble")
      context.system.scheduler.scheduleOnce(300.millis) {
        coffeeSource ! EspressoRequest
      }
    case Terminated(barista) =>
      log.info("Oh well, let's find another coffeehouse...")
  }
}
```



We start watching our coffeeSource in our constructor, and we added a new case for messages of type Terminated – this is the kind of message we will receive from Akka if an actor we watch dies.

Now, if we send a ClosingTime to the message and the Barista tells its context to stop itself, the Customer actors will be notified. Give it a try, and you should see their output in the log.

Instead of simply logging that we are not amused, this could just as well initiate some failover logic, for instance.

-----


## Summary

In this part of the series, which is the second one dealing with actors and Akka, you got to know some of the important components of an actor system, all while learning how to put the tools provided by Akka and the ideas behind it to use in order to make your system more fault-tolerant.

While there is still a lot more to learn about the actor model and Akka, we shall leave it at that for now, as this would go beyond the scope of this series. In the next part, which shall bring this series to a conclusion, I will point you to a bunch of Scala resources you may want to peruse to continue your journey through Scala land, and if actors and Akka got you excited, there will be something in there for you, too.


To be continued...
