---
layout: post
title: JavaScript基础之原型与原型链的理解
tags: prototype
categories: JavaScript
date: 2018-03-02
---

理解原型与原型链对于我们实现继承、new关键字原理、甚至代码优化等非常重要，接下来我们对这块做一个总结。

## 概念

我们都知道，访问一个对象实例上的属性和方法时，首先在实例本身查找。 如果没有找到，会到实例对象的隐式属性`__proto__`上查找，就是该实例的构造函数的原型对象。如果还没有，会继续访问原型对象的原型,直到`Object.prototype.__proto__`(null)为止。这也是所有对象都有toString 方法的原因，因为toString方法是Object.prototype上面的方法。所有对象沿着原型链最终都会到Object.prototype.

这其中沿着`__proto__`查找属性的这一链条，就是我们说的原型链。

1. js分为函数对象和普通对象，每个对象都有`__proto__`属性，但是只有函数对象才有prototype属性，并指向函数的原型对象。Object、Function都是js内置的函数对象，类似的还有Array、RegExp、Date、Boolean、Number、String
2. 属性`__proto__`是一个对象，来描述对象的原型，它有两个属性，分别是constructor和`__proto__`
3. 原型对象prototype有一个默认的constructor属性，用于记录实例是由哪个构造函数创建的。

举例代码如下：

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h540oksdc4j20u30f7wg8.jpg)

```javascript
function Person() {
    this.name = 'yuxingxin';
    this.age = 28;
}

let person = new Person()；
```

这里注意原型以及原型链的时候的两条准则：

> 准则1： 原型对象（即Person.prototype）的constructor指向构造函数本身
>
> `Person.prototype.constructor == Person`
>
> 准则2：实例（即person）的`__proto__`和原型对象指向同一个地方，所有函数的原型对象的`__proto__`会指向Object.prototype
>
> `person.__proto__ == Person.prototype` 

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h53za15qt9j20dk0ge756.jpg)

我们来分析一下上面这张图：

Function是最顶层的构造器，Object是最顶层的对象；从原型链讲Function继承了Object，从构造器讲Function构造了Object

```javascript
// 从上方function Foo()开始分析
function Foo() {}

let f1 = new Foo();
let f2 = new Foo();

f1.__proto__ = Foo.prototype;  // 准则2
f2.__proto__ = Foo.prototype;  // 准则2
Foo.prototype.__proto__ = Object.prototype; // 准则2  (Foo.prototype本质也是普通对象)
Object.prototype.__proto__ = null; // 原型链到此停止

Foo.prototype.constructor = Foo;  // 准则1
Foo.__proto__ = Function.prototype;  // 准则2
Function.prototype.__proto__ = Object.prototype // 准则2 (Function.prototype本质也是普通对象)
Object.prototype.__proto__ = null; // 原型链到此停止


// 从中间function Object()开始分析
function Object() {}

let o1 = new Object();
let o2 = new Object();

o1.__proto__ = Object.prototype;  //准则2
o2.__proto__ = Object.prototype;  //准则2
Object.prototype.__proto__ = null;  // 原型链到此停止
Object.prototype.constructor = Object;   // 准则1
Object.__proto__ = Function.prototype;  // 准则2 (Object本质也是函数)

Function.prototype.__proto__ = Object.prototype; // 准则2 (Function.prototype本质也是普通对象)
Object.prototype.__proto__ = null;  // 原型链到此为止


// 从下方function Function()开始分析
function Function() {}
Function.__proto__ = Function.prototype // 准则2
Function.prototype.constructor = Function  // 准则1
```

由此可以得出结论： 除了Object的原型对象（Object.prototype）的__proto__指向null，其他内置函数对象的原型对象（例如：Array.prototype）和自定义构造函数的 `__proto__`都指向Object.prototype, 因为原型对象本身是普通对象。

```javascript
Object.prototype.__proto__ = null;
Array.prototype.__proto__ = Object.prototype;
Foo.prototype.__proto__  = Object.prototype;
```

普通函数可以通过new Function()创建，所以普通函数的`__proto__`指向`Function.prototype`。那么内置的这些函数同样属于函数，也是通过new Function()创建出来的函数对象，同理，他们的`__proto__`也指向`Function.prototype`

```javascript
console.log(Object.__proto__ === Function.prototype)  // true
console.log(Array.__proto__ === Function.prototype)   // true
console.log(Number.__proto__ === Function.prototype)  // true
console.log(String.__proto__ === Function.prototype)  // true
console.log(Boolean.__proto__ === Function.prototype)  // true
console.log(Date.__proto__ === Function.prototype)  // true
```

原型对象的作用，是用来存放实例中共有的那部份属性、方法，可以大大减少内存消耗。实例对象重写原型上继承的属相、方法，相当于“属性覆盖、属性屏蔽”，这一操作不会改变原型上的属性、方法，自然也不会改变由统一构造函数创建的其他实例，只有修改原型对象上的属性、方法，才能改变其他实例通过原型链获得的属性、方法。

