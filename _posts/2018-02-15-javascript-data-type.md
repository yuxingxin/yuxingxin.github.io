---
layout: post
title: JavaScript基础之数据类型及检测和转换
tags: 数据类型
categories: JavaScript
date: 2018-02-15
---

## 数据类型

JavaScript数据类型一共分为8种：

- Number（数值，包含NaN）
- String（字符串）
- Boolean（布尔值）
- Undefined（未定义/未初始化）
- Null（空对象）
- Symbol（独一无二的值，ES6 新增）
- BigInt （大整数，能够表示超过 Number 类型大小限制的整数，ES 2020新增）
- Object（对象。Array/数组 和 function/函数 也属于对象的一种）


其中前7种为基本数据类型（原始值），最后一种为复杂数据类型(引用类型)，其中基础类型的值存储在栈内存，是不可变的，你无法修改值本身，只能代表它的变量对其重新赋值，而复杂数据类型存储在堆内存，是可变的，它们的值可以被修改。

而引用数据类型又分为几种常见的类型：

- Array  数组类型
- RegExp 正则对象
- Date  日期对象
- Math  数学函数
- Function 函数对象

## 数据类型检测

数据类型检测也是平常写代码过程中经常会遇到的，这里总结了四种常见的数据类型检测方法：

### 1. typeof

这是一种比较常见的数据类型检测方法：

```javascript
console.log(typeof 10)     // number
console.log(typeof '10')   // string
console.log(typeof undefined)  // undefined
console.log(typeof true)       // boolean
console.log(typeof Symbol())   // symbol
console.log(typeof null)       // object
console.log(typeof [])         // object
console.log(typeof {})         // object
console.log(typeof console)    // object
console.log(typeof console.log)   //function
```

由上面的结果可知typeof只可以测试出`number`、`string`、`boolean`、`symbol`、`undefined`及`function`，而对于`null`及数组、对象，typeof均检测出为object，不能进一步判断它们的类型。

### 2. instanceof

通过instanceof对象我们可以判断出这个对象是否是之前那个构造函数生成的对象，其底层的一个的大致实现：

```javascript
function isInstanceOf(left, right){
    // 基本类型
    if(typeof left != 'object' || left === 'null') return false;
    var proto = Object.getPrototypeOf(left)
    while(true){  // 循环往下找，直到找到相同的原型对象
        if(proto === null) return false;
        if(proto === right.prototype) return true;
        proto = Object.getPrototypeOf(proto);
    }
}
```

举例如下：

```javascript
let Parent = function() {}
let parent1 = new Parent()
parent1 instanceof Parent  // true

'Child' instanceof String // false
```

虽然instanceof可以检测出复杂的引用数据类型，但是对于基本数据类型却不能判断，所以不管是typeof还是instanceof都不能满足所有场景的需求，而只能结合两者才能实现判断

### 3. constructor

当一个函数 Fun被定义时，JS引擎会为Fun添加 prototype 原型，然后再在 prototype上添加一个 constructor 属性，并让其指向 Fun 的引用。

```javascript
function Fun() {}
var f = new Fun()
f.constructor === Fun   // true

Fun.prototype = {}
var f1 = new Fun()
f1.constructor === Fun   // false
```

可以看出，Fun 利用原型对象上的 constructor 引用了自身，当 Fun 作为构造函数来创建对象时，原型上的 constructor 就被遗传到了新创建的对象上， 从原型链角度讲，构造函数 Fun 就是新对象的类型。这样做的意义是，让新对象在诞生以后，就具有可追溯的数据类型。

但是这里面也有一些问题：

- null 和 undefined 是无效的对象，因此是不会有 constructor 存在的，这两种类型的数据需要通过其他方式来判断。
- 函数的 constructor 是不稳定的，不安全的，因为contructor的指向是可以改变的，这个主要体现在自定义对象上，当开发者重写 prototype 后，原有的 constructor 引用会丢失，constructor 会默认为 Object

### 4. Object.prototype.toString

toString() 是 Object 的原型方法，调用该方法，可以统一返回格式为 “[object Xxx]” 的字符串，其中 Xxx 就是对象的类型。对于 Object 对象，直接调用 toString() 就能返回 [object Object]；而对于其他对象，则需要通过 call 来调用，才能返回正确的类型信息。我们来看一下代码。

```javascript
console.log(Object.prototype.toString.call(true));//[object Boolean]
console.log(Object.prototype.toString.call(1));//[object Number]
console.log(Object.prototype.toString.call('str'));//[object String]
console.log(Object.prototype.toString.call(undefined));//[object Undefined]
console.log(Object.prototype.toString.call(null));//[object Null]
console.log(Object.prototype.toString.call([]));//[object Array]
console.log(Object.prototype.toString.call({}));//[object Object]
console.log(Object.prototype.toString.call(function(){});//[object Function]
console.log(Object.prototype.toString.call(/456/g);//[object RegExp]
console.log(Object.prototype.toString.call(new Date());//[object Date]
console.log(Object.prototype.toString.call(document);//[object HTMLDocument]
console.log(Object.prototype.toString.call(window);//[object Window]
```

从上面实例可以看出使用这个方法可以返回统一的字符串格式"[Object Xxx]" ，其中首字母大写，和typeof刚好相反

到目前为止我们可以实现一个全局通用的数据类型判断方法，代码如下：

```javascript
function getType(obj) {
    let type = typeof obj
    if(type !== 'object'){  // 先对基础数据类型进行判断
        return type
    }
    // return Object.prototype.toString.call(obj).slice(8, -1).toLowerCase();
    return Object.prototype.toString.call(obj).replace(/^\[object (\S+)\]$/,'$1')
}
```

## 数据类型转换

数据类型转换主要分显式类型转换和隐式类型转换

### 显式类型转换

其中显式类型转换的常见方法有：

- Number()
- parseInt()
- parseFloat()
- toString()
- String()
- Boolean()

这里参考了 [ECMA-262 的官方文档](https://262.ecma-international.org/7.0/#sec-abstract-operations)来总结一下这几种常见类型转换

| 原始值    | ToNumber                          |
| --------- | --------------------------------- |
| Undefined | NaN                               |
| Null      | 0                                 |
| true      | 1                                 |
| false     | 0                                 |
| String    | 根据语法和转换规则来转换          |
| Symbol    | Throw a TypeError exception       |
| Object    | 先调用toPrimitive, 再调用toNumber |

`String` 转换为 `Number` 类型的规则：

1. 如果字符串中只包含数字，那么就转换为对应的数字。
2. 如果字符串中只包含十六进制格式，那么就转换为对应的十进制数字。
3. 如果字符串为空，那么转换为0。
4. 如果字符串包含上述之外的字符，那么转换为 NaN。

| 原始值    | ToBoolean                         |
| --------- | --------------------------------- |
| Undefined | false                             |
| Boolean   | true / false                      |
| Number    | 0和NaN转换为false，其他转换为true |
| Symbol    | true                              |
| Object    | true                              |

| 原始值    | ToString                          |
| --------- | --------------------------------- |
| Undefined | 'Undefined'                       |
| Boolean   | 'true' / 'false'                  |
| Number    | 对应的字符串类型                  |
| String    | String                            |
| Symbol    | Throw a TypeError exception       |
| Object    | 先调用ToPrimitive，再调用toNumber |

### 隐式类型转换

凡是通过`逻辑运算符 (&&、||、 !)、运算符 (+、-、*、/)、关系操作符 (>;、 <;、 >= 、<=)、相等运算符 (==) 或者 if/while 条件的操作`，如果遇到两个数据类型不一样的情况，都会出现隐式类型转换。这里你需要重点关注一下，因为比较隐蔽，特别容易让人忽视。

其中我们举例==运算符和+ 号运算符：

#### == 运算符

- 如果其中一个操作值是 null 或者 undefined，那么另一个操作符必须为 null 或者 undefined，才会返回 true，否则都返回 false；
- 如果其中一个是 Symbol 类型，那么返回 false；
- 两个操作值如果为 string 和 number 类型，那么就会将字符串转换为 number；
- 如果一个操作值是 boolean，那么转换成 number；
- 如果一个操作值为 object 且另一方为 string、number 或者 symbol，就会把 object 转为原始类型再进行判断（调用 object 的 valueOf/toString 方法进行转换）。

所以在JavaScript语言中，我们不使用双等号运算符`==`进行等式判断，而是采用三等号运算符`===`进行等式判断。

#### + 运算符

- '+' 号操作符，不仅可以用作数字相加，还可以用作字符串拼接。仅当 '+' 号两边都是数字时，进行的是加法运算；如果两边都是字符串，则直接拼接，无须进行隐式类型转换。
- 如果其中有一个是字符串，另外一个是 undefined、null 或布尔型，则调用 toString() 方法进行字符串拼接；如果是纯对象、数组、正则等，则默认调用对象的转换方法会存在优先级，然后再进行拼接。
- 如果其中有一个是数字，另外一个是 undefined、null、布尔型或数字，则会将其转换成数字进行加法运算，对象的情况还是参考上一条规则。
- 如果其中一个是字符串、一个是数字，则按照字符串规则进行拼接。

#### Object的转换规则

对象转换的规则，会先调用内置的 [ToPrimitive] 函数，其规则逻辑如下：

- 如果部署了 Symbol.toPrimitive 方法，优先调用再返回；
- 调用 valueOf()，如果转换为基础类型，则返回；
- 调用 toString()，如果转换为基础类型，则返回；
- 如果都没有返回基础类型，会报错。