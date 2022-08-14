---
layout: post
title: JavaScript基础之发布-订阅模式EventEmitter的实现
tags: EventEmitter
categories: JavaScript
date: 2018-03-26
---

我们知道EventEmitter是在Node.js中events模块封装的，但是在浏览器端还没有，在Vue中非父子组件通信，官方推荐过一种EventBus的解决方案，它是将EventBus作为组件传递数据的桥梁，所有组件共用的相同事件中心，可以向该中心发送事件或者接收事件，所有组件都可以收到通知，使用起来非常便利，其核心便是发布-订阅模式的实现。

接下来实现一个简易版的发布-订阅模式EventEmitter:

```javascript
function EventEmitter {
    this._events = {}
}

// 订阅事件
EventEmitter.prototype.on = function(eventName, callback) {
    const listeners = this._events[eventName] || [];
    listeners.push(callback);
    this._events[eventName] = listeners;
}

// 触发事件
EventEmitter.prototype.emit = function(eventName, ...args) {
    const listeners = this._events[eventName];
    const params = [].slice.call(args);
    listeners.forEach(fn => {
        fn.call(params);
    });
}

// 取消事件
EventEmitter.prototype.off = function(eventName, callback) {
	const listeners = this._events[eventName];
    this._events[eventName] = listeners && listeners.filter(fn => fn !== callback);
}

// 单次触发该事件
EventEmitter.prototype.once = function(eventName, callback) {
    const wrapperFun = (...args) => {
        callback.call(this, args);
        this.off(eventName, wrapperFun);
    }
    this.on(eventName, wrapperFun);
}
```

从上面实例也可以看出，一个基本的发布订阅模式主要有四部分组成：

1. 提供一个事件订阅方法on，并将事件存储到this._events中，等需要出发的时候，则直接从通过获取 __events 中对应事件的 listener 回调函数，而后直接执行该回调方法就能实现想要的效果。
2. 提供一个触发事件的方法emit，并发送对应的参数触发回调函数执行。
3. 提供一个取消订阅事件的方法off，能够对指定事件取消订阅。
4. 提供一个单词触发事件的方法。该事件的回调函数在执行完成后自动取消订阅。