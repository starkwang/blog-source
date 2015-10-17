title: Constructor同学你到底是谁？
date: 2015-05-02 11:21:32
categories:
- javascript
------
众所周知，Javascript里面有一系列非常“码农不友好”的东西：
constructor、prototype、原型链、匿名函数、闭包……
更可怕的是，这些东西竟然有时候还会混到一起！
今天先讲讲constructor这东西到底是什么

<!-- more -->
----------
## 入门 ##

我们经常会看到类似这样的构造器函数：
```
function a(){
  this.value = 1;
}

var b = new a();

b.constructor === a;   //true
b.value === 1;    //true
```
a是一个构造器函数，所以b对象的constructor自然是a。


----------


但是我们也可以这样写：

```
b={};
b.value = 1;

b.constructor === Object // true
```
这个时候b对象的构建是以“直接赋值”的方式，所以自然地，b此时的构造器变成了原生的Object。

简单易懂，每个人都很开心。


----------

## Constructor是如何被访问的 ##

首先要提到JS的原型链这个概念，所谓的原型链，就是\_\_proto\_\_、prototype这些东西。

还是以构造器的方式来构造对象：
```
function a(){
  this.value = 1;
}

a.prototype.proto_value = "proto_value";   //给构造器原型添加了一个属性

var b = new a();

console.log( b.proto_value );   //"proto_value"，通过__proto__链访问到此属性
b.__proto__ === a.prototype   //true
```
我们可以看出：

 - 任何对象都有一个隐藏的\_\_proto\_\_属性，可以直接访问到构造器的prototype。
 
 
 - 构造器的prototype也是一个对象。
 - **如果在对象中没有查找到相应的属性值，那么会沿着\_\_proto\_\_链一直向上查找，直到找到为止。**

为何我要加粗第三句话呢？

因为要引入下面的结论：

**1、大多数对象（除Function的prototype对象外）创建的时候是没有constructor这个属性的，所以当我们访问对象的constructor属性时，其实是沿着\_\_proto\_\_链在向上查找，直到找到constructor这个属性。**

**2、Function的prototype对象中，有constructor这个属性，是指向自己的。**

什么意思呢？

我们还是用构造器a来生成b对象
```
function a(){
  this.value = 1;
}

var b = new a();
```
这个时候你可以访问到 b.constructor

```
console.log(b.constructor);
```
但实际上是这样访问的：

```
console.log(b.__proto__.constructor);   //等价于 a.prototype.constructor

b.__proto__ === a.prototype    //true
```
实际上发生的事情是：
**b的属性中没有constructor，于是乎，沿着b的\_\_proto\_\_链，我们找到了a.prototype，里面找到了constructor这个属性，值为a本身。**


----------
## 一个小实验 ##

所以我们可以做一下这个实验：

```
function a(){
  this.value = 1;
}
```
然后把a.prototype直接赋值成一个对象
```
a.prototype = {
  proto_value : "proto_value";
}
```
我们来看看b会怎么样
```

var b = new a();

console.log( b.proto_value );  //沿着proto链找到了"proto_value"

console.log( b.constructor );  // Object()
```
这里我们用一个对象直接覆盖了a.prototype，这个对象是由Object函数构造出来的，而且a.prototype里没有constructor这个属性了。

所以当我们访问b.constructor时，发现b.\_\_proto\_\_（即a.prototype）里没有constructor，于是再向上查找，我们找到了b.\_\_proto\_\_.\_\_proto\_\_（即Object.prototype），里面有一个constructor属性，指向Object()，于是返回了Object()
