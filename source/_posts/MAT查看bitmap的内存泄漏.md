---
title: MAT查看bitmap的内存泄漏
date: 2018-11-30 14:38:32
tags:
categories:
- 内存检测
---
#### 前言
在开发中，一些类似Bitmap的对象会占用很大的内存，即使使用弱引用、代码优化及时释放，可以有效减少内存泄漏现象的产生。但这依然不够，很多时候，我们需要尽量少的使用内存。对用户来说，用户并不懂内存泄漏，但是用户可以通过后台查看你的内存使用情况，如果占用过大，一些用户会选择卸载来清理门户。
我们可以通过分析，找出内存占用较大的模块，通过代码或者其他一些方式，减少内存使用。

#### 准备工具
在所有操作开始之前，我们需要准备一些材料
首先是MAT内存分析工具，基本没有什么安装过程，这里附上下载地址：
http://www.eclipse.org/mat/downloads.php

还有就是Bitmap查看工具GIMP，下载地址：
http://www.gimp.org/

#### 获取.hprof文件
在Android Studio中，我们可以在Android Profile窗口中看到内存的使用情况，并进行手动GC，获取.hprof文件等操作

**在获取.hprof文件之前，我们需要手动回收内存，由于系统内存回收是优先级很低的线程，如果不去手动回收，会造成我们分析的不必要失误**

但是此时的.hprof是不能用MAT打开的，需要导出才能继续分析。需要通过命令转换hprof文件。

#### 分析内存占用
MAT的强大在于它把所有的内存占用情况以数据的形式表达出来，一眼就能了解当前的内存占用情况。
双击打开解压的Memory Analyzer，File-->Open Heap Dump,选择保存的.hprof文件，等待加载，点击Finish。

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/nei12.webp)

等待加载，在Overview中可以查看饼状图：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/nei13.webp)

选择Histogram，如图：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/nei14.webp)

Histogram视图下，点击Retained Heap查看当前内存占用情况，并从大到小排序，如图：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/nei15.webp)

到这里为止，其实我们可以看出大概了，Bitmap对内存的占用简直令人发指。

选择android.graphics.Bitmap，查看Bitmap占用情况，右键-->List Objects-->withoutgoing references，如图：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/nei16.webp)

点击Retained Heap排序，我们可以看到单单一个Bitmap对象就占用了3M，而这其中类似大小的不在少数。

选择需要查看的Bitmap对象，双击打开：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/nei17.webp)

选中mBuffer，导出byte，右键-->Copy-->Save Value to File，保存到指定目录，文件后缀名可为.data

打开安装好的GIMP，文件-->打开，选择刚刚保存的.data文件：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/nei18.webp)

其中，图像类型选择RGBAlpha，宽度和高度填入Eclipse Analyzer中获取的。

点击打开，即可看到内存占用的到底是个什么图片，并且针对这些图片进行优化。