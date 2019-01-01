---
title: Gradle出现的问题及解决
date: 2019-01-01 18:35:48
tags:
- gradle
categories:
- 开发工具
---
**各种XXX的问题**
Gradle 出现各种莫名其妙的问题， 如果不是代码的问题就是配置，或资源的问题 特别是.9图片导致的问题。

#### AAPT2 error问题
1、一般是.9图片造成的，就是9patch图片有问题造成报错的，选中Show bad patches，就会看到那个.9被红框圈出来了，这是AS在提示你，你的这个.9图片是有问题的，所以编译不通过。

2、res的values目录下有style文件，这个文件有时也会造成AAPT2 error这个错误。

3、gradlew compileDebug --stacktrace -info  查看出问题的详细情况 不一定会显示准确的log
