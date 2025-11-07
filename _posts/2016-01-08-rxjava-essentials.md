---
layout: post
title: RxJava Essentials翻译总结
tags: RxJava
categories: Android
date: 2016-01-08
---
##前言
在前年的时候，一直忙于工作，偶尔关注下开源社区，平时在使用retrofit的库时一直采用传统的回调，当时看官网发现也可以Observable对象，很好奇，但是一直不知道这是什么？慢慢的，关注Jake大神，才知道是RxJava，当时并没有引起我对RxJava 的好奇，也就没有太在意，但是我的心里一直有个梗就是在使用回调时如何让嵌套回调的代码看起来不是那么槽糕，用今天的话说就是回调地狱，直到去年，国内一些积极推动RxJava的大神们才真正让我认识了它，可能最让我印象深刻的一点就是它解决了我这么多年的那个梗，从大头鬼的深入浅出系列到扔物线的给Android开发者的RxJava详解，我决定想系统的学一下它，一直想找本中文书，可是没找到，直到有一天发现国外的这本《RxJava Essentials》，看了一下之后，欣喜之余，决定把它翻译出来，算是巩固学习。

##书内容介绍
全书分了八章，前两章介绍Rx，并引出Rx在Android中的引用，三四五六章着重讲RxJava的操作符，从创建、过滤一直讲到变换、组合，最后一张结合Android开源库Retrofit来一起使用，总之对于Android开发者来讲是本不错的基础书。

### **1.RX-from .NET to RxJava**

> 本章带你进入reactive的世界。我们会比较reactive 方法和传统方法，进而探索它们之间的相似和不同的地方。

### **2.Why Observables?**

> 本章会对观察者模式做一个概述，如何实现它以及怎样用RxJava来进行扩展，被观察者是什么，以及被观察者如何与迭代联系到一起的。

### **3.Hello Reactive World**

> 本章会利用我们所学的知识来创建第一个reactive Android应用。

### **4.Filtering Observables**

> 本章我们会研究Observable序列的本质:filtering.我们也将学到如何从一个发出的Observable中选取我们想要的值，如何获得一个有限的数值，如何处理溢出的场景，以及更多有用的技巧。

### **5.Transforming Observables**

> 本章将讲述如何通过变换Observable序列来创建出我们所需要的序列。

### **6.Combining Observables**

> 本章将研究与函数结合，同时也会学到当创建我们想要的Observable时又如何与多个Observable协同工作。

### **7.Schedulers-Defeating the Android MainThread Issue**

> 本章将介绍如何使用RxJava Schedulers 来处理多线程和并发编程。我们也将用reactive的方式来创建网络操作、内存访问、耗时处理。

### **8.REST in peace-RxJava and Retrofit**

> 本章教会你如何让Square公司的Retrofit和RxJava结合来一起使用，来创建一个更高效的REST客户端程序。

## 后续
Rx给了我们一种新的学习扩展和并发的方式，它和面向对象一样是一种新的思想，熟练的使用它可以很容易的帮助我们处理日常繁杂的业务逻辑，同时又不会搞乱你的代码，建议开发者都可以学习下，另外这里也推荐另外一本《Learning Reactive Programming》，是它的姊妹篇，介绍响应式编程，同样推荐看看。
![RxJava](https://github.com/yuxingxin/RxJava-Essentials-CN/blob/master/images/rxjava.jpg)
【RxJava Essentials】

![Learning Reactive Programming](https://tva1.sinaimg.cn/large/e6c9d24ely1h58t0ygq91j20u0110gr9.jpg)
【Learning Reactive Programming】

《RxJava Essentials》[翻译中文版电子书下载地址](https://www.gitbook.com/book/yuxingxin/rxjava-essentials-cn/)

《RxJava Essentials》[英文版下载地址](https://vdisk.weibo.com/s/CeH3i0tfvZMVq)

《Learning Reactive Programming》[英文版下载地址](https://vdisk.weibo.com/s/CeH3i0tfvZMfT)
