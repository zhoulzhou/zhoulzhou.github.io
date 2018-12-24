---
title: LoaderManager用法
date: 2018-12-19 13:36:48
tags:
- LoaderManager
categories:
- 多线程
---
#### 一.基本概念
 **1.LoaderManager**
 LoaderManager用来负责管理与Activity或者Fragment联系起来的一个或多个Loaders对象.
 每个Activity或者Fragment都有唯一的一个LoaderManager实例(通过getLoaderManager()方法获得),用来启动,停止,保持,重启,关闭它的Loaders,这些功能可通过调用initLoader()/restartLoader()/destroyLoader()方法来实现.
 LoaderManager并不知道数据如何装载以及何时需要装载.相反,它只需要控制它的Loaders们开始,停止,重置他们的Load行为,在配置变换或数据变化时保持loaders们的状态,并使用接口来返回load的结果.

 **2.Loader**
 Loades负责在一个单独线程中执行查询,监控数据源改变,当探测到改变时将查询到的结果集发送到注册的监听器上.Loader是一个强大的工具,具有如下特点
 (1)它封装了实际的数据载入.
 Activity或Fragment不再需要知道如何载入数据.它们将该任务委托给了Loader,Loader在后台执行查询要求并且将结果返回给Activity或Fragment.
 (2)客户端不需要知道查询如何执行.Activity或Fragment不需要担心查询如何在独立的线程中执行,Loder会自动执行这些查询操作.
 这种方式不仅减少了代码复杂度同事也消除了线程相关bug的潜在可能.
 (3)它是一种安全的事件驱动方式.
 Loader检测底层数据,当检测到改变时,自动执行并载入最新数据.
 这使得使用Loader变得容易,客户端可以相信Loader将会自己自动更新它的数据.
 Activity或Fragment所需要做的就是初始化Loader,并且对任何反馈回来的数据进行响应.除此之外,所有其他的事情都由Loader来解决.

#### 二.异步Loader的实现原理
 (1)执行异步载入的任务.为了确保在一个独立线程中执行载入操作,Loader的子类必须继承AsyncTaskLoader<D>而不是Loader<D>类.
 AsyncTaskLoader<D>是一个抽象Loader,它提供了一个AsyncTask来做它的执行操作.
 当定义子类时,通过实现抽象方法loadInBackground方法来实现异步task.该方法将在一个工作线程中执行数据加载操作.
 (2)在一个注册监听器中接收载入完成返回的结果.
 对于每个Loader来说,LoaderManager注册一个OnLoadCompleteListener<D>,该对象将通过调用onLoadFinished(Loader<D> loader, D result)方法使Loader将结果传送给客户端.
 Loader通过调用Loader#deliverResult(D result),将结果传递给已注册的监听器.

#### 三.Loader三种不同状态.
 已启动: 处于已启动状态的Loader会执行载入操作,并在任何时间将结果传递到监听器中.已启动的Loader将会监听数据改变,当检测到改变时执行新的载入.
 一旦启动,Loader将一直处在已启动状态,一直到转换到已停止和重置,这是唯一一种onLoadFinished永远都会调用的状态。
 
 已停止: 处于已停止状态的Loader将会继续监听数据改变,但是不会将结果返回给客户端.在已停止状态,Loader可能被启动或者重启.
 
 重置: 当Loader处于重置状态时,将不会执行新的载入操作,也不会发送新的结果,也不会检测数据变化.
 当一个Loader进入重置状态,它必须解除对应的数据引用,方便垃圾回收(客户端也必须确定,在Loader无效之后,移除了所有对该数据的引用).
 通常,重置Loader不会两次调用.然而,在某些情况下他们可能会启动,所以如果必要的话,它们必须能够适时重置.

#### 四.接收Loader数据改变的通知.
 必须有一个观察者接受数据源改变的通知.
 Loader必须实现这些Observer其中之一(ContentObserver，BroadcastReceiver等),来检测底层数据源的改变.
 当检测到数据改变,观察者必须调用Loader#onContentChanged().在该方法中执行两种不同操作:
 (1)如果Loader已经处于启动状态，就会执行一个新的载入操作; 
 (2)设置一个flag标识数据源有改变，这样当Loader再次启动时，就知道应该重新载入数据了.
 
#### 五.CursorLoader实现LoaderManager.LoaderCallbacks接口方法.
接口声明及使用如下:
```
 public interface LoaderCallbacks<D> { 
  public Loader<D> onCreateLoader(int id, Bundle args); 
  public void onLoadFinished(Loader<D> loader, D data); 
  public void onLoaderReset(Loader<D> loader); 
 }
 ```
 onCreateLoader方法将在创建Loader时候调用，此时需要提供查询的配置,如监听一个URI.
 这个方法会在loader初始化的时调用，即调用下面的代码时调用：
  getLoaderManager().initLoader(id, bundle, loaderCallbacks);
  initLoader函数原型为:
  <D> Loader<D> android.app.LoaderManager.initLoader(int id, Bundle bundle, LoaderCallbacks<D> loaderCallbacks);
  第1个参数loader的ID,可自定义一个常量值,便于实现多个Loader;
  第2个参数一般置null;
  第3个参数是实现了LoaderManager.LoaderCallbacks实现类对象.
  onLoadFinished方法,在Loader完成任务后调用,一般在此读取结果.
  onLoaderReset方法是在配置发生变化时调用,一般调用下面的代码后调用:
  getLoaderManager().restartLoader(id, bundle, loaderCallbacks);
  restartLoader方法参数同initLoader,重新初始化loader之后,需要用来释放对前面loader查询到的结果引用.
  
  #### 六.使用场景
  在之前呢，我们经常会有这种需求，比如在某个activity，或者某个fragment里面，我们需要查找某个数据源，并且显示出来，当数据源自己更新的时候，界面也要及时响应。

  当然咯，查找数据这个过程可能很短，但是也可能很漫长，为了避免anr，我们都是开启一个子线程去查找，然后通过handler来更新我们的ui界面。但是，考虑到activity和fragment 复杂的生命周期，上述的方法 使用起来会很不方便，毕竟你要考虑到保存现场 还原现场 等等复杂的工作来保证你的app无懈可击。所以后来呢谷歌就帮我们推出了一个

  新的东西---Loader。他可以帮我们完成上述所有功能！实在是很强大。