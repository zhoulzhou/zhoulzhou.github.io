---
title: Handler机制-为何要有handler
date: 2018-12-08 15:22:32
tags:
- Handler
categories:
- Handler机制
---
#### 设计Handler 的初衷
在分析Handler之前，需要先搞清楚两个概念：
>同步与异步的区别
>线程与多线程的概念

#### Java多线程通信
Java中有很多种方法实现线程之间相互通信访问数据，大概先简单的介绍两个典型的，就不上代码了。

通过synchronized关键字以“上锁”机制实现线程间的通信。多个线程持有同一个对象，他们可以访问同一个共享变量，利用synchronized“上锁”机制，哪个线程拿到了锁，它就可以对共享变量进行修改，从而实现了通信。
使用Object类的wait/notify机制，执行代码obj.wait();后这个对象obj所在的线程进入阻塞状态，直到其他线程调用了obj.notify();方法后线程才会被唤醒。

#### Android多线程的特殊性
在上面的两个Java多线程通信的方法中都有一个共同的特点，那就是线程的阻塞。利用synchronized机制拿不到锁的线程需要等拿到锁了才会继续执行操作，obj.wait();需要等obj.notify();才会继续执行操作。
虽然Android系统是由Java封装的，但是由于Android系统的特殊性，Google的开发人员对Android线程的设计进行了改造。他们把启动APP时运行的主线程定义为UI线程。
UI线程负责所有你能想到的所有的跟界面相关的操作，例如分发绘制事件，分发交互事件等可多了。由于其特殊性Android系统强制要求以下两点：

为保持用户界面流畅UI线程不能被阻塞，如果线程阻塞界面会卡死，若干秒后Android系统抛出ANR。
除UI线程外其他线程不可执行UI操作。

（此处只是简单介绍一下UI线程，后面会有专门一节分析Android UI线程。）

#### Android 多线程通信
既然UI线程中不能被阻塞，那么查询数据库和访问网络这类的耗时操作肯定就不能在UI线程中执行了，我们就需要单独开个线程操作。
但是除UI线程外其他线程又不可执行UI操作，最后还是要回到UI线程更新UI，这就需要多线程之间的通信。
可Java中线程间通信又都是阻塞式方法，所以传统的Java多线程通信方式在Android中并不适用。
为此Google开发人员就不得不设计一套UI线程与Worker线程通信的方法。既能实现多线程之间的通信，又能完美解决UI线程不能被阻塞的问题。具体方法有以下几类：

>view.post(Runnable action)系列,通过View对象引用切换回UI线程。
>activity.runOnUiThread(Runnable action)，通过Activity对象引用切换回UI线程。
>AsyncTask，内部封装了UI线程与Worker线程切换的操作。
>Handler，本文的主角，异步消息处理机制，多线程通信。

#### 小结
说到了这里应该大概明白了当初设计Handler的初衷。
由于Android系统的特殊性创造了UI线程。
由于UI线程的特殊性创造了若干个UI线程与Worker线程通信的方法。
在这若干个线程通信方法中就包含了Handler。
Handler就是针对Android系统中与UI线程通信而专门设计的多线程通信机制。


#### Handler实现原理 - 理论分析
关于源码分析的文章我真是有一肚子吐槽的话想说，总的来说就是介绍源码的作者嗨的不行，而看文章的人一脸懵逼。吸取了前辈们的教训，这里分析源码前先来一波理论上的分析。

#### 线程中接收消息端的特殊性
首先我们得知道理想状态下使用Handler是希望它被实例化在哪个线程，哪个线程就是消息的接收端，虽然在其他线程内发送消息时调用的同样是这个Handler的引用。这没错吧？

{% asset_img ha.webp %}

{% asset_img ha1.webp %}

根据上面的结论可以知道Handler接收消息端是线程独立的，不管handler的引用在哪个线程发送消息都会传回自己被实例化的那个线程中。
但显而易见的是Handler不可能是线程独立的，因为它的引用会在别的线程作为消息的发送端，也就是说它本身就是多线程共享的引用，不可能独立存在于某个线程内。
所以！Handler需要一个**独立存在于线程内部且私有使用的类**帮助它接收消息！这个类就是Looper！

#### Looper - 线程独立
通过上节分析我们已经知道设计Looper就是为了辅助Handler接收消息且仅独立于线程内部。那如何才能实现线程独立的呢？
好消息是Java早就考虑到了这一点，早在JDK 1.2的版本中就提供ThreadLocal这么一个工具类来帮助开发者实现线程独立。这里简单分析一下ThreadLocal的使用方法，不分析实现原理了。Android官网 - ThreadLocal API
ThreadLocal 支持泛型，用于定义线程私有化变量的类型，实例化对象时可选Override一个初始化方法initialValue()，这个方法的作用就是给你的引用变量赋初始值，如果没有Override这个方法那么默认你的引用变量就是null的：

```
//定义一个线程私有的String类型变量
    private static final ThreadLocal<String> local = new ThreadLocal<String>(){
        
        // 设置引用变量的初始化值
        @Override
        protected String initialValue() {
            return super.initialValue();
        }
    };

```

定义好了ThreadLocal之后还需要了解三个方法：
>get() 得到你的本地线程引用变量。
>set(T value)为你的本地线程引用变量赋值。
>remove() 删除本地线程引用变量。

是不是很简单呢？有了ThreadLocal之后我们只需要把Looper存进去就能实现线程独立了。

```
private static final ThreadLocal<Looper> mLooper = new ThreadLocal<Looper>();
```

到这里再梳理一下流程：

1、Handler 引用可以多线程间共享。
2、当Handler对象在其他线程发送消息时，通过Handler的引用找到它所在线程的Looper接收消息。
3、Looper 负责接收消息再分发给Handler的接收消息方法。

{% asset_img ha2.webp %}

但是！这样还会有一个问题，如果多个线程同时使用一个Handler发消息，Looper该怎么办？给接收消息的方法上锁吗？显然不能这样做啊！于是就设计了MessageQueue来解决这个问题。

#### MessageQueue - 多线程同时发消息
为了防止多个线程同时发送消息Looper一下着忙不过来，于是设计一个MessageQueue类以队列的方式保存着待发送的消息，这样Looper就可以一个个的有序的从MessageQueue中取出消息处理了。
既然MessageQueue是为Looper服务的，而Looper又是线程独立的，所以MessageQueue也是线程独立的。

{% asset_img ha3.webp %}

#### 小结
现在我们已经知道为了完成异步消息功能需要有Handler家族的四位成员共同合作：


>Handler： 负责发送消息，为开发者提供发送消息与接收消息的方法。
>Message： 消息载体，负责保存消息具体的数据。
>MessageQueue：消息队列，以队列形式保存着所有待处理的消息。
>Looper：消息接受端，负责不断从MessageQueue中取出消息分发给Handler接受消息端。

这四位成员哪个都不是平白无故出现的。因为要规范化消息传递格式而定义了Message；为了实现消息接收端只存在线程内部私有化使用而定义了Looper；为了解决多线程同时发送数据Looper分发消息处理时会产生的问题而设计MessageQueue队列化消息。
到这里你应该知道了Handler家族四位成员各自负责的是什么工作，以及他们自身的特点特殊性，比如Handler是线程间共享的而Looper是线程独立的，MessageQueue跟Looper又是一对一的。
