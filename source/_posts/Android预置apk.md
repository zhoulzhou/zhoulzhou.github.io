---
title: Android预置apk
date: 2018-12-06 16:57:38
tags:
- 预置apk
categories:
- Android build系统
---
#### 一、如何将带源码的APK预置进系统？
1)     在 packages/apps 下面以需要预置的 APK的 名字创建一个新文件夹，以预制一个名为Test的APK 为例
2)     将 Test APK的Source code 拷贝到 Test 文件夹下，删除 /bin 和 /gen 目录
3)     在 Test 目录下创建一个名为 Android.mk的文件，内容如下：
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE_TAGS := optional
LOCAL_SRC_FILES := $(call all-subdir-java-files)
LOCAL_PACKAGE_NAME := Test
include $(BUILD_PACKAGE)

4)     打开文件 build/target/product/${Project}.mk （其中 ${Project} 表示工程名）
将 Test 添加到 PRODUCT_PACKAGES 里面。
5)     重新 build 整个工程

#### 二、如何将无源码的 APK 预置进系统？
1)     在 packages/apps 下面以需要预置的 APK 名字创建文件夹，以预制一个名为Test的APK为例
2)     将 Test.apk 放到 packages/apps/Test 下面
3)     在  packages/apps/Test 下面创建文件 Android.mk，文件内容如下：
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
Module name should match apk name to be installed
LOCAL_MODULE := Test
LOCAL_MODULE_TAGS := optional
LOCAL_SRC_FILES := $(LOCAL_MODULE).apk
LOCAL_MODULE_CLASS := APPS
LOCAL_MODULE_SUFFIX := $(COMMON_ANDROID_PACKAGE_SUFFIX)
LOCAL_CERTIFICATE := PRESIGNED
include $(BUILD_PREBUILT)
 
4)     打开文件 build/target/product/${Project}.mk （其中 ${Project} 表示工程名）
    将 Test 添加到 PRODUCT_PACKAGES 里面。
5)     将从Test.apk解压出来的 so库拷贝到alps/vendor/mediatek/${Project}/artifacts/out/target/product/${Project}/system/lib/目录下，若无 so 库，则去掉此步；
6)     重新 build 整个工程

#### 三、如何预置APK使得用户可以卸载？
有两种方法：
方法一：
7)     在 packages/apps 下面以需要预置的 APK 名字创建文件夹，以 预制一个名为Test的APK为例
8)     将 Test.apk 放到 packages/apps/Test 下面；
9)     在  packages/apps/Test 下面创建文件 Android.mk，文件内容如下：
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
Module name should match apk name to be installed
LOCAL_MODULE := Test
LOCAL_MODULE_TAGS := optional
LOCAL_SRC_FILES := $(LOCAL_MODULE).apk
LOCAL_MODULE_CLASS := APPS
LOCAL_MODULE_SUFFIX := $(COMMON_ANDROID_PACKAGE_SUFFIX)
LOCAL_CERTIFICATE := PRESIGNED
LOCAL_MODULE_PATH := $(TARGET_OUT_DATA_APPS)
include $(BUILD_PREBUILT)
 
10)   打开文件 build/target/product/${Project}.mk （其中 ${Project} 表示工程名）
    将 Test 添加到 PRODUCT_PACKAGES 里面。
11)   将从Test.apk解压出来的 so库拷贝到alps/vendor/mediatek/${Project}/artifacts/out/target/product/${Project}/system/lib/目录下，若无 so 库，则去掉此步；
12)   重新 build 整个工程

注意：这个比不能卸载的多了一句
LOCAL_MODULE_PATH := $(TARGET_OUT_DATA_APPS)
 
方法二：
4） 将需要预置的 apk 拷贝到：
vendor/mediatek/${Project}/artifacts/out/target/product/${Project}/data/app/
5） 重新 build 整个工程
注意：如果没有相应目录则需手动创建。

#### 四、如何使得用户在将预置的 APK 卸载后，恢复出厂设置时能恢复？
为了让用户在将预置的 APK 卸载后，恢复出厂设置时能恢复，
做法是：
在vendor/mediatek/project_name/artifacts/out/target/product/project_name/system目录下新建一个名为appbackup的文件夹，将该应用的apk文件copy到appbackup文件夹下
在mediatek/config/project_name/ProjectConfig.mk文件中添加定义：MTK_SPECIAL_FACTORY_RESET=yes
在vendor/mediatek/project_name/artifacts/out/target/product/project_name/data/app下创建一个.restore_list，并且在其中添加语句：
/system/appbackup/xxx.apk（注意，.restore_list中的每一行都要以"/system” 开头）
当卸载了data/app下的apk后，再恢复出厂设置，系统会从 .restore_list 中读取apk的名字，然后从 appbackup 文件中把apk重新拷贝到data/app下，从而恢复data/app下已经卸载了的apk。

