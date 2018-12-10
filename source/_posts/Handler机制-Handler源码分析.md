---
title: Handler机制 Handler源码分析
date: 2018-12-10 13:56:47
tags:
- Handler
- Handler内存泄漏
categories:
- Handler机制
---
Handler本身可在多线程之间调用，不管它在哪个线程发送消息，都会回到它被初始化的哪个线程中接收到消息。

#### 初始化
Handler有7个构造方法，分别对应不同的参数来初始化不同的Handler属性，但是真正完成初始化操作的只有两个构造方法：

```
// 是否需要查找潜在的漏洞
    private static final boolean FIND_POTENTIAL_LEAKS = false;

    /**
     * 将Callback接口作为构造方法参数，可以用作接收消息的回调
     * 这样就可以省去自己重写Handler自身的handleMessage方法
     *
     * @param msg 接收到的消息
     * @return 是否需要进一步处理，即调用Handler自身的handleMessage方法
     */
    public interface Callback {
        public boolean handleMessage(Message msg);
    }

    final Looper mLooper;
    final MessageQueue mQueue;
    final Callback mCallback;

    // 发送的消息是否为异步的，默认是false
    final boolean mAsynchronous;

    /**
     * @hide 隐藏的构造方法，外部不可见
     */
    public Handler(Callback callback, boolean async) {

        // 检查是否存在内存泄漏的可能
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        // 得到当前线程的Looper
        mLooper = Looper.myLooper();

        // 如果Looper还没初始化抛出异常
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }

        // 得到当前线程的MessageQueue
        mQueue = mLooper.mQueue;

        
        mCallback = callback;
        mAsynchronous = async;
    }

 
    public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

从上面代码的得知，构造方法初始化的工作就是给mLooper,mQueue,mCallback和mAsynchronous这几个关键的属性赋值.mLooper和mQueue自然就是当前线程下的Looper和MessageQueue了，如果传递了Looper参数就直接赋值，如果没传递就调用Looper.myLooper();得到当前线程的Looper。

mCallback是Handler内部定义的一个简单接口，其目的是为了替代传统的接收消息方法。当然使用mCallback的同时并不会影响正常的Handler消息分发。此处解释从后面接收消息时的逻辑就可以看到。

mAsynchronous的意思是该Handler发送的消息是否是异步的，从前面Message源码的文章中我们知道Message中有一个设置消息是否为异步消息的方法，MessageQueue对异步消息的处理也与同步消息不同。此处如果设置了mAsynchronous为true，那么这个Handler发送的所有消息就都是异步消息。

在有两个参数的构造方法中我们会发现有一段检查是否存在内存泄漏的代码，为什么会这样呢？在分析完发送消息和接收消息后再说这个。

当然，除了上面两个构造方法外还有其它几个构造方法，但均是调用上面两个方法：

```
public Handler() {
        this(null, false);
    }

    public Handler(Callback callback) {
        this(callback, false);
    }

    public Handler(Looper looper) {
        this(looper, null, false);
    }

    public Handler(Looper looper, Callback callback) {
        this(looper, callback, false);
    }

    /**
     * @hide 隐藏的构造方法，外部不可见
     */
    public Handler(boolean async) {
        this(null, async);
    }
```

#### 发送消息
使用Handler发送消息时我们知道它分为两类：

>postXXX()方法切换回原线程。
>sendMessageXXX()方法发送消息到原线程。

其实这两种方法本质都是发送一个Message对象到原线程，只不过PostXXX()方法是发送了一个只有Runnable callback属性的Message对象。
先来看一下sendMessageXXX()类的方法：

```
// 发送一条普通消息
    public final boolean sendMessage(Message msg){
        return sendMessageDelayed(msg, 0);
    }

    // 发送一条空消息
    public final boolean sendEmptyMessage(int what){
        return sendEmptyMessageDelayed(what, 0);
    }

    // 发送一条空的延时消息
    public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageDelayed(msg, delayMillis);
    }

    // 发送一条空的定时消息
    public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageAtTime(msg, uptimeMillis);
    }

    // 发送一个普通的延时消息
    public final boolean sendMessageDelayed(Message msg, long delayMillis) {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

    // 发送一个普通的定时消息
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        //得到消息队列
        MessageQueue queue = mQueue;
        // 如果消息队列是空的记录日志然后结束方法
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        // 执行消息入队操作
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```

从上面的代码中发现，不管调用何种发送消息的方法，最后真正调用的都是sendMessageAtTime()方法。而真正发送的核心方法也就是入队方法是Handler的enqueueMessage()方法。

```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        // 将消息的宿主设置为当前Handler自身
        msg.target = this;

        //如果Handler被设置成了异步就把消息也设置成异步的
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }

        // 执行消息队列的入队操作
        return queue.enqueueMessage(msg, uptimeMillis);
    }

```

看到这里终于明白了为啥Looper和MessageQueue一直在使用Message的target属性而我们却从来没有给它赋值过，是Handler在发送消息前自己赋值上去的。

看完了发送消息类的方法在看看切换线程类的方法干了什么：

```
private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }

    private static Message getPostMessage(Runnable r, Object token) {
        Message m = Message.obtain();
        m.obj = token;
        m.callback = r;
        return m;
    }

    public final boolean post(Runnable r){
       return  sendMessageDelayed(getPostMessage(r), 0);
    }
    
    public final boolean postAtTime(Runnable r, long uptimeMillis) {
        return sendMessageAtTime(getPostMessage(r), uptimeMillis);
    }
    
    public final boolean postAtTime(Runnable r, Object token, long uptimeMillis){
        return sendMessageAtTime(getPostMessage(r, token), uptimeMillis);
    }
    
    public final boolean postDelayed(Runnable r, long delayMillis){
        return sendMessageDelayed(getPostMessage(r), delayMillis);
    }
    
    public final boolean postAtFrontOfQueue(Runnable r){
        return sendMessageAtFrontOfQueue(getPostMessage(r));
    }
```

上本节开头讲的一样，postXXX()类方法就是构造了一个只有Runnable callback的Message对象，然后走正常发送消息的方法。唯一有一个特例就是postAtFrontOfQueue()方法，它调用了sendMessageAtFrontOfQueue()方法是之前发送消息没有用到过的：

```
public final boolean sendMessageAtFrontOfQueue(Message msg) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, 0);
    }
```

原来这个方法特殊的地方就是在入队的时候时间参数为0，我们在MessageQueue源码知道如果入队消息的时间参数为0那么这个消息会被直接放在队列头。所以，postAtFrontOfQueue()方法就是直接在消息队列头部插入了一个消息。

#### 接收消息
在Looper 的源码中我们知道每当从MessageQueue中取出一个消息时就会调用这个消息的宿主target中分发消息的方法：

```
// Looper分发消息
msg.target.dispatchMessage(msg);
```

而这个宿主target也就是我们的Handler，所有Handler接收消息就是在这个dispatchMessage()方法中了：

```
public void dispatchMessage(Message msg) {
        // 如果Message的callback不为空，说明它是一个通过postXXX()方法发送的消息
        if (msg.callback != null) {
            // 直接运行这个callback
            handleCallback(msg);
        } else {
            //如果mCallback 不为空说明Handler设置了Callback接口
            // 先执行接口处理消息的方法
            if (mCallback != null) {
                // 如果callback接口处理完消息返回true说明它将消息拦截
                // 不再执行Handler自身的处理消息方法，直接结束方法
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            // 调用Handler自身处理消息的方法
            handleMessage(msg);
        }
    }

    private static void handleCallback(Message message) {
        // 运行callback，也就是这个Runnable接口
        message.callback.run();
    }

    // Handler自身处理消息的方法，开发者需要重新该方法来实现接收消息
    public void handleMessage(Message msg) {
    }
```

由此可见，如果是单纯的PostXXX()方法发送的消息，Handler接收到了之后直接运行Message对象的Runnable接口，不会将它当做一个消息进行处理。

而我们的mCallback接口是完全可以替代Handler自身接收消息的方法，因为其高优先处理等级，它甚至可以选择拦截掉Handler自身的接收消息方法。

#### 内存泄漏的可能
我们在使用Handler的时候写法一般如下：

```
private final Handler handler = new Handler(){

        @Override
        public void handleMessage(Message msg) {

        }
    };
```

由于重写了handleMessage()方法相当于生成了一个匿名内部类，也就相当于如下代码：

```
private final Handler handler = new MyHandler ();

    class MyHandler extends Handler{

        @Override
        public void handleMessage(Message msg) {
            
        }
    }

```

可是你有没有想过内部类凭什么能够调用外部类的属性和方法呢？答案就是**内部类隐式的持有着外部类的引用，编译器在创建内部类时把外部类的引用传入了其中，只不过是你看不到而已。**

既然Handler作为内部类持有着外部类（多数情况为Activity）的引用，而Handler对应的一般都是耗时操作。当我们在子线程执行一项耗时操作时，用户退出程序，Activity需要被销毁，而Handler还在持有Activity的引用导致无法回收，就会引发内存泄漏。

#### 解决方法分为两步

1、生成内部类时把内部类声明为静态的。
2、使用弱引用来持有外部类引用。

**静态内部类不会持有外部类的引用，且弱引用不会阻止JVM回收对象。**

```
static class MyHandler extends Handler{

        WeakReference<Activity> mActivity ;

        public MyHandler(Activity activity){
            mActivity = new WeakReference<Activity>(activity);
        }

        @Override
        public void handleMessage(Message msg) {
            Activity activity = mActivity.get();

            if (activity == null){
                return;
            }

            // do something

        }
    }
```
