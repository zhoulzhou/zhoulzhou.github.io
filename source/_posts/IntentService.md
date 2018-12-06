---
title: IntentService
date: 2018-12-06 09:56:30
tags:
- IntentService
categories:
- 多线程
---
#### The Zen of IntentService
默认的Service是执行在主线程的，可是通常情况下，这很容易影响到程序的绘制性能(抢占了主线程的资源)。除了前面介绍过的AsyncTask与HandlerThread，我们还可以选择使用IntentService来实现异步操作。IntentService继承自普通Service同时又在内部创建了一个HandlerThread，在onHandlerIntent()的回调里面处理扔到IntentService的任务。所以IntentService就不仅仅具备了异步线程的特性，还同时保留了Service不受主页面生命周期影响的特点。

{% asset_img in.png %}

如此一来，我们可以在IntentService里面通过设置闹钟间隔性的触发异步任务，例如刷新数据，更新缓存的图片或者是分析用户操作行为等等，当然处理这些任务需要小心谨慎。

**使用IntentService需要特别留意以下几点：**
>首先，因为IntentService内置的是HandlerThread作为异步线程，所以每一个交给IntentService的任务都将以队列的方式逐个被执行到，一旦队列中有某个任务执行时间过长，那么就会导致后续的任务都会被延迟处理。

>其次，通常使用到IntentService的时候，我们会结合使用BroadcastReceiver把工作线程的任务执行结果返回给主UI线程。使用广播容易引起性能问题，我们可以使用LocalBroadcastManager来发送只在程序内部传递的广播，从而提升广播的性能。我们也可以使用runOnUiThread()快速回调到主UI线程。

>最后，包含正在运行的IntentService的程序相比起纯粹的后台程序更不容易被系统杀死，该程序的优先级是介于前台程序与纯后台程序之间的。