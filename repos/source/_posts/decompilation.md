---
title: Android 反编译
date: 2017-06-19 15:52:52
categories: [Android, 其他]
tags: [反编译, 打包]
---
在Android开发过程中有时候会好奇想知道别人的项目是怎么架构的，用到那些第三方库，或者某一个功能是怎么实现的等等。
无奈我们只有别人的APK文件，没法看到源码，这时候就需要用到反编译技术了。

### APK编译的原理
说起反编译首先要只要APK是怎么被编译出来的，也就是说要了解APK的构建原理以及步骤
具体可以参考[《Android 打包那些事》](http://kurtishu.github.io/2017/06/27/android-package)

#### APK内容分析
APK的实质是一个压缩文件，类似我们常见的zip、rar格式的文件，可以直接加压缩，下图为APK解压缩的文件结构图：
![APK 解压缩后](http://upload-images.jianshu.io/upload_images/1430132-9db81651ccbf76a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* assets文件夹：原始资源文件夹，对应着Android工程的assets文件夹，一般用于存放原始的图片、txt、css等资源文件。
* lib：存放应用需要的引用第三方SDK的so库。比如一些底层实现的图片处理、音视频处理、数据加密的库等。而该文件夹下有时会多一个层级，这是根据不同CPU 型号而划分的，如 ARM，ARM-v7a，x86等。
* META-INF：保存apk签名信息，保证apk的完整性和安全性。
* res：资源文件夹，其中的资源文件包括了布局(layout)，常量值(values)，颜色值(colors)，尺寸值(dimens)，字符串(strings)，自定义样式(styles)等。
* AndroidManifest.xml文件：全局配置文件，里面包含了版本信息、activity、broadcasts等基本配置。不过这里的是二进制的xml文件，无法直接查看，需要反编译后才能查看。
* classes.dex文件：这是安卓代码的核心部分，dex是在Dalvik虚拟机上可以执行的文件。这里有classes.dex和classes2.dex两个文件，说明工程的方法数较多，进行了dex拆分。
* resources.arsc文件：记录资源文件和资源id的映射关系。

对于assets和res里面的一些资源文件例如图片文件是可以直接查看的(有些APK解压后图片也是无法被查看)，但是其他例如dex文件就需要借助其他的工具才能查看。

### 反编译工具
#### Apktools-目前最强大的反编译工具
Apktool [官网](https://ibotpeaches.github.io/Apktool/)
> It is a tool for reverse engineering 3rd party, closed, binary Android apps. It can decode resources to nearly original form and rebuild them after making some modifications; it makes possible to debug smali code step by step. Also it makes working with app easier because of project-like files structure and automation of some repetitive tasks like building apk, etc.

它不仅能够反编译apk，解析出资源文件，xml文件，生成smali文件，还可以把修改后的文件你想生成apk。作用：资源文件获取，可以提取出图片文件和布局文件进行使用查看

#### dex2jar
将apk中的classes.dex转化成Jar文件。
[dex2jar下载地址](https://bitbucket.org/pxb1988/dex2jar)

#### jd-gui
查看APK中classes.dex转化成出的jar文件.
[jd-gui下载地址](http://jd.benow.ca/)

#### ClassyShark
> ClassyShark is an Android executables browser. It can reliably open any Android executable and analyse its content. ClassyShark supports multiple formats including jar, class, apk, dex, so, aar and Android XML.

[ClassyShark 下载地址](http://classyshark.com/)
ClassyShark 是一个可视化的查看Android APK内容(源代码，资源文件等)的工具。
它使用起来很方便，下载下来之后是一个可执行的jar文件，在终端执行`java -jar classyshark.jar`即可打开图形化界面。在打开的图形操作界面中拖入待目标apk，即可展示出反编译之后的结果。
![ClassyShark](http://upload-images.jianshu.io/upload_images/1430132-4d0b0120a2cf6429.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**Methods count**里面即可看到引用了哪些包，方法数。

#### APK Analyzer
Android Studio新增了APK Analyzer功能，使用方法很简单，只需要将目标apk拖入到Android Studio中即可。

### 防止APK程序被反编译

#### 代码混淆(Proguard)
所谓代码混淆就是把源代码中有意义的变量名，类名用简单字母代替，使得代码变得不可读，进而达到保护源代码的目的。

1. Gradle ProGuard配置
在Android Studio中，使用构建工具是Gradle，要启动ProGuard,可以在Module的build.gradle文件中设置：
```java
android {
    buildTypes {
        release {
		    // ProGuard 启用代码压缩
            minifyEnabled true
			// 启用资源压缩
			shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android.txt'),
                    'proguard-rules.pro'
        }
    }
    ...
}
```

proguardFiles指定ProGuard配置文件，proguard-rules.pro是Module下的自定义配置文件；

接下来就是要配置ProGuard规则：
这里给出一个Android项目的ProGuard规则[来自官网的androidapplication example](https://www.guardsquare.com/en/proguard/manual/examples#androidapplication)
参考：
https://developer.android.google.cn/studio/build/shrink-code.html#shrink-resources
2. DexGuard
[官网地址](https://www.guardsquare.com/en/dexguard), 它比ProGuard更安全，功能更多，不过是商业软件，需要收费。

#### 代码加固
代码加固一般使用第三方软件，比如360加固、百度加固、腾讯乐固等
[Android市面常见加固方案评测](http://www.jianshu.com/p/340507049ded)

#### 使用Native层
一些关键的算法，或者数据可以使用C/C++来实现，毕竟反编译so文件要比java文件的难度大太多了。

### 总结
安卓应用主要基于Java开发，可以借助一些反编译工具可以轻松的破解，获取源代码或者接口信息等，所以为了避免代码泄漏，我们需要对我们的源码进行混淆、加固处理。加大反编译的难度。
作为一个有节操的程序员我们出于学习的目的反编译别人项目是可以的，切不可以商业目的窃取别人源代码，甚至加入广告，病毒等二次打包发布。

### 更多
Apk混淆成中文语言代码或者其他语言-[《Android安全防护之旅—带你把Apk混淆成中文语言代码》](http://www.wjdiankong.cn/android%E5%AE%89%E5%85%A8%E9%98%B2%E6%8A%A4%E4%B9%8B%E6%97%85-%E5%B8%A6%E4%BD%A0%E6%8A%8Aapk%E6%B7%B7%E6%B7%86%E6%88%90%E4%B8%AD%E6%96%87%E8%AF%AD%E8%A8%80%E4%BB%A3%E7%A0%81/)

### 参考

[Android APK 反编译实践](http://www.jianshu.com/p/9e0d1c3e342e)
https://developer.android.google.cn/studio/build/shrink-code.html#shrink-resources
http://www.jianshu.com/p/340507049ded
http://proguard.sourceforge.net/index.html
