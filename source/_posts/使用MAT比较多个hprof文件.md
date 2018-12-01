---
title: 使用MAT比较多个hprof文件
date: 2018-11-30 09:58:08
tags:
categories:
- 内存检测
---
调试内存泄露时，有时候适时比较2个或多个heap dump文件是很有用的。这时需要生成多个单独的HPROF文件。

下面是一些关于如何在MAT里比较多个heap dumps的内容（有一点复杂）：

1. 第一个HPROF 文件(usingFile > Open Heap Dump ).

2. 打开Histogram view.

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/nei2.jpg)

3. 在NavigationHistory view里 (如果看不到就从Window > Navigation History找 ), 右击histogram然后选择Add to Compare Basket .

4. 打开第二个HPROF 文件然后重做步骤2和3.

5. 切换到Compare Basket view, 然后点击Compare the Results (视图右上角的红色"!"图标)。

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/nei3.jpg)

 如上，结果图中，Objects #1所代表的weak.create.hprof比Objects#0所代表的main.hporf多出了一个WeakReferencesActivity；Objects #2更是多出10000个WFObject对象出来，结果很明显。

