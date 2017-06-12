---
title: Retrofit 入门以及源码分析
date: 2017-06-09 9:34:20
categories: [Android, 源码分析]
tags: [Retrofit, OkHttp, 源码分析]
---
### Retrofit 简介

>A type-safe HTTP client for Android and Java

Retrofit是Square公司开发的一款针对Java和Android平台开发的网络请求框架，目前的[Retrofit2](https://github.com/square/retrofit)底层是基于[OkHttp](https://github.com/square/okhttp)。Retrofit + OkHttp基本成为目前App开发网络请求的标配了。

### Retrofit 使用
#### 加入Retrofit的依赖
```java
# Retrofit 自身的依赖
compile 'com.squareup.retrofit2:retrofit:2.3.0'

# OkHttp的依赖
compile 'com.squareup.okhttp3:okhttp:3.8.0'

# Gson 转换器的依赖(根据数据类型json，xml选择相应的转换器)
compile 'com.squareup.retrofit2:converter-gson:2.0.0-beta4'
```

####  ProGuard 配置 
```java
# Platform calls Class.forName on types which do not exist on Android to determine platform.
-dontnote retrofit2.Platform
# Platform used when running on Java 8 VMs. Will not be used at runtime.
-dontwarn retrofit2.Platform$Java8
# Retain generic type information for use by reflection by converters and adapters.
-keepattributes Signature
# Retain declared checked exceptions for use by a Proxy instance.
-keepattributes Exceptions
```

####  创建接口
以下以访问Gank的API接口为例：
```java
public interface GankApi {

    /**
     *
     * 分类数据: http://gank.io/api/data/数据类型/请求个数/第几页

     * 数据类型： 福利 | Android | iOS | 休息视频 | 拓展资源 | 前端 | all
     * 请求个数： 数字，大于0
     * 第几页：数字，大于0
     */

    @GET("day/{year}/{month}/{day}")
    Observable<DailyEntity> getDailyData(@Path("year")int year, @Path("month")int month, @Path("day")int day);

    @GET("data/{type}/{pageSize}/{page}")
    Observable<HttpResult<List<GankEntity>>> getGankData(@Path("type") String type, @Path("pageSize") int PageSize, @Path("page") int page);

}
```
Retrofit 提供了很多注解，比如GET， POST， DELETE 代表请求的类型，@Path，用来组装URL path等

#### 实例化Retrofit对象
```java
public class GankService {
    private static GankService mGankService;
    private GankApi gankApi;
    private GankService() {
        OkHttpClient okHttpClient = new OkHttpClient();
        // 构造Gson对象
        Gson gson = new GsonBuilder()
                .setDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'")
                .create();
        // 采用Builder模式构建retrofit对象
        Retrofit retrofit = new Retrofit.Builder()
                // 情况的Host，完全RestFull风格
                .baseUrl(GankConfig.API_HOST) 
                .client(okHttpClient)
                .addConverterFactory(GsonConverterFactory.create(gson))
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                .build();
        // 生成GankApi接口对应的代理类，通过代理类访问请求
        gankApi = retrofit.create(GankApi.class);
    }
}
```
#### 小结
Retrofit 通过接口代理的方式，将设置在接口上的参数（请求类型，请求URL，请求参数，返回类型）等进行组织成一个完整的Request 然后交给OkHttp框架发送请求，OkHttp获取结果后交给Retrofit的Converter转换成期望的对象。综上所述，Retrofit可以理解为一个API请求的 ‘皮包工厂’。

### Retrofit源码分析
以下基于Retrofit 2.0.2进行分析， Retrofit 整个源码非常简洁，总共37个类文件， 22个是注解类，其他类才15个还包括Util类。

#### Retofit成员变量分析
```java
// 用作缓存解析Annotation数据用的，也就是说如何annotiation被解析了会缓存起来，毕竟通过反射是很消耗性能的
private final Map<Method, ServiceMethod> serviceMethodCache = new LinkedHashMap<>();

  // OkHttp 实例
  private final okhttp3.Call.Factory callFactory;
  //　RESTFULL　API的BaseUrl
  private final HttpUrl baseUrl;
  //　转化器的集合
  private final List<Converter.Factory> converterFactories;
  //　返回结果的适配器集合
  private final List<CallAdapter.Factory> adapterFactories;
  //　．．．
  private final Executor callbackExecutor;
  //　．．．
  private final boolean validateEagerly;
  ...
  }
```

#### Retrofit 核心代码
Retrofit通过动态代理生产代理类，在此过程中去解析注解，封装请求，发送请求，解析请求结果等操作。

所谓动态代理，是在程序Runtime时，通过反射机制在目标方法的前后加上一定的逻辑的过程。
动态代理分**JDK的动态代理**和**CGLib动态代理**。

** Retrofit 正是使用了JDK的动态代理**。

```java
public <T> T create(final Class<T> service) {
    // 检查service是否是接口，是否存在继承关系。也就是传入的代理类必须是接口，而且不能继承其他接口。
    Utils.validateServiceInterface(service);

    // 检查是否存在Default方法
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }

   // 使用JDK动态代理来产生代理类， 核心代码在InvocationHandler里面
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          // 获取代码运行的平台是Android，纯Java，iOS(为什么会有iOS平台?) 
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            // 是否是Default方法，Android平台默认都不是default方法。
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            // 1. 首先从serviceMethodCache中获取ServiceMethod
            // 2. 解析Annotation成ServiceMethod对象
            // 3. 再将ServiceMethod放入serviceMethodCache，做cache用
            ServiceMethod serviceMethod = loadServiceMethod(method);
            // 根据解析后的数据组装一个真正的OkHttpCall对象
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            // 发送请求，获取请求结果，然后解析请求结果
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```

#### 获取Platform
它是通过是否存在特定的文件，来推断出对应的平台，只不过这里面有iOS平台，难道iOS平台也可以使用?
```java
private static Platform findPlatform() {
    try {
      Class.forName("android.os.Build");
      if (Build.VERSION.SDK_INT != 0) {
        return new Android();
      }
    } catch (ClassNotFoundException ignored) {
    }
    try {
      Class.forName("java.util.Optional");
      return new Java8();
    } catch (ClassNotFoundException ignored) {
    }
    try {
      Class.forName("org.robovm.apple.foundation.NSObject");
      return new IOS();
    } catch (ClassNotFoundException ignored) {
    }
    return new Platform();
  }
```
#### 解析注解
1. loadServiceMethod
```java
/**
 * 1. 首先从serviceMethodCache中获取ServiceMethod
 * 2. 解析Annotation成ServiceMethod对象
 * 3. 再将ServiceMethod放入serviceMethodCache，做cache用
 * 
 */
ServiceMethod loadServiceMethod(Method method) {
    ServiceMethod result;
    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
       // 重点在这里，同样是使用采用Builder模式，直接看build方法
        result = new ServiceMethod.Builder(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```
2. build方法
```java
public ServiceMethod build() {
      // 获取callAdapter，Android对应的是ExecutorCallAdapterFactory
      callAdapter = createCallAdapter();
      // 获取返回类型数据
      responseType = callAdapter.responseType();
      // 创建转换器对象
      responseConverter = createResponseConverter();
      
      for (Annotation annotation : methodAnnotations) {
        // 解析添加在在method的注解
        parseMethodAnnotation(annotation);
      }
      ...
      
      return new ServiceMethod<>(this);
    }
```

3. parseMethodAnnotation
根据请求类型进行分类解析
```java
private void parseMethodAnnotation(Annotation annotation) {
      if (annotation instanceof DELETE) {
       // 解析参数和路径
        parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
      } else if (annotation instanceof GET) {
        parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
      } else if (annotation instanceof HEAD) {
        parseHttpMethodAndPath("HEAD", ((HEAD) annotation).value(), false);
        if (!Void.class.equals(responseType)) {
          throw methodError("HEAD method must use Void as response type.");
        }
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
          throw methodError("@Headers annotation is empty.");
        }
        // 解析header
        headers = parseHeaders(headersToParse);
      } else if (annotation instanceof Multipart) {
        if (isFormEncoded) {
          throw methodError("Only one encoding annotation is allowed.");
        }
        isMultipart = true;
      } else if (annotation instanceof FormUrlEncoded) {
        if (isMultipart) {
          throw methodError("Only one encoding annotation is allowed.");
        }
        isFormEncoded = true;
      }
    }

```

#### 如何在Retrofit中使用RxJava
由于Retrofit设计的扩展性非常强，你只需要添加一个CallAdapter就可以了

```java
	Retrofit retrofit = new Retrofit.Builder()
	  .baseUrl(GankConfig.API_HOST)
	  .addConverterFactory(ProtoConverterFactory.create())
	  .addConverterFactory(GsonConverterFactory.create())
	  // 设置CallAdapter
	  .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
	  .build();
```
上面代码创建了一个Retrofit对象，支持Proto和Gson两种数据格式，并且还支持RxJava
更多可以参考[RxJava 与 Retrofit 结合的最佳实践](http://gank.io/post/56e80c2c677659311bed9841)

### 总结
Retrofit就是在组装请求，把请求交给Adapter(OkHttp)去处理请求，然后拿回结果用Converter将原始数据转化成期望的对象。 通过Retrofit的封装，使得App网络请求层代码简洁，方便维护。


