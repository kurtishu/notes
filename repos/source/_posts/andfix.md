---
title: AndFix 使用
date: 2017-06-07 15:44:02
categories: [Android, Hot Fix]
tags: [AndFix, Hot fix]
---
### AndFix 简介
AndFix是由Alibaba开源的Android App在线热补丁框架。使用此框架，我们能够在不重复发版的情况下，在线修改App中的Bug。AndFix就是 “Android Hot-Fix”的缩写。
AndFix支持Android 2.3到7.0版本，并且支持arm 与 X86系统架构的设备。支持Dalvik与ART的Runtime
AndFix Github地址 [https://github.com/alibaba/AndFix](https://github.com/alibaba/AndFix)

### 原理
AndFix的原理就是方法的替换，把有bug的方法替换成补丁文件中的方法。
![AndFix的原理](https://github.com/alibaba/AndFix/raw/master/images/principle.png)

### Bug 修复流程
![Bug 修复流程](https://github.com/alibaba/AndFix/raw/master/images/process.png)

### 集成步骤
#### 添加AndFix Library依赖
```java
dependencies {
	compile 'com.alipay.euler:andfix:0.5.0@aar'
}
```

#### 初始化PatchManager
```java
public class AndFixApplication extends Application {

    private static AndFixApplication mInstance;

    private PatchManager mPatchManager;

    public void onCreate() {
        super.onCreate();
        mInstance = this;
        
        // 初始化patch管理类
        mPatchManager = new PatchManager(this);

        // 初始化patch版本
        mPatchManager.init("1.0");
//        String appVersion = getPackageManager().getPackageInfo(getPackageName(), 0).versionName;
//        mPatchManager.init(appVersion);

        // 加载已经添加到PatchManager中的patch
        mPatchManager.loadPatch();

    }

    public static PatchManager getPatchManager() {
        return mInstance.mPatchManager;
    }
}
```
#### 修改代码
会出现Bug的代码
```java
public class MainActivity extends AppCompatActivity {

    // 补丁文件名
    private static final String APATCH_PATH = "/fix.apatch";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    public void showToast(View view) {
        Toast.makeText(this, "打补丁之前", Toast.LENGTH_LONG).show();
    }

    public void onClick(View view) {
        String patchFileStr = Environment.getExternalStorageDirectory().getAbsolutePath() + APATCH_PATH;
        try {
            AndFixApplication.getPatchManager().addPatch(patchFileStr);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
PatchManager的addPatch方法加载新补丁，项目中可以在下载补丁文件之后调用,这里为了演示就把补丁文件放在本地的SD卡中了，代码中patchFileStr就是补丁存放的位置，.apatch就是生成补丁文件的后缀名，fix就是补丁文件的名字，这里我们将其名字写死。

#### 将以上应用打包，我们命名为app-release_V1.0.apk
#### 修改代码，再次打包成app-release_V2.0.apk
```java 
    public void showToast(View view) {
        Toast.makeText(this, "打补丁之后", Toast.LENGTH_LONG).show();
    }

```

#### 使用官方提供的工具apkpatch生成.apatch补丁文件
apkpatch [下载地址](https://github.com/alibaba/AndFix/blob/master/tools/apkpatch-1.0.3.zip)
 * 点击上面的链接下载apkpatch之后解压
 * 将两个apk文件和该app的签名文件放入到该目录中
 * 使用命令生成补丁

命令格式：
> apkpatch.sh(bat) -f 新apk -t 旧apk -o 输出目录 -k app签名文件 -p 签名文件密码 -a 签名文件别名 -e 别名密码

```java
Kurtis-Hu:apkpatch-1 kurtishu$ ./apkpatch.sh  -f app-release_V2.0.apk -t app-release_V1.0.apk -o output -k keystore.jks -p abc123_ -a kurtis -e abc123_
```
 * 生成的output文件,获取.patch文件
 * 将生成的.apatch补丁文件改成代码中写死的fix.apatch

#### 开始修复
使用adb push 命令将patch放入SD卡中，运行apkV1.0验证修复
```java
kurtishu$ adb push /Users/kurtishu/Desktop/Workspace/apkpatch-1/output/fix.apatch /sdcard/ 
/Users/kurtishu/Desktop/Workspace/apkpatch-1/output/fix.apatch: 1 file pushed. 1.0 MB/s (3320 bytes in 0.003s)
```
具体参考[Android热修复之AndFix使用教程](http://www.jianshu.com/p/907a2c599996)

### 大致原理
apkpatch将两个apk做一次对比，然后找出不同的部分。可以看到生成的apatch了文件，后缀改成zip再解压开，里面有一个dex文件。通过jadx查看一下源码，里面就是被修复的代码所在的类文件,这些更改过的类都加上了一个_CF的后缀，并且变动的方法都被加上了一个叫@MethodReplace的annotation，通过clazz和method指定了需要替换的方法。

然后客户端sdk得到补丁文件后就会根据annotation来寻找需要替换的方法。最后由JNI层完成方法的替换

### 局限性

* 不支持YunOS
* 无法添加新类和新的字段
* 需要使用加固前的apk制作补丁，但是补丁文件很容易被反编译，也就是修改过的类源码容易泄露。
* 使用加固平台可能会使热补丁功能失效（看到有人在360加固提了这个问题，自己还未验证）。

### 总结
AndFix 的使用是非常方便的，

### 参考
> [Android插件化之使用AndFix进行Hot fix](https://mp.weixin.qq.com/s?__biz=MzI0MjE3OTYwMg==&mid=2649547594&idx=1&sn=5428ab6ec18c21df37db7423e009418c&scene=21#wechat_redirect)








