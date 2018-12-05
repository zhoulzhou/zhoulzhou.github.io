---
title: HandlerThread
date: 2018-12-05 14:55:25
tags:
- HandlerThread
categories:
- ANDROID
---
#### HandlerThread是啥
通俗的来说就是一个子线程具有了 looper和handler这样的机制 ，当这个子线程创建了handler的时候，别的线程可以通过handler来发送信息，并且可以在这个handler里面执行耗时的操作。

#### 特别注意：
**1、HandlerThread不能更新ui**
**2、调用HandlerThread的handler.post(Runnable) 也是在子线程运行的**

#### 使用实例说明
正确使用是 我们仍然需要主线程的handler来更新ui，HandlerThread来接受其他子线程发送过来的消息，这些消息的处理不需要更新ui

```
public class MainActivity extends AppCompatActivity {
    private TextView textView;
    private Button   button;

    MainHandler mainHandler = new MainHandler(this);

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        textView = findViewById(R.id.text);
        button = findViewById(R.id.button);
        HandlerThread handlerThread = new HandlerThread("first");
        handlerThread.start();

		//这是子线程联系的handler 传入了handlerThread的looper
        final Handler childHandler = new Handler(handlerThread.getLooper(), new HandlerCallback()) {
            int count = 0;

            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                System.out.println("子线程 的handler 接受到数据  现在 将数据发送给 主线程 执行 " + Thread.currentThread().getName());
                 //发送消息给 主线程的handler 让她更新ui
				Message.obtain(mainHandler, 1, ++count).sendToTarget();
            }
        };
        
		// 开启一个子线程 调用childHandler 发送信息  
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {

                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        System.out.println("子线程发送数据 " + Thread.currentThread().getName());
						//发送给handlerThread的handler 
                        childHandler.sendEmptyMessage(1);
                    }
                }).start();
            }
        });
    }
    //这个是主线程的Handler 定义成这种形式，可以防止内存泄露
    static class MainHandler extends Handler {
        private WeakReference<MainActivity> mainActivityWeakReference;


        MainHandler(MainActivity mainActivity) {
            this.mainActivityWeakReference = new WeakReference<>(mainActivity);
        }

        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            mainActivityWeakReference.get().textView.setText(msg.obj + "");
        }
    }

    //在实例化处理程序时可以使用回调接口，以避免必须实现自己的处理程序子类。
    private class HandlerCallback implements Handler.Callback {
        @Override
        public boolean handleMessage(Message msg) {
            System.out.println("我是handler Callback " + Thread.currentThread().getName());
			//返回true意味着即使子类重写了handlerMessage方法 也不会被调用。
            //返回false  如果有子类的话，会调用handlerMessager方法
            return true;
        }
    }
}
```

#### 源码分析
HandlerThread的源码不多只有140多行，那就一步一步来分析吧，先来看看其构造函数

```
/**
 * Handy class for starting a new thread that has a looper. The looper can then be 
 * used to create handler classes. Note that start() must still be called.
 */
public class HandlerThread extends Thread {
    int mPriority;//线程优先级
    int mTid = -1;
    Looper mLooper;//当前线程持有的Looper对象
    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }

    /**
     * Constructs a HandlerThread.
     * @param name
     * @param priority The priority to run the thread at. The value supplied must be from 
     * {@link android.os.Process} and not from java.lang.Thread.
     */
    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }

    /**
     * Call back method that can be explicitly overridden if needed to execute some
     * setup before Looper loops.
     */
    protected void onLooperPrepared() {
    }
```

从源码可以看出HandlerThread继续自Thread,构造函数的传递参数有两个，一个是name指的是线程的名称，一个是priority指的是线程优先级，我们根据需要调用即可。其中成员变量mLooper就是HandlerThread自己持有的Looper对象。onLooperPrepared()该方法是一个空实现，是留给我们必要时可以去重写的，但是注意重写时机是在Looper循环启动前，再看看run方法：

```
Override
public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll(); //唤醒等待线程
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
   }
```

前面我们在HandlerThread的常规使用中分析过，在创建HandlerThread对象后必须调用其start()方法才能进行其他操作，而调用start()方法后相当于启动了线程，也就是run方法将会被调用，而我们从run源码中可以看出其执行了Looper.prepare()代码，这时Looper对象将被创建，当Looper对象被创建后将绑定在当前线程（也就是当前异步线程），这样我们才可以把Looper对象赋值给Handler对象，进而确保Handler对象中的handleMessage方法是在异步线程执行的。接着将执行代码：

```
synchronized (this) {
       mLooper = Looper.myLooper();
       notifyAll(); //唤醒等待线程
   }
```

这里在Looper对象创建后将其赋值给HandlerThread的内部变量mLooper，并通过notifyAll()方法去唤醒等待线程，最后执行Looper.loop();代码，开启looper循环语句。那这里为什么要唤醒等待线程呢？我们来看看，getLooper方法

```
public Looper getLooper() {
 //先判断当前线程是否启动了
   if (!isAlive()) {
       return null;
   }
   // If the thread has been started, wait until the looper has been created.
   synchronized (this) {
       while (isAlive() && mLooper == null) {
           try {
               wait();//等待唤醒
           } catch (InterruptedException e) {
           }
       }
   }
   return mLooper;
}
```

事实上可以看出外部在通过getLooper方法获取looper对象时会先先判断当前线程是否启动了，如果线程已经启动，那么将会进入同步语句并判断Looper是否为null，为null则代表Looper对象还没有被赋值，也就是还没被创建，此时当前调用线程进入等待阶段，直到Looper对象被创建并通过 notifyAll()方法唤醒等待线程，最后才返回Looper对象，之所以需要等待唤醒机制，是因为Looper的创建是在子线程中执行的，而调用getLooper方法则是在主线程进行的，这样我们就无法保障我们在调用getLooper方法时Looper已经被创建，到这里我们也就明白了在获取mLooper对象时会存在一个同步的问题，只有当线程创建成功并且Looper对象也创建成功之后才能获得mLooper的值，HandlerThread内部则通过等待唤醒机制解决了同步问题。

```
public boolean quit() {  
       Looper looper = getLooper();  
       if (looper != null) {  
           looper.quit();  
           return true;  
       }  
       return false;  
   }  
public boolean quitSafely() {  
    Looper looper = getLooper();  
    if (looper != null) {  
           looper.quitSafely();  
           return true;  
       }  
       return false;  
   }  
```

从源码可以看出当我们调用quit方法时，其内部实际上是调用Looper的quit方法而最终执行的则是MessageQueue中的removeAllMessagesLocked方法（Handler消息机制知识点），该方法主要是把MessageQueue消息池中所有的消息全部清空，无论是延迟消息（延迟消息是指通过sendMessageDelayed或通过postDelayed等方法发送）还是非延迟消息。 
当调用quitSafely方法时，其内部调用的是Looper的quitSafely方法而最终执行的是MessageQueue中的removeAllFutureMessagesLocked方法，该方法只会清空MessageQueue消息池中所有的延迟消息，并将消息池中所有的非延迟消息派发出去让Handler去处理完成后才停止Looper循环，quitSafely相比于quit方法安全的原因在于清空消息之前会派发所有的非延迟消息。最后需要注意的是Looper的quit方法是基于API 1，而Looper的quitSafely方法则是基于API 18的

**最后需要注意的是在我们不需要这个looper线程的时候需要手动停止掉；**

```
protected void onDestroy() {
        super.onDestroy();
        mHandlerThread.quit();
    }
```

#### 小提示

```
public Handler(Looper looper, Callback callback)
```
这个handler的构造方法中的callback 官方是这么说的

>在实例化处理程序时可以使用回调接口，以避免必须实现自己的处理程序子类。

简而言之呢，就是 你可以实现这个回调接口，这样你在实例化handler时候，就不要重写handleMessag()这个方法了。

#### 总结：
>HandlerThread本质上是一个Thread对象，只不过其内部帮我们创建了该线程的Looper和MessageQueue；
>通过HandlerThread我们不但可以实现UI线程与子线程的通信同样也可以实现子线程与子线程之间的通信；
>HandlerThread在不需要使用的时候需要手动的回收掉；
>比如，你同时有好几个子线程，你需要得到他们运算时候发出的信息，以前使用接口回调，还得实现的 ，如果 好几个子线程 岂不是要好几个接口。
>比如 还有 一个子线程运算完 你想知道结果，也可以通过这个来实现。以前你使用callable，接口的话 也还是 比较复杂一点的。



