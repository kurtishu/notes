---
title: Handler 消息机制源码分析
date: 2017-04-07 10:34:20
categories: [Android, 源码分析]
tags: [Handler, Looper, Message]
---

### 概述
概述Handler 、 Looper 、Message 这三者都与Android异步消息处理线程相关的概念。那么什么叫异步消息处理线程呢？
异步消息处理线程启动后会进入一个无限的循环体之中，每循环一次，从其内部的消息队列中取出一个消息，然后回调相应的消息处理函数，执行完成一个消息后则继续循环。若消息队列为空，线程则会阻塞等待。
说了这一堆，那么和Handler 、 Looper 、Message有啥关系？其实Looper负责的就是创建一个MessageQueue，然后进入一个无限循环体不断从该MessageQueue中读取消息，而消息的创建者就是一个或多个Handler 。
接下来从源码级别分析他们之间的关系。

### 源码分析

#### Handler构造方法
```java
    // 无参构造方法
    public Handler() {
        this(null, false);
    }
	
	// 一个参数Callback的构造方法，callback是处理消息的回调方法
	// public boolean handleMessage(Message msg)
	public Handler(Callback callback) {
        this(callback, false);
    }
	
	// 一个参数带Looper，传入自定义looper对象，一般使用默认的是mainLooper
	public Handler(Looper looper) {
        this(looper, null, false);
    }
```
所有的消息都会封装成Message对象,并添加到消息队列中去。
```java
   private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        // 关键点：target对象指的是handler本身，target就是将来处理消息的对象。
		// 也就是说消息最终还是会回到Handler中来处理。
		msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

#### 发送消息方法
Handler发送消息最终都会先组装成一个Message对象，然后将Message对象放入
Looper对象的MessageQuene队列中，Looper通过调用loop()不断的去检测消息队列中是否有消息需要处理，调用的是MessageQueue的next()方法，这个方法返回的是MessageQueue的存储的消息.
```java
    // 发送一个runnable，runnable最终会组装成一个Message对象，
	//该runnable赋值给Message的callback成员变量
    public final boolean post(Runnable r) {
       return sendMessageDelayed(getPostMessage(r), 0);
    }
	
	// 发送一个Message对象，直接将其放入MessageQuenue中
	public final boolean sendMessage(Message msg) {
        return sendMessageDelayed(msg, 0);
    }
	
	// 发送一个空消息，该消息最终还是会组装成一个Message对象，
	// 该对象的what值被传入参数赋值
	public final boolean sendEmptyMessage(int what) {
        return sendEmptyMessageDelayed(what, 0);
    }
```

#### Looper对象分析
先看看Looper对象中几个重要成员变量
```java
    // sThreadLocal.get() will return null unless you've called prepare().
   //保证单个线程中只有一个Looper对象
   static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
   
   // 主线程的Looper
   private static Looper sMainLooper;  // guarded by Looper.class

   // Message消息队列
   final MessageQueue mQueue;
```

Looper对象初始化
```java
    // 在当前线程中初始化一个Looper对象，
	public static void prepare() {
        prepare(true);
    }
	
	//  prepareMainLooper 由ActivityThread的main方法中调用Looper.prepareMainLooper()，生成一个主线程的Looper
	// 所以主线程中默认就有mainLooper，但是子线程中必须手动调用该方法初始化一个Looper对象。
	public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }

```
** 消息抽取的方法，也是最重要的方法 **  删除了无用代码，留下核心代码方便理解。
```java
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
		// 通过当前Looper对象获取消息队列
        final MessageQueue queue = me.mQueue;

		// 此处是个死循环，也就是说一有消息来就会处理
        for (;;) {
		    // 调用MessageQueue的next()方法，返回MessageQueue的存储的消息
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
 
            try {
			    // 回调target的dispatchMessage方法来分发消息。
				//target指的是handler，也就是说looper抽取消息转交给hanlder来处理。
                msg.target.dispatchMessage(msg);
            } finally {
            }
			// 回收已经处理过的消息对象
            msg.recycleUnchecked();
        }
    }
```

来看下dispatchMessage方法，看下Handler类中的该方法源码
```java
    /**
     * Handle system messages here.
     */
    public void dispatchMessage(Message msg) {
	    // 上面提到callback属性是runnable对象，所以要单独处理
        if (msg.callback != null) {
		    // 调用runnable的run方法
            handleCallback(msg);
        } else {
		    // 如何构造方法传入了mCallback对象，就调用它的handleMessage
            if (mCallback != null) {
			    // 如何handleMessage已经消费了该消息，就结束处理否则交给Handler的handleMessage方法处理
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
			// 通过继承Handler类，在其子类中实现该方法可以接受到消息。
            handleMessage(msg);
        }
    }
```

#### 带有Delay的message
前面我们都知道message都会调用enqueueMessage放入MessageQueue队列中

```java
	boolean enqueueMessage(Message msg, long when) {
        synchronized (this) {

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
	
```

然后通过next方法获取message， next方法中判断当前时间可delay的时间差来判断是否要执行该message

```java

    Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
				    // 检查当前时间和message开始时间
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```

### 总结
以上是通过源码级别分析了Handler、Looper、Message Queue之间的关系。关于Handler更多认识，会不断更新。

---

了解更多
[Android开发：Handler异步通信机制全面解析（包含Looper、Message Queue）](http://www.jianshu.com/p/9fe944ee02f7)





