---
layout: post
title: Android三方开源库之OkHttp源码分析
tags: 源码分析
categories: Android
date: 2020-07-14
---

OkHttp 是目前应用最广泛的开源网络库了，而且在Android6.0之后也将内部的HttpUrlConnection替换成了OkHttp，这篇文章来分析一下源码。

## 基本使用

```java
OkHttpClient client = new OkHttpClient();
Request request = new Request.Builder()
    .url("https://github.com")
    .build();
// 同步请求
try {
    okhttp3.Response response = client.newCall(request).execute();
} catch (IOException e) {
    e.printStackTrace();
}

// 异步请求
client.newCall(request).enqueue(new okhttp3.Callback() {
    @Override
    public void onFailure(@NonNull okhttp3.Call call, @NonNull IOException e) {

    }

    @Override
    public void onResponse(@NonNull okhttp3.Call call, @NonNull okhttp3.Response response) throws IOException {

    }
});

```

代码中涉及到几个对象，来一起看下：

OkHttpClient：OkHttp的核心配置类，采用建造者模式，多个方便配置的参数。

Request：请求参数配置类，也采用建造者模式，可以配置的参数有请求URL`、`请求方法`、`请求头`、`请求体

Response: 返回结果类，包含Code、Message、header、body等

Call：请求接口，表示一个已经准备的好的请求，可以执行或取消。

RealCall：通过OkHttpClient的newCall方法返回的一个Call对象，它是上面Call接口的实现类，负责请求的调度，内部通过同步或者异步请求，构造内部逻辑责任链，并执行责任链相关的逻辑，直到获取结果

## 源码分析

### 请求

接下来我们就先从RealCall请求看起：

RealCall有两个最重要的方法，execute() 和 enqueue()，一个是处理同步请求，一个是处理异步请求。先看下同步请求：

```kotlin
// RealCall.kt
override fun execute(): Response {
    check(executed.compareAndSet(false, true)) { "Already Executed" }

    timeout.enter()
    // 请求监听
    callStart()
    try {
        client.dispatcher.executed(this)
        return getResponseWithInterceptorChain()
    } finally {
        client.dispatcher.finished(this)
    }
}
```

上面首先采用CAS检查当前线程是否被执行过，如果没有执行过就将标识首先置为true，然后开始执行，紧接着请求超时倒计时开始计时，然后监听请求，最后执行调度器中的executed方法，将当前RealCall对象加入runningSyncCalls队列，然后调用`getResponseWithInterceptorChain`方法拿到`response`。

接着看下异步请求：

```kotlin
// RealCall.kt
override fun enqueue(responseCallback: Callback) {
    check(executed.compareAndSet(false, true)) { "Already Executed" }

    callStart()
    client.dispatcher.enqueue(AsyncCall(responseCallback))
}
```

上面首先也是采用CAS检查当前线程是否被执行过，然后监听请求，最后创建一个AsyncCall对象，然后通过调度器的 enqueue方法将其加入到readyAsyncCalls队列中。

进阶着看下调度器的enqueue方法：

```kotlin
// Dispatcher.kt

internal fun enqueue(call: AsyncCall) {
    // 加锁保证线程安全
    synchronized(this) {
        // 加入队列
        readyAsyncCalls.add(call)

        // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to
        // the same host.
        if (!call.call.forWebSocket) {
            // 查看 有没有相同的域名请求，如果有就可以复用
            val existingCall = findExistingCallWithHost(call.host)
            if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)
        }
    }
    // 执行请求
    promoteAndExecute()
}

private fun promoteAndExecute(): Boolean {
    this.assertThreadDoesntHoldLock()

    val executableCalls = mutableListOf<AsyncCall>()
    val isRunning: Boolean
    synchronized(this) {
        // 遍历准备请求队列
        val i = readyAsyncCalls.iterator()
        while (i.hasNext()) {
            val asyncCall = i.next()
			//runningAsyncCalls 的数量不能大于最大并发请求数 64
            if (runningAsyncCalls.size >= this.maxRequests) break // Max capacity.
            // 同一个域名最大请求数为5
            if (asyncCall.callsPerHost.get() >= this.maxRequestsPerHost) continue // Host max capacity.
			// 从请求队列移除，并加入到​executableCalls执行队列和​runningAsyncCalls运行队列
            i.remove()
            asyncCall.callsPerHost.incrementAndGet()
            executableCalls.add(asyncCall)
            runningAsyncCalls.add(asyncCall)
        }
        // 通过判断运行队列的请求数量来判断是否有请求正在执行
        isRunning = runningCallsCount() > 0
    }

    // 遍历可执行队列，调用线程池来执行AsyncCall
    for (i in 0 until executableCalls.size) {
        val asyncCall = executableCalls[i]
        asyncCall.executeOn(executorService)
    }

    return isRunning
}
```

上面代码中，调度器的enqueue方法将AsyncCall加入到readyAsyncCalls准备请求队列，然后调用promoteAndExecute方法开始执行请求，遍历上面的准确请求队列，将符合条件的AsyncCall通过其executeOn方法交给线程池来执行。即执行AsyncCall的run方法。AsyncCall是RealCall的内部类，看下它的run方法：

```kotlin
// RealCall.kt
inner class AsyncCall(
    private val responseCallback: Callback
) : Runnable {
    fun executeOn(executorService: ExecutorService) {
        client.dispatcher.assertThreadDoesntHoldLock()

        var success = false
        try {
            // 交给线程池执行
            executorService.execute(this)
            success = true
        } catch (e: RejectedExecutionException) {
            val ioException = InterruptedIOException("executor rejected")
            ioException.initCause(e)
            noMoreExchanges(ioException)
            responseCallback.onFailure(this@RealCall, ioException)
        } finally {
            if (!success) {
                client.dispatcher.finished(this) // This call is no longer running!
            }
        }
    }

    override fun run() {
        threadName("OkHttp ${redactedUrl()}") {
            var signalledCallback = false
            // 请求超时倒计时开始
            timeout.enter()
            try {
                // 通过责任链获取返回结果
                val response = getResponseWithInterceptorChain()
                signalledCallback = true
                responseCallback.onResponse(this@RealCall, response)
            } catch (e: IOException) {
                if (signalledCallback) {
                    // Do not signal the callback twice!
                    Platform.get().log("Callback failure for ${toLoggableString()}", Platform.INFO, e)
                } else {
                    responseCallback.onFailure(this@RealCall, e)
                }
            } catch (t: Throwable) {
                cancel()
                if (!signalledCallback) {
                    val canceledException = IOException("canceled due to $t")
                    canceledException.addSuppressed(t)
                    responseCallback.onFailure(this@RealCall, canceledException)
                }
                throw t
            } finally {
                client.dispatcher.finished(this)
            }
        }
    }
}
```

可以看到上面run方法，就是调用`getResponseWithInterceptorChain`方法拿到`response`，然后通过`Callback.onResponse`方法传递出去。反之，如果请求失败，捕获了异常，就通过`Callback.onFailure`将异常信息传递出去。 最终，请求结束，调用调度器`finish`方法。

```kotlin
// Dispatcher.kt
internal fun finished(call: AsyncCall) {
    call.callsPerHost.decrementAndGet()
    finished(runningAsyncCalls, call)
}

/** Used by [Call.execute] to signal completion. */
internal fun finished(call: RealCall) {
    finished(runningSyncCalls, call)
}

private fun <T> finished(calls: Deque<T>, call: T) {
    val idleCallback: Runnable?
    synchronized(this) {
        // 从运行队列中移除当前请求
        if (!calls.remove(call)) throw AssertionError("Call wasn't in-flight!")
        idleCallback = this.idleCallback
    }
    
    //继续执行剩余请求，将call从readyAsyncCalls中取出加入到runningAsyncCalls，然后执行
    val isRunning = promoteAndExecute()

    if (!isRunning && idleCallback != null) {
        //如果执行完了所有请求，处于闲置状态，调用闲置回调方法
        idleCallback.run()
    }
}
```

上面finish方法中，首先将当前请求从运行队列中移除，然后继续执行剩余请求，最后如果所有请求都执行完了，就调用闲置回调方法。

### 响应

其实是响应获取结果主要是看getResponseWithInterceptorChain方法是如何返回结果的：

```kotlin
// RealCall.kt 
@Throws(IOException::class)
internal fun getResponseWithInterceptorChain(): Response {
    // Build a full stack of interceptors.
    // 拦截器列表
    val interceptors = mutableListOf<Interceptor>()
    interceptors += client.interceptors
    interceptors += RetryAndFollowUpInterceptor(client)
    interceptors += BridgeInterceptor(client.cookieJar)
    interceptors += CacheInterceptor(client.cache)
    interceptors += ConnectInterceptor
    if (!forWebSocket) {
        interceptors += client.networkInterceptors
    }
    interceptors += CallServerInterceptor(forWebSocket)

    // 创建拦截器责任链
    val chain = RealInterceptorChain(
        call = this,
        interceptors = interceptors,
        index = 0,
        exchange = null,
        request = originalRequest,
        connectTimeoutMillis = client.connectTimeoutMillis,
        readTimeoutMillis = client.readTimeoutMillis,
        writeTimeoutMillis = client.writeTimeoutMillis
    )

    var calledNoMoreExchanges = false
    try {
        // 执行拦截器责任链获取结果
        val response = chain.proceed(originalRequest)
        if (isCanceled()) {
            response.closeQuietly()
            throw IOException("Canceled")
        }
        return response
    } catch (e: IOException) {
        calledNoMoreExchanges = true
        throw noMoreExchanges(e) as Throwable
    } finally {
        if (!calledNoMoreExchanges) {
            noMoreExchanges(null)
        }
    }
}
```

从上面代码看出，这里采用责任链设计模式，通过拦截器构建了一个RealInterceptorChain责任链，然后通过执行其proceed方法来获得结果。

#### 拦截器

上面代码中，可以看出这个责任链是由一系列的拦截器组成：

- client.Interceptors：开发者设置的拦截器，会在所有拦截器处理之前处理

- RetryAndFollowUpInterceptor：失败重试和重定向拦截器

- BridgeInterceptor：负责处理Request和Response的拦截器

- CacheInterceptor：负责缓存处理的拦截器

- ConnectInterceptor：负责和服务器建立连接服务的拦截器

- client.networkInterceptors：开发者自己设置的拦截器

- CallServerInterceptor：进行数据请求和响应的拦截器

先看下拦截器的定义：

```kotlin
// Interceptor.kt
fun interface Interceptor {
  @Throws(IOException::class)
  fun intercept(chain: Chain): Response

  companion object {
    /**
     * Constructs an interceptor for a lambda. This compact syntax is most useful for inline
     * interceptors.
     *
     * ```kotlin
     * val interceptor = Interceptor { chain: Interceptor.Chain ->
     *     chain.proceed(chain.request())
     * }
     * ```
     */
    inline operator fun invoke(crossinline block: (chain: Chain) -> Response): Interceptor =
      Interceptor { block(it) }
  }

  interface Chain {
    fun request(): Request

    @Throws(IOException::class)
    fun proceed(request: Request): Response

    /**
     * Returns the connection the request will be executed on. This is only available in the chains
     * of network interceptors; for application interceptors this is always null.
     */
    fun connection(): Connection?

    fun call(): Call

    fun connectTimeoutMillis(): Int

    fun withConnectTimeout(timeout: Int, unit: TimeUnit): Chain

    fun readTimeoutMillis(): Int

    fun withReadTimeout(timeout: Int, unit: TimeUnit): Chain

    fun writeTimeoutMillis(): Int

    fun withWriteTimeout(timeout: Int, unit: TimeUnit): Chain
  }
}
```

上面声明了一个SAM函数接口，内部有一个Chain接口，其核心方法是proceed，通过它获取结果，紧接着看下该接口的实现类RealInterceptorChain：

```kotlin
// RealInterceptorChain.kt
@Throws(IOException::class)
override fun proceed(request: Request): Response {
    check(index < interceptors.size)

    calls++

    if (exchange != null) {
        check(exchange.finder.routePlanner.sameHostAndPort(request.url)) {
            "network interceptor ${interceptors[index - 1]} must retain the same host and port"
        }
        check(calls == 1) {
            "network interceptor ${interceptors[index - 1]} must call proceed() exactly once"
        }
    }

    // Call the next interceptor in the chain.
    // 创建新的责任链，调用责任链中下一个拦截器
    val next = copy(index = index + 1, request = request)
    val interceptor = interceptors[index]

    // 执行拦截器中的拦截方法
    @Suppress("USELESS_ELVIS")
    val response = interceptor.intercept(next) ?: throw NullPointerException(
        "interceptor $interceptor returned null")

    if (exchange != null) {
        check(index + 1 >= interceptors.size || next.calls == 1) {
            "network interceptor $interceptor must call proceed() exactly once"
        }
    }

    return response
}
```

上面代码中会按照拦截器列表的中顺序执行，返回Response。

下面介绍下上面的各个拦截器：

##### client.Interceptors

这是用户自己定义的拦截器，会被添加到interceptors列表，这是第一个被执行的拦截器，一般就是通过实现Interceptor的intercept方法，先处理自己的逻辑，比如设置Header等信息，然后通过chain.proceed(request)返回Response

##### RetryAndFollowUpInterceptor

它负责请求失败重试与重定向的后续请求工作：

```kotlin
@Throws(IOException::class)
override fun intercept(chain: Interceptor.Chain): Response {
    val realChain = chain as RealInterceptorChain
    var request = chain.request
    val call = realChain.call
    var followUpCount = 0
    var priorResponse: Response? = null
    var newRoutePlanner = true
    var recoveredFailures = listOf<IOException>()
    while (true) {
        call.enterNetworkInterceptorExchange(request, newRoutePlanner, chain)

        var response: Response
        var closeActiveExchange = true
        try {
            if (call.isCanceled()) {
                throw IOException("Canceled")
            }

            try {
                response = realChain.proceed(request)
                newRoutePlanner = true
            } catch (e: IOException) {
                // An attempt to communicate with a server failed. The request may have been sent.		// 判断是否请求重试
                if (!recover(e, call, request, requestSendStarted = e !is ConnectionShutdownException)) {
                    throw e.withSuppressed(recoveredFailures)
                } else {
                    recoveredFailures += e
                }
                newRoutePlanner = false
                continue
            }

            // Clear out downstream interceptor's additional request headers, cookies, etc.
            response = response.newBuilder()
            .request(request)
            .priorResponse(priorResponse?.stripBody())
            .build()

            val exchange = call.interceptorScopedExchange
            // 根据状态码来判断重试或者重定向
            val followUp = followUpRequest(response, exchange)

            if (followUp == null) {
                if (exchange != null && exchange.isDuplex) {
                    call.timeoutEarlyExit()
                }
                closeActiveExchange = false
                return response
            }

            val followUpBody = followUp.body
            if (followUpBody != null && followUpBody.isOneShot()) {
                closeActiveExchange = false
                return response
            }

            response.body.closeQuietly()

            if (++followUpCount > MAX_FOLLOW_UPS) {
                throw ProtocolException("Too many follow-up requests: $followUpCount")
            }

            request = followUp
            priorResponse = response
        } finally {
            call.exitNetworkInterceptorExchange(closeActiveExchange)
        }
    }
}

private fun recover(
    e: IOException,
    call: RealCall,
    userRequest: Request,
    requestSendStarted: Boolean
): Boolean {
    // The application layer has forbidden retries.
    if (!client.retryOnConnectionFailure) return false

    // We can't send the request body again.
    if (requestSendStarted && requestIsOneShot(e, userRequest)) return false

    // This exception is fatal.
    if (!isRecoverable(e, requestSendStarted)) return false

    // No more routes to attempt.
    if (!call.retryAfterFailure()) return false

    // For failure recovery, use the same route selector with a new connection.
    return true
}
```

上面代码中，当请求内部发生异常时，会判断是否请求重试，判定逻辑在recover方法中：

- client的retryOnConnectionFailure参数设置为false，不进行重试
- 请求的body已经发出，不进行重试
- 特殊的异常类型不进行重试（如ProtocolException，SSLHandshakeException、SSLPeerUnverifiedException等）
- 没有更多的routes（包含proxy和inetaddress），不进行重试

##### BridgeInterceptor

它负责将用户请求转换为服务器需要的请求，比如设置内容长度，编码、gzip压缩、添加cookie以及其他header，同时将服务器返回的结果进行转换为用户需要的结果，是从应用程序到服务器的桥梁

##### CacheInterceptor

它根据OkHttpClient配置的缓存，首先通过Request尝试获取缓存，然后通过工厂模式CacheStrategy.Factory生成的缓存策略，来判断如何使用缓存

- 如果缓存策略设置网络不可用，并且缓存也没有，直接构造一个Response返回
- 如果缓存策略设置网络不可用，但是有缓存，就返回缓存
- 执行后续流程
- 当缓存存在，如果返回304，就使用缓存的Response
- 否则就构建网络请求的Response，如果OkHttpClient配置的有缓存，就把该Response缓存起来
- 最后返回Response

##### ConnectInterceptor

该拦截器负责和服务器建立链接，先初始化一个exchange对象，根据这个对象赋值一个新的连接责任链，最后执行该连接责任链

```kotlin
// ConnectInterceptor.kt
object ConnectInterceptor : Interceptor {
  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    val realChain = chain as RealInterceptorChain
    val exchange = realChain.call.initExchange(realChain)
    val connectedChain = realChain.copy(exchange = exchange)
    return connectedChain.proceed(realChain.request)
  }
}

// RealCall.kt
internal fun initExchange(chain: RealInterceptorChain): Exchange {
    synchronized(this) {
        check(expectMoreExchanges) { "released" }
        check(!responseBodyOpen)
        check(!requestBodyOpen)
    }

    val exchangeFinder = this.exchangeFinder!!
    val connection = exchangeFinder.find()
    val codec = connection.newCodec(client, chain)
    val result = Exchange(this, eventListener, exchangeFinder, codec)
    this.interceptorScopedExchange = result
    this.exchange = result
    synchronized(this) {
        this.requestBodyOpen = true
        this.responseBodyOpen = true
    }

    if (canceled) throw IOException("Canceled")
    return result
}
```

上面代码中初始化Exchange的过程，首先通过exchangeFinder的find方法返回一个RealConnection对象，然后根据这个对象的newCodeC方法，获得ExchangeCodec对象，最后构造出一个ExChange对象，可见这个对象中包含了一个RealConnection对象，它是Socket的包装类，也就是说最终获得的是一个建立连接的socket对象。

#####  client.networkInterceptors

这是另一个自定义的网络拦截器networkInterceptors，按照顺序执行的规定，这时候连接已经建立了，可以获得数据了，因此可以利用它做一些网络调试，它和第一个自定义拦截器不同的地方在于，它们所处的拦截器位置不一样，第一个应用拦截器有且只能执行一次，而这个网络拦截器它可能执行0次（直接返回缓存）或者多次（失败重试）

##### CallServerInterceptor

它是最后一个拦截器了，前面的拦截器已经完成了socket连接和tls连接，那么这一步就是请求头与请求体发送给服务器以及解析服务器返回的数据。

- 首先向服务器发送header，如果有body也一并发送
- 解析Response header来构造Response对象，如果有body，就在前面的Response对象基础上添加上body
- 由于是最后一个拦截器，他不会再调用chain.proceed方法，而是将得到的Response返回给前面每一个拦截器

## 总结

最后用一张图，我们总结下整个工作流程：

![image-20220904184914258](https://tva1.sinaimg.cn/large/e6c9d24ely1h5urmpj7scj210e0n0n0h.jpg)

