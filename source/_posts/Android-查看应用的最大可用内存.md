---
title: Android 查看应用的最大可用内存
date: 2018-11-30 13:31:49
tags:
categories:
- ANDROID
---
#### 单个应用可用的最大内存
Android设备出厂以后，java虚拟机对单个应用的最大内存分配就确定下来了，超出这个值就会OOM。这个属性值是定义在/system/build.prop文件中的
**dalvik.vm.heapstartsize=16m**
它表示堆分配的初始大小，它会影响到整个系统对RAM的使用程度，和第一次使用应用时的流畅程度。
它值越小，系统ram消耗越慢，但一些较大应用一开始不够用，需要调用gc和堆调整策略，导致应用反应较慢。它值越大，这个值越大系统ram消耗越快，但是应用更流畅。

**dalvik.vm.heapgrowthlimit=192m**
单个应用可用最大内存主要对应的是这个值,它表示单个进程内存被限定在64m,即程序运行过程中实际只能使用64m内存，超出就会报OOM。（仅仅针对dalvik堆，不包括native堆）

**dalvik.vm.heapsize=512m**
heapsize参数表示单个进程可用的最大内存，但如果存在heapgrowthlimit参数，则以heapgrowthlimit为准.

heapsize表示不受控情况下的极限堆，表示单个虚拟机或单个进程可用的最大内存。而android上的应用是带有独立虚拟机的，也就是每开一个应用就会打开一个独立的虚拟机（这样设计就会在单个程序崩溃的情况下不会导致整个系统的崩溃）。
注意：在设置了heapgrowthlimit的情况下，单个进程可用最大内存为heapgrowthlimit值。在android开发中，如果要使用大堆，需要在manifest中指定android:largeHeap为true，这样dvm heap最大可达heapsize。

不同设备，这些个值可以不一样。一般地，厂家针对设备的配置情况都会适当的修改/system/build.prop文件来调高这个值。随着设备硬件性能的不断提升，从最早的16M限制（G1手机）到后来的24m,32m，64m等，都遵循Android框架对每个应用的最小内存大小限制，参考http://source.android.com/compatibility/downloads.html 3.7节。

**通过代码查看每个进程可用的最大内存，即heapgrowthlimit值：**

```
ActivityManager activityManager = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
int memClass = activityManager.getMemoryClass();//以m为单位
```

或：

```
$adb shell getprop dalvik.vm.heapgrowthlimit

192m

$adb shell getprop dalvik.vm.heapsize

512m

$adb shell getprop dalvik.vm.heapstartsize

16m
```

**检查你应该使用多少的内存**

正如前面提到的，每一个Android设备都会有不同的RAM总大小与可用空间，因此不同设备为app提供了不同大小的heap限制。你可以通过调用getMemoryClass())来获取你的app的可用heap大小。如果你的app尝试申请更多的内存，会出现OutOfMemory的错误。

在一些特殊的情景下，你可以通过在manifest的application标签下添加largeHeap=true的属性来声明一个更大的heap空间。如果你这样做，你可以通过getLargeMemoryClass())来获取到一个更大的heap size。

然而，能够获取更大heap的设计本意是为了一小部分会消耗大量RAM的应用(例如一个大图片的编辑应用)。不要轻易的因为你需要使用大量的内存而去请求一个大的heap size。只有当你清楚的知道哪里会使用大量的内存并且为什么这些内存必须被保留时才去使用large heap. 因此请尽量少使用large heap。使用额外的内存会影响系统整体的用户体验，并且会使得GC的每次运行时间更长。在任务切换时，系统的性能会变得大打折扣。

另外, large heap并不一定能够获取到更大的heap。在某些有严格限制的机器上，large heap的大小和通常的heap size是一样的。因此即使你申请了large heap，你还是应该通过执行getMemoryClass()来检查实际获取到的heap大小。