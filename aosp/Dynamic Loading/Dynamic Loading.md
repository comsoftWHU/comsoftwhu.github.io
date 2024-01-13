---
layout: default
title: Dynamic Loading
nav_order: 3
parent: Dynamic Loading
grand_parent: AOSP
author: DimancheH
---
{% assign author = site.data.authors[page.author] %}
<div> 作者: {{ author.name }}  
 邮箱：{{ author.email }}
</div>

# 动态加载介绍(2016)

[动态加载PPT](./dynamic loading.pptx)

[动态加载PDF](./dynamic loading.pdf)

使用动态加载技术

- 让用户不用重新安装APK就能升级应用的功能（特别是 SDK项目）
- 大大提高应用新版本的覆盖率
- 减少了服务器对旧版本接口兼容的压力
- 可以快速修复一些线上的BUG。

# 技术背景

案例1：

大部分APP都有一个启动页面，如果到了一些重要的节日，APP的服务器会配置一些与时节相关的图片，APP启动时候再把原有的启动图换成这些新的图片，这样就能提高用户的体验了。

---

案例2：

在安卓市场可能拒绝上架含有广告的应用。那么就通过在服务器配置一个开关，审核应用的时候先把开关关闭，这样应用就不会显示广告了；安卓市场审核通过后，再把服务器的广告开关给打开，以这样的手段规避市场的审核。

---

**道高一尺魔高一丈**。安卓市场开始扫描APK里面的Manifest甚至dex文件，查看开发者的APK包里是否有广告的代码，如果有就有可能审核不通过。

在用户运行APP的时候，再从服务器下载广告的代码，运行，再显示广告呢？

——动态加载：

在程序运行的时候，加载一些程序自身原本不存在的可执行文件并运行这些文件里的代码逻辑。

看起来就像是应用从服务器下载了一些代码，然后再执行这些代码！

# 传统PC软件中的动态加载技术

动态加载技术在PC软件领域广泛使用，如输入法的截图功能。输入法软件可能没有截图功能，第一次使用的时候，输入法会先从服务器下载并安装截图软件，然后再执行截图功能。

许多PC软件的安装目录里面都有大量的DLL文件（Dynamic Link Library）,PC软件通过调用这些DLL里面的代码执行特定的功能，这就是一种动态加载技术。

Java的可执行文件是Jar（Java Archive），运行在虚拟机上JVM上，虚拟机通过ClassLoader加载Jar文件并执行里面的代码。所以Java程序也可以通过动态调用Jar文件达到动态加载的目的。

# Android应用的动态加载技术

应用类似于Java，虚拟机换成了Dalvik/ART，而Jar换成了Dex。

如果下载一个新的APK下来，不安装这个APK的话可不能运行。如果让用户手动安装完这个APK再启动，那可不像是动态加载，纯粹就是用户安装了一个新的应用，然后再启动这个新的应用（这种做法也叫做“静默安装”）。

**动态调用外部的Dex文件则是完全没有问题的。**在APK文件中往往有一个或者多个Dex文件，每一句代码都会被编译到这些文件里面，Android应用运行的时候就是通过执行这些Dex文件完成应用的功能的。虽然一个APK一旦构建出来，我们是无法更换里面的Dex文件的，但是我们**可以通过加载外部的Dex文件来实现动态加载**，这个外部文件可以放在外部存储，或者从网络下载。

所以说不能下载新的APK并安装，需要加载外部Dex

# 动态加载的定义

动态加载是

1. 应用在运行的时候通过加载一些**本地不存在**的可执行文件实现一些特定的功能;
2. 这些可执行文件是**可以替换**的;
3. 更换静态资源（比如换启动图、换主题、或者用服务器参数开关控制广告的隐藏现实等）**不属于** 动态加载;
4. Android中动态加载的**核心思想是动态调用外部的dex文件**，极端的情况下，Android APK自身带有的Dex文件只是一个程序的入口（或者说空壳），所有的功能都通过从服务器下载最新的Dex文件完成;

# Android动态加载的类型

Android项目中，动态加载技术按照加载的可执行文件的不同大致可以分为两种：

1. 动态加载so库；
  
    Android的NDK中，可以动态加载.so库并通过JNI（Java平台提供的编程框架和接口，用于在Java程序中调用和使用本地代码（Native Code））调用其封装好的方法。后者一般是由C/C++编译而成，运行在Native层，效率会比执行在虚拟机层的Java代码高很多。
    
    用于对性能比较有需求的工作（比如T9搜索、Bitmap的解码、图片高斯模糊处理）。
    
    由于so库是由C/C++编译而来的，只能被反编译成汇编代码，相比中dex文件反编译得到的Smali代码更难被破解，因此so库也可以被用于安全领域。
    
    **一般情况把so库一并打包在APK内部，但是so库也可以从外部存储文件加载。**
    
2. 动态加载dex/jar/apk文件（动态加载）；
  
    使用ClassLoader动态加载dex/jar/apk文件，即在Android中动态加载由Java代码编译而来的dex包并执行
    
    Android项目中，所有Java代码都会被编译成dex文件，app运行就是执行dex文件里的代码。动态加载可以在app运行时加载外部的dex文件。通过网络下载新的dex并替换原有的dex就可以达到不安装新APK文件就改变代码的。
    
    使用动态加载技术，一般来说会使得Android开发工作变得更加复杂，开发方式不是官方推荐的，不是目前主流的Android开发方式，目前只有在天朝才有比较深入的研究和应用，如一些SDK组件项目和BAT的项目上，Github上的相关开源项目基本是国人在维护。
    

# 动态加载的大致过程

出于安全问题，Android并不允许直接加载手机外部存储这类noexec（不可执行）存储路径上的可执行文件。

对于外部的可执行文件，调用前都要先拷贝到data/packagename/内部储存文件路径，**确保库不会被第三方应用恶意修改或拦截**，然后再将他们加载到当前运行环境执行。

动态加载流程：

1. 把可执行文件（.so/dex/jar/apk）拷贝到应用APP内部存储；
2. 加载可执行文件；
3. 调用具体的方法执行；

# 动态加载SO

[动态加载so](https://segmentfault.com/a/1190000004062899)

SO库的加载是使用System类的（由此可见对SO库的支持也是Android的基础功能）。

不过，如果使用ClassLoader加载SD卡里插件APK，而插件APK里面包含有SO库，这就涉及到了对插件APK里的SO库的加载，所以我们也要知道如何加载SD卡里面的SO库。

## 一般SO文件的使用

[一般so的使用](https://github.com/kaedea/android-dynamical-loading)

以一个“图片高斯模糊”的功能为例，使用开源的[高斯模糊项目](https://github.com/kikoso/android-stackblur)

1. 定位到Android.mk文件所在目录，运行NDK工具的ndk-build命令编译出SO库
2. 把SO库复制到Android Studio项目的jniLibs目录中

---

Android NDK是一个允许开发者使用C、C++和其他本地编程语言编写Android应用程序的工具集。它提供了一组库和工具，使开发者能够将本地代码集成到Android应用中，并通过JNI与Java代码进行交互。

在使用Android NDK开发项目时，开发者通常会使用以下两个关键元素：

- android.mk：android.mk是一个用于构建本地代码的Makefile文件。它是一个文本文件，定义了编译和构建本地代码所需的各种设置和规则。在android.mk文件中，开发者可以指定需要编译的源文件、依赖项、编译选项、链接选项等。
- ndk-build：ndk-build是一个用于构建Android NDK项目的命令行工具。它读取android.mk文件，并根据其中的规则和设置来编译、链接和构建本地代码。开发者可以通过运行ndk-build命令来触发构建过程，生成可执行文件、共享库或其他本地代码产物。

---

1. 在Java中加载SO库对应的模块

```java
// load so file from internal directory
        try {
            System.loadLibrary("stackblur");
						// 使用System类的loadLibrary()方法加载名为"stackblur"的共享库。
						// loadLibrary()方法是Java提供的用于加载本地共享库的方法。
						// 通过传递共享库的名称作为参数，它会在运行时加载该共享库。
            NativeBlurProcess.isLoadLibraryOk.set(true);
            Log.i("MainActivity", "loadLibrary success!");
        } catch (Throwable throwable) {
            Log.i("MainActivity", "loadLibrary error!" + throwable);
        }
```

1. 加载成功后就可以直接使用Native方法了

```java
public class NativeBlurProcess {
    public static AtomicBoolean isLoadLibraryOk = new AtomicBoolean(false);
    //native method
    private static native void functionToBlur(Bitmap bitmapOut, 
				int radius, int threadCount, int threadIndex, int round);
    }
```

## 如何把SO文件存放在外部存储

System类还有一个load方法

```java
/**
     * See {@link Runtime#load}.
     */
    public static void load(String pathName) {
        Runtime.getRuntime().load(pathName, VMStack.getCallingClassLoader());
    }

    /**
     * See {@link Runtime#loadLibrary}.
     */
    public static void loadLibrary(String libName) {
        Runtime.getRuntime().loadLibrary(libName, 																				VMStack.getCallingClassLoader());
    }
```

### 先看loadLibrary：

loadLibrary使用了ClassLoader：

```java
/*
     * Searches for and loads the given shared library using 
				the given ClassLoader.
     */
    void loadLibrary(String libraryName, ClassLoader loader) {
        if (loader != null) {
            String filename = loader.findLibrary(libraryName);
            String error = doLoad(filename, loader);
            return;
        }
        ……
    }
```

其中loader.findLibrary(libraryName)为：

```java
protected String findLibrary(String libName) {
        return null;
    }
```

ClassLoader只是一个抽象类，它的大部分工作都在子类BaseDexClassLoader类中实现，需要在官方AOSP代码中查看，[BaseDexClassLoader.java](https://android.googlesource.com/platform/libcore-snapshot/+/ics-mr1/dalvik/src/main/java/dalvik/system/BaseDexClassLoader.java)

```java
@Override
    public String findLibrary(String name) {
        return pathList.findLibrary(name);
				// 这里pathList用到了构造函数中传入的new DexPathList(...)
    }
```

里面用到了DexPathList类：

```java
/**
     * Finds the named native code library on any of the library
     * directories pointed at by this instance. This will find the
     * one in the earliest listed directory, ignoring any that are not
     * readable regular files.
     *
     * @return the complete path to the library or {@code null} if no
     * library was found
     */
    public String findLibrary(String libraryName) {
				// 参数转换为与平台相关的库文件名。
				// 例如，对于库名 "mylibrary"，如果运行在不同的平台上，
				// 可能会得到类似 "libmylibrary.so" 或 "mylibrary.dll" 的文件名。
        String fileName = System.mapLibraryName(libraryName);
        for (File directory : nativeLibraryDirectories) {
            File file = new File(directory, fileName);
						// 找到满足条件的库文件
            if (file.exists() && file.isFile() && file.canRead()) {
                return file.getPath();
            }
        }
        return null;
    }
```

根据传进来的libraryName，扫描APK内部的nativeLibrary目录，获取并返回内部SO库文件的完整路径filename。

回溯到Runtime类，获取filename后调用了“doLoad”方法：

```java
private String doLoad(String name, ClassLoader loader) {
        String ldLibraryPath = null;
        String dexPath = null;
        // loader 如果为 null，则将 ldLibraryPath 设置为系统属性
				// "java.library.path" 的值。表示本地库文件的搜索路径。
				if (loader == null) {
            ldLibraryPath = System.getProperty("java.library.path");
        } else if (loader instanceof BaseDexClassLoader) {
				// loader 如果是 BaseDexClassLoader 
				// 调用 getLdLibraryPath() 方法获取 ldLibraryPath 的值。
            BaseDexClassLoader dexClassLoader = (BaseDexClassLoader) loader;
            ldLibraryPath = dexClassLoader.getLdLibraryPath();
        }
				// 确保多个线程之间的安全访问
        synchronized (this) {
            return nativeLoad(name, loader, ldLibraryPath);
        }
    }
```

即LoadLibrary方法是通过找到完整的SO库路径filename，把目标SO库加载进来。

### 再看Load方法

如果使用loadLibrary方法，最后找到目标SO库的完整路径加载SO库。

能不能一开始就给出SO库的完整路径直接加载？

猜想load方法就是干这个的。

```java
void load(String absolutePath, ClassLoader loader) {
        if (absolutePath == null) {
            throw new NullPointerException("absolutePath == null");
        }
        String error = doLoad(absolutePath, loader);
        if (error != null) {
            throw new UnsatisfiedLinkError(error);
        }
    }
```

### 实验测试

貌似是对的？

实验证明：先把SO放在Asset里，然后再复制到内部存储，再使用load方法加载。

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        File dir = this.getDir("jniLibs", Activity.MODE_PRIVATE);
        File distFile = new File(dir.getAbsolutePath() + File.separator 
																						+ "libstackblur.so");

        if (copyFileFromAssets(this, "libstackblur.so", 
																				distFile.getAbsolutePath())){
            //使用load方法加载内部储存的SO库
            System.load(distFile.getAbsolutePath());
            NativeBlurProcess.isLoadLibraryOk.set(true);
        }
    }

    public void onDoBlur(View view){
        ImageView imageView = (ImageView) findViewById(R.id.iv_app);
        Bitmap bitmap = BitmapFactory.decodeResource(getResources(), 
																android.R.drawable.sym_def_app_icon);
        Bitmap blur = NativeBlurProcess.blur(bitmap,20,false);
        imageView.setImageBitmap(blur);
    }

    public static boolean copyFileFromAssets(Context context, 
		String fileName, String path) {
        boolean copyIsFinish = false;
        try {
            InputStream is = context.getAssets().open(fileName);
            File file = new File(path);
            file.createNewFile();
            FileOutputStream fos = new FileOutputStream(file);
            byte[] temp = new byte[1024];
            int i = 0;
            while ((i = is.read(temp)) > 0) {
                fos.write(temp, 0, i);
            }
            fos.close();
            is.close();
            copyIsFinish = true;
        } catch (IOException e) {
            e.printStackTrace();
            Log.e("MainActivity", 
									"[copyFileFromAssets] IOException "+e.toString());
        }
        return copyIsFinish;
    }
}
```

点击onDoBlur按钮，加载成功！

那能不能直接加载外部存储上面的SO库呢，把SO库拷贝到SD卡上面试试。

> java.lang.UnsatisfiedLinkError: dlopen failed: couldn't map "/storage/emulated/0/libstackblur.so" segment 1: Permission denied
> 

Google开发者论坛：

        SD卡等外部存储路径是一种可拆卸的（mounted）不可执行（noexec）的储存媒介，不能直接用来作为可执行文件的运行目录，使用前应该把可执行文件复制到APP内部存储再运行。

# 动态加载dex/jar/apk

[动态加载dex/jar/apk](https://segmentfault.com/a/1190000004062880)

## 基础知识：类加载器ClassLoader和dex文件

动态加载的基础是类加载器ClassLoader，包路径是java.lang。虚拟机通过类加载器加载其需要用的Class。

### 类加载器ClassLoader

Eclipse可以加载许多第三方的插件或扩展，即动态加载。这些插件大多是一些Jar包，而使用插件就是动态加载Jar包里的Class。Java代码都是写在Class里面的，虚拟机需要把需要的Class加载进来才能创建实例对象并工作，即使用ClassLoader。

对于Java，编写程序就是编写类，运行程序也就是运行类（编译得到的class文件），其中起到关键作用的就是类加载器ClassLoader。

### 有几个ClassLoader实例？

一个运行中的APP **不仅只有一个类加载器。**

[项目参考](https://github.com/kaedea/android-dynamical-loading)

- Boot类型的ClassLoader实例
  
    在Android系统启动的时候会创建一个Boot类型的ClassLoader实例，用于加载一些系统Framework层级需要的类，app里也需要用到一些系统的类，所以APP启动的时候也会把这个Boot类型的ClassLoader传进来。
    
    > 以下是关于 BootClassLoader 的一些重要信息：
    > 
    > - 根类加载器：BootClassLoader 是 Android 系统的根类加载器，它是类加载器层次结构中的最顶层。它没有父类加载器，它自己是顶级加载器。
    > - 加载核心类和资源：BootClassLoader 负责加载核心系统类和资源，包括 Android Framework 的核心类和系统级库。这些类和库对于整个系统的运行非常重要，因此它们需要在系统启动时被加载和初始化。
    > - 引导类路径：BootClassLoader 使用一组预定义的引导类路径，这些路径指定了系统类和库的位置。这些路径通常包括 Android 系统的内置库、系统框架库和其他系统级资源。
    > - 不可更改性：BootClassLoader 是系统级加载器，其实例在系统启动时创建，并在整个系统的生命周期中保持不变。它的实现是 Android 系统的一部分，开发者无法直接创建或修改 BootClassLoader 的实例。
- APP自己的ClassLoader实例
  
    APP也有自己的类，这些类保存在APK的dex文件里面，所以APP启动的时候，也会创建一个自己的ClassLoader实例，用于加载自己dex文件中的类。当应用启动时，Android 系统会创建一个专门用于加载应用程序的类的类加载器实例，称为应用程序类加载器（Application Class Loader）。下面我们在项目里验证看看
    
    ```java
    @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            ClassLoader classLoader = getClassLoader();
            if (classLoader != null){
                Log.i(TAG, "[onCreate] classLoader " + i + " : " + classLoader.toString());
                while (classLoader.getParent()!=null){
                    classLoader = classLoader.getParent();
                    Log.i(TAG,"[onCreate] classLoader " + i + " : "
    										 + classLoader.toString());
                }
            }
        }
    ```
    
    输出为：
    
    ```java
    [onCreate] classLoader 1 : dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/me.kaede.anroidclassloadersample-1/base.apk"],nativeLibraryDirectories=[/vendor/lib, /system/lib]]]
    
    [onCreate] classLoader 2 : java.lang.BootClassLoader@14af4e32
    ```
    
    有2个Classloader实例，一个是BootClassLoader（系统启动的时候创建的），另一个是PathClassLoader（应用启动时创建的，用于加载“/data/app/me.kaede.anroidclassloadersample-1/base.apk”中的类）。
    
    **一个运行的Android应用至少有2个ClassLoader**。
    

### 创建自己ClassLoader实例

ClassLoader的构造函数：

```java
/*
     * constructor for the BootClassLoader which needs parent to be null.
     */
    ClassLoader(ClassLoader parentLoader, boolean nullAllowed) {
        if (parentLoader == null && !nullAllowed) {
            throw new NullPointerException("parentLoader == null && !nullAllowed");
        }
        parent = parentLoader;
    }
```

创建一个ClassLoader实例需要使用现有的ClassLoader实例作为Parent。因此，一个APP，甚至整个Android系统里所有的ClassLoader实例都会被一棵树关联起来，这也是ClassLoader的 **双亲代理模型**（Parent-Delegation Model）的特点。

> 如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此。
> 
> 
> 因此所有的加载请求最终都应该传送到顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（**它的搜索范围中没有找到所需的类**）时，子加载器才会尝试自己去加载。
> 

### ClassLoader双亲代理模型加载类的特点和作用

**特点**

loadClass方法在加载一个类的实例时：

1. 会先查询当前ClassLoader实例是否加载过此类，有就返回；
2. 如果没有。查询Parent是否已经加载过此类，如果已经加载过，就直接返回Parent加载的类；
3. 如果继承路线上的ClassLoader都没有加载，才由Child执行类的加载工作；

这样做有个明显的特点，如果一个类被位于树根的ClassLoader加载过，那么在以后整个系统的生命周期内，这个类永远不会被重新加载。

**作用**

- 共享功能，Framework层级的类一旦被顶层的ClassLoader加载过就缓存在内存中，不需要重新加载。
- 隔离功能，不同继承路线上的ClassLoader加载的类肯定不是同一个类。避免了用户自己的代码冒充核心类库的类访问核心类库包可见成员的情况。

### 使用ClassLoader一些需要注意的问题

- 新类替换旧类
  
    如果通过动态加载的方式，加载一个新版本的dex文件，使用里面的新类替换原有的旧类，必须保证在加载新类的时候，旧类还没有被加载，因为如果已经加载过旧类，那么ClassLoader会一直优先使用旧类。
    
- 新类在旧类后加载
  
    如果旧类总是优先于新类被加载，可以使用一个与加载旧类的ClassLoader**没有树的继承关系**的另一个ClassLoader来加载新类，因为ClassLoader只会检查其Parent有没有加载过当前要加载的类，**如果两个ClassLoader没有继承关系，那么旧类和新类都能被加载。**
    
- 类型匹配问题
  
    在Java中，只有当两个实例的**类名**、**包名**以及**加载其的ClassLoader**都相同，才会被认为是**同一种类型**。上面分别加载的新类和旧类，虽然包名和类名都完全一样，但是由于加载的ClassLoader不同，所以并不是同一种类型，在实际使用中可能会出现类型不符异常。
    
    同一个Class = 相同的 ClassName + PackageName + ClassLoader
    

### DexClassLoader 和 PathClassLoader

ClassLoader是一个抽象类，有两个子类：

- DexClassLoader可以加载jar/apk/dex，可以从SD卡中加载未安装的apk；
- PathClassLoader只能加载系统中已经安装过的apk；

### 类加载器的初始化

```java
// DexClassLoader.java
public class DexClassLoader extends BaseDexClassLoader {
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), libraryPath, parent);
    }
}

// PathClassLoader.java
public class PathClassLoader extends BaseDexClassLoader {
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }

    public PathClassLoader(String dexPath, String libraryPath,
            ClassLoader parent) {
        super(dexPath, null, libraryPath, parent);
    }
}
```

是对基类BaseDexClassLoader做了一下封装，具体实现在父类里。

PathClassLoader的optimizedDirectory只能是null，看看基类BaseDexClassLoader：

```java
public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(parent);
        this.originalPath = dexPath;
        this.pathList = new DexPathList(this, dexPath, libraryPath, 																				optimizedDirectory);
    }
```

创建了一个DexPathList实例，看看DexPathList：

```java
public DexPathList(ClassLoader definingContext, String dexPath,
            String libraryPath, File optimizedDirectory) {
        ……
        this.dexElements = makeDexElements(splitDexPath(dexPath), 
																							optimizedDirectory);
    }

    private static Element[] makeDexElements(ArrayList<File> files,
            File optimizedDirectory) {
        ArrayList<Element> elements = new ArrayList<Element>();
        for (File file : files) {
            ZipFile zip = null;
            DexFile dex = null;
            String name = file.getName();
            if (name.endsWith(DEX_SUFFIX)) {
                dex = loadDexFile(file, optimizedDirectory);
            } else if (name.endsWith(APK_SUFFIX) || name.endsWith(JAR_SUFFIX)
                    || name.endsWith(ZIP_SUFFIX)) {
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
        if (optimizedDirectory == null) {
            return new DexFile(file);
        } else {
            String optimizedPath = optimizedPathFor(file, 
																								optimizedDirectory);
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

dexPath所在路径的dex文件会被加载到optimizedDirectory中，并创建一个DexFile对象，如果它为null，那么会直接使用dex文件原有的路径来创建DexFile对象。

对于 DexClassLoader：

- optimizedDirectory 参数必须是一个内部存储路径
  
    因为它用于存储经过优化处理的 DEX 文件。
    
- DexClassLoader 允许加载外部的 DEX 文件
  
    因为它会将外部的 DEX 文件复制到内部存储路径的 optimizedDirectory 目录中，并在那里进行优化处理。这样做是为了提高加载和执行外部 DEX 文件的性能。
    

对于 PathClassLoader：

- 没有 optimizedDirectory 参数。
- 主要用于加载已安装在系统中的 APK 文件中的 DEX 文件
  
    这些 DEX 文件通常位于内部存储中。PathClassLoader 不会进行额外的复制和优化操作，而是直接加载 APK 文件中的 DEX 文件。
    

### 加载类的过程

创建类加载器的实例的时候创建了一个DexFile实例，用来保存dex文件，猜想这个实例就是用来加载类的。

ClassLoader用loadClass方法来加载需要的类：

```java
public Class<?> loadClass(String className) throws ClassNotFoundException {
        return loadClass(className, false);
    }

    protected Class<?> loadClass(String className, boolean resolve) throws 
																									ClassNotFoundException {
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

loadClass方法调用了findClass方法，而BaseDexClassLoader重载了这个方法：

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

调用了DexPathList的findClass：

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

遍历了之前所有的DexFile实例，即遍历了所有加载过的dex文件，再调用loadClassBinaryName方法一个个尝试能不能加载想要的类

```java
public Class loadClassBinaryName(String name, ClassLoader loader) {
        return defineClass(name, loader, mCookie);
    }
private native static Class defineClass(String name, ClassLoader loader, 																				int cookie);
```

loadClassBinaryName中调用了Native方法defineClass加载类。

> Native方法是在Java代码中声明但在底层实现中由本地语言（如C或C++）编写的方法。它们允许Java代码与底层系统和硬件交互。
> 
> 
> 当在Java中声明一个方法为native时，意味着该方法的具体实现并不在Java虚拟机（JVM）的运行时环境中，而是在本地代码中。在运行时，当Java代码调用该native方法时，JVM会将控制权转移到底层本地代码中去执行对应的功能。
> 
> Native方法通常用于以下情况：
> 
> - 访问底层系统功能：例如，操作系统API的调用、硬件设备的交互等。
> - 与本地库进行交互：Java可以通过本地方法调用本地库（如C/C++编写的动态链接库），以便利用本地库提供的功能。
> 
> 在Java中声明native方法时，方法体通常为空。Java代码只负责定义方法的签名（名称、参数和返回类型），而方法的具体实现由本地代码提供。
> 

> 加载类需要使用 native 方法主要是因为类加载涉及到底层的虚拟机实现和操作系统交互，这些操作无法完全通过Java代码来实现。
> 
> 
> 以下是加载类需要使用 native 方法的几个主要原因：
> 
> 1. 虚拟机内部实现：类加载是虚拟机的核心功能之一，它涉及到虚拟机内部的数据结构和处理逻辑。为了提高性能和灵活性，类加载通常会直接由虚拟机的本地实现来完成，因为本地代码可以更接近底层操作系统和硬件。
> 2. 本地库调用：在类加载过程中，可能需要调用底层的本地库来完成一些特定的操作，例如操作系统的API调用、硬件设备的交互等。这些本地库通常是用C或C++等语言编写的，通过本地方法可以在Java代码中调用这些本地库的功能。
> 3. 安全和权限控制：类加载是一个敏感的操作，涉及到加载和执行外部代码。通过使用 native 方法，可以限制对底层系统资源的直接访问，并提供更好的安全和权限控制机制。底层本地代码可以进行更严格的验证和权限检查，以确保加载的类的安全性。
> 4. 平台相关性：类加载的实现可能会因操作系统、硬件架构或虚拟机的不同而有所差异。通过使用 native 方法，可以针对不同的平台提供特定的实现，以确保类加载的正确性和性能。

### 自定义ClassLoader

动态加载开发的时候，使用DexClassLoader就够了。

也可以创建自己的类去继承ClassLoader，loadClass方法并不是final类型的，所以我们可以重载loadClass方法并改写类的加载逻辑。

ClassLoader双亲代理的实现很大一部分就是在loadClass方法里，我们可以通过重写loadClass方法避开双亲代理的框架，这样一来就可以在**重新加载已经加载过的类**，也可以在**加载类的时候注入一些代码**。

### Android程序比起一般Java程序在使用动态加载时麻烦在哪里

比起一般的Java程序，在Android程序中使用动态加载主要有两个麻烦的问题：

1. Android中许多组件类（如Activity、Service等）是需要在Manifest文件里面注册后才能工作（系统会检查该组件有没有注册），所以即使动态加载了一个新的组件类，没有注册的话还是无法工作；
2. Resource资源是Android开发中经常用到的，而Android是把这些资源用对应的R.id注册好，运行时通过这些ID从Resource实例中获取对应的资源。如果是运行时动态加载进来的新类，那类里面用到R.id的地方将会抛出找不到资源或者用错资源的异常，因为新类的资源ID根本和现有的Resource实例中保存的资源ID对不上；

---

**动态加载一般分为三种模式：**

1. **简单的动态加载模式**
2. **代理Activity模式**
3. **动态创建Activity模式**

---

## 简单的动态加载模式

[简单的动态加载](https://segmentfault.com/a/1190000004062952)

Android却很难使用插件APK里的res资源，这意味着无法使用新的XML布局等资源，同时由于无法更改本地的Manifest清单文件，所以无法启动新的Activity等组件。

可以先把要用到的全部res资源都放到主APK里面，同时把所有需要的Activity先全部写进Manifest里，**只通过动态加载更新代码，不更新res资源**，如果需要改动UI界面，可以通过使用纯Java代码创建布局的方式绕开XML布局。同时也可以使用Fragment代替Activity，这样可以最大限度得避开“无法注册新组件的限制”。

1. 无法使用res目录下的资源，特别是使用XML布局，以及无法通过res资源到达自适应
2. 无法动态加载新的Activity等组件，因为这些组件需要在Manifest中注册，动态加载无法更改当前APK的Manifest

以上问题可以通过**反射调用Framework层代码**以及**代理Activity**的方式解决，可以把这种的动态加载框架成为“代理模式”。

## 代理Activity模式

[代理Activity](https://segmentfault.com/a/1190000004062972)

宿主APK需要先注册一个空壳的Activity用于代理执行插件APK的Activity的生命周期。

有以下特点：

1. 宿主APK可以启动未安装的插件APK；
2. 插件APK也可以作为一个普通APK安装并且启动；
3. 插件APK可以调用宿主APK里的一些功能；
4. 宿主APK和插件APK都要接入一套指定的接口框架才能实现以上功能；

限制如下：

1. 需要在Manifest注册的功能都无法在插件实现，如应用权限、LaunchMode、静态广播等；
2. 宿主一个代理用的Activity难以满足插件一些特殊的Activity的需求，插件Activity的开发受限于代理Activity；
3. 宿主项目和插件项目的开发都要接入共同的框架，大多时候，插件需要依附宿主才能运行，无法独立运行；

核心在于“**使用宿主的一个代理Activity为插件所有的Activity提供组件工作需要的环境**”，

## 动态创建Activity模式

[动态创建Activity](https://segmentfault.com/a/1190000004077469)

核心是“运行时字节码操作”。宿主注册一个不存在的Activity，启动插件的某个Activity时都把想要启动的Activity替换成前面注册的Activity，从而使插件的Activity能正常启动。

特点：

1. 主APK可以启动一个未安装的插件APK；
2. 插件APK可以是任意第三方APK，无需接入指定的接口，理所当然也可以独立运行；

### 代理Activity模式的限制

由于插件里的Activity没在主项目的Manifest里面注册，所以无法经历系统Framework层级的一系列初始化过程，最终导致获得的Activity实例并没有生命周期和无法使用res资源。

使用代理Activity能够解决这两个问题，但是有一些限制

1. 实际运行的Activity实例其实都是ProxyActivity，并不是真正想要启动的Activity；
2. ProxyActivity只能指定一种LaunchMode，所以插件里的Activity无法自定义LaunchMode；
3. 不支持静态注册的BroadcastReceiver；
4. 往往不是所有的apk都可作为插件被加载，插件项目需要依赖特定的框架，还有需要遵循一定的"开发规范"；

如何解决？

使其成为标准的Activity——需要在主项目里注册这些Activity

**总不能把插件APK所有的Activity都事先注册到主项目里面吧（还真行？）**

在主项目里**注册一个通用的Activity**（比如TargetActivity）给插件里所有的Activity用呢？在需要启动插件的某一个Activity（比如PlugActivity）时，创建一个TargetActivity，新创建的TargetActivity会继承PlugActivity的所有共有行为，而这个TargetActivity的包名与类名与事先注册的TargetActivity一致，我们就能以标准的方式启动这个Activity。

### 动态创建类的工具

[dexmaker](https://github.com/linkedin/dexmaker)和[asmdex](https://asm.ow2.io/)

前者创建dex文件，而后者是创建class文件。

### 使用dexmaker动态创建一个类

运行时创建一个编译好并能运行的类叫做“动态字节码操作”，使用dexmaker工具能创建一个dex文件，之后我们再反编译这个dex看看创建出来的类是什么样子。

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    public void onMakeDex(View view){
        try {
            DexMaker dexMaker = new DexMaker();
            // Generate a HelloWorld class.
            TypeId<?> helloWorld = TypeId.get("LHelloWorld;");
            dexMaker.declare(helloWorld, "HelloWorld.generated", Modifier.PUBLIC, TypeId.OBJECT);
            generateHelloMethod(dexMaker, helloWorld);
            // Create the dex file and load it.
            File outputDir = new File(Environment.getExternalStorageDirectory() + File.separator + "dexmaker");
            if (!outputDir.exists())outputDir.mkdir();
            ClassLoader loader = dexMaker.generateAndLoad(this.getClassLoader(), outputDir);
            Class<?> helloWorldClass = loader.loadClass("HelloWorld");
            // Execute our newly-generated code in-process.
            helloWorldClass.getMethod("hello").invoke(null);
        } catch (Exception e) {
            Log.e("MainActivity","[onMakeDex]",e);
        }
    }

    /**
     * Generates Dalvik bytecode equivalent to the following method.
     *    public static void hello() {
     *        int a = 0xabcd;
     *        int b = 0xaaaa;
     *        int c = a - b;
     *        String s = Integer.toHexString(c);
     *        System.out.println(s);
     *        return;
     *    }
     */
    private static void generateHelloMethod(DexMaker dexMaker, TypeId<?> declaringType) {
        // Lookup some types we'll need along the way.
        TypeId<System> systemType = TypeId.get(System.class);
        TypeId<PrintStream> printStreamType = TypeId.get(PrintStream.class);

        // Identify the 'hello()' method on declaringType.
        MethodId hello = declaringType.getMethod(TypeId.VOID, "hello");

        // Declare that method on the dexMaker. Use the returned Code instance
        // as a builder that we can append instructions to.
        Code code = dexMaker.declare(hello, Modifier.STATIC | Modifier.PUBLIC);

        // Declare all the locals we'll need up front. The API requires this.
        Local<Integer> a = code.newLocal(TypeId.INT);
        Local<Integer> b = code.newLocal(TypeId.INT);
        Local<Integer> c = code.newLocal(TypeId.INT);
        Local<String> s = code.newLocal(TypeId.STRING);
        Local<PrintStream> localSystemOut = code.newLocal(printStreamType);

        // int a = 0xabcd;
        code.loadConstant(a, 0xabcd);

        // int b = 0xaaaa;
        code.loadConstant(b, 0xaaaa);

        // int c = a - b;
        code.op(BinaryOp.SUBTRACT, c, a, b);

        // String s = Integer.toHexString(c);
        MethodId<Integer, String> toHexString
                = TypeId.get(Integer.class).getMethod(TypeId.STRING, "toHexString", TypeId.INT);
        code.invokeStatic(toHexString, s, c);

        // System.out.println(s);
        FieldId<System, PrintStream> systemOutField = systemType.getField(printStreamType, "out");
        code.sget(systemOutField, localSystemOut);
        MethodId<PrintStream, Void> printlnMethod = printStreamType.getMethod(
                TypeId.VOID, "println", TypeId.STRING);
        code.invokeVirtual(printlnMethod, null, localSystemOut, s);

        // return;
        code.returnVoid();
    }
    
}
```

运行后在SD卡的dexmaker目录下找到刚创建的文件“Generated1532509318.jar”，把里面的“classes.dex”解压出来，然后再用“dex2jar”工具转化成jar文件，最后再用“jd-gui”工具反编译jar的源码。

成功在运行时创建一个编译好的类。

### 修改需要启动的目标Activity

如何把需要启动的、在Manifest中没有注册的PlugActivity换成有注册的TargetActivity？

加载类时通过ClassLoader的loadClass方法，而loadClass方法并不是final类型的，所以可以创建自己的类去继承ClassLoader，以重载loadClass方法并改写类的加载逻辑，在需要加载PlugActivity的时候，偷偷把其换成TargetActivity。

```java
public class CJClassLoader extends ClassLoader{

@override
    public Class loadClass(String className){
      if(当前上下文插件不为空) {
        if( className 是 TargetActivity){
             找到当前实际要加载的原始PlugActivity，动态创建类（TargetActivity extends PlugActivity ）的dex文件
             return  从dex文件中加载的TargetActivity
        }else{
             return  使用对应的PluginClassLoader加载普通类
        }  
     }else{
         return super.loadClass() //使用原来的类加载方法
     }   
    } 
}
```

通过自定义类加载器加载插件类（例如 `PlugActivity`），然后动态创建继承自插件类的新类（例如 `TargetActivity`）的 dex 文件。然后使用原有的加载方式（例如使用系统的类加载器）加载这个新类的 dex 文件，从而实现加载插件类但生成的是目标类的效果。

主项目启动插件Activity的时候，我们可以替换Activity，但是如果在插件Activity启动另一个Activity的时候怎么办？插件时普通的第三方APK，我们无法更改里面跳转Activity的逻辑。

其实，从主项目启动插件MainActivity的时候，其实启动的是我们动态创建的TargetActivity（extends MainActivity），而我们知道Activity启动另一个Activity的时候都是使用其“startActivityForResult”方法，所以我们可以在创建TargetActivity时，重写其“startActivityForResult”方法，**让它在启动其他Activity的时候，也采用动态创建Activity的方式**，这样就能解决问题。

### **动态创建Activity开源项目 [android-pluginmgr](https://github.com/houkx/android-pluginmgr/)**

`android-pluginmgr`项目中有三种ClassLoader，

- 用于替换宿主APK的Application的`CJClassLoader`
- 用于加载插件APK的`PluginClassLoader`
- 用于加载启动插件Activity时动态生成的PlugActivity的dex包的`DexClassLoader`（存放在Map集合`proxyActivityLoaderMap`里面）。

其中`CJClassLoader`是`PluginClassLoader`的Parent，而`PluginClassLoader`是`DexClassLoader`的Parent。

### 存在的问题

动态类创建的方式，使得注册一个通用的Activity就能给多给Activity使用，对这种做法存在的问题也是明显的

1. 使用同一个注册的Activity，所以一些需要在Manifest注册的属性无法做到每个Activity都自定义配置；**（只做应用解耦可以全部自定义？）**
2. 无法动态添加权限；
  
    插件中的权限，无法动态注册，插件需要的权限都得在宿主中注册；
    
3. 插件的Activity无法开启独立进程，因为需要在Manifest里面注册；
4. 动态字节码操作涉及到Hack开发，所以相比代理模式起来不稳定；

其中不稳定的问题出现在对Service的支持上，使用动态创建类的方式可以搞定Activity和Broadcast Receiver，但是使用类似的方式处理Service却不行，因为“ContextImpl.getApplicationContext” 期待得到一个非ContextWrapper的context，如果不是则继续下次循环，目前的Context实例都是wrapper，所以会进入死循环。