---
title: 深入理解Android build系统
date: 2018-12-08 10:07:49
tags:
- build
- 编译
categories:
- ANDROID
---
#### 概述

Android Build 系统是用来编译 Android 系统、Android SDK 以及相关文档的一套框架。在Android系统中，Android 的源码中包含了许许多多的模块。 不同产商的不同设备对于 Android 系统的定制都是不一样的。如何将这些模块统一管理起来，如何能够在不同的操作系统上进行编译，如何在编译时能够支持面向不同的硬件设备，不同的编译类型，且还要提供面向各个产商的定制扩展，Android系统如何解决这些问题呢？这就是我们不得不谈的Android Build 系统。

Android源码目录结构： 

{% asset_img build.png %}

#### Linux系统的make命令
在讲解Android编译系统之前，我们首先需要了解Linux系统的make命令。在Linux系统中，我们可以通过make命令来编译代码。Make命令在执行的时候，默认会在当前目录找到一个Makefile文件，然后根据Makefile文件中的指令来对代码进行编译。如gcc，Linux系统中的shell命令cp、rm等等。

看到这里，有的小伙伴可能会说，在Linux系统中，shell和make命令有什么区别呢？ 
make命令事实也是通过shell命令来完成任务的，但是它的神奇之处是可以帮我们处理好文件之间的依赖关系。例如有一个文件T，它依赖于另外一个文件D，要求只有当文件D的内容发生变化，才重新生成文件T。

Make命令是怎么知道两个文件之间存在依赖关系，以及当被依赖文件发生变化时如何处理目标文件的呢？答案就在前面提到的Makefile文件。Makefile文件实际上是一个脚本文件，就像普通的shell脚本文件一样，只不过它遵循的是Makefile语法。Makefile文件最基础的功能就是描述文件之间的依赖关系，以及怎么处理这些依赖关系。

#### Android Build简介
Android Build 系统是 Android 系统的一部分，主要用来编译 Android 系统，Android SDK 以及相关文档。该系统主要由 Make 文件，Shell 脚本以及 Python 脚本组成。 
Android build分类：

>build/core 目录下的文件，这是Android Build的系统框架核心；
>device目录下的文件，存放的是具体的产品配置文件；
>各个模块的编译文件：Android.mk，位于模块的原文件目录下。

#### Android Build系统核心
Android Build系统核心在目录build/core，这个目录中有mk文件、shell脚本和per脚本，他们构成Android Build系统的基础和架构。

在核心的buil/core里，系统主要干了三件事情： 

{% asset_img build1.png %}

**常用命令：**

```
source build/envsetup.sh
lunch
make
```

**envsetup.sh**
而在build/envsetup.sh中主要完成了三件事： 

{% asset_img build3.png %}

执行Android系统的编译，必须先执行envsetup.sh脚本，这个脚本会建立Android的编译环境。其具体执行的是建立shell命令以及调用add_lunch_combo命令，这个命令的将调用该命令的所传递的参数存放到一个全局的数组变量LUNCH_MENU_CHOICES中。

envsetup.sh脚本中定义的常用shell命令：
命令	说明
contact-button	指定当前编译的产品
croot   快速切换到源码的根目录，方便开始编译
m	编译整个源码，但不用将当前的目录切换到源码的根目录
mm	编译当前目录下的所有模块，但是不编译他们的依赖项
mm	编译当前目录下的所有模块，但是不编译他们的依赖项
cgrep	对系统中所有的C/C++文件执行grep命令
sgrep	对系统中所有的源文件执行grep命令

#### 编译 Android 系统
Android 系统的编译环境目前只支持 Ubuntu 以及 Mac OS 两种操作系统。在编译Android系统之前我们需要先获取完整的 Android 源码。打开控制台之后转到 Android 源码的根目录，然后执行如下命名：

```
source build/envsetup.sh 
lunch full-eng 
make -j8
```

关于这几条命令的意思，我们上面提过。 
第一步命令“source build/envsetup.sh”引入了 build/envsetup.sh脚本，该脚本的作用是初始化编译环境，并引入一些辅助的 Shell 函数；

第二步命令“lunch full-eng”是调用 lunch 函数，并指定参数为“full-eng”。lunch 函数的参数用来指定此次编译的目标设备以及编译类型。

第三部命令“make -j8”才真正开始执行编译。make 的参数“-j”指定了同时编译的 Job 数量，这是个整数，该值通常是编译主机 CPU 支持的并发线程总数的 1 倍或 2 倍（例如：在一个 4 核，每个核支持两个线程的 CPU 上，可以使用 make -j8 或 make -j16）。 
完整的编译时间依赖于编译主机的配置。

#### Build 结果
所有的编译产物都将位于 /out 目录下，该目录下主要包含：

>/out/host/：该目录下包含了针对主机的 Android 开发工具的产物。即 SDK 中的各种工具，例如：emulator，adb，aapt 等。
>/out/target/common/：该目录下包含了针对设备的共通的编译产物，主要是 Java 应用代码和 Java 库。
>/out/target/product//：包含了针对特定设备的编译结果以及平台相关的 C/C++ 库和二进制文件。其中，是具体目标设备的名称。
>/out/dist/：包含了为多种分发而准备的包，通过“make disttarget”将文件拷贝到该目录，默认的编译目标不会产生该目录。

#### Build 生成的镜像文件
Build 的产物中最重要的是三个镜像文件，它们都位于 /out/target/product// 目录下:

>system.img：包含了 Android OS 的系统文件，库，可执行文件以及预置的应用程序，将被挂载为根分区。
>ramdisk.img：在启动时将被 Linux 内核挂载为只读分区，它包含了 /init文件和一些配置文件。它用来挂载其他系统镜像并启动 init 进程。
>userdata.img：将被挂载为 /data，包含了应用程序相关的数据以及和用户相关的数据。

#### Make 文件
整个 Build 系统的入口文件是源码树根目录下名称为“Makefile”的文件，当在源代码根目录上调用 make 命令时，make 命令首先将读取该文件。 
Makefile 文件的内容只有一行：“include build/core/main.mk”。该行代码的作用很明显：包含 build/core/main.mk 文件。在 main.mk 文件中又会包含其他的文件，其他文件中又会包含更多的文件，这样就引入了整个 Build 系统。

在整个Build系统中，Make 文件间的关系是相当复杂的。看一张make文件主要的关系图： 

{% asset_img build4.png %}

Make 常用文件：
文件名	说明
main.mk	主要的 Make 文件，该文件中首先将对编译环境进行检查，同时引入其他的 Make 文件。另外，该文件中还定义了几个最主要的 Make 目标，例如 droid，sdk，等（参见后文“Make 目标说明”）。
help.mk	含了名称为 help 的 Make 目标的定义，该目标将列出主要的 Make 目标及其说明。
envsetup.mk	配置 Build 系统需要的环境变量，例如：TARGET_PRODUCT，TARGET_BUILD_VARIANT，HOST_OS，HOST_ARCH 等。 当前编译的主机平台信息（例如操作系统，CPU 类型等信息）就是在这个文件中确定的。 另外，该文件中还指定了各种编译结果的输出路径。
pathmap.mk	将许多头文件的路径通过名值对的方式定义为映射表，并提供 include-path-for 函数来获取。例如，通过 $(call include-path-for, frameworks-native)便可以获取到 framework 本地代码需要的头文件路径。
combo/select.mk	根据当前编译器的平台选择平台相关的 Make 文件。
dumpvar.mk	在 Build 开始之前，显示此次 Build 的配置信息。
config.mk	整个 Build 系统的配置文件，最重要的 Make 文件之一。该文件中主要包含以下内容： 定义了许多的常量来负责不同类型模块的编译。 定义编译器参数以及常见文件后缀，例如 .zip,.jar.apk。 根据 BoardConfig.mk 文件，配置产品相关的参数。 设置一些常用工具的路径，例如 flex，e2fsck，dx。
definitions.mk	最重要的 Make 文件之一，在其中定义了大量的函数。这些函数都是 Build 系统的其他文件将用到的。例如：my-dir，all-subdir-makefiles，find-subdir-files，sign-package 等，关于这些函数的说明请参见每个函数的代码注释。
distdir.mk	针对 dist 目标的定义。dist 目标用来拷贝文件到指定路径
dex_preopt.mk	针对启动 jar 包的预先优化。
pdk_config.mk	顾名思义，针对 pdk（Platform Developement Kit）的配置文件。
post_clean.mk	在前一次 Build 的基础上检查当前 Build 的配置，并执行必要清理工作。
legacy_prebuilts.mk	该文件中只定义了 GRANDFATHERED_ALL_PREBUILT 变量。
Makefile	被 main.mk 包含，该文件中的内容是辅助 main.mk 的一些额外内容。

Android 源码中包含了许多的模块，模块的类型有很多种，例如：Java 库，C/C++ 库，APK 应用，以及可执行文件等 。并且，Java 或者 C/C++ 库还可以分为静态的或者动态的，库或可执行文件既可能是针对设备（本文的“设备”指的是 Android 系统将被安装的设备，例如某个型号的手机或平板）的也可能是针对主机（本文的“主机”指的是开发 Android 系统的机器，例如装有 Ubuntu 操作系统的 PC 机或装有 MacOS 的 iMac 或 Macbook）的。不同类型的模块的编译步骤和方法是不一样，为了能够一致且方便的执行各种类型模块的编译，在 config.mk 中定义了许多的常量，这其中的每个常量描述了一种类型模块的编译方式。常见的有： BUILD_HOST_STATIC_LIBRARY BUILD_HOST_SHARED_LIBRARY BUILD_STATIC_LIBRARY BUILD_SHARED_LIBRARY BUILD_EXECUTABLE BUILD_HOST_EXECUTABLE BUILD_PACKAGE BUILD_PREBUILT BUILD_MULTI_PREBUILT BUILD_HOST_PREBUILT BUILD_JAVA_LIBRARY BUILD_STATIC_JAVA_LIBRARY BUILD_HOST_JAVA_LIBRARY 不同类型的模块的编译过程会有一些相同的步骤，例如：编译一个 Java 库和编译一个 APK 文件都需要定义如何编译 Java 文件。为了减少代码冗余，需要将共同的代码复用起来，复用的方式是将共同代码放到专门的文件中，然后在其他文件中包含这些文件的方式来实现的。 模块的编译方式定义文件的包含关系：

 Make 编译镜像 make /make droid 如果在源码树的根目录直接调用“make”命令而不指定任何目标，则会选择默认目标：“droid”（在 main.mk 中定义）。因此，这和执行“make droid”效果是一样的。 droid 目标将编译出整个系统的镜像。从源代码到编译出系统镜像，整个编译过程非常复杂。这个过程并不是在 droid 一个目标中定义的，而是 droid 目标会依赖许多其他的目标，这些目标的互相配合导致了整个系统的编译。 那么需要编译出系统镜像，需要哪些依赖呢？
 android 所依赖的其他 Make目标说明：

 称	说明
apps_only	该目标将编译出当前配置下不包含 user，userdebug，eng 标签（关于标签，请参见后文“添加新的模块”）的应用程序。
droidcore	该目标仅仅是所依赖的几个目标的组合，其本身不做更多的处理。
dist_files	该目标用来拷贝文件到 /out/dist 目录。
files	该目标仅仅是所依赖的几个目标的组合，其本身不做更多的处理
prebuilt	该目标依赖于 (ALLPREBUILT)，(ALLPREBUILT)，(ALL_PREBUILT)的作用就是处理所有已编译好的文件。
$(modules_to_install)	modules_to_install 变量包含了当前配置下所有会被安装的模块（一个模块是否会被安装依赖于该产品的配置文件，模块的标签等信息），因此该目标将导致所有会被安装的模块的编译。
$(modules_to_check)	该目标用来确保我们定义的构建模块是没有冗余的。
$(INSTALLED_ANDROID_INFO_TXT_TARGET)	该目标会生成一个关于当前 Build 配置的设备信息的文件，该文件的生成路径是：out/target/product//android-info.txt
systemimage	生成 system.img。

**Build 系统中包含的其他一些 Make 目标：**
Make目标说明	说明
make clean	执行清理，等同于：rm -rf out/
make sdk	编译出 Android 的 SDK
Make目标说明	说明
make clean-sdk	清理 SDK 的编译产物
make update-api	更新 API。在 framework API 改动之后，需要首先执行该命令来更新 API，公开的 API 记录在 frameworks/base/api 目录下。
make dist	执行 Build，并将 MAKECMDGOALS 变量定义的输出文件拷贝到 /out/dist 目录
make all	编译所有内容，不管当前产品的定义中是否会包含
make help	帮助信息
make snod	从已经编译出的包快速重建系统镜像
make libandroid_runtime	编译所有 JNI framework 内容
makeframework	编译所有 Java framework 内容
makeservices	编译系统服务和相关内容
make	编译一个指定的模块，local_target 为模块的名称
make clean-	清理一个指定模块的编译结果
makedump-products	显示所有产品的编译配置信息，例如：产品名，产品支持的地区语言，产品中会包含的模块等信息
makePRODUCT-xxx-yyy	编译某个指定的产品
makebootimage	生成 boot.img

定制 Build 系统中内容 当我们要开发一款新的 Android 产品的时候，我们首先就需要在 Build 系统中添加对于该产品的定义。在 Android Build 系统中对产品定义的文件通常位于 device 目录下，device 目录下可以公司名以及产品名分为二级目录，然后加入到系统中，如以前小米等基于Android深度定制的系统。 通常，对于一个产品的定义通常至少会包括四个文件：AndroidProducts.mk，产品版本定义文件，BoardConfig.mk 以及 verndorsetup.sh。 ## AndroidProducts.mk 该文件只需要定义一个变量，名称为“PRODUCT_MAKEFILES”。

```
PRODUCT_MAKEFILES := \ 
 $(LOCAL_DIR)/full_stingray.mk \ 
 $(LOCAL_DIR)/stingray_emu.mk \ 
 $(LOCAL_DIR)/generic_stingray.mk
```

产品版本定义文件 该文件中包含了对于特定产品版本的定义。该文件可能不只一个，因为同一个产品可能会有多种版本。 通常情况下，我们并不需要定义所有这些变量。Build 系统的已经预先定义好了一些组合，它们都位于 /build/target/product 下，每个文件定义了一个组合，我们只要继承这些预置的定义，然后再覆盖自己想要的变量定义即可。

```
继承 full_base.mk 文件中的定义
 $(call inherit-product, $(SRC_TARGET_DIR)/product/full_base.mk) 
 # 覆盖其中已经定义的一些变量
 PRODUCT_NAME := full_lt26 
 PRODUCT_DEVICE := lt26 
 PRODUCT_BRAND := Android 
 PRODUCT_MODEL := Full Android on LT26

```

BoardConfig.mk 该文件用来配置硬件主板，它其中定义的都是设备底层的硬件特性。例如：该设备的主板相关信息，Wifi 相关信息，还有 bootloader，内核，radioimage 等信息。 ## vendorsetup.sh 该文件中作用是通过 add_lunch_combo 函数在 lunch 函数中添加一个菜单选项。该函数的参数是产品名称加上编译类型，中间以“-”连接，例如：add_lunch_combo full_lt26-userdebug。/build/envsetup.sh 会扫描所有 device 和 vender 二 级目 录下的名称 为”vendorsetup.sh”文件，并根据其中的内容来确定 lunch 函数的 菜单选项。 在配置了以上的文件之后，便可以编译出我们新添加的设备的系统镜像了。 我们可以使用命令

```
source build/envsetup.sh
```

来查看Build 系统已经引入了刚刚添加的 vendorsetup.sh 文件。 添加新模块 在源码树中，一个模块的所有文件通常都位于同一个文件夹中。为了将当前模块添加到整个 Build 系统中，每个模块都需要一个专门的 Make 文件，该文件的名称为“Android.mk”。Build 系统会扫描名称为“Android.mk”的文件，并根据该文件中内容编译出相应的产物。 注： 在 Android Build 系统中，编译是以模块（而不是文件）作为单位的，每个模块都有一个唯一的名称，一个模块的依赖对象只能是另外一个模块，而不能是其他类型的对象。对于已经编译好的二进制库，如果要用来被当作是依赖对象，那么应当将这些已经编译好的库作为单独的模块。对于这些已经编译好的库使用 BUILD_PREBUILT 或 BUILD_MULTI_PREBUILT。例如：当编译某个 Java 库需要依赖一些 Jar 包时，并不能直接指定 Jar 包的路径作为依赖，而必须首先将这些 Jar 包定义为一个模块，然后在编译 Java 库的时候通过模块的名称来依赖这些 Jar 包。 那么怎么编写Android.mk 文件呢？ Android.mk 文件通常以以下两行代码作为开头：

```
LOCAL_PATH := $(call my-dir) //设置当前模块的编译路径为当前文件夹路径
 include $(CLEAR_VARS)//清理编译环境中用到的变量
```

为了方便模块的编译，Build 系统设置了很多的编译环境变量。要编译一个模块，只要在编译之前根据需要设置这些变量然后执行编译即可。常见的如： - LOCAL_SRC_FILES：当前模块包含的所有源代码文件。 - LOCAL_MODULE：当前模块的名称，这个名称应当是唯一的，模块间的依赖关系就是通过这个名称来引用的。 - LOCAL_C_INCLUDES：C 或 C++ 语言需要的头文件的路径。 - LOCAL_STATIC_LIBRARIES：当前模块在静态链接时需要的库的名称。 - LOCAL_SHARED_LIBRARIES：当前模块在运行时依赖的动态库的名称。 - LOCAL_CFLAGS：提供给 C/C++ 编译器的额外编译参数。 - LOCAL_JAVA_LIBRARIES：当前模块依赖的 Java 共享库。 - LOCAL_STATIC_JAVA_LIBRARIES：当前模块依赖的 Java 静态库。 - LOCAL_PACKAGE_NAME：当前 APK 应用的名称。 - LOCAL_CERTIFICATE：签署当前应用的证书名称。 - LOCAL_MODULE_TAGS：当前模块所包含的标签，一个模块可以包含多个标签。标签的值可能是 debug, eng,user，development 或者 optional。其中，optional是默认标签。标签是提供给编译类型使用的，不同的编译类型会安装包含不同标签的模块。 编译类型说明:

名称	说明
eng	默认类型，该编译类型适用于开发阶段。 当选择这种类型时，编译结果将： 安装包含 eng, debug, user，development 标签的模块 安装所有没有标签的非 APK 模块 安装所有产品定义文件中指定的 APK 模块
user	该编译类型适合用于最终发布阶段。 当选择这种类型时，编译结果将： 安装所有带有 user 标签的模块 安装所有没有标签的非 APK 模块 安装所有产品定义文件中指定的 APK 模块，APK 模块的标签将被忽略
userdebug	该编译类型适合用于 debug 阶段。 该类型和 user 一样，除了： 会安装包含 debug 标签的模块 编译出的系统具有 root 访问权限

根据上表各种类型模块的编译方式，要执行编译，只需要引入表 3 中对应的 Make 文件即可。例如，要编译一个 APK 文件，只需要在 Android.mk 文件中，加入“include $(BUILD_PACKAGE)。 
除此以外，Build 系统中还定义了一些便捷的函数以便在 Android.mk 中使用，包括：

>$(call my-dir)：获取当前文件夹路径。
>$(call all-java-files-under, xx)：获取指定目录xx下的所有 Java 文件。
>$(call all-subdir-java-files) 获取所有子目录中的 Java 文件
>$(call all-c-files-under, xx)：获取指定目录xx下的所有 C 语言文件。
>$(call all-Iaidl-files-under, xx) ：获取指定目录xx下的所有 AIDL 文件。
>$(call all-makefiles-under, xx)：获取指定目录xx下的所有 Make 文件。

**编译一个 apk：**
```
 LOCAL_PATH := $(call my-dir) 
  include $(CLEAR_VARS) 
  # 获取指定目录src下的所有 Java 文件。
  LOCAL_SRC_FILES := $(call all-java-files-under,src )           
  # 当前模块依赖的静态 Java 库，如果有多个以空格分隔
  LOCAL_STATIC_JAVA_LIBRARIES := static-library 
  # 当前模块的名称
  LOCAL_PACKAGE_NAME := LocalPackage 
  # 编译 APK 文件
  include $(BUILD_PACKAGE)
```
  
  **编译一个 Java 的静态库：**
  
  ```
  LOCAL_PATH := $(call my-dir) 
  include $(CLEAR_VARS) 
  # 获取所有子目录中的 Java 文件
  LOCAL_SRC_FILES := $(call all-subdir-java-files) 
  # 当前模块依赖的动态 Java 库名称
  LOCAL_JAVA_LIBRARIES := android.test.runner 
  # 当前模块的名称
  LOCAL_MODULE := sample 
  # 将当前模块编译成一个静态的 Java 库
  include $(BUILD_STATIC_JAVA_LIBRARY)
```
