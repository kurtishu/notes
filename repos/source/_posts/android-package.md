---
title: Android 打包
date: 2017-06-27 17:15:07
categories: [Android, 其他]
tags: [打包, package]
---
Android打包也称作构建，及Android 构建系统编译应用资源和源代码，然后将它们打包成可供您测试、部署、签署和分发的 APK
之前一直使用的Ant作为Android构建工具，目前基本上使用Gradle 这一高级构建工具包来自动化执行和管理构建流程的。

### Android 打包流程
构建流程涉及许多将您的项目转换成 Android 应用软件包 (APK) 的工具和流程。
![典型 Android 应用模块的构建流程](https://developer.android.google.cn/images/tools/studio/build-process_2x.png)
如图所示，典型 Android 应用模块的构建流程通常依循下列步骤：
- [x] 编译器将您的源代码转换成 DEX（Dalvik Executable) 文件（其中包括运行在 Android 设备上的字节码），将所有其他内容转换成已编译资源。
- [x] APK 打包器将 DEX 文件和已编译资源合并成单个 APK。不过，必须先签署 APK，才能将应用安装并部署到 Android 设备上。
- [x] APK 打包器使用调试或发布密钥库签署您的 APK：
  * 如果您构建的是调试版本的应用（即专用于测试和分析的应用），打包器会使用调试密钥库签署您的应用。Android Studio 自动使用调试密钥库配置新项目。
  * 如果您构建的是打算向外发布的发布版本应用，打包器会使用发布密钥库签署您的应用。要创建发布密钥库，请阅读在 Android Studio 中签署您的应用。
- [x] 在生成最终 APK 之前，打包器会使用 zipalign 工具对应用进行优化，减少其在设备上运行时的内存占用。

构建流程结束时，您将获得可用来进行部署、测试的调试 APK，或者可用来发布给外部用户的发布 APK。

### 使用Ant打包
使用Eclipse的时期，都是借助Ant作为Android的构建工具，不过现在基本上都是使用Android studio作为开发工具，并使用更强大的Gradle构建工具，因此这个构建方式已经被淘汰了。
#### Ant安装
ant的安装比较简单，下载ant压缩包  http://ant.apache.org  ，配置环境变量。
#### 为Android项目生成Ant配置build.xml
```java
android update project --name <project_name> --target <target_ID>--path <path_to_your_project>
```
项目根目录下多了build.xml，以及local.properties两个文件。

　　其中local.properties写明了我们的android SDK的目录(其实是环境变量ANDROID_HOME的值，所以如果环境变量中没有这个的，请增加)。
　　build.xml则是ant构建的最重要脚本，打开一看，发现里面其实大部分都是写注释，有用的没几行，这是因为生成的这个build.xml引用了android SDK自带的构建脚本，具体目录是{sdk目录}/tools/ant/build.xml 。
#### 编译打包项目
在项目的根目录下建一个ant.properties文件，输入如下内容，其中keystore密码和alias密码可以不指定（防泄漏），那么在命令执行的过程中会要求你输入
```java
   #keystore的路径，必须使用正斜杠  
   key.store= "E:/wp_android_sample/me.key" 
   #keystore的密码  
   key.store.password=*****
   #alias名  
   key.alias=me
   #alias密码  
   key.alias.password=****** 
```
在项目根目录下运行 ant release 命令就会帮你生成一个经过签名和aligned的apk，生成的apk（your_project_name-release.apk）在bin目录下

### 使用gradle 打包
通常有2种打包方式：

Android Studio图形界面 点击run按钮
命令行方式 gradlew assembleDebug, gradlew assembleRelease
方式1使用自动生成的debug keystore签名；方式2如果是Release包使用release keystore签名，如果是Debug包则使用debug keystore签名。

#### 配置 Gradle 以签署您的 APK

```java
android {
    ...
    defaultConfig { ... }
    signingConfigs {
        release {
            storeFile file("my-release-key.jks")
            storePassword "password"
            keyAlias "my-alias"
            keyPassword "password"
        }
    }
    buildTypes {
        release {
            signingConfig signingConfigs.release
            ...
        }
    }
}
```
#### Build项目
在您的项目根目录中打开一个命令行，并调用 assembleRelease 任务：
```java
gradlew assembleRelease
```
这将在 project_name/module_name/build/outputs/apk/ 中创建一个名为 module_name-release.apk 的 APK。这个 APK 文件已经使用 build.gradle 文件中指定的私钥签署，并使用 zipalign 进行了对齐。

### 手动签名

#### 生成一个私钥
使用 keytool 生成一个私钥。例如：
```java
keytool -genkey -v -keystore my-release-key.jks
-keyalg RSA -keysize 2048 -validity 10000 -alias my-alias
```
> 注：keytool 位于 JDK 中的 bin/ 目录中。要从 Android Studio 查找您的 JDK，选择 File > Project Structure，然后点击 SDK Location，您将看到 JDK location.
#### zipalign 对齐
使用 zipalign 对齐未签署的 APK
```java
zipalign -v -p 4 my-app-unsigned.apk my-app-unsigned-aligned.apk
```
zipalign 可以确保所有未压缩的数据的开头均相对于文件开头部分执行特定的字节对齐，这样可减少应用消耗的 RAM 量。

#### 通过 apksigner 签名
```java
apksigner sign --ks my-release-key.jks --out my-app-release.apk my-app-unsigned-aligned.apk
```

#### 验证您的 APK 是否已签署：
```java
apksigner verify my-app-release.apk
```

### 其他
在做自动化打包的时候，有时候需要获取版本控制工具的版本号，以此来作为将要构建的APK的versionName
一下列出了两个常用的版本控制工具SVN、Git获取版本号的方法：
#### SVN获取版本号
```java
uildscript {

    dependencies {
	    // 添加SVN的插件
        classpath group: 'org.tmatesoft.svnkit', name: 'svnkit', version: '1.8.9'
    }
}

def getSvnRevision() {
    org.tmatesoft.svn.core.wc.ISVNOptions options = SVNWCUtil.createDefaultOptions(true);
    org.tmatesoft.svn.core.wc.SVNClientManager clientManager = SVNClientManager.newInstance(options);
    org.tmatesoft.svn.core.wc.SVNStatusClient statusClient = clientManager.getStatusClient();
    org.tmatesoft.svn.core.wc.SVNStatus status = statusClient.doStatus(projectDir, false);
    org.tmatesoft.svn.core.wc.SVNRevision revision = status.getRevision();
    return revision.getNumber();
}


ext {
    // JDK version
    sourceCompatibility = JavaVersion.VERSION_1_8
    targetCompatibility = JavaVersion.VERSION_1_8

    // App version
    appVersion = "2.0.0.REVISION"
    appVersionCode = 123456
    appVersionName = appVersion.replace(".REVISION", "." + getSvnRevision())
}
```

#### Git 获取版本号

采用java来实行process
```java

def getGitVersion() {
  return 'git rev-parse --short HEAD'.execute().text.trim()
}
```
如果我们要把取到的Git提交的version显示到界面上

```java
defaultConfig {
	minSdkVersion rootProject.ext.minSdkVersion
	targetSdkVersion rootProject.ext.targetSdkVersion
	buildConfigField "String", "GIT_REVISION", "\"${getGitVersion()}\""
    }
```
这样系统会自动生成BuildConfig这个类，类里面就有

```java
  public static final String GIT_REVISION = "0";
  
  // 获取直接赋值给AppVersionName
  ext {
    // JDK version
    sourceCompatibility = JavaVersion.VERSION_1_8
    targetCompatibility = JavaVersion.VERSION_1_8

    // App version
    appVersion = "2.0.0.REVISION"
    appVersionCode = 123456
    appVersionName = appVersion.replace(".REVISION", "." + getGitVersion())
}
```
### 总结
详细的流程图
![流程图](https://user-gold-cdn.xitu.io/2016/11/29/8643b670708aa421a9883df897496289)
我们可以将整个打包过程概括为以下几步：

1. 通过aapt打包res资源文件，生成R.java、resources.arsc和res文件（二进制 & 非二进制如res/raw和pic保持原样）
2. 处理.aidl文件，生成对应的Java接口文件
3. 通过Java Compiler编译R.java、Java接口文件、Java源文件，生成.class文件
4. 通过dex命令，将.class文件和第三方库中的.class文件处理生成classes.dex
5. 通过apkbuilder工具，将aapt生成的resources.arsc和res文件、assets文件和classes.dex一起打包生成apk
6. 通过zipalign工具，对apk进行对齐处理。
7. 通过apksigner/Jarsigner工具，对上面的apk进行debug或release签名

### 参考
https://developer.android.google.cn/studio/build/index.html
https://developer.android.google.cn/studio/publish/app-signing.html
https://developer.android.google.cn/studio/publish/app-signing.html#release-mode
[Android 打包系列——打包流程梳理](https://juejin.im/entry/57a5bc735bbb500064246d00)
https://developer.android.google.cn/studio/command-line/index.html
https://developer.android.google.cn/studio/command-line/apksigner.html
[Ant自动编译打包&发布 android项目](http://www.cnblogs.com/yaozhongxiao/p/3523061.html)
