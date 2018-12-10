---
title: Handler机制 MessageQueue源码分析
date: 2018-12-10 09:59:01
tags:
- MessageQueue
categories:
- Handler机制
---
MessageQueue属于低层类且依附于Looper，Looper外其他类不应该单独创建它，如果想使用MessageQueue可以从Looper类中得到它。

#### 消息队列存储原理
再上一章Message源码分析中我们知道了Message内部维持了一个链表缓存池来避免重复创建Message对象造成的额外消耗，以静态属性private static Message sPool作为缓存池链表头，Message next;作为链表的next指针。
有意思的是Message对象中next指针的不止用于链表缓存池，在MessageQueue中也采用同样的方法存储消息对象：

```
public final class MessageQueue {
    ......
    Message mMessages;
    ......
}
```

上面代码中的mMessages就是MessageQueue用来维持消息队列的链表头，至于它是如何存储的，后面再说。

#### 使用JNI实现的native方法
MessageQueue的源码调用了多个的C/C++方法，这类方法使用前都会用关键字native声明一下。
这些方法所属的底层C++代码创建了属于native层自己的NativeMessageQueue和NativeLooper消息模型。它们对Java层作用其实就是控制线程是否阻塞。
当我们想要从MessageQueue中取出消息时，碰巧队列是空的或即将取出的消息还没到被处理时间，那么我们就需要将线程阻塞掉等待队列中有消息时再取出。
下面就是MessageQueue中的native方法：

```
// 初始化
    private native static long nativeInit();
    // 注销
    private native static void nativeDestroy(long ptr);
    // 让线程阻塞timeoutMillis毫秒
    private native void nativePollOnce(long ptr, int timeoutMillis); 
    // 立刻唤醒线程
    private native static void nativeWake(long ptr);
    // 线程是否处于阻塞状态
    private native static boolean nativeIsPolling(long ptr);
    // 设置文件描述符
    private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);
```

#### 文件描述符与ePoll指令
消息队列里控制线程阻塞状态的native代码本质是用Linux指令ePoll完成的，在这之前需要先了解一点，Linux内核依靠“文件描述符（file descriptor）”来完成所有的文件读写访问操作，它更像是一个文件的索引值。而我们用到的ePoll指令就是用来监听文件描述符是否有可I/O操作的。（这都是一些Linux相关知识）

#### 创建与销毁
上面说了MessageQueue是依附于Looper的，所以本节分析的创建与销毁方法其实都是给Looper调用的，MessageQueue只提供了一个带参的构造方法来创建对象：
```
// 当前MessageQueue是否可以退出
    private final boolean mQuitAllowed;

    // native层中NativeMessageQueue队列指针的地址
    // mPtr等于0时表示退出队列
    private long mPtr;

    // native层代码，创建native层中NativeMessageQueue
    private native static long nativeInit();

    // 构造方法
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        // 执行native层方法
        mPtr = nativeInit();
    }
```
构造方法中boolean quitAllowed参数的意思是当前这个MessageQueue是否可以手动退出，为什么要控制能否手动退出呢？这里先说一个结论：Android 系统中要求UI线程不可手动退出，而其他Worker线程则全部都是可以的。（具体的操作在Looper和UI线程中）

#### 那么退出是什么意思呢？
退出就是当前这个MessageQueue停止服务，将队列中已存在的所有消息全部清空，看看源码中退出方法都做了什么：

```
// 是否已经退出了
    private boolean mQuitting;

    // native方法，退出队列
    private native static void nativeDestroy(long ptr);


    // 退出队列
    void quit(boolean safe) {
        
        // 如果不是可以手动退出的，抛出异常
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }

        synchronized (this) {
            // 如果已经退出了直接结束方法
            if (mQuitting) {
                return;
            }
            // 标记为已退出状态
            mQuitting = true;

            // 两种清除队列中消息的方法
            if (safe) {
                removeAllFutureMessagesLocked();
            } else {
                removeAllMessagesLocked();
            }

            // 注销
            nativeWake(mPtr);
        }
    }
```

方法quit(boolean safe)中的参数safe决定了到底执行哪种清除消息的方法：
>removeAllMessagesLocked()，简单暴力直接清除掉队列中所有的消息。
>removeAllFutureMessagesLocked()，清除掉可能还没有被处理的消息。

removeAllMessagesLocked()方法的逻辑很简单，从队列头中取消息，有一个算一个，全部拿出来回收掉。：

```
private void removeAllMessagesLocked() {
        Message p = mMessages;
        while (p != null) {
            Message n = p.next;
            p.recycleUnchecked();
            p = n;
        }
        mMessages = null;
    }
```

然而，removeAllFutureMessagesLocked()方法的逻辑稍微多一点：

```
private void removeAllFutureMessagesLocked() {
    
        // 获取当前系统时间
        final long now = SystemClock.uptimeMillis();
        Message p = mMessages;
        if (p != null) {
            // 判断当前消息对象的预处理时间是否晚于当前时间
            if (p.when > now) {
                // 如果当前消息对象的预处理时间晚于当前时间直接全部暴力清除
                removeAllMessagesLocked();
            } else {
                Message n;
                // 如果当前消息对象的预处理时间并不晚于当前时间
                // 说明有可能这个消息正在被分发处理
                // 那么就跳过这个消息往后找晚于当前时间的消息
                for (;;) {
                    n = p.next;
                    if (n == null) {
                        return;
                    }
                    if (n.when > now) {
                        // 如果找到了晚于当前时间的消息结束循环
                        break;
                    }
                    p = n;
                }
                p.next = null;
                do {
                    // n就是那个晚于当前时间的消息
                    // 从n开始之后的消息全部回收
                    p = n;
                    n = p.next;
                    p.recycleUnchecked();
                } while (n != null);
            }
        }
    }
```

这么看来这个方法名字起的还挺靠谱的，很好的解释了是要删除还没有被处理的消息。

#### 消息入队管理enqueueMessage()方法

```
// 消息入队
    // 参数when就是此消息应该被处理的时间
    boolean enqueueMessage(Message msg, long when) {
        // 如果此消息的target也就是宿主handler是空的抛异常
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }

        // 如果此消息是in-use状态抛异常，in-use的消息不可拿来使用
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        //上锁
        synchronized (this) {

            // 如果当前MessageQueue已经退出了抛异常并释放掉此消息
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }
            
            // 将消息标记为in-use状态
            msg.markInUse();
            // 设置应该被处理的时间
            msg.when = when;
            // 拿到队列头
            Message p = mMessages;


            // 是否需要唤醒线程
            boolean needWake;

            // p等于空说明队列是空的
            // when等于0表示强制把此消息插入队列头部，最先处理
            // when小于队列头的when说明此消息应该被处理的时间比队列中第一个要处理的时间还早
            // 以上情况满足任意一种直接将消息插入队列头部
            if (p == null || when == 0 || when < p.when) {
                // 将消息插入队列头部
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                
                // 线程已经被阻塞&&消息存在宿主Handler&&消息是异步的
                needWake = mBlocked && p.target == null && msg.isAsynchronous();

            
                //如果上述条件都不满足就要按照消息应该被处理的时间插入队列中    
                Message prev;
                for (;;) {
                    // 两根相邻的引用一前一后从队列头开始依次向后移动
                    prev = p;
                    p = p.next;
                    // 如果队列到尾部了或者找到了处理时间早于自身的消息就结束循环
                    if (p == null || when < p.when) {
                        break;
                    }

                    // 如果入队的消息是异步的而排在它前面的消息有异步的就不需要唤醒
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                // 将新消息插在这一前一后两个引用中间，完成入队
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // 判断是否需要唤醒线程
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```

总结一下消息入队的逻辑大致分为如下几步：

1、检查消息合法性，包括宿主target是否为空，是否为in-use状态，队列是否还存活。
2、如果满足条件【队列为空、when等于0、此消息应被处理的时间比队列中第一个要处理的时间还早】中的任意一个直接将此消息插在队列头部最先被处理。
3、如果以上三个条件均不满足，那么就从头遍历队列根据被处理时间找到它的位置。

#### 同步消息拦截器
除了enqueueMessage()方法可以向队列中添加消息外，还有一个postSyncBarrier()方法也可以向队列添加消息，但它不是添加普通的消息，我们将它添加的特殊Message称为同步消息拦截器。
顾名思义，该拦截器只会影响同步消息。复习一下上节中分析到的东西，我们默认发送的消息都是同步的，只有某个Message被调用了setAsynchronous(true)后才是异步消息。同步消息受队列限制依次有序的等待处理，异步消息也不受限制。
消息拦截器与普通消息的差异在于拦截器的target是空的，正常我们通过enqueueMessage()方法入队的消息由于限制target是不能为空的。

```
// 标识拦截器的token
    private int mNextBarrierToken;

    private int postSyncBarrier(long when) {

        synchronized (this) {
            // 得到拦截器token
            final int token = mNextBarrierToken++;
            // 实例化一个消息对象
            final Message msg = Message.obtain();
            // 将对象设置为in-use状态
            msg.markInUse();
            // 设置时间
            msg.when = when;
            // 将token存于消息的常用属性arg1中
            msg.arg1 = token;

            Message prev = null;
            Message p = mMessages;

            // 如果when不等于0就在队列中按时间找到它的位置
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            // 如果prev不等于空就把拦截器插入
            // 如果prev等于空直接插入队列头部
            if (prev != null) { // invariant: p == prev.next
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
          
            // 拦截器入队成功，返回对应token
            return token;
        }
    }
```

总体来说添加拦截器的方法跟正常消息入队差不多，值得一提的就是Message的target是空的，然后arg1保存着拦截器的唯一标识token。

token的作用是找到对应的拦截器删除，看看删除拦截器的方法。

```
public void removeSyncBarrier(int token) {
        synchronized (this) {
            Message prev = null;
            Message p = mMessages;
            // 遍历队列找到指定拦截器
            // 查找条件：target为空，arg1等于指定token值
            while (p != null && (p.target != null || p.arg1 != token)) {
                prev = p;
                p = p.next;
            }
            // 如果p等于空说明没找到
            if (p == null) {
                throw new IllegalStateException("The specified message queue synchronization "
                        + " barrier token has not been posted or has already been removed.");
            }

            // 是否需要唤醒线程
            final boolean needWake;

            // 在队列中移除掉拦截器
            if (prev != null) {
                prev.next = p.next;
                // 如果prev不等于空说明拦截器前面还有别的消息，就不需要唤醒
                needWake = false;
            } else {
                mMessages = p.next;
                // 拦截器在队列头部，移除它之后如果队列空了或者它的下一个消息是个正常消息就需要唤醒
                needWake = mMessages == null || mMessages.target != null;
            }

            // 回收
            p.recycleUnchecked();

            // 判断是否需要唤醒
            if (needWake && !mQuitting) {
                nativeWake(mPtr);
            }
        }
    }
```

#### 队列空闲处理器IdleHandler
于在从队列中取出消息时队里可能是空的，这时候就会阻塞线程等待消息到来。每次队列中没有消息而进入的阻塞状态，我们叫它为“空闲状态”。
讲道理实际使用中队列空闲状态的情况还是很常见的，为了更好的利用资源，也为了更好的掌握线程的状态，开发人员就设计了这么一个“队列空闲处理器IdleHandler”。
IdleHandler是MessageQueue类下的一个子接口，只包含了一个方法：

```
public static interface IdleHandler {
        /**
         * 当线程的MessageQueue等待更多消息时会调用该方法。
         *
         * 返回值：true代表只执行一次，false代表会一直执行它
         */
        boolean queueIdle();
    }
```

MessageQueue为我们提供了添加和删除IdleHandler的方法：

```
//使用一个ArrayList存储IdleHandler
   private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();

   // 添加一个IdleHandler
   public void addIdleHandler(@NonNull IdleHandler handler) {
        if (handler == null) {
            throw new NullPointerException("Can't add a null IdleHandler");
        }
        synchronized (this) {
            mIdleHandlers.add(handler);
        }
    }

    // 删除一个IdleHandler
    public void removeIdleHandler(@NonNull IdleHandler handler) {
        synchronized (this) {
            mIdleHandlers.remove(handler);
        }
    }

```

#### 消息出队管理next()方法

next()方法很长，先大致看一下源码：

```
private IdleHandler[] mPendingIdleHandlers;

    Message next() {

        // mPtr是从native方法中得到的NativeMessageQueue地址
       // 如果mPtr等于0说明队列不存在或被清除掉了
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        // 待处理的IdleHandler数量，因为代表数量，所以只有第一次初始化时为-1
        int pendingIdleHandlerCount = -1;


        // 线程将被阻塞的时间
        // -1：一直阻塞
        // 0：不阻塞
        // >0:阻塞nextPollTimeoutMillis 毫秒
        int nextPollTimeoutMillis = 0;


        // 开始死循环，下面的代码都是在循环中，贼长！
        for (;;) {

            // 如果nextPollTimeoutMillis 不等于0说明要阻塞线程了
            if (nextPollTimeoutMillis != 0) {
                // 为即将长时间阻塞做准备把该释放的对象都释放了
                Binder.flushPendingCommands();
            }

            // 阻塞线程操作
            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // 得到当前时间
                final long now = SystemClock.uptimeMillis();

                Message prevMsg = null;
                Message msg = mMessages;

                // 判断队列头是不是同步拦截器
                if (msg != null && msg.target == null) {
                    // 如果是拦截器就向后找一个异步消息
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
    
                // 判断队列是否有可以取出的消息
                if (msg != null) {

                    if (now < msg.when) {
                        // 如果待取出的消息还没有到应该被处理的时间就让线程阻塞到应该被处理的时间
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // 直接就能取出消息，所以不用阻塞线程
                        mBlocked = false;

                        将消息从队列中剥离出来
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        // 让消息脱离队列
                        msg.next = null;

                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        
                        // 设置为in-use状态
                        msg.markInUse();
                        // 返回取出的消息，结束循环，结束next()方法
                        return msg;
                    }
                } else {
                    // 队列中没有可取出的消息，nextPollTimeoutMillis 等于-1让线程一直阻塞
                    nextPollTimeoutMillis = -1;
                }

                // 如果队列已经退出了直接注销和结束方法
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // IdleHandler初始化为-1，所以在本循环中该条件成立的次数 <= 1
                if (pendingIdleHandlerCount < 0 && (mMessages == null || now < mMessages.when)) {
                    // 得到IdleHandler的数量
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }

                // pendingIdleHandlerCount 小于或等于0说明既没有合适的消息也没有合适的闲时处理
                if (pendingIdleHandlerCount <= 0) {
                    // 直接进入下次循环阻塞线程
                    mBlocked = true;
                    continue;
                }

                // 代码执行到此处就说明线程中有待处理的IdleHandler
                // 那么就从IdleHandler集合列表中取出待处理的IdleHandler
                if (mPendingIdleHandlers == null) {
                    // 初始化待处理IdleHandler数组，最小长度为4
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
    
                // 从IdleHandler集合中获取待处理的IdleHandler
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }
          
            // ==========到此处同步代码块已经结束==========


            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                // 取出一个IdleHandler
                final IdleHandler idler = mPendingIdleHandlers[i];
                // 释放掉引用
                mPendingIdleHandlers[i] = null; 

                // IdleHandler的执行模式，true=执行一次，false=总是执行
                boolean keep = false;

                try {
                    // 执行IdleHandler的queueIdle()代码，得到执行模式
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                // 通过执行模式判断是否需要移除掉对应的IdleHandler
                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // 处理完了所有IdleHandler把数量清0
            pendingIdleHandlerCount = 0;

            // 因为执行了IdleHandler的代码块，有可能已经有新的消息入队了
            // 所以到这里就不阻塞线程，直接去查看有没有新消息
            nextPollTimeoutMillis = 0;
        }
    }

```

消息出队的核心代码的逻辑都在一个庞大的死循环for(;;)中，其流程如下：
0，循环开始。
1，根据nextPollTimeoutMillis值阻塞线程，初始值为0：不阻塞线程。
2，将【待取出消息指针】指向队列头。
3，如果队列头是同步拦截器的话就将【待取出消息指针】指向队列头后面最近的一个异步消息。
4，如果【待取出消息指针】不可用（msg == null）说明队列中没有可取出的消息，让nextPollTimeoutMillis 等于-1让线程一直阻塞，等待新消息到来时唤醒它。
5，如果【待取出消息指针】可用（msg != null）再判断一下消息的待处理时间。

  如果消息的待处理时间大于当前时间（now < msg.when）说明当前消息还没到要处理的时间，让线程阻塞到消息待处理的指定时间。
  如果消息的待处理时间小于当前时间（now > msg.when）就直接从队列中取出消息返回给调用处。（此处会直接结束整个循环，结束next()方法。）

6，如果队列已经退出了直接结束next()方法。
7，如果是第一次循环就初始化IdleHandler数量的局部变量pendingIdleHandlerCount 。
8，如果IdleHandler数量小于等于0说明没有合适的IdleHandler，直接进入下次循环阻塞线程。（此处会直接结束本次循环。）
9，初始化IdleHandler数组，里面保存着本地待处理的IdleHandler。
10，遍历IdleHandler数组，执行对应的queueIdle()方法。
11，执行完所有IdleHandler之后，将IdleHandler数量清0。
12，因为执行了IdleHandler的代码块，有可能已经有新的消息入队了， 所以让nextPollTimeoutMillis 等于0不阻塞线程，直接去查看有没有新消息。
13，本次循环结束，开始新一轮循环。

#### 总结
>MessageQueue队列消息是有序的，按消息待处理时间依次排序。
>同步拦截器可以拦截它之后的所有同步消息，直到这个拦截器被移除。
>取出消息时如果没有合适的消息线程会阻塞。