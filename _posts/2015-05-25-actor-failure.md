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

更令人感兴趣的部分是在 `Barista` 类中。
为了使用Akka 的**询问**语句而且让他返回`future`变量，我们需要导入`dispatcher`、`ask` 和 `pipe` ，并定义一个 隐式的 `timeout` 值：
当咖啡师接收到一条 `EspressoRequest` 浓缩咖啡订单后，我们以询问的方式发送一个 `Transaction` 交易消息给 `Register` 收银机，并等待其返回一个 `Receipt` 收据消息。
获取得的收据随后会和一杯倒满了咖啡的的咖啡杯一起以一个元组 `(EspressoCup, Receipt)` 的形式输送给 `EspressoRequest` 的发送者（也就是顾客）。
这种委托子任务给子行动者、并把他们的任务整合或修整的策略，正是典型的基于行动者的应用的处理方式。

并且，请注意我们如何通过调用 `ActorContext` 而不是用 `ActorSystem` 的 `actorOf` 方法创建子行动者。
通过这样做，我们创建的行动者成为了调用者的子行动者，避免其成为守卫行动者子女的顶级的行动者。

最后，下面是 `Customer` 行动者的定义。像是 `Barista` ，他也是顶级的、仅处于守卫行动者之下的行动者：

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

上面的代码中，我们首次使用了 `ActorLogging`，它允许我们把信息打印到日志类中，而不是到控制台。

现在，如果我们创建一个行动者系统，并在其中填入一个咖啡师和两个顾客，我们就可以开心的从这两个需要解决咖啡瘾的顾客中榨取黑金了：

```scala
import Customer._
val system = ActorSystem("Coffeehouse")
val barista = system.actorOf(Props[Barista], "Barista")
val customerJohnny = system.actorOf(Props(classOf[Customer], barista), "Johnny")
val customerAlina = system.actorOf(Props(classOf[Customer], barista), "Alina")
customerJohnny ! CaffeineWithdrawalWarning
customerAlina ! CaffeineWithdrawalWarning
```

如果你试着运行上面的代码，你可以看到这由满意的顾客们产生的两条日志记录。

----

## To crash or not to crash?

当然，这篇文章的主题不是关于满意的顾客，而是糟糕的事情发生的时我们应如何应对。

假设我们的收银机是个脆弱的设备 - 它的打印功能不是很靠得住。
时常的，小票纸会卡住机器，导致打印失败。
我们在收银机的伴生对象里加入一个 `PaperJamException` 卡纸异常：

```scala
class PaperJamException(msg: String) extends Exception(msg)
```

然后，我们相应的改变 `createReceipt` 生成收据方法：

```scala
def createReceipt(price: Int): Receipt = {
  import util.Random
  if (Random.nextBoolean())
    throw new PaperJamException("OMG, not again!")
  Receipt(price)
}
```

现在，当处理 `Transaction` 交易消息时，收银机会以50%的几率抛出一个 `PaperJamException` 异常。

这会怎样影响我们的行动者系统乃至整个应用呢？
幸运的是，Akka是个很强健的系统，而且不会受我们代码中的异常影响。
当异常出现时，产生异常的子行动者的父行动者会接到通知 - 还记得我们之前提到过，父行动者会监视它的子行动者们吗？
这个时候，就是由父行动者决定应采取什么样的措施来处理子行动者的异常了。

#### 监护人的异常处理策略

当接收到子行动者产生了异常的通知时，父行动者不是在 `onReceive` 方法中处理子行动者的失败行为的，因为这会混淆父行动者自己的正常处理逻辑。
就是说，处理自身的正常消息的逻辑和处理子行动者失败行为的逻辑是完全分开的。

每一个行动者都可以定义一个他自己的 **监护人策略**。它向Akka声明了当子行动者出现某种异常出现时，父行动者应该如何处理。

基本上来说，我们会使用两种监护人策略：`OneForOneStrategy` 和 `AllForOneStrategy`。
选择前者，意味着处理一个子行动者时只会影响到这一个子行动者，反之就会影响所有的子行动者。
使用哪种策略应由你的实际使用情况决定。

在选择使用哪种 `SupervisorStrategy` 策略以外，你还需要给你的行动者指明一个 `Decider` (`PartialFunction[Throwable, Directive]` 的别名)。
定义它你可以为每种异常决定一个或所有的子行动者出现异常时需要做一些什么。

#### 基本指令

下面是可供选择的基本指令：

```scala
sealed trait Directive
case object Resume extends Directive
case object Restart extends Directive
case object Stop extends Directive
case object Escalate extends Directive
```

- **Resume**：如果你选择了继续, 也许就意味着你认为你的子行动者过于鸡婆，觉得她们抛出的异常可以忽略。
子行动者们这时就会的继续处理异常，就像是什么也没发生过一样。

- **Restart**：重启指令会使得Akka为你创建一个或者多个新的子行动者。
这样做的一个原因之一是你假设了你的子行动者们会在抛出异常时，内部的状态就已经不稳定了，而且不能够继续处理更多的信息。
通过重启行动者，你希望会使得行动者重新进入一个干净的运行状态。

- **Stop**：直接杀死行动者，他们就不能被重启了。

- **Escalate**: 如果你选择了升级（指把错误递交给父行动者去处理），也许这一意味着这个行动者不知道如何去处理子行动者的异常。
通过把异常传给上级，你把处理异常的决定委托给了他的上一级的父行动者，并祈祷他比你更擅长处理它。
不过这样做后，行动者自己也许会被他的上级重启，因为行动者们只知道如何重启他的子行动者，而不能直接重启隔了两代的行动者。

#### 默认策略

你不必在每个行动者里指明一个监护人策略。
实际上，我们到目前为止都没主动的这样做过。
这意味着默认的监护人策略在起作用，像是这样：

```scala
final val defaultStrategy: SupervisorStrategy = {
  def defaultDecider: Decider = {
    case _: ActorInitializationException => Stop
    case _: ActorKilledException         => Stop
    case _: Exception                    => Restart
  }
  OneForOneStrategy()(defaultDecider)
}
```

这意味着，除了 `ActorInitializationException` 和 `ActorKilledException`，抛出其他异常的子行动者会被自动重启。

因此，当 `PaperJamException` 异常发生时，由于我们没有指定监护人策略，根据收银机的父行动者（也就是咖啡师）的默认策略，收银机会自动被重启。

如果你试过运行代码，你会在日志中发现一个异常的对战追踪信息，但收银机被重启的消息并没有出现在日志中。

为了验证一下到底发生了什么，我们先来学习一下行动者的生命周期。

#### 行动者的生命周期

为了理解监护人策略的每种指令，我们需要了解一点行动者生命周期的一些知识。
基本上，可以被分解为如下：
当行动者通过 `actorOf` 方法创建后，行动者开始运作；
他可以在错误出现时被重启任意次；
最后当行动者被停止是，也就意味着他迎来了它的死亡。

一个行动者生命周期有多个方法可以被重载，并且了解他们的默认实现也很重要。
让我们简略的过一遍这几个方法：

- **preStart**: 预开始阶段，会在行动者 `start` 即将启动前被调用，允许你做一些初始化逻辑。默认实现为空。
- **postStop**: 后停止阶段，在 `stop` 停止方法被调用后被调用，允许你做一些资源清理工作。默认实现为空。
- **preRestart**: 预重启阶段，会在一个崩溃的行动者 `restart` 即将重启前被调用。
默认实现中，此方法会停掉所有的他的子行动者，并调用 `postStop` 方法以清理资源。
- **postRestart**: 后重启阶段，会在行动者刚刚重启完成后被调用。默认实现为调用 `preStart` 方法。

通过在 `postRestart` 方法中加入一些日志输出信息，让我们看一下我们的收银机是否真的在出现错误时被重启了。
给 `Register` 加入对于 `ActorLogging` 的继承，并加上下面的方法：


```scala
override def postRestart(reason: Throwable) {
  super.postRestart(reason)
  log.info(s"Restarted because of ${reason.getMessage}")
}
```
现在，如果你给两个顾客行动者发送一堆 `CaffeineWithdrawalWarning` 消息，你会在日志中看到几条可以确定收银机有时被重启了的信息。

#### 行动者的死亡

很多时候，不停地重启同一个行动者在道理上说不通。
比方说，一个行动者需要与网络上的服务进行交互，但服务器有时会在很长时间内没有作出应答。
在这种情况下，让Akka在一定时间内重启行动者是个好主意；超时后，行动者就会被停掉，也就让他赢来了死亡。
这个时间限制可以通过监护人策略的构造函数配置：

```scala
import scala.concurrent.duration._
import akka.actor.OneForOneStrategy
import akka.actor.SupervisorStrategy.Restart
OneForOneStrategy(10, 2.minutes) {
  case _ => Restart
}
```

#### 可自愈的系统？

我们的系统是否能平稳的运行，并在卡纸的时候自我修复么？
让我们修改一下后重启阶段的日志输出：

```scala
override def postRestart(reason: Throwable) {
  super.postRestart(reason)
  log.info(s"Restarted, and revenue is $revenue cents")
}
```

再加入一点输出信息给 `receive` 函数，像是这样：

```scala
def receive = {
  case Transaction(article) =>
    val price = prices(article)
    sender ! createReceipt(price)
    revenue += price
    log.info(s"Revenue incremented to $revenue cents")
}
```

啊哦！好像有些东西没搞对。
在日志中，你会看到收入额会逐步提高，但只要收银机卡纸重启后，销售额就被重置为0.
这是因为重启一个行动者意味着之前的实例就被完全抛弃，以一个全新的通过 `actorOf()` 调用生成的行动者替代。

当然，我们可以改变监护人策略，让他在 `PaperJamException` 异常抛出时直接继续运行。
我们可以把下面的代码加进咖啡师的定义内：

```scala
val decider: PartialFunction[Throwable, Directive] = {
  case _: PaperJamException => Resume
}
override def supervisorStrategy: SupervisorStrategy =
  OneForOneStrategy()(decider.orElse(SupervisorStrategy.defaultStrategy.decider))
```

现在，收银机在卡纸后不会被重启，它的状态也就不会被重置了。

#### 错误核心模式

这是否意味着我们有了一个保持收银机状态的好的解决方案了呢？

有时候，简单的恢复行动者的运行状态是最好的解决思路。
不过假设我们真的需要重启一个收银机，因为不重启也就意味着卡住的纸不会自己消失。
我们可以通过加入一个布尔标志位来模拟一下收银机是否处于卡纸状态。
如下，将 `Register` 收银机行动者改为：

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
同时移除之前加到咖啡师里的监护者策略。

现在，卡纸状况会永远保持，直到我们重启了收银机行动者。
但是我们也不能简单地重启他，因为这会导致营业额的重置。

这时候就需要引入**错误核心模式** (error kernel pattern) 概念了。
他的含义是，当你的行动者内包含着重要的状态的时候，应把危险的任务交给子行动者去做，这样就能避免携带状态的行动者在崩溃时会导致的问题了。
有时候，为每个类似的任务创建一个新的子行动者是有道理的，但这不是必须的。
基本上，你应该总是效仿这种模式设计行动者系统的层级结构。

这种设计模式的基本元素是保证最重要的系统状态处于行动者架构越高层越好，并且将错误尽可能的压在架构的底层。

让我们为我们的收银机行动者实现这种模式。
我们依然让收银机保持营业额状态，但将容易出错的打印收据的行为放入一个新的子行动者 `ReceiptPrinter` 内。
`ReceiptPrinter` 的定义如下：

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

再一次的，我们通过一个布尔标志位来模拟卡纸异常，并在卡纸的状态下打印收据时抛出一个异常。
抽出了收银机的打印逻辑后，我们在这里定义了一个新的消息类型 `PrintJob` 并处理这条消息。

这是一种比较好的处理方式，不仅是因为把危险的操作从持有重要状态的收银机行动者中抽出来，并且他让我们的代码也变得更清晰和阐述：
`ReceiptPrinter` 只负责打印收据，`Register` 也变得更清晰了 - 它只负责管理营业额，并把剩下的功能委托给子行动者：

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

我们不会在处理每个交易消息的时候都创建一个新的 `ReceiptPrinter`。
同时，我们利用默认的监护人策略，在错误出现时重启收据打印机。

上面的代码中，增加营业额的逻辑部分值得讨论一下：
首先，询问 (`ask`) 打印机以获取一张收据，然后映射询问函数的 `future` 的返回值，使其变为一个含有询问结果和请求者引用（也就是发送给收银机 `Transaction` 交易消息的行动者）的一个元组，随后将其传送给自己。
当这个元组消息被处理时，我们才增加销售额，最终将收据发送回给交易的请求者。

这样迂回的处理销售额的方式，是因为我们需要确保在收据成功打印后才增加销售额。
其遵循了一个重要原则：**从来不要在 `Future` 中修改一个行动者的内部状态**。
它使得我们能够确保营业额是在行动者的范围内增加的，而不是在其他的线程中被处理。

将发送者赋值给一个 `val` 定量是必要的，其原因类似：
映射一个未来对象时所在的上下文不是行动者的上下文。因为 `sender` 是一个方法，在映射函数内调用它时，返回值很可能会是发送什么别的消息的行动者，而不是我们想用的那个。

现在，我们的 `Register` 收银机可以安全自由的重启啦，✌️！

当然，将打印收据和管理营业额业务放在一起不是什么好主意。
我们为此是为了展示错误核心模式的一个用例。
不过，实际中将这两个业务分开处理要好的多，因为它们在概念上不是在一起的。

#### 超时设定

我们另一个需要改进的地方是对于超时的处理。
目前，当 `ReceiptPrinter` 中出现异常时，由于使用了 `ask`，这会引发 `AskTimeoutException` 询问超时异常，返回给 `Barista` 咖啡师一个没有成功完成的 `Future` 对象。

由于 `Barista` 简单的映射了未来对象成功时的结果，然后把处理后的结果传给顾客，这样就使得顾客对象接收到一个含有 `AskTimeoutException` 异常的 `Failure` 对象。

但是，`Customer` 并没有询问过任何事物，自然他也没期待接收到这样的消息。事实上，他目前也无法处理这样的消息。
我们以友好的方式，在出现超时异常时，发送给顾客们一条 `ComebackLater` 稍后再来消息。
这是一条顾客可以明白的消息，它会使得顾客会在之后的某个时间再来购买咖啡。
这样优化了之前的实现，因为我们可以使得顾客知道是否能取得想要的咖啡。

为了实现它，我们可以调用含有 `AskTimeoutException` 超时异常的 `Failure` 对象的 `recover` 方法，将它映射成一个 `ComebackLater` 消息。
`Barista` 咖啡师的接受部分函数修改如下：

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
现在，顾客行动者知道如果咖啡没有购买成功时，他们可以重复尝试多次，直到成功购买到咖啡来满足咖啡瘾。

#### 死亡监视

为保证系统的容错性，除子行动者之外，另一个重要原则是行动者需要监视着自己重要的依赖对象。

有时，你拥有一些需要依赖于其他的非自己子行动者的行动者。
这意味着我们不能使用监护人策略来处理依赖关系。
不过，我们依然需要有另一种方式监视其他行动者的状态，并在他们出问题时得到通知。

思考一下一个需要访问数据库的行动者。
你会有其他的行动者需要这个行动者处于可用的健康状态，并在它不可用时接收到通知。
也许你想在数据库访问行动者故障时时，将系统转入维护模式；或是简单的将死掉的行动者用某种备份行动者替换掉是更好地解决方案。

不管在哪种情况下，你都需要以某种方式监视行动者逝去的悲伤消息。
这可以通过调用定义在 `ActorContext` 上的 `watch` 方法实现。
为了展示，我们将 `Customer` 顾客改造一下，使其监视 `Barista` 咖啡师。
因为顾客们对咖啡因极为上瘾，因此让他们依赖于咖啡师是合情合理的：

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

我们在顾客的构造函数里使其开始监视 `coffeeSource`，并加入对于 `Terminated` 消息的处理 -
这是种我们可以从Akka中所监视的对象死亡时可以接收到的消息。

现在，如果我们发送一个 `ClosingTime` 打烊时间消息给咖啡师，咖啡师将告诉行动者上下文结束自己的生命，这时顾客就会接受到咖啡师死亡的消息了。
请读者自己尝试一下，你就会在控制台中看到顾客关于"重新找一家咖啡店"的日志信息。

你也可以以此为基础，加入更好的失败处理逻辑。

-----

## 总结

这是系列的第二篇关于行动者和Akka的文章。
你已经了解了一个行动者系统内几个重要的组成部分，还有如果利用Akka提供的工具和方法使得你的系统可以变得更有容错性。

不过还有还有很多你需要了解的Akka和行动者模型的知识，但我们把它留给你，因为这超出了此系列的范畴。
在下一部分，此系列将进入总结部分。我会为你提供一些Scala语言还有Akka工具包的学习资源。

Daniel Westheide · Mar 20th, 2013 
