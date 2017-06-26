---
title: Android 事件分发机制
date: 2017-05-26
categories: Android
tags: [事件分发, OnClick, OnTouch]
---

在Android中事件分发是很重要的一块知识，了解并熟悉整套的分发机制有助于更好的分析各种点击滑动失效问题，更好去扩展控件的事件功能和开发自定义控件，同时事件分发机制也是Android面试必问考点之一。

###  MotionEvent事件初探
MotionEvent在Android View事件分发、处理过程中很重要的一个类，我们对屏幕的点击，滑动，抬起等一系的动作都是由一个一个MotionEvent对象组成的。根据不同动作，主要有以下三种事件类型：
* **ACTION_DOWN**：手指刚接触屏幕，按下去的那一瞬间产生该事件
* **ACTION_MOVE**：手指在屏幕上移动时候产生该事件
* **ACTION_UP**：手指从屏幕上松开的瞬间产生该事件

**从ACTION_DOWN开始到ACTION_UP结束我们称为一个事件序列**
正常情况下，无论你手指在屏幕上有多么骚的操作，最终呈现在MotionEvent上来讲无外乎下面两种:
* 点击后抬起，也就是单击操作：ACTION_DOWN -> ACTION_UP
* 点击后再风骚的滑动一段距离，再抬起：ACTION_DOWN -> ACTION_MOVE -> ... -> ACTION_MOVE -> ACTION_UP

### 事件分发中重要方法
事件分发需要View的三个重要方法来共同完成：
* **public boolean dispatchTouchEvent(MotionEvent event)**
如果一个MotionEvent传递给了View，那么dispatchTouchEvent方法一定会被调用！
返回值：表示是否消费了当前事件。可能是View本身的onTouchEvent方法消费，也可能是子View的dispatchTouchEvent方法中消费。
**返回 true**表示事件被消费，本次的事件终止传递。
**返回 false**表示View以及子View均没有消费事件，将调用父View的onTouchEvent方法。

* **public boolean onInterceptTouchEvent(MotionEvent ev) **
事件拦截，当一个ViewGroup在接到MotionEvent事件序列时候，首先会调用此方法判断是否需要拦截。
**特别注意，这是ViewGroup特有的方法**，View并没有拦截方法
返回值：是否拦截事件传递，返回true表示拦截了事件，那么事件将不再向下分发而是调用View本身的onTouchEvent方法。返回false表示不做拦截，事件将向下分发到子View的dispatchTouchEvent方法。

* **public boolean onTouchEvent(MotionEvent ev)**
真正对MotionEvent进行处理或者说消费的方法。在dispatchTouchEvent进行调用。
返回值：返回true表示事件被消费，本次的事件终止。返回false表示事件没有被消费，将调用父View的onTouchEvent方法

以下是伪代码（基于ViewGroup源码)描述以上三个方法的关系：
```java
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
            // 检查是否拦截该事件 
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                // 子View请求是否需要父View拦截标志
               if (!disallowIntercept) {
                    // 调用onIntercepteTouchEvent判断时候拦截该事件
                    intercepted = onInterceptTouchEvent(ev);
                } 
            } 
        
        if (intercepted)  {
           // 该事件被拦截，交由onTouchEvent方法来处理
           handled = onTouchEvent(ev)
        } else {
           // 该事件未被拦截，交由其子类的dispatchTouchEvent处理
           handled = child.dispatchTouchEvent(ev);
        }
        return handled;
    }
```
我们再看看View的dispatchTouchEvent方法是如何实现的：
```java
    public boolean dispatchTouchEvent(MotionEvent event) {
        // 事件是否被消费
        boolean result = false;
        // 相关的监听器信息
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            // 检查事件被OnTouchListener的onTouch方法消费了
            result = true;
        }
        // 如果没有被执行，才执行onTouchEvent
        if (!result && onTouchEvent(event)) {
            result = true;
        }

        return result;
    }
```

在ViewGroup的dispatchTouchEvent我们发现disallowIntercept这个标志， 通过它我们可以找到另一个重要的方法requestDisallowInterceptTouchEvent：
* **requestDisallowInterceptTouchEvent**
requestDisallowInterceptTouchEvent 是ViewGroup类中的一个公用方法,  android系统中，一次点击事件是从父view传递到子view中，每一层的view可以决定是否拦截并处理点击事件或者传递到下一层，如果子view不处理点击事件，则该事件会传递会父view，由父view去决定是否处理该点击事件。但子view可以通过设置此方法去告诉父view不要拦截并处理点击事件，父view应该接受这个请求直到此次点击事件结束。

实际的应用中，可以在子view的onTouch事件中注入父ViewGroup的实例，并调用requestDisallowInterceptTouchEvent去阻止父view拦截点击事件
```java
public boolean onTouchEvent(View v, MotionEvent event) {
     ViewGroup viewGroup = (ViewGroup) v.getParent();
     switch (event.getAction()) {
     case MotionEvent.ACTION_MOVE: 
         viewGroup.requestDisallowInterceptTouchEvent(true);
         break;
     case MotionEvent.ACTION_UP:
     case MotionEvent.ACTION_CANCEL:
         viewGroup .requestDisallowInterceptTouchEvent(false);
         break;
     }
}
```

### 事件分发过程
了解Android整个事件分发过程，比较好的方式是写一个Demo，步骤大致是先重写一个ViewGroup，一个View，然后重写Activity的dispatchTouchEvent、onTouchEvent，ViewGroup的dispatchTouchEvent、onInterceptTouchEvent、onTouchEvent以及View的dispatchTouchEvent、onTouchEvent，然后针对各个Event打Log，网上的很多关于Android事件分发的博文都是这么讲解的, 有兴趣可以自行查找。
通过下面的流程图，会更加清晰的帮助我们梳理事件分发机制：
![Android事件流程图](http://mmbiz.qpic.cn/mmbiz_png/y5HvXaQmpqnNkrnc8wQ9gAQcm76IKQPD7HDXqyRAGfepiahzaTJYx1K5KTEiaR0KgRWvmAIa7ylM6iaAGkvXMZvLg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)
<center> **View结构图**</center >
![](http://mmbiz.qpic.cn/mmbiz_png/y5HvXaQmpqnNkrnc8wQ9gAQcm76IKQPDO5VQXDR85NbaFHVGmKME4G3xMsXe0o74O0eRuMW6LHDjm2H17NqNiaQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)
<center> **View事件分发流程图**</center >
* 仔细看的话， 从上往下依次是Activity(Window)、ViewGroup(DecorView, RootViewGroup)、View， Activity是事件的起源。
*事件由Activity的dispatchTouchEvent做分发,  传递到ViewGroup的dispatchTouchEvent， onIntercepterTouchEvent，接着传递到View的dispatchTouchEvent。
* 如果事件不被消费，整个事件又会回传到顶层Activity
* dispatchTouchEvent 和 onTouchEvent 一旦return true,事件就停止传递了（到达终点）。
* dispatchTouchEvent 和 onTouchEvent return false的时候事件都回传给父控件的onTouchEvent处理。
* ViewGroup 想把自己分发给自己的onTouchEvent，需要拦截器onInterceptTouchEvent方法return true，而ViewGroup 的拦截器onInterceptTouchEvent 默认是不拦截的。
* View 没有拦截器，为了让View可以把事件分发给自己的onTouchEvent，View的dispatchTouchEvent默认实现（super）就是把事件分发给自己的onTouchEvent。


### OnTouch & OnTouchEvent的关系
其实在前面已经提到过，还是直接看源码
```java
   public boolean dispatchTouchEvent(MotionEvent event) {
        // 事件是否被消费
        boolean result = false;
        // 相关的监听器信息
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            // 检查事件被OnTouchListener的onTouch方法消费了
            result = true;
        }
        // 如果没有被执行，才执行onTouchEvent
        if (!result && onTouchEvent(event)) {
            result = true;
        }

        return result;
    }
```
onTouch是OnTouchListener的接口方法，它是用来获取Touch 信息用的，onTouchEvent方法用来处理诸如down, move, up的消息， 通过以上源码可以看出:
onTouchListener的onTouch方法优先级比onTouchEvent高，会先触发。
假如onTouch方法返回false会接着触发onTouchEvent，反之onTouchEvent方法不会被调用。
内置诸如click事件的实现等等都基于onTouchEvent，假如onTouch返回true，这些事件将不会被触发。

### OnTouchEvent & OnClick & OnLongClick的关系
还是来看下伪代码
```java
 public boolean onTouchEvent(MotionEvent event) {
    switch (action) {
       case MotionEvent.ACTION_UP:
         if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
            // This is a tap, so remove the longpress check
            removeLongPressCallback();
            if (!focusTaken) {
                if (mPerformClick == null) {
                    mPerformClick = new PerformClick();
                }
                if (!post(mPerformClick)) {
                    // 触发OnClickListener的onClick回调
                    performClick();
                }
            }
        }
        break:
      case MotionEvent.ACTION_DOWN:
        setPressed(true, x, y);
        checkForLongClick(0, x, y);
        break;
      case MotionEvent.ACTION_CANCEL:
        setPressed(false);
        removeTapCallback();
        removeLongPressCallback();
        break;
    }
 }
```
以上源码看一看出，onClick是在onTouchEvent ACTION_UP的时候出发的，如果该View的onTouchEvent或者它的ACTION_UP不被调用是不会触发onClick回调的。

关于onLongClick就稍微复杂一些，通过以上源码可以看到，当用户按下时（ACTION_DOWN）执行了checkForLongClick方法
```java
private void checkForLongClick(int delayOffset, float x, float y) {
        if ((mViewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) {
            mHasPerformedLongPress = false;

            if (mPendingCheckForLongPress == null) {
                mPendingCheckForLongPress = new CheckForLongPress();
            }
            mPendingCheckForLongPress.setAnchor(x, y);
            mPendingCheckForLongPress.rememberWindowAttachCount();
            postDelayed(mPendingCheckForLongPress,
                    ViewConfiguration.getLongPressTimeout() - delayOffset);
        }
    }
```
该方法实际就是创建了个CheckForLongPress 的 Runnable， 它的run方法里面就是performLongClick(OnLongClickListener的回调就是在该方法中执行的)，然后把它放入一个HandlerActionQueue队列中，让它延迟一段时间（DEFAULT_LONG_PRESS_TIMEOUT = 500）以后执行。
同时如果500毫秒之内，ACTION_UP或者ACTION_CANCEL执行了，就会移除该Runnable。


### 参考
> [可能是讲解Android事件分发最好的文章](https://mp.weixin.qq.com/s/rgQrJv8ghXO2HFt5Y5ISqA)
> [图解 Android 事件分发机制](http://www.jianshu.com/p/e99b5e8bd67b)
> [一文读懂Android View事件分发机制](https://mp.weixin.qq.com/s?__biz=MzIwMzYwMTk1NA==&mid=2247484615&idx=1&sn=d34d4035e6ecf03abbead56cd9eafa4c&chksm=96cda58aa1ba2c9c5663ff1f3e6a2ef72a64db81c813c74b7ebbc389fbcbb0ce3b3eaefe85a5&scene=21#wechat_redirect)
> [一文解决Android View滑动冲突](https://mp.weixin.qq.com/s?__biz=MzIwMzYwMTk1NA==&mid=2247484662&idx=1&sn=7b8a8831b37975936a9ea95c7a54d52a&chksm=96cda5bba1ba2cad32081316ad0771aab42fa64782f7b2c726acc2bb5809fb04f4ef7088ab29&mpshare=1&scene=23&srcid=0527RJ2k4z4mAQyPQqhbtYIG#rd)