---
layout: post
title: 西安GDG上关于主题[当Android遇上RxJava]的分享总结
tags: RxJava
categories: Android
date: 2016-01-10
---

## 前言
1月10号这一天，也是值得高兴的日子，一大早起来打开手机看到《RxJava Essentials》一书的作者Ivan.Morgillo给我在推特上发的消息点赞并转发后关注了我，这让我激动不已，可能对于我这种没见过大世面的人来说，这点小事或许就足以让我自己心里乐上三天。然后就是昨天下午在西安GDG做了关于RxJava的分享，下面是分享内容的总结。

##分享内容总结

大致分为以下三个主题线：

* 1.介绍了ReactiveX、RxJava
* 2.Android开发中遇到的常见场景
* 3.关于RxJava与Android的学习

### ReactiveX的介绍

我把它总结为以下三点：

* 1.扩展的观察者模式：通过订阅可观测对象的序列流然后做出反应。
* 2.迭代器模式：对对象序列进行迭代输出从而使订阅者可以依次对其处理。
* 3.函数式编程思想：简化问题的解决的步骤，让你的代码更优雅和简洁

然后介绍了ReactiveX在各个语言和平台上的实现,[官方地址](http://reactivex.io/languages.html)
最后对上面三点展开进行详细介绍：

* 1.先介绍GoF书中的观察者模式，被观察者发出事件，然后观察者（事件源）订阅然后进行处理。并指出其中的不足：比如观察者不知道是否出错与完成，还有就是整个过程是同步，会阻塞线程，从而引出所谓的“扩展”的观察者模式，除了提到的不足作为补充外，另外还有一点：如果没有观察者，被观察者是不会发出任何事件的。
* 2.迭代器模式：提供一种方法顺序访问一个聚合对象中的各种元素,而又不暴露该对象的内部表示，用《RxJava Essentials》一书做的的对比：迭代器模式在事件处理上采用的是“同步/拉式”的方式，而被观察者采用的是“异步/推式”的方式，而对观察者而言，显然后者更灵活。
* 3.对于函数式编程举例展示了代码风格的不同。

### RxJava的介绍

我也按照三点作为介绍

* 1.RxJava的核心
* 2.RxJava操作符
* 3.RxJava的扩展

然后对这三点展开来讲，其中最花时间的也是这一部分。
#### RxJava的核心对象

先是通过一个`Hello World`的例子介绍了几个容易混淆的类/接口：

* Observable
* OnSubscribe
* Observer
* Subscription
* Subscriber

代码展示如下：
```java
Observable<String> myObservable = Observable
        .create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                subscriber.onNext("Hello World!");
                subscriber.onCompleted();
            }
        });

Subscriber<String> mySubscriber = new Subscriber<String>() {
    @Override
    public void onCompleted() {}

    @Override
    public void onError(Throwable e) {}

    @Override
    public void onNext(String s) {
        Log.i("基础写法：", s);
    }
};

myObservable.subscribe(mySubscriber);

```

简化一下：
```java
Observable<String> myObservable = Observable.just("Hello World!");

Action1<String> onNextAction = new Action1<String>() {

    @Override
    public void call(String s) {
        Log.v("Action1简化后:", s);
    }
};

myObservable.subscribe(onNextAction);
```

匿名函数写法：
```java
Observable.just("Hello World!").subscribe(new Action1<String>() {
            @Override
            public void call(String s) {
                Log.v("匿名函数写法：",s);
            }
        });
```

最后用Java 8 lambdas(Retrolambda)表达式：
```java
Observable.just("Hello World!").subscribe(s -> Log.v("lambdas写法",s));
```

其中这一部分从源码角度简单概括了从被观察者创建，到观察者创建，最后再订阅的过程，并顺便指出了上面几个易混淆的类/接口之间的关系。

第二部分是关于异步的话题：

先是介绍这几种调度器：

* 1.Schedulers.immediate()
* 2.Schedulers.newThread()
* 3.Schedulers.trampoline()
* 4.Schedulers.io()
* 5.Schedulers.computation()
* 6.AndroidSchedulers.mainThread()

然后就是两个操作符：
* subscribeOn()：指定 subscribe() 所发生的线程，事件产生的线程
* observeOn()：指定 Subscriber 所运行在的线程，事件消费的线程

#### RxJava操作符

在讲操作符之前，先是介绍了直观有趣的宝石图：这里引用了一张官方的[图片](http://reactivex.io/documentation/observable.html),另外再附上一个国外程序员创建的动态的宝石图[网站](http://rxmarbles.com/)，虽然不全，但是作者一直在更新，相信后面会有更多，这有助于我们来理解操作符。

大致分为这几类展开介绍：
* 创建操作符：Create, Defer, From, Interval, Just, Range, Repeat, Timer等。
* 变换操作符：Map、FlatMap、ConcatMap等。
* 过滤操作符：Debounce, Distinct, ElementAt, Filter, First, Last, Sample, Skip, SkipLast, Take, TakeLast等。
* 合买操作符以及自定义操作符。

最后用一个例子做了下总结，需求如下：

* 将一个为数字的字符串数组元素转换为数字
* 过滤掉大于10的数字
* 去重
* 取最后面3个元素
* 累计求和

用代码实现就是：
```java
String[] numbers = {"11", "2", "2", "13", "4", "5","7"};
Observable.from(numbers)
        .map(s -> Integer.parseInt(s))
        .filter(i -> i < 10)
        .distinct()
        .takeLast(3)
        .reduce((number1,number2) -> number1 + number2)
        .subscribe(i -> System.out.println(i));
```

其中创建操作符例子如下：
```java
String[] strings = {"张三","李四","王五","赵六"};
Observable.from(strings)
        .subscribe(new Action1<String>() {
            @Override
            public void call(String s) {
                Log.i("name", s);
            }
        });
```

变换操作符例子：
```java
public void showUserName(String userName){
    textView.setText(userName);
}
```

```java
public void showUserName(String userName){
    Observable.just(userName).subscribe( new  Action1<String>(){
             @Override
             public void call(String s){
                textView.setText(s);
            }
    });
}
```

如果需要在显示前对这个字符串做处理，然后再展示，比如加“张三，你好”

* 方法1：我们可以对字符串本身操作   (不合适)
* 方法2：我们可以放到Action1.call()方法里做处理   （不合适）
* 方法3：使用操作符做变换：map    （RxJava的做法）

```java
public void showUserName(String userName){
    Observable.just(userName).map(new Func1<String,String>(){
          public String call(String text){
             return handleUserName(text);
           }
    }).subscribe( new Action1<String>(){
        public void call(String s){
            textView.setText(s);
          }
    });
}
```

关于flatMap()
```java
//打印出中国的所有省份名称。
List<Province>  provinceList = …
Observable.from(provinceList)
    .map(new Func1<Province,String>(){
        @Override
        public String call(Province province){
            return province.getName();
        }
    }).subscribe(new Action1<String>(){
         @Override
         public void call(String s){
            Log.i(“省份名称”,s)
         }
    });
```

```java
//打印出中国每个省份的所有城市  (不合适)
List<Province>  provinceList = …
Observable.from(provinceList)
    .subscribe(new Action1<Province>(){
        @Override
        public void call(Province province){
            List<City> cities = province.getCities();
            for (int i = 0; i < cities.size(); i++) {
                   City city = cities.get(i);
                   Log.i(“城市”, city.getName());
            }
        }
    });
```

```java
//RxJava做法
List<Province>  provinceList = …
Observable.from(provinceList)
    .flatMap(new Func1<Province,Observable<City>>(){
        @Override
        public Observable<City> call(Province province){
                 return Observable.from(province.getCities());
        }
    })
    .subscribe(new Action1<City>(){
            @Override
            public void call(City city){
               Log.i(“城市”, city.getName());
            }
    });
```

关于flatMap的应用扩展
```java
//介绍回调地狱
restAdapter.getApiService().getToken(new Callback<String>(){
    @Override
    public void success(String token) {
        restAdapter.getApiService().getUserInfo(token,new Callback<UserInfo>(){
            @Override
            public void success(UserInfo userInfo) {
                     showMessage(userInfo.getUser);
            }

            @Override
            public void failure(RetrofitError error) {
                //处理错误
                ...
            }
        });
    }

    @Override
    public void failure(RetrofitError error) {
        // Error handling
        ...
    }
});
```

如何用RxJava来解决：
```java
restAdapter.getApiService()
    .getToken()
    .flatMap(new Func1<String,Observable<UserInfo>>(){
            @Override
            public Observable<UserInfo> call(String token){
                     return restAdapter.getApiService().getUserInfo(token);
            }
    })
    .subscribe(new Action1<UserInfo>(){
            @Override
            public void call(UserInfo userInfo){
                     showMessage(userInfo.getUser);
            }
    })
```

加入线程切换

```java
restAdapter.getApiService().getToken()
    .flatMap(new Func1<String,Observable<UserInfo>>(){
            @Override
            public Observable<UserInfo> call(String token){
                     return restAdapter.getApiService().getUserInfo(token);
            }
    })
    .subscribeOn(Schedulers.io()) // 指定 subscribe() 发生在 IO 线程
    .observeOn(AndroidSchedulers.mainThread())// 指定 Subscriber 的回调发生在主线程
    .subscribe(new Action1<UserInfo>(){
            @Override
            public void call(UserInfo userInfo){
                     showMessage(userInfo.getUser);
            }
    })
```

#####操作符复用：

先介绍不合适的做法：

```java
<T> Observable<T> applySchedulers(Observable<T> observable) {
    return observable.subscribeOn(Schedulers.io())
     .observeOn(AndroidSchedulers.mainThread());
}
```

应用后，破坏了链式调用
```java
applySchedulers(restAdapter.getApiService().getToken()
    .flatMap(new Func1<String,Observable<UserInfo>>(){
            @Override
            public Observable<UserInfo> call(String token){
                     return restAdapter.getApiService().getUserInfo(token);
            }
    })
    ).subscribe(new Action1<UserInfo>(){
            @Override
            public void call(UserInfo userInfo){
                     showMessage(userInfo.getUser);
            }
    })
```

##### 我们加入转换器与Compose()
Transformer：继承Func1<Observable<T>, Observable<R>>的一个接口，其实是将一个Observable转换为另一个Observable
```java
<T> Transformer<T, T> applySchedulers() {
     return new Transformer<T, T>() {
       @Override
       public Observable<T> call(Observable<T> observable) {
         return observable.subscribeOn(Schedulers.io())
             .observeOn(AndroidSchedulers.mainThread());
       }
     };
}
```

用compose操作符做法：

```java
restAdapter.getApiService().getToken()
    .flatMap(new Func1<String,Observable<UserInfo>>(){
            @Override
            public Observable<UserInfo> call(String token){
                     return restAdapter.getApiService().getUserInfo(token);
            }
    })
    .compose(applySchedulers())
    .subscribe(new Action1<UserInfo>(){
            @Override
            public void call(UserInfo userInfo){
                     showMessage(userInfo.getUser);
            }
    })
```

#### RxJava的扩展
主要是针对以下几个开源库展开来说：
* 1.Rxbinding：用RxJava实现onClick,TextWatcher,check等事件绑定。
* 2.RxBus：用RxJava实现EventBus或者Otto。
* 3.RxPreferences：用RxJava实现Android中的SharedPreferences。
* 4.RxLifecycle：用来严格控制由于发布了一个订阅后，由于没有及时取消，导致Activity/Fragment无法销毁导致的内存泄露。
* 5.ReactiveNetwork：使用RxJava来监听网络连接状态和wifi信号强度变化。
* 6.RxPermissions：针对 Android 6.0 权限管理进行一个 Rx 封装的一个类库。
* 7.rxloader：用RxJava对loader的一个封装。
* 还有更多...


### Android应用场景

* 1.避免嵌套回调地狱问题。
* 2.使用debounce减少频繁的网络请求。避免每输入（删除）一个字就做一次联想。
* 3.使用combineLatest合并最近N个结点,注册的时候所有输入信息（邮箱、密码、电话号码等）合法才点亮注册按钮。
* 4.使用merge合并两个数据源,最后做统一处理。
* 5.使用concat和first做缓存，依次检查memory、disk和network中是否存在数据，任何一步一旦发现数据后面的操作都不执行。
* 6.使用timer做定时操作。
* 7.使用interval做周期性操作。
* 8.使用throttleFirst防止按钮重复点击
* 9.做响应式的界面。
* 更多...

部分例子代码：
```java
RxTextView.textChanges(searchEditText)
     .debounce(150, MILLISECONDS)
     .switchMap(Api::searchItems)
     .subscribe(this::updateList, t->showError());
```

```java
Observable.merge(getNews(), getHotNews(),
    new Func2<Response<News>, MyResponse<News>, Boolean>() {
        @Override
        public Boolean call(Response<News> response, Response<News> response2) {
            mData.clear();
            mData.addAll(response);
            mData.addAll(response2.msg);
            return true;
        }
    })
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Subscriber<Boolean>() {
        @Override
        public void onCompleted() {}
        @Override
        public void onError(Throwable e) {}
        @Override
        public void onNext(Boolean o) {
               mAdapter.notifyDataSetChanged();
        }
    });
```

```java
RxView.clicks(button)
      .throttleFirst(1, TimeUnit.SECONDS)
      .subscribe(new Observer<Object>() {
          @Override
          public void onCompleted() {
                log.d ("completed");
          }

          @Override
          public void onError(Throwable e) {
                log.e("error");
          }

          @Override
          public void onNext(Object o) {
               log.d("button clicked");
          }
      });
```


```java
SharedPreferences preferences = PreferenceManager.getDefaultSharedPreferences(this);
RxSharedPreferences rxPreferences = RxSharedPreferences.create(preferences);

Preference<Boolean> checked = rxPreferences.getBoolean("checked", true);

CheckBox checkBox = (CheckBox) findViewById(R.id.cb_test);
RxCompoundButton.checkedChanges(checkBox)
        .subscribe(checked.asAction());
```

```java
Observable<String> memory = Observable
    .create(new Observable.OnSubscribe<String>() {
        @Override
        public void call(Subscriber<? super String> subscriber) {
            if (memoryCache != null) {
                subscriber.onNext(memoryCache);
            } else {
                subscriber.onCompleted();
            }
        }
    });

Observable<String> disk = Observable
    .create(new Observable.OnSubscribe<String>() {
        @Override
        public void call(Subscriber<? super String> subscriber) {
            String cachePref = rxPreferences.getString("cache").get();
            if (!TextUtils.isEmpty(cachePref)) {
                subscriber.onNext(cachePref);
            } else {
                subscriber.onCompleted();
            }
        }
    });

Observable<String> network = Observable.just("network");

//依次检查memory、disk、network
Observable.concat(memory, disk, network)
    .first()
    .subscribeOn(Schedulers.newThread())
    .subscribe(s -> {
        memoryCache = "memory";
        System.out.println("subscribe: " + s);
    });
```

#### 感谢/参考

感谢Jake大神的启蒙，感谢Ivan.Morgillo的书《RxJava Essentials》,感谢扔物线的文章和大头鬼的翻译文章，同时也是一路看他们的文章走过来的，最后感谢所有分享RxJava的小伙伴们。

* http://gank.io/post/560e15be2dca930e00da1083
* http://blog.csdn.net/lzyzsd/article/details/41833541
* http://blog.csdn.net/theone10211024/article/details/50435325
* 《RxJava Essentials》
* http://reactivex.io

### 关于RxJava和Android的学习

主要从渠道，知识点和资源几方面介绍了下学习，提到[MobDevGroup](http://mobdevgroup.com)这个资源站。

## GDG总结

由于时间上的问题，没有对一些原理进行讲解，尤其是变换等，大部分是在讲应用，总之呢除了对自己知识点一次不错的总结外，也是对自己的一次历练，接下来再接再厉。最后附上这次的PPT下载地址：

* Mac [keynote](http://vdisk.weibo.com/s/CeH3i0tfvuvLU)
* Windows [PowerPoint](http://vdisk.weibo.com/s/CeH3i0tfvuvMd)


