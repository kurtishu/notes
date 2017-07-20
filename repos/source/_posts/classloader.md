---
title: 浅谈 Java ClassLoader
date: 2017-07-19 10:49:15
categories: [Java, ClassLoader]
tags: [ClassLoader, 双亲委托, 自定义加载器]
---

![Image](http://www.cssxt.com/uploadfile/2016/0902/20160902024039799.png)

### 什么是ClassLoader
ClassLoader 就是所谓的类加载器，我们在编写的java程序的时候，首先会将源文件(.java)通过Java编译器编译成Java字节代码（.class）。类加载器将.class文件中二进制数据读入到内存中，将其放在运行时数据区的方法区中，然后在堆区创建一个java.lang.Class对象，用来封装类在方法区内的数据结构。
ClassLoader是用来动态加载class文件到内存中的，一旦要加载的class文件不存在就会报ClassNotFoundException。 
### Java中的ClassLoader类型

#### Java虚拟机自带的加载器
Java默认提供的三个ClassLoader：
*  根类加载器（Bootstrap ClassLoader）
> 启动类加载器，是用C++语言写的，是虚拟机自身的一部分。它是在Java虚拟机启动后初始化的，它用来加载java核心的API， 如：rt.jar、resources.jar、charsets.jar等，以满足java程序最基本的需求，是用原生代码实现的，并不继承自java.lang.ClassLoader。

*  扩展类加载器（Extension ClassLoader）
> 扩展类加载器,该加载器使用java写的，它用来加载Java的扩展类库，默认加载JAVA_HOME/jre/lib/ext/目下的所有jar。

*  系统类加载器（System ClassLoader）
> 也叫做应用类加载器(App ClassLoader), 它也是用java写的, 负责加载应用程序classpath目录下的所有jar和class文件。通常，在没有指定ClassLoader的情况下，程序员自定义的类就由该ClassLoader加载。ClassLoader中有个getSystemClassLoader方法,此方法返回的正是AppclassLoader.

接下来通过代码查看下ClassLoader以及其加载文件的路径
```java	
public class TestDemo {

	public static void main(String[] args) {
	    // 1. 获取当前类的ClassLoader
		ClassLoader mainClassLoader = TestDemo.class.getClassLoader();
		System.out.println("Main Class Loader's Name= " + mainClassLoader.toString());
		
		// 2. 获取当前类的父的ClassLoader
		ClassLoader parentClassLoader = mainClassLoader.getParent();
		System.out.println("Parent Class Loader's Name= " + parentClassLoader.toString());
		// 3. 获取父的ClassLoder的父的ClassLoader，也就是曾祖父ClassLoader
		ClassLoader superClassLoader = parentClassLoader.getParent();
		System.out.println("Super Class Loader's Name= " + superClassLoader);
		
		System.out.println("==========AppClassLoaderPath================== ");
		// 4. 打印当前类的ClassLoader加载的文件路径
		URLClassLoader mainUrlClassLoader = (URLClassLoader) mainClassLoader;
		URL[] mainURLs = mainUrlClassLoader.getURLs();
		printUrls(mainURLs);
		
		System.out.println("===========ExtClassLoaderPath================= ");
		// 5. 打印当前类的Parent的ClassLoader加载的文件路径
		URLClassLoader parentUrlClassLoader = (URLClassLoader) parentClassLoader;
		URL[] parentURLs = parentUrlClassLoader.getURLs();
		printUrls(parentURLs);
		
		System.out.println("=========BootstrapClassLoaderPath=================== ");
		// 5. 打印的BootstrapClassLoader ClassLoader加载的文件路径
		try {
			// URL[] urls = sun.misc.Launcher.getBootstrapClassPath().getURLs();
			Class clazz = Class.forName("sun.misc.Launcher");
			Method method = clazz.getDeclaredMethod("getBootstrapClassPath", null);
			method.setAccessible(true);
			Object loaderClazz = method.invoke(clazz, null);
			Method urlMethod = loaderClazz.getClass().getDeclaredMethod("getURLs", null);
			urlMethod.setAccessible(true);
			URL[] superURLs = (URL[]) urlMethod.invoke(loaderClazz, null);
			printUrls(superURLs);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
```

输出的结果如下：
```java
// 1. 当前类的ClassLoader为AppClassLoader
Main Class Loader's Name= sun.misc.Launcher$AppClassLoader@73d16e93
// 2. 它的父的ClassLoader为ExtClassLoader
Parent Class Loader's Name= sun.misc.Launcher$ExtClassLoader@15db9742
// 3. 它的曾祖父的ClassLoader为空，为什么为空，留个问题，稍后解释？
Super Class Loader's Name= null

// 4. AppClassLoaderPath 为当前工程的Classpath
==========AppClassLoaderPath================== 
file:/C:/Users/kurtis.hu/workspace/DateSimple/bin/

// 5. ExtClassLoaderPath为jre /lib/ext目录下的文件
===========ExtClassLoaderPath================= 
file:/C:/Program%20Files/Java/jre1.8.0_91/lib/ext/access-bridge-64.jar
file:/C:/Program%20Files/Java/jre1.8.0_91/lib/ext/cldrdata.jar
file:/C:/Program%20Files/Java/jre1.8.0_91/lib/ext/dnsns.jar
file:/C:/Program%20Files/Java/jre1.8.0_91/lib/ext/jaccess.jar
file:/C:/Program%20Files/Java/jre1.8.0_91/lib/ext/jfxrt.jar
file:/C:/Program%20Files/Java/jre1.8.0_91/lib/ext/localedata.jar
file:/C:/Program%20Files/Java/jre1.8.0_91/lib/ext/nashorn.jar
file:/C:/Program%20Files/Java/jre1.8.0_91/lib/ext/sunec.jar
file:/C:/Program%20Files/Java/jre1.8.0_91/lib/ext/sunjce_provider.jar
file:/C:/Program%20Files/Java/jre1.8.0_91/lib/ext/sunmscapi.jar
file:/C:/Program%20Files/Java/jre1.8.0_91/lib/ext/sunpkcs11.jar
file:/C:/Program%20Files/Java/jre1.8.0_91/lib/ext/zipfs.jar

// 6. BootstrapClassLoaderPath 为jre lib获取其他class文件
=========BootstrapClassLoaderPath=================== 
file:/C:/Program%20Files/Java/jre1.8.0_91/lib/resources.jar
file:/C:/Program%20Files/Java/jre1.8.0_91/lib/rt.jar
file:/C:/Program%20Files/Java/jre1.8.0_91/lib/sunrsasign.jar
file:/C:/Program%20Files/Java/jre1.8.0_91/lib/jsse.jar
file:/C:/Program%20Files/Java/jre1.8.0_91/lib/jce.jar
file:/C:/Program%20Files/Java/jre1.8.0_91/lib/charsets.jar
file:/C:/Program%20Files/Java/jre1.8.0_91/lib/jfr.jar
file:/C:/Program%20Files/Java/jre1.8.0_91/classes
```

#### 用户自定义的类加载器
在绝大多数情况下系统默认提供的类加载器实现已经可以满足需求。但是在某些情况下，还是需要为应用开发出自己的类加载器。比如您的应用通过网络来传输 Java 类的字节代码，为了保证安全性，这些字节码经过了加密处理。这个时候您就需要自己的类加载器来从某个网络地址上读取加密后的字节代码，接着进行解密和验证，最后定义出要在 Java 虚拟机中运行的类来。

具体的实现后面会详细讲解。

### ClassLoader加载类的原理
#### 双亲委托模型
所谓的双亲委托模型，也称作父类委托模式，是指某个类加载器要加载一个类的时候，不是自己先加载而是交个其父加载器（这里的父子关系不是继承关系，是包含关系, 即ClasLoader对象中有Parent classLoader的引用）, 依次递归，如果父类加载器可以完成类加载任务，就成功返回；只有父类加载器无法完成此加载任务时，才自己去加载。
![类加载过程](http://img.my.csdn.net/uploads/201202/25/0_13301699801ybH.gif)

直接上代码, 直接查看java.lang.ClassLoader 源码  
以下代码基于JDK 1.8.0_91：
```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 首先检查该类是否已经被加载过，如果已经加载过直接返回
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    // 父加载器不为空则调用父加载器的loadClass
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        //父加载器为空则调用Bootstrap Classloader
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                     //父加载器没有找到，则调用findclass
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```
以上代码解释了双亲委托模型，看懂了么！
另外，要注意的是如果要编写一个classLoader的子类，也就是自定义一个classloader，建议覆盖findClass()方法，而不要直接改写loadClass()方法

#### 双亲委托模型优势
主要从一下几个方面考虑的：
* 可以避免重复加载：当父亲已经加载了该类的时候，就没有必要子ClassLoader再加载一次。
* 安全因素： 限制避免了用户自己的代码冒充核心类库的类访问核心类库包可见成员的情况


** 遗留问题 ：ExtClassLoader 的父ClassLoader为什么为空？ ** 
还是直接看源码，从源码中找答案是最直接的：
看[sun.misc.Launcher](http://www.grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8u40-b25/sun/misc/Launcher.java),它是一个java虚拟机的入口应用。
```java
 public More ...Launcher() {
         // Create the extension class loader
         ClassLoader extcl;
         try {
             // 1. 初始化 ExtClassLoader
             extcl = ExtClassLoader.getExtClassLoader();
         } catch (IOException e) {
             throw new InternalError(
                 "Could not create extension class loader", e);
         }
 
         // Now create the class loader to use to launch the application
         try {
             // 2. 初始化 AppClassLoader，将extcl作为Parent ClassLoader
             loader = AppClassLoader.getAppClassLoader(extcl);
         } catch (IOException e) {
             throw new InternalError(
                 "Could not create application class loader", e);
         }
 
         // Also set the context class loader for the primordial thread.
         // 设置AppClassLoader为线程上下文类加载器，
         Thread.currentThread().setContextClassLoader(loader);
 
         // Finally, install a security manager if requested
         String s = System.getProperty("java.security.manager");
         if (s != null) {
             SecurityManager sm = null;
             if ("".equals(s) || "default".equals(s)) {
                 sm = new java.lang.SecurityManager();
             } else {
                 try {
                     sm = (SecurityManager)loader.loadClass(s).newInstance();
                 } catch (IllegalAccessException e) {
                 } catch (InstantiationException e) {
                 } catch (ClassNotFoundException e) {
                } catch (ClassCastException e) {
                }
            }
            if (sm != null) {
                System.setSecurityManager(sm);
            } else {
                throw new InternalError(
                    "Could not create SecurityManager: " + s);
            }
        }
    }

  * Returns the class loader used to launch the main application.
     */
    public ClassLoader getClassLoader() {
        return loader;
    }
    /*
     * The class loader used for loading installed extensions.
     */
    static class ExtClassLoader extends URLClassLoader {...}

/**
     * The class loader used for loading from java.class.path.
     * runs in a restricted security context.
     */
    static class AppClassLoader extends URLClassLoader {...}

```
源码有精简，我们可以得到相关的信息。 
1. Launcher初始化了ExtClassLoader和AppClassLoader。 
2. Launcher中并没有看见BootstrapClassLoader，但通过System.getProperty("sun.boot.class.path")得到了字符串bootClassPath,这个应该就是BootstrapClassLoader加载的jar包路径。
3. ExtClassLoader没有Parent ClassLoader，所以获取它parentClassLoader为null
4. AppClassLoader和ExtClassLoader都是继承URLClassLoader，这就是为什么前面可以强转的原因。

** 结合ClassLoad的loadClass方法，即使ExtClassLoader没有Parent ClassLoader，但是其还是会使用BootstrapClassLoader加载class文件，所以还是会将BootstrapClassLoader称作ExtClassLoader的Parent ClassLoader**。

### 定义自已的ClassLoader
要创建自定义的类加载器，只要扩展java.lang.ClassLoader类，然后覆盖它的findClass（String name） 方法即可，该方法根据参数指定的类的名字，返回对应的Class对象的引用。
ClassLoader源码中给了个Example, 可以参照写一个自定义的ClassLoader
```java
class NetworkClassLoader extends ClassLoader {
         String host;
         int port;

         public Class findClass(String name) {
             byte[] b = loadClassData(name);
             return defineClass(name, b, 0, b.length);
         }

         private byte[] loadClassData(String name) {
             // load the class data from the connection
              . . .
         }
     }
```
1. 自定义ClassLoader继承ClassLoader类
```java

public class MyClassLoader extends ClassLoader {
	 static {
	        System.out.println("MyClassLoader");
	   }
	 
	 private static final String URL = "C:/Users/kurtis.hu/workspace/DateSimple/bin/com/dreamfactoy/";
	private static final String CLASS = ".class";

	@Override
	protected Class<?> findClass(String name) throws ClassNotFoundException {
		 // 加载指定路径下的.class文件
		 byte[] b = loadClassData(name);
         return defineClass(name, b, 0, b.length);
	}
	 
	private byte[] loadClassData(String name) {
	  // load the class data from the connection
		FileInputStream fis = null;
        byte[] data = null;
        try {
            File file = new File(URL + name + CLASS);
            System.out.println(file.getAbsolutePath());
            fis = new FileInputStream(file);
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            int ch = 0;
            while ((ch = fis.read()) != -1) {
                baos.write(ch);
            }
            data = baos.toByteArray();
        } catch (IOException e) {
            e.printStackTrace();
            System.out.println("loadClassData-IOException");
        }
        return data;
     }
```
2. 编写需要加载的Class文件，待会指定它作为要加载的文件
```java
package com.dreamfactoy;

public class Hello {
	static {
		System.out.println("Hello Class loaded!");
	}
	
	public Hello() {
		System.out.println("This is the Hello Class");
	}
}
```
3. 编写测试类
```java
public static void main(String[] args) {
		try {
			MyClassLoader myClassLoader = new MyClassLoader();
			// 如何测试类有包名，必须写全路径
			Class<?> clazz = myClassLoader.loadClass("com.dreamfactoy.Hello");
			Object obj = clazz.newInstance();
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	}
```

4. 测试结果
```java
MyClassLoader
Hello Class loaded!
This is the Hello Class
```

最后要说下defineClass，Java SDK 用native方法实现了加载二进制文件到内存，开发者可以不用关系具体过程。
```java
    protected final Class<?> defineClass(String name, byte[] b, int off, int len,
                                         ProtectionDomain protectionDomain)
        throws ClassFormatError
    {
        protectionDomain = preDefineClass(name, protectionDomain);
        String source = defineClassSourceLocation(protectionDomain);
        // defineClass1 是native方法
		Class<?> c = defineClass1(name, b, off, len, protectionDomain, source);
        postDefineClass(c, protectionDomain);
        return c;
    }
```

### 总结
1. ClassLoader用来加载class文件的。
2. 系统内置的ClassLoader通过双亲委托来加载指定路径下的class和资源。
3. 自定义ClassLoader一般覆盖findClass()方法。

以上是Java的类加载，然而[Android也有一套动态加载工作机制](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/1130/3730.html)，这个以后再详细学习后再做描述。

### 参考
[一看你就懂，超详细java中的ClassLoader详解](http://blog.csdn.net/briblue/article/details/54973413)
[深入分析Java ClassLoader原理](http://blog.csdn.net/xyang81/article/details/7292380)