---
layout: post
title: Scala学习笔记(二)：函数编程&集合
categories: [scala]
tags: [scala, functional, collection]
---

# Functional Programming

## lambda

匿名函数（至少python是叫lambda）算是FO的基本功能了。scala定义一个匿名函数按照如下格式：

```scala
object Main {

  def main(args: Array[String]): Unit = {
    val say = (sth: String) => {println("say: %s".format(sth))}
    say("hello")
    say("world")
  }
}

// output:
// say: hello
// say: world
```

## Using pattern matching

scala中，pattern matching的逻辑语句使用频率很高，配合case class可以代替if用在一些条件判断的情况

```scala
object Main {

  def main(args: Array[String]): Unit = {
    def fibonacci(in: Any): Int = {
      in match {
        case 0 => 0
        case 1 => 1
        case n: Int => fibonacci(n - 1) + fibonacci(n -2)
        case _ => 0
      }
    }

    println(fibonacci("ABC"))
    println(fibonacci(3))
  }
}

// output:
// 0
// 2
```

## Case Class

Case class是特殊的class：

1. Case class的创建不需要new操作符
2. Case class可作为case matching参数

```scala
object Main {

  def main(args: Array[String]): Unit = {
    trait Exp {}

    case class Value(value: Int) extends Exp {}
    case class Sum(l: Value, r: Value) extends Exp {}
    case class Sub(l: Value, r: Value) extends Exp {}

    def compute(exp: Exp): Int = {
      exp match {
        case Value(v) => v
        case Sum(l, r) => compute(l) + compute(r)
        case Sub(l, r) => compute(l) - compute(r)
        case _ => 0
      }
    }

    // 1 + 2
    val case1 = Sum(Value(1), Value(2))
    // 10 - 5
    val case2 = Sub(Value(10), Value(5))

    println(compute(case1))
    println(compute(case2))
  }
}

// output
// 3
// 5
```

## option and check null pointer

scala加入了option的概念。option用来封装一个值，并将java的null判断语句封装在其中。个人感觉，和java相比，option的引入带来的好处是将`null的判断`由optional级别提升到了mandatory：java中可能由于代码的不规范，有可能漏掉是否为null的判断，但是scala加了option，在写代码的时候，至少起到reminder的作用。

```scala
object Main {

  def main(args: Array[String]): Unit = {

    def printEnv = (name: String) => {

      val getEnv = (name: String) => {
        if (sys.env.contains(name)) {
          Some(sys.env(name))
        } else {
          None
        }
      }

      def handleEnv(name: String, opt: Option[String]): Unit = {
        opt match {
          case Some(h) => {
            println("%s=%s".format(name, h))            // get value through case matching
            println("%s get %s".format(name, opt.get))  // get value through get()
          }
          case None => {
            println("%s is undefined".format(name))     // get none value through case matching
          }
        }
        println("%s getOrElse %s".format(name, opt.getOrElse("none_env")))  //get value through getOrElse
      }

      handleEnv(name, getEnv(name))
    }

    printEnv("HOME")
    println()
    printEnv("JAVA_HOME")
  }
}

// output:
// HOME is undefined
// HOME getOrElse none_env

// JAVA_HOME=C:\Program Files\Java\jdk1.8.0_101
// JAVA_HOME get C:\Program Files\Java\jdk1.8.0_101
// JAVA_HOME getOrElse C:\Program Files\Java\jdk1.8.0_101
```

在通过`get()`拿值的时候，如果为空，则会抛出`NoSuchElementException`.

## Lazy initialization

在属性前添加`lazy`可以让该属性在第一次使用时才被初始化。
默认情况下，是在创建对象时初始化的：

```scala
object Main {

  def main(args: Array[String]): Unit = {
    class Person {
      val name = {			// without lazy
        println("init name...")
        "Lilei"
      }
    }

    val person = new Person()
    println("create a person")
    println("name: %s".format(person.name))
  }
}

// output:
// init name...
// create a person
// name: Lilei
```

添加`lazy`后则是使用时才初始化：

```scala
object Main {

  def main(args: Array[String]): Unit = {
    class Person {
      lazy val name = {			// without lazy
        println("init name...")
        "Lilei"
      }
    }

    val person = new Person()
    println("create a person")
    println("name: %s".format(person.name))
  }
}

// output:
// create a person
// init name...
// name: Lilei
```

## For loop and yield:

`for`循环和java基本一致，多了一些额外的增强。

### Simple For

1. 简单遍历index可以通过range的方式：`to`和`until`。`to`包含最后一个index，`until`不包括
2. 也可以通过range generator实现：

* 一个for循环申明也可以实现多个嵌套的循环

```scala
object Main {

  def main(args: Array[String]):Unit = {
    for (i <- 1 to 3; j <- 1 until 3) println("%d - %d".format(i, j))		// range & nested for

    println()
    val numbers = List(1,2,3,4,5,6)
    for (i <- numbers) println("%d".format(i))		// range generator
  }
}

// output:
// 1 - 1
// 1 - 2
// 2 - 1
// 2 - 2
// 3 - 1
// 3 - 2

// 1
// 2
// 3
// 4
// 5
// 6
```

### filter & yield

另外还多了`filter`和`yield`的概念。
`filter`即在for循环申明时可以多加些判断条件进行过滤
`yield`可以从for循环返回一个List

```scala
object Main {

  def main(args: Array[String]): Unit = {
    case class Person(name: String, age: Int)   // case class has no need to new it

    val persons: List[Person] = List(
      Person("Lilei",     14),
      Person("HanMeimei", 15),
      Person("Lily",      16),
      Person("Lucy",      17)
    )

    val adults = for (p<-persons  if p.name startsWith "L"; if p.age >= 16)   // filters
      yield p

    for (a <- adults) {
      println("%s is an adult starts with L".format(a.name))
    }
  }
}

// output:
// Lily is an adult starts with L
// Lucy is an adult starts with L
```

## implicit

implicit用来标示一个隐式转换。
转换的选择规则如下：

1. 编译器会选取当前作用域定义的implicit，如果某个implicit定义在包`scala.abc`下，则需要执行`import scala.abc._`
2. 编译器也会在companion内查找对应的implicit
3. 编译器只会调用一次implicit，不会出现嵌套调用

在如下三个情况下会选择隐式转换：

```scala
object Main {

  def main(args: Array[String]): Unit = {
    implicit def intToStr(i: Int): String = {
      "string(%d)".format(i)
    }

    // 1. when type mismatching
    def printStr(str: String): Unit = {
      println(str)
    }
    // output:
    // string(1)
    printStr(1)

    // 2. when being used as other type
    // output:
    // s
    println(1.charAt(0))

    // 3. when using curring func, a implicit arg can be detected
    implicit val implicitStr = "world"
    def printImpStr(l: String)(implicit r: String): Unit = {
      print(l)
      print(r)
    }

    // output:
    // helloworld
    printImpStr("hello")
  }
}
```

# Collection

scala中collecion分为`strict`和`lazy`两种，`lazy`的collection中的元素只会在使用的时候被分配内存，比如range
另外collection还可以被申明为`mutable`和`immutable`两种（immutable collection也能包含mutable元素），分别在`scala.collection`, `scala.collection.immutable`, `scala.collection.mutable`包中。

他们各自适应于不同的场景，`immutable`更适合与多线程场景。建议的做法是先申明为`immutable`的，当出现需要时，再改为`mutable`的。

## Collection API

`filter`，`MapReduce`方法可以参考：[https://www.tutorialspoint.com/scala/scala_collections.htm](https://www.tutorialspoint.com/scala/scala_collections.htm)

## Collection Operatin

默认情况下申明的collection都是`immutable`的，mutable collection需要导入包`scala.collection.mutable`

```scala
// create immutable collection
val list = List("a", "b")
val map = Map("a" -> 1, "b" -> 2)
val set = Set(1, 2)

// create mutable collection
import scala.collection.mutable._

val listBuffer = ListBuffer("a", "b")
val hashMap = HashMap("a" -> 1, "b" -> 2)
val hashSet = HashSet(1, 2)
```

`immutable`不支持append操作，对他们的操作会返回一个新的collection：

```scala
// add opertions: the original collections change nothing
val newList = list + "c"				// newList: ("a", "b", "c")
val newMap = map + ("c" -> 3)
val newSet = set + 3

// mutable
listBuffer += "c"
hashMap += ("c" -> 3)
hashSet += 3
```

* `+`，`-`：`immutable`和`mutable`都支持，自身不变，返回新的结果集合
* `+=`, `-=`：仅`mutable`支持
* 修改操作可通过`+`重复key实现
* `:::`：连接两个集合

```
val col = colA ::: colB
```

* Tail Recursion:

```scala
list match {
  case List() => init
  case head :: tail => ... 		// head is the first elem, tail is the rest
}
```

## Parallel Collection

通过`.par`方法可以返回当前集合的一个并行实现。

* 一般情况下，返回的都是一个copy，这会花费线性时间，并且新生成的集合和原集合的数据是不共享的。
* 但是对于`ParArray`和`mutable.ParHashMap`（HashMap.par的返回类型），是会共享原数据的，所以花费的时间是固定的。

```scala
object Parallel {

  def main(args: Array[String]):Unit = {
    val list = List(1, 2, 3, 4)
    val newList = list.par.map((e) => {e + 1})

    println(newList)
  }
}
```