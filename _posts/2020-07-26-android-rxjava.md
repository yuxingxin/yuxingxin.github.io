---
layout: post
title: Android三方开源库之RxJava3源码分析
tags: 源码分析
categories: Android
date: 2020-07-26
---

RxJava作为主流的框架之一，有着丰富的功能操作符以及便捷的线程切换，深受Android开发者喜爱，本文尝试从源码角度分析其工作原理。

## Single.just

Single.just最为最简单的模型，可以看下它是如何工作的：

```java
Single<Integer> single = Single.just(1);
single.subscribe(new SingleObserver<Integer>() {
    @Override
    public void onSubscribe(Disposable d) {
    }

    @Override
    public void onSuccess(Integer integer) {
    }

    @Override
    public void onError(Throwable e) {
    }
});
```

先看下just方法：

```java
// Single.java
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.NONE)
@NonNull
public static <T> Single<T> just(final T item) {
    ObjectHelper.requireNonNull(item, "value is null");
    return RxJavaPlugins.onAssembly(new SingleJust<T>(item));
}
```

通过RxJavaPlugins.onAssembly方法返回一个SingleJust对象：

```java
// RxJavaPlugins.java
@NonNull
public static <T> Single<T> onAssembly(@NonNull Single<T> source) {
    Function<? super Single, ? extends Single> f = onSingleAssembly;
    if (f != null) {
        return apply(f, source);
    }
    return source;
}
```

这里的onAssembly是一个钩子函数，如果f不为空的时候，处理完后可以看到，返回了参数本身，即上面说的SingleJust对象，紧接着看下subscribe方法：

```java
// Single.java
@SchedulerSupport(SchedulerSupport.NONE)
@Override
public final void subscribe(SingleObserver<? super T> observer) {
    ...
    try {
        subscribeActual(observer);
    } catch (NullPointerException ex) {
        throw ex;
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        NullPointerException npe = new NullPointerException("subscribeActual failed");
        npe.initCause(ex);
        throw npe;
    }
}
```

这里实际上调用的是subscribeActual方法，也就是实际订阅的方法，即SingleJust的subscribeActual方法：

```java
// SingleJust.java
@Override
protected void subscribeActual(SingleObserver<? super T> observer) {
    observer.onSubscribe(Disposables.disposed());
    observer.onSuccess(value);
}
```

可以看到这里直接将值，直接传给观察者么也就是上面示例中的SingleObserver对象，然后回调其onSubscribe和onSuccess方法，因为不会出错，所以没有onError方法，这样就完成一次最简单的订阅。

接下来看下如果中间有操作符时，该怎么处理。

## Map操作符

```java
Single<Integer> single = Single.just(1);
Single<String> singleString = single.map(new Function<Integer, String>() {
    @Override
    public String apply(Integer integer) throws Exception {
        return integer.toString();
    }
});
singleString.subscribe(new SingleObserver<String>() {
    @Override
    public void onSubscribe(Disposable d) {

    }

    @Override
    public void onSuccess(String s) {

    }

    @Override
    public void onError(Throwable e) {

    }
});
```

首先看下map方法：

```java
// Single.java
@CheckReturnValue
@NonNull
@SchedulerSupport(SchedulerSupport.NONE)
public final <R> Single<R> map(Function<? super T, ? extends R> mapper) {
    ObjectHelper.requireNonNull(mapper, "mapper is null");
    return RxJavaPlugins.onAssembly(new SingleMap<T, R>(this, mapper));
}
```

同样是一个钩子函数，返回一个SingleMap对象，其中构造函数中第一个参数为上游的Single，mapper为一个转换器对象，紧接着看下SingleMap的subscribe方法，其实也是在调用其subscribeActual方法:

```java
// SingleMap.java
public final class SingleMap<T, R> extends Single<R> {
    final SingleSource<? extends T> source;

    final Function<? super T, ? extends R> mapper;

    public SingleMap(SingleSource<? extends T> source, Function<? super T, ? extends R> mapper) {
        this.source = source;
        this.mapper = mapper;
    }

    @Override
    protected void subscribeActual(final SingleObserver<? super R> t) {
        source.subscribe(new MapSingleObserver<T, R>(t, mapper));
    }

    static final class MapSingleObserver<T, R> implements SingleObserver<T> {

        final SingleObserver<? super R> t;

        final Function<? super T, ? extends R> mapper;

        MapSingleObserver(SingleObserver<? super R> t, Function<? super T, ? extends R> mapper) {
            this.t = t;
            this.mapper = mapper;
        }

        @Override
        public void onSubscribe(Disposable d) {
            t.onSubscribe(d);
        }

        @Override
        public void onSuccess(T value) {
            R v;
            try {
                // 执行方法转换
                v = ObjectHelper.requireNonNull(mapper.apply(value), "The mapper function returned a null value.");
            } catch (Throwable e) {
                Exceptions.throwIfFatal(e);
                onError(e);
                return;
            }

            t.onSuccess(v);
        }

        @Override
        public void onError(Throwable e) {
            t.onError(e);
        }
    }
}
```

其中source为我们上游的被观察者，MapSingleObserver为我们内部创建的观察者对象，它将下游的观察者t，即示例中的SingleObserver和转换器对象包装起来，通过调用上游被观察者的subscribe方法，完成订阅关系。即该方法内部调用onSubscribe、onSuccess、onError方法时，会将事件传递给下游，即通过t调用其onSubscribe、onSuccess、onError方法。

综上：

1. 基于上游观察者对象Single通过操作符创建了一个新的被观察者对象SingleMap
2. 在被观察者对象SingleMap内部创建了一个新的观察者MapSingleObserver
3. 通过SingMap的subscribe方法（实际上是subscribeActual方法）和中介MapSingleObserver将上游被观察者对象Single与下游观察者SingleObserver联系起来。

## Dispose

我们可以通过dispose方法来让上游或内部调度器停止工作，达到丢失的效果。

| 操作符              | 上下游 | 后续事件 | 延迟 |
| ------------------- | ------ | -------- | ---- |
| Single.just         | 无     | 无       | 无   |
| Single.delay        | 无     | 无       | 有   |
| Single.map          | 有     | 无       | 无   |
| Observable.delay    | 无     | 无       | 有   |
| Observable.interval | 无     | 有       | 有   |
| Observable.map      | 有     | 有       | 无   |

#### Single.just

我们看下其subscribeActual方法，该方法中给观察者的是一个全局的Disposable对象，因为时间太短，所以不用对其进行取消。

```java
// SingleJust.java
@Override
protected void subscribeActual(SingleObserver<? super T> observer) {
    observer.onSubscribe(Disposables.disposed());
    observer.onSuccess(value);
}
```

#### Observable.interval

首先来看下示例：

```java
Observable<Long> integerObservable = Observable.interval(0, 1, TimeUnit.SECONDS);
integerObservable.subscribe(new Observer<Long>() {
    @Override
    public void onSubscribe(Disposable d) {

    }

    @Override
    public void onNext(Long aLong) {

    }


    @Override
    public void onError(Throwable e) {

    }

    @Override
    public void onComplete() {

    }
});
```

上面示例中Observable.interval方法返回一个ObservableInterval对象，然后看下其subscribeActual方法：

```java
// ObservableInterval.java
@Override
public void subscribeActual(Observer<? super Long> observer) {
    // 创建观察者，实现了Disposable和Runnable
    IntervalObserver is = new IntervalObserver(observer);
    observer.onSubscribe(is);
	// 线程调度器
    Scheduler sch = scheduler;

    if (sch instanceof TrampolineScheduler) {
        Worker worker = sch.createWorker();
        is.setResource(worker);
        worker.schedulePeriodically(is, initialDelay, period, unit);
    } else {
        // 将上面创建的观察者交给线程调度器去执行，并返回Disposable对象
        Disposable d = sch.schedulePeriodicallyDirect(is, initialDelay, period, unit);
        is.setResource(d);
    }
}
```

上面方法中首先创建了一个实现Disposable和Runnable的观察者，并将该观察者对象通过onSubscribe方法传递给了下游，方便下游取消，然后将该观察者交给线程调度器去执行，同时将返回值Disposable对象赋值给该观察者。

```java
// ObservableInterval.java
static final class IntervalObserver extends AtomicReference<Disposable> implements Disposable, Runnable {

    private static final long serialVersionUID = 346773832286157679L;

    final Observer<? super Long> downstream;

    long count;

    IntervalObserver(Observer<? super Long> downstream) {
        this.downstream = downstream;
    }

    @Override
    public void dispose() {
        // 取消自己
        DisposableHelper.dispose(this);
    }

    @Override
    public boolean isDisposed() {
        return get() == DisposableHelper.DISPOSED;
    }

    @Override
    public void run() {
        if (get() != DisposableHelper.DISPOSED) {
            // 往下游传递
            downstream.onNext(count++);
        }
    }

    public void setResource(Disposable d) {
        // 设置Disposable给自己
        DisposableHelper.setOnce(this, d);
    }
}
```

ObservableInterval继承了一个原子引用类，它保证线程读写安全，并实现了Disposable和Runnable接口，通过构造函数将下游观察者的事件传递给下游

当订阅开始时，将IntervalObserver传递给下游，并且它可以被下游取消，接着传递给调度器，执行调度器的run方法，将数据往下游传递，并返回一个Disposable对象，意味着可以随时取消调度器里面的该任务，然后又将该对象通过setResource方法设置给IntervalObserver自己，所以下游调用disposable方法时，会调用IntervalObserver的dispose，然后IntervalObserver内部随即调用自己的dispose方法，完成了取消。

#### Single.map

首先看下示例：

```java
Single<Integer> single = Single.just(1);
Single<String> singleString = single.map(new Function<Integer, String>() {
    @Override
    public String apply(Integer integer) throws Exception {
        return integer.toString();
    }
});
singleString.subscribe(new SingleObserver<String>() {
    @Override
    public void onSubscribe(Disposable d) {

    }

    @Override
    public void onSuccess(String s) {

    }

    @Override
    public void onError(Throwable e) {

    }
});
```

通过上面我们知道，上游创建了一个SingleJust对象，在调用map方法时，将自己传递给下游的SingleMap对象并返回。在调用subscribe方法时，其实是调用SingleMap的subscribeActual方法：

```java
// SingleMap.java
@Override
protected void subscribeActual(final SingleObserver<? super R> t) {
    source.subscribe(new MapSingleObserver<T, R>(t, mapper));
}
```

上面方法直接调用上游source去订阅MapSingleObserver这个观察者，然后在上游调用onSubscribe时调用下游的onSubscribe方法；在上游调用onSuccess时自己做了一下`mapper.apply(value)`转换操作，将数据转换成下游所需要的，然后再调用下游的onSuccess传递给下游；onError同onSubscribe原理是一样的。

#### Single.delay

首先来看下示例：

```java
Single<Integer> single = Single.just(1);
Single<Integer> singleDelay = single.delay(1, TimeUnit.SECONDS);
singleDelay.subscribe(new SingleObserver<Integer>() {
    @Override
    public void onSubscribe(Disposable d) {

    }

    @Override
    public void onSuccess(Integer integer) {

    }

    @Override
    public void onError(Throwable e) {

    }
});
```

上面single.delay方法返回SingleDelay对象，然后调用它的subscribe放，其实是调用subscribeActual方法：

```java
@Override
protected void subscribeActual(final SingleObserver<? super T> observer) {

    final SequentialDisposable sd = new SequentialDisposable();
    observer.onSubscribe(sd);
    source.subscribe(new Delay(sd, observer));
}
```

内部创建了一个Disposable对象，并将该对象通过onSubscribe方法传递给下游观察者，最后让上游订阅这个观察者，下面是SequentialDisposable代码：

```java
// SequentialDisposable.java
public final class SequentialDisposable
extends AtomicReference<Disposable>
implements Disposable {

    private static final long serialVersionUID = -754898800686245608L;

    public SequentialDisposable() {
        // nothing to do
    }

    public SequentialDisposable(Disposable initial) {
        lazySet(initial);
    }

    public boolean update(Disposable next) {
        return DisposableHelper.set(this, next);
    }

    public boolean replace(Disposable next) {
        return DisposableHelper.replace(this, next);
    }

    @Override
    public void dispose() {
        DisposableHelper.dispose(this);
    }

    @Override
    public boolean isDisposed() {
        return DisposableHelper.isDisposed(get());
    }
}
```

代码和上面IntervalObserver有点像，紧接着看下Delay类：

```java
// SingleDelay.java
final class Delay implements SingleObserver<T> {
        private final SequentialDisposable sd;
        final SingleObserver<? super T> downstream;

        Delay(SequentialDisposable sd, SingleObserver<? super T> observer) {
            this.sd = sd;
            this.downstream = observer;
        }

        @Override
        public void onSubscribe(Disposable d) {
            sd.replace(d);
        }

        @Override
        public void onSuccess(final T value) {
            sd.replace(scheduler.scheduleDirect(new OnSuccess(value), time, unit));
        }

        @Override
        public void onError(final Throwable e) {
            sd.replace(scheduler.scheduleDirect(new OnError(e), delayError ? time : 0, unit));
        }

        final class OnSuccess implements Runnable {
            private final T value;

            OnSuccess(T value) {
                this.value = value;
            }

            @Override
            public void run() {
                downstream.onSuccess(value);
            }
        }

        final class OnError implements Runnable {
            private final Throwable e;

            OnError(Throwable e) {
                this.e = e;
            }

            @Override
            public void run() {
                downstream.onError(e);
            }
        }
    }
```

subscribeActual方法最后在上游订阅Delay的时候，触发onSubscribe，Delay内部随即将该Disposable存入SequentialDisposable对象（需要注意的是下游拿到的Disposable始终是这个SequentialDisposable）中，此时如果下游调用dispose，也就是调用SequentialDisposable的dispose，也就是上游的dispose，dispose流程在这个节点上就完成了向上传递。

当上游产生数据，通过onSuccess方法传递给观察者Delay，在其onSuccess方法内部将调度器返回的Disposable替换SequentialDisposable内部，这样下游取消任务时，就直接把任务取消了，当调度器执行的OnSuccess的run方法， 下游就可以接收到数据了。

#### Observable.map

Observable.map所对应的是ObservableMap，直接看其subscribeActual方法：

```java
// ObservableMap.java
public final class ObservableMap<T, U> extends AbstractObservableWithUpstream<T, U> {
    final Function<? super T, ? extends U> function;

    public ObservableMap(ObservableSource<T> source, Function<? super T, ? extends U> function) {
        super(source);
        this.function = function;
    }

    @Override
    public void subscribeActual(Observer<? super U> t) {
        source.subscribe(new MapObserver<T, U>(t, function));
    }

    static final class MapObserver<T, U> extends BasicFuseableObserver<T, U> {
        final Function<? super T, ? extends U> mapper;

        MapObserver(Observer<? super U> actual, Function<? super T, ? extends U> mapper) {
            super(actual);
            this.mapper = mapper;
        }

        @Override
        public void onNext(T t) {
            if (done) {
                return;
            }

            if (sourceMode != NONE) {
                downstream.onNext(null);
                return;
            }

            U v;

            try {
                v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
            } catch (Throwable ex) {
                fail(ex);
                return;
            }
            downstream.onNext(v);
        }

        @Override
        public int requestFusion(int mode) {
            return transitiveBoundaryFusion(mode);
        }

        @Nullable
        @Override
        public U poll() throws Exception {
            T t = qd.poll();
            return t != null ? ObjectHelper.<U>requireNonNull(mapper.apply(t), "The mapper function returned a null value.") : null;
        }
    }
}
```

这里通过mapper.apply转换一下将数据传递给下游，同时在subscribeActual中并没有直接调用onSubscribe，MapObserver类中也没有，那应该是在父类中完成的：

```java
// BasicFuseableObserver.java
public final void onSubscribe(Disposable d) {
    if (DisposableHelper.validate(this.upstream, d)) {

        this.upstream = d;
        if (d instanceof QueueDisposable) {
            this.qd = (QueueDisposable<T>)d;
        }

        if (beforeDownstream()) {

            // 调用下游的onSubscribe方法
            downstream.onSubscribe(this);

            afterDownstream();
        }

    }
}
```

这里将上游的Disposable存储起来，并将中间节点自己传递给了下游，同时调用下游的onSubscribe

#### Observable.delay 

该方法返回的是ObservableDelay类，看其subscribeActual方法：

```java
// ObservableDelay.java
public final class ObservableDelay<T> extends AbstractObservableWithUpstream<T, T> {
    final long delay;
    final TimeUnit unit;
    final Scheduler scheduler;
    final boolean delayError;

    public ObservableDelay(ObservableSource<T> source, long delay, TimeUnit unit, Scheduler scheduler, boolean delayError) {
        super(source);
        this.delay = delay;
        this.unit = unit;
        this.scheduler = scheduler;
        this.delayError = delayError;
    }

    @Override
    @SuppressWarnings("unchecked")
    public void subscribeActual(Observer<? super T> t) {
        Observer<T> observer;
        if (delayError) {
            observer = (Observer<T>)t;
        } else {
            observer = new SerializedObserver<T>(t);
        }

        Scheduler.Worker w = scheduler.createWorker();

        source.subscribe(new DelayObserver<T>(observer, delay, unit, w, delayError));
    }

    static final class DelayObserver<T> implements Observer<T>, Disposable {
        final Observer<? super T> downstream;
        final long delay;
        final TimeUnit unit;
        final Scheduler.Worker w;
        final boolean delayError;

        Disposable upstream;

        DelayObserver(Observer<? super T> actual, long delay, TimeUnit unit, Worker w, boolean delayError) {
            super();
            this.downstream = actual;
            this.delay = delay;
            this.unit = unit;
            this.w = w;
            this.delayError = delayError;
        }

        @Override
        public void onSubscribe(Disposable d) {
            if (DisposableHelper.validate(this.upstream, d)) {
                this.upstream = d;
                // 调用下游的onSubscribe方法，并传递自己给下游
                downstream.onSubscribe(this);
            }
        }

        @Override
        public void onNext(final T t) {
            w.schedule(new OnNext(t), delay, unit);
        }

        @Override
        public void onError(final Throwable t) {
            w.schedule(new OnError(t), delayError ? delay : 0, unit);
        }

        @Override
        public void onComplete() {
            w.schedule(new OnComplete(), delay, unit);
        }

        @Override
        public void dispose() {
            upstream.dispose();
            w.dispose();
        }

        @Override
        public boolean isDisposed() {
            return w.isDisposed();
        }

        final class OnNext implements Runnable {
            private final T t;

            OnNext(T t) {
                this.t = t;
            }

            @Override
            public void run() {
                downstream.onNext(t);
            }
        }

        final class OnError implements Runnable {
            private final Throwable throwable;

            OnError(Throwable throwable) {
                this.throwable = throwable;
            }

            @Override
            public void run() {
                try {
                    downstream.onError(throwable);
                } finally {
                    w.dispose();
                }
            }
        }

        final class OnComplete implements Runnable {
            @Override
            public void run() {
                try {
                    downstream.onComplete();
                } finally {
                    w.dispose();
                }
            }
        }
    }
}
```

上面在subscribeActual没有调用下游的onSubscribe，而是在DelayObserver的onSubscribe方法完成调用，即先验证一下上游 然后将上游的Disposable赋值给upstream，调用下游的onSubscribe，把自己传给下游。当下游调用dispose时，在DelayObserver的dispose方法中将上游的Disposable给取消掉，然后把自己的调度器任务也给取消掉。当上游调用到DelayObserver的onNext时，OnNext任务（Runnable）提交给调度器执行，在执行任务时调用下游的onNext方法。

## 线程切换

### subscribeOn

该方法返回SingleSubscribeOn对象，然后看其subscribeActual方法：

```java
public final class SingleSubscribeOn<T> extends Single<T> {
    final SingleSource<? extends T> source;

    final Scheduler scheduler;

    public SingleSubscribeOn(SingleSource<? extends T> source, Scheduler scheduler) {
        this.source = source;
        this.scheduler = scheduler;
    }

    @Override
    protected void subscribeActual(final SingleObserver<? super T> observer) {
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer, source);
        observer.onSubscribe(parent);
		
        // 切换线程
        Disposable f = scheduler.scheduleDirect(parent);

        parent.task.replace(f);

    }

    static final class SubscribeOnObserver<T>
    extends AtomicReference<Disposable>
    implements SingleObserver<T>, Disposable, Runnable {

        private static final long serialVersionUID = 7000911171163930287L;

        final SingleObserver<? super T> downstream;

        final SequentialDisposable task;

        final SingleSource<? extends T> source;

        SubscribeOnObserver(SingleObserver<? super T> actual, SingleSource<? extends T> source) {
            this.downstream = actual;
            this.source = source;
            this.task = new SequentialDisposable();
        }

        @Override
        public void onSubscribe(Disposable d) {
            DisposableHelper.setOnce(this, d);
        }

        @Override
        public void onSuccess(T value) {
            downstream.onSuccess(value);
        }

        @Override
        public void onError(Throwable e) {
            downstream.onError(e);
        }

        @Override
        public void dispose() {
            DisposableHelper.dispose(this);
            task.dispose();
        }

        @Override
        public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }

        @Override
        public void run() {
            source.subscribe(this);
        }
    }
}
```

上面代码将parent交给线程调度器去执行，看下它的Run方法，在scheduleDirect那里切了线程，然后在另一个线程中去执行`source.subscribe(this)`，即这里在Scheduler指定的线程里启动了subscribe（订阅），所以多次执行subscribeOn，只会对最开始的Observable起作用。

### observeOn

该方法返回SingleObserveOn对象，直接看其subscribeActual方法：

```java
// SingleObserveOn.java
public final class SingleObserveOn<T> extends Single<T> {

    final SingleSource<T> source;

    final Scheduler scheduler;

    public SingleObserveOn(SingleSource<T> source, Scheduler scheduler) {
        this.source = source;
        this.scheduler = scheduler;
    }

    @Override
    protected void subscribeActual(final SingleObserver<? super T> observer) {
        source.subscribe(new ObserveOnSingleObserver<T>(observer, scheduler));
    }

    static final class ObserveOnSingleObserver<T> extends AtomicReference<Disposable>
    implements SingleObserver<T>, Disposable, Runnable {
        private static final long serialVersionUID = 3528003840217436037L;

        final SingleObserver<? super T> downstream;

        final Scheduler scheduler;

        T value;
        Throwable error;

        ObserveOnSingleObserver(SingleObserver<? super T> actual, Scheduler scheduler) {
            this.downstream = actual;
            this.scheduler = scheduler;
        }

        @Override
        public void onSubscribe(Disposable d) {
            if (DisposableHelper.setOnce(this, d)) {
                downstream.onSubscribe(this);
            }
        }

        @Override
        public void onSuccess(T value) {
            this.value = value;
            Disposable d = scheduler.scheduleDirect(this);
            DisposableHelper.replace(this, d);
        }

        @Override
        public void onError(Throwable e) {
            this.error = e;
            Disposable d = scheduler.scheduleDirect(this);
            DisposableHelper.replace(this, d);
        }

        @Override
        public void run() {
            Throwable ex = error;
            if (ex != null) {
                downstream.onError(ex);
            } else {
                downstream.onSuccess(value);
            }
        }

        @Override
        public void dispose() {
            DisposableHelper.dispose(this);
        }

        @Override
        public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }
    }
}
```

可以看到，上游订阅了ObserveOnSingleObserver这个观察者，当下游调用onSuccess方法时，会执行构造函数传进来的scheduler.scheduleDirect方法，即调用ObserveOnSingleObserver的run函数，所以该函数会运行在scheduler指定的线程中，即下游的onSuccess和onError方法都会在Scheduler指定的线程中运行。所以每次调用observeOn，都会进行一次线程切换，下游的每个Observer都会运行在指定的切换的线程中。

### Scheduler调度器

Schedulers.newThread()里面是创建了一个线程池`Executors.newScheduledThreadPool(1, factory)`来执行任务，但是这个线程池里面的线程不会得到重用，每次都是新建的线程池。当 scheduleDirect() 被调用的时候，会创建一个 Worker，Worker 的内部 会有一个 Executor，由 Executor 来完成实际的线程切换；同时该方法也会返回一个Disposable对象，交给外层的Observer，从而执行dispose方法，取消订阅链。

```java
// NewThreadScheduler.java

@NonNull
@Override
public Worker createWorker() {
    return new NewThreadWorker(threadFactory);
}

// NewThreadWorker.java

public NewThreadWorker(ThreadFactory threadFactory) {
    executor = SchedulerPoolFactory.create(threadFactory);
}

// SchedulerPoolFactory.java
public static ScheduledExecutorService create(ThreadFactory factory) {
    final ScheduledExecutorService exec = Executors.newScheduledThreadPool(1, factory);
    tryPutIntoPool(PURGE_ENABLED, exec);
    return exec;
}
```

Schedulers.io()方法和上面区别不大，它的线程可以被重用，所以Android中我们常用它来做线程切换。

```java
// IoScheduler.java
public final class IoScheduler extends Scheduler {
    ...
    @NonNull
    @Override
    public Worker createWorker() {
        return new EventLoopWorker(pool.get());
    }
    ...
}


static final class EventLoopWorker extends Scheduler.Worker {
    private final CompositeDisposable tasks;
    private final CachedWorkerPool pool;
    private final ThreadWorker threadWorker;

    final AtomicBoolean once = new AtomicBoolean();

    EventLoopWorker(CachedWorkerPool pool) {
        this.pool = pool;
        this.tasks = new CompositeDisposable();
        // 从缓存池取
        this.threadWorker = pool.get();
    }

    @Override
    public void dispose() {
        if (once.compareAndSet(false, true)) {
            tasks.dispose();

            // releasing the pool should be the last action
            pool.release(threadWorker);
        }
    }

    @Override
    public boolean isDisposed() {
        return once.get();
    }

    @NonNull
    @Override
    public Disposable schedule(@NonNull Runnable action, long delayTime, @NonNull TimeUnit unit) {
        if (tasks.isDisposed()) {
            // don't schedule, we are unsubscribed
            return EmptyDisposable.INSTANCE;
        }

        return threadWorker.scheduleActual(action, delayTime, unit, tasks);
    }
}

static final class ThreadWorker extends NewThreadWorker {
    private long expirationTime;

    ThreadWorker(ThreadFactory threadFactory) {
        super(threadFactory);
        this.expirationTime = 0L;
    }

    public long getExpirationTime() {
        return expirationTime;
    }

    public void setExpirationTime(long expirationTime) {
        this.expirationTime = expirationTime;
    }
}
```

而AndroidSchedulers.mainThread()内部直接是使用Handler进行线程切换，将任务放到主线程去做

```java
// AndroidSchedulers.java
public final class AndroidSchedulers {
	...
    private static final class MainHolder {
        static final Scheduler DEFAULT
            = new HandlerScheduler(new Handler(Looper.getMainLooper()), false);
    }
    ...
}
```

## 总结

以上从源码角度分析了RxJava3中的订阅流程，取消订阅流程和线程切换流程，这样可以更好的理解RxJava工作机制。

