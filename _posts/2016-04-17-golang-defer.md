---
layout: post
title: Golang defer
categories: [golang]
tags: [golang, defer]
---

话题起于同事在微信群里发了张图，是[Go Newsletter](https://twitter.com/golangweekly)于13日在Twitter上转的一个代码截图，问输出是啥？
代码如下，*输出结果在文末*：

```
package main

import "fmt"

type number int

func (n number) print()   { fmt.Println(n) }
func (n *number) pprint() { fmt.Println(*n) }

func main() {
	var n number

	defer n.print()
	defer n.pprint()
	defer func() { n.print() }()
	defer func() { n.pprint() }()

	n = 3
}
```
我如果不跑下代码，反正是不知道结果的。后面跑去看了下Twitter的留言，有个哥们儿说他写了5年go，还不知道会是这结果，哈哈。
周末有空补了下，基本上就是golang里面defer的用法。

# defer
官方给出的文档上介绍defer的执行有三条基本规则：

## 规则一：defer函数是在外部函数return后，按照后申明先执行（栈）的顺序执行的；

```
package main

import "fmt"

func main() {
	defer fmt.Println("1")
	defer fmt.Println("2")
	defer fmt.Println("3")
}
```

输出是

```
3
2
1
```

## 规则二：defer函数的参数的值，是在**申明defer时确定下来的**；
看几个例子吧，注意第一条规则。
这是一个*普通类型*的例子：

```
package main

import "fmt"

func main() {
	i := 0

	defer fmt.Println(i)  // 这也算是作为defer函数的参数
	defer func(j int) { fmt.Println(j) }(i)  // 作为参数
	defer func() { fmt.Println(i) }()  // 作为闭包（closure）进行引用

	i++
}
```

输出是：

```
1
0
0
```

如果是*引用类型*也是一样的道理：
* 当修改引用对象的属性时：

```
package main

import "fmt"

type Person struct {
	name string
}

func main() {
	person := &Person{"Lilei"}

	defer fmt.Println(person.name)  // person.name作为普通类型当做defer函数的参数
	defer fmt.Printf("%v\n", person)  // 引用类型作为参数
	defer func(p *Person) { fmt.Println(p.name) }(person)  // 同上
	defer func() { fmt.Println(person.name) }()  // 闭包引用，对引用对象属性的修改不影响引用

	person.name = "HanMeimei"
}
```

输出是：

```
HanMeimei
HanMeimei
&{HanMeimei}
Lilei
```

* 当修改引用本身时：

```
package main

import "fmt"

type Person struct {
	name string
}

func main() {
	person := &Person{"Lilei"}

	defer fmt.Println(person.name)  // 同上，person.name作为普通类型当做defer函数的参数
	defer fmt.Printf("%v\n", person)  // 作为defer函数的参数，申明时指向“Lilei”
	defer func(p *Person) { fmt.Println(p.name) }(person)  // 同上
	defer func() { fmt.Println(person.name) }()  // 作为闭包引用，随着person的改变而指向“HanMeimei”

	person = &Person{"HanMeimei"}
}
```

输出是：

```
HanMeimei
Lilei
&{Lilei}
Lilei
```

在defer函数申明时，对外部变量的引用是有两种方式的，分别是作为函数参数和作为闭包引用。作为函数参数，则在defer申明时就把值传递给defer，并被cache起来。作为闭包引用的话，则会在defer函数执行时根据整个上下文确定当前的值。

## 规则三：defer函数可以读取和修改外部函数申明的返回值。
同样给一个例子

```
package main

import "fmt"

func main() {
	fmt.Printf("output: %d\n", f())
}

func f() (i int) {
	defer fmt.Println(i)  // 参数引用
	defer func(j int) { fmt.Println(j) }(i)  // 同上
	defer func() { fmt.Println(i) }()  // 闭包引用
	defer func() { i++ }()  // 执行前，i=2
	defer func() { i++ }()  // 执行前，i=1

	i++

	return
}
```

有了之前的基础，这个输出可以轻松推测出来：

```
3
0
0
output: 3
```

defer的基本规则就这三条了。了解之后，瞬间明朗了。

## 最开始的答案
通过defer的三条规则（其实只要前两条）就能知道输出了：

```
3
3
3
0
```