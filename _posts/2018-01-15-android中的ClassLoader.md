---
layout:     post
title:      android中的ClassLoader
subtitle:   android中的ClassLoader学习
date:       2018-01-15
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - android
    - framework
    - java
    - classloader
---

  <h3><a id="_9"></a><strong>前言</strong></h3> 
  <p>在上一篇文章我们学习了Java的ClassLoader，很多同学会把Java和Android的ClassLoader搞混，甚至会认为Android中的ClassLoader和Java中的ClassLoader是一样的，这显然是不对的。这一篇文章我们就来学习Android中的ClassLoader，来看看它和Java中的ClassLoader有何不同。</p> 
  <h3><a id="1ClassLoader_13"></a><strong>1.ClassLoader的类型</strong></h3> 
  <p>我们知道Java中的ClassLoader可以加载jar文件和Class文件（本质是加载Class文件），这一点在Android中并不适用，因为无论是DVM还是ART它们加载的不再是Class文件，而是dex文件，这就需要重新设计ClassLoader相关类，我们先来学习ClassLoader的类型。<br /> Android中的ClassLoader类型和Java中的ClassLoader类型类似，也分为两种类型，分别是系统ClassLoader和自定义ClassLoader。其中系统ClassLoader主要有3种分别是BootClassLoader、PathClassLoader和DexClassLoader。</p> 
  <h4><a id="11_BootClassLoader_16"></a><strong>1.1 BootClassLoader</strong></h4> 
  <p>Android系统启动时会使用BootClassLoader来预加载常用类，与Java中的BootClassLoader不同，它并不是由C/C++代码实现，而是由Java实现的，BootClassLoade的代码如下所示。<br /> <strong>libcore/ojluni/src/main/java/java/lang/ClassLoader.java</strong></p> 
  <pre><code class="prism language-java"><span class="token keyword">class</span> <span class="token class-name">BootClassLoader</span> <span class="token keyword">extends</span> <span class="token class-name">ClassLoader</span> <span class="token punctuation">{
     <!-- --></span>
    <span class="token keyword">private</span> <span class="token keyword">static</span> BootClassLoader instance<span class="token punctuation">;</span>
    <span class="token annotation punctuation">@FindBugsSuppressWarnings</span><span class="token punctuation">(</span><span class="token string">&quot;DP_CREATE_CLASSLOADER_INSIDE_DO_PRIVILEGED&quot;</span><span class="token punctuation">)</span>
    <span class="token keyword">public</span> <span class="token keyword">static</span> <span class="token keyword">synchronized</span> BootClassLoader <span class="token function">getInstance</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span>
        <span class="token keyword">if</span> <span class="token punctuation">(</span>instance <span class="token operator">==</span> null<span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span>
            instance <span class="token operator">=</span> <span class="token keyword">new</span> <span class="token class-name">BootClassLoader</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span>
        <span class="token keyword">return</span> instance<span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
<span class="token punctuation">.</span><span class="token punctuation">.</span><span class="token punctuation">.</span>
<span class="token punctuation">}</span>
</code></pre> 
  <p>BootClassLoader是ClassLoader的内部类，并继承自ClassLoader。BootClassLoader是一个单例类，需要注意的是BootClassLoader的访问修饰符是默认的，只有在同一个包中才可以访问，因此我们在应用程序中是无法直接调用的。</p> 
  <h4><a id="12_DexClassLoader_34"></a><strong>1.2 DexClassLoader</strong></h4> 
  <p>DexClassLoader可以加载dex文件以及包含dex的压缩文件（apk和jar文件），不管是加载哪种文件，最终都是要加载dex文件，为了方便理解和叙述，将dex文件以及包含dex的压缩文件统称为dex相关文件。<br /> 来查看DexClassLoader的代码，如下所示。<br /> <strong>libcore/dalvik/src/main/java/dalvik/system/DexClassLoader.java</strong></p> 
  <pre><code>public class DexClassLoader extends BaseDexClassLoader {
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), librarySearchPath, parent);
	    }
}
</code></pre> 
  <p>DexClassLoader的构造方法有四个参数：</p> 
  <ul>
   <li>dexPath：dex相关文件路径集合，多个路径用文件分隔符分隔，默认文件分隔符为‘：’</li>
   <li>optimizedDirectory：解压的dex文件存储路径，这个路径必须是一个内部存储路径，一般情况下使用当前应用程序的私有路径：<code>/data/data/&lt;Package Name&gt;/...</code>。</li>
   <li>librarySearchPath：包含 C/C++ 库的路径集合，多个路径用文件分隔符分隔分割，可以为null。</li>
   <li>parent：父加载器。</li>
  </ul> 
  <p>DexClassLoader 继承自BaseDexClassLoader ，方法实现都在BaseDexClassLoader中。</p> 
  <h4><a id="13_PathClassLoader_55"></a><strong>1.3 PathClassLoader</strong></h4> 
  <p>Android系统使用PathClassLoader来加载系统类和应用程序的类，来查看它的代码：<br /> <strong>libcore/dalvik/src/main/java/dalvik/system/PathClassLoader.java</strong></p> 
  <pre><code>public class PathClassLoader extends BaseDexClassLoader {
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }
    public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent) {
        super(dexPath, null, librarySearchPath, parent);
    }
}
</code></pre> 
  <p>PathClassLoader继承自BaseDexClassLoader，实现也都在BaseDexClassLoader中。</p> 
  <p>PathClassLoader的构造方法中没有参数optimizedDirectory，这是因为PathClassLoader已经默认了参数optimizedDirectory的值为：/data/dalvik-cache，很显然PathClassLoader无法定义解压的dex文件存储路径，因此PathClassLoader通常用来加载已经安装的apk的dex文件(安装的apk的dex文件会存储在/data/dalvik-cache中)。</p> 
  <h3><a id="2ClassLoader_74"></a><strong>2.ClassLoader的继承关系</strong></h3> 
  <p>运行一个Android程序需要用到几种类型的类加载器呢？如下所示。</p> 
  <pre><code class="prism language-java"><span class="token keyword">public</span> <span class="token keyword">class</span> <span class="token class-name">MainActivity</span> <span class="token keyword">extends</span> <span class="token class-name">AppCompatActivity</span> <span class="token punctuation">{
     <!-- --></span>
    <span class="token annotation punctuation">@Override</span>
    <span class="token keyword">protected</span> <span class="token keyword">void</span> <span class="token function">onCreate</span><span class="token punctuation">(</span>Bundle savedInstanceState<span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span>
        <span class="token keyword">super</span><span class="token punctuation">.</span><span class="token function">onCreate</span><span class="token punctuation">(</span>savedInstanceState<span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token function">setContentView</span><span class="token punctuation">(</span>R<span class="token punctuation">.</span>layout<span class="token punctuation">.</span>activity_main<span class="token punctuation">)</span><span class="token punctuation">;</span>
        ClassLoader loader <span class="token operator">=</span> MainActivity<span class="token punctuation">.</span><span class="token keyword">class</span><span class="token punctuation">.</span><span class="token function">getClassLoader</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token keyword">while</span> <span class="token punctuation">(</span>loader <span class="token operator">!=</span> null<span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span>
            Log<span class="token punctuation">.</span><span class="token function">d</span><span class="token punctuation">(</span><span class="token string">&quot;zhh&quot;</span><span class="token punctuation">,</span>loader<span class="token punctuation">.</span><span class="token function">toString</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span><span class="token comment">//1</span>
            loader <span class="token operator">=</span> loader<span class="token punctuation">.</span><span class="token function">getParent</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span>
    <span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre> 
  <p>首先我们得到MainActivity的类加载器，并在注释1处通过Log打印出来，接着打印出当前类的类加载器的父加载器，直到没有父加载器终止循环。打印结果如下所示。</p> 
  <p>10-07 07:23:02.835 8272-8272/? D/zhh: dalvik.system.PathClassLoader[DexPathList[[zip file “/data/app/com.example.zhh.moonclassloader-2/base.apk”, zip file “/data/app/com.example.zhh.moonclassloader-2/split_lib_dependencies_apk.apk”, zip file “/data/app/com.example.zhh.moonclassloader-2/split_lib_slice_0_apk.apk”, zip file “/data/app/com.example.zhh.moonclassloader-2/split_lib_slice_1_apk.apk”, zip file “/data/app/com.example.zhh.moonclassloader-2/split_lib_slice_2_apk.apk”, zip file “/data/app/com.example.zhh.moonclassloader-2/split_lib_slice_3_apk.apk”, zip file “/data/app/com.example.zhh.moonclassloader-2/split_lib_slice_4_apk.apk”, zip file “/data/app/com.example.zhh.moonclassloader-2/split_lib_slice_5_apk.apk”, zip file “/data/app/com.example.zhh.moonclassloader-2/split_lib_slice_6_apk.apk”, zip file “/data/app/com.example.zhh.moonclassloader-2/split_lib_slice_7_apk.apk”, zip file “/data/app/com.example.zhh.moonclassloader-2/split_lib_slice_8_apk.apk”, zip file “/data/app/com.example.zhh.moonclassloader-2/split_lib_slice_9_apk.apk”],nativeLibraryDirectories=[/data/app/com.example.zhh.moonclassloader-2/lib/x86, /vendor/lib, /system/lib]]]<br /> 10-07 07:23:02.835 8272-8272/? D/zhh: java.lang.BootClassLoader@e175998</p> 
  <p>可以看到有两种类加载器，一种是PathClassLoader，另一种则是BootClassLoader。DexPathList中包含了很多apk的路径，其中/data/app/com.example.zhh.moonclassloader-2/base.apk就是示例应用安装在手机上的位置。关于DexPathList后续文章会进行介绍。</p> 
  <p>和Java中的ClassLoader一样，虽然系统所提供的类加载器主要有3种类型，但是系统提供的ClassLoader相关类却不只3个。ClassLoader的继承关系如下图所示。<br /> <img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly9zMi5heDF4LmNvbS8yMDE5LzA1LzI5L1ZuTTVaVC5wbmc" alt="VnM5ZT.png" /><br /> 可以看到上面一共有8个ClassLoader相关类，其中有一些和Java中的ClassLoader相关类十分类似，下面简单对它们进行介绍：</p> 
  <ul>
   <li>ClassLoader是一个抽象类，其中定义了ClassLoader的主要功能。BootClassLoader是它的内部类。</li>
   <li>SecureClassLoader类和JDK8中的SecureClassLoader类的代码是一样的，它继承了抽象类ClassLoader。SecureClassLoader并不是ClassLoader的实现类，而是拓展了ClassLoader类加入了权限方面的功能，加强了ClassLoader的安全性。</li>
   <li>URLClassLoader类和JDK8中的URLClassLoader类的代码是一样的，它继承自SecureClassLoader，用来通过URl路径从jar文件和文件夹中加载类和资源。</li>
   <li>InMemoryDexClassLoader是Android8.0新增的类加载器，继承自BaseDexClassLoader，用于加载内存中的dex文件。</li>
   <li>BaseDexClassLoader继承自ClassLoader，是抽象类ClassLoader的具体实现类，PathClassLoader和DexClassLoader都继承它。</li>
  </ul> 
  <h3><a id="3BootClassLoader_106"></a><strong>3.BootClassLoader的创建</strong></h3> 
  <p>BootClassLoader是在何时被创建的呢？答案是在zygote中，ZygoteInit的main方法如下所示。<br /> <strong>frameworks/base/core/java/com/android/internal/os/ZygoteInit.java</strong></p> 
  <pre><code class="prism language-java"> <span class="token keyword">public</span> <span class="token keyword">static</span> <span class="token keyword">void</span> <span class="token function">main</span><span class="token punctuation">(</span>String argv<span class="token punctuation">[</span><span class="token punctuation">]</span><span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span>
   <span class="token punctuation">.</span><span class="token punctuation">.</span><span class="token punctuation">.</span>
        <span class="token keyword">try</span> <span class="token punctuation">{
     <!-- --></span>
             <span class="token punctuation">.</span><span class="token punctuation">.</span><span class="token punctuation">.</span>
                <span class="token function">preload</span><span class="token punctuation">(</span>bootTimingsTraceLog<span class="token punctuation">)</span><span class="token punctuation">;</span>
             <span class="token punctuation">.</span><span class="token punctuation">.</span><span class="token punctuation">.</span> 
        <span class="token punctuation">}</span>
    <span class="token punctuation">}</span>
</code></pre> 
  <p>main方法是ZygoteInit入口方法，其中调用了ZygoteInit的preload方法，preload方法中又调用了ZygoteInit的preloadClasses方法，如下所示。<br /> <strong>frameworks/base/core/java/com/android/internal/os/ZygoteInit.java</strong></p> 
  <pre><code class="prism language-java"> <span class="token keyword">private</span> <span class="token keyword">static</span> <span class="token keyword">void</span> <span class="token function">preloadClasses</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span>
        <span class="token keyword">final</span> VMRuntime runtime <span class="token operator">=</span> VMRuntime<span class="token punctuation">.</span><span class="token function">getRuntime</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        InputStream is<span class="token punctuation">;</span>
        <span class="token keyword">try</span> <span class="token punctuation">{
     <!-- --></span>
            is <span class="token operator">=</span> <span class="token keyword">new</span> <span class="token class-name">FileInputStream</span><span class="token punctuation">(</span>PRELOADED_CLASSES<span class="token punctuation">)</span><span class="token punctuation">;</span><span class="token comment">//1</span>
        <span class="token punctuation">}</span> <span class="token keyword">catch</span> <span class="token punctuation">(</span><span class="token class-name">FileNotFoundException</span> e<span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span>
            Log<span class="token punctuation">.</span><span class="token function">e</span><span class="token punctuation">(</span>TAG<span class="token punctuation">,</span> <span class="token string">&quot;Couldn't find &quot;</span> <span class="token operator">+</span> PRELOADED_CLASSES <span class="token operator">+</span> <span class="token string">&quot;.&quot;</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
            <span class="token keyword">return</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span>
        <span class="token punctuation">.</span><span class="token punctuation">.</span><span class="token punctuation">.</span>
        <span class="token keyword">try</span> <span class="token punctuation">{
     <!-- --></span>
            BufferedReader br
                <span class="token operator">=</span> <span class="token keyword">new</span> <span class="token class-name">BufferedReader</span><span class="token punctuation">(</span><span class="token keyword">new</span> <span class="token class-name">InputStreamReader</span><span class="token punctuation">(</span>is<span class="token punctuation">)</span><span class="token punctuation">,</span> <span class="token number">256</span><span class="token punctuation">)</span><span class="token punctuation">;</span><span class="token comment">//2</span>

            <span class="token keyword">int</span> count <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">;</span>
            String line<span class="token punctuation">;</span>
            <span class="token keyword">while</span> <span class="token punctuation">(</span><span class="token punctuation">(</span>line <span class="token operator">=</span> br<span class="token punctuation">.</span><span class="token function">readLine</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">)</span> <span class="token operator">!=</span> null<span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span><span class="token comment">//3</span>
                line <span class="token operator">=</span> line<span class="token punctuation">.</span><span class="token function">trim</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
                <span class="token keyword">if</span> <span class="token punctuation">(</span>line<span class="token punctuation">.</span><span class="token function">startsWith</span><span class="token punctuation">(</span><span class="token string">&quot;#&quot;</span><span class="token punctuation">)</span> <span class="token operator">||</span> line<span class="token punctuation">.</span><span class="token function">equals</span><span class="token punctuation">(</span><span class="token string">&quot;&quot;</span><span class="token punctuation">)</span><span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span>
                    <span class="token keyword">continue</span><span class="token punctuation">;</span>
                <span class="token punctuation">}</span>
                  Trace<span class="token punctuation">.</span><span class="token function">traceBegin</span><span class="token punctuation">(</span>Trace<span class="token punctuation">.</span>TRACE_TAG_DALVIK<span class="token punctuation">,</span> line<span class="token punctuation">)</span><span class="token punctuation">;</span>
                <span class="token keyword">try</span> <span class="token punctuation">{
     <!-- --></span>
                    <span class="token keyword">if</span> <span class="token punctuation">(</span><span class="token boolean">false</span><span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span>
                        Log<span class="token punctuation">.</span><span class="token function">v</span><span class="token punctuation">(</span>TAG<span class="token punctuation">,</span> <span class="token string">&quot;Preloading &quot;</span> <span class="token operator">+</span> line <span class="token operator">+</span> <span class="token string">&quot;...&quot;</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
                    <span class="token punctuation">}</span>
                    Class<span class="token punctuation">.</span><span class="token function">forName</span><span class="token punctuation">(</span>line<span class="token punctuation">,</span> <span class="token boolean">true</span><span class="token punctuation">,</span> null<span class="token punctuation">)</span><span class="token punctuation">;</span><span class="token comment">//4</span>
                    count<span class="token operator">++</span><span class="token punctuation">;</span>
                <span class="token punctuation">}</span> <span class="token keyword">catch</span> <span class="token punctuation">(</span><span class="token class-name">ClassNotFoundException</span> e<span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span>
                    Log<span class="token punctuation">.</span><span class="token function">w</span><span class="token punctuation">(</span>TAG<span class="token punctuation">,</span> <span class="token string">&quot;Class not found for preloading: &quot;</span> <span class="token operator">+</span> line<span class="token punctuation">)</span><span class="token punctuation">;</span>
                <span class="token punctuation">}</span> 
        <span class="token punctuation">.</span><span class="token punctuation">.</span><span class="token punctuation">.</span>
        <span class="token punctuation">}</span> <span class="token keyword">catch</span> <span class="token punctuation">(</span><span class="token class-name">IOException</span> e<span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span>
            Log<span class="token punctuation">.</span><span class="token function">e</span><span class="token punctuation">(</span>TAG<span class="token punctuation">,</span> <span class="token string">&quot;Error reading &quot;</span> <span class="token operator">+</span> PRELOADED_CLASSES <span class="token operator">+</span> <span class="token string">&quot;.&quot;</span><span class="token punctuation">,</span> e<span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span> <span class="token keyword">finally</span> <span class="token punctuation">{
     <!-- --></span>
            <span class="token punctuation">.</span><span class="token punctuation">.</span><span class="token punctuation">.</span>
        <span class="token punctuation">}</span>
    <span class="token punctuation">}</span>
</code></pre> 
  <p>preloadClasses方法用于Zygote进程初始化时预加载常用类。注释1处将/system/etc/preloaded-classes文件封装成FileInputStream，preloaded-classes文件中存有预加载类的目录，这个文件在系统源码中的路径为frameworks/base/preloaded-classes，这里列举一些preloaded-classes文件中的预加载类名称，如下所示。</p> 
  <pre><code>android.app.ApplicationLoaders
android.app.ApplicationPackageManager
android.app.ApplicationPackageManager$OnPermissionsChangeListenerDelegate
android.app.ApplicationPackageManager$ResourceName
android.app.ContentProviderHolder
android.app.ContentProviderHolder$1
android.app.ContextImpl
android.app.ContextImpl$ApplicationContentResolver
android.app.DexLoadReporter
android.app.Dialog
android.app.Dialog$ListenersHandler
android.app.DownloadManager
android.app.Fragment
</code></pre> 
  <p>可以看到preloaded-classes文件中的预加载类的名称有很多都是我们非常熟知的。预加载属于拿空间换时间的策略，Zygote环境配置的越健全越通用，应用程序进程需要单独做的事情也就越少，预加载除了预加载类，还有预加载资源和预加载共享库，因为不是本文重点，这里就不在延伸讲下去了。<br /> 回到preloadClasses方法的注释2处，将FileInputStream封装为BufferedReader，并注释3处遍历BufferedReader，读出所有预加载类的名称，每读出一个预加载类的名称就调用注释4处的代码加载该类，Class的forName方法如下所示。<br /> <strong>libcore/ojluni/src/main/java/java/lang/Class.java</strong></p> 
  <pre><code class="prism language-java">    <span class="token annotation punctuation">@CallerSensitive</span>
    <span class="token keyword">public</span> <span class="token keyword">static</span> Class<span class="token operator">&lt;</span><span class="token operator">?</span><span class="token operator">&gt;</span> <span class="token function">forName</span><span class="token punctuation">(</span>String name<span class="token punctuation">,</span> <span class="token keyword">boolean</span> initialize<span class="token punctuation">,</span>
                                   ClassLoader loader<span class="token punctuation">)</span>
        <span class="token keyword">throws</span> ClassNotFoundException
    <span class="token punctuation">{
     <!-- --></span>
        <span class="token keyword">if</span> <span class="token punctuation">(</span>loader <span class="token operator">==</span> null<span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span>
            loader <span class="token operator">=</span> BootClassLoader<span class="token punctuation">.</span><span class="token function">getInstance</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span><span class="token comment">//1</span>
        <span class="token punctuation">}</span>
        Class<span class="token operator">&lt;</span><span class="token operator">?</span><span class="token operator">&gt;</span> result<span class="token punctuation">;</span>
        <span class="token keyword">try</span> <span class="token punctuation">{
     <!-- --></span>
            result <span class="token operator">=</span> <span class="token function">classForName</span><span class="token punctuation">(</span>name<span class="token punctuation">,</span> initialize<span class="token punctuation">,</span> loader<span class="token punctuation">)</span><span class="token punctuation">;</span><span class="token comment">//2</span>
        <span class="token punctuation">}</span> <span class="token keyword">catch</span> <span class="token punctuation">(</span><span class="token class-name">ClassNotFoundException</span> e<span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span>
            Throwable cause <span class="token operator">=</span> e<span class="token punctuation">.</span><span class="token function">getCause</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
            <span class="token keyword">if</span> <span class="token punctuation">(</span>cause <span class="token keyword">instanceof</span> <span class="token class-name">LinkageError</span><span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span>
                <span class="token keyword">throw</span> <span class="token punctuation">(</span>LinkageError<span class="token punctuation">)</span> cause<span class="token punctuation">;</span>
            <span class="token punctuation">}</span>
            <span class="token keyword">throw</span> e<span class="token punctuation">;</span>
        <span class="token punctuation">}</span>
        <span class="token keyword">return</span> result<span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
</code></pre> 
  <p>注释1处创建了BootClassLoader，并将BootClassLoader实例传入到了注释2处的classForName方法中，classForName方法是Native方法，它的实现由c/c++代码来完成，如下所示。</p> 
  <pre><code class="prism language-java">    <span class="token annotation punctuation">@FastNative</span>
    <span class="token keyword">static</span> <span class="token keyword">native</span> Class<span class="token operator">&lt;</span><span class="token operator">?</span><span class="token operator">&gt;</span> <span class="token function">classForName</span><span class="token punctuation">(</span>String className<span class="token punctuation">,</span> <span class="token keyword">boolean</span> shouldInitialize<span class="token punctuation">,</span>
            ClassLoader classLoader<span class="token punctuation">)</span> <span class="token keyword">throws</span> ClassNotFoundException<span class="token punctuation">;</span>
</code></pre> 
  <h3><a id="4PathClassLoader_212"></a><strong>4.PathClassLoader的创建</strong></h3> 
  <p>PathClassLoader的创建也得从Zygote进程开始说起，Zygote进程启动SyetemServer进程时会调用ZygoteInit的startSystemServer方法，如下所示。<br /> <strong>frameworks/base/core/java/com/android/internal/os/ZygoteInit.java</strong></p> 
  <pre><code class="prism language-java"><span class="token keyword">private</span> <span class="token keyword">static</span> <span class="token keyword">boolean</span> <span class="token function">startSystemServer</span><span class="token punctuation">(</span>String abiList<span class="token punctuation">,</span> String socketName<span class="token punctuation">)</span>
           <span class="token keyword">throws</span> MethodAndArgsCaller<span class="token punctuation">,</span> RuntimeException <span class="token punctuation">{
     <!-- --></span>
    <span class="token punctuation">.</span><span class="token punctuation">.</span><span class="token punctuation">.</span>
        <span class="token keyword">int</span> pid<span class="token punctuation">;</span>
        <span class="token keyword">try</span> <span class="token punctuation">{
     <!-- --></span>
            parsedArgs <span class="token operator">=</span> <span class="token keyword">new</span> <span class="token class-name">ZygoteConnection<span class="token punctuation">.</span>Arguments</span><span class="token punctuation">(</span>args<span class="token punctuation">)</span><span class="token punctuation">;</span><span class="token comment">//2</span>
            ZygoteConnection<span class="token punctuation">.</span><span class="token function">applyDebuggerSystemProperty</span><span class="token punctuation">(</span>parsedArgs<span class="token punctuation">)</span><span class="token punctuation">;</span>
            ZygoteConnection<span class="token punctuation">.</span><span class="token function">applyInvokeWithSystemProperty</span><span class="token punctuation">(</span>parsedArgs<span class="token punctuation">)</span><span class="token punctuation">;</span>
            <span class="token comment">/*1*/</span>
            pid <span class="token operator">=</span> Zygote<span class="token punctuation">.</span><span class="token function">forkSystemServer</span><span class="token punctuation">(</span>
                    parsedArgs<span class="token punctuation">.</span>uid<span class="token punctuation">,</span> parsedArgs<span class="token punctuation">.</span>gid<span class="token punctuation">,</span>
                    parsedArgs<span class="token punctuation">.</span>gids<span class="token punctuation">,</span>
                    parsedArgs<span class="token punctuation">.</span>debugFlags<span class="token punctuation">,</span>
                    null<span class="token punctuation">,</span>
                    parsedArgs<span class="token punctuation">.</span>permittedCapabilities<span class="token punctuation">,</span>
                    parsedArgs<span class="token punctuation">.</span>effectiveCapabilities<span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span> <span class="token keyword">catch</span> <span class="token punctuation">(</span><span class="token class-name">IllegalArgumentException</span> ex<span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span>
            <span class="token keyword">throw</span> <span class="token keyword">new</span> <span class="token class-name">RuntimeException</span><span class="token punctuation">(</span>ex<span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span>
       <span class="token keyword">if</span> <span class="token punctuation">(</span>pid <span class="token operator">==</span> <span class="token number">0</span><span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span><span class="token comment">//2</span>
           <span class="token keyword">if</span> <span class="token punctuation">(</span><span class="token function">hasSecondZygote</span><span class="token punctuation">(</span>abiList<span class="token punctuation">)</span><span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span>
               <span class="token function">waitForSecondaryZygote</span><span class="token punctuation">(</span>socketName<span class="token punctuation">)</span><span class="token punctuation">;</span>
           <span class="token punctuation">}</span>
           <span class="token function">handleSystemServerProcess</span><span class="token punctuation">(</span>parsedArgs<span class="token punctuation">)</span><span class="token punctuation">;</span><span class="token comment">//3</span>
       <span class="token punctuation">}</span>
       <span class="token keyword">return</span> <span class="token boolean">true</span><span class="token punctuation">;</span>
   <span class="token punctuation">}</span>
</code></pre> 
  <p>注释1处，Zygote进程通过forkSystemServer方法fork自身创建子进程（SystemServer进程）。注释2处如果forkSystemServer方法返回的pid等于0，说明当前代码是在新创建的SystemServer进程中执行的，接着就会执行注释3处的handleSystemServerProcess方法：<br /> <strong>frameworks/base/core/java/com/android/internal/os/ZygoteInit.java</strong></p> 
  <pre><code class="prism language-java"> <span class="token keyword">private</span> <span class="token keyword">static</span> <span class="token keyword">void</span> <span class="token function">handleSystemServerProcess</span><span class="token punctuation">(</span>
            ZygoteConnection<span class="token punctuation">.</span>Arguments parsedArgs<span class="token punctuation">)</span>
            <span class="token keyword">throws</span> Zygote<span class="token punctuation">.</span>MethodAndArgsCaller <span class="token punctuation">{
     <!-- --></span>

    <span class="token punctuation">.</span><span class="token punctuation">.</span><span class="token punctuation">.</span>
        <span class="token keyword">if</span> <span class="token punctuation">(</span>parsedArgs<span class="token punctuation">.</span>invokeWith <span class="token operator">!=</span> null<span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span>
           <span class="token punctuation">.</span><span class="token punctuation">.</span><span class="token punctuation">.</span>
        <span class="token punctuation">}</span> <span class="token keyword">else</span> <span class="token punctuation">{
     <!-- --></span>
            ClassLoader cl <span class="token operator">=</span> null<span class="token punctuation">;</span>
            <span class="token keyword">if</span> <span class="token punctuation">(</span>systemServerClasspath <span class="token operator">!=</span> null<span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span>
                cl <span class="token operator">=</span> <span class="token function">createPathClassLoader</span><span class="token punctuation">(</span>systemServerClasspath<span class="token punctuation">,</span> parsedArgs<span class="token punctuation">.</span>targetSdkVersion<span class="token punctuation">)</span><span class="token punctuation">;</span><span class="token comment">//1</span>
                Thread<span class="token punctuation">.</span><span class="token function">currentThread</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">.</span><span class="token function">setContextClassLoader</span><span class="token punctuation">(</span>cl<span class="token punctuation">)</span><span class="token punctuation">;</span>
            <span class="token punctuation">}</span>
            ZygoteInit<span class="token punctuation">.</span><span class="token function">zygoteInit</span><span class="token punctuation">(</span>parsedArgs<span class="token punctuation">.</span>targetSdkVersion<span class="token punctuation">,</span> parsedArgs<span class="token punctuation">.</span>remainingArgs<span class="token punctuation">,</span> cl<span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span>
    <span class="token punctuation">}</span>
</code></pre> 
  <p>注释1处调用了createPathClassLoader方法，如下所示。<br /> <strong>frameworks/base/core/java/com/android/internal/os/ZygoteInit.java</strong></p> 
  <pre><code class="prism language-java">  <span class="token keyword">static</span> PathClassLoader <span class="token function">createPathClassLoader</span><span class="token punctuation">(</span>String classPath<span class="token punctuation">,</span> <span class="token keyword">int</span> targetSdkVersion<span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span>
      String libraryPath <span class="token operator">=</span> System<span class="token punctuation">.</span><span class="token function">getProperty</span><span class="token punctuation">(</span><span class="token string">&quot;java.library.path&quot;</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
      <span class="token keyword">return</span> PathClassLoaderFactory<span class="token punctuation">.</span><span class="token function">createClassLoader</span><span class="token punctuation">(</span>classPath<span class="token punctuation">,</span>
                                                      libraryPath<span class="token punctuation">,</span>
                                                      libraryPath<span class="token punctuation">,</span>
                                                      ClassLoader<span class="token punctuation">.</span><span class="token function">getSystemClassLoader</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span>
                                                      targetSdkVersion<span class="token punctuation">,</span>
                                                      <span class="token boolean">true</span> <span class="token comment">/* isNamespaceShared */</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
</code></pre> 
  <p>createPathClassLoader方法中又会调用PathClassLoaderFactory的createClassLoader方法，看来PathClassLoader是用工厂来进行创建的。<br /> <strong>frameworks/base/core/java/com/android/internal/os/PathClassLoaderFactory.java</strong></p> 
  <pre><code class="prism language-java">  <span class="token keyword">public</span> <span class="token keyword">static</span> PathClassLoader <span class="token function">createClassLoader</span><span class="token punctuation">(</span>String dexPath<span class="token punctuation">,</span>
                                                    String librarySearchPath<span class="token punctuation">,</span>
                                                    String libraryPermittedPath<span class="token punctuation">,</span>
                                                    ClassLoader parent<span class="token punctuation">,</span>
                                                    <span class="token keyword">int</span> targetSdkVersion<span class="token punctuation">,</span>
                                                    <span class="token keyword">boolean</span> isNamespaceShared<span class="token punctuation">)</span> <span class="token punctuation">{
     <!-- --></span>
        PathClassLoader pathClassloader <span class="token operator">=</span> <span class="token keyword">new</span> <span class="token class-name">PathClassLoader</span><span class="token punctuation">(</span>dexPath<span class="token punctuation">,</span> librarySearchPath<span class="token punctuation">,</span> parent<span class="token punctuation">)</span><span class="token punctuation">;</span>
      <span class="token punctuation">.</span><span class="token punctuation">.</span><span class="token punctuation">.</span>
        <span class="token keyword">return</span> pathClassloader<span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
</code></pre> 
  <p>在PathClassLoaderFactory的createClassLoader方法中会创建PathClassLoader。</p> 
  <h3><a id="_294"></a><strong>结语</strong></h3> 
  <p>在这篇文章中我们学习了Android的ClassLoader的类型、ClassLoader的继承关系以及BootClassLoader和PathClassLoader是何时创建的。BootClassLoader是在Zygote进程的入口方法中创建的，PathClassLoader则是在Zygote进程创建SystemServer进程时创建的。后续有时间会分析下Android中的ClassLoader的其他知识点。</p> 
