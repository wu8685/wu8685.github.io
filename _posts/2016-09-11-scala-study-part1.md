---
layout: post
title: Scala学习笔记(一)
categories: [scala]
tags: [scala, extends, with]
---

scala的官方文档写得太烂了。没办法，只能找点资料，或者看scala编译的class文件，反正最后都是java的语法，然后自己总结一下。

*为便于阅读，以下展示的编译后的class文件有做适当调整,且大部分的main函数的object编译内容没有展示*

# OO: Class & Object

## Hierarchy
基本的继承结构如下图，图片来自官网：
![scala继承结构]({{ "/img/posts/2016-09-11-scala-study-part1.md/hierarchy.png" | prepend: site.baseurl }})

`Object`只是AnyRef的一个子类。

## class defination
整个`class`的结构内都是构造函数，构造函数的参数直接跟在申明后面。在new对象的时候，里面的所有代码都会执行。
```scala
class Person(name: String) {

	println("person info: " + info())	 // 这里可以先引用info

	val age: Int = 18
	var hobby: String = "swimming"

	println("person info: " + info())	 // 这里申明变量后引用

	def info(): String = {
		"name: %s; age: %d; hobby: %s".format(name, age, hobby)
	}
}

object Person {

	def main(args: Array[String]): Unit = {
		new Person("Lilei")
	}
}

// output:
// person info: name: Lilei; age: 0; hobby: null	 <--- 变量都没被初始化
// person info: name: Lilei; age: 18; hobby: swimming
```

查看编译后的class文件可以知道大致原因：

```java
public class Person {

	private final String name;
	private final int age;
	private String hobby;

	public int age() {
		return this.age;
	}

	public String hobby() {
		return this.hobby;
	}

	public void hobby_$eq(String x$1) {
		this.hobby = x$1;
	}

	public Person(String name) {
		this.name = name;

		Predef$.MODULE$.println(new StringBuilder().append("person info: ").append(info()).toString());

		this.age = 18;
		this.hobby = "";

		Predef..MODULE$.println(new StringBuilder().append("person info: ").append(info()).toString());
	}

	public String info() {
		return new StringOps(Predef$.MODULE$.augmentString("name: %s; age: %d; hobby: %s")).format(Predef$.MODULE$.genericWrapArray(new Object[] { this.name, BoxesRunTime.boxToInteger(age()), hobby() }));
	}

	public static void main(String[] paramArrayOfString) {
		Person$.MODULE$.main(paramArrayOfString);
	}
}

public final class Person$
{
	public static final Person$ MODULE$;

	static {
		new Person$();
	}

	public void main(String[] args) {
		new Person("Lilei");
	}

	private Person$() {
		MODULE$ = this;
	}
}
```

可见，在类的定义中，除了`def`,**其他语句都是顺序执行的**
而且`object`申明的变量和方法都是在一个`object_name`+`$`命名的类中以static形式定义出来的

## Extends class
语法基本和java一样，有几个前提需要注意的：

1. `var`类型不能被子类重写
2. 从父类继承的变量通过直接引用的方式使用
3. 不能使用`super`去调用父类的变量，var和val类型的都不可以。
4. 因为上一点，所以之后对此变量的所有返回值，**都是重写后的值**，哪怕是调用的父类方法

scala中定义的继承关系
```scala
class Singer(name: String) {

	val prefix:String = "singer: "
	var gender:String = " male"

	def info(): String = {
		prefix + name
	}
}

class Star(name: String) extends Singer(name) {

	override val prefix: String = "star: "

	override def info(): String = {
		prefix + super.info() + gender
	}
}

object Main {

	def main(args: Array[String]): Unit = {
		val star = new Star("Lilei")
		println(star.info())
	}
}

// output:
// star: star: Lilei male
```

这样的输出其实跟java语言的期望是不一样的。接着在编译出的class文件里找原因：

```java
public class Singer {

	private final String name;

	public String prefix() {
		return this.prefix;
	}

	private final String prefix = "singer: ";

	public String gender() {
		return this.gender;
	}

	public void gender_$eq(String x$1) {
		this.gender = x$1;
	}

	private String gender = " male";

	public String info() {
		return new StringBuilder().append(prefix()).append(this.name).toString();
	}

	public Singer(String name) {
		this.name = name;
	}
}

public class Star extends Singer {

	public Star(String name) {
		super(name);
	}

	public String prefix() {
		return this.prefix;
	}

	private final String prefix = "star: ";

	public String info() {
		return new StringBuilder().append(prefix()).append(super.info()).append(gender()).toString();
	}
}
```
可以看到：

1. 父类的属性是通过调用`super()`进行初始话的，这点和java一样
2. 父类的属性都是`private`修饰符，对外暴露`public`的getter。如果子类重写父类的属性，那么对应的getter就是子类重写的方法，且只会返回子类的属性值，所以父类的就丢掉了，即**子类继承的属性与父类脱离关系**。

## Trait
trait类似于java中的接口，但支持部分的实现。一个`trait`其实是被编译成了对应的接口和抽象类两个文件。

```scala
trait Singer {

	val song:String = "sunshine"

	def singe(): Unit = {
		println("singing " + song)
	}

	def name(): String
}

class Star extends Singer {

	override def name(): String = {
		return "Lilei"
	}
}

object Star {

	def main(args: Array[String]): Unit = {
		val s = new Star()
		s.singe()
		println(s.name())
	}
}

// output:
// singing sunshine
// Lilei
```

编译出的class文件：
```java
public abstract interface Singer {

	public abstract void com$study$classdef$witht$Singer$_setter_$song_$eq(String paramString);

	public abstract String song();

	public abstract void singe();

	public abstract String name();
}

public abstract class Singer$class {

	public static void $init$(Singer $this) {
		$this.com$study$classdef$witht$Singer$_setter_$song_$eq("sunshine");
	}

	public static void singe(Singer $this) {
		Predef..MODULE$.println(new StringBuilder().append("singing").append($this.song()).toString());
	}
}

public class Star implements Singer {
	private final String song;

	public String song() {
		return this.song;
	}

	public void com$study$classdef$witht$Singer$_setter_$song_$eq(String x$1) {
		this.song = x$1;
	}

	public void singe() {
		Singer$class.singe(this);
	}

	public Star() {
		Singer$class.$init$(this);
	}

	public String name() {
		return "Lilei";
	}
}
```
可以看到：

* 在Star的构造函数中会调用trait的抽象类的static `$init$()`方法来初始化从trait继承来的song属性。继承来的song属性被申明到了Star中，**与scala中的trait已经脱离了关系**。在java中，interface里申明的都是static变量，所以scala这样实现也是很合理的。


## Mixin Class Composition

scala中除了标准的继承，还能通过`with`关键字组合`trait`类型。


```scala
class Person(name: String, age: Int) {

	println("person info: " + info())

	def info(): String = {
		"name: %s; age: %d".format(name, age)
	}
}

trait Child {

	println("child info: " + info())

	def info(): String = {
		" I am a child"
	}

	def play(): Unit
}

trait Chinese {

	println("chinese info: " + info())

	def info(): String = {
		"I am a Chinese, %s, %s".format(province(), gender)
	}

	def province(): String = {
		"Hunan"
	}

	val gender: String	= "male"
}

class Student(name: String, age: Int, grade: Int) extends Person(name, age) with Child with Chinese {

	println("student info: " + info())

	override def info(): String = {
		super.info() + " grade: %d".format(grade)
	}

	override def play(): Unit = {}
}

object Student {

	def main(args: Array[String]): Unit = {
		new Student("Lilei", 18, 1)
	}
}

// output:
// person info: I am a Chinese, Hunan, null grade: 1
// child info: I am a Chinese, Hunan, null grade: 1
// chinese info: I am a Chinese, Hunan, null grade: 1
// student info: I am a Chinese, Hunan, male grade: 1
```

编译的class文件是这样的：
```java
public abstract interface Child {

	public abstract String info();

	public abstract void play();
}

public abstract class Child$class {

	public static void $init$(Child $this) {
		Predef$.MODULE$.println(new StringBuilder().append("child info: ").append($this.info()).toString());
	}

	public static String info(Child $this) {
		return " I am a child";
	}
}

public abstract interface Chinese {

	public abstract void com$study$classdef$Chinese$_setter_$gender_$eq(String paramString);

	public abstract String info();

	public abstract String province();

	public abstract String gender();
}

public abstract class Chinese$class
{
	public static String info(Chinese $this) {
		return new StringOps(Predef..MODULE$.augmentString("I am a Chinese, %s, %s")).format(Predef..MODULE$.genericWrapArray(new Object[] { $this.province(), $this.gender() }));
	}

	public static String province(Chinese $this) {
		return "Hunan";
	}

	public static void $init$(Chinese $this) {
		Predef$.MODULE$.println(new StringBuilder().append("chinese info: ").append($this.info()).toString());

		$this.com$study$classdef$Chinese$_setter_$gender_$eq("male");
	}
}

public class Person {

	private final String name;

	public Person(String name, int age) {
		Predef$.MODULE$.println(new StringBuilder().append("person info: ").append(info()).toString());
	}

	public String info() {
		return new StringOps(Predef$.MODULE$.augmentString("name: %s; age: %d")).format(Predef..MODULE$.genericWrapArray(new Object[] { this.name, BoxesRunTime.boxToInteger(this.age) }));
	}
}

public class Student extends Person implements Child, Chinese {

	private final String gender;

	public String gender() {
		return this.gender;
	}

	public void com$study$classdef$Chinese$_setter_$gender_$eq(String x$1) {
		this.gender = x$1;
	}

	public String province() {
		return Chinese.class.province(this);
	}

	public Student(String name, int age, int grade) {
		super(name, age);
		Child$class.$init$(this);
		Chinese$class.$init$(this);

		Predef$.MODULE$.println(new StringBuilder().append("student info: ").append(info()).toString());
	}

	public String info() {
		return new StringBuilder().append(Chinese.class.info(this)).append(new StringOps(Predef..MODULE$.augmentString(" grade: %d")).format(Predef..MODULE$.genericWrapArray(new Object[] { BoxesRunTime.boxToInteger(this.grade) }))).toString();
	}

	public static void main(String[] paramArrayOfString) {
		Student..MODULE$.main(paramArrayOfString);
	}

	public void play() {}
}
```

可以看出，这只是前面extends和trait两种情况的复合版。在子类student的构造函数中，会顺序执行`super()`和`with`trait的`$init$()`方法。结合前面`trait`的实现，可以知道一定是后一个父类/trait的变量的值会作为新值，赋值给子类的同名变量。
并且从`info()`的实现可以看到`super`引用被翻译成了`Chinese`，也就是后一个父类/trait定义的方法也会覆盖前一个。

## 总结：继承的特点

1. 继承`class A extends B with C with D`其实可以看作是`class A extends << class B with C with D >>`,切对于这个*父类*`class B with C with D`:
	1. 多重继承的关键字`with`只使用于`trait`
	2. 当出现多重继承，后面`trait`申明的属性和方法会覆盖前面的类/trait
	最终组成的新class，作为`class A`的*父类*
2. 从*父类*继承的属性会作为子类独立的属性被引用，且子类无法通过`super`引用*父类*的属性。
3. 对于方法的继承，一切同java一致


## Duck Typing

只要会鸭子叫，就视为鸭子。

```scala
class Person {

	def singe(): Unit = {
		println("singing")
	}
}

object Main {

	def singe(singer: {def singe():Unit}): Unit = {	 // 只要实现的singe方法就可以作为参数
		singer.singe()
	}

	def main(args: Array[String]): Unit = {
		val person = new Person()
		singe(person)
	}
}

// output:
// singing
```
这里需要注意的是：因为得是鸭子叫，所以方法名也必须一致。

## Currying

柯里化。不废话了，看例子。

```scala
object Main {

	def currying(left: Int)(middle: Int)(right: Int): Int = left + middle + right

	def main(args: Array[String]): Unit = {
		val two = currying(2)(_)
		println(two(3)(4))

		val towPlusThree = two(3)
		println(towPlusThree(4))
	}
}

// output:
// 9
// 9
```

函数可以依次传入多个参数，切每传入一个参数就能生成一个可多次使用的新函数。

## Generic

比较简单的例子

```scala
class Valuable(value:Int) {

	def getValue(): Int = value
}

object Comparable {

	def add[E <: { def getValue():Int }](l: E, r: E):Int = l.getValue() + r.getValue()

	def main(args: Array[String]): Unit = {
		println(add(new Valuable(1), new Valuable(2)))
	}
}

// output:
// 3
```