---
title: AsynTask说明及注意事项
date: 2018-12-06 09:44:36
tags:
- AsynTask
categories:
- 多线程
---
#### Good AsyncTask Hunting
syncTask是一个让人既爱又恨的组件，它提供了一种简便的异步处理机制，但是它又同时引入了一些令人厌恶的麻烦。一旦对AsyncTask使用不当，很可能对程序的性能带来负面影响，同时还可能导致内存泄露。

举个例子，常遇到的一个典型的使用场景：用户切换到某个界面，触发了界面上的图片的加载操作，因为图片的加载相对来说耗时比较长，我们需要在子线程中处理图片的加载，当图片在子线程中处理完成之后，再把处理好的图片返回给主线程，交给UI更新到画面上。

{% asset_img asy.png %}

AsyncTask的出现就是为了快速的实现上面的使用场景，AsyncTask把在主线程里面的准备工作放到onPreExecute()方法里面进行执行，doInBackground()方法执行在工作线程中，用来处理那些繁重的任务，一旦任务执行完毕，就会调用onPostExecute()方法返回到主线程。

{% asset_img asy1.png %}

使用AsyncTask需要注意的问题有哪些呢？请关注以下几点：
>首先，默认情况下，所有的AsyncTask任务都是被线性调度执行的，他们处在同一个任务队列当中，按顺序逐个执行。假设你按照顺序启动20个AsyncTask，一旦其中的某个AsyncTask执行时间过长，队列中的其他剩余AsyncTask都处于阻塞状态，必须等到该任务执行完毕之后才能够有机会执行下一个任务。情况如下图所示：

{% asset_img asy2.png %}

为了解决上面提到的线性队列等待的问题，我们可以使用AsyncTask.executeOnExecutor()强制指定AsyncTask使用线程池并发调度任务。

{% asset_img asy3.png %}

>其次，如何才能够真正的取消一个AsyncTask的执行呢？我们知道AsyncTaks有提供cancel()的方法，但是这个方法实际上做了什么事情呢？线程本身并不具备中止正在执行的代码的能力，为了能够让一个线程更早的被销毁，我们需要在doInBackground()的代码中不断的添加程序是否被中止的判断逻辑，如下图所示：

{% asset_img asy4.png %}

一旦任务被成功中止，AsyncTask就不会继续调用onPostExecute()，而是通过调用onCancelled()的回调方法反馈任务执行取消的结果。我们可以根据任务回调到哪个方法（是onPostExecute还是onCancelled）来决定是对UI进行正常的更新还是把对应的任务所占用的内存进行销毁等。

>最后，使用AsyncTask很容易导致内存泄漏，一旦把AsyncTask写成Activity的内部类的形式就很容易因为AsyncTask生命周期的不确定而导致Activity发生泄漏。

{% asset_img asy5.png %}

综上所述，AsyncTask虽然提供了一种简单便捷的异步机制，但是我们还是很有必要特别关注到他的缺点，避免出现因为使用错误而导致的严重系统性能问题。

