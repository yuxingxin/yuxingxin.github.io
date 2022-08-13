---
layout: post
title: JavaScript基础之new、apply、call、bind实现原理
tags: 关键字
categories: JavaScript
date: 2018-03-09
---

在 javascript 中，new通常被用来创建一个对象，`call` 和 `apply` 都是为了改变某个函数运行时的上下文（context）而存在的，换句话说，就是为了改变函数体内部 `this` 的指向。而bind 是返回对应函数，便于稍后调用；apply 、call 则是立即调用 。

## new关键词原理

new 关键词的主要作用就是执行一个构造函数、返回一个实例对象，在 new 的过程中，根据构造函数的情况，来确定是否可以接受参数的传递，我们先看一段代码：

```javascript
function Person() {
    this.name = 'yuxingxin';
}
let p = new Person();
console.log(p.name);   // yuxingxin
```

这段代码从结果可以看出p是通过一个构造函数生成的实例对象，那么new这个关键词在这个过程中都发生了什么呢 ？我们用伪代码来模拟一下过程：

```javascript
new Person() = {
    var p = {};
    p.__proro__ = Person.prototype;
    var res = Person.call(this);
    return typeof res === 'object'? res : p
}
```

总结下来大致分为这么几步：

1. 创建一个新的空对象{}
2. 设置这个对象原型指向构造函数，即this指向新对象
3. 执行构造函数，为这个新对象添加属性
4. 返回新创建的对象

当上面构造函数中存在return语句时，结果就又不一样了。

```javascript
function Person() {
    this.name = 'yuxingxin';
    return {age: 28};
}

var p = new Person();
console.log(p)  // {age: 28}
console.log(p.name) // undefined
console.log(p.age)  // 28
```

通过这段代码又可以看出，当构造函数最后 return 出来的是一个和 this 无关的对象时，new 命令会直接返回这个新对象，而不是通过 new 执行步骤生成的 this 对象。

但是这里要求构造函数必须是返回一个对象，如果返回的不是对象，那么还是会按照 new 的实现步骤，返回新生成的对象。如下实例：

```javascript
function Person() {
    this.name = 'yuxingxin';
    return "宇行信";
}

var p = new Person();
console.log(p)  // {name: 'yuxingxin'}
console.log(p.name) // yuxingxin
```

可以看出，当构造函数中 return 的不是一个对象时，那么它还是会根据 new 关键词的执行逻辑，生成一个新的对象（绑定了最新 this），最后返回出来。
因此我们总结一下：new 关键词执行之后总是会返回一个对象，要么是实例对象，要么是 return 语句指定的对象。

## apply、call、bind原理

我们先来看下这三个函数的基本语法：

> func.call(thisArg, param1, param2, ...)
> func.apply(thisArg, [param1,param2,...])
> func.bind(thisArg, param1, param2, ...)

其中 func 是要调用的函数，thisArg 一般为 this 所指向的对象，后面的 param1、2 为函数 func 的多个参数，如果 func 不需要参数，则后面的 param1、2 可以不写。
这三个方法共有的、比较明显的作用就是，都可以改变函数 func 的 this 指向。call 和 apply 的区别在于，传参的写法不同：apply 的第 2 个参数为数组； call 则是从第 2 个至第 N 个都是给 func 的传参；而 bind 和这两个（call、apply）又不同，bind 虽然改变了 func 的 this 指向，但不是马上执行，而这两个（call、apply）是在改变了函数的 this 指向之后立马执行。

```javascript
let yuxingxin = {
    name: 'yuxingxin',
    getName: function(msg){
        return msg+this.name;
    }
}

let zhangsan = {
    name: 'zhangsan'
}

console.log(yuxingxin.getName('I am '))   // I am yuxingxin
console.log(yuxingxin.getName.call(zhangsan, 'I am '))  // I am zhangsan
console.log(yuxingxin.getName.apply(zhangsan, ['I am ']))  // I am zhangsan
 
let name = yuxingxin.getName.bind(zhangsan, 'I am ');
console.log(name())  // I am zhangsan
```

从上面的代码执行的结果中可以发现，使用这三种方式都可以达成我们想要的目标，即通过改变 this 的指向，让 一个对象可以直接使用 另一个对象中的 getName 方法。从结果中可以看到，最后三个方法输出的都是预期的结果。

## 方法的应用场景

### 判断数据类型

用Object.prototype.toString来判断类型是最合适的，借用它我们可以几乎判断所有的类型。

```javascript
function getType(obj) {
    let type = typeof obj
    if(type !== 'object'){
        return type
    }
    return Object.prototype.toString.call(obj).replace(/^\[object (\S+)\]$/,'$1')
}
```

结合上面这段代码，以及在前面讲的 call 的方法的 “借用” 思路，那么判断数据类型就是借用了 Object 的原型链上的 toString 方法，最后返回用来判断传入的 obj 的字符串，来确定最后的数据类型，这里就不再多做讲解了。

### 类数组借用方法

```javascript
var programs = {
    0: 'java',
    1: 'javascript',
    length: 2
}
Array.prototype.push.call(programs, 'C++', 'C#');
console.log(typeof programs)   // 'object
console.log(programs)  // {0: 'java', 1: 'javascript', 2: 'C++', 3: 'C#', lenght: 4}
```

从上面的代码可以看出，programs是一个对象，模拟数组的一个类数组。从数据类型上看，它是一个对象。用 typeof 来判断输出的是 'object'，它自身是不会有数组的 push 方法的，这里我们就用 call 的方法来借用 Array 原型链上的 push 方法，可以实现一个类数组的 push 方法，给 programs 添加新的元素。

### 获取数组最大/最小值

```javascript
let array = [12,6,9,3];
const max = Math.max.apply(Math, array);
const min = Math.min.apply(Math, array);

console.log(max)  // 12
console.log(min)  // 3
```

### 继承

在继承实现方式中，有一种组合继承，代码如下：

```javascript
function Parent(name) {
    this.name = name;
    this.content = [1,2,3];
    
    this.getContent1 = function() {
        return this.content
    }
}

Parent.prototype.getContent = function() {
    console.log(this.content);
}

function Child(name) {
    Parent.call(this, name)  // 创建子类实例时执行一次
}

Child.prototype = new Parent();  // 指定子类原型会执行一次
Child.prototype.constructor = Child; // 校正构造函数

var child1 = new Child("这是子类1");
child1.content.push(4);
console.log(child1.getContent())  // Array [1,2,3,4]
console.log(child1.getContent1())  // Array [1,2,3,4]

var child2 = new Child("这是子类2");
child2.content.push(5);  
console.log(child2.getContent1())  // Array [1,2,3,5]

```

## 手写实现方法

### new手写实现

回顾一下上面new的执行过程，大致做了这几件事情：

1. 让实例可以访问到私有属性；

1. 让实例可以访问构造函数原型（constructor.prototype）所在原型链上的属性；

1. 构造函数返回的最后结果是引用数据类型。

代码实现如下：

```javascript
function _new() {
    var obj = new Object();
    var constructor = [].shift.call(arguments); // 拿到arguments中的第一个参数，即构造函数constructor;
    if(constructor.prototype !== null) {
        obj.__proto__ = constructor.prototype;
    }
    var ret = constructor.apply(obj, arguments);
    let isObject = typeof res === 'object' && res !== null;
    
    let isFunction = typeof res === 'function';
    // 如果传入的构造函数已经指定返回的对象，那么就返回该对象，否则返回内部构造的obj。
    return isObject || isFunction? res: obj;
}

/** 
1.new关键字会首先创建一个空对象
2.将这个空对象的原型对象指向构造函数的原型属性，从而继承原型上的方法
3.将this指向这个空对象，执行构造函数中的代码，以获取私有属性
4.如果构造函数返回了一个对象res，就将该返回值res返回，如果返回值不是对象，就将创建的对象返回
*/
function _new(ctor, ...args) {
    if(typeof ctor !== 'function') {
        throw 'ctor must be a function';
    }

    let obj = new Object();
    obj.__proto__ = Object.create(ctor.prototype);
    let res = ctor.apply(obj, [...args]);
    let isObject = typeof res === 'object' && res !== null;
    
    let isFunction = typeof res === 'function';
    // 如果传入的构造函数已经指定返回的对象，那么就返回该对象，否则返回内部构造的obj。
    return isObject || isFunction? res: obj;
}
```

总结下：

1. new产生一个新对象;

1. 拿到传入的参数中的第一个参数，即构造函数constructor;

1. 执行构造函数，并将this指向创建的空对象obj;

1. 将传入构造函数的参数，在obj上下文中执行一遍;

1. 如果构造函数返回一个对象，则直接返回这个对象；

### apply和call方法手写实现

由于apply和call方法实现的基本原理差不多，都是借助eval来实现，只是参数不同，这里放一起来看。

```javascript
Function.prototype.apply = function(context, ...args) {
    var context = context || window;
    context.fn = this;
    var result = eval('context.fn(...agrs)');
    delete context.fn;
    return result;
}
Function.prototype.call = function(context, ...args) {
    var context = context || window;
    context.fn = this;
    var result = eval('context.fn(...agrs)');
    delete context.fn;
    return result;
}
```

从上面的代码可以看出，实现 call 和 apply 的关键就在 eval 这行代码。其中显示了用 context 这个临时变量来指定上下文，然后还是通过执行 eval 来执行 context.fn 这个函数，最后返回 result。
要注意这两个方法和 bind 的区别就在于，这两个方法是直接返回执行结果，而 bind 方法是返回一个函数，因此这里直接用 eval 执行得到结果。

### bind手写实现

bind实现和上面最大的变化就是返回结果这里不同，他不需要执行，而是通过返回一个函数的方式将结果返回，之后再执行这个结果，得到想要的效果。

```javascript
Function.prototype.bind = function(context, ...args) {
    if(typeof this !== 'function') {
        throw new Error('this must be a function')
    }
    var self = this;
    
    var fbound = function() { //绑定函数被当做普通函数调用（context）或者被当做构造函数使用（this）
        self.apply(this instanceof self ? this: context, args.concat(Array.prototype.slice.call(arguments));
    }
    // 修正new调用带来的this指向绑定问题
    if(this.prototype) {
        fbound.prototype = Object.create(this.prototype)
    }
    return fbound;
}
```

从上面的代码中可以看到，实现 bind 的核心在于返回的时候需要返回一个函数，故这里的 fbound 需要返回，但是在返回的过程中原型链对象上的属性不能丢失。因此这里需要用Object.create 方法，将 this.prototype 上面的属性挂到 fbound 的原型上面，最后再返回 fbound。这样调用 bind 方法接收到函数的对象，再通过执行接收的函数，即可得到想要的结果。

## 总结

最后我们一起梳理一下这些方法的异同点，表格如下：

| 方法/特点 | call             | apply            | bind             |
| --------- | ---------------- | ---------------- | ---------------- |
| 方法参数  | 多个             | 单个数组         | 多个             |
| 方法功能  | 函数调用改变this | 函数调用改变this | 函数调用改变this |
| 返回结果  | 直接执行         | 直接执行         | 返回待执行函数   |
| 底层实现  | 通过eval         | 通过eval         | 间接调用apply    |
