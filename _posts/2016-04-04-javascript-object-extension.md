---
layout: post
title: Javascript对象继承
categories: [javascript]
---

开门见山，以下根据继承方式的不同逐一介绍：
# 基于原型链的方式
这应该是基于js中prototype的特点实现的最简单的继承
```
var SuperType = function() {
  this.property= true;
}

SuperType.prototype.getSuperValue= function() {
  return this.property;
}

var SubType = function() {
  this.subproperty = false;
}

SubType.prototype = new SuperType();  // 继承

SubType.prototype.getSubValue = function() {
  return this.subproperty;
}

var instance = new SubType();
alert(instance.getSuperValue());  // true
```
此时，上面几个对象的引用关系是这样的:
![JS对象继承应用关系（引用自JavaScript高级程序设计）]({{ "/img/posts/2016-04-04-javascript-object-extension/extension.png" | prepend: site.baseurl }})
那如果子类有同名属性，super的getter会返回那个呢：
```
var Sub = function() {
  this.property = false;
}

Sub.prototype = new SuperType();

var instance = new Sub();
alert(instance.getSuperValue());  // true，返回子类的属性
```
可见JS中子类的属性会重写父类的，此外，方法的重写也一样，这是由于JS查找对象的属性是由子对象开始遍历，子类没有才继续去父对象上找。
另外，JS还提供了方法来判断继承关系：
```
alert(instance instanceof SubType);     // true
alert(instance instanceof SuperType);  // true
alert(instance instanceof Object);        // true

alert(SubType.prototype.isPrototypeOf(instance));     //true
alert(SuperType.prototype.isPrototypeOf(instance));  //true
alert(Object.prototype.isPrototypeOf(instance));        //true
```
使用原型链有个很明显的问题，就是子对象都share同一个父对象的属性，如果是引用类型，就可能对一个对象修改，却影响了其他对象的数据。
# 构造函数继承
构造函数的方式继承可以解决share同一个父对象属性的问题，并且可以引用父对象的带参构造函数：
```
var SuperType = function(name) {
  this.name = name;
  this.colors = ['red', 'yellow'];
}

var SubType = function(name) {
  SuperType.call(this, name);

  this.age = 18;
}

var subA = new SubType('Lilei');
var subB = new SubType('HanMeimei');

subA.colors.push('blue');  // 修改引用属性

alert(subA.name);   // Lilei
alert(subA.colors);  // 'red,yellow,blue'

alert(subB.name);   // HanMeimei
alert(subB.colors);  // 'red,yellow'
```
和[对象创建](../../../2016/03/26/javascript-object-create.html)的情况一样，单纯的构造函数方式会使得每个对象都不能share父对象的函数。于是有了组合继承的方式，各取所长。
# 组合继承的方式
组合继承是通过原型链来继承对象的方法，构造函数来继承对象的属性：
```
var SuperType = function(name) {
  this.name = name
}

SuperType.prototype.getName = function() {
  return this.name
}

var SubType = function(name) {
  SuperType.call(this, name);
}

SubType.prototype = new SuperType();

var sub = new SubType('Lilei');
alert(sub.getName());  // Lilei
```
在new子对象时，会在调用构造函数时重新定义该对象自己的属性，从而和父对象的属性分离。但是组合继承要求父对象的构造方式也是通过组合构造的方式定义的，否则子类用组合继承也是白搭。
# 通过工厂模式创建子对象
以下提供的继承方式是在《JS高级程序设计》中看到的，此书把它们也作为继承方式，但我觉得只能算是对之前继承方式的工厂模式化罢了。
### 原型式继承
给我任何一个原型，还你一个他的子对象（的工厂模式。。。）
```
// 就是他了
function subInstance(o) {
  function F(){}
  F.prototype = o;
  return new F();
}

var superInstance = {
  name: 'Lilei'
}

var sub = subInstance(superInstance);
alert(sub.name);  // Lilei
```
但是很显然，子对象是share父对象的所有属性的。用组合方式实现subInstance？可以试试，但是由于是工厂模式，考虑到通用性，这不是个好的选择。
### 寄生式继承
如果工厂不是那么通用，则可以在创建子对象是再加点额外的属性：
```
function subInstanceWithAge(o) {
  function F(){}
  F.prototype = o;
  
  var sub = new F();
  sub.age = 18;
  return sub;
}

var superInstance = {
  name: 'Lilei'
}

var sub = subInstanceWithAge(superInstance);
alert(sub.name);  // Lilei
alert(sub.age);    // 18
```
同样是share父对象的属性。
### 寄生组合式继承
这种继承方式简直伤心病况。
```
function inherit(subType, superType) {
  function F() {}
  F.prototype = superType.prototype;
  var sub_prototype = new F();

  sub_prototype.constructor = subType;
  subType.prototype = sub_prototype;
}

var SuperType = function(name) {
  this.name = name;
}

SuperType.prototype.getName = function() {
  return this.name;
}

var SubType = function(name, age) {
  SuperType.call(this, name);
  
  this.age = age;
}

inherit(SubType, SuperType);

SubType.prototype.getAge = function() {
  return this.age;
}

var sub = new SubType('Lilei', 18);
alert(sub.getName());  // Lilei
alert(sub.getAge());     // 18
alert(SuperType.isPrototypeOf(sub));  // false
```
虽然子对象能继承父对象的属性，但子对象完全不认识父对象了。与其说是继承，不如说是copy，而且会破坏subType本来的继承解构。

最后介绍的三种工厂方式算是对传统的三种继承方式的发散吧。事实上，只要掌握前三种继承，就可以根据实际需求发散出各种满足使用需求的继承工具方法。