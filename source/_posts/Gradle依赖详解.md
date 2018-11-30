---
title: Gradle依赖详解
date: 2018-11-30 16:01:17
tags:
categories:
- ANDROID
---
#### 仓库
1、一个仓库可以被视为一些文件的集合体。Gradle不会默认为你的项目添加任何仓库。所以你需要把它们添加到repositories方法体内。如果是使用的是Android studio，那么工具已经为你准备好了这一切：

```
repositories {
    jcenter()
}
```

Gradle支持三种不同的仓库，分别是：Maven和Ivy以及文件夹。依赖包会在你执行build构建的时候从这些远程仓库下载，当然Gradle会为你在本地保留缓存，所以一个特定版本的依赖包只需要下载一次。

2、为了方便，Gradle会默认预定义三个maven仓库：Jcenter和mavenCentral以及本地maven仓库。你可以同时申明它们：

```
repositories {
       mavenCentral()
       jcenter()
       mavenLocal()
   }  
```

Maven和Jcenter仓库是很出名的两大仓库。我们没必要同时使用他们，在这里我建议你们使用jcenter，jcenter是maven中心库的一个分支，这样你可以任意去切换这两个仓库。当然jcenter也支持了https，而maven仓库并没有。

本地maven库是你曾使用过的所有依赖包的集合，当然你也可以添加自己的依赖包。默认情况下，你可以在你的home文件下找到.m2的文件夹。除了这些仓库外，你还可以使用其他的公有的甚至是私有仓库。

3、**一个依赖需要定义三个元素：group，name和version。group意味着创建该library的组织名，通常这会是包名，name是该library的唯一标示，version是该library的版本号。**我们来看看如何申明依赖：

```
dependencies {
       compile 'com.google.code.gson:gson:2.3'
       compile 'com.squareup.retrofit:retrofit:1.9.0'
}
```

上述的代码是基于groovy语法的，所以其完整的表述应该是这样的：

```
dependencies {
      compile group: 'com.google.code.gson', name: 'gson', version:'2.3'
      compile group: 'com.squareup.retrofit', name: 'retrofit'
           version: '1.9.0'
     }
```

4、有些组织，创建了一些有意思的插件或者library,他们更愿意把这些放在自己的maven库，而不是maven中心库或jcenter。那么当你需要是要这些仓库(远程仓库)的时候，你只需要在maven方法中加入url地址就好：

```
repositories {
       maven {
           url "http://repo.acmecorp.com/maven2"
       }
}
```

#### 本地依赖

#### 文件依赖
如果你想为你的工程添加jar文件作为依赖，你可以这样：

```
dependencies {
       compile files('libs/domoarigato.jar')
}
```

如果你这么做，那会很愚蠢，因为当你有很多这样的jar包时，你可以改写为：

```
dependencies {
       compile fileTree('libs')
 } 
```

默认情况下，新建的Android项目会有一个lib文件夹，并且会在依赖中这么定义（即添加所有在libs文件夹中的jar）：

```
dependencies {
       compile fileTree(dir: 'libs', include: ['*.jar'])
}
```

这也意味着，在任何一个Android项目中，你都可以把一个jar文件放在到libs文件夹下，其会自动的将其添加到编译路径以及最后的APK文件。

#### native包（so包）
用c或者c++写的library会被叫做so包，Android插件默认情况下支持native包，你需要把.so文件放在对应的文件夹中：
app
├── AndroidManifest.xml
└── jniLibs
├── armeabi
│ └── nativelib.so
├── armeabi-v7a
│ └── nativelib.so
├── mips
│ └── nativelib.so
└── x86
└── nativelib.so

#### aar文件
如果你想分享一个library,该依赖包使用了Android api，或者包含了Android 资源文件，那么aar文件适合你。依赖库和应用工程是一样的，你可以使用相同的tasks来构建和测试你的依赖工程，当然他们也可以有不同的构建版本。应用工程和依赖工程的区别在于输出文件，应用工程会生成APK文件，并且其可以安装在Android设备上，而依赖工程会生成.aar文件。该文件可以被Android应用工程当做依赖来使用。
创建依赖工程模块，你需要加不同的插件：

```
apply plugin: 'com.android.library'
```

**我们有两种方式去使用一个依赖工程。一个就是在你的工程里面，直接将其作为一个模块，另外一个就是创建一个aar文件，这样其他的应用也就可以复用了。**

1、如果你把其作为模块，那你需要在settings.gradle文件中添加其为模块：

```
  include ':app', ':library'
```

在这里，我们就把它叫做library吧，如果你想使用该模块，你需要在你的依赖里面添加它，就像这样：

```
dependencies {
       compile project(':library')
  }
```

2、如果你想复用你的library，那么你就可以创建一个aar文件，并将其作为你的工程依赖。当你构建你的library项目，aar文件将会在 build/output/aar/下生成。把该文件作为你的依赖包，你需要创建一个文件夹来放置它，我们就叫它aars文件夹吧，然后把它拷贝到该文件夹里面，然后添加该文件夹作为依赖库：

```
repositories {
    flatDir {
        dirs 'aars' 
    }
}
```

这样你就可以把该文件夹下的所有aar文件作为依赖，同时你可以这么干：

```
dependencies {
       compile(name:'libraryname', ext:'aar')
}
```

这个会告诉Gradle，在aars文件夹下，添加一个叫做libraryname的文件，且其后缀是aar的作为依赖。

#### 依赖动态版本
在一些情形中，你可能想使用最新的依赖包在构建你的app或者library的时候。实现他的最好方式是使用动态版本。我现在给你们展示几种不同的动态控制版本方式：

```
dependencies {
       compile 'com.android.support:support-v4:22.2.+'    //得到最新的生产版本
       compile 'com.android.support:appcompat-v7:22.2+'   //得到最新的minor版本，并且其最小的版本号是2
       compile 'com.android.support:recyclerview-v7:1.2.2'    //得到1.2.2版本
}
```

你应该小心去使用动态版本，如果当你允许gradle去挑选最新版本，可能导致挑选的依赖版本并不是稳定版，这将会对构建产生很多问题，更糟糕的是你可能在你的服务器和私人pc上得到不同的依赖版本，这直接导致你的应用不同步。


