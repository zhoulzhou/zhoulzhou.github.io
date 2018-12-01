---
title: MAT查找内存泄漏
date: 2018-11-30 11:04:46
tags:
categories:
- 内存检测
---
#### 获取堆转储HPROF文件
通过点击时间线下方工具栏中的 Export heap dump as HPROF file，将堆转储导出到一个 HPROF 文件中。 在显示的对话框中，确保使用 .hprof 后缀保存文件。

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/nei5.png)

要使用其他 HPROF 分析器，您需要将 HPROF 文件从 Android 格式转换为 Java SE HPROF 格式。 您可以使用 android_sdk/platform-tools/ 目录中提供的 hprof-conv 工具执行此操作。 运行包括以下两个参数的 hprof-conv 命令：原始 HPROF 文件和转换后 HPROF 文件的写入位置。 例如：

```
hprof-conv heap-original.hprof heap-converted.hprof
```

MAT查看HPROF
然後用MAT打开我们导出的文件，我导出了两个文件，test1.hprof和test2.hprof,其中test1.hprof是内存未泄露时的快照，test2.hprof是内存已经泄露的快照。我们用MAT的Histogram（直方图）和Dominator Tree （支配树）来分析内存情况。Histogram可以列出内存中每个对象的名字、数量以及大小。Dominator Tree会将所有内存中的对象按大小进行排序，并且我们可以分析对象之间的引用结构。 

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/nei6.png)

一、 Histogram（直方图）
可列出每一个类的实例数。支持正则表达式查找，也可以计算出该类所有对象的retained size。默认是通过class（group by class）分类展示的。 

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/nei7.png)

这就是test2.hprof的直方图。现在要说两个名词解释。Shallow Heap/Retained Heap 
- **Shallow Heap **
Shallow size就是对象本身占用内存的大小，不包含其引用的对象内存，实际分析中作用不大。在堆上，看起来是一堆原生的byte[], char[], int[]，对象本身的内存都很小。所以我们可以看到以Shallow Heap进行排序的Histogram图中，排在第一位第二位的是byte，char 
- **Retained Heap **
Retained size是该对象自己的shallow size，加上从该对象能直接或间接访问到对象的shallow size之和。换句话说，retained size是该对象被GC之后所能回收到内存的总和。RetainedHeap可以更精确的反映一个对象实际占用的大小（因为如果该对象释放，retained heap都可以被释放）。

Histogram中是可以显示对象的数量的，那么比如说我们现在怀疑MainActivity中有可能存在内存泄漏，就可以在第一行的正则表达式框中搜索“MainActivity”，（一般是查看自己应用的内存泄漏直接搜索包名列出应用的所有类）如下所示： 

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/nei8.png)

可以看到MainActivity的数量是2，这不正常，解决这个，就需要查看MainActivity被谁引用了，不能被释放。我们右键选择exclude all phantom/weak/soft etc.references, 意思是查看排除虚引用/弱引用/软引用等的引用链 （这些引用最终都能够被GC干掉，所以排除） 

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/nei9.png)

**还有其他菜单供选择 **
- List objects with （以Dominator Tree的方式查看） 
- incoming references 引用到该对象的对象 
- outcoming references 被该对象引用的对象 
- Show objects by class （以class的方式查看） 
- incoming references 引用到该对象的对象 
- outcoming references 被该对象引用的对象

#### 通用分析方法
**通常的操作是先选择 List objects with  选择 incoming references 获取到引用该对象的对象，
然后选择上图所示的操作去掉可被内存回收的软/弱引用的对象**
**当你按上面操作之后，凶手就出现了**

这里，原来是UserManger的实例。 

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/nei10.png)

还有更强大的，通过OQL语句查询,有点像写SQL语句。

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/nei11.png)

是不是分分钟查出MainActivity有两个对象。哈哈！ 
比如：查找size＝0并且未使用过的ArrayList

```
select * from java.util.ArrayList where size=0 and modCount=0
```

这个地方可以多研究研究。



