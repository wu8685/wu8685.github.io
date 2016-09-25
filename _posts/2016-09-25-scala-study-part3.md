---
layout: post
title: Scala学习笔记(三)：Actor & Future
categories: [scala]
tags: [scala, actor, future]
---

关于scala的Actor和经常配合使用的Future

# Actor

## Creating Actor & best practice

* 步骤一：实现trait`akka.actor.Actor`即可创建一个Actor类型，其中的`receive()`方法，即处理接受信息的方法。

```scala
object Main {
  def main(args: Array[String]):Unit = {
    import akka.actor.Actor
    class MyActor extends Actor {

      override def receive: Unit = {
        case "hello" => println("hello")
        case _ => println("error mesg")
      }
    }
  }
}
```

* 步骤二：通过`akka.actor.Props`申明actor的构造函数

* 步骤三：通过`akka.actor.ActorSystem`或`akka.actor.ActorContext`创建actor：

```scala
import akka.actor._

class MyActor(param: Int) extends Actor {
  override def receive: Receive = {
    case _ => {
      // 2. create with ActorContext within a Actor
      context.actorOf(MyActor.props(param))
    }
  }
}

object MyActor {

  // Best Practice:
  // to declare what messages an Actor can receive in the companion object of the Actor
  case class Message1(content:String)
  case object Message2

  // Best Practice:
  // provide factory methods on the companion object of each Actor
  // which help keeping the creation of suitable Props as close to the actor definition as possible
  def props(param: Int): Props = {
    Props(new MyActor(param))
  }

  def main(args: Array[String]): Unit = {
    val props = Props(new MyActor(1))

    // 1. create top level Actor with ActorSystem
    // ActorSystem is a heavy object: create only one per application
    val system = ActorSystem("mySystem")
    val myActor1 = system.actorOf(props, "myactor")
  }
}
```

* `ActorSystem`是重量级组件，一个应用最好只创建一个。他创建的Actor都是top level的
* `ActorContext`用于在Actor内部创建子actor
* `actorOf`返回的是immutable的`ActorRef`，他是对actor的引用。
* `actorOf`的第二个参数是对actor的标示，该标示`不能为空，不能以$开头，但是可以包含URL encode的字符`

**Best Practice**
1. 在class的companion中创建`props`方法，返回对应actor class的`Props`对象
2. 在class的companion中创建`case class`或`case object`，用于枚举actor的信息类型

* [Dependency Injection](http://doc.akka.io/docs/akka/2.4/scala/actors.html#Dependency_Injection)可以用来解决创建actor的依赖问题，具体还没研究。

## Actor API

常用的api有如下几个

1. `self`，返回对自己的引用
2. `sender`，返回给当前actor发信息的引用，可以通过`sender() ! "hello"`回复信息。如果当前actor是被其他actor调用，则sender为调用的actor，如果不是actor调用，则sender为默认的`deadLetter` actor。
3. `supervisorStrategy`，可被用户重写，以实现对子actor的监控策略
4. `context`，ActorContext

还有一些lifecycle hook如下

```scala
def preStart(): Unit = ()

def postStop(): Unit = ()

def preRestart(reason: Throwable, message: Option[Any]): Unit = {
  context.children foreach { child ?
    context.unwatch(child)
    context.stop(child)
  }
  postStop()
}

def postRestart(reason: Throwable): Unit = {
  preStart()
}
```

## Actor Selection (local/remote)

当actor由ActorContext一级级往下创建时，就会形成一个类似目录的父子关系，如`/parent/child/grandchild`。其中可以是相对路径(`../myActor`)或者绝对路径(`/user/myActor`)。

* 绝对路径的起点是`/user`

```scala
import akka.actor.{Actor, ActorSystem, Props}
import akka.util.Timeout

import scala.concurrent.duration._

class MyActor extends Actor {
  override def receive: Receive = {
    case MyActor.Greeting => {
      sender() ! "hello"
    }
  }
}

object MyActor {

  case object Greeting

  def props(): Props = {
    Props(new MyActor())
  }

  def main(args: Array[String]): Unit = {
    import akka.pattern.ask
    implicit val timeout = Timeout(5 seconds)

    val system = ActorSystem("mySys")
    system.actorOf(MyActor.props(), "myactor")
    import system.dispatcher

    system.actorSelection("/user/myactor").resolveOne().onSuccess {
      case ma => {
        val future = ask(ma, MyActor.Greeting).mapTo[String]
        future onSuccess {
          case result => println(result)
        }
      }
    }

    Thread.sleep(1000)
    system.terminate()
  }
}

// output
// hello
```

具体参考：[http://doc.akka.io/docs/akka/2.4/scala/actors.html#Identifying_Actors_via_Actor_Selection](http://doc.akka.io/docs/akka/2.4/scala/actors.html#Identifying_Actors_via_Actor_Selection)

## Become/Unbecome

`become(PartialFunction[Any, Unit])`会用参数中的行为替换当前的行为，而`unbecome()`会还原成上一个行为。
这一对方法相当于压栈/出栈，所以需要注意栈溢出。
参考：[http://doc.akka.io/docs/akka/2.4/scala/actors.html#Become_Unbecome](http://doc.akka.io/docs/akka/2.4/scala/actors.html#Become_Unbecome)

## Send Message

可以通过下面两个方法发送信息：

1. `!`，fire-forget，即`tell`。这种方式不会被block住
2. `?`，send-and-receive-future，即`ask`。他会返回一个类似java `Future`的引用。信息的接受者必须通过`!`返回消息以**结束**Future。ask操作也需要创建一个handler来接受回复，以及一个`timeout`来处理超时后的资源回收。

`timeout`可以分两种方式定义：
显式：

```scala
import scala.concurrent.duration._
import akka.pattern.ask
val future = myActor.ask("hello")(5 seconds)
```

隐式：

```scala
import scala.concurrent.duration._
import akka.util.Timeout
import akka.pattern.ask
implicit val timeout = Timeout(5 seconds)
val future = myActor ? "hello"
```

同步ask

```scala
import akka.actor.{Actor, ActorSystem, Props}
import akka.pattern.ask
import akka.util.Timeout

import scala.concurrent.duration._
import scala.concurrent.Await

class MyActor extends Actor {
  override def receive: Receive = {
    case MyActor.Greeting => {
      sender() ! "hello"
    }
  }
}

object MyActor {

  case object Greeting

  def props(): Props = {
    Props(new MyActor())
  }

  def main(args: Array[String]): Unit = {
    val mySystem = ActorSystem("mySys")
    val ma = mySystem.actorOf(MyActor.props(), "myactor")

    implicit val timeout = Timeout(5 seconds)

		// use asInstanceOf
    val future = ma ? MyActor.Greeting

    val reply = Await.result(future, timeout.duration).asInstanceOf[String]		// this will block until future finishes
    println("waiting")
    println(reply)
    println("I m here")

		// using mapTo is better
    // val future = ask(ma, MyActor.Greeting).mapTo[String]
    // val reply = Await.result(future, timeout.duration)

    mySystem.terminate()
  }
}

// output:
// waiting
// hello
// I m here
```

* Await.result将会block住，知道future结束，或者timeout
* Actor返回的future是Future[Any]，所以这里需要`asInstanceOf`。或者通过`mapTo`返回一个指定类型的Future


## Forward Message

可以通过`target forward message`来转发消息，这样的话，中间actor就相当与一个router

## Receive Message

通过实现`akka.actor.Actor`的`receive`方法来接受信息。该方法返回PartialFunction，如case/match

```scala
type Receive = PartialFunction[Any, Unit]

def receive: Actor.Receive
```

## Stop Actor

通过`ActorSystem`和`ActorContext`的`stop`方法可以停止一个actor，`stop`方法是异步的。
一个actor执行stop分三个步骤：

1. actor挂起自己的mailbox，不再处理消息，并且给自己的子actor发送stop命令；
2. actor会开始等待子actor的停止通知，直到所有的actor都停止；
3. 最终停止自己

* 如果某个子actor没能停止成功，则停止过程会被stuck。

`ActorSystem.terminate`会停止ActorSystem

`graceful stop`：[http://doc.akka.io/docs/akka/2.4/scala/actors.html#Graceful_Stop](http://doc.akka.io/docs/akka/2.4/scala/actors.html#Graceful_Stop)

# Future

future是用来获取同步操作结果的，类似java

## mapTo

如上面的例子，可以通过`mapTo`将Future[Any]转换成Future[String]等其他类型。

## pipeTo

可以将Future的结果转发给一个Actor

```scala
import akka.pattern.pipe
import mySystem.dispatcher	// ExecutionContext

future pipeTo ma
// or
pipe(future) to ma
```

* `ExecutionContext`是Future运行需要的环境，类似java的Executor，每个Actor都被配置了`MessageDispatcher`，他也是一个ExecutionContext。所以可以通过import他来提供：

```scala
import mySystem.dispatcher	// the created ActorSystem
// or
import myContext.dispatcher	// the Actor's ActorContext
```

也可以自己创建：

```scala
import scala.concurrent.{ ExecutionContext, Promise }

implicit val ec = ExecutionContext.fromExecutorService(yourExecutorServiceGoesHere)

// Do stuff with your brand new shiny ExecutionContext
val f = Promise.successful("foo")

// Then shut your ExecutionContext down at some
// appropriate place in your program/application
ec.shutdown()
```

## Use Directly

直接创建使用

```scala
import scala.concurrent.Await
import scala.concurrent.Future
import scala.concurrent.duration._

val future = Future {
  "Hello" + "World"
}
future foreach println
```

## Callback

future支持设置对结果的handler

```scala
object Main {

  def main(args: Array[String]): Unit = {
    import scala.concurrent.Future
    import scala.util.Success
    import scala.util.Failure
    import scala.concurrent.ExecutionContext.Implicits.global

    val future = Future {
      val result = "hello"
      result + "world"
    }

    future onSuccess {
      case "hello" => println("sucess")
      case x => println("success: " + x)
    }

    future onSuccess {
      case "hello" => println("sucess")
      case x => println("success: " + x)
    }

    future onFailure {
      case ise: IllegalStateException if ise.getMessage == "OHNOES" =>
      case e: Exception =>
    }

    future onComplete {
      case Success(result)  => println("complete with success: " + result)
      case Failure(exception) => println("complete with failure: " + exception)
    }

    Thread.sleep(1000)
  }

}

// output:
// complete with success: helloworld
// success: helloworld
// success: helloworld
```

## Future in Order

通过`andThen`实现future序列，前面的future的结果会陆续作为后续future的输入

```scala
object Main {

  def main(args: Array[String]): Unit = {
    import scala.concurrent.Future
    import scala.util.Success
    import scala.util.Failure
    import scala.concurrent.ExecutionContext.Implicits.global

    Future {
      val result = "hello"
      result + "world"
    } andThen {
      case Success(result) => println("complete with success: " + result)
      case Failure(exception) => println("complete with failure: " + exception)
    } andThen {
      case result => println("here: " + result)
    }

    Thread.sleep(1000)
  }

}

// output:
// complete with success: helloworld
// here: Success(helloworld)
```

## flow

`fallbackTo`返回一个新的future，当第一个future失败，则拿第二个future的结果

```scala
object Main {

  def main(args: Array[String]): Unit = {
    import scala.concurrent.Future
    import scala.util.Success
    import scala.concurrent.ExecutionContext.Implicits.global

    case class CustomException(message: String = "", cause: Throwable = null)
      extends Exception(message, cause)

    val future1 = Future {
      throw new CustomException("whatever")
    }

    val future2 = Future {
      "hello"
    }

    val future = future1 fallbackTo future2

    future onComplete {
      case Success(result) => println(result)
    }

    Thread.sleep(1000)
  }

}

// outut:
// hello
```

`zip`返回一个新的future，可以将两个future的结果绑定到一个tuple

```scala
object Main {

  def main(args: Array[String]): Unit = {
    import scala.concurrent.Future
    import scala.concurrent.ExecutionContext.Implicits.global

    val future1 = Future {
      "hello"
    }

    val future2 = Future {
      "world"
    }

    val future = future1 zip future2
    future foreach println

    Thread.sleep(1000)
  }

}

// output
// (hello,world)
```

## Exception

`recover`可以catch future抛出的异常

```scala
object Main {

  def main(args: Array[String]): Unit = {
    import scala.concurrent.Future
    import scala.util.Success
    import scala.concurrent.ExecutionContext.Implicits.global

    case class CustomException(message: String = "", cause: Throwable = null)
      extends Exception(message, cause)

    val futureFail = Future {
      throw new CustomException("whatever")
    } recover {
      case e => "catch: " + e
    }

    futureFail onComplete {
      case Success(result) => println(result)
    }

    Thread.sleep(1000)

    val futureSucc = Future {
      "ok"
    } recover {
      case e => "catch: " + e
    }

    futureSucc onComplete {
      case Success(result) => println(result)
    }

    Thread.sleep(1000)
  }

}

// output
// catch: com.study.concurrency.actor.future.Main$CustomException$3: whatever
// ok
```

也可以用`recoverWith`返回一个新的future

```scala
object Main {

  def main(args: Array[String]): Unit = {
    import scala.concurrent.Future
    import scala.util.Success
    import scala.concurrent.ExecutionContext.Implicits.global

    case class CustomException(message: String = "", cause: Throwable = null)
      extends Exception(message, cause)

    val futureFail = Future {
      throw new CustomException("whatever")
    } recoverWith {
      case e => Future.failed[Int](new CustomException("fail"))		// use method of Future companion to create a future
      case _ => Future.successful("succ")
    }

    futureFail onComplete {
      case Success(result) => println(result)
    }

    Thread.sleep(1000)
  }

}
```

## After

`after`可以给future一个timeout，超时返回指定的值

```scala
// TODO after is unfortunately shadowed by ScalaTest, fix as part of #3759
// import akka.pattern.after

val delayed = akka.pattern.after(200 millis, using = system.scheduler)(Future.failed(
  new IllegalStateException("OHNOES")))
val future = Future { Thread.sleep(1000); "foo" }
val result = Future firstCompletedOf Seq(future, delayed)
```
