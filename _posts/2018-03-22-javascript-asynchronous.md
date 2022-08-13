---
layout: post
title: JavaScript基础之异步编程演进
tags: Asynchronous
categories: JavaScript
date: 2018-03-22
---

在说异步之前，先介绍一下同步的概念，即在执行某段代码时，在该段代码没有返回结果之前，其他代码暂时无法执行，但是一旦拿到返回值后，就可以接着执行其他代码了，换句话说，在此段代码执行完未返回结果前，会阻塞之后的代码的执行，这样的情况就成为同步。

再回过头来说异步，就是当某一代码执行异步过程调用发出后，这段代码不会立刻得到返回结果。而是在异步调用发出之后，一般通过回调函数处理这个调用之后拿到结果。异步调用发出后，不会影响阻塞后面的代码执行，这样的情形称为异步。

我们都知道JavaScript是单线程的，如果代码都是同步执行，这样可能会造成阻塞，但如果是以异步的方式执行，程序不需要等待异步代码执行的结果，可以继续执行该异步任务之后的代码逻辑，这样不会造成阻塞。

## 回调函数

早年实现异步的方法就是采用回调函数的方式，比如setTimeout或者setInterval，以及一些事件回调，但是写过异步的人都知道使用回调函数最大的一个问题就是会产生回调地狱。

```javascript
var readData = function(fileName, callback) {
    fs.readFile(fileName, callback);
}
readData("file1", function() {
    readData("file2", function(){
        readData("file3",function(){
            console.log("end")
        })
    })
})
```

从上面的代码逻辑可以看出，要实现的功能是模拟读取文件file的内容，即先读取file1的内容，再读取file2的内容，最后读取file3的内容，为了实现上面的需求，就很容易产生回调地狱。那么针对这个问题，如何解决呢？那么接下来看下Promise如何实现异步编程的。

## Promise

为了更好的解决回调地狱问题，社区提出了Promise的方案。

```javascript
var readData = function(fileName){
    return new Promise((resolve, reject) => {
        fs.readFile(fileName, (err, data) => {
            if(err) reject(err);
            resolve(data);
        });
    });
}

readData("file1").then(function() {
    return readData("file2");
}).then(function() {
    return readData("file3")
}).then(function() {
   console.log('end');
}).catch(function(err){
   console.log(err); 
});

```

从上面的代码来看。可读性有一定的提升，优点是可以将异步的操作以同步的流程表达出来，避免了层层嵌套的回调函数。但如果说上面需求改为，同时读取3个文件，这个时候就需要用到Promise.all方法了。

```javascript
Promise.all([readData("file1"), readData("file2"), readData("file3")]).then(data => {
    console.log(data);
}).catch(function(err) {
    console.log(err);
})
```

其实Promise相当于是一个容器，它里面保存着未来才会结束的事件，通常是异步操作的结果，从语法上面来讲，Promise是一个对象，从 它可以获取异步操作的信息。

一般Promise在执行的 过程中，会分成以下几个状态：

1. 待定（pending）：初始状态，既没有被完成，也没有被拒绝。
2. 已完成（fulfilled）：操作成功完成。
3. 已拒绝（rejected）：操作失败。

待定状态的 Promise 对象执行的话，最后要么会通过一个值完成，要么会通过一个原因被拒绝。当其中一种情况发生时，我们用 Promise 的 then 方法排列起来的相关处理程序就会被调用。因为最后 Promise.prototype.then 和 Promise.prototype.catch 方法返回的是一个 Promise， 所以它们可以继续被链式调用。

另外除了上面提到的Promise.all方法以外，还有几个方法：

Promise.allSettled 的语法及参数跟 Promise.all 类似，其参数接受一个 Promise 的数组，返回一个新的 Promise。唯一的不同在于，执行完之后不会失败，也就是说当 Promise.allSettled 全部处理完成后，我们可以拿到每个 Promise 的状态，而不管其是否处理成功。

 Promise.any（iterable），参数为 iterable 可迭代的对象，例如 Array。
它返回一个 Promise，只要参数 Promise 实例有一个变成 fulfilled 状态，最后 any 返回的实例就会变成 fulfilled 状态；如果所有参数 Promise 实例都变成 rejected 状态，包装实例就会变成 rejected 状态。

 Promise.race（iterable），参数为 iterable 可迭代的对象，例如 Array。
它返回一个 Promise，只要参数的 Promise 之中有一个实例率先改变状态，则 race 方法的返回状态就跟着改变。那个率先改变的 Promise 实例的返回值，就传递给 race 方法的回调函数。

## Generator

Generator 也是一种异步编程解决方案，它最大的特点就是可以交出函数的执行权，Generator 函数可以看出是异步任务的容器，需要暂停的地方，都用 yield 语法来标注。Generator 是一个带星号的函数，一般配合 yield 使用来暂停或者执行函数，它最后返回的是迭代器。

```javascript
function* makeRangeIterator(start = 0, end = Infinity, step = 1) {
    for (let i = start; i < end; i += step) {
        yield i;
    }
}
var a = makeRangeIterator(1,10,2)
a.next() // {value: 1, done: false}
a.next() // {value: 3, done: false}
a.next() // {value: 5, done: false}
a.next() // {value: 7, done: false}
a.next() // {value: 9, done: false}
a.next() // {value: undefined, done: true}
```

从实例可以看出，yield关键词最后返回一个迭代器对象，配合着 Generator，再同时使用 next 方法，可以主动控制 Generator 执行进度。每一次执行next函数都会返回一个对象，其存在两个属性，value和done，其中value就是我们打印出来的值，而done表示该迭代器是否执行完，

从上面这个例子，还很难和异步编程联系到一起，接下来我们先说下thunk函数

### thunk函数

它的基本思想都是接收一定的参数，然后生产出定制化的函数，最后使用定制化的函数去完成想要实现的功能。

```javascript
const readDataThunk = (fileName) => {
    return (callback) => {
        fs.readFile(fileName, callback);
    }
}

const gen = function* () {
    const file1 = yield readDataThunk('file1');
    const file2 = yield readDataThunk('file2');
    const file3 = yield readDataThunk('file3');
}

let g = gen();
g.next().value((err, file1) => {
    g.next(file1).value((err,file2) => {
        g.next(file2).value((err, file3) => {
            g.next(file3);
        })
    })
})

// 进一步封装，改进嵌套问题
function run(gen){
    const next = (err, data) => {
        let res = get.next(data);
        if(res.done) return;
        res.value(next);
    }
    next();
}
run(g)
```

### co函数库

co 函数库是著名程序员 TJ 发布的一个小工具，用于处理 Generator 函数的自动执行。核心原理其实就是上面讲的通过和 thunk 函数以及 Promise 对象进行配合，包装成一个库。它使用起来非常简单，比如还是用上面那段代码，第三段代码就可以省略了，直接引用 co 函数，包装起来就可以使用了

```javascript
const co = require('co');
let g = gen();
co(g).then(res =>{
  console.log(res);
})
```

这段代码比较简单，几行就完成了之前写的递归的那些操作。那么为什么 co 函数库可以自动执行 Generator 函数，它的处理原理是什么呢？

其实Generator 函数就是一个异步操作的容器。它的自动执行需要一种机制，当异步操作有了结果，能够自动交回执行权。

1. 回调函数。将异步操作包装成 Thunk 函数，在回调函数里面交回执行权。
2. Promise 对象。将异步操作包装成 Promise 对象，用 then 方法交回执行权。

**co 函数库其实就是将两种自动执行器（Thunk 函数和 Promise 对象），包装成一个库。**使用 co 的前提条件是，Generator 函数的 yield 命令后面，只能是 Thunk 函数或 Promise 对象。所以上面的例子用Promise就是：

```javascript
const readData = (fileName) => {
    return new Promise((resolve, reject) => {
        fs.readFile(fileName, (err, data) => {
            if(err) reject(err);
            resolve(data);
        });
    });
};

const gen = function* () {
    const file1 = yield readData('file1');
    const file2 = yield readData('file2');
    const file3 = yield readData('file3');
}

let g = gen();
g.next().value.then((err, file1) => {
    g.next(file1).value.then((err,file2) => {
        g.next(file2).value.then((err, file3) => {
            g.next(file3);
        })
    })
})

// 进一步封装，改进嵌套问题
function run(gen){
    const next = (err, data) => {
        let res = get.next(data);
        if(res.done) return;
        res.value.then(next);
    }
    next();
}
run(g)
```

关于更多co函数库的内部原理，可以去[源码库](https://github.com/tj/co/blob/master/index.js)学习，也可以参考[这里](https://www.ruanyifeng.com/blog/2015/05/co.html)

## async/await

ES6 之后 ES7 中又提出了新的异步解决方案：async/await，async 是 Generator 函数的语法糖，async/await 的优点是代码清晰（不像使用 Promise 的时候需要写很多 then 的方法链），可以处理回调地狱的问题。async/await 写起来使得 JS 的异步代码看起来像同步代码，其实异步编程发展的目标就是让异步逻辑的代码看起来像同步一样容易理解。

```javascript
var readData = function(fileName){
    return new Promise((resolve, reject) => {
        fs.readFile(fileName, (err, data) => {
            if(err) reject(err);
            resolve(data);
        });
    });
}
// 这里把 Generator的 * 换成 async，把 yield 换成 await
async function testReadData() {
    await readData("file1");
    await readData("file2");
    await readData("file3");
    console.log('end');
}
console.log(testReadData());  // file1  file2  file3  end
```

执行上面的代码，从结果中可以看出，在正常的执行顺序下，readData 这个函数由于使用的是 setTimeout 的定时器，回调会在一秒之后执行，但是由于执行到这里采用了 await 关键词，testReadData 函数在执行的过程中需要等待3个 readData 函数执行完成之后，再执行打印 end 的操作。

总结下来，async 函数对 Generator 函数的改进，主要体现在以下三点。

1. 内置执行器：Generator 函数的执行必须靠执行器，因为不能一次性执行完成，所以之后才有了开源的 co 函数库。但是，async 函数和正常的函数一样执行，也不用 co 函数库，也不用使用 next 方法，而 async 函数自带执行器，会自动执行。

1. 适用性更好：co 函数库有条件约束，yield 命令后面只能是 Thunk 函数或 Promise 对象，但是 async 函数的 await 关键词后面，可以不受约束。

1. 可读性更好：async 和 await，比起使用 * 号和 yield，语义更清晰明了。

所以，async/await也是javascript异步的终极解决方案。