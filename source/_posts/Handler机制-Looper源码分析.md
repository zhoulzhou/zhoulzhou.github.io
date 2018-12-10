---
title: Handler机制 Looper源码分析
date: 2018-12-10 13:40:08
tags:
- Looper
- Handler
categories:
- Handler机制
---
Looper的职责很单一，就是单纯的从MessageQueue中取出消息分发给消息对应的宿主Handler，因此它的代码不多（300行左右）。

Looper是线程独立的且每个线程只能存在一个Looper。

Looper会根据自己的存活情况来创建和退出属于它自己的MessageQueue。

#### 创建与退出Looper
上面的结论中提到了Looper是线程独立的且每个线程只能存在一个Looper。所以构造Looper实例的方法类似于单例模式。隐藏构造方法,对外提供了两个指定的获取实例方法prepare()和prepareMainLooper()。

```
// 应用主线程（UI线程）Looper实例
    private static Looper sMainLooper;

    // Worker线程Looper实例，用ThreadLocal保存的对象都是线程独立的
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

    // 与当前Looper对应的消息队列
    final MessageQueue mQueue;

    // 当前Looper所以的线程
    final Thread mThread;

    /**
     * 对外公开初始化方法
     *
     * 在普通线程中初始化Looper调用此方法
     */
    public static void prepare() {
        // 初始化一个可以退出的Looper
        prepare(true);
    }

    /**
     * 对外公开初始化方法
     *
     * 在应用主线程（UI线程）中初始化Looper调用此方法
     */
    public static void prepareMainLooper() {
        
        // 因为是主线程，初始化一个不允许退出的Looper
        prepare(false);

        synchronized (Looper.class) {
            // 如果sMainLooper不等于空说明已经创建过主线程Looper了，不应该重复创建
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }

    /**
     * 内部私有初始化方法
     * @param quitAllowed 是否允许退出Looper
     */
    private static void prepare(boolean quitAllowed) {
        // 每个线程只能有一个Looper
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        // 保存实例
        sThreadLocal.set(new Looper(quitAllowed));
    }

    /**
     * 私有构造方法
     * @param quitAllowed 是否允许退出Looper
     */  
    private Looper(boolean quitAllowed) {
        // 初始化MessageQueue
        mQueue = new MessageQueue(quitAllowed);
        // 得到当前线程实例
        mThread = Thread.currentThread();
    }
```

真正创建Looper实例的构造方法中其实很简单，就是创建了对应的MessageQueue实例，然后得到当前线程，值得注意的是MessageQueue和线程实例都是被final关键字修饰的，只能被赋值一次。

对外公开初始化方法prepareMainLooper()是为应用主线程（UI线程）准备的，应用刚被创建就会调用该方法，所以我们不该再去调用它。

开发者可以通过调用对外公开初始化方法prepare()对自己的worker线程创建Looper，但是要注意只能初始化一次。

调用Looper.prepare()方法初始化完成后，可以调用myLooper()和myQueue()方法得到当前线程对应的实例。

```
public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }

    public static @NonNull MessageQueue myQueue() {
        return myLooper().mQueue;
    }
```

#### 退出Looper

退出Looper有安全与不安全两种退出方法，其实对应的就是MessageQueue的安全与不安全方法：

```
  public void quit() {
        mQueue.quit(false);
    }

    public void quitSafely() {
        mQueue.quit(true);
    }
```

什么安全退出，什么是不安全退出，在MessageQueue源码中分析过。

#### 运行Looper处理消息

调用Looper.prepare()方法初始化完成Looper后就可以让Looper去工作了，只需要调用Looper.loop()方法即可。

```
public static void loop() {
        // 得到当前线程下的Looper
        final Looper me = myLooper();

        // 如果还没初始化过抛异常
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }

         // 得到当前线程下与Looper对应的消息队列
        final MessageQueue queue = me.mQueue;

        // 得到当前线程的唯一标识（uid+pid），作用是下面每次循环都判断一下线程有没有被切换
        // 不知道为什么要调用两次该方法
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        // 进入死循环不断取出消息
        for (;;) {

            // 从队列中取出一个消息，这可能会阻塞线程
            Message msg = queue.next(); 

            // 如果消息是空的，说明队列已经退出了，直接结束循环，结束方法
            if (msg == null) {
                return;
            }

            // 打印日志
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            // 性能分析相关的东西
            final long traceTag = me.mTraceTag;
            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }

            try {
                //尝试将消息分发给宿主（Handler）
                //dispatchMessage为宿主Handler的接收消息方法
                msg.target.dispatchMessage(msg);
            } finally {
                 // 性能分析相关的东西
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }

            //打印日志
            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }


            //得到当前线程的唯一标识
            final long newIdent = Binder.clearCallingIdentity();

            //如果本次循环所在的线程与最开始不一样，打印日志记录
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }
      
            //消息分发完毕，回收消息到缓存池
            msg.recycleUnchecked();
        }
    }
```

#### 总结
Looper的功能很简单，核心方法Looper.loop()就是不断的从消息队列中取出消息分发给对应的宿主Handler，它与对应MessageQueue息息相关，一起创建，一起退出。

Looper更想强调的是线程的独立性与唯一性，利用ThreadLocal保证每个线程只有一个Looper实例的存在。利用静态构造实例方法保证不能重复创建Looper。

Looper.prepareMainLooper()是比较特殊的方法，它是给UI线程准备，理论上开发者在任何情况下都不应该调用它。
