---
title: Binder机制 实例分析
date: 2018-12-10 17:17:39
tags:
categories:
- Binder机制
---
可以说 Binder 是 Android 底层系统的一个特色了，它很好地解决了进程间通讯的问题。其实网上有很多介绍 Binder 的文章，那么本文还是想将 Binder 这部分内容细化一下，更适合于初学者阅读。

#### Binder 产生的背景
首先我们说说为什么会出现 Binder 这个东西。作为 iOS 开发者，我还是情不自禁地想去谈谈 iOS app，事实上，iOS 的每一个 app 都是一个独立的进程，它没有 Android 那种比较开放的多进程通讯能力，甚至 App 与 Extension （如通知中心插件）之间都不能有一种非常直接的通讯方式，当然不是说 iOS 没有 IPC 技术，其实 mach 内核也是有着不错的 IPC 技术的，但这不是本文的重点。Android 则不太一样，Android apps 基本上都需要各式各样的 IPC 需求，甚至启动一个 Activity 也需要用到 IPC，有一些 IPC 调用也许你并不知晓，可能对开发者最可见的就是用 AIDL 去写一个 Remote Service 接口了。

Android 很多核心功能都是由一系列 Services 支持的，比如 ActivityService、WindowService 等等等等，应用需要频繁地与这些 Services 发生交互，正是基于这种场景，就亟需一种好的 IPC 解决方案。

你可能会想，为什么不是 Local Socket？或者 Shared Memory，那是因为安全性无法得到保障。Android 的权限系统需要一种可靠的方式来保证各种 Services 的访问是在权限系统的监控下进行的，上述提到的解决方案就做不到了，因为不管是套接字还是共享内存，现有的 Linux 内核都不存在一种检验双方身份的方法存在，任何通过套接字或者共享内存走的数据都可以伪造，而在这个基础上做任何验证，代价都是相当高的。Android 的选择是基于内核，重新开发一套 IPC 机制，让它固有这些特性，也就是让系统可以在 Ring0 级保障交互双方身份的正确性，并且这种基于内核的方案效率还很高。

既然要基于内核，就一定要对内核动手脚，Android 采用驱动的方式实现这个技术，而不是直接修改 Linux 内核。这样你就可以假设，手机中有一个“设备”，应用之间通过这个设备来交互，而这个设备自身有一套身份校验机制，这样就比基于用户态的 IPC 方案来的安全得多，也快得多了

#### Binder 是怎么工作的
我们暂且不需要深入理解 Binder 驱动底层的实现，也不需要知道 Binder 驱动提供了什么接口，我们就来看看一个 app 是如何通过 Binder 这个机制来实现跨进程通信的。

到这里，你可以把 Binder 驱动看作一个机器，它连接着所有 Services（假设 app 只调用 Services 提供的接口）。

一个 app（客户端）想要找一个 Service 办点事，它就要去操作这个机器，而这个机器会检测 app 的身份，如果身份合法，则可以继续操作。假设 app 要调用 A 服务的 foo 函数，那么它可以告诉这个机器，这个机器随后就会检查连在它身上的所有 Services，找到 app 需要的那一个，让它执行 foo 函数，并且再将返回值告诉 app，这样 app 就成功隔着进程操作到 A 服务了。

这当然是很抽象的说法，系统在 Framework 层做了很好地封装，让我们可以友好地使用 Binder 驱动来进行 IPC 操作，下面我们就直接看应用层所提供的接口是如何工作的。

#### 探究与 Binder 相关的 Java 类
要说这些类，我们首先要用到它们，最简单的方式就是去创建一个 Service，让它运行在单独的进程中，然后编写 AIDL，实现一个接口，再到 Activity 中去使用这个接口。（注意：本文不介绍 Remote Service 与 AIDL 的相关知识，如果读者对这部分内容还不够了解，请先将它们弄懂再回来看本文）

AIDL 代码生成器为我们创建了一个 java 文件，这里面涉及到几个类：

{% asset_img bin.webp %}

这些就是在应用层使用 Binder 所需的所有类了，看类图有些错综，但实际还是很简单的。

AIDL 生成的就是 IMyService 这个接口，而 Stub 和 Proxy 则是这个接口的两个内部类，分别是 Binder 类和 BinderProxy 类的子类（Proxy 类虽然是用组合方式来持有 BinderProxy 的，但实际就是在直接用这个类，只不过做了一层封装，让其更易使用而已），Stub 和 Proxy 都实现了 IMyService。

所以 IInterface 到底是什么，它就是一个用于表达 Service 提供的功能的一个契约，也就是说 IInterface 里有的方法，Service 都能提供，调用者你别管用的是 BinderProxy 还是什么，只要拿到 IInterface，你就可以直接调用里面的方法，它就是一个接口。

同时 Stub 虽然实现了 IMyService，但是并没有实现厘米的任何方法，它是一个抽象类，开发者需要自己子类化 Stub 去实现具体的功能。

Proxy 实现了 IMyService，并且实现了里面的方法，这些方法的实现我们下面再说。

为什么 IMyService 要分 Stub 和 Proxy 呢？这是为了要适用于本地调用和远程调用两种情况。如果 Service 运行在同一个进程，那就直接用 Stub，因为它直接实现了 Service 提供的功能，不需要任何 IPC 过程。如果 Service 运行在其他进程，那客户端使用的就是 Proxy，这里这个 Proxy 的功能大家应该能想到了吧，就是把参数封装后发送给 Binder 驱动，然后执行一系列 IPC 操作最后再取出结果返回。

好，到这里请求用 Proxy 发出去了，Service 怎么接受请求并作出响应呢，这就需要 Stub 了，还记得我们的 Stub 是继承自 Binder 的吗？Binder 有一个 onTransact 方法，而 Stub 重写了这个函数，这个函数三个重要参数：int code、Parcel data、Parcel reply，分别对应了被调函数编号、参数包、响应包。当 Proxy 发起了一个请求，服务端中相应的响应线程就会通过 JNI 调用到 Stub 类，然后执行里面的 execTransact 方法，进而转到 onTransact 方法。（这一系列调用链大家可以从源码中分析出来，我这里作为抛砖引玉，就不带大家分析 native 层的代码了）

我们来看一眼这个 onTransact：

```
@Override
public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    switch (code)    {
        case INTERFACE_TRANSACTION:
        {
            reply.writeString(DESCRIPTOR);
            return true;
        }
        case TRANSACTION_increaseCounter:
        {
            data.enforceInterface(DESCRIPTOR);
            int _arg0;
            _arg0 = data.readInt();
            int _result = this.increaseCounter(_arg0);
            reply.writeNoException();
            reply.writeInt(_result);
            return true;
        }
    }
    return super.onTransact(code, data, reply, flags);
}

```

是不是非常清晰易懂，就是根据传过来的数据包来做相应的操作，然后把结果写回数据包，Binder 驱动会来帮我们做好这些数据包的分发工作。而这段代码是运行在 Service 本地进程中的，它可以直接调用实现好的 Stub 类中的相关方法（本例子中是 increaseCounter 方法）。

下面我们趁热打铁再来看一眼 Proxy 类中的 increaseCounter 是怎么实现的：

```
@Override
public int increaseCounter(int increment) throws RemoteException{
    Parcel _data = Parcel.obtain();
    Parcel _reply = Parcel.obtain();
    int _result;
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        _data.writeInt(increment);
        mRemote.transact(Stub.TRANSACTION_increaseCounter, _data, _reply, 0);
        _reply.readException();
        _result = _reply.readInt();
    }
    finally {
        _reply.recycle();
        _data.recycle();
    }
    return _result;
}
```

正好与 Stub 中的处理能够对应起来，其实这两段代码就是整个 IPC 的核心了，Binder 驱动和 Binder 类在底层帮我们做好了其他一切事情。

休息一下。

下面我们来思考另一件事情，如何判断 Service 运行在同一进程还是不同进程？

我们知道，Service 有一个 onBind 方法，这里面就返回了我们实现好的 Stub 类，而客户端 bind service 时拿到的又是一个 IBinder 对象，我们每次只需要调用 Stub 的 asInterface 静态方法，把这个 IBinder 对象传进去就能拿到 Stub 类或者 Proxy 类了，看起来十分智能！那么这个 asInterface 又蕴藏什么玄机呢？我们来看一眼实现代码：

```
public static IBackgroundService asInterface(IBinder obj){
    if ((obj==null)) {
        return null;
    }
    IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
    if (((iin != null) && (iin instanceof IBackgroundService))) {
        return ((IBackgroundService) iin);
    }
    return new IBackgroundService.Stub.Proxy(obj);
}
```

是不是有点莫名其妙，queryLocalInterface 是什么鬼？
我们再来看看 queryLocalInterface 的实现：

```
public IInterface queryLocalInterface(String descriptor) {
    if (mDescriptor.equals(descriptor)) {
        return mOwner;
    }
    return null;
}

```

它判断了一下 descriptor 参数是否与自身 owner 的 descriptor 一致，如果一致就直接返回 owner，那么 owner 和 descriptor 是在哪被设置的呢，就是在 Stub 的构造函数中被设置的。
于是乎，如果 Service 运行在同一进程，那么客户端拿到的 IBinder 就是 Stub 类，而 Stub 的 queryLocalInterface 又会返回自己；而 Service 运行在单独进程中时，客户端拿到的 IBinder 就是系统提供好的 BinderProxy，BinderProxy 中的 queryLocalInterface 默认直接返回 null，根据代码，asInterface 就会构造一个 Proxy 返回给客户端，那么接下来的故事就是上面我们讲过的了。

#### 自己利用 Binder 来进行 IPC
有了上面的基础，其实我们完全不需要 AIDL 了有木有，自己用 Binder 类和 BinderProxy 类就完全可以实现 Service 与客户端的通讯，下面我就速速写一个简单的例子。

Service 中的 onBinder 方法我这样实现：

{% asset_img bin2.webp %}

客户端就这样：

{% asset_img bin5.webp %}

我们甚至不必按照 AIDL 的通信规范，自己处理 data 和 reply 也是完全可以的，但这只是为了演示 Binder 的原理，日常开发中还是不要这么做了吧。

#### 小结
Binder 的设计非常优秀，分析 Binder 的实现也是一件十分有趣的事情，本文作为对 Binder 深入研究的「引入」，站在了一个很高的角度俯视整套系统，并没有对其中的细节做深入探讨，如果大家对内核开发或者操作系统底层有兴趣的话，也可以去看看 Binder 驱动的实现，相信也会有不少收获，本文到这里就先结束了 ;-)
