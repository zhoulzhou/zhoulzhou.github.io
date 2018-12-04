---
title: Android 对象池的原理及使用
date: 2018-12-04 16:05:32
tags:
- 对象池
categories:
- 性能优化
---
#### 对象池的使用
在阅读Glide源码时，发现Glide大量使用对象池Pools来对频繁需要创建和销毁的代码进行优化。
比如Glide中，每个图片请求任务，都需要用到 EngineJob 、DecodeJob类。若每次都需要重新new这些类，并不是很合适。而且在大量图片请求时，频繁创建和销毁这些类，可能会导致内存抖动，影响性能。
Glide使用对象池的机制，对这种频繁需要创建和销毁的对象保存在一个对象池中。每次用到该对象时，就取对象池空闲的对象，并对它进行初始化操作，从而提高框架的性能。

#### 了解Android 垃圾回收
Android里面是一个三级Generation的内存模型，最近分配的对象会存放在Young Generation区域，当这个对象在这个区域停留的时间达到一定程度，它会被移动到Old Generation，最后到Permanent Generation区域。每一个级别的内存区域都有固定的大小，此后不断有新的对象被分配到此区域，当这些对象总的大小快达到这一级别内存区域的阀值时，会触发GC的操作，以便腾出空间来存放其他新的对象。每次GC发生的时候，所有的线程都是暂停状态的。GC所占用的时间和它是哪一个Generation也有关系，Young Generation的每次GC操作时间是最短的，Old Generation其次，Permanent Generation最长。

#### 导致GC频繁执行有两个原因
Memory Churn内存抖动，内存抖动是因为大量的对象被创建又在短时间内马上被释放。 
瞬间产生大量的对象会严重占用Young Generation的内存区域，当达到阀值，剩余空间不够的时候，也会触发GC。即使每次分配的对象占用了很少的内存，但是他们叠加在一起会增加Heap的压力，从而触发更多其他类型的GC。这个操作有可能会影响到帧率，并使得用户感知到性能问题。

#### 如何解决
如果我们发现了大量对象的创建该如何处理呢？ 
可以优化就优化，比如在onDraw中初始化了一些对象，我们可以考虑是否可以将这些对象初始化到外部（比如构造方法），而不要在视图绘制需要反复调用的方法中去new 
不能优化的采用对象池解决，如果我们这些对象的初始化不可避免，那么我们要考虑对象的复用，采用对象池来解决

#### 对象池
{% asset_img dui.jpg %}
我们在Android开发中其实可能已经使用过，只是我们没用关注而已。比如在handler发送消息时，Message的初始化经常会用Message.obtain()来实例化Message对象；在View自定义中用到手势速度控制的VelocityTracker。根据源码虽然两者对实现方式不同（Message使用链表、VelocityTracker使用数组），但是原理是一样的。即：
>初始化一个固定大小池子，我们每次创建对象时候先去池子中找有没有，
>如果有直接取出，没有new出来使用后还到池子里。这样便可达到对象复用的目的

#### 使用对象池的代价以及注意事项
**当然使用对象池也是要有一定代价的：**
>1、开发者保存到对象池的对象一定要足够"重",如果随便保存一个简单类型的到对象池,可能访问对象池需要消耗的内存比节省下来的还要多,得不偿失.
>2、短时间内生成了大量的对象占满了池子，那么后续的对象是不能复用的 
>3、对象池是静态的，如果池子被占满，当我们离开该页面这些对象可能不再需要，那么池子不释放其中的无用对象还是要占用一定的内存空间

#### 注意事项
**使用时候申请(obtain)和释放(recycle)成对出现，使用一个对象后一定要释放还给池子 **
池子的大小要根据实际情况合理指定。池子太大上面提到的不释放而占用的内存会很大，池子太小对象过多而且因为操作耗而不能立即释放还给池子时候，池子满了，后续对象还是不能复用。所以，根据项目实际场景制定合理的大小是很必要的 

#### 代码介绍
**Pool接口类 **
acquire(): 从对象池请求对象的函数:
release(): 释放对象回对象池的函数
```
public static interface Pool<T> {

    public T acquire();
    
    public boolean release(T instance);
}
```
**Android官方对象池的简单实现：SimplePool，也是用得最多的实现**
原理：使用了“懒加载”的思想。当SimplePool初始化时，不会生成N个T类型的对象存放在对象池中。而是当每次外部调用release()时，才把释放的T类型对象存放在对象池中。要先放入，才能取出来。
```
public static class SimplePool<T> implements Pool<T> {
    private final Object[] mPool;
    private int mPoolSize;
    
    public SimplePool(int maxPoolSize) {
        if (maxPoolSize <= 0) {
            throw new IllegalArgumentException("The max pool size must be > 0");
        }
        mPool = new Object[maxPoolSize];
    }

    @Override
    @SuppressWarnings("unchecked")
    public T acquire() {
        if (mPoolSize > 0) {
            final int lastPooledIndex = mPoolSize - 1;
            T instance = (T) mPool[lastPooledIndex];
            mPool[lastPooledIndex] = null;
            mPoolSize--;
            return instance;
        }
        return null;
    }

    @Override
    public boolean release(T instance) {
        if (isInPool(instance)) {
            throw new IllegalStateException("Already in the pool!");
        }
        if (mPoolSize < mPool.length) {
            mPool[mPoolSize] = instance;
            mPoolSize++;
            return true;
        }
        return false;
    }

    private boolean isInPool(T instance) {
        for (int i = 0; i < mPoolSize; i++) {
            if (mPool[i] == instance) {
                return true;
            }
        }
        return false;
    }
}
```
**SynchronizedPool**
```
/**
     * Synchronized) pool of objects.
     *
     * @param <T> The pooled type.
     */
    public static class SynchronizedPool<T> extends SimplePool<T> {
        private final Object mLock = new Object();

        /**
         * Creates a new instance.
         *
         * @param maxPoolSize The max pool size.
         *
         * @throws IllegalArgumentException If the max pool size is less than zero.
         */
        public SynchronizedPool(int maxPoolSize) {
            super(maxPoolSize);
        }

        @Override
        public T acquire() {
            synchronized (mLock) {
                return super.acquire();
            }
        }

        @Override
        public boolean release(T element) {
            synchronized (mLock) {
                return super.release(element);
            }
        }
    }
```

#### 实例对SynchronizedPool的简单封装1
由于对象池设计是要先放入，才能取出来。所以当没有放入对象时，调用acquire()，返回都是null，所以可以对 对象池进行以下封装，方便其使用：
```
public class MyPooledClass {
  private static final SynchronizedPool<ObjectClass> sPool =
          new SynchronizedPool<ObjectClass>(10);

  public static ObjectClass obtain() {
      ObjectClass instance = sPool.acquire();
      return (instance != null) ? instance : new ObjectClass();
  }

  public void recycle() {
      sPool.release(this);
  }
}
```
#### 实例对SynchronizedPool的简单封装2
```
public class User {
 
    public String id;
 
      public String name;
 
      private static final SynchronizedPool<User> sPool = new SynchronizedPool<User>(
          10);
 
      public static User obtain() {
      User instance = sPool.acquire();
      return (instance != null) ? instance : new User();
      }
 
      public void recycle() {
          sPool.release(this);
        }
    }
}
```



