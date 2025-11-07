---
layout: post
title: Android Framework之Binder原理分析
tags: Framework
categories: Android
date: 2019-07-08
--- 

Binder是Android Framework层一个不可或缺的存在，了解Framework的前提必须先掌握Binder原理，它是Android进程间通信的一种方式，我们在应用程序使用的四大组件，可以运行在同一个进程，也可以运行在不同进程，进程之间通信就依赖Binder，另外我们之前了解到的系统服务，像AMS、PMS、WMS等都是基于Binder IPC 进行通信的。

## 进程空间

Linux系统对于进程空间的划分，主要分为用户空间和内核空间，用户空间（User Space）是用户程序运行的空间，内核空间（Kernel）是系统内核运行的空间。而如果它们之间需要进行交互时就需要通过系统调用的方式，常见的函数有：

1. copy_from_user()：将用户空间的数据拷贝到内核空间
2. copy_to_user()：将内核空间的数据拷贝到用户空间

上面当程序执行系统调用进入内核执行时，此时进程就处于内核态，相反在执行自己的代码时，就处于用户态。

## 进程隔离与传统IPC

操作系统中，进程与进程间的内存是不共享的，即A 进程无法直接访问 B 进程的数据，B进程也无法访问A进程的数据，如果双方要进行通信，就必须采用特殊的通信机制，即IPC，传统的Linux通信原理可以用下面这张图表示：

![image-20220902182418616](https://tva1.sinaimg.cn/large/e6c9d24ely1h5sfo5sy9yj20yd0oego8.jpg)

上面两个进程如果按照传统IPC进程一次通信的话，步骤如下：

1. 数据发送进程通过系统调用copy_from_user将要发送的数据从用户空间拷贝到内核缓存区
2. 数据接收进程同样通过系统调用copy_to_user将内核缓存区中的数据拷贝到用户空间。

这样一来就完成了一次进程间通信。

但是这样的方式有以下问题：

1. 性能低下，一次通信需要拷贝2次数据
2. 接收数据的缓存由接收方提供，但是它却不知道需要多大的缓存空间才能满足要求，因此只能开辟尽可能大的空间或者先调用API接收消息头来获取消息体的大小，不管哪一种，很显然都很浪费空间。

## 内存映射

内存映射的实现过程主要是通过`Linux`系统下的系统调用函数：`mmap（）`，该函数可以创建一块虚拟的内存区域，并且将这块内存区域与共享对象之间建立映射关系

![image-20220902184457738](https://tva1.sinaimg.cn/large/e6c9d24ely1h5sg9n4v5oj20nr0gq3zz.jpg)

这样以来，就会发现不但减少了数据拷贝的次数，也提高了用户空间和内核空间之间的高校交互，而且用内存读写代替了I/O读写，提高文件读取效率。

## Binder

通过上面我们知道，跨进程IPC需要内核空间支持，但是Binder并不是Linux系统的一部分，需要实现IPC就必须依赖Linux的动态内核可加载模块（Loadable Kernel Module）,简称LKM，它在运行时被链接到内核作为内核的一部分运行。这样，Android 系统就可以通过动态添加一个内核模块运行在内核空间，用户进程之间通过这个内核模块作为桥梁来实现通信。在 Android 系统中，这个运行在内核空间，负责各个用户进程通过 Binder 实现通信的内核模块就叫 **Binder 驱动**（Binder Dirver）。

传统的IPC通信需要两次拷贝，但是Android为了提高效率，就借助上面提到的内存映射，即将用户空间的一块内存区域映射到内核空间。映射关系建立后，用户对这块内存区域的修改可以直接反应到内核空间；反之内核空间对这段区域的修改也能直接反应到用户空间。我们看下面这张图：

![image-20220902190328270](https://tva1.sinaimg.cn/large/e6c9d24ely1h5sgswloltj20wk0noq61.jpg)

它的通信过程大体是这样：

1. 首先 Binder 驱动在内核空间创建一个数据接收缓存区；
2. 接着在内核空间开辟一块内核缓存区，建立**内核缓存区**和**内核中Binder建立的数据接收缓存区**之间的映射关系，以及**内核中Binder建立的数据接收缓存区**和**数据接收接收进程用户空间**的映射关系；
3. 数据发送进程通过系统调用 copy_from_user() 将数据 copy 到内核中的**内核缓存区**，由于内核缓存区和数据接收进程的用户空间存在内存映射，因此也就相当于把数据发送到了数据接收进程的用户空间，这样便完成了一次进程间的通信。

## Android中的Binder通信模型

在Android中，一次完整的进程间通信，通信的双方我们称为客户端进程和服务端进程，由于进程隔离机制，要实现两者通信就需要借助Binder驱动。它是基于C/S 架构，由一系列组件组成，包括Client、Server、ServiceManager、Binder驱动，其中 Client、Server、Service Manager 运行在用户空间，Binder 驱动运行在内核空间。其中 Service Manager 和 Binder 驱动由系统提供，而 Client、Server 由应用程序来实现。Client、Server 和 ServiceManager 均是通过系统调用 open、mmap 和 ioctl 来访问设备文件 /dev/binder，从而实现与 Binder 驱动的交互来间接的实现跨进程通信。

![image-20220902192250013](https://tva1.sinaimg.cn/large/e6c9d24ely1h5shd1tsfej212d0ib75x.jpg)

从上面我们大致梳理出Binder的大体过程:

1. Server进程向Binder驱动发起服务注册请求
2. Binder驱动将服务注册请求发送给Service Manager进程
3. Service Manager进程添加该Server进程，即已注册服务
4. Client进程向Binder驱动发起获取服务的请求，传递获取服务的名称
5. Binder驱动将请求转发给Service Manager进程
6. Service Manager进程根据服务名称查找到Client请求的Server对于服务信息
7. 通过Binder驱动将上述服务信息返回给Client进程
8. Binder驱动为跨进程通信做准备，实现内存映射
9. Client进程将数据发送到Server进程
10. Server进程根据Client进程请求要求，调用对应的目标方法
11. Server进程将目标方法的结果返回给Client进程

## 代理模式

当Client进程想要获取Server进程的数据对象时，并不是真的会返回对象给Client进程，而是返回一个和目标对象一模一样的代理对象，这个代理对象有着和目标对象一样的方法，当Client进程获取服务请求，调用对应的方法，这时候代理对象会利用Binder驱动找到真的Binder对象，并通知Server进程调用目标方法。并把结果返回给代理对象的Binder驱动，然后转发给Client进程，一次通信就完成了。

![image-20220902193902307](https://tva1.sinaimg.cn/large/e6c9d24ely1h5shtwlgcrj20x507fjs6.jpg)

综上，Client进程的操作其实是对代理对象的操作，代理对象利用Binder驱动找到真正的Binder实体，然后通知Server进程调用对应的目标方法完成操作。

## Android AIDL

我们通过AIDL文件生成的Java文件，包含一个接口，一个Stub静态的抽象类，一个Proxy的静态类，其中Proxy是Stub的静态内部类，Stub又是借口的静态内部类，Android这样设计的目的是为了避免当有多个AIDL文件时，放在同一个文件夹Stub、Proxy会产生重名问题。

- **IBinder** : IBinder 是一个接口，代表了一种跨进程通信的能力。只要实现了这个借口，这个对象就能跨进程传输。
- **IInterface** : IInterface 代表的就是 Server 进程对象具备什么样的能力（能提供哪些方法，其实对应的就是 AIDL 文件中定义的接口）
- **Binder** : Java 层的 Binder 类，代表的其实就是 Binder 本地对象。BinderProxy 类是 Binder 类的一个内部类，它代表远程进程的 Binder 对象的本地代理；这两个类都继承自 IBinder, 因而都具有跨进程传输的能力；实际上，在跨越进程的时候，Binder 驱动会自动完成这两个对象的转换。
- **Stub** : AIDL 的时候，编译工具会给我们生成一个名为 Stub 的静态内部类；这个类继承了 Binder, 说明它是一个 Binder 本地对象，它实现了 IInterface 接口，表明它具有 Server 承诺给 Client 的能力；Stub 是一个抽象类，具体的 IInterface 的相关实现需要开发者自己实现。

## 总结

对Binder一个相对完整的解释：

从进程间通信的角度看，Binder 是一种进程间通信的机制；

从 Server 进程的角度看，Binder 指的是 Server 中的 Binder 实体对象；

从 Client 进程的角度看，Binder 指的是对 Binder 代理对象，是 Binder 实体对象的一个远程代理

从传输过程的角度看，Binder 是一个可以跨进程传输的对象；Binder 驱动会对这个跨越进程边界的对象对一点点特殊处理，自动完成代理对象和本地对象之间的转换。

相比其他进程间通信方式来说，它有以下优点：

1. 性能上只需要一次数据拷贝，而Socket需要两次，其传输效率低，开销大，主要用在跨网络的进程间通信和本机上进程间的低速通信。管道、消息队列采用存储-转发方式，即数据先从发送方缓存区拷贝到内核开辟的缓存区中，然后再从内核缓存区拷贝到接收方缓存区，至少也需要两次拷贝过程。共享内存虽然无需拷贝，但控制复杂，难以使用
2. 安全性上为每个App分配UID/PID，进程的UID/PID是鉴别进程身份的标识，而传统的 IPC 没有任何安全措施，完全依赖上层协议来确保
3. 稳定性上基于C/S架构，职责明确，架构清晰，方便使用