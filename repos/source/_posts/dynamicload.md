---
title: Android 动态加载机制
date: 2017-07-20 08:51:35
categories: [Android, ClassLoader]
tags: [DynamicLoad, ClassLoader, 双亲委托, 自定义加载器]
---
试想下我们如何让Android应用，像Eclipse一样，可以动态加载插件？ 如何让Android应用执行服务器上的代码（也就是最近比较火的插件化或者热修复、热更新等）？ 如何让Android应用家里，而在执行的时解密，以防止破解？ 以上这些可以使用类加载器来灵活的加载执行的类。

### 类加载机制
Android Dalvik/ART虚拟机和Java虚拟机一样，在运行程序时需要将对应的类加载到内存中。具体内容可以参考[《浅谈 Java ClassLoader》](http://kurtishu.github.io/2017/07/19/classloader/)。

Android虚拟机和标准的Java虚拟机有所不同，使用标准的Java虚拟机，我们可以自定义ClassLoader的类加载器。然后通过defineClass方法从一个二进制中加载Class。然而，这在Android里是行不通。原因是Android中的ClassLoader的defineClass方法具体的调用VMClassLoader的defineClass方法，该方法直接抛出“UnsupportedOperationException”。

```java
 @Deprecated
    protected final Class<?> defineClass(byte[] b, int off, int len) throws ClassFormatError {
        throw new UnsupportedOperationException("can't load this type of class file");
    }
```

### Android的虚拟机类加载机制
Android中的类加载机制到底是怎样的呢？下面直接通过代码看下：
```java
@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ClassLoader classLoader = getClassLoader();
        Log.i("ClassLoader", "Activity classLoader = " + classLoader.toString());
        classLoader = classLoader.getParent();
        Log.i("ClassLoader", "parent classLoader = " + classLoader.toString());
   
    }
```
输出结果为
```java
Activity classLoader = dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/com.dreamfactory.recorder-1/base.apk"],nativeLibraryDirectories=[/vendor/lib64, /system/lib64]]]

Activity classLoader's parent = java.lang.BootClassLoader@294fa575
```
可以看见有2个Classloader实例，一个是BootClassLoader（系统启动的时候创建的），另一个是PathClassLoader（应用启动时创建的，用于加载“/data/app/com.dreamfactory.recorder-1/base.apk”里面的类）。由此也可以看出，一个运行的Android应用至少有2个ClassLoader。

其实，在Android系统启动的时候会创建一个Boot类型的ClassLoader实例，用于加载一些系统Framework层级需要的类，我们的Android应用里也需要用到一些系统的类，所以APP启动的时候也会把这个Boot类型的ClassLoader传进来。

这里提到了PathClassLoader，其实aAndroid为我们从ClassLoader派生出了两个类：DexClassLoader和PathClassLoader。使用这两个类可以实现Android的动态加载功能
- [ ] DexClassLoader：这个可以加载"jar/apk/dex"，也可以从SD卡中加载，它也是用的最多的。
- [ ] PathClassLoader： 只能加载已经安装到Android系统中（/data/data...）的APK文件。

### Android 类加载源码分析

我们发现Android SDK 源码里面是不包含dalvik包的，不过可以在线查看源码：[Dalvik 包源码地址](http://androidxref.com/5.1.0_r1/xref/libcore/dalvik/src/main/java/dalvik/system/)。

DexClassLoader 和 PathClassLoader都是继承BaseDexClassLoader
```java

public class DexClassLoader extends BaseDexClassLoader {
    public DexClassLoader(String dexPath, String optimizedDirectory, String librarySearchPath, ClassLoader parent) {
    super(dexPath, new File(optimizedDirectory), librarySearchPath, parent);
    }
    
}

public class PathClassLoader extends BaseDexClassLoader {
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }
    
    public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent) {
        super(dexPath, null, librarySearchPath, parent);
    }
}
```
BaseDexClassLoader的源码:
```java
public class BaseDexClassLoader extends ClassLoader {
    private final DexPathList pathList;
 
    public BaseDexClassLoader(String dexPath, File optimizedDirectory, String librarySearchPath, ClassLoader parent) { 
     // 调用ClassLoader的构造方法，赋值parent classLoader       
     super(parent);
     // 
     this.pathList = new DexPathList(this, dexPath, librarySearchPath, optimizedDirectory);
   }

}

```

以上构造方法中创建了一个DexPathList实例，进去看看
```java

public DexPathList(ClassLoader definingContext, String dexPath,
            String libraryPath, File optimizedDirectory) {
        // 这里是一些参数判空逻辑，所以省略
        // ...

        // 创建dexElements对象, Element抽象类对zip文件和DexFile做了封装,
        this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory);
    }
 
    private static Element[] makeDexElements(ArrayList<File> files,
            File optimizedDirectory) {
        ArrayList<Element> elements = new ArrayList<Element>();
        for (File file : files) {
            ZipFile zip = null;
            DexFile dex = null;
            String name = file.getName();
            if (name.endsWith(DEX_SUFFIX)) {
                 // Raw dex file (not inside a zip/jar).
                 // 如果是.dex文件就加载该文件
                dex = loadDexFile(file, optimizedDirectory);
            } else if (name.endsWith(APK_SUFFIX) || name.endsWith(JAR_SUFFIX)
                    || name.endsWith(ZIP_SUFFIX)) {
                // 如果是.ApK或者.zip文件就转化成ZipFile对象
                zip = new ZipFile(file);
            } 
            ……
            if ((zip != null) || (dex != null)) {
                elements.add(new Element(file, zip, dex));
            }
        }
        return elements.toArray(new Element[elements.size()]);
    }
    
    private static DexFile loadDexFile(File file, File optimizedDirectory)
            throws IOException {
         // optimizedDirectory是否为空来分两条路径执行加载Dex的功能
        if (optimizedDirectory == null) {
            return new DexFile(file);
        } else {
            String optimizedPath = optimizedPathFor(file, optimizedDirectory);
            return DexFile.loadDex(file.getPath(), optimizedPath, 0);
        }
    }
    
    /**
     * Converts a dex/jar file path and an output directory to an
     * output file path for an associated optimized dex file.
     */
    private static String optimizedPathFor(File path,
            File optimizedDirectory) {
        String fileName = path.getName();
        if (!fileName.endsWith(DEX_SUFFIX)) {
            int lastDot = fileName.lastIndexOf(".");
            if (lastDot < 0) {
                fileName += DEX_SUFFIX;
            } else {
                StringBuilder sb = new StringBuilder(lastDot + 4);
                sb.append(fileName, 0, lastDot);
                sb.append(DEX_SUFFIX);
                fileName = sb.toString();
            }
        }
        File result = new File(optimizedDirectory, fileName);
        return result.getPath();
    }
```

看到这里我们明白了，optimizedDirectory是用来缓存我们需要加载的dex文件的，并创建一个DexFile对象，如果它为null，那么会直接使用dex文件原有的路径来创建DexFile
对象。

### 加载类的过程
上面还只是创建了类加载器的实例，其中创建了一个DexFile实例，用来保存dex文件，我们猜想这个实例就是用来加载类的。

Android中，ClassLoader用loadClass方法来加载我们需要的类：
```java
public Class<?> loadClass(String className) throws ClassNotFoundException {
        return loadClass(className, false);
    }
 
    protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {
        Class<?> clazz = findLoadedClass(className);
        if (clazz == null) {
            ClassNotFoundException suppressed = null;
            try {
                clazz = parent.loadClass(className, false);
            } catch (ClassNotFoundException e) {
                suppressed = e;
            }
 
            if (clazz == null) {
                try {
                    clazz = findClass(className);
                } catch (ClassNotFoundException e) {
                    e.addSuppressed(suppressed);
                    throw e;
                }
            }
        }
        return clazz;
    }
```
loadClass方法调用了findClass方法，而BaseDexClassLoader重载了这个方法，得到BaseDexClassLoader看看:
```java
@Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        Class clazz = pathList.findClass(name);
        if (clazz == null) {
            throw new ClassNotFoundException(name);
        }
        return clazz;
    }
```

最终调用了DexPathList的findClass
```java
 public Class findClass(String name) {
        for (Element element : dexElements) {
            DexFile dex = element.dexFile;
            if (dex != null) {
                Class clazz = dex.loadClassBinaryName(name, definingContext);
                if (clazz != null) {
                    return clazz;
                }
            }
        }
        return null;
	    }
```

这里遍历了之前所有的DexFile实例，其实也就是遍历了所有加载过的dex文件，再调用loadClassBinaryName方法一个个尝试能不能加载想要的类

```java
  public Class loadClassBinaryName(String name, ClassLoader loader, List<Throwable> suppressed) {
        return defineClass(name, loader, mCookie, suppressed);
    }

private static Class defineClass(String name, ClassLoader loader, int cookie, List<Throwable> suppressed) {
        Class result = null;
        try {
            result = defineClassNative(name, loader, cookie);
        } catch (NoClassDefFoundError e) {
            if (suppressed != null) {
                suppressed.add(e);
            }
        } catch (ClassNotFoundException e) {
            if (suppressed != null) {
                suppressed.add(e);
            }
        }
        return result;
    }
```

至此，ClassLoader的创建和加载类的过程的完成了。有趣的是，标准JVM中，ClassLoader是用defineClass加载类的，而Android中defineClass被弃用了，改用了loadClass方法，而且加载类的过程也挪到了DexFile中，在DexFile中加载类的具体方法也叫defineClass，不知道是Google故意写成这样的还是巧合。

### 演练动态加载
具体参考[《Android动态加载入门 简单加载模式》](https://segmentfault.com/a/1190000004062952)

[《Android插件化开发之DexClassLoader动态加载dex、jar小Demo》(http://blog.csdn.net/u011068702/article/details/53263442)]

### 总结
Android Dalvik/ART虚拟机和Java虚拟机一样，在运行程序时需要将对应的类的二进制文件加载到内存中。但是Android虚拟机和Java标准虚拟机在加载方式上存在不同。Java 虚拟机通过ClassLoader类的defineClass方法加载，Android这是通过DeFile的defineClass的native 方法加载的。并且Android提供了两个类：DexClassLoader和PathClassLoader，用来实现Android的动态加载。

### 参考
《深入解析Android虚拟机》 (http://download.csdn.net/detail/gaojiaxing/9849358)
[《Android动态加载基础 ClassLoader工作机制》](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/1130/3730.html)
