---
title: Android 打包
date: 2017-06-27 17:15:07
categories: [Android, 其他]
tags: [打包, package]
---

### Android 打包流程

https://developer.android.google.cn/studio/build/index.html
https://developer.android.google.cn/studio/publish/app-signing.html
https://developer.android.google.cn/studio/publish/app-signing.html#release-mode
[Android 打包系列——打包流程梳理](https://juejin.im/entry/57a5bc735bbb500064246d00)
https://developer.android.google.cn/studio/command-line/index.html
https://developer.android.google.cn/studio/command-line/apksigner.html
### 使用Ant打包

### 使用gradle 打包


### 其他
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

### Git 获取版本号

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



