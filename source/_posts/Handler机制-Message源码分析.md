---
title: Handler机制-Message源码分析
date: 2018-12-08 15:42:08
tags:
- handler
- message
categories:
- Handler机制
---
#### Message 源码分析
本文出现的源码版本均为Android 7.1.1(Nougat) - API 25 版本。

Message作为消息传递的载体，源码主要分为以下几个部分：

1、操作数据相关，类似getter()和setter()这种方法还有之前提到过的what和obj这类属性。
2、创建与回收对象实例相关，除了用关键字new外，其他得到对象实例的方法。
3、其他工具类性质的扩展方法。

#### Message中的数据属性与方法
首先说一个本篇文章忽略的属性及相关方法：public Messenger replyTo;。
为什么要忽略过去呢？因为Messenger类是基于Message上实现进程间通信的类。注意，是进程间通信，不是线程间通信。一方面进程间通信不是本文分析的重点，另一方面进程间通信需要掌握AIDL方面的知识。
接下来就让我们看看Message源码有哪些可供我们使用的属性吧：

>public int what;：开发者可自定义的消息标识代码，用于区分不同的消息。
>public int arg1;：如果要传递的消息只有少量的integer型数据，可以使用这个属性。
>public int arg2;：同上面arg1。
>public Object obj;开发者可自定义类型的传输数据。

上面四个属性作为常用的消息传递的数据载体可直接赋值，例如msg.arg1 = 100;。基本可以满足我们日常开发中简单消息传递。

如果上面几个数据属性不能满足我们的需求，可以使用扩展数据：Bundle来传递（Bundle是啥应该都知道吧？）：

```
Bundle data;

    // 得到Bundle数据，如果data是空的就new一个
    public Bundle getData() {
        if (data == null) {
            data = new Bundle();
        }
        return data;
    }

    // 得到Bundle数据，如果data是空的就返回 null
    public Bundle peekData() {
        return data;
    }
    // 设置Bundle数据
    public void setData(Bundle data) {
        this.data = data;
    }

```

这段代码也没什么逻辑好分析的，值得一提就是Bundle data不是public的，所以我们不能直接操作这个属性，需要通过上面三个方法操作数据。使用Bundle数据也非常简单：

```
Bundle bundle = new Bundle();
    bundle.putString("String","value");
    bundle.putFloat("float",0.1f);
        
    Message msg = Message.obtain();
    msg.setData(bundle);
```

#### 创建与回收Message对象的基本方法
先看一下源码中Meesage的构造方法：

```
/** Constructor (but the preferred way 
        to get a Message is to call {@link #obtain() Message.obtain()}).
    */
    public Message() {
    }

```

没错，这货的构造方法里什么也没有，不过它的注释却告诉我们想要得到Message对象首选的方法应该是调用静态方法Message.obtain()。那这个obtain()方法干了什么呢？其实就是**内部维持了一个链表形式的Meesage对象缓存池，**这样会节省重复实例化对象产生的开销成本。
老样子还是理论分析一波，数据结构中的链表一个单元有两个值，当前单元的值（head）和下一个单元的地址指针（next），如果下一个单元不存在那么next就是null的。

{% asset_img mes.webp %}

所以，想要实现Message对象链表式缓存池就需要额外的两个Message类型的引用head和next，都说了叫缓存池，所以把head叫pool更合适一点。
有了链表的基础结构我们再想实例化对象的时候就可以先去链表缓存池中看看有没有，有的话直接从缓存池中拿出来用，没有再new一个。

{% asset_img mes1.webp %}

由于代码多起来逻辑有些复杂，这样不太好分析，所以我在源码中加了许多自己的注释，下面代码看上去很长，其实把注释都去掉后并没有多少。

```
// 用于标识当前对象是否存在于缓存池，0代表不在缓存池中
    int flags;

    /** 
     * 这个常量是供上面的 flags 使用的，它表示in use（正在使用）状态
     *
     * 如果Message对象被存入了MessageQueue消息队列排队等待Looper处
     * 理或者被回收到缓存池中等待重复利用时，那么它就是in use（正在使用）状态
     * 
     * 只有在new Message()和Message.obtain()时候才可以清除掉flags上的in use状态
     *
     * 你不可以让一个in use状态的Message对象去传递消息。
     *
     *  1<< 0 还是1，真不知道为啥要这么写，直接写等于1不就得了
     */
    static final int FLAG_IN_USE = 1 << 0;

    /** 静态常量对象，通过synchronized (sPoolSync)让它作为线程并发操作时的锁
     * 确保同一时刻只有一个线程可以访问当前对象的引用
     */
    private static final Object sPoolSync = new Object();
    
    // 当前链表缓存池的入口，装载着缓存池中第一个可用的对象
    private static Message sPool;

    // 链表缓存池中指向下一个对象引用的next指针
    Message next;
    
    // 当前链表缓存池中对象的数量
    private static int sPoolSize = 0;

    /**
     * 从缓存池中拿出来一个Message对象给你
     * 可以让我们在许多情况下避免分配新对象。
     */
    public static Message obtain() {
        // 上锁，这期间只有一个线程可以执行这段代码
        synchronized (sPoolSync) {
            // pool不等于空就说明缓存池中还有可用的对象，直接取出来
            if (sPool != null) {
                // 声明一个Message引用指向缓存池中的pool对象
                Message m = sPool;
                // 让缓存池中pool引用指向它的next引用的对象
                sPool = m.next;
                // 因为该对象已经从缓存池中被取出，所以将next指针置空
                m.next = null;
                // 将从缓存池中取出的对象的flags的in use标识清除掉
                m.flags = 0; 
                // 缓存池中Message对象数量减去一个
                sPoolSize--;
                return m;
            }
        }
        // 如果缓存池中没有可用的对象就new一个吧
        return new Message();
    }

```

理论上我们希望sPool引用指向了链表缓存池中的第一个对象，让它作为整个缓存池的出入口。所以我们把它设置成static的，这样它就与实例化出来的对象无关，也就是说无论我们在哪个Message对象中进行操作，sPool还是sPool。

#### 静态方法obtain()的代码逻辑流程：
先判断缓存池是不是空的：if(sPool != null),如果是空的就直接：return new Message();，不是空的就声明一个引用让它指向缓存池第一个对象：Message m = sPool;，而缓存池的链表头部sPool引用就指向了链表中下一个对象：sPool = m.next;，因为这个时候缓存池中第一个对象已经取出交给了引用Message m，所以需要清除掉这个对象身上的特殊标识，包括缓存池中的next引用和用来标记对象状态的flags值：m.next = null; m.flags = 0;,最后将缓存池中的对象数量减一：sPoolSize--;。
逻辑理清了整个流程就显得很简单了，再看看图解逻辑流程：

{% asset_img mes2.webp %}

分析到这里我们知道了为什么官方推荐我们使用Message.obtain()得到对象了，因为它是在缓存池中取出来重复利用的，但是通过上面代码也看可以看到，只有缓存池里有东西时也就是sPool != null的时候才可以取，Message是怎么把对象回收到缓存池中的呢？

#### 回收Message对象到缓存池的方法
阅读源码后发现有一个public void recycle()方法用于回收Message对象，但是它也牵扯出了一堆其他方法与属性：

```
// 缓存池最大存储值
    private static final int MAX_POOL_SIZE = 50;

    // 区分当前Android版本是否大于或者等于LOLLIPOP版本的全局静态变量，默认初始值为true
    private static boolean gCheckRecycle = true;

    /**
     *  用于区分当前Android版本是否大于或者等于LOLLIPOP版本
     *  内部隐藏方法，在APP启动时就会执行该方法，开发者是不可见的
     *  @hide
     */
    public static void updateCheckRecycle(int targetSdkVersion) {
        if (targetSdkVersion < Build.VERSION_CODES.LOLLIPOP) {
            gCheckRecycle = false;
        }
    }

    /**
     * 判断当前对象的flags是否为in-use状态
     */
    boolean isInUse() {
        return ((flags & FLAG_IN_USE) == FLAG_IN_USE);
    }

    /**
     * 调用这个方法后，当前对象就会被回收入缓存池中。
     * 你不能回收一个在MessageQueue排队等待处理或者正在交付给Handler处理的Message对象
     * 说白了就是in-use状态的不可回收
     */
    public void recycle() {
        // 判断当前对象是否为in-use状态
        if (isInUse()) {
            // 如果当前版本大于或者等于LOLLIPOP则抛出异常
            if (gCheckRecycle) {
                throw new IllegalStateException("This message cannot be recycled because it "
                        + "is still in use.");
            }
            // 如果当前版本小于LOLLIPOP什么也不干直接结束方法
            return;
        }
        // 回收Message对象
        recycleUnchecked();
    }

    /**
     * 回收一个可能是in use状态的Message对象
     * 在MessageQueue和Looper内部处理排队Message时也会使用这个方法
     */
    void recycleUnchecked() {
        // 将当前Message对象置为in-use状态
        flags = FLAG_IN_USE;

        // 清除当前Message对象的所有数据属性
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        target = null;
        callback = null;
        data = null;

        // 上锁
        synchronized (sPoolSync) {
            // 如果当前缓存池对象中的数量小于缓存池最大存储值（50）就存入缓存池中
            if (sPoolSize < MAX_POOL_SIZE) {
                // 存入缓存池
                next = sPool;
                sPool = this;
                // 缓存池数量加1
                sPoolSize++;
            }
        }
    }

```

上面代码的逻辑很清晰，执行recycle()方法后先判断当前对象是否为in-use状态：if (isInUse())，如果是in-use状态的话当前Android版本是LOLLIPOP(5.0)版本之前直接结束程序，LOLLIPOP及之后版本抛出异常。如果当前对象不是in-use状态，那么就执行recycleUnchecked()方法先将它切换到in-use状态：flags = FLAG_IN_USE;，再把所有的数据属性全部清除，最后把对象存入缓存池链表中。

{% asset_img mes6.webp %}

**为什么要区分Android LOLLIPOP(5.0)前后版本？**
源码刚开始就有两个用于区分Android版本的全局属性和方法：

private static boolean gCheckRecycle = true;
public static void updateCheckRecycle(int targetSdkVersion)

通过查看源码发现Message类在LOLLIPOP版本进行了一次更新也就是我们现在看到的源码，在LOLLIPOP版本之前虽然recycle()方法的注释上同样警告了我们不能回收in-use对象，但是如果你坚持让in-use状态的对象调用recycle()的话也会也会被回收：

```
/**
     * Android LOLLIPOP版本源码
     *
     * Return a Message instance to the global pool.  You MUST NOT touch
     * the Message after calling this function -- it has effectively been
     * freed.
     */
    public void recycle() {
        // 清除数据
        clearForRecycle();
        // 存入缓存池
        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
```

所以在LOLLIPOP版本的时候Google进行了改进，强制要求不可以回收in-use状态的对象否则抛出异常，但是为了兼容之前的版本，所以新增加了个内部私有的区分Android版本的方法。

#### 我们需要手动回收吗
现在我们知道了通过执行recycle()方法回收Message对象，但是如果要为每个Message对象都进行手动回收岂不是很麻烦？
庆幸的是开发人员也想到了这一点，从源码中可以看到其实最终真正执行回收操作的调用recycleUnchecked()方法，且注释中告诉我们MessageQueue和Looper内部也会调用该方法执行回收。
这里先说一个结论，MessageQueue和Looper内部分发处理消息时，当它们得知当前这个Message对象已经使用完毕后就会直接调用recycleUnchecked()方法将它回收掉，等分析到MessageQueue和Looper再具体讲这个地方。
所以，如果我们用实例化Message对象是放入Handler中去传消息的，那么我们就不需要手动回收，他们内部自己就回收了。如果我们使用的Message对象跟Handler，Looper，MessageQueue一点交互都没有，那我们就自己去回收。

#### 包含Handler参数的obtain()方法
给Message内部装了一个Handler起了什么作用呢？首先，我们可以通过上面讲解我们可以得出以下已知的结论：

1、表面上看我们使用Handler发送消息后，消息直接传回到了Handler内部的handleMessage(Message msg)方法中。
2、实际上是先把消息传入了MessageQueue中，Looper再从MessageQueue依次取出消息分发给Handler。
3、Looper是线程独立的， Looper和MessageQueue是一对一的。

但是，你有没有想过Looper和Handler是不是一对一的？答案当然是否定的，MessageQueue只负责队列消息，Looper只负责取出消息分发。他们的功能很明确而且通用。
所以，无论当前线程有多少个Handler，同样都只有一个Lopper和一个MessageQueue。

一个线程存在多个Handler

既然每个线程只有一个Looper和MessageQueue的话那么Looper分发消息的时候要如何判断当前这个Message是哪个Handler的呢？所以开发人员就给Message内部配置了一个Handler属性，这样Looper分发消息时直接调用Messgae内部的Handler属性就能找到它对应的handleMessage(Message msg)接收消息的方法了。
源码很简单，就是在空参方法obtain()基础上加了个Handler属性，还有它的getter()和setter()：

```
Handler target;

    public static Message obtain(Handler h) {
        Message m = obtain();
        m.target = h;
        return m;
    }

    public void setTarget(Handler target) {
        this.target = target;
    }

    public Handler getTarget() {
        return target;
    }
```

#### 包含Runnable参数的obtain()方法

跟上面类似，该方法就是在上面基础加了个Runnable参数，源码如下：

```
Runnable callback;

    public static Message obtain(Handler h, Runnable callback) {
        Message m = obtain();
        m.target = h;
        m.callback = callback;
        return m;
    }

    public Runnable getCallback() {
        return callback;
    }

```

这个Runnable的作用是：在Looper分发消息时如果Runnable callback不是空的，那么就不调用Handler的handleMessage(Message msg)方法，直接运行这个Runnable callback。注意，这里的运行是已经回到了Handler被创建的线程上，也就是说Runnable会运行在Handler被创建的线程上。

更多包含参数的obtain()方法

下面这些带参的obtain()方法我相信不用介绍大家也都能看的懂：

```
public static Message obtain(Handler h, int what) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        return m;
    }

    public static Message obtain(Handler h, int what, Object obj) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        m.obj = obj;
        return m;
    }

    public static Message obtain(Handler h, int what, int arg1, int arg2) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        m.arg1 = arg1;
        m.arg2 = arg2;
        return m;
    }

    public static Message obtain(Handler h, int what, 
            int arg1, int arg2, Object obj) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        m.arg1 = arg1;
        m.arg2 = arg2;
        m.obj = obj;
        return m;
    }

```

#### 扩展方法
**特殊属性long when;**
```
  long when;

    public long getWhen() {
        return when;
    }
```

既然这个属性的名字都叫when了那肯定就是跟时间有关了。还记得Handler给我们提供的方法中有几个可以控制时间的方法吗？例如XXXAtTime()和XXXDelayed()，when这个属性就是就用存储当前这个Message应该被处理的时间。当我们讲Handler和MessageQueue时会在提到它。

#### 序列化对象
Message支持对象的序列化，就是可以把对象转为字节形式，可以保存到本地也可以用于网络传输。如果不了解这方面的知识建议先查阅相关文章。
为了实现对象序列化，我们需要实现Parcelable接口，实例化Parcelable.Creator接口，并重写describeContents()和writeToParcel()方法。先看源码，再讲他们都是干啥的。

```
// 实现Parcelable接口
public final class Message implements Parcelable {

    ...

    // 实例化Parcelable.Creator接口，完成Parcel对象转Message对象的操作
    public static final Parcelable.Creator<Message> CREATOR
            = new Parcelable.Creator<Message>() {

        public Message createFromParcel(Parcel source) {
            Message msg = Message.obtain();
            msg.readFromParcel(source);
            return msg;
        }
        
        public Message[] newArray(int size) {
            return new Message[size];
        }
    };

    // 序列化对象时的特殊种类对象描述，这里开发人员没有修改，就是默认的0
    public int describeContents() {
        return 0;
    }

    // 重写Parcelable接口的writeToParcel方法，将Message对象转为Parcel对象，
    public void writeToParcel(Parcel dest, int flags) {
       
         // 以下代码均是将Message对象中的属性写入Parcel对象中

        if (callback != null) {
            throw new RuntimeException(
                "Can't marshal callbacks across processes.");
        }
        dest.writeInt(what);
        dest.writeInt(arg1);
        dest.writeInt(arg2);
        if (obj != null) {
            try {
                Parcelable p = (Parcelable)obj;
                dest.writeInt(1);
                dest.writeParcelable(p, flags);
            } catch (ClassCastException e) {
                throw new RuntimeException(
                    "Can't marshal non-Parcelable objects across processes.");
            }
        } else {
            dest.writeInt(0);
        }
        dest.writeLong(when);
        dest.writeBundle(data);
        Messenger.writeMessengerOrNullToParcel(replyTo, dest);
        dest.writeInt(sendingUid);
    }
    
    // 从Parcel对象中读取数据转为当前对象的属性
    private void readFromParcel(Parcel source) {
        what = source.readInt();
        arg1 = source.readInt();
        arg2 = source.readInt();
        if (source.readInt() != 0) {
            obj = source.readParcelable(getClass().getClassLoader());
        }
        when = source.readLong();
        data = source.readBundle();
        replyTo = Messenger.readMessengerOrNullFromParcel(source);
        sendingUid = source.readInt();
    }

}
```

代码看上去很长，其实很简单，就是实现了Parcelable接口，接着重写将Message对象转为Parcel对象的方法writeToParcel()，再重写了接口Parcelable.Creator<T>完成Parcel对象转Message对象的方法。
这里用到的主要都是序列化对象Parcelable接口相关的知识。

#### 设置Message是异步传输还是同步传输
正常情况下，我们的消息其实是同步处理的，为什么这么说呢？
Looper的工作就是把消息队列MessageQueue中的消息依次取出然后分发，每个消息传输都是有时间顺序的，这个动作都是可控制的。
然而，将消息设置成异步传输后那么Message对象将不再受Looper的控制，传输的顺序可能会被打断，不一定哪个消息先传过来。
所以，请谨慎使用异步传输。

```
// 该常量代表为异步传输方式
    static final int FLAG_ASYNCHRONOUS = 1 << 1;

    // 判断是否为异步传输
    public boolean isAsynchronous() {
        return (flags & FLAG_ASYNCHRONOUS) != 0;
    }

    // 设置当前对象是否为异步传输
    public void setAsynchronous(boolean async) {
        if (async) {
            flags |= FLAG_ASYNCHRONOUS;
        } else {
            flags &= ~FLAG_ASYNCHRONOUS;
        }
    }

```
值得一提的是，用于标记是否为异步传输的标识跟用于判断是否为in-use状态的标识是共用的一个属性flags。