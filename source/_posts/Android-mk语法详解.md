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

** Android.mk语法 **
package/app/Settings/Android.mk

LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE_TAGS := optional
LOCAL_SRC_FILES := $(call all-java-files-under, src)
LOCAL_PACKAGE_NAME := Settings
LOCAL_CERTIFICATE := platform
include $(BUILD_PACKAGE)
Use the folloing include to make our test apk.
include $(call all-makefiles-under,$(LOCAL_PATH))

>PS：GNU Make‘功能’宏，必须通过使用'$(call  )'来调用，调用他们将返回文本化的信息。

#### Android.mk语法详解
>LOCAL_PATH:= $(call my-dir)

一个Android.mk file首先必须定义好LOCAL_PATH变量。它用于在开发树中查找源文件。宏函数’my-dir’，由编译系统提供，用于返回当前路径（即包含Android.mk file文件的目录）。
(2) Android.mk中可以定义多个编译模块，每个编译模块都是以include $(CLEAR_VARS)开始，以include $(BUILD_XXX)结束。

>include $(CLEAR_VARS)

CLEAR_VARS 变量由Build System提供。并指向一个指定的GNU Makefile，由它负责清理很多LOCAL_xxx.
例如：LOCAL_MODULE, LOCAL_SRC_FILES, LOCAL_STATIC_LIBRARIES等等。但不清理LOCAL_PATH.
这个清理动作是必须的，因为所有的编译控制文件由同一个GNU Make解析和执行，其变量是全局的。所以清理后才能避免相互影响。
			
>LOCAL_MODULE_TAGS := optional

 解析：LOCAL_MODULE_TAGS :=user eng tests optional；
      user:  指该模块只在user版本下才编译；
      eng:  指该模块只在eng版本下才编译；
      tests: 指该模块只在tests版本下才编译；
      optional:指该模块在所有版本下都编译；
 取值范围debug eng tests optional samples shell_ash shell_mksh。注意不能取值user，如果要预装，则应定义core.mk。

>LOCAL_MODULE_PATH :=$(TARGET_ROOT_OUT)

指定最后生成的模块的目标地址
    TARGET_ROOT_OUT:根文件系统，路径为out/target/product/generic/root
    TARGET_OUT:system文件系统，路径为out/target/product/generic/system
    TARGET_OUT_DATA:data文件系统，路径为out/target/product/generic/data
除了上面的这些，NDK还提供了很多其他的TARGET_XXX_XXX变量，用于将生成的模块拷贝到输出目录的不同路径,默认是TARGET_OUT

>LOCAL_SRC_FILES := $(call all-java-files-under, src)

1、如果要包含的是java源码的话，可以调用all-java-files-under得到。(这种形式来包含local_path目录下的所有java文件)；
2、当涉及到C/C++时，LOCAL_SRC_FILES变量就必须包含将要编译打包进模块中的C或C++源代码文件。注意，在这里你可以不用列出头文件和包含文件，因为编译系统将会自动为你找出依赖型的文件；仅仅列出直接传递给编译器的源代码文件就好。
3、all-java-files-under宏的定义是在build/core/definitions.mk中。
4、 $(call all-subdir-java-files)  获取所有子目录中的 Java 文件  $(call all-java-files-under, src)：获取指定目录src下的所有 Java 文件。

>LOCAL_PACKAGE_NAME := Settings

package的名字，这个名字在脚本中将标识这个app或package。

>LOCAL_CERTIFICATE := platform

LOCAL_CERTIFICATE 后面是签名文件的文件名，说明Settings.apk是一个需要platform key签名的APK

>include $(BUILD_PACKAGE)      # Tell it to build an APK

$(BUILD_PACKAGE)是用来编译生成package/app/下的apk。还有其他几种编译情况：
include $(BUILD_STATIC_LIBRARY)         表示编译成静态库；
include $(BUILD_SHARED_LIBRARY)         表示编译成动态库；
include $(BUILD_EXECUTABLE)             表示编译成可执行程序；
include $(BUILD_STATIC_JAVA_LIBRARY)    表示编译生成JAVA库（打包成.jar文件）

>include $(call all-makefiles-under,$(LOCAL_PATH))

加载当前目录下的所有makefile文件，all-makefiles-under会返回一个位于当前'my-dir'路径的子目录中的所有Android.mk的列表。all-makefiles-under宏的定义是在build/core/definitions.mk中。

编译此Android.mk文件最后就生成了Settings.apk。


