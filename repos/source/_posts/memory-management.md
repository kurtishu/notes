---
title: Android 内存管理机制
date: 2017-06-06 14:08:45
categories: [Android, Memory]
tags: [Performance, Memory Management]
---

### Android 内存管理机制简介

* 每个应用程序都是一个独立的进程，互不影响，进程死掉不会导致系统崩溃、重启。
* 每个应用程序都有内存上限，接近、超过都会被kill。

### App内存设置
Android是一个多任务系统, 为了保证多任务的运行, Android给每个App可使用的Heap大小设定了一个限定值.

这个值是系统设置的prop值, 系统编译时内置的, 保存在system/build.prop中,  可以通过如下命令查看:

```java
 $ adb shell
shell@n7100:/ $ cat /system/build.prop
```
查询结果(只保留内存分配相关配置)：
```java
// App启动后, 系统分配给它的Heap初始大小. 随着App使用可增加.
alvik.vm.heapstartsize=8m
// 默认情况下, App可使用的Heap的最大值, 超过这个值就会产生OOM.
dalvik.vm.heapgrowthlimit=192m
// 如果App的manifest文件中配置了largeHeap（android:largeHeap="true"）属性, 如下.则App可使用的Heap的最大值为此项设定值.
dalvik.vm.heapsize=512m
// 当前理想的堆内存利用率. GC后, Dalvik的Heap内存会进行相应的调整, 调整到当前存活的对象的大小和 / Heap大小 接近这个选项的值, 即这里的0.75. 注意, 这只是一个参考值.
dalvik.vm.heaptargetutilization=0.75
// 单次Heap内存调整的最小值.
dalvik.vm.heapminfree=512k
// 单次Heap内存调整的最大值.
dalvik.vm.heapmaxfree=8m
```
也可以直接使用getprop查看单项prop:
```java
$ adb shell getprop dalvik.vm.heapsize
512m
```

### Android进程分类
在Android中，进程主要分为以下几个种类：
* 前台进程（Foreground）
目前正在屏幕上显示的进程和一些系统进程。举例来说，Dialer，Storage，Google Search等系统进程就是前台进程；再举例来说，当你运行一个程序，如浏览器，当浏览器界面在前台显示时，浏览器属于前台进程（foreground），但一旦你按home回到主界面，浏览器就变成了后台程序（background）。我们最不希望终止的进程就是前台进程。
* 可见进程（Visible）
可见进程是一些不再前台，但用户依然可见的进程，举个例来说：widget、输入法等，都属于visible。这部分进程虽然不在前台，但与我们的使用也密切相关，我们也不希望它们被终止。
* 桌面进程（Home app）
即launcher，保证在多任务切换之后，可以快速返回到home界面而不需重新加载launcher。
* 次要服务（Secondary server）
目前正在运行的一些服务（主要服务，如拨号等，是不可能被进程管理终止的，故这里只谈次要服务），举例来说：谷歌企业套件，Gmail内部存储，联系人内部存储等。这部分服务虽然属于次要服务，但很一些系统功能依然息息相关，我们时常需要用到它们，所以也太希望他们被终止。
* 后台进程（Hidden）
 后台进程（background）就是我们通常意义上理解的启动后被切换到后台的进程，如浏览器，阅读器等。当程序显示在屏幕上时，他所运行的进程即为前台进程（foreground），一旦我们按home返回主界面（注意是按home，不是按back），程序就驻留在后台，成为后台进程（background）。
* 内容供应节点（Content provider）
没有程序实体，进提供内容供别的程序去用的，比如日历供应节点，邮件供应节点等。在终止进程时，这类程序应该有较高的优先权。
* 空进程（Empty）
没有任何东西在内运行的进程，有些程序，比如BTE，在程序退出后，依然会在进程中驻留一个空进程，这个进程里没有任何数据在运行，作用往往是提高该程序下次的启动速度或者记录程序的一些历史信息。这部分进程无疑是应该最先终止的。

### Android 进程级别 与 oom_adj
当系统的内存不足时, Android系统将根据进程优先级选择杀死一些不太重要的进程。
那么进程的优先级是怎样判别的呢？对，就是这个根据进程的oom_adj值。oom_adj的值越小，进程的优先级越高。

| 进程级别        |    oom_adj         |
| :-------------:  | :-------------:  |
| 前台进程 （Foreground Process） |  0 |
| 已启动服务的进程(Started Service Process) |  0 |
| 可见进程 （Visible Process） |  1 |
| 后台进程 （Backgroud Process） |  2 |
| 主界面 （Home process）： |  4 |
| 隐藏进程 （Hidden process | 7 |
| 内容提供者 （content provider） | 14 |
| 空进程 （Empty process） | 15 |

可以通过adb 命令查看oom_adj值
```java
Kurtis-Hu:Desktop kurtishu$ adb shell
shell@n7100:/ $ ps
USER      PID   PPID  VSIZE  RSS   WCHAN            PC  NAME
root      1     0     1332   804   sys_epoll_ 00000000 S /init
u0_a35    6230  2077  885440 33812 sys_epoll_ 00000000 S com.android.calendar
u0_a48    6250  2077  876176 33788 sys_epoll_ 00000000 S com.google.android.inputmethod.latin.dictionarypack
u0_a2     6265  2077  877416 34560 sys_epoll_ 00000000 S com.android.providers.calendar
u0_a29    7605  2077  906064 49436 sys_epoll_ 00000000 S com.androidmarket.dingzhi:usbhelpersdk

// 或者直接搜索进程
1|shell@n7100:/ $ ps | grep wandoujia 
u0_a72    5274  2077  1739236 108684 sys_epoll_ 00000000 S com.wandoujia.phoenix2
``` 
以上命令可以获取进程ID, 例如豌豆荚App的ID 5274，然后cat  /proc/PID/oom_adj查看oom_adj值。
```java
// App处在前台
1|shell@n7100:/ $ cat  /proc/5274/oom_adj
0

// 点击Home，App进入后台
shell@n7100:/ $ cat  /proc/5274/oom_adj 
6
``` 
通常修改指定包名应用的oom_adj，避免被系统回收，以到达进程保活的目的。

用以下命令可以查看程序的内存使用情况：
```java
//使用程序的包名或者进程id
adb shell dumpsys meminfo $package_name or $pid    
```
	  

未完待续...

### 参考
> [Android内存管理分析总结](http://www.jianshu.com/p/8b1d9c86fa84)
> [Android是如何管理App内存的--Android内存优化第二弹](http://www.jianshu.com/p/4ad716c72c12)







