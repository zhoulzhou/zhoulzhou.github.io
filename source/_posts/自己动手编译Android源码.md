---
title: 自己动手编译Android源码
date: 2018-11-30 15:03:28
tags:
categories:
- Android build系统
---
本文使用最新的Ubuntu 16.04,请首先确保自己已经安装了Git.没安装的同学可以通过以下命令进行安装:

```
sudo apt-get install git 
git config –global user.email “test@test.com” 
git config –global user.name “test”
```

#### 简要说明
android源码编译的四个流程:1.源码下载;2.构建编译环境;3.编译源码;4运行.下文也将按照该流程讲述.

#### 源码下载
由于某墙的原因,这里我们采用国内的镜像源进行下载.
目前,可用的镜像源一般是科大和清华的,具体使用差不多,这里我选择清华大学镜像进行说明.(参考:清华源)
https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/

#### repo工具下载及安装
通过执行以下命令实现repo工具的下载和安装

```
mkdir ~/bin
PATH=~/bin:$PATH
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
```

总结一下:repo就是这么一种工具,由一系列python脚本组成,通过调用Git命令实现对AOSP项目的管理.

#### 建立源码文件夹
熟悉Git的同学都应该知道,我们需要为项目在本地创建对应的仓库.同样,这里为了方便对代码进行管理,我们为其创建一个文件夹.这里我在当前用户目录下创建了source文件夹,后面所有的下载的源码和编译出的产物也都放在这里,命令如下:

```
mkdir source
cd source
```

#### 初始化仓库
我们将上面的source文件夹作为仓库,现在需要来初始化这个仓库了.通过执行初始化仓库命令可以获取AOSP项目master上最新的代码并初始化该仓库,命令如下:

```
repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest
```
如果执行该命令的过程中,如果提示无法连接到 gerrit.googlesource.com，那么我们只需要编辑 ~/bin/repo文件，找到REPO_URL这一行,然后将其内容修改为:

```
REPO_URL = 'https://gerrit-google.tuna.tsinghua.edu.cn/git-repo'
```

然后重新执行上述命令即可.
**补充说明**
不带参数的manifest命令用于获取master上最新的代码,但是可以通过-b参数指定获取某个特定的android版本,比如我们想要获取android-4.0.1_r1分支,那么命令如下:

```
repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest -b android-4.0.1_r1
```

#### 同步源码到本地
初始化仓库之后,就可以开始正式同步代码到本地了,命令如下:

```
repo sync
```

以后如果需要同步最新的远程代码到本地,也只需要执行该命令即可.在同步过程中,如果因为网络原因中断,使用该命令继续同步即可.不出意外,5个小时便可以将全部源码同步到本地.所以呢,这个过程可以放在晚上睡觉期间完成.

(提示:一定要确定代码完全同步了,不然在下面编译过程出现的错误会让你痛不欲生,不确定的童鞋可以多用repo sync同步几次)

#### 构建编译环境
源码下载完成后,就可以构建编译环境了:
我现在在Ubuntu 16.04下编译AOSP主线代码,因此需要安装OpenJDK 8,执行命令如下:

```
sudo apt-get install openjdk-8-jdk
```

下面是Ubuntu16.04中的依赖设置:

```
sudo apt-get install libx11-dev:i386 libreadline6-dev:i386 libgl1-mesa-dev g++-multilib 
sudo apt-get install -y git flex bison gperf build-essential libncurses5-dev:i386 
sudo apt-get install tofrodos python-markdown libxml2-utils xsltproc zlib1g-dev:i386 
sudo apt-get install dpkg-dev libsdl1.2-dev libesd0-dev
sudo apt-get install git-core gnupg flex bison gperf build-essential  
sudo apt-get install zip curl zlib1g-dev gcc-multilib g++-multilib 
sudo apt-get install libc6-dev-i386 
sudo apt-get install lib32ncurses5-dev x11proto-core-dev libx11-dev 
sudo apt-get install libgl1-mesa-dev libxml2-utils xsltproc unzip m4
sudo apt-get install lib32z-dev ccache
```

#### 初始化编译环境
确保上述过程完成后,接下来我们需要初始化编译环境,命令如下:

```
source build/envsetup.sh
```

执行该命令结果如下:

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/bian1.webp)

不难发现该命令只是引入了其他执行脚本,至于这些脚本做什么,目前不在本文中细说.
该命令执行成功后,我们会得到了一些有用的命令,比如最下面要用到的lunch命令.

#### 编译源码
初始化编译环境之后,就进入源码编译阶段.这个阶段又包括两个阶段:选择编译目标和执行编译.

#### 选择编译目标
通过lunch指令设置编译目标,所谓的编译目标就是生成的镜像要运行在什么样的设备上.这里我们设置的编译目标是aosp_arm64-eng,因此执行指令:

```
lunch aosp_arm64-eng
```

**编译目标格式说明**
编译目标的格式:BUILD-BUILDTYPE,比如上面的aosp_arm-eng的BUILD是aosp_arm,BUILDTYPE是eng.

**什么是BUILD**
BUILD指的是特定功能的组合的特定名称,即表示编译出的镜像可以运行在什么环境.其中,aosp(Android Open Source Project)代表Android开源项目;arm表示系统是运行在arm架构的处理器上,arm64则是指64位arm架构;处理器,x86则表示x86架构的处理器;此外,还有一些单词代表了特定的Nexus设备,下面是常用的设备代码和编译目标,更多参考官方文档
|受型号|设备代码|编译目标|
|---|----|---|
|Nexus 6P|angler|aosp_angler-userdebug|
|Nexus 5X|bullhead|aosp_bullhead-userdebug|
|Nexus 6|shamu|aosp_shamu-userdebug|
|Nexus 5|hammerhead|aosp_hammerhead-userdebug|

提示:如果你没有Nexus设备,那么通常选择arm或者x86即可

**什么是BUILDTYPE**

BUILD TYPE则指的是编译类型,通常有三种:
-user:代表这是编译出的系统镜像是可以用来正式发布到市场的版本,其权限是被限制的(如,没有root权限,不鞥年dedug等)
-userdebug:在user版本的基础上开放了root权限和debug权限.
-eng:代表engineer,也就是所谓的开发工程师的版本,拥有最大的权限(root等),此外还附带了许多debug工具

了解编译目标的组成之后,我们就可以根据自己目前的情况选择了.那不知道编译目标怎么办?
我们只需要执行不带参数的lunch指令,稍后,控制台会列出所有的编译目标,如下:

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/bian2.webp)

接着我们只需要输入相应的数字即可.

来举个例子:你没有Nexus设备,只想编译完后运行看看,那么就可以选择aosp_arm-eng.
(我在ubuntu 16.04(64位)中编译完成后启动虚拟机时,卡在黑屏,尝试编译aosp_arm64-eng解决.因此,这里我使用了aosp_arm64-eng)

#### 开始编译
通过make指令进行代码编译,该指令通过-j参数来设置参与编译的线程数量,以提高编译速度.比如这里我们设置8个线程同时编译:

```
make  -j16 2>&1 | tee buildlog-userdebug.txt
```

需要注意的是,参与编译的线程并不是越多越好,通常是根据你机器cup的核心来确定:core*2,即当前cpu的核心的2倍.比如,我现在的笔记本是双核四线程的,因此根据公式,最快速的编译可以make -j8.
(通过cat /proc/cpuinfo查看相关cpu信息)

如果一切顺利的化,在几个小时之后,便可以编译完成.看到### make completed successfully (01:18:45(hh:mm:ss)) ###表示你编译成功了.

#### 运行模拟器
在编译完成之后,就可以通过以下命令运行Android虚拟机了,命令如下:

```
source build/envsetup.sh
lunch(选择刚才你设置的目标版本,比如这里了我选择的是2)
emulator
```

不出意外,在等待一会之后,你会看到运行界面:

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/bian3.webp)

**补充**
既然谈到了模拟器运行,这里我们顺便介绍模拟器运行所需要四个文件:
Linux Kernel
system.img
userdate.img
ramdisk.img

如果你在使用lunch命令时选择的是aosp_arm-eng,那么在执行不带参数的emualtor命令时,Linux Kernel默认使用的是/source/prebuilds/qemu-kernel/arm/kernel-qemu目录下的kernel-qemu文件;而android镜像文件则是默认使用source/out/target/product/generic目录下的system.img,userdata.img和ramdisk.img,也就是我们刚刚编译出来的镜像文件.

上面我在使用lunch命令时选择的是aosp_arm64-eng,因此linux默认使用的/source/prebuilds/qemu-kernel/arm64/kernel-qemu下的kernel-qemu,而其他文件则是使用的source/out/target/product/generic64目录下的system.img,userdata.img和ramdisk.img.
当然,emulator指令允许你通过参数制定使用不同的文件,具体用法可以通过emulator --help查看

#### 模块编译
除了通过make命令编译可以整个android源码外,Google也为我们提供了相应的命令来支持单独模块的编译.

编译环境初始化(即执行source build/envsetup.sh)之后,我们可以得到一些有用的指令,除了上边用到的lunch,还有以下:

```
- croot: Changes directory to the top of the tree.
  - m: Makes from the top of the tree.
  - mm: Builds all of the modules in the current directory.
  - mmm: Builds all of the modules in the supplied directories.
  - cgrep: Greps on all local C/C++ files.
  - jgrep: Greps on all local Java files.
  - resgrep: Greps on all local res/*.xml files.
  - godir: Go to the directory containing a file.
```

其中mmm指令就是用来编译指定目录.通常来说,每个目录只包含一个模块.比如这里我们要编译Launcher2模块,执行指令:

```
mmm packages/apps/Launcher2/
```

稍等一会之后,如果提示:
###make completed success fully###
即表示编译完成,此时在out/target/product/gereric/system/app就可以看到编译的Launcher2.apk文件了.

#### 重新打包系统镜像
编译好指定模块后,如果我们想要将该模块对应的apk集成到系统镜像中,需要借助make snod指令重新打包系统镜像,这样我们新生成的system.img中就包含了刚才编译的Launcher2模块了.重启模拟器之后生效.

#### 单独安装模块
我们在不断的修改某些模块,总不能每次编译完成后都要重新打包system.img,然后重启手机吧?有没有什么简单的方法呢?
在编译完后,借助adb install命令直接将生成的apk文件安装到设备上即可,相比使用make snod,会节省很多事件.

**补充**
我们简单的来介绍out/target/product/generic/system目录下的常用目录:
Android系统自带的apk文件都在out/target/product/generic/system/apk目录下;
一些可执行文件(比如C编译的执行),放在out/target/product/generic/system/bin目录下;
动态链接库放在out/target/product/generic/system/lib目录下;
硬件抽象层文件都放在out/targer/product/generic/system/lib/hw目录下.

#### SDK编译
如果你需要自己编译SDK使用,很简单,只需要执行命令make sdk即可.