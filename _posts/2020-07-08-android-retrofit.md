---
layout: post
title: Android三方开源库之Retrofit源码分析
tags: 源码分析
categories: Android
date: 2020-07-08
---

Retrofit是目前Android开发中主流的网络请求客户端，其底层基于OkHttp封装，提供了方便高效的网络请求框架。

## 基本用法

```java
Retrofit retrofit = new Retrofit.Builder()
                .baseUrl("https://api.github.com")
                .addConverterFactory(GsonConverterFactory.create())
                .build();

GitHubApiService service = retrofit.create(GitHubApiService.class);
Call<List<String>> call = service.listRepo();

call.enqueue(new Callback<List<String>>() {
    @Override
    public void onResponse(Call<List<String>> call, Response<List<String>> response) {

    }

    @Override
    public void onFailure(Call<List<String>> call, Throwable t) {

    }
});

public interface GitHubAPIService {

    @GET("/users/{user}/repos")
    Call<List<String>> listRepo(@Path("user") String user);
}
```

上面代码中，说下具体的每个类：

- Retrofit，全局配置类，通过内部类Builder来构建
- GitHubApiService是用户自己创建的接口，通过Retrofit的create方法实例化
- Call是执行网络请求的一个顶层接口，具体在源码中真正执行的是OkHttpCall
- Callback是请求结果的回调

## 源码分析

### OkHttp的封装

首先通过call.enqueue方法来看：

```java
// Call.java
public interface Call<T> extends Cloneable {
    ...
    void enqueue(Callback<T> callback);
	...
}
```

这是一个接口，我们自定义的接口返回的Call对象，所以说重点是看喜爱Retrofit的create方法：

```java
// Retrofit.java
public <T> T create(final Class<T> service) {
    validateServiceInterface(service);
    return (T)
        Proxy.newProxyInstance(
        service.getClassLoader(),
        new Class<?>[] {service},
        new InvocationHandler() {
            private final Platform platform = Platform.get();
            private final Object[] emptyArgs = new Object[0];

            @Override
            public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
                throws Throwable {
                // If the method is a method from Object then defer to normal invocation.
                if (method.getDeclaringClass() == Object.class) {
                    return method.invoke(this, args);
                }
                args = args != null ? args : emptyArgs;
                return platform.isDefaultMethod(method)
                    ? platform.invokeDefaultMethod(method, service, proxy, args)
                    : loadServiceMethod(method).invoke(args);
            }
        });
}
```

这里面主要是用到了Java的动态代理，在运行期，动态的创建我们自定义接口的实现类，作为代理对象，当调用接口方法时，会执行InvocationHandler 的 `invoke` 方法。

紧接着看invoke方法代码：

如果要执行的方法是来自于Object，那么就Object调用自己的方法，否则就会根据平台（Java8）执行默认的方法，因为是在Android平台，会走到loadServiceMethod(method).invoke(args)这里，紧接着看下这个方法：

```java
// Retrofit.java

ServiceMethod<?> loadServiceMethod(Method method) {
    ServiceMethod<?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
        result = serviceMethodCache.get(method);
        if (result == null) {
            result = ServiceMethod.parseAnnotations(this, method);
            serviceMethodCache.put(method, result);
        }
    }
    return result;
}
```

该方法通过ServiceMethod.parseAnnotations(this, method)创建一个ServiceMethod对象，并缓存起来，当下次调用的时候，会先从缓存取。

```java
// ServiceMethod.java
static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);

    Type returnType = method.getGenericReturnType();
    if (Utils.hasUnresolvableType(returnType)) {
        throw methodError(
            method,
            "Method return type must not include a type variable or wildcard: %s",
            returnType);
    }
    if (returnType == void.class) {
        throw methodError(method, "Service methods cannot return void.");
    }

    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
}
```

上面ServiceMethod的方法返回一个HttpServiceMethod的泛型对象，紧接着看下该对象的invoke方法：

```java
// HttpServiceMethod.java
@Override
final @Nullable ReturnT invoke(Object[] args) {
    Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
    return adapt(call, args);
}
```

从上面代码也可以看出来，内部利用OkHttpCall创建了一个Call对象，其中adapt方法是一个抽象方法，我们通过HttpServiceMethod.parseAnnotations方法可以看到：

```java
// HttpServiceMethod.java
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
      Retrofit retrofit, Method method, RequestFactory requestFactory) {
    boolean isKotlinSuspendFunction = requestFactory.isKotlinSuspendFunction;
    boolean continuationWantsResponse = false;
    boolean continuationBodyNullable = false;

    Annotation[] annotations = method.getAnnotations();
    Type adapterType;
    if (isKotlinSuspendFunction) {
      Type[] parameterTypes = method.getGenericParameterTypes();
      Type responseType =
          Utils.getParameterLowerBound(
              0, (ParameterizedType) parameterTypes[parameterTypes.length - 1]);
      if (getRawType(responseType) == Response.class && responseType instanceof ParameterizedType) {
        // Unwrap the actual body type from Response<T>.
        responseType = Utils.getParameterUpperBound(0, (ParameterizedType) responseType);
        continuationWantsResponse = true;
      } else {
        // TODO figure out if type is nullable or not
        // Metadata metadata = method.getDeclaringClass().getAnnotation(Metadata.class)
        // Find the entry for method
        // Determine if return type is nullable or not
      }

      adapterType = new Utils.ParameterizedTypeImpl(null, Call.class, responseType);
      annotations = SkipCallbackExecutorImpl.ensurePresent(annotations);
    } else {
      adapterType = method.getGenericReturnType();
    }

    // 创建CallAdapter对象
    CallAdapter<ResponseT, ReturnT> callAdapter =
        createCallAdapter(retrofit, method, adapterType, annotations);
    Type responseType = callAdapter.responseType();
    
    ...

    Converter<ResponseBody, ResponseT> responseConverter =
        createResponseConverter(retrofit, method, responseType);

    okhttp3.Call.Factory callFactory = retrofit.callFactory;
    if (!isKotlinSuspendFunction) {
        
        // 如果不是kotlin挂起函数，返回CallAdapted
      return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
    } else if (continuationWantsResponse) {
      //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
      return (HttpServiceMethod<ResponseT, ReturnT>)
          new SuspendForResponse<>(
              requestFactory,
              callFactory,
              responseConverter,
              (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter);
    } else {
      //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
      return (HttpServiceMethod<ResponseT, ReturnT>)
          new SuspendForBody<>(
              requestFactory,
              callFactory,
              responseConverter,
              (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter,
              continuationBodyNullable);
    }
  }
```

从方法返回值可以看出，如果不是kotlin挂起函数，这里返回的其实CallAdapted对象，

```java
static final class CallAdapted<ResponseT, ReturnT> extends HttpServiceMethod<ResponseT, ReturnT> {
    private final CallAdapter<ResponseT, ReturnT> callAdapter;

    CallAdapted(
        RequestFactory requestFactory,
        okhttp3.Call.Factory callFactory,
        Converter<ResponseBody, ResponseT> responseConverter,
        CallAdapter<ResponseT, ReturnT> callAdapter) {
      super(requestFactory, callFactory, responseConverter);
      this.callAdapter = callAdapter;
    }

    @Override
    protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
      return callAdapter.adapt(call);
    }
  }
```

从上面可以看出，该对象继承自HttpServiceMethod，所以先是调用其父类的invoke方法，在该方法里面调用它自身（CallAdapted）的adapt方法，进而 调用CallAdapter的adapt方法，而CallAdapter其实是在上面HttpServiceMethod.parseAnnotations这个方法里通过下面一段代码创建出来的对象：

```
CallAdapter<ResponseT, ReturnT> callAdapter =
    createCallAdapter(retrofit, method, adapterType, annotations);
```

我们接着看下该方法：

```java
// HttpServiceMethod.java
private static <ResponseT, ReturnT> CallAdapter<ResponseT, ReturnT> createCallAdapter(
    Retrofit retrofit, Method method, Type returnType, Annotation[] annotations) {
    try {
        //noinspection unchecked
        return (CallAdapter<ResponseT, ReturnT>) retrofit.callAdapter(returnType, annotations);
    } catch (RuntimeException e) { // Wide exception range because factories are user code.
        throw methodError(method, e, "Unable to create call adapter for %s", returnType);
    }
}
```

该方法会调用Retrofit的callAdapter方法:

```java
// Retrofit.java
public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
}

public CallAdapter<?, ?> nextCallAdapter(
    @Nullable CallAdapter.Factory skipPast, Type returnType, Annotation[] annotations) {
    ...

    int start = callAdapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
        CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
        if (adapter != null) {
            return adapter;
        }
    }
    ...
}

```

经过一系列调用发现，callAdapter对象是遍历callAdapterFactories这个集合，然后调用里面对象的get方法，根据返回值得到CallAdapter，如果不为空，就直接返回该CallAdapter对象，我们看下这个这个列表是从什么时候添加进去值的：

```java
public Retrofit build() {
    ...

    okhttp3.Call.Factory callFactory = this.callFactory;
    if (callFactory == null) {
        callFactory = new OkHttpClient();
    }

    Executor callbackExecutor = this.callbackExecutor;
    if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
    }

    // Make a defensive copy of the adapters and add the default Call adapter.
    List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
    callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));
    
    ...

    return new Retrofit(
        callFactory,
        baseUrl,
        unmodifiableList(converterFactories),
        unmodifiableList(callAdapterFactories),
        callbackExecutor,
        validateEagerly);
}

```

我们看到通过Retrofit的build方法，在`callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));`处添加了一个默认的CallAdpaterFactory：

```java
// Platform.java
  List<? extends CallAdapter.Factory> defaultCallAdapterFactories(
      @Nullable Executor callbackExecutor) {
      // 上面的CallAdapter
    DefaultCallAdapterFactory executorFactory = new DefaultCallAdapterFactory(callbackExecutor);
    return hasJava8Types
        ? asList(CompletableFutureCallAdapterFactory.INSTANCE, executorFactory)
        : singletonList(executorFactory);
  }

```

我们看下它的get方法：

```java
final class DefaultCallAdapterFactory extends CallAdapter.Factory {
  private final @Nullable Executor callbackExecutor;

  DefaultCallAdapterFactory(@Nullable Executor callbackExecutor) {
    this.callbackExecutor = callbackExecutor;
  }

  @Override
  public @Nullable CallAdapter<?, ?> get(
      Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    if (!(returnType instanceof ParameterizedType)) {
      throw new IllegalArgumentException(
          "Call return type must be parameterized as Call<Foo> or Call<? extends Foo>");
    }
    final Type responseType = Utils.getParameterUpperBound(0, (ParameterizedType) returnType);

    final Executor executor =
        Utils.isAnnotationPresent(annotations, SkipCallbackExecutor.class)
            ? null
            : callbackExecutor;

    return new CallAdapter<Object, Call<?>>() {
      @Override
      public Type responseType() {
        return responseType;
      }

      @Override
      public Call<Object> adapt(Call<Object> call) {
        return executor == null ? call : new ExecutorCallbackCall<>(executor, call);
      }
    };
  }
}

```

到这里其实我们也就看出来了，经过一连串的调用，我们发现CallAdapted的adapt方法调用其实就是DefaultCallAdapterFactory中get方法返回的CallAdapter对象的adapt方法调用，继续往上看，就是HttpMethodService的invoke方法中adapt方法的调用，又因为上面的executor在Retrofit的build方法中已经初始化，所以这个adapt方法返回一个ExecutorCallbackCall对象

```java
// Retorfit.java
public Retrofit build() {
    ...
    Executor callbackExecutor = this.callbackExecutor;
    if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
    }

    // Make a defensive copy of the adapters and add the default Call adapter.
    List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
    callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));
    ...
}


// Platform.java
@Override
public Executor defaultCallbackExecutor() {
    return new MainThreadExecutor();
}

static final class MainThreadExecutor implements Executor {
    private final Handler handler = new Handler(Looper.getMainLooper());

    @Override
    public void execute(Runnable r) {
        handler.post(r);
    }
}
```

而ExecutorCallbackCall是对OkHttpCall和MainThreadExecutor的封装，它也是DefaultCallAdapterFactory的内部类：

```java
// DefaultCallAdapterFactory.java
static final class ExecutorCallbackCall<T> implements Call<T> {
    final Executor callbackExecutor;
    final Call<T> delegate;

    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
        this.callbackExecutor = callbackExecutor;
        this.delegate = delegate;
    }

    @Override
    public void enqueue(final Callback<T> callback) {
        Objects.requireNonNull(callback, "callback == null");

        delegate.enqueue(
            new Callback<T>() {
                @Override
                public void onResponse(Call<T> call, final Response<T> response) {
                    callbackExecutor.execute(
                        () -> {
                            if (delegate.isCanceled()) {
                                // Emulate OkHttp's behavior of throwing/delivering an IOException on
                                // cancellation.
                                callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
                            } else {
                                callback.onResponse(ExecutorCallbackCall.this, response);
                            }
                        });
                }

                @Override
                public void onFailure(Call<T> call, final Throwable t) {
                    callbackExecutor.execute(() -> callback.onFailure(ExecutorCallbackCall.this, t));
                }
            });
    }

    @Override
    public boolean isExecuted() {
        return delegate.isExecuted();
    }

    @Override
    public Response<T> execute() throws IOException {
        return delegate.execute();
    }
    ...
}
```

通过执行ExecutorCallbackCall的enqueue方法，本质其实就是在执行OkHttpCall的enqueue方法：

```java
// OKHttpCall.java
@Override
public void enqueue(final Callback<T> callback) {
    Objects.requireNonNull(callback, "callback == null");

    okhttp3.Call call;
    Throwable failure;

    synchronized (this) {
        if (executed) throw new IllegalStateException("Already executed.");
        executed = true;

        call = rawCall;
        failure = creationFailure;
        if (call == null && failure == null) {
            try {
                // 创建okhttp3.Call对象
                call = rawCall = createRawCall();
            } catch (Throwable t) {
                throwIfFatal(t);
                failure = creationFailure = t;
            }
        }
    }

    if (failure != null) {
        callback.onFailure(this, failure);
        return;
    }

    if (canceled) {
        call.cancel();
    }

    call.enqueue(
        new okhttp3.Callback() {
            @Override
            public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
                Response<T> response;
                try {
                    response = parseResponse(rawResponse);
                } catch (Throwable e) {
                    throwIfFatal(e);
                    callFailure(e);
                    return;
                }

                try {
                    callback.onResponse(OkHttpCall.this, response);
                } catch (Throwable t) {
                    throwIfFatal(t);
                    t.printStackTrace(); // TODO this is not great
                }
            }

            @Override
            public void onFailure(okhttp3.Call call, IOException e) {
                callFailure(e);
            }

            private void callFailure(Throwable e) {
                try {
                    callback.onFailure(OkHttpCall.this, e);
                } catch (Throwable t) {
                    throwIfFatal(t);
                    t.printStackTrace(); // TODO this is not great
                }
            }
        });
}

private okhttp3.Call createRawCall() throws IOException {
    okhttp3.Call call = callFactory.newCall(requestFactory.create(args));
    if (call == null) {
        throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
}
```

上面代码首先通过createRawCall方法创建了一个okhttp3.Call对象，其实就是通过callFactory的newCall方法创建的，而callFactory对象是在在 OkHttpCall 构造中直接赋值的

```java
// OKHttpCall.java  
OkHttpCall(
    RequestFactory requestFactory,
    Object[] args,
    okhttp3.Call.Factory callFactory,
    Converter<ResponseBody, T> responseConverter) {
    this.requestFactory = requestFactory;
    this.args = args;
    this.callFactory = callFactory;
    this.responseConverter = responseConverter;
}
```

那么OkHttpCall是在哪创建的呢？继续往回追代码，可以看到是在HttpServiceMethod的invoke方法里

```java
@Override
final @Nullable ReturnT invoke(Object[] args) {
  Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
  return adapt(call, args);
}
```

而这里callFactory是HttpServiceMethod的成员变量，在其parseAnnotations方法中被赋值：

```java
// HttpServiceMethod.java
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
    Retrofit retrofit, Method method, RequestFactory requestFactory) {
    
    ...
        
    okhttp3.Call.Factory callFactory = retrofit.callFactory;
    ...
}


```

而Retrofit的callFactory是通过Builder构建的

```java
// Retorfit.java
public Retrofit build() {
    if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
    }

    okhttp3.Call.Factory callFactory = this.callFactory;
    if (callFactory == null) {
        callFactory = new OkHttpClient();
    }

    Executor callbackExecutor = this.callbackExecutor;
    if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
    }

    // Make a defensive copy of the adapters and add the default Call adapter.
    List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
    callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));

    // Make a defensive copy of the converters.
    List<Converter.Factory> converterFactories =
        new ArrayList<>(
        1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());

    // Add the built-in converter factory first. This prevents overriding its behavior but also
    // ensures correct behavior when using converters that consume all types.
    converterFactories.add(new BuiltInConverters());
    converterFactories.addAll(this.converterFactories);
    converterFactories.addAll(platform.defaultConverterFactories());

    return new Retrofit(
        callFactory,
        baseUrl,
        unmodifiableList(converterFactories),
        unmodifiableList(callAdapterFactories),
        callbackExecutor,
        validateEagerly);
}


```

从上面也可以发现原来一直寻找的callFactory就是OkHttpClient，综上也就是通过OkHttpClient的newCall方法创建了Call对象，从这里也就可以看出其实主要用的是OkHttp这一网络请求库，也验证了我们一开始说的Retrofit其实就是对OkHttp的封装。

### Request对象构建

我们还回到ServiceMethod.parseAnnotations方法：

```java
static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);
    ...
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
}

```

再来看下RequestFactory.parseAnnotations方法：

```java
final class RequestFactory {
  static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
    return new Builder(retrofit, method).build();
  }
    Builder(Retrofit retrofit, Method method) {
        this.retrofit = retrofit;
        this.method = method;
        this.methodAnnotations = method.getAnnotations();
        this.parameterTypes = method.getGenericParameterTypes();
        this.parameterAnnotationsArray = method.getParameterAnnotations();
    }
    RequestFactory build() {
      for (Annotation annotation : methodAnnotations) {
          // 解析自定义接口的方法注解
        parseMethodAnnotation(annotation);
      }

      ...

      int parameterCount = parameterAnnotationsArray.length;
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0, lastParameter = parameterCount - 1; p < parameterCount; p++) {
        parameterHandlers[p] =
            parseParameter(p, parameterTypes[p], parameterAnnotationsArray[p], p == lastParameter);
      }

      ...      

      return new RequestFactory(this);
    }
    private void parseMethodAnnotation(Annotation annotation) {
      if (annotation instanceof DELETE) {
        parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
      } else if (annotation instanceof GET) {
        parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
      } else if (annotation instanceof HEAD) {
        parseHttpMethodAndPath("HEAD", ((HEAD) annotation).value(), false);
      } else if (annotation instanceof PATCH) {
        parseHttpMethodAndPath("PATCH", ((PATCH) annotation).value(), true);
      } else if (annotation instanceof POST) {
        parseHttpMethodAndPath("POST", ((POST) annotation).value(), true);
      } else if (annotation instanceof PUT) {
        parseHttpMethodAndPath("PUT", ((PUT) annotation).value(), true);
      } else if (annotation instanceof OPTIONS) {
        parseHttpMethodAndPath("OPTIONS", ((OPTIONS) annotation).value(), false);
      } else if (annotation instanceof HTTP) {
        HTTP http = (HTTP) annotation;
        parseHttpMethodAndPath(http.method(), http.path(), http.hasBody());
      } else if (annotation instanceof retrofit2.http.Headers) {
        String[] headersToParse = ((retrofit2.http.Headers) annotation).value();
        if (headersToParse.length == 0) {
          throw methodError(method, "@Headers annotation is empty.");
        }
        headers = parseHeaders(headersToParse);
      } else if (annotation instanceof Multipart) {
        if (isFormEncoded) {
          throw methodError(method, "Only one encoding annotation is allowed.");
        }
        isMultipart = true;
      } else if (annotation instanceof FormUrlEncoded) {
        if (isMultipart) {
          throw methodError(method, "Only one encoding annotation is allowed.");
        }
        isFormEncoded = true;
      }
    }
    ...
}

```

上面这个方法解析方法注解参数，然后返回RequestFactory 对象，最后传到HttpServiceMethod.parseAnnotations方法里面，后面传递给CallAdapted构造函数，从而调用父类HttpServiceMethod的构造函数，赋值给其成员变量requestFactory，然后在OkHttpCall构造函数中传递，赋值给其成员变量requestFactory，即一开始我们看的代码中的requestFactory值：

`okhttp3.Call call = callFactory.newCall(requestFactory.create(args));`

紧接着我们看下其create方法：

```java
// RequestFactory.java
okhttp3.Request create(Object[] args) throws IOException {
    
    ...

    RequestBuilder requestBuilder =
        new RequestBuilder(
            httpMethod,
            baseUrl,
            relativeUrl,
            headers,
            contentType,
            hasBody,
            isFormEncoded,
            isMultipart);

    if (isKotlinSuspendFunction) {
      // The Continuation is the last parameter and the handlers array contains null at that index.
      argumentCount--;
    }

    List<Object> argumentList = new ArrayList<>(argumentCount);
    for (int p = 0; p < argumentCount; p++) {
      argumentList.add(args[p]);
      handlers[p].apply(requestBuilder, args[p]);
    }

    return requestBuilder.get().tag(Invocation.class, new Invocation(method, argumentList)).build();
  }
```

从上面代码可以看出，最后通过RequestBuilder来构造**okhttp3.Request 的对象**

### Response解析

我们回到OkHttpCall方法中，

```java
// OkHttpCall.java
@Override
public void enqueue(final Callback<T> callback) {
   ...

    call.enqueue(
        new okhttp3.Callback() {
            @Override
            public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
                Response<T> response;
                try {
                    // 解析返回结果
                    response = parseResponse(rawResponse);
                } catch (Throwable e) {
                    throwIfFatal(e);
                    callFailure(e);
                    return;
                }

                try {
                    callback.onResponse(OkHttpCall.this, response);
                } catch (Throwable t) {
                    throwIfFatal(t);
                    t.printStackTrace(); // TODO this is not great
                }
            }

            @Override
            public void onFailure(okhttp3.Call call, IOException e) {
                callFailure(e);
            }

            private void callFailure(Throwable e) {
                try {
                    callback.onFailure(OkHttpCall.this, e);
                } catch (Throwable t) {
                    throwIfFatal(t);
                    t.printStackTrace(); // TODO this is not great
                }
            }
        });
}
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();

    // Remove the body's source (the only stateful object) so we can pass the response along.
    rawResponse =
        rawResponse
            .newBuilder()
            .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
            .build();

    int code = rawResponse.code();
    if (code < 200 || code >= 300) {
      try {
        // Buffer the entire body to avoid future I/O.
        ResponseBody bufferedBody = Utils.buffer(rawBody);
        return Response.error(bufferedBody, rawResponse);
      } finally {
        rawBody.close();
      }
    }

    if (code == 204 || code == 205) {
      rawBody.close();
      return Response.success(null, rawResponse);
    }

    ExceptionCatchingResponseBody catchingBody = new ExceptionCatchingResponseBody(rawBody);
    try {
        // 通过 responseConverter 转换 ResponseBody
      T body = responseConverter.convert(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
  }
```

从上面方法我们看到，解析返回结果的方法为parseResponse方法，在该方法中通过 responseConverter 转换 ResponseBody，而responseConverter同样是OkHttpCall构造方法传进来的，继续往回查找代码，最后找到该对象来自下面这行代码：

```java
// HttpServiceMethod.java
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
      Retrofit retrofit, Method method, RequestFactory requestFactory) {
    ...
	Converter<ResponseBody, ResponseT> responseConverter =
    createResponseConverter(retrofit, method, responseType);
    ...
}

private static <ResponseT> Converter<ResponseBody, ResponseT> createResponseConverter(
    Retrofit retrofit, Method method, Type responseType) {
    Annotation[] annotations = method.getAnnotations();
    try {
        return retrofit.responseBodyConverter(responseType, annotations);
    } catch (RuntimeException e) { // Wide exception range because factories are user code.
        throw methodError(method, e, "Unable to create converter for %s", responseType);
    }
}

```

最后还是调用Retrofit的responseBodyConverter方法：

```java
// Retrofit.java
public <T> Converter<ResponseBody, T> responseBodyConverter(Type type, Annotation[] annotations) {
    return nextResponseBodyConverter(null, type, annotations);
}

public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
    @Nullable Converter.Factory skipPast, Type type, Annotation[] annotations) {
    ...

    int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {
        Converter<ResponseBody, ?> converter =
            converterFactories.get(i).responseBodyConverter(type, annotations, this);
        if (converter != null) {
            //noinspection unchecked
            return (Converter<ResponseBody, T>) converter;
        }
    }

    ...
}

public Retrofit build() {
      // Make a defensive copy of the converters.
      List<Converter.Factory> converterFactories =
          new ArrayList<>(
              1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());

      // Add the built-in converter factory first. This prevents overriding its behavior but also
      // ensures correct behavior when using converters that consume all types.
    // 添加默认的构建的转换器
      converterFactories.add(new BuiltInConverters());
    // 添加自己配置的转换器
      converterFactories.addAll(this.converterFactories);
    // 如果是 Java8 就添加一个 OptionalConverterFactory 的转换器，否则就是一个空的
      converterFactories.addAll(platform.defaultConverterFactories());

      return new Retrofit(
          callFactory,
          baseUrl,
          unmodifiableList(converterFactories),
          unmodifiableList(callAdapterFactories),
          callbackExecutor,
          validateEagerly);
    }
```

可以看到通过遍历converterFactories，然后根据返回值类型type来找到对应的 Converter 解析，如果不为空，直接返回此 Converter 对象，该对象也就是我们要找的Converter对象，这里通过Retrofit的build方法往converterFactories添加自己配置的转换器GsonConverterFactory，从而调用其responseBodyConverter方法：

```java
// GsonConverterFactory.java
@Override
public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
                                                        Retrofit retrofit) {
    TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
    return new GsonResponseBodyConverter<>(gson, adapter);
}
```

上面返回GsonResponseBodyConverter对象，然后调用其convert方法，返回我们需要的结果：

```java
// GsonResponseBodyConverter.java
@Override public T convert(ResponseBody value) throws IOException {
    JsonReader jsonReader = gson.newJsonReader(value.charStream());
    try {
        T result = adapter.read(jsonReader);
        if (jsonReader.peek() != JsonToken.END_DOCUMENT) {
            throw new JsonIOException("JSON document was not fully consumed.");
        }
        return result;
    } finally {
        value.close();
    }
}
```

## RxJava支持

我们在初始化一个Retrofit时加入 `addCallAdapterFactory(RxJava2CallAdapterFactory.create())`这行

```java
val retrofit = Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    //加入 RxJava2CallAdapterFactory 支持
    .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
    .addConverterFactory(GsonConverterFactory.create())
    .build()
    
interface GitHubApiService {
    @GET("users/{user}/repos")
    fun listReposRx(@Path("user") user: String?): Single<Repo>
}

//创建出GitHubApiService对象
val service = retrofit.create(GitHubApiService::class.java)
service.listReposRx("octocat")
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe({ repo ->
        "response name = ${repo[0].name}".logE()
    }, { error ->
        error.printStackTrace()
    })

```

通过该方法会把RxJava2CallAdapterFactory加入到callAdapterFactories这个list集合中，接下来同样回到**HttpServiceMethod的parseAnnotations方法**中：

```java
//HttpServiceMethod.java
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
    Retrofit retrofit, Method method, RequestFactory requestFactory) {

  ....
 
  CallAdapter<ResponseT, ReturnT> callAdapter =
      createCallAdapter(retrofit, method, adapterType, annotations);

  okhttp3.Call.Factory callFactory = retrofit.callFactory;
  if (!isKotlinSuspendFunction) {
    
    return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
  } 
    ...
}

```

在接下来和上面一样的过程，经过一系列方法追溯，最后到了Retrofit的nextCallAdapter方法中：

```java
// Retrofit.java
public CallAdapter<?, ?> nextCallAdapter(
      @Nullable CallAdapter.Factory skipPast, Type returnType, Annotation[] annotations) {
    ...
    for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
      CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
    ...
}

```

遍历 callAdapterFactories 根据 **returnType类型** 来找到对应的 CallAdapter 返回

比如：我们在 GitHubApiService 的 returnType 类型为 Single，那么返回的就是 RxJava2CallAdapterFactory 所获取的 CallAdapter，这里看下通过RxJava2CallAdapterFactory的 `get`方法

```java
// RxJava2CallAdapterFactory.java
@Override public @Nullable CallAdapter<?, ?> get(
      Type returnType, Annotation[] annotations, Retrofit retrofit) {
    
    Class<?> rawType = getRawType(returnType);

    if (rawType == Completable.class) {
      // Completable is not parameterized (which is what the rest of this method deals with) so it
      // can only be created with a single configuration.
      return new RxJava2CallAdapter(Void.class, scheduler, isAsync, false, true, false, false,
          false, true);
    }

    boolean isFlowable = rawType == Flowable.class;
    // 当期是Single，所以isSingle返回true
    boolean isSingle = rawType == Single.class;
    boolean isMaybe = rawType == Maybe.class;
    if (rawType != Observable.class && !isFlowable && !isSingle && !isMaybe) {
      return null;
    }

   ...

    return new RxJava2CallAdapter(responseType, scheduler, isAsync, isResult, isBody, isFlowable,
        isSingle, isMaybe, false);
  }
```

上面 方法返回RxJava2CallAdapter类，然后调用其adapt方法，

```java
// RxJava2CallAdapter.java
@Override public Object adapt(Call<R> call) {
    Observable<Response<R>> responseObservable = isAsync
        ? new CallEnqueueObservable<>(call)
        : new CallExecuteObservable<>(call);

    Observable<?> observable;
    if (isResult) {
        observable = new ResultObservable<>(responseObservable);
    } else if (isBody) {
        observable = new BodyObservable<>(responseObservable);
    } else {
        observable = responseObservable;
    }

    if (scheduler != null) {
        observable = observable.subscribeOn(scheduler);
    }

    if (isFlowable) {
        return observable.toFlowable(BackpressureStrategy.LATEST);
    }
    if (isSingle) {
        return observable.singleOrError();
    }
    if (isMaybe) {
        return observable.singleElement();
    }
    if (isCompletable) {
        return observable.ignoreElements();
    }
    return RxJavaPlugins.onAssembly(observable);
}
```

对于Single则调用observable.singleOrError()方法，剩下的就交给RxJava来处理了。

## 总结

1. 通过自定义接口，通过Retrofit生成对应的接口实例，然后调用接口方法时，会调用InvocationHandler 的 `invoke`方法
2. 然后执行 `loadServiceMethod`方法并返回一个 HttpServiceMethod 对象并调用它的 `invoke`方法
3. 最后执行OkHttpCall的enqueue方法，本质也是在执行okhttp3.Call 的 `enqueue`方法
4. 当然在这期间会解析方法上的注解，构建 okhttp3.Call 需要的 okhttp3.Request 对象
5. 然后通过 Converter 来解析返回的响应数据，并回调 CallBack 接口