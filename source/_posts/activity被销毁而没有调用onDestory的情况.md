---
title: activity被销毁而没有调用onDestory的情况
date: 2018-12-19 09:42:55
tags:
- activity被销毁
categories:
- ANDROID
---
关于activity在被系统回收会不会在走onDestory的问题，不管是在开发中，还是在面试过程中经常会遇到或者被问。现在我们先从整个项目上来分析。

首先一个应用（项目）正常情况下在linux中只会有一个进程（不是绝对的），应用只有在进程存活的情况下才会按照正常的生命周期进行执行，如果进程突然被kill掉，相当于System.exit(0);   进程被杀死，根本不会走（activity，fragment）生命周期。例如安装的一键清理等功能，同样不会被调用。只有在进程不被kill掉，正常情况下才会执行ondestory（）方法。

activity被手机内存强制回收是不会调用destory方法的

外部强制关闭进程，或者异常崩溃的时候