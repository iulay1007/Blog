# 设计模式

首先看一下create()方法的使用流程

我们需要根据自己的需要创建要进行的网络请求的接口

```java
public interface MyApi {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```

然后将这个接口类传给retrofit.create()方法

```java
MyApi api = retrofit.create(MyApi.class);
```

接着，我们就可以根据这个对象去调用它的网络请求方法

```java
Call<List<Repo>> list = api.listRepos("a");
```

观察上面的代码，可以发现，进行create()操作时，我们需要传入一个接口类。但在运行前，retrofit是不知道我们要传入什么样的接口类的，所以其实retrofit也应该不知道create方法该返回什么样的对象。

但实际上，retrofit不仅知道该返回什么对象，还知道该对象具有的方法，以及方法上的注解（如@GET）和参数等。

一般来说，如果想要动态获取类的对象，我们是可以用反射来实现的。

但是这里是要获取接口的实例，我们不能通过简单的反射方法（new实例）实现，因为这是个接口，没有构造器。

所以我们需要使用代理的方法，将对接口的调用交由代理类处理，即代理类相当于接口的实现。

但我们不能使用简单的代理模式，因为传统的代理模式要求我们事先知道将要被代理的类的信息。这样与我们想要的动态获取类的信息的思想不符合。所以应该要使用动态代理实现。

动态代理的好处是，我们不需要为每一个类写一个代理类，也不需要事先知道我们将会代理什么类。

**总**：Retrofit 中的动态代理：

- 在代码运行中，会动态创建 Api 接口的实现类，作为代理对象，代理接口的方法
- 在我们调用Api接口的实现类的listRepos方法时，会调用了 InvocationHandler 的 invoke方法。
- 本质上是在运行期，生成了 Api 接口的实现类，调用了 InvocationHandler 的 invoke方法。

接下来看create()方法理解动态代理的过程。

首先我们要知道实现动态代理的两个方法：Proxy类的newProxyInstance和InvocationHandler的invoke方法。

### Proxy#newProxyInstance()

参数含义：

**loader**：用于定义代理类的类加载器

**interfaces**：代理类要实现的接口列表

**h**：将方法调用分派给h（InvocationHandler）

```java
newProxyInstance(ClassLoader loader,Class<?>[] interfaces,
InvocationHandler h){}
```



### InvocationHandler#invoke()

参数含义：

**proxy**：代理类的实例

**method**：需要代理的方法

**args**：代理方法的参数数组

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
```

其实当我们调用listRepos（）方法的时候，实际上是调用InvocationHandler 的invoke（）方法。它获取到了我们的方法，参数等信息。然后根据我们在方法上的注解，去拼接为一个正常的OkHttp 请求，然后执行。

以上两个方法在Retrofit中的具体运用如下：

### Retrofit#create()

```Java
public <T> T create(final Class<T> service) {
    //判断传进来的是不是接口，不是接口就抛出异常
  validateServiceInterface(service);
  return (T)
      Proxy.newProxyInstance(
          service.getClassLoader(),
          new Class<?>[] {service},
          new InvocationHandler() {
            private final Object[] emptyArgs = new Object[0];
@Override
        public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
            throws Throwable {
          // 如果该方法是来自Object的方法，则遵循正常调用。
          if (method.getDeclaringClass() == Object.class) {
            return method.invoke(this, args);
          }
          args = args != null ? args : emptyArgs;
          Platform platform = Platform.get();
            
          return platform.isDefaultMethod(method)
              ? platform.invokeDefaultMethod(method, service, proxy, args) :loadServiceMethod(method).invoke(args);//1
        }
      });
}
```
注释1处

**return platform.isDefaultMethod(method) ? platform.invokeDefaultMethod(method, service, proxy, args) :loadServiceMethod(method).invoke(args);**

首先，根据平台的特性去判断该方法是否为default方法 ，有些平台不支持default方法。platform.isDefaultMethod(method) 。

Java8以后，接口中可以有default方法，该方法可以在接口中实现，且一定要实现。

所以，当该方法在当前平台下属于default方法时，直接调用该方法即可，即platform.invokeDefaultMethod(method, service, proxy, args)

若该方法不是default方法，则调用loadServiceMethod(method).invoke(args)

### Retrofit#loadServiceMethod

根据method去获取该方法的信息，如注解，参数，返回值等

```java
ServiceMethod<?> loadServiceMethod(Method method) {
    //serviceMethodCache是一个ConcurrentHashMap，键是method，每次解析获取method的信息后，将其存入ConcurrentHashMap，这样可以避免重复的解析
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

### ServiceMethod#parseAnnotations()

ServiceMethod位于retrofit的包下

```java
static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
  RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);//1

  Type returnType = method.getGenericReturnType();
    //处理不正确的返回形式
  if (Utils.hasUnresolvableType(returnType)) {
    throw methodError(
        method,
        "Method return type must not include a type variable or wildcard: %s",
        returnType);
  }
  if (returnType == void.class) {
    throw methodError(method, "Service methods cannot return void.");
  }
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);//2

}
```

#### 注释1

**RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);**

##### RequestFactory#parseAnnotations()

```java
static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
  return new Builder(retrofit, method).build();
}
```

##### Builder()

RequestFactory的内部类

```java
Builder(Retrofit retrofit, Method method) {
  this.retrofit = retrofit;
  this.method = method;
   //反射获取方法的注解
  this.methodAnnotations = method.getAnnotations();
   //反射获取方法的参数类型
  this.parameterTypes = method.getGenericParameterTypes();
  //反射获取方法的参数的注解
  this.parameterAnnotationsArray = method.getParameterAnnotations();
}
```

##### Builder#build()

```java
RequestFactory build() {
  for (Annotation annotation : methodAnnotations) {
      //解析方法注解并赋值给其成员变量
    parseMethodAnnotation(annotation);
  }

  if (httpMethod == null) {
    throw methodError(method, "HTTP method annotation is required (e.g., @GET, @POST, etc.).");
  }

  if (!hasBody) {
    if (isMultipart) {
      throw methodError(
          method,
          "Multipart can only be specified on HTTP methods with request body (e.g., @POST).");
    }
    if (isFormEncoded) {
      throw methodError(
          method,
          "FormUrlEncoded can only be specified on HTTP methods with "
              + "request body (e.g., @POST).");
    }
  }

  int parameterCount = parameterAnnotationsArray.length;
  parameterHandlers = new ParameterHandler<?>[parameterCount];
  for (int p = 0, lastParameter = parameterCount - 1; p < parameterCount; p++) {
    parameterHandlers[p] =
        parseParameter(p, parameterTypes[p], parameterAnnotationsArray[p], p == lastParameter);
  }

  if (relativeUrl == null && !gotUrl) {
    throw methodError(method, "Missing either @%s URL or @Url parameter.", httpMethod);
  }
  if (!isFormEncoded && !isMultipart && !hasBody && gotBody) {
    throw methodError(method, "Non-body HTTP method cannot contain @Body.");
  }
  if (isFormEncoded && !gotField) {
    throw methodError(method, "Form-encoded method must contain at least one @Field.");
  }
  if (isMultipart && !gotPart) {
    throw methodError(method, "Multipart method must contain at least one @Part.");
  }
	//将builder对象作为参数传给RequestFactory的构造函数
  return new RequestFactory(this);
}
```

#### 注释2

**HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);**

```java
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
    Retrofit retrofit, Method method, RequestFactory requestFactory) {
  boolean isKotlinSuspendFunction = requestFactory.isKotlinSuspendFunction;
  boolean continuationWantsResponse = false;
  boolean continuationBodyNullable = false;

  Annotation[] annotations = method.getAnnotations();
  Type adapterType;
  if (isKotlinSuspendFunction) {
    //...省略了关于kotlin的逻辑
  } else {
    adapterType = method.getGenericReturnType();
  }

   
  CallAdapter<ResponseT, ReturnT> callAdapter =
      createCallAdapter(retrofit, method, adapterType, annotations);//2.1
    
  Type responseType = callAdapter.responseType();
  if (responseType == okhttp3.Response.class) {
    throw methodError(
        method,
        "'"
            + getRawType(responseType).getName()
            + "' is not a valid response body type. Did you mean ResponseBody?");
  }
  if (responseType == Response.class) {
    throw methodError(method, "Response must include generic type (e.g., Response<String>)");
  }
  // TODO support Unit for Kotlin?
  if (requestFactory.httpMethod.equals("HEAD") && !Void.class.equals(responseType)) {
    throw methodError(method, "HEAD method must use Void as response type.");
  }
  Converter<ResponseBody, ResponseT> responseConverter =
      createResponseConverter(retrofit, method, responseType);//2.2

    
  okhttp3.Call.Factory callFactory = retrofit.callFactory;//2.3
  if (!isKotlinSuspendFunction) {
    return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);//2.4
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

##### 注释2.1

**CallAdapter<ResponseT, ReturnT> callAdapter = createCallAdapter(retrofit, method, adapterType, annotations);**

最终执行的主要逻辑如下，即到retrofit中的callAdapterFactories寻找对应的callAdapter

```java
for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
  CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
  if (adapter != null) {
    return adapter;
  }
```

retrofit中的callAdapterFactories的初始值其实是根据根据不同平台默认创建的

```java
List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
List<? extends CallAdapter.Factory> defaultCallAdapterFactories =
    platform.createDefaultCallAdapterFactories(callbackExecutor);
callAdapterFactories.addAll(defaultCallAdapterFactories);
```

但也可以通过代码动态添加CallAdapterFactory

如：

```java
CallAdapter.Factory factory = RxJava2CallAdapterFactory.create();
Retrofit retrofit;
retrofit = new Retrofit.Builder()
    .baseUrl("http://localhost:1")
    .addConverterFactory(new StringConverterFactory())
    .addCallAdapterFactory(factory)
    .build();
```

###### Retrofit#addCallAdapterFactory

```java
public Builder addCallAdapterFactory(CallAdapter.Factory factory) {
  callAdapterFactories.add(Objects.requireNonNull(factory, "factory == null"));
  return this;
}
```

callAdapterFactories中的一个常用的CallAdapterFactory为DefaultCallAdapterFactory，我们可以看一下它的get方法来理解CallAdapter的获取流程

###### DefaultCallAdapterFactory#get()

```java
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
```

##### 注释2.2

**Converter<ResponseBody, ResponseT> responseConverter =
​      createResponseConverter(retrofit, method, responseType);**

  遍历找到合适的转换器 ，也就是我们在构造retrofit时常用的addConverterFactory，通过addConverterFactory添加的转换器会在这里被寻找到。

```cpp
addConverterFactory(GsonConverterFactory.create())
```

##### 注释2.3

callFactory是Retrofit类里的，是在build时创建的

```java
if (callFactory == null) {
  callFactory = new OkHttpClient();
}
```

##### 注释2.4

如果不是 Kotlin 挂起函数，则返回 CallAdapted 对象

CallAdapted是HttpServiceMethod的子类，HttpServiceMethod是ServiceMethod的子类

```java
return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
```



所以，create里的loadServiceMethod(method).invoke(args);实际调用的是CallAdapted对象的invoke(),也就是该对象从HttpServiceMethod继承的invoke()。

```java
final @Nullable ReturnT invoke(Object[] args) {
  Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);//2.4.1
  return adapt(call, args);//2.4.2
}
```

###### 注释2.4.1

 **Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);**

用OkHttpCall去创建Call类型的实例，看一下它对execute()方法的实现。有过安卓开发经验的人应该知道，我们通常用call.execute()开启网络请求。

###### OkHttpCall#execute()

```java
public Response<T> execute() throws IOException {
  okhttp3.Call call;

  synchronized (this) {
    if (executed) throw new IllegalStateException("Already executed.");
    executed = true;

    call = getRawCall();
  }

  if (canceled) {
    call.cancel();
  }

  return parseResponse(call.execute());
}
```

继续看这里的call = getRawCall();

###### OkHttpCall#getRawCall()

```java
private okhttp3.Call getRawCall() throws IOException {
  okhttp3.Call call = rawCall;
  if (call != null) return call;

  // Re-throw previous failures if this isn't the first attempt.
  if (creationFailure != null) {
    if (creationFailure instanceof IOException) {
      throw (IOException) creationFailure;
    } else if (creationFailure instanceof RuntimeException) {
      throw (RuntimeException) creationFailure;
    } else {
      throw (Error) creationFailure;
    }
  }

  // Create and remember either the success or the failure.
  try {
    return rawCall = createRawCall();
  } catch (RuntimeException | Error | IOException e) {
    throwIfFatal(e); // Do not assign a fatal error to creationFailure.
    creationFailure = e;
    throw e;
  }
}
```

getRawCall()方法就是判断有没有初始化call，没有就create

###### OkHttpCall#createRawCall()

```java
private okhttp3.Call createRawCall() throws IOException {
  okhttp3.Call call = callFactory.newCall(requestFactory.create(args));
  if (call == null) {
    throw new NullPointerException("Call.Factory returned null.");
  }
  return call;
}
```

调用我们创建OkHttpCall时传入的**callFactory**，它是okhttp3.Call.Factory类型的。即注释2.3创建的 callFactory = new OkHttpClient();

**requestFactory**是在注释1处创建的那个解析了注释的requestFactory。

继续看注释2.4.1中的parseResponse方法

###### OkHttpCall#parseResponse

```java
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

​    **T body = responseConverter.convert(catchingBody);**

调用我们创建OkHttpCall传入的responseConverter解析返回值

###### 注释2.4.2

**return adapt(call, args);**

这里的adapt方法是由CallAdapte实现的

```java
protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
  return callAdapter.adapt(call);
}
```

这里的callAdapter就是注释2.1处我们遍历去寻找的那个callAdapterFactories中生成的。

在DefaultCallAdapterFactory中，返回值如下：

```java
new CallAdapter<Object, Call<?>>() {
    @Override
    public Type responseType() {
      return responseType;
    }

    @Override
    public Call<Object> adapt(Call<Object> call) {
      return executor == null ? call : new ExecutorCallbackCall<>(executor, call);
    }
  };
```

在DefaultCallAdapterFactory中，一般情况，adapt方法只是将参数返回。

这里的返回值是Call<Object>，其实也就是我们在定义Api接口时规定的返回值类型。

这里就释了为什么我们create后生成的对象具有与Api接口定义相同的方法。

比如在我们不设置CallAdapterFactory时，默认返回的是Call<Object>类型，也就是Api接口里定义的返回类型。

当然，我们常用的CallAdapterFactory还有上面提到的RxJava2CallAdapterFactory。

###### RxJava2CallAdapterFactory#get()

```java
public @Nullable CallAdapter<?, ?> get(
    Type returnType, Annotation[] annotations, Retrofit retrofit) {
  Class<?> rawType = getRawType(returnType);

  if (rawType == Completable.class) {
    // Completable is not parameterized (which is what the rest of this method deals with) so it
    // can only be created with a single configuration.
    return new RxJava2CallAdapter(
        Void.class, scheduler, isAsync, false, true, false, false, false, true);
  }
```

继续看RxJava2CallAdapter的adpt()方法

###### RxJava2CallAdapter#adpt()

```java
public Object adapt(Call<R> call) {
  Observable<Response<R>> responseObservable =
      isAsync ? new CallEnqueueObservable<>(call) : new CallExecuteObservable<>(call);

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
    // We only ever deliver a single value, and the RS spec states that you MUST request at least
    // one element which means we never need to honor backpressure.
    return observable.toFlowable(BackpressureStrategy.MISSING);
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

一般情况下返回的Observable的子类。这是因为它要适配RxJava，通常我们使用RxJava时Api接口的返回值为Observable。



## 策略模式

### 简介

在开发中常常遇见一种情况：实现某一个功能可以有多种算法，我们根据实际情况选择不同的算法或策略来完成该功能。针对这种情况，一种常规的方法是将多种算法写在一个类中，该类提供多种算法，每一个方法对应一个算法，用户根据自己的需求调用这些方法。也可以将这些算法封装在一个统一的方法中，通过if...else...或者case等条件判断语句来选择具体的算法。

这两种实现方式我们都可以称为硬编码。当很多个算法集中在一个类中时，这个类就会变得臃肿，也会使维护成本提高。这明显违法了开闭原则和单一职责原则。

如果将这些算法或策略抽象出来，提供一个统一的接口，不同的算法或策略有不同的实现类，这样在程序客户端就可以通过注入不同的实现对象来实现算法的动态替换，这种模式的可扩展性、可维护性也就更高。这就是策略模式的思想。

上面我们提到了转换器，可以在构建retrofit时用

```java
addConverterFactory(GsonConverterFactory.create())
```

添加转换器的工厂，以便之后用它来构造转换器。

所有的转换器都需要实现接口Converter。

Converter接口有一个convert方法，用于将从服务器上获取的返回值转换为需要的类型。不同的返回类型，需要不同的转换器去解析。每一种解析方式，就相当于一种策略，我们根据不同需求选择不同的策略，便是策略模式的应用场景。



### 在Retrofit中解决的问题

Retrofit在进行网络请求获得从服务端发来的数据后，需要对这些Json或Xml格式的数据进行解析，变成用户所需要的数据类型。

在Retrofit中通过转换器将获得的数据进行转换，然而不同的数据格式需要不同的转换器，不同的转换器中有不同的算法。需要根据实际情况进行转换器的选择。所以Retrofit将转换这一功能进行抽象，定义为Converter接口，它拥有convert方法，可以进行数据的解析与转换。

Retrofit内置的转换器有GsonResponseBodyConverter**、**

JacksonResponseBodyConverter、JaxbResponseConverter等。它们可以对Json或Xml格式的数据进行解析与转换。除了内置的转换器以外，我们也可以实现Converter接口自定义转换器。

这些转换器可以在我们创建Retrofit时添加进去，这样，就是完成了对转换策略的选择。它使得程序的扩展性提高。



接下来看一下我们常用的转换器。

### GsonResponseBodyConverter

```java
final class GsonResponseBodyConverter<T> implements Converter<ResponseBody, T> {
  private final Gson gson;
  private final TypeAdapter<T> adapter;

  GsonResponseBodyConverter(Gson gson, TypeAdapter<T> adapter) {
    this.gson = gson;
    this.adapter = adapter;
  }

  @Override
  public T convert(ResponseBody value) throws IOException {
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
}
```

除此之外，还有支持Json解析的JacksonResponseBodyConverter、支持Xml解析的JaxbResponseConverter



## 适配器模式

### 简介

将一个类的接口转换成用户希望的另一个接口



### 在Retrofit的应用

在上面介绍动态代理模式的时候我们提到过CallAdapter接口，它拥有方法

```java
T adapt(Call<R> call);
```

RxJava2CallAdapter也是CallAdapter的实现类。

这个adapt方法的作用是将传进来的Call类型对象转换成另一个类型。

在我们不手动添加CallAdapterFactory时，默认会将传入的Call类型对象返回

而若我们添加了RxJava2CallAdapterFactory，RxJava2CallAdapter会将传入的Call转换成为Observable类型或者Observable的变种Flowable、Single等。



## 装饰模式

### 简介

动态地给一个对象增加一些额外的功能，但不会改变对象的接口。这是与适配器模式的最大区别。



### 在Retrofit中的应用

在DefaultCallAdapterFactory中对CallAdapter的实现为

```java
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
```

可以看到它在adapt中定义了两种返回值类型，即传入的Call类型（实际上是OkHttpCall）和ExecutorCallbackCall类型。

OkHttpCall是Call接口的实例。而ExecutorCallbackCall是Call接口的装饰类，它不仅实现了Call接口，它还拥有一个Call类型的引用，这个引用对象是在创建ExecutorCallbackCall时传入的，在这里为传入的OkHttpCall类型对象。

真正执行网络请求的是传入的Call类型的实例，即OkHttpCall类型对象。而ExecutorCallbackCall为Call增加了指定线程的功能。在构造ExecutorCallbackCall时，可以传入Executor类型的实例。

在调用enqueue()方法时，实际调用的是OkHttpCall的enqueue方法。而回调的方法是在传入的Executor类型实例的线程中的。

如果你希望在主线程接受回调，通常需要通过Handler转换到主线程上去。而ExecutorCallbackCall可以指定线程，所以可以用于转换到主线程中。



