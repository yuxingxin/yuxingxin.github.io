---
layout: post
title: JavaScript基础之闭包原理
tags: closure
categories: JavaScript
date: 2018-03-13
---

闭包是JavaScript中重要的概念，并且与作用域紧密相关，这里做个总结。在闭包之前，我们先对作用域做下基本介绍。

## 作用域

JavaScript 的作用域通俗来讲，就是指变量能够被访问到的范围，在 JavaScript 中作用域也分为好几种，ES5 之前只有全局作用域和函数作用域两种。ES6 出现之后，又新增了块级作用域。接下来我们一一介绍。

### 全局作用域

在编程语言中，变量通常被分为全局变量和局部变量两种，那么变量定义在最外部，代码前面的一般情况下都是全局变量，在JavaScript中，全局变量是挂在window对象下面，所以在网页中的任何位置都可以使用并且访问它。

```javascript
var globalVar = 'global var';
function getGlobalVar() {
    console.log(globalVar);  // global var
    var localVar = 'local var'; 
    console.log(localVar)   // local var
}

getGlobalVar()

console.log(localVar)  // 报错
console.log(globalVar)  // global var
console.log(window.globalVar)  // global var
```

从上面实例中可以看出，globalVar在什么地方都可以访问的到，所以它就是全局变量。而变量localVar就不具备这样的能力。另外全局变量也有全局的作用域，在控制台可以通过window对象访问它。

当然全局作用域有相应的缺点，我们定义很多全局变量的时候，会容易引起变量命名的冲突，所以在定义变量的时候应该注意作用域的问题。

### 函数作用域

在 JavaScript 中，函数中定义的变量叫作函数变量，这个时候只能在函数内部才能访问到它，所以它的作用域也就是函数的内部，称为函数作用域。

```javascript
function getLocalVar() {
    var localVar = 'local var'; 
    console.log(localVar)   // local var
}
getLocalVar()
console.log(localVar)  // 报错
```

上面实例中，localVar变量是在函数内部中定义的，所以它是一个局部变量，它的作用域就是函数内部，也称函数作用域。除了函数内部以外，其他地方是无法访问到它的，同时，当这个函数被执行完之后，这个局部变量也相应会被销毁。所以你会看到在 getLocalVar 函数外面的 localVar 是访问不到的。

### 块级作用域

ES6 中新增了块级作用域，最直接的表现就是新增的 let 关键词，使用 let 关键词定义的变量只能在块级作用域中被访问，有“暂时性死区”的特点，也就是说这个变量在定义之前是不能被使用的。
所谓的块其实就是在 JS 编码过程中 if 语句及 for 语句后面 {...} 这里面所包括的，就是块级作用域。

```javascript
console.log(a)  //报错
if(true){
    let a = 1;
    console.log(1)  // 1
}

console.log(a)  // 报错
```

从这段代码可以看出，变量 a 是在 if 语句{...} 中由 let 关键词进行定义的变量，所以它的作用域是 if 语句括号中的那部分，而在外面进行访问 a 变量是会报错的，因为这里不是它的作用域。所以在 if 代码块的前后输出 a 这个变量的结果，控制台会报错显示 a 并没有被定义。

## 闭包

摘录一段红宝书和MDN中的定义：

> 红宝书：闭包是指有权访问另外一个函数作用域中的变量的函数。
> MDN：一个函数和对其周围状态的引用捆绑在一起（或者说函数被引用包围），这样的组合就是闭包（closure）。也就是说，闭包让你可以在一个内层函数中访问到其外层函数的作用域。

### 概念

通俗的讲，闭包其实就是一个可以访问其他函数内部变量的函数。即一个定义在函数内部的函数，或者直接说闭包是个内嵌函数也可以。

因为通常情况下，函数内部变量是无法在外部访问的（即全局变量和局部变量的区别），因此使用闭包的作用，就具备实现了能在外部访问某个函数内部变量的功能，让这些内部变量的值始终可以保存在内存中。

```javascript
function fun() {
    var a = 1;
    return function() {
        console.log(a);
    };
}

fun();
var result = fun();
result();  // 1
```

结合闭包的概念，以及前面对作用域的铺垫，那么可以很清楚地发现，a 变量作为一个 fun 函数的内部变量，正常情况下作为函数内的局部变量，是无法被外部访问到的。但是通过闭包，我们最后还是可以拿到 a 变量的值。

### 闭包产生的原因-作用域链

我们在前面介绍了作用域的概念，那么你还需要明白作用域链的基本概念。其实很简单，当访问一个变量时，代码解释器会首先在当前的作用域查找，如果没找到，就去父级作用域去查找，直到找到该变量或者不存在父级作用域中，这样的链路就是作用域链。
需要注意的是，每一个子函数都会拷贝上级的作用域，形成一个作用域的链条。

```javascript
var a = 1;
function fun1() {
    var a = 2;
    function fun2() {
        var a = 3;
        console.log(a);
    }
}
```

从中可以看出，fun1 函数的作用域指向全局作用域（window）和它自己本身；fun2 函数的作用域指向全局作用域 （window）、fun1 和它本身；而作用域是从最底层向上找，直到找到全局作用域 window 为止，如果全局还没有的话就会报错。这样一来，当前函数一般都会存在上层函数的作用域的引用，那么他们就形成了一条作用域链。

由此可见，闭包产生的本质就是：**当前环境中存在指向父级作用域的引用**。

```javascript
function fun1() {
    var a = 2;
    function fun2() {
        console.log(a);
    }
    return fun2;
}

var result = fun1();
result()   // 2
```

从上面实例可以看出，这里 result 会拿到父级作用域中的变量，输出 2。因为在当前环境中，含有对 fun2 函数的引用，fun2 函数恰恰引用了 window、fun1 和 fun2 的作用域。因此 fun2 函数是可以访问到 fun1 函数的作用域的变量。

其实上面这里不返回函数也可以产生闭包，我们只需要让父作用域的引用存在即可。

```javascript
var fun3;
function fun1() {
    var a = 2;
    fun3 = function() {
        console.log(a);
    }
}
fun1(); 
fun3();
```

可以看出，其中实现的结果和前一段代码的效果其实是一样的，就是在给 fun3 函数赋值后，fun3 函数就拥有了 window、fun1 和 fun3 本身这几个作用域的访问权限；然后还是从下往上查找，直到找到 fun1 的作用域中存在 a 这个变量；因此输出的结果还是 2，最后产生了闭包，形式变了，本质没有改变。
因此最后返回的不管是不是函数，也都不能说明没有产生闭包。

### 闭包的表现形式

1. 返回一个函数，上面讲原因的时候已经说过，这里就不赘述了。

2. 在定时器、事件监听、Ajax 请求、Web Workers 或者任何异步中，只要使用了回调函数，实际上就是在使用闭包。

   ```javascript
   // 定时器
   setTimeout(function handler(){
     console.log('1');
   }，1000);
   // 事件监听
   $('#app').click(function(){
     console.log('Event Listener');
   });
   ```

1. 作为函数参数传递的形式

   ```javascript
   var a = 1;
   function foo(){
     var a = 2;
     function baz(){
       console.log(a);
     }
     bar(baz);
   }
   function bar(fn){
     // 这就是闭包
     fn();
   }
   foo();  // 输出2，而不是1
   ```

1.  IIFE（立即执行函数），创建了闭包，保存了全局作用域（window）和当前函数的作用域，因此可以输出全局的变量

   ```javascript
   var a = 2;
   (function IIFE(){
     console.log(a);  // 输出2
   })();
   ```

   IIFE 这个函数会稍微有些特殊，算是一种自执行匿名函数，这个匿名函数拥有独立的作用域。这不仅可以避免了外界访问此 IIFE 中的变量，而且又不会污染全局作用域，我们经常能在高级的 JavaScript 编程中看见此类函数。

### 闭包产生的问题

一个常见的问题就是循环输出问题，代码如下：

```javascript
for(var i = 1; i <= 5; i ++){
  setTimeout(function() {
    console.log(i)
  }, 0)
}
```

上面执行结果是5个6，先来看下输出5个6的原因是什么？

1. setTimeout 为宏任务，由于 JS 中单线程 eventLoop 机制，在主线程同步任务执行完后才去执行宏任务，因此循环结束后 setTimeout 中的回调才依次执行。
2. 因为 setTimeout 函数也是一种闭包，往上找它的父级作用域链就是 window，变量 i 为 window 上的全局变量，开始执行 setTimeout 之前变量 i 已经就是 6 了，因此最后输出的连续就都是 6。

但是如果想让输出1,2,3,4,5的话，如何修改呢 ？

#### 利用IIFE

可以利用 IIFE（立即执行函数），当每次 for 循环时，把此时的变量 i 传递到定时器中，然后执行，改造之后的代码如下。

```javascript
for(var i = 1;i <= 5;i++){
  (function(j){
    setTimeout(function timer(){
      console.log(j)
    }, 0)
  })(i)
}
```

#### 使用ES6中的let

ES6 中新增的 let 定义变量的方式，使得 ES6 之后 JS 发生革命性的变化，让 JS 有了块级作用域，代码的作用域以块级为单位进行执行。通过改造后的代码，可以实现上面想要的结果。

```javascript
for(let i = 1; i <= 5; i++){
  setTimeout(function() {
    console.log(i);
  },0)
}
```

从上面的代码可以看出，通过 let 定义变量的方式，重新定义 i 变量，则可以用最少的改动成本，解决该问题。

#### 定时器传入第三个参数

setTimeout 作为经常使用的定时器，它是存在第三个参数的，日常工作中我们经常使用的一般是前两个，一个是回调函数，另外一个是时间，而第三个参数用得比较少。那么结合第三个参数，调整完之后的代码如下。

```javascript
for(var i=1;i<=5;i++){
  setTimeout(function(j) {
    console.log(j)
  }, 0, i)
}
```

从中可以看到，第三个参数的传递，可以改变 setTimeout 的执行逻辑，从而实现我们想要的结果，这也是一种解决循环输出问题的途径。