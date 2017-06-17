---
layout: post
title: Javascript对象创建
categories: [javascript]
tags: [js, object, create]
---

这几天花时间好好把js中的对象创建整理了一下，这块也是我之前比较薄弱的环节。

# 创建对象
在js中对于如下代码，有这样的引用关系

```
var Person = function(name) {
  this.name = name;
}

var person1 = new Person();
var person2 = new Person();

alert(Person == person1.constructor);  // true
alert(Person == person2.constructor);  // true
alert(Person.prototype.constructor == Person);  // true
```

也就是这样的引用关系
![JS Objects references 引用自《JavaScript高级程序设计》]({{ "/img/posts/2016-03-26-javascript-object-create/hierarchy.png" | prepend: site.baseurl }})

**Note**: 实际上，构造函数`Person constructor`和他的原型`prototype`是通过`Person constructor`.prototype和`prototype`.constructor相互引用的。因此实例person1和person2会有原型的属性constructor，指向构造函数`Person constructor`。

## 一般模式
这是用的最多的，也是最简单的方式。

```
var person = new Object();
person.name = 'Lilei';
person.age = 18;

alert(person.name);
alert(person.age);
```

或者简单点的方法就是

```
var person = {
  name: 'Lilei',
  age: 18
}

alert(person.name);
alert(person.age);
```

## 构造函数模式
先看代码

```
var Person = function(name) {
  this.name = name;
  this.age = 18;

  this.myName = function() {
    return this.name;
  }
}

var person = new Person('Lilei');
alert(person.myName());  // Lilei
alert(person.age);  // 18
```

new出person的过程中，实际上执行了一下这些操作：
1. 创建一个新的Object对象；
2. 将构造函数的作用域赋值给对象；（这时，this指针就指向了这个对象）
3. 执行构造函数中的代码；（这就为对象中添加了属性和方法）
4. 返回对象。

因此，构造函数方式创建的对象，他的所有属性和方法都是在新对象上的，而不定义在prototype上，所以同一个构造函数创建出来的对象，他们之间不会share相同的属性或方法。
这种方式有一个不足之处，就是构造函数中对this定义的函数也会有两个备份。

## 原型模式

```
var Person = function() {}

Person.prototype.name = 'Lilei';
Person.prototype.age = 18;

Person.prototype.myName = function() {
  return this.name;
}

var person = new Person();
alert(person.myName());
alert(person.age);
```

这种方式是直接把属性和方法都定义在了prototype上。这样，所有new出来的对象都会从prototype上继承到这些东西，他们share这些属性和方法。
这种方法解决了方法的share问题。
而属性的share问题呢？

```
var Person = function() {}

Person.prototype.name = 'Lilei';

var personA = new Person();
var personB = new Person();

personA.name = 'HanMeimei'  // 会在personA对象上创建name属性，因此不会去访问prototype上的name

alert(personA.name);  // HanMeimei
alert(personB.name);  // Lilei
alert(Person.prototype.name)  // Lilei
```

因为属性的查找链中，会先在当前对象上查找属性，如果没有就会去prototype上查找。而当修改personA的name时，会在personA上添加一个name属性。因此不会访问到share的name属性。而直接修改引用类型的属性呢。。。呵呵

```
var Person = function() {}

Person.prototype.addresses = ['addrA', 'addrB'];
Person.prototype.relationships = ['father', 'mother'];

var personA = new Person();
var personB = new Person();

personA.addresses.push('addrC');  // 会修改prototype上share的属性
personB.relationships = ['grandpa', 'grandma'];  // 会在对象上创建新的属性

alert(Person.prototype.addresses)  // addrA,addrB,addrC
alert(personA.addresses);  // addrA,addrB,addrC
alert(personB.addresses);  // addrA,addrB,addrC

alert(Person.prototype.relationships )  // father,mother
alert(personA.relationships );  // father,mother
alert(personB.relationships );  // grandpa,grandma
```

其实这个现象还是好理解的，也比较“合理”。
另外一个要**注意**的。如果想直接用json的方式对prototype赋值，以达到定义对象的作用时，一定要修改constructor，保证行为的一致：

```
var Person = function() {}

Person.prototype = {
  name: 'Lilei'
}

alert(Person.prototype.constructor == Person);  // false, 这时候constructor是Object

Person.prototype = {
  name: 'Lilei',
  constructor: Person  // 指向Person
}

alert(Person.prototype.constructor == Person);  // true
```

## 组合构造模式

就是对于share的属性或方法，通过原型模式定义，对于不share的，通过构造函数方式定义。

```
var Person = function(addresses) {
  this.addresses = addresses;  // no shared
}

// shared
Person.prototype.myAddresses = function() {
  return this.addresses;
}
```

## getter&setter

```
var person = {
  _age: 18,
  isAdult: false
}

Object.defineProperty(person, 'age', {
  get: function() {
    return this._age;
  },
  set: function(age) {
    this._age = age;
    if (this._age > 18) {
      this.isAdult = true;
    }
  }
})

person.age = 25;
alert(person.isAdult);  // true
```

这里为person定义一个属性**age**。在操作age时，实际上会调用他的getter和setter，去操作**age**属性。
当然，这里直接调用_age也是可以的- -!

## 寄生构造模式
长这样的

```
var Person = function(name) {
  var person = new Object();
  person.name = name;

  return person;  // return重写了new的操作
}

var person = new Person('Lilei');  // 这里还是用new，但是return的是构造函数里创建的对象
alert(person.name);  // Lilei
alert(person.constructor);  // 他已经是Object的实例了
```

这个其实就是和工厂模式一样。实际上，构造函数在不返回值的时候会默认返回新对象实例。而这里return重写了new的返回值，就是person对象，因此name属性是不share的。
**值得注意的是，这时person已经是Object的实例了**

## 闭包模式

这样的

```
var Person = function(name) {
  var person = new Object();

  person.getName = function() {
    return name;
  }

  person.setName = function(newName) {
    this.name = newName;
  }

  return person;
}

var lilei = Person('Lilei');  // 可以不使用new，使用new效果一样
var han = Person('HanMeimei');

lilei.setName('Lilei Gates');

alert(lilei.getName());  // Lilei
alert(han.getName());  // HanMeimei
alert(lilei.constructor);  // 这时他也是Object的实例了
```

这种模式可以提供非常安全的访问方式
**这里和寄生构造一样，对象已经是Object的实例了，跟Person没啥关系**