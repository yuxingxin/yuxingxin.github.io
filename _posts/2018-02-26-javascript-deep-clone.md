---
layout: post
title: JavaScript基础之对象深浅拷贝原理以及实现
tags: 数据类型
categories: JavaScript
date: 2018-02-26
---

我们知道JavaScript数据类型分为基本数据类型和引用数据类型，对于基本数据类型，变量存储的是基本类型的值，而对于引用数据类型， 变量存储的是其在内存中的地址，而通常拷贝一个对象，又分为浅拷贝和深拷贝两种，下面我们分别来讨论这种拷贝的区别。

## 浅拷贝

复制一个对象，如果对象属性是基本数据类型，复制的就是基本类型的值给新对象，但如果是引用数据类型，复制的就是内存中的地址，如果其中一个对象改变了内存中的地址，肯定会影响到另一个对象。

接下来整理一些浅拷贝的方法，结合着我们上面的定义来理解：

### 1. Object.assign

这个方法是ES6中提供的方法，它常用于对象的合并，它的第一个参数为拷贝的目标对象，后面的参数是拷贝的源对象，当然也可以是多个源对象。

> Object.assign(target, ...sources)

举例：

```javascript
let target = {}
let source = {a: {b: 1}};
Object.assign(target, source);

source.a.b = 2

console.log(target)  // {a: {b: 2}}
console.log(source)  // {a: {b: 2}}
```

但是该方法有几点需要注意：

- 它不会拷贝对象的继承属性
- 它不会拷贝对象的不可枚举属性
- 他可以拷贝Symbol类型的属性

来重新看下面这段代码：

```javascript
let target = {}
let source = {a: {b: 1}, sym: Symbol(1)};
Object.defineProperty('source','innumerable', {
    value: "这是一个不可枚举属性",
    enumerable: false
})
Object.assign(target, source);

source.a.b = 2

console.log(target)  // {a: {b: 2}, sym: Symbol(1)}
console.log(source)  // {a: {b: 2}, sym: Symbol(1), innumerable: "这是一个不可枚举属性"}
```

### 2. 扩展运算符

这个也是ES6新增的语法

> let target = {...source}

举例如下：

```javascript
let source = {a: 1, b: {c: 2}}
let target = {...source}

source.a = 2
console.log(source)   // {a: 2, b:{c:2}}
console.log(target)  //  {a: 1, b:{c:2}}

source.b.c = 3
console.log(source)  // {a: 2, b:{c:3}}
console.log(target)  // {a: 1, b:{c:3}}

// 数组拷贝
let array = [1,2,3]
let newArray = [...array] //跟array.slice()方法效果一样
```

通过上面例子可以看出扩展运算符同样具备浅拷贝的特性，如果是基本类型，使用它会更加方便

### 3. concat拷贝数组

该方法局限性在于只能浅拷贝数组，使用场景比较有限，需要值得注意的是它会修改原数组中农的元素属性，因此它会影响拷贝后连接的数组

```javascript
let array = [1,2,3]
let newArray = array.concat();
newArray[1] = 5
console.log(array)
console.log(newArray)
```

### 4. slice拷贝数组

slice方法同上也只能应用于数组的浅拷贝，不过它返回一个新的数组对象，这一对象由该方法前两个参数决定原数组的开始和结束位置，是不会影响和改变原数组的

```javascript
let array = [1,2,3,4,5,{val: 6}]
let newArray = array.slice(2)
newArray[3].val = 7
console.log(array)  // [1,2,3,4,5,{val:7}]
console.log(newArray)  // [3,4,5,{val:7}]
```

从上面的例子可以看出，浅拷贝只能拷贝一层对象，如果存在对象的嵌套，那么浅拷贝就无能为力，因此如果想要拷贝多层对象，就需要用到深拷贝

所以大致实现一个浅拷贝的思路：

- 对于基础类型做一个基本的拷贝
- 对于引用类型，开辟一个新的内存空间，拷贝一层对象属性

跟着这个思路，我们实现一个自己的浅拷贝：

```javascript
/**
基本类型拷贝其值
引用类型拷贝内存地址
不可枚举属性不拷贝
继承属性不拷贝
Symbol也可以拷贝
*/
function shallowClone(source){
    if(typeof source === 'object' && source !== null) {
        const target = Array.isArray(source)? []:{};
        for(let prop in source){
            if(source.hasOwnProperty(prop)) {
                target[prop] = source[prop]
            }
        }
        return target
    }else {
        return source
    }
}
```



## 深拷贝

深拷贝对于复杂引用数据类型，在内存中完全开辟了一个新的内存地址，并将原来的对象完整复制过来，拷贝后的对象和原来的对象相互独立，互不影响，彻底实现了内存的分离，总结如下：

> 将一个对象从内存中完整地拷贝出来一份给目标对象，并从堆内存中开辟一个全新的空间存放新对象，且新对象的修改并不会改变原对象，二者实现真正的分离。

接下来整理一些深拷贝的方法，结合着我们上面的定义来理解：

### 1. JSON.stringfy

它借助JSON对象的方法把一个对象序列化成为JSON字符串，并将对象里面的内容转换成字符串，最后再用JSON.parse()方法将JSON字符串生成一个新的对象。

```javascript
let source = {a: 1, b: [1,2,3]}
let str = JSON.stringfy(source)
let target = JSON.parse(str)
console.log(target)  // {a:1, b: [1,2,3]}
source.a = 2
source.b.push(4)
console.log(source)  // {a:2, b: [1,2,3,4]}
console.log(target)  // {a:1, b: [1,2,3]}
```

通过以上实例可以看出深拷贝的对象不受原来对象的影响，但是该方法也有需要注意的地方：

- 拷贝的对象的值中如果有函数、undefined、symbol 这几种类型，经过 JSON.stringify 序列化之后的字符串中这个键值对会消失；
- 拷贝 Date 引用类型会变成字符串；
- 无法拷贝不可枚举的属性；
- 无法拷贝对象的原型链；
- 拷贝 RegExp 引用类型会变成空对象；
- 对象中含有 NaN、Infinity 以及 -Infinity，JSON 序列化的结果会变成 null；
- 无法拷贝对象的循环应用，即对象成环 (obj[key] = obj)。

### 2. 简易版深拷贝实现

```javascript
function deepClone(source){
    let target = {};
    for(let key in source){
        if(typeof source[key] === 'object'){
            target[key] = deepClone(source[key]);
        }else{
            target[key] = source[key];
        }
    }
    return target;
}
```

虽然上面实现一个深拷贝，但是还有一些问题没有解决：

- 这个深拷贝函数并不能复制不可枚举的属性以及 Symbol 类型；

- 这种方法只是针对普通的引用类型的值做递归复制，而对于 Array、Date、RegExp、Error、Function 这样的引用类型并不能正确地拷贝；

- 对象的属性里面成环，即循环引用没有解决。

针对上面这些问题，我们对此进行改进

### 3. 改进版深拷贝

针对上面提出的几个问题，我们先通过以下几个方法来分别解决

- 针对能够遍历对象的不可枚举属性以及 Symbol 类型，我们可以使用 Reflect.ownKeys 方法；

- 当参数为 Date、RegExp 类型，则直接生成一个新的实例返回；

- 利用 Object 的 getOwnPropertyDescriptors 方法可以获得对象的所有属性，以及对应的特性，顺便结合 Object 的 create 方法创建一个新对象，并继承传入原对象的原型链；

- 利用 WeakMap 类型作为 Hash 表，因为 WeakMap 是弱引用类型，可以有效防止内存泄漏（你可以关注一下 Map 和 weakMap 的关键区别，这里要用 weakMap），作为检测循环引用很有帮助，如果存在循环，则引用直接返回 WeakMap 存储的值。

```javascript
function deepClone(src, hash=new WeakMap()) {
    const isComplexType = (typeof src === 'object' || src === 'function') && src !== null;

    if(src.constructor === Date) {
        return new Date(src)
    }
    if(src.constructor === RegExp) {
        return new RegExp(src)
    }
    // 如果是循环引用，通过WeakMap来解决
    if(hash.has(src)) return hash.get(src)
    // 获取对象所有自有属性描述符
    const allDesc = Object.getOwnPropertyDescriptors(src)
    // 继承原型链
    let cloneObj = Object.create(Object.getPrototypeOf(src), allDesc);
    hash.set(src, cloneObj)
    for(let key in Reflect.ownKeys(src)){
        cloneObj[key] = (isComplexType(src[key]) && typeof src[key] !== 'function' )? deepclone(src[key], hash): src[key]
    }
    return cloneObj;
}
```

当然一个完整的深拷贝还远远不止于此，需要各种边界条件的判断，以及不同数据类型等，而实现完整的前提是需要我们对基础知识有足够的掌握，但上面已经可以满足我们日常的需求，感兴趣的也可以继续完善它。