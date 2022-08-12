---
layout: post
title: JavaScript基础之继承实现
tags: 继承
categories: JavaScript
date: 2018-03-06
---

继承可以使得子类具有父类的方法和属性，同时也可以重新定义父类的某些属性，并重写或者覆盖某些属性和方法。使得子类具有与父类不同的属性和方法。

接下来我们就来一起看看实现继承的几种方式

## 1. 原型链继承

原型链继承是比较常见的继承方式之一，其基本思想是利用原型让一个引用类型继承另一个引用类型的属性和方法。其中涉及的构造函数、原型和实例，三者之间存在着一定的关系，即每一个构造函数都有一个原型对象（构造函数的`prototype`属性），原型对象又包含一个指向构造函数的指针（原型对象的`constructor`属性），而实例则包含一个原型对象的指针（实例对象的`__proto__`属性）。

```javascript
function Parent() {
    this.name = "这是父类";
    this.content = [1,2,3];
}

Parent.prototype.getContent = function() {
    console.log(this.content);
}

function Child() {
    
}

Child.prototype = new Parent();

var child1 = new Child();
child1.content.push(4);
console.log(child1.getContent())  // Array [1,2,3,4]

var child2 = new Child();
child2.content.push(5);  
console.log(child2.getContent())  // Array [1,2,3,4,5]
```

虽然上面子类可以复用父类的属性和方法，但是这里面存在着一个问题：两个实例共用一个原型对象，它们的内存空间是共享的，当一个发生变化的时候，另外一个也随之进行了变化。下面我们看看其他继承方式如何解决这个问题

## 2. 构造函数继承

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
    Parent.call(this, name)
}

var child1 = new Child("这是子类1");
child1.content.push(4);
console.log(child1.getContent())  // 报错
console.log(child1.getContent1())  // Array [1,2,3,4]

var child2 = new Child("这是子类2");
child2.content.push(5);  
console.log(child2.getContent1())  // Array [1,2,3,5]

```

从上面实例我们也可以总结该方法的优缺点：

优点：

1. 解决了第一种继承方式的缺点，即避免了引用类型属性被所有实例共享
2. 可以直接在Child中向Parent传参

缺点：

1. 只能继承父类的实例属性和方法，不能继承其原型属性或者方法。

那么上面两者都有不足的地方， 我们可以组合上面两种实现方式

## 3. 组合继承

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

从上面实例可以看出，组合继承解决了前面提到的两个问题，既具有原型链继承能够复用函数的特性，又有借用构造函数方式能够保证每个子类实例能够拥有自己的属性以及向父类传参的特性，但组合继承也并不是完美实现继承的方式，因为这种方式在创建子类时会调用两次父类的构造函数，这是我们不愿意看到的，因为每一次的调用都会是一次性能开销，接着看我们如何解决这个问题。

## 4. 原型式继承

ECMAScript5通过新增Object.create()方法规范化了原型式继承，这个方法接收两个参数：一个用作新对象原型的对象和为新对象定义属性的对象（可选参数）。

```javascript
let parent = {
    name: '这是父类',
    content: [1,2,3],
    
    getContent: function() {
        return this.content;
    },
    getName: function(){
        return this.name;
    }
}

let child1 = Object.create(parent);
child1.name = "这是子类child1"
child1.content.push(4);
console.log(child1.name);  // 这是子类child1
console.log(child1.name === child1.getName()) // true
console.log(child1.getContent())  // Array [1,2,3,4]

let child2 = Object.create(parent);
child2.content.push(5) 
console.log(child2.getContent())  // Array [1,2,3,4,5] 
```

通过以上代码也可以知道，这种继承方式缺点也很明显，多个实例引用类型属性指向相同的内存，接下来看如何优化它。

## 5. 寄生式继承

寄生式继承和上面的原型式继承密切相关，即创建一个仅用于封装继承函数过程的函数，在该函数内部以某种方式来增强对象，最后返回该对象。

```javascript
let parent = {
    name: '这是父类',
    content: [1,2,3],
    
    getContent: function() {
        return this.content;
    }
}

function createChild(){
    let child = Object.create(parent);
    child.getName = function() {
        return this.name;
    }
    return child;
}

let child = createChild();
console.log(child.getName());   
console.log(child.getContent());
```

该方式通过createChild函数在继承过程中又增加了一个getName方法，这样的继承方式就是寄生式继承。而上面第三点提到的性能开销问题，可以通过下面方式来解决:寄生式+组合继承

## 6. 寄生组合式继承

所谓寄生组合式继承，即通过构造函数来继承属性，通过原型链继承方法，背后的基本思路是：不必为了指定子类的原型而调用父类的构造函数，我们所需要的无非就是父类原型的一个副本而已。

```javascript
function Parent() {
    this.name = '这是父类';
    this.content = [1,2,3];
}

Parent.prototype.getName = function() {
    return this.name;
}

function Child() {
    Parent.call(this);
    this.age = 18;
}

function create(parent,child) { 
    //改变子类的原型，同时纠正构造函数
    child.prototype = Object.create(parent.prototype);
    child.prototype.constructor = child;
}

create(Parent, Child);
Child.prototype.getContent = function() {
    return this.content;
}
var child1 = new Child(); 

console.log(child1.getName())
console.log(child1.getContent());
```

通过以上代码可以看出寄生组合继承的高效率在于它只调用了一次父类构造函数，同时还能够保持原型链不变，能够正常使用 instanceof 和 isPrototypeOf() ，寄生组合继承被普遍认为是引用类型最理想的继承方式。

## ES6 关键字extends

最后我们一起来看下ES6的语法糖，深入底层逻辑研究extends是如何实现的，这里借助[babel转换](https://www.babeljs.cn/repl/)，代码如下：

```javascript
class Parent {
  constructor(name) {
    this.name = name
  }
  // 原型方法
  // 即 Person.prototype.getName = function() { }
  // 下面可以简写为 getName() {...}
  getName = function () {
    console.log('Parent:', this.name)
  }
}
class Child extends Person {
  constructor(name, age) {
    // 子类中存在构造函数，则需要在使用“this”之前首先调用 super()。
    super(name)
    this.age = age
  }
}
const child = new Child('yuxingxin', 32)
child.getName() // 成功访问到父类的方法
```

我们借助Babel转译成ES5，来看下代码是如何实现的？

```javascript
"use strict";

function _typeof(obj) {
  "@babel/helpers - typeof";
  return (
    (_typeof =
      "function" == typeof Symbol && "symbol" == typeof Symbol.iterator
        ? function (obj) {
            return typeof obj;
          }
        : function (obj) {
            return obj &&
              "function" == typeof Symbol &&
              obj.constructor === Symbol &&
              obj !== Symbol.prototype
              ? "symbol"
              : typeof obj;
          }),
    _typeof(obj)
  );
}

function _inherits(subClass, superClass) {
  if (typeof superClass !== "function" && superClass !== null) {
    throw new TypeError("Super expression must either be null or a function");
  }
  // 注意看这里
  subClass.prototype = Object.create(superClass && superClass.prototype, {
    constructor: { value: subClass, writable: true, configurable: true }
  });
  Object.defineProperty(subClass, "prototype", { writable: false });
  if (superClass) _setPrototypeOf(subClass, superClass);
}

function _setPrototypeOf(o, p) {
  _setPrototypeOf = Object.setPrototypeOf
    ? Object.setPrototypeOf.bind()
    : function _setPrototypeOf(o, p) {
        o.__proto__ = p;
        return o;
      };
  return _setPrototypeOf(o, p);
}

function _createSuper(Derived) {
  var hasNativeReflectConstruct = _isNativeReflectConstruct();
  return function _createSuperInternal() {
    var Super = _getPrototypeOf(Derived),
      result;
    if (hasNativeReflectConstruct) {
      var NewTarget = _getPrototypeOf(this).constructor;
      result = Reflect.construct(Super, arguments, NewTarget);
    } else {
      result = Super.apply(this, arguments);
    }
    return _possibleConstructorReturn(this, result);
  };
}

function _possibleConstructorReturn(self, call) {
  if (call && (_typeof(call) === "object" || typeof call === "function")) {
    return call;
  } else if (call !== void 0) {
    throw new TypeError(
      "Derived constructors may only return object or undefined"
    );
  }
  return _assertThisInitialized(self);
}

function _assertThisInitialized(self) {
  if (self === void 0) {
    throw new ReferenceError(
      "this hasn't been initialised - super() hasn't been called"
    );
  }
  return self;
}

function _isNativeReflectConstruct() {
  if (typeof Reflect === "undefined" || !Reflect.construct) return false;
  if (Reflect.construct.sham) return false;
  if (typeof Proxy === "function") return true;
  try {
    Boolean.prototype.valueOf.call(
      Reflect.construct(Boolean, [], function () {})
    );
    return true;
  } catch (e) {
    return false;
  }
}

function _getPrototypeOf(o) {
  _getPrototypeOf = Object.setPrototypeOf
    ? Object.getPrototypeOf.bind()
    : function _getPrototypeOf(o) {
        return o.__proto__ || Object.getPrototypeOf(o);
      };
  return _getPrototypeOf(o);
}

function _defineProperties(target, props) {
  for (var i = 0; i < props.length; i++) {
    var descriptor = props[i];
    descriptor.enumerable = descriptor.enumerable || false;
    descriptor.configurable = true;
    if ("value" in descriptor) descriptor.writable = true;
    Object.defineProperty(target, descriptor.key, descriptor);
  }
}

function _createClass(Constructor, protoProps, staticProps) {
  if (protoProps) _defineProperties(Constructor.prototype, protoProps);
  if (staticProps) _defineProperties(Constructor, staticProps);
  Object.defineProperty(Constructor, "prototype", { writable: false });
  return Constructor;
}

function _classCallCheck(instance, Constructor) {
  if (!(instance instanceof Constructor)) {
    throw new TypeError("Cannot call a class as a function");
  }
}

function _defineProperty(obj, key, value) {
  if (key in obj) {
    Object.defineProperty(obj, key, {
      value: value,
      enumerable: true,
      configurable: true,
      writable: true
    });
  } else {
    obj[key] = value;
  }
  return obj;
}

var Parent = /*#__PURE__*/ _createClass(
  function Parent(name) {
    _classCallCheck(this, Parent);

    _defineProperty(this, "getName", function () {
      console.log("Parent:", this.name);
    });

    this.name = name;
  } // 原型方法
  // 即 Person.prototype.getName = function() { }
  // 下面可以简写为 getName() {...}
);

var Child = /*#__PURE__*/ (function (_Person) {
  _inherits(Child, _Person);

  var _super = _createSuper(Child);

  function Child(name, age) {
    var _this;

    _classCallCheck(this, Child);

    // 子类中存在构造函数，则需要在使用“this”之前首先调用 super()。
    _this = _super.call(this, name);
    _this.age = age;
    return _this;
  }

  return _createClass(Child);
})(Person);

var child = new Child("yuxingxin", 20);
child.getName(); // 成功访问到父类的方法
```

从上面编译完的源代码也可以看出，它采用的也是寄生组合式继承的方式。

## 总结

通过 Object.create 来划分不同的继承方式，最后的寄生式组合继承方式是通过组合继承改造之后的最优继承方式，而 extends 的语法糖和寄生组合继承的方式基本类似。

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h53un87lgpj21ce0i2juj.jpg)

