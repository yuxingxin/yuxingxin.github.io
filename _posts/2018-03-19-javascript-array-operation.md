---
layout: post
title: JavaScript基础之数组常用操作实现
tags: Array
categories: JavaScript
date: 2018-03-19
---

## 数组的去重

给定一个数组，如果对其中的元素去重？

### 1. 利用Set

我们知道ES6引入了Set，它的一个特性就是不能有重复的元素存在，我们可以利用这一特性来实现。另外Set是一个可迭代的对象，我们可以利用Array.from或者展开运算符将其转换为一个新的数组

```javascript
const result = Array.from(new Set(arr))

const result = [...new Set(arr)]
```

### 2. 利用indexOf、includes、filter

```javascript
// indexOf
const unique1 = arr => {
    const result = [];
    for(let i = 0; i< arr.length; i++) {
        if(result.indexOf(arr[i]) === -1) result.push(arr[i]);
    }
    return result;
}

// includes
const unique2 = arr => {
    const result = [];
    for(let i = 0; i< arr.length; i++) {
        if(!result.includes(arr[i])) result.push(arr[i]);
    }
    return result;
}

// filter
const unique3 = arr => {
    return arr.filter((item, index) => {
        return arr.indexOf(item) === index;
    });
}
```

## 数组扁平化

数组扁平化就是将一个嵌套多层的数组 array（嵌套可以是任何层数）转换为只有一层的数组。

### 1. 使用flat()

```javascript
const result = arr.flat(Infinity)
```

### 2. 使用递归

```javascript
function flatten(arr) {
    let result = [];
    for(let i = 0;i < arr.lenght; i++){
        if(Array.isArray(arr[i])) {
            result = result.concat(flatten(arr[i]));
        }else {
            result.push(arr[i]);
        }
    }
    return result;
}
```

### 3. 使用reduce函数迭代

```javascript
function flatten(arr) {
    return arr.reduce((prev, curr) => {
        return prev.concat(Array.isArray(curr)? flatten(curr): curr);
    }, []);
}
```

### 4. 使用扩展运算符

```javascript
function flatten(arr) {
    while(arr.some(item => Array.isArray(item))) {
        arr = [].concat(...arr);
    }
    return arr;
}

```

### 5. 使用toString和split

```javascript
function flatten(arr) {
    return arr.toString().split(',');
}
```

### 6. 使用正则和JSON

```javascript
function flatten(arr) {
    let str = JSON.stringfy(arr);
    str = str.replace(/(\[|\])/g,'');
    str = '[' + str + ']';
    return JSON.parse(str);
}
```

## 类数组转化为数组

类数组是具有**length**属性，但不具有数组原型上的方法。常见的类数组有**arguments**、DOM操作方法返回的结果。

### 1. Array.from

```javascript
Array.from(document.querySelectorAll('div'))
```

### 2. Array.prototype.slice.call()

```javascript
Array.prototype.slice.call(document.querySelectorAll('div'))

[].slice.call(document.querySelectorAll('div'))
```

### 3. 扩展运算符

```javascript
[...document.querySelectorAll('div')]
```

### 4. 利用concat

```javascript
Array.prototype.concat.apply([], document.querySelectorAll('div'));
```

## 常见数组原型方法实现

### forEach

```javascript
Array.prototype.forEach2 = function(callback, thisArg) {
    if(this == null) {
        throw new TypeError('this is null or not defined')
    }
    if(typeof callback !== 'function') {
        throw new TypeError(callback+' is not a function');
    }
    
    const O = Object(this)  // this是当前数组
    const len = O.length >>> 0  // 无符号右移
    let k = 0;
    while(k < len) {
        if(k in O) {
            callback.call(thisArg, O[k], k, O);
        }
        k++;
    }
}
```

参考[forEach#polyfill](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/forEach#polyfill)

O.length >>> 0 是什么操作？就是无符号右移 0 位，它就是为了保证转换后的值为正整数。其实底层做了 2 层转换，第一是非 number 转成 number 类型，第二是将 number 转成 Uint32 类型。

### Map

该方法创建一个新数组，这个新数组由原数组中的每个元素都调用一次提供的函数后的返回值组成。基于 forEach 的实现能够很容易写出 map 的实现：

```javascript
Array.prototype.map2 = function(callback, thisArg) {
    if(this == null) {
        throw new TypeError('this is null or not defined');
    }
    if(typeof callback !== 'function')  {
        throw new TypeError(callback + ' is not a function');
    }
    const O = Object(this);
    const len = O.length >>> 0;
    let k = 0, res = [];
    while(k < len) {
        if(k in O) {
            res[k] = callback.call(thisArg, O[k], k, O);
        }
        k++;
    }
    
    return res;
}
```

### filter

该方法创建一个新数组，其包含通过所提供函数实现的测试的所有元素。同样，基于 forEach 的实现能够很容易写出 filter 的实现：

```javascript
Array.prototype.filter2 = function(callback,thisArg) {
    if(this == null) {
        throw new TypeError('this is null or not defined');
    }
    if(typeof callback !== 'function')  {
        throw new TypeError(callback + ' is not a function');
    }
    const O = Object(this);
    const len = O.length >>> 0;
    let k = 0, res = [];
    while(k < len) {
        if(k in O) {
            if(callback.call(thisArg, O[k], k, O)) {
                res.push(O[k]);
            }
        }
        k++;
    }
    return res;
}
```

### some

该方法测试数组中是不是至少有 1 个元素通过了被提供的函数测试。它返回的是一个 Boolean 类型的值。同样，基于 forEach 的实现能够很容易写出 some 的实现：

```javascript
Array.prototype.some2 = function(callback,thisArg) {
    if (this == null) {
        throw new TypeError('this is null or not defined');
    }
    if (typeof callback !== "function") {
        throw new TypeError(callback + ' is not a function');
    }
    const O = Object(this);
    const len = O.length >>> 0;
    let k = 0;
    while(k < len) {
        if(k in O) {
            if(callback.call(thisArg, O[k], k, O)) {
                return true
            }
        }
        k++;
    }
    return false;
}
```

### reduce

该方法对数组中的每个元素按序执行一个由您提供的 **reducer** 函数，每一次运行 **reducer** 会将先前元素的计算结果作为参数传入，最后将其结果汇总为单个返回值。

第一次执行回调函数时，不存在“上一次的计算结果”。如果需要回调函数从数组索引为 0 的元素开始执行，则需要传递初始值。否则，数组索引为 0 的元素将被作为初始值 *initialValue*，迭代器将从第二个元素开始执行（索引为 1 而不是 0）。

```javascript
Array.prototype.reduce2 = function(callback, initialValue) {
    if (this == null) {
        throw new TypeError('this is null or not defined')
    }
    if (typeof callback !== "function") {
        throw new TypeError(callback + ' is not a function')
    }
	const O = Object(this);
    const len = O.length >>> 0;
    let k = 0, acc;
    if(arguments.length > 1) {
        acc = initialValue;
    } else {
        // 没传入初始值的时候，取数组中第一个非empty的值为初始值
        while( k < len && !(k in O)) {
            k++;
        }
        // 如果超出数组界限还没有找到累加器的初始值，则TypeError
        if(k > len) {
            throw new TypeError('Reduce of empty array with no initial value');
        }
        acc = O[k++];
    }
    while(k < len) {
        if(k in O) {
            acc = callback.call(acc, O[k], k, O);
        }
        k++;
    }
    return acc;
}

```

