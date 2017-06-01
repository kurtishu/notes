---
title: Android App性能优化技巧
date: 2017-06-01 10:43:11
categories: [Android, Performance]
tags: [Performance, Performance Optimizing]
---

当出现App启动慢、界面跳转慢、事件相应慢、滑动和动画卡顿、展现内容慢等问题的时候意味着App性能出现问题，这个时候就有必要对App做一下性能优化，其实不是非等到App出现性能问题的时候才开始做优化，而是要把优化做到平时的开发和维护过程中。但性能优化如何下手？下面结合Android[官方文档](https://developer.android.google.cn/training/best-performance.html)以及平时开发中的经验来聊聊Andrid性能优化的技巧。

### Layout布局优化
Layout的优化目的是处理应用UI卡顿、ANR问题，使App用起来丝般顺滑。
* 优化布局结构，避免复杂的Layout嵌套。比如使用<code>merge</code> 标签减少布局嵌套。
使用 [Hierarchy Viewer](https://developer.android.google.cn/studio/profile/hierarchy-viewer.html) 分析布局，对出现红色、或者黄色的布局需要尤为注意。
![Tree View](https://developer.android.google.cn/images/developing/hv_treeview_screenshot.png)
* 使用Lint进行资源及冗余UI布局分析，根据Lint report提示做相应的优化
* 使用<code>include</code> 标签达到布局重用的目的， 
* 使用懒加载的Views，比如ViewStub
* 使用Memory监测及GC打印与Allocation Tracker进行UI卡顿分析。
* 使用RelativeLayout扁平化布局，ListView item布局，一定要扁平化。
* TextView组合图标，代替LinearLayout+TextView+ImageView。
* 在项目中使用[StrictMode](https://developer.android.google.cn/reference/android/os/StrictMode.html)来检查主线程耗时操作等。
```java
StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder().detectAll().penaltyLog().penaltyDialog().build());
 StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder().detectAll().penaltyLog().build());
```
* 自定义View时，避免onDraw方法过度绘制或者重复绘制，避免在里面初始化变量等。
* ListView或者RecyclerView渲染item时，将耗时操作（文件，网络， DB）放在子线程中，item要是用ViewHolder布局缓存复用机制。
* 使用traces.txt文件进行ANR分析优化，或者使用[Blockcanary](https://github.com/markzhai/AndroidPerformanceMonitor)，使用方法和Leackcanary类似。
* 使用SVG代替图片
* 使用xml代替图片

### 优化设备的电池寿命
电池优化一般只针对耗电量比较大的App，一般应用貌似都还没有考虑到这一点。对于电池使用优化主要考虑：
* 监控电池电量和充电状态，监控显著的电池电量变化，一般而言，最好在电池电量极低时停用所有后台更新。
* 确定和监控插接状态和基座类型，具体参考[官方文档](https://developer.android.google.cn/training/monitoring-device-state/docking-monitoring.html)
* 确定和监控网络连接状态，可以利用 [ConnectivityManager](https://developer.android.google.cn/reference/android/net/ConnectivityManager.html) 来检查是否已实际连入互联网以及已连入情况下的连接类型。

### 高效地利用线程
为了加快响应速度，需要把费时的操作（比如网络请求、数据库操作或者复杂的计算）从主线程移动到一个单独的线程中。
* 可以使用自定义Thread，AsyncTask或者IntentService来创建后台操作。
* 可以使用线程池，并用创建一个Manager类对多线程统一管理
```java
 private PhotoManager() {
        ...
        // Sets the amount of time an idle thread waits before terminating
        private static final int KEEP_ALIVE_TIME = 1;
        // Sets the Time Unit to seconds
        private static final TimeUnit KEEP_ALIVE_TIME_UNIT = TimeUnit.SECONDS;
        // Creates a thread pool manager
        mDecodeThreadPool = new ThreadPoolExecutor(
                NUMBER_OF_CORES,       // Initial pool size
                NUMBER_OF_CORES,       // Max pool size
                KEEP_ALIVE_TIME,
                KEEP_ALIVE_TIME_UNIT,
                mDecodeWorkQueue);
    }
```
* 当View销毁或者不可见的情况下，线程有必要取消、停止, 通常在线程内部设置flag取消耗时操作，尽快让线程结束。
```java
/*
 * Before continuing, checks to see that the Thread hasn't
 * been interrupted
 */
if (Thread.interrupted()) {
    return;
}

/*
 * isCanceled标志
 */
if (isCanceled()) {
   return;
 }
```

### 优化网络
* 如果没有网络连接，请让你的应用跳过网络操作；只在有网络连接并且无漫游的情况下更新数据；
* 选择兼容的数据格式，把含有文本数据和二进制数据的请求全部转化成二进制数据格式请求；
* 使用高效的转换工具，多考虑使用流式转换工具，少用树形的转换工具；
* 为了更快的用户体验，请减少重复访问服务器的操作；
* 如果可以的话，请使用framework的GZIP库来压缩文本数据以高效使用CPU资源。
目前App针对网络的优化，主要使用一个好的网络下载框架，目前OkHttp + Retrofit基本成为了标配。

###  内存泄露分析工具
* 使用AS的Memory monitor，平时用来直观了解自己应用的全局内存情况，大的泄露才能有感知。
* DDMS-Heap内存监测工具	同上，大的泄露才能有感知。
* MAT工具全称为Memory Analyzer Tool，详细分析Java堆内存的工具，该工具非常强大。
* Leakcanary神器，比较强大，可以感知泄露且定位泄露；实质是MAT原理，只是更加自动化了，当现有代码量已经庞大成型，且无法很快察觉掌控全局代码时极力推荐；或者是偶现泄露的情况下极力推荐。

### 规避内存泄露建议
* Context使用不当造成内存泄露；不要对一个Activity Context保持长生命周期的引用（譬如上面概念部分给出的示例）。尽量在一切可以使用应用ApplicationContext代替Context的地方进行替换（原理我前面有一篇关于Context的文章有解释）。
* 非静态内部类的静态实例容易造成内存泄漏；即一个类中如果你不能够控制它其中内部类的生命周期（譬如Activity中的一些特殊Handler等），则尽量使用静态类和弱引用来处理（譬如ViewRoot的实现）。
* 警惕线程未终止造成的内存泄露；譬如在Activity中关联了一个生命周期超过Activity的Thread，在退出Activity时切记结束线程。一个典型的例子就是HandlerThread的run方法是一个死循环，它不会自己结束，线程的生命周期超过了Activity生命周期，我们必须手动在Activity的销毁方法中中调运thread.getLooper().quit();才不会泄露。
* 对象的注册与反注册没有成对出现造成的内存泄露；譬如注册广播接收器、注册观察者（典型的譬如数据库的监听）等。
* 创建与关闭没有成对出现造成的泄露；譬如Cursor资源必须手动关闭，WebView必须手动销毁，流等对象必须手动关闭等。
* 不要在执行频率很高的方法或者循环中创建对象，可以使用HashTable等创建一组对象容器从容器中取那些对象，而不用每次new与释放。
* 避免代码设计模式的错误造成内存泄露；譬如循环引用，A持有B，B持有C，C持有A，这样的设计谁都得不到释放。

### 代码规范制定并遵守
一致的代码风格，有利于代码维护、查看和发现问题所在。参考[CodingStyle](http://kurtishu.github.io/2017/05/24/android-codingstyle/)

### 其他
* 使用Lint、 Proguard工具给APK包减肥。
* 开启GPU过度绘制调试，粉红色尽量优化，界面尽量保持蓝绿颜色， 红色肯定是有问题的，不能忍受。
* 合理使用数据类型，比如StringBuilder代替String，少用父类声明(List, Map)， 尽量使用Android提供的数据结构例如SparseArray。
* 少用枚举enum，可以使用static final的变量或者intDef 注解代替。
* bitmap 必要时进行压缩，降低分辨率处理，并且使用完了记得recycle。
* 使用LruCache最Memory或者Disk Cache
* OnTrimMemory 方法监听，收到内存报警，及时释放内存。
* 你要知道for loop中不要声明临时变量，不到万不得已不要在里面写try catch.
* 设计模式合理的使用，不要一味的为了设计模式而过分的抽象代码，因为代码抽象系数与代码加载执行时间成正比
* Handler发送消息时尽量使用obtain去获取已经存在的Message对象进行复用，而不是新new Message对象，这样可以减轻内存压力。
* 使用Parcelable代替Serializable，Parcelable更适合Android。

### 写在最后
性能优化是一个大的话题，设计到的知识太多太多，上面只是提到部分，后续会不但的更新，如有不对之处，还望指正。

### 参考
[Android性能优化的方方面面](http://www.jianshu.com/p/b3b09fa29f65)
[那些Android中的性能优化](http://www.cnblogs.com/stay/p/4784014.html)

