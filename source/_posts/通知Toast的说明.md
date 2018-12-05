---
title: 通知Toast的说明
date: 2018-12-05 15:39:10
tags:
- 通知
- Toast
categories:
- ANDROID
---
下面在子线程中可以发出Toast消息，却不能更新UI(setText) why?

```
Looper.prepare();
Handler inner = new Handler() {
    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
        System.out.println("内部handler 里面" + Thread.currentThread().getName() + "     ");
        Toast.makeText(MainActivity.this,"haahh",Toast.LENGTH_LONG).show();
        //textView.setText(">>>>>>>>");
    }
};
inner.sendEmptyMessage(1);
Looper.loop();
```
**运行 你会发现toast弹出了！！！！**
这是怎么回事了？首先，我们看toast的show方法

{% asset_img toast.png %}

看我划红线的地方。service服务。我们看一下 getService()方法

{% asset_img toast1.png %}

我们可以看见 返回了 INotificationManager对象的静态实例。那么这个INotificationManager是什么呢。看上面的英文注释。翻译一下的意思是：

>下面的所有内容都是与通知服务的交互，通知服务处理这些系统范围内的正确排序。

这个呢 我们可以发现通知服务，其实到这里呢，应该有人就明白了。Android 有一个专门的后台服务Service 来进行Toast 等一些通知的显示 弹出。 而我们知道在Service里面 我们是可以弹出Toast的。所以 ，这个说明了 一些通知信息的显示，都是由后台的 Service来承担的。我们在前台 只是一个通知作用

#### Android-在子线程中显示Toast和Dialog

Android中有句话说，只能在主线程（UI线程）中更新UI，这是因为Android的主线程（UI线程）是不安全的。所以在子线程如果要显示Toast或者Dialog，我们需要通知主线程来显示 ，有两种方法可以解决此问题：
（1）在UI代码的前后加上Loop.prepare()和Loop.loop();例如： 
```
Looper.prepare(); 
showExitDialog(App.getInstance().getCurrentActivity()); 
Looper.loop(); 
```
（2）通过handler消息来创建，具体方法是创建一个handler，然后在子线程中发送一个Message消息，在handler收到消息后创建Toast或Dialog.

注： 
1、Looper用来为一个线程开启一个消息循环。默认情况下，子线程是有没有Looper的。Looper通过MessageQueue来存放消息和事件，一个线程只能有一个Looper，对应一个MessageQueue。 
2、在子线程中直接new Handler（）会报错，原因是没有创建Looper，需先用Looper.prepare启用Looper。 
3、Looper.loop()让Looper开始工作。

#### 子线程中mProgressBar.setProgress(progress);
Thread中执行了 这么一句
>mProgressBar.setProgress(progress);

且执行正常，progressbar确实一直在更新。
顿觉疑惑，View在更新时，会检查当前线程是否是创建view所在的线程（即UI线程），若不一致，则会抛出异常的。
in ViewRootImpl.java 中：
```
void checkThread() {
    if (mThread != Thread.currentThread()) {
        throw new CalledFromWrongThreadException(
                "Only the original thread that created a view hierarchy can touch its views.");
    }
}
```

后来查看了setProgress（）的源码后，才恍然大悟，这个方法内已经处理了子线程里调用的情况了。

```

@android.view.RemotableViewMethod
    public synchronized void setProgress(int progress) {
        setProgress(progress, false);
    }
 
    @android.view.RemotableViewMethod
    synchronized void setProgress(int progress, boolean fromUser) {
        if (mIndeterminate) {
            return;
        }
 
        if (progress < 0) {
            progress = 0;
        }
 
        if (progress > mMax) {
            progress = mMax;
        }
 
        if (progress != mProgress) {
            mProgress = progress;
            refreshProgress(R.id.progress, mProgress, fromUser);
        }
    }
 
    private synchronized void refreshProgress(int id, int progress, boolean fromUser) {
        if (mUiThreadId == Thread.currentThread().getId()) {
            doRefreshProgress(id, progress, fromUser, true);
        } else {
            if (mRefreshProgressRunnable == null) {
                mRefreshProgressRunnable = new RefreshProgressRunnable();
            }
 
            final RefreshData rd = RefreshData.obtain(id, progress, fromUser);
            mRefreshData.add(rd);
            if (mAttached && !mRefreshIsPosted) {
                post(mRefreshProgressRunnable);
                mRefreshIsPosted = true;
            }
        }
    }

```

关键就是refreshProgress（）了，这里处理了若当前线程不是ui thread，则将更新消息post到ui thread中去执行了。而且可以发现这几个方法都是加了同步控制的，是线程安全的，保证了多线程调用也是正常的了。

真的是“源码之下无秘密“啊。

p.s：
之前一次面试的时候，对方有问到能否在子线程里更新界面？
我回到不能，因为会抛出异常。对方接着问，为什么？
我就答不上来了，现在的理解是，因为android下的界面更新不是线程安全的，所以要保证在单一线程中同步执行。其他线程要更新UI的话，需要通过handler，Looper消息机制把更新事情pass到UI thread的消息队列中，由UI thread来完成界面的更新绘制。
面试官又继续问道，能否在子线程里去更新progressbar的进度呢？
我当时回答的是不能，必须通过handler去发消息。但现在看来progressbar的代码后，这个问题的答案其实应该是可以的，因为progressbar自己内部就处理了子线程的更新问题。
