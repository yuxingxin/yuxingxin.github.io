---
layout: post
title: JavaScript基础之数组与类数组梳理
tags: Array
categories: JavaScript
date: 2018-03-16
---

JavaScript中数组的API纷繁复杂，其中还有不少坑，这里一起做个总结。

## 构造器

通常我们使用对象字面量的方式创建一个数组，例如`var array = [];`，但是，有的时候对象字面量不能满足我们要求，比如，我想给数组限制长度，用对象字面量就无法创建了，这时候只能使用构造器：

1. new Array(arg1,arg2,...)，参数长度为0或者长度大于等于2时，传入的参数将按照顺序依次成为数组的第0至第N项（参数长度为0时，返回空数组）
2. new Array(len)，当len不是数值时，返回一个只包含len元素一项的数组，当len为数值时，len最大不能超过32位无符号整型，即小于2的32次方，否则将抛出RangeError.

ES6 以后，新增了几个方法：

### Array.of

Array.of 用于将参数依次转化为数组中的一项，然后返回这个新数组，而不管这个参数是数字还是其他，它基本上与 Array 构造器功能一致，唯一的区别就在单个数字参数的处理上。

```javascript
Array.of(8);  // [8]
new Array(8)    // [empty X 8]

Array.of(3,5)  // [3,5]
new Array(3,5)    // [3,5]
Array.of('3')  // ['3']
new Array('3')   // ['3']
```

### Array.from

Array.from 的设计初衷是快速便捷地基于其他对象创建新数组，准确来说就是从一个类似数组的可迭代对象中创建一个新的数组实例。其实就是，只要一个对象有迭代器，Array.from 就能把它变成一个数组（注意：是返回新的数组，不改变原对象）。

从语法上看，它有3个参数：

1. 类似数组的对象。
2. 加工函数，新生成的数组会经过该函数的加工后返回。
3. this作用域，表示加工函数执行的this的值。

这三个参数中，第一个参数是必选的，后面两个参数是可选的。

```javascript
Array.from([1, 2, 3], x => x + x);  // [2, 4, 6]

Array.from(new Set(['foo', 'bar', 'baz', 'foo'])) // [ "foo", "bar", "baz" ]

const map = new Map([[1, 2], [2, 4], [4, 8]]);
Array.from(map);
// [[1, 2], [2, 4], [4, 8]]

const mapper = new Map([['1', 'a'], ['2', 'b']]);
Array.from(mapper.values());
// ['a', 'b'];

Array.from(mapper.keys());
// ['1', '2'];
```

## Array的判断

这里总结至少6种判断数组的方法：

1. 基于instanceof
2. 基于constructor
3. 基于Object.prototype.isPrototypeOf
4. 基于getPrototypeOf
5. 基于Object.prototype.toString
6. 基于ES6 新增的Array.isArray方法

```javascript
var a = [];
a instanceof Array

a.constructor === Array

Array.prototype.isPrototypeOf(a);

Object.getPrototypeOf(a) === Array.prototype

Object.prototype.toString.apply(a) === '[object Array]'
```

## 改变自身的方法

基于 ES6，会改变自身值的方法一共有 9 个，分别为 pop、push、reverse、shift、sort、splice、unshift，以及两个 ES6 新增的方法 copyWithin 和 fill。

### pop

**pop()** 方法从数组中删除最后一个元素，并返回该元素的值。此方法会更改数组的长度。

### push

**push()** 方法将一个或多个元素添加到数组的末尾，并返回该数组的新长度。

### reverse

**reverse()** 方法将数组中元素的位置颠倒，并返回该数组。数组的第一个元素会变成最后一个，数组的最后一个元素变成第一个。该方法会改变原数组。

### shift

**shift()** 方法从数组中删除**第一个**元素，并返回该元素的值。此方法更改数组的长度。

### unshift

**unshift()** 方法将一个或多个元素添加到数组的**开头**，并返回该数组的**新长度**（该方法修改原有数组）。

### sort

**sort()** 方法用[原地算法](https://en.wikipedia.org/wiki/In-place_algorithm)对数组的元素进行排序，并返回数组。默认排序顺序是在将元素转换为字符串，然后比较它们的 UTF-16 代码单元值序列时构建的

如果指明了 `compareFunction` ，那么数组会按照调用该函数的返回值排序。即 a 和 b 是两个将要被比较的元素：

- 如果 `compareFunction(a, b)` 小于 0 ，那么 a 会被排列到 b 之前；
- 如果 `compareFunction(a, b)` 等于 0 ， a 和 b 的相对位置不变。备注： ECMAScript 标准并不保证这一行为，而且也不是所有浏览器都会遵守（例如 Mozilla 在 2003 年之前的版本）；
- 如果 `compareFunction(a, b)` 大于 0 ， b 会被排列到 a 之前。
- `compareFunction(a, b)` 必须总是对相同的输入返回相同的比较结果，否则排序的结果将是不确定的。

### splice

**splice()** 方法通过删除或替换现有元素或者原地添加新的元素来修改数组，并以数组形式返回被修改的内容。此方法会改变原数组。

参数：start指定修改的开始位置，deleteCount表示要移除的数组元素的个数，item1,item2……要添加进数组的元素。

### copyWithin

**copyWithin()** 方法浅复制数组的一部分到同一数组中的另一个位置，并返回该数组，不会改变原数组的长度。

参数target为基底的索引，复制序列到该位置，start为索引，开始复制元素的起始位置，end为索引，开始复制元素的结束位置。

### fill

**fill()** 方法用一个固定值填充一个数组中从起始索引到终止索引内的全部元素，并返回修改后的数组，不包括终止索引。

参数value用来填充数组元素的值，start（可选）起始索引，默认为0，end（可选）终止索引，默认为this.length

## 不改变自身的方法

基于 ES7，不会改变自身的方法也有 9 个，分别为 concat、join、slice、toString、toLocaleString、indexOf、lastIndexOf、未形成标准的 toSource，以及 ES7 新增的方法 includes。

另外这里面需要注意的就是，slice不改变自身，而splice改变自身，其中，slice 的语法是：arr.slice([start[, end]])，而 splice 的语法是：arr.splice(start,deleteCount[, item1[, item2[, …]]])。我们可以看到从第二个参数开始，二者就已经有区别了，splice 第二个参数是删除的个数，而 slice 的第二个参数是 end 的坐标（可选）。

### concat

**concat()** 方法用于合并两个或多个数组。此方法不会更改现有数组，而是返回一个新数组。

### join

**join()** 方法将一个数组（或一个[类数组对象](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Indexed_collections#working_with_array-like_objects)）的所有元素连接成一个字符串并返回这个字符串。如果数组只有一个项目，那么将返回该项目而不使用分隔符。

### slice

**slice()** 方法返回一个新的数组对象，这一对象是一个由 `begin` 和 `end` 决定的原数组的**浅拷贝**（包括 `begin`，不包括`end`）。原始数组不会被改变。

### indexOf

**`indexOf()`**方法返回在数组中可以找到一个给定元素的第一个索引，如果不存在，则返回-1。

### lastIndexOf

**lastIndexOf()** 方法返回指定元素（也即有效的 JavaScript 值或变量）在数组中的最后一个的索引，如果不存在则返回 -1。从数组的后面向前查找，从 `fromIndex` 处开始。

### includes

**includes()** 方法用来判断一个数组是否包含一个指定的值，根据情况，如果包含则返回 `true`，否则返回 `false`。



## 数组遍历的方法

基于 ES6，不会改变自身的遍历方法一共有 14 个，分别为 forEach、every、some、filter、map、reduce、reduceRight，以及 ES6 新增的方法 entries、find、findLast、findIndex、findLastIndex、keys、values。

### forEach

**forEach()** 方法对数组的每个元素执行一次给定的函数，注意不返回值

```javascript
arr.forEach(callback(currentValue [, index [, array]])[, thisArg])
```

### every

**every()** 方法测试一个数组内的所有元素是否都能通过某个指定函数的测试。它返回一个布尔值。

```javascript
arr.every(callback(element[, index[, array]])[, thisArg])
```

### some

**some()** 方法测试数组中是不是至少有 1 个元素通过了被提供的函数测试。它返回的是一个 Boolean 类型的值。

```javascript
arr.some(callback(element[, index[, array]])[, thisArg])
```

### filter

**filter()** 方法创建一个新数组，其包含通过所提供函数实现的测试的所有元素。注意它返回一个新数组

```javascript
var newArray = arr.filter(callback(element[, index[, array]])[, thisArg])
```

### map

**map()** 方法创建一个新数组，这个新数组由原数组中的每个元素都调用一次提供的函数后的返回值组成。注意它返回一个新数组。

```javascript
var new_array = arr.map(function callback(currentValue[, index[, array]]) {
 // Return element for new_array
}[, thisArg])

```

### reduce

**reduce()** 方法对数组中的每个元素按序执行一个由您提供的 **reducer** 函数，每一次运行 **reducer** 会将先前元素的计算结果作为参数传入，最后将其结果汇总为单个返回值。

第一次执行回调函数时，不存在“上一次的计算结果”。如果需要回调函数从数组索引为 0 的元素开始执行，则需要传递初始值。否则，数组索引为 0 的元素将被作为初始值 *initialValue*，迭代器将从第二个元素开始执行（索引为 1 而不是 0）。

```javascript
// Arrow function
reduce((previousValue, currentValue) => { /* ... */ } )
reduce((previousValue, currentValue, currentIndex) => { /* ... */ } )
reduce((previousValue, currentValue, currentIndex, array) => { /* ... */ } )
reduce((previousValue, currentValue, currentIndex, array) => { /* ... */ }, initialValue)

// Callback function
reduce(callbackFn)
reduce(callbackFn, initialValue)

// Inline callback function
reduce(function(previousValue, currentValue) { /* ... */ })
reduce(function(previousValue, currentValue, currentIndex) { /* ... */ })
reduce(function(previousValue, currentValue, currentIndex, array) { /* ... */ })
reduce(function(previousValue, currentValue, currentIndex, array) { /* ... */ }, initialValue)

```

### keys

**keys()** 方法返回一个包含数组中每个索引键的**`Array Iterator`**对象。

### values

**values()** 方法返回一个新的 **Array Iterator** 对象，该对象包含数组每个索引的值。

### entries

**entries()** 方法返回一个新的**Array Iterator**对象，该对象包含数组中每个索引的键/值对。

```javascript
var arr = ["a", "b", "c"];
var iterator = arr.entries();
console.log(iterator.next());

/*{value: Array(2), done: false}
          done:false
          value:(2) [0, "a"]
           __proto__: Object
*/
// iterator.next() 返回一个对象，对于有元素的数组，
// 是 next{ value: Array(2), done: false }；
// next.done 用于指示迭代器是否完成：在每次迭代时进行更新而且都是 false，
// 直到迭代器结束 done 才是 true。
// next.value 是一个 ["key","value"] 的数组，是返回的迭代器中的元素值。


```

### find/findLast

**find()** 方法返回数组中满足提供的测试函数的第一个元素的值。否则返回 [`undefined`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/undefined)。

**findLast()** 方法返回数组中满足提供的测试函数条件的最后一个元素的值。如果没有找到对应元素，则返回 [`undefined`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/undefined)。

```javascript
arr.find(callback[, thisArg])

// Arrow function
findLast((element) => { /* ... */ } )
findLast((element, index) => { /* ... */ } )
findLast((element, index, array) => { /* ... */ } )

// Callback function
findLast(callbackFn)
findLast(callbackFn, thisArg)

// Inline callback function
findLast(function(element) { /* ... */ })
findLast(function(element, index) { /* ... */ })
findLast(function(element, index, array){ /* ... */ })
findLast(function(element, index, array) { /* ... */ }, thisArg)
```

### findIndex/findLastIndex

**findIndex()方法返回数组中满足提供的测试函数的第一个元素的索引**。若没有找到对应元素则返回-1。

**findLastIndex()** 方法返回数组中满足提供的测试函数条件的最后一个元素的索引。若没有找到对应元素，则返回 -1。

```javascript
arr.findIndex(callback[, thisArg])

// Arrow function
findLastIndex((element) => { /* ... */ } )
findLastIndex((element, index) => { /* ... */ } )
findLastIndex((element, index, array) => { /* ... */ } )

// Callback function
findLastIndex(callbackFn)
findLastIndex(callbackFn, thisArg)

// Inline callback function
findLastIndex(function(element) { /* ... */ })
findLastIndex(function(element, index) { /* ... */ })
findLastIndex(function(element, index, array){ /* ... */ })
findLastIndex(function(element, index, array) { /* ... */ }, thisArg)
```

## 数组总结

这里面需要注意一些返回数组的方法，比如：reverse、sort、copyWithin、fill、concat、slice、filter、map，同时也注意前面四个返回原数组，即改变自身，后面四个返回新数组，即不改变自身。

| 数组\分类 | 改变自身的方法                                   | 不改变自身的方法                                             | 遍历方法（不改变自身）                                       |
| --------- | ------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| ES5及以前 | pop、push、reverse、shift、sort、splice、unshift | concat、join、slice、toString、toLocateString、indexOf、lastIndexOf | forEach、every、some、filter、map、reduce、reduceRight       |
| ES6/7/8   | copyWithin、fill                                 | includes、toSource                                           | entries、find、findLast、findIndex、findLastIndex、keys、values |

同时以上这些方法之间还存在很多共性，如下：

- 所有插入元素的方法，比如 push、unshift 一律返回数组新的长度；

- 所有删除元素的方法，比如 pop、shift、splice 一律返回删除的元素，或者返回删除的多个元素组成的数组；

- 部分遍历方法，比如 forEach、every、some、filter、map、find、findIndex，它们都包含 function(value,index,array){} 和 thisArg 这样两个形参。

## 类数组

我们日常开发者中经常会遇到类数组对象，最常见的便是在函数中使用的arguments，它的对象只定义在函数体中，包括了函数的参数和其他属性。

如下代码实例：

```javascript
function foo(name, age, sex) {
    console.log(arguments);
    console.log(typeof arguments);  // 'object'
    console.log(Object.prototype.toString.call(arguments));  // [object Arguments]
}
foo('yuxingxin', '28', 'male');
```

从结果中可以看到，typeof 这个 arguments 返回的是 object，通过 Object.prototype.toString.call 返回的结果是 '[object arguments]'，可以看出来返回的不是 '[object array]'，说明 arguments 和数组还是有区别的。

另外length属性表示函数参数的长度，callee属性代表函数自身，如果在函数内部直接执行调用 callee 的话，那它就会不停地执行当前函数，直到执行到内存溢出。

除此之外，还有其他类数组，比如下面的HTMLCollection和NodeList

### HTMLCollection

HTMLCollection 简单来说是 HTML DOM 对象的一个接口，这个接口包含了获取到的 DOM 元素集合，返回的类型是类数组对象，如果用 typeof 来判断的话，它返回的是 'object'。它是及时更新的，当文档中的 DOM 变化时，它也会随之变化。

### NodeList

NodeList 对象是节点的集合，通常是由 querySlector 返回的。NodeList 不是一个数组，也是一种类数组。虽然 NodeList 不是一个数组，但是可以使用 for...of 来迭代。在一些情况下，NodeList 是一个实时集合，也就是说，如果文档中的节点树发生变化，NodeList 也会随之变化。

### 应用场景

#### 遍历参数

我们在函数内部可以直接获取 arguments 这个类数组的值，那么也可以对于参数进行一些操作

#### 定义链接字符串函数

我们可以通过 arguments 这个例子定义一个函数来连接字符串。这个函数唯一正式声明了的参数是一个字符串，该参数指定一个字符作为衔接点来连接字符串。该函数定义如下：

```javascript
function myConcat(separa) {
  var args = Array.prototype.slice.call(arguments, 1);
  return args.join(separa);
}
myConcat(", ", "red", "orange", "blue");
// "red, orange, blue"
myConcat("; ", "elephant", "lion", "snake");
// "elephant; lion; snake"
myConcat(". ", "one", "two", "three", "four", "five");
// "one. two. three. four. five"
```

#### 传递参数

```javascript
// 使用 apply 将 foo 的参数传递给 bar
function foo() {
    bar.apply(this, arguments);
}
function bar(a, b, c) {
   console.log(a, b, c);
}
foo(1, 2, 3)   //1 2 3
```

上述代码中，通过在 foo 函数内部调用 apply 方法，用 foo 函数的参数传递给 bar 函数，这样就实现了借用参数的妙用。

### 类数组借用数组方法转换为数组

```javascript
function sum(a, b) {
  let args = Array.prototype.slice.call(arguments);
 // let args = [].slice.call(arguments); // 这样写也是一样效果
  console.log(args.reduce((sum, cur) => sum + cur));
}
sum(1, 2);  // 3

function sum(a, b) {
  let args = Array.prototype.concat.apply([], arguments);
  console.log(args.reduce((sum, cur) => sum + cur));
}
sum(1, 2);  // 3

// 借助ES6新增的Array.from和展开运算符
function sum(a, b) {
  let args = Array.from(arguments);
  console.log(args.reduce((sum, cur) => sum + cur));
}
sum(1, 2);    // 3

function sum(a, b) {
  let args = [...arguments];
  console.log(args.reduce((sum, cur) => sum + cur));
}
sum(1, 2);    // 3

function sum(...args) {
  console.log(args.reduce((sum, cur) => sum + cur));
}
sum(1, 2);    // 3
```

## 总结

数组和类数组还是有一些区别的，这里可以通过一个表格来重新梳理一下两者的异同点

| 方法\特点  | 数组     | 类数组 |
| ---------- | -------- | ------ |
| 自带方法   | 多个方法 | 无     |
| length属性 | 有       | 有     |
| callee属性 | 无       | 有     |

