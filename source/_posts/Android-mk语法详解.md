---
title: Android .mk语法详解
date: 2018-12-06 16:58:35
tags:
- mk语法
categories:
- ANDROID
---
一个Android.mk文件可以编译多个模块，每个模块属下列类型之一：
  1）APK程序
  一般的Android程序，编译打包生成apk文件
  2）JAVA库
  java类库，编译打包生成jar文件
  3)C\C++应用程序
 可执行的C\C++应用程序
  4）C\C++静态库 
编译生成C\C++静态库，并打包成.a文件
  5）C\C++共享库
编译生成共享库（动态链接库），并打包成.so文， 有且只有共享库才能被安装/复制到您的应用软件（APK）包中。


#### Android.mk语法详解
**LOCAL_PATH := $(call my-dir) **
每个Android.mk文件必须以定义LOCAL_PATH为开始。它用于在开发tree中查找源文件。宏my-dir 则由Build System提供。返回包含Android.mk的目录路径。

**include $(CLEAR_VARS) **
CLEAR_VARS 变量由Build System提供。并指向一个指定的GNU Makefile，由它负责清理很多LOCAL_xxx.
例如：LOCAL_MODULE, LOCAL_SRC_FILES, LOCAL_STATIC_LIBRARIES等等。但不清理LOCAL_PATH.
这个清理动作是必须的，因为所有的编译控制文件由同一个GNU Make解析和执行，其变量是全局的。所以清理后才能避免相互影响。

**LOCAL_MODULE    := hello-jni **

LOCAL_MODULE模块必须定义，以表示Android.mk中的每一个模块。名字必须唯一且不包含空格。Build System会自动添加适当的前缀和后缀。例如，foo，要产生动态库，则生成libfoo.so. 但请注意：如果模块名被定为：libfoo.则生成libfoo.so. 不再加前缀

**LOCAL_MODULE_PATH :=$(TARGET_ROOT_OUT) **
指定最后生成的模块的目标地址

>TARGET_ROOT_OUT:根文件系统，路径为out/target/product/generic/root
>TARGET_OUT:system文件系统，路径为out/target/product/generic/system
>TARGET_OUT_DATA:data文件系统，路径为out/target/product/generic/data
>除了上面的这些，NDK还提供了很多其他的TARGET_XXX_XXX变量，用于将生成的模块拷贝到输出目录的不同路径
>默认是TARGET_OUT

**LOCAL_SRC_FILES := hello-jni.c **

LOCAL_SRC_FILES变量必须包含将要打包如模块的C/C++ 源码。不必列出头文件，build System 会自动帮我们找出依赖文件。缺省的C++源码的扩展名为.cpp. 也可以修改，通过LOCAL_CPP_EXTENSION

**include $(BUILD_SHARED_LIBRARY) **
BUILD_SHARED_LIBRARY：是Build System提供的一个变量，指向一个GNU Makefile Script。
它负责收集自从上次调用 include $(CLEAR_VARS)  后的所有LOCAL_XXX信息。并决定编译为什么。

>BUILD_STATIC_LIBRARY    编译为静态库。 
>BUILD_SHARED_LIBRARY    编译为动态库 
>BUILD_EXECUTABLE        编译为Native C可执行程序  
>BUILD_PREBUILT          该模块已经预先编译
>BUILD_PACKAGE     编译打包成APK文件
>BUILD_STATIC_JAVA_LIBRARY     用它来编译生成JAVA库（打包成.jar文件）
>NDK还定义了很多其他的BUILD_XXX_XXX变量，它们用来指定模块的生成方式。
