---
layout:     post
title:      Android10系统启动之Zygote进程详解
subtitle:   这篇文章我们来详细学习下Android10系统启动中Zygote进程的启动过程
date:       2020-03-11
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android10
    - 系统启动
    - Zygote
--- 

<h1>1.概述</h1> 
<p>上一节接讲解了InIt进程的整个启动流程。Init进程启动后&#xff0c;最重要的一个进程就是Zygote进程,Zygote是所有应用的鼻祖。SystemServer和其他所有Dalivik虚拟机进程都是由Zygote fork而来。</p> 
<p>Zygote进程由app_process启动&#xff0c;Zygote是一个C/S模型&#xff0c;Zygote进程作为服务端&#xff0c;其他进程作为客户端向它发出“孵化-fork”请求&#xff0c;而Zygote接收到这个请求后就“孵化-fork”出一个新的进程。</p> 
<p>由于Zygote进程在启动时会创建Java虚拟机&#xff0c;因此通过fork而创建的应用程序进程和SystemServer进程可以在内部获取一个Java虚拟机的实例拷贝。</p> 
<p> </p> 
<h1>2.核心源码</h1> 
<pre class="has"><code class="language-cpp">/system/core/rootdir/init.rc
/system/core/init/main.cpp
/system/core/init/init.cpp
/system/core/rootdir/init.zygote64_32.rc
/frameworks/base/cmds/app_process/app_main.cpp
/frameworks/base/core/jni/AndroidRuntime.cpp
/libnativehelper/JniInvocation.cpp
/frameworks/base/core/java/com/android/internal/os/Zygote.java
/frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
/frameworks/base/core/java/com/android/internal/os/ZygoteServer.java
/frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java
/frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
/frameworks/base/core/java/android/net/LocalServerSocket.java
/system/core/libutils/Threads.cpp</code></pre> 
<h1>3.架构</h1> 
<h2>3.1 架构图</h2> 
<p> </p> 
<p><img alt="" class="has" height="293" src="https://img-blog.csdnimg.cn/20191215161738755.png?x-oss-process&#61,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpcmFuZmVuZw&#61;&#61;,size_16,color_FFFFFF,t_70" width="644" /></p> 
<h2>3.2 Zygote 是如何被启动的</h2> 
<p><strong>rc解析和进程调用</strong></p> 
<p>Init进程启动后&#xff0c;会解析init.rc文件&#xff0c;然后创建和加载service字段指定的进程。zygote进程就是以这种方式&#xff0c;被init进程加载的。</p> 
<p>在 /system/core/rootdir/init.rc中&#xff0c;通过如下引用来load Zygote的rc&#xff1a;</p> 
<pre class="has"><code>import /init.${ro.zygote}.rc</code></pre> 
<p>其中${ro.zygote} 由各个厂家使用&#xff0c;现在的主流厂家基本使用zygote64_32&#xff0c;因此&#xff0c;我们的rc文件为 init.zygote64_32.rc</p> 
<p> </p> 
<h3>3.2.1 init.zygote64_32.rc </h3> 
<p><strong>第一个Zygote进程</strong>&#xff1a;</p> 
<p><strong>进程名&#xff1a;</strong>zygote</p> 
<p>进程通过 /system/bin/app_process64来启动</p> 
<p><strong>启动参数&#xff1a;</strong>-Xzygote /system/bin --zygote --start-system-server --socket-name&#61;zygote</p> 
<p><strong>socket的名称&#xff1a;</strong>zygote</p> 
<pre class="has"><code class="language-cpp">   service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name&#61;zygote
            class main
            priority -20
            user root                                   //用户为root
            group root readproc reserved_disk           //访问组支持 root readproc reserved_disk
            socket zygote stream 660 root system        //创建一个socket&#xff0c;名字叫zygote&#xff0c;以tcp形式  &#xff0c;可以在/dev/socket 中看到一个 zygote的socket
            socket usap_pool_primary stream 660 root system
            onrestart write /sys/android_power/request_state wake       // onrestart 指当进程重启时执行后面的命令
            onrestart write /sys/power/state on
            onrestart restart audioserver
            onrestart restart cameraserver
            onrestart restart media
            onrestart restart netd
            onrestart restart wificond
            onrestart restart vendor.servicetracker-1-1
            writepid /dev/cpuset/foreground/tasks       // 创建子进程时&#xff0c;向 /dev/cpuset/foreground/tasks 写入pid</code></pre> 
<p><strong>第二个Zygote进程&#xff1a;</strong></p> 
<p> </p> 
<p><strong>zygote_secondary</strong></p> 
<p>进程通过 /system/bin/app_process32来启动</p> 
<p><strong>启动参数&#xff1a;</strong>-Xzygote /system/bin --zygote --socket-name&#61;zygote_secondary --enable-lazy-preload</p> 
<p><strong>socket的名称&#xff1a;</strong>zygote_secondary</p> 
<pre class="has"><code class="language-cpp">     service zygote_secondary /system/bin/app_process32 -Xzygote /system/bin --zygote --socket-name&#61;zygote_secondary --enable-lazy-preload
            class main
            priority -20
            user root
            group root readproc reserved_disk
            socket zygote_secondary stream 660 root system      //创建一个socket&#xff0c;名字叫zygote_secondary&#xff0c;以tcp形式  &#xff0c;可以在/dev/socket 中看到一个 zygote_secondary的socket
            socket usap_pool_secondary stream 660 root system
            onrestart restart zygote
            writepid /dev/cpuset/foreground/tasks</code></pre> 
<p>从上面我们可以看出&#xff0c;zygote是通过进程文件 /system/bin/app_process64 和/system/bin/app_process32 来启动的。对应的代码入口为:</p> 
<p><strong>frameworks/base/cmds/app_process/app_main.cpp</strong></p> 
<p> </p> 
<h3>3.2.2 Zygote进程在什么时候会被重启</h3> 
<p>Zygote进程重启&#xff0c;主要查看rc文件中有没有 “restart zygote” 这句话。在整个Android系统工程中<strong>搜索“restart zygote”</strong>&#xff0c;会发现以下文件&#xff1a;</p> 
<pre class="has"><code class="language-cpp">/frameworks/native/services/inputflinger/host/inputflinger.rc      对应进程&#xff1a; inputflinger
/frameworks/native/cmds/servicemanager/servicemanager.rc           对应进程&#xff1a; servicemanager
/frameworks/native/services/surfaceflinger/surfaceflinger.rc       对应进程&#xff1a; surfaceflinger
/system/netd/server/netd.rc                                        对应进程&#xff1a; netd</code></pre> 
<p>通过上面的文件可知&#xff0c;zygote进程能够重启的时机&#xff1a;</p> 
<ol><li>inputflinger 进程被杀 &#xff08;onrestart&#xff09;</li><li>servicemanager 进程被杀 &#xff08;onrestart&#xff09;</li><li>surfaceflinger 进程被杀 &#xff08;onrestart&#xff09;</li><li>netd 进程被杀 &#xff08;onrestart&#xff09;</li><li>zygote进程被杀 (oneshot&#61;false)</li><li> system_server进程被杀&#xff08;waitpid&#xff09;</li></ol>
<p> </p> 
<h2>3.3 Zygote 启动后做了什么</h2> 
<p> </p> 
<p> Zygote启动时序图&#xff1a;</p> 
<p><img alt="" class="has" height="408" src="https://img-blog.csdnimg.cn/2019121516215969.png?x-oss-process&#61,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpcmFuZmVuZw&#61;&#61;,size_16,color_FFFFFF,t_70" width="667" /></p> 
<p> </p> 
<ol><li>init进程通过init.zygote64_32.rc来调用/system/bin/app_process64 来启动zygote进程&#xff0c;入口app_main.cpp</li><li>调用AndroidRuntime的startVM()方法创建虚拟机&#xff0c;再调用startReg()注册JNI函数&#xff1b;</li><li>通过JNI方式调用ZygoteInit.main()&#xff0c;第一次进入Java世界&#xff1b;</li><li>registerZygoteSocket()建立socket通道&#xff0c;zygote作为通信的服务端&#xff0c;用于响应客户端请求&#xff1b;</li><li>preload()预加载通用类、drawable和color资源、openGL以及共享库以及WebView&#xff0c;用于提高app启动效率&#xff1b;</li><li>zygote完毕大部分工作&#xff0c;接下来再通过startSystemServer()&#xff0c;fork得力帮手system_server进程&#xff0c;也是上层framework的运行载体。</li><li>zygote任务完成&#xff0c;调用runSelectLoop()&#xff0c;随时待命&#xff0c;当接收到请求创建新进程请求时立即唤醒并执行相应工作。</li></ol>
<p> </p> 
<h2>3.4 Zygote启动相关主要函数&#xff1a;</h2> 
<p><strong>C空间&#xff1a;</strong></p> 
<pre class="has"><code>[app_main.cpp] main()
[AndroidRuntime.cpp] start()
[JniInvocation.cpp] Init()
[AndroidRuntime.cpp] startVm()
[AndroidRuntime.cpp] startReg()
[Threads.cpp] androidSetCreateThreadFunc
[AndroidRuntime.cpp] register_jni_procs()    --&gt; gRegJNI.mProc</code></pre> 
<p><strong>Java空间&#xff1a;</strong></p> 
<pre class="has"><code>[ZygoteInit.java] main()
[ZygoteInit.java] preload()
[ZygoteServer.java] ZygoteServer
[ZygoteInit.java] forkSystemServer
[Zygote.java] forkSystemServer
[Zygote.java] nativeForkSystemServer
[ZygoteServer.java] runSelectLoop
</code></pre> 
<h1>4. Zygote进程启动源码分析</h1> 
<p>我们主要是分析Android Q(10.0) 的Zygote启动的源码。</p> 
<p> </p> 
<h2>4.1 Nativate-C世界的Zygote启动要代码调用流程&#xff1a;</h2> 
<p> <img alt="" class="has" height="454" src="https://img-blog.csdnimg.cn/20191215162441836.PNG?x-oss-process&#61,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpcmFuZmVuZw&#61;&#61;,size_16,color_FFFFFF,t_70" width="936" /></p> 
<h3>4.1.1 [app_main.cpp] main()</h3> 
<p> </p> 
<pre class="has"><code>int main(int argc, char* const argv[])
{
	 //zygote传入的参数argv为“-Xzygote /system/bin --zygote --start-system-server --socket-name&#61;zygote”
	 //zygote_secondary传入的参数argv为“-Xzygote /system/bin --zygote --socket-name&#61;zygote_secondary”
	...
    while (i &lt; argc) {
        const char* arg &#61; argv[i&#43;&#43;];
        if (strcmp(arg, &#34;--zygote&#34;) &#61;&#61; 0) {
            zygote &#61; true;
			//对于64位系统nice_name为zygote64; 32位系统为zygote
            niceName &#61; ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, &#34;--start-system-server&#34;) &#61;&#61; 0) {
			 //是否需要启动system server
            startSystemServer &#61; true;
        } else if (strcmp(arg, &#34;--application&#34;) &#61;&#61; 0) {
			//启动进入独立的程序模式
            application &#61; true;
        } else if (strncmp(arg, &#34;--nice-name&#61;&#34;, 12) &#61;&#61; 0) {
			//niceName 为当前进程别名&#xff0c;区别abi型号
            niceName.setTo(arg &#43; 12);
        } else if (strncmp(arg, &#34;--&#34;, 2) !&#61; 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }
	..
	if (!className.isEmpty()) { //className不为空&#xff0c;说明是application启动模式
		...
	} else {
	  //进入zygote模式&#xff0c;新建Dalvik的缓存目录:/data/dalvik-cache
        maybeCreateDalvikCache();
		if (startSystemServer) { //加入start-system-server参数
            args.add(String8(&#34;start-system-server&#34;));
        }
		String8 abiFlag(&#34;--abi-list&#61;&#34;);
		abiFlag.append(prop);
		args.add(abiFlag);	//加入--abi-list&#61;参数
		// In zygote mode, pass all remaining arguments to the zygote
		// main() method.
		for (; i &lt; argc; &#43;&#43;i) {	//将剩下的参数加入args
			args.add(String8(argv[i]));
		}
	}
	...
    if (!niceName.isEmpty()) {
//设置一个“好听的昵称” zygote\zygote64&#xff0c;之前的名称是app_process
        runtime.setArgv0(niceName.string(), true /* setProcName */);
    }
    if (zygote) {	 //如果是zygote启动模式&#xff0c;则加载ZygoteInit
        runtime.start(&#34;com.android.internal.os.ZygoteInit&#34;, args, zygote);
    } else if (className) {	//如果是application启动模式&#xff0c;则加载RuntimeInit
        runtime.start(&#34;com.android.internal.os.RuntimeInit&#34;, args, zygote);
    } else {
		//没有指定类名或zygote&#xff0c;参数错误
        fprintf(stderr, &#34;Error: no class name or --zygote supplied.\n&#34;);
        app_usage();
        LOG_ALWAYS_FATAL(&#34;app_process: no class name or --zygote supplied.&#34;);
    }
}</code></pre> 
<p>Zygote本身是一个Native的应用程序&#xff0c;刚开始的进程名称为“app_process”&#xff0c;运行过程中&#xff0c;通过调用setArgv0将名字改为zygote 或者 zygote64(根据操作系统而来)&#xff0c;最后通过runtime的start()方法来真正的加载虚拟机并进入JAVA世界。</p> 
<p> </p> 
<h3>4.1.2 [AndroidRuntime.cpp] start()</h3> 
<p> </p> 
<pre class="has"><code>void AndroidRuntime::start(const char* className, const Vector&lt;String8&gt;&amp; options, bool zygote)
{
    ALOGD(&#34;&gt;&gt;&gt;&gt;&gt;&gt; START %s uid %d &lt;&lt;&lt;&lt;&lt;&lt;\n&#34;,
            className !&#61; NULL ? className : &#34;(unknown)&#34;, getuid());
	...
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
	 // 虚拟机创建&#xff0c;主要是关于虚拟机参数的设置
    if (startVm(&amp;mJavaVM, &amp;env, zygote, primary_zygote) !&#61; 0) {
        return;
    }
    onVmCreated(env);	//空函数&#xff0c;没有任何实现

	// 注册JNI方法
    if (startReg(env) &lt; 0) {
        ALOGE(&#34;Unable to register all android natives\n&#34;);
        return;
    }

    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;

	//等价 strArray&#61; new String[options.size() &#43; 1];
    stringClass &#61; env-&gt;FindClass(&#34;java/lang/String&#34;);
    assert(stringClass !&#61; NULL);
	
	//等价 strArray[0] &#61; &#34;com.android.internal.os.ZygoteInit&#34;
    strArray &#61; env-&gt;NewObjectArray(options.size() &#43; 1, stringClass, NULL);
    assert(strArray !&#61; NULL);
    classNameStr &#61; env-&gt;NewStringUTF(className);
    assert(classNameStr !&#61; NULL);
    env-&gt;SetObjectArrayElement(strArray, 0, classNameStr);

	//strArray[1] &#61; &#34;start-system-server&#34;&#xff1b;
    //strArray[2] &#61; &#34;--abi-list&#61;xxx&#34;&#xff1b;
    //其中xxx为系统响应的cpu架构类型&#xff0c;比如arm64-v8a.
    for (size_t i &#61; 0; i &lt; options.size(); &#43;&#43;i) {
        jstring optionsStr &#61; env-&gt;NewStringUTF(options.itemAt(i).string());
        assert(optionsStr !&#61; NULL);
        env-&gt;SetObjectArrayElement(strArray, i &#43; 1, optionsStr);
    }

	//将&#34;com.android.internal.os.ZygoteInit&#34;转换为&#34;com/android/internal/os/ZygoteInit&#34;
    char* slashClassName &#61; toSlashClassName(className !&#61; NULL ? className : &#34;&#34;);
    jclass startClass &#61; env-&gt;FindClass(slashClassName);
	//找到Zygoteinit类
    if (startClass &#61;&#61; NULL) {
        ALOGE(&#34;JavaVM unable to locate class &#39;%s&#39;\n&#34;, slashClassName);
    } else {
		//找到这个类后就继续找成员函数main方法的Mehtod ID
        jmethodID startMeth &#61; env-&gt;GetStaticMethodID(startClass, &#34;main&#34;,
            &#34;([Ljava/lang/String;)V&#34;);
        if (startMeth &#61;&#61; NULL) {
            ALOGE(&#34;JavaVM unable to find main() in &#39;%s&#39;\n&#34;, className);
        } else {
			// 通过反射调用ZygoteInit.main()方法
            env-&gt;CallStaticVoidMethod(startClass, startMeth, strArray);
        }
    }
	//释放相应对象的内存空间
    free(slashClassName);
    ALOGD(&#34;Shutting down VM\n&#34;);
    if (mJavaVM-&gt;DetachCurrentThread() !&#61; JNI_OK)
        ALOGW(&#34;Warning: unable to detach main thread\n&#34;);
    if (mJavaVM-&gt;DestroyJavaVM() !&#61; 0)
        ALOGW(&#34;Warning: VM did not shut down cleanly\n&#34;);
}
</code></pre> 
<p>start()函数主要做了三件事情&#xff0c;一调用startVm开启虚拟机&#xff0c;二调用startReg注册JNI方法&#xff0c;三就是使用JNI把Zygote进程启动起来。</p> 
<p> </p> 
<p><strong>相关log&#xff1a;</strong></p> 
<p>01-10 11:20:31.369 722 722 D AndroidRuntime: &gt;&gt;&gt;&gt;&gt;&gt; START com.android.internal.os.ZygoteInit uid 0 &lt;&lt;&lt;&lt;&lt;&lt;<br /> 01-10 11:20:31.429 722 722 I AndroidRuntime: Using default boot image<br /> 01-10 11:20:31.429 722 722 I AndroidRuntime: Leaving lock profiling enabled</p> 
<p> </p> 
<h3>4.1.3 [JniInvocation.cpp] Init()</h3> 
<p>Init函数主要作用是初始化JNI&#xff0c;具体工作是首先通过dlopen加载libart.so获得其句柄&#xff0c;然后调用<strong>dlsym从libart.so</strong>中找到<br /><strong>JNI_GetDefaultJavaVMInitArgs</strong>、<strong>JNI_CreateJavaVM</strong>、<strong>JNI_GetCreatedJavaVMs</strong>三个函数地址&#xff0c;赋值给对应成员属性&#xff0c;这三个函数会在后续虚拟机创建中调用.</p> 
<p> </p> 
<pre class="has"><code>bool JniInvocation::Init(const char* library) {
  char buffer[PROP_VALUE_MAX];
  const int kDlopenFlags &#61; RTLD_NOW | RTLD_NODELETE;
  /*
   * 1.dlopen功能是以指定模式打开指定的动态链接库文件&#xff0c;并返回一个句柄
   * 2.RTLD_NOW表示需要在dlopen返回前&#xff0c;解析出所有未定义符号&#xff0c;如果解析不出来&#xff0c;在dlopen会返回NULL
   * 3.RTLD_NODELETE表示在dlclose()期间不卸载库&#xff0c;并且在以后使用dlopen()重新加载库时不初始化库中的静态变量
   */
  handle_ &#61; dlopen(library, kDlopenFlags); // 获取libart.so的句柄
  if (handle_ &#61;&#61; NULL) { //获取失败打印错误日志并尝试再次打开libart.so
    if (strcmp(library, kLibraryFallback) &#61;&#61; 0) {
      // Nothing else to try.
      ALOGE(&#34;Failed to dlopen %s: %s&#34;, library, dlerror());
      return false;
    }
    library &#61; kLibraryFallback;
    handle_ &#61; dlopen(library, kDlopenFlags);
    if (handle_ &#61;&#61; NULL) {
      ALOGE(&#34;Failed to dlopen %s: %s&#34;, library, dlerror());
      return false;
    }
  }
  /*
   * 1.FindSymbol函数内部实际调用的是dlsym
   * 2.dlsym作用是根据 动态链接库 操作句柄(handle)与符号(symbol)&#xff0c;返回符号对应的地址
   * 3.这里实际就是从libart.so中将JNI_GetDefaultJavaVMInitArgs等对应的地址存入&amp;JNI_GetDefaultJavaVMInitArgs_中
   */
  if (!FindSymbol(reinterpret_cast&lt;void**&gt;(&amp;JNI_GetDefaultJavaVMInitArgs_),
                  &#34;JNI_GetDefaultJavaVMInitArgs&#34;)) {
    return false;
  }
  if (!FindSymbol(reinterpret_cast&lt;void**&gt;(&amp;JNI_CreateJavaVM_),
                  &#34;JNI_CreateJavaVM&#34;)) {
    return false;
  }
  if (!FindSymbol(reinterpret_cast&lt;void**&gt;(&amp;JNI_GetCreatedJavaVMs_),
                  &#34;JNI_GetCreatedJavaVMs&#34;)) {
    return false;
  }
  return true;
}
</code></pre> 
<h3>4.1.4 [AndroidRuntime.cpp] startVm()</h3> 
<p>该函数主要作用就是配置虚拟机的相关参数&#xff0c;再调用之前 JniInvocation初始化得到的 JNI_CreateJavaVM_来启动虚拟机。</p> 
<p> </p> 
<p> </p> 
<pre class="has"><code>int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote, bool primary_zygote)
{
    JavaVMInitArgs initArgs;
	...
	 // JNI检测功能&#xff0c;用于native层调用jni函数时进行常规检测&#xff0c;比较弱字符串格式是否符合要求&#xff0c;资源是否正确释放。
	 //该功能一般用于早期系统调试或手机Eng版&#xff0c;对于User版往往不会开启&#xff0c;引用该功能比较消耗系统CPU资源&#xff0c;降低系统性能。
    bool checkJni &#61; false;
    property_get(&#34;dalvik.vm.checkjni&#34;, propBuf, &#34;&#34;);
    if (strcmp(propBuf, &#34;true&#34;) &#61;&#61; 0) {
        checkJni &#61; true;
    } else if (strcmp(propBuf, &#34;false&#34;) !&#61; 0) {
        /* property is neither true nor false; fall back on kernel parameter */
        property_get(&#34;ro.kernel.android.checkjni&#34;, propBuf, &#34;&#34;);
        if (propBuf[0] &#61;&#61; &#39;1&#39;) {
            checkJni &#61; true;
        }
    }
    ALOGV(&#34;CheckJNI is %s\n&#34;, checkJni ? &#34;ON&#34; : &#34;OFF&#34;);
    if (checkJni) {
        /* extended JNI checking */
        addOption(&#34;-Xcheck:jni&#34;);
    }

    addOption(&#34;exit&#34;, (void*) runtime_exit); //将参数放入mOptions数组中

	 //对于不同的软硬件环境&#xff0c;这些参数往往需要调整、优化&#xff0c;从而使系统达到最佳性能
    parseRuntimeOption(&#34;dalvik.vm.heapstartsize&#34;, heapstartsizeOptsBuf, &#34;-Xms&#34;, &#34;4m&#34;);
    parseRuntimeOption(&#34;dalvik.vm.heapsize&#34;, heapsizeOptsBuf, &#34;-Xmx&#34;, &#34;16m&#34;);
    parseRuntimeOption(&#34;dalvik.vm.heapgrowthlimit&#34;, heapgrowthlimitOptsBuf, &#34;-XX:HeapGrowthLimit&#61;&#34;);
    parseRuntimeOption(&#34;dalvik.vm.heapminfree&#34;, heapminfreeOptsBuf, &#34;-XX:HeapMinFree&#61;&#34;);
    parseRuntimeOption(&#34;dalvik.vm.heapmaxfree&#34;, heapmaxfreeOptsBuf, &#34;-XX:HeapMaxFree&#61;&#34;);
	
	...
	
	//检索生成指纹并将其提供给运行时这样&#xff0c;anr转储将包含指纹并可以解析。
    std::string fingerprint &#61; GetProperty(&#34;ro.build.fingerprint&#34;, &#34;&#34;);
    if (!fingerprint.empty()) {
        fingerprintBuf &#61; &#34;-Xfingerprint:&#34; &#43; fingerprint;
        addOption(fingerprintBuf.c_str());
    }

    initArgs.version &#61; JNI_VERSION_1_4;
    initArgs.options &#61; mOptions.editArray(); //将mOptions赋值给initArgs
    initArgs.nOptions &#61; mOptions.size();
    initArgs.ignoreUnrecognized &#61; JNI_FALSE;

     //调用之前JniInvocation初始化的JNI_CreateJavaVM_, 参考[4.1.3]
    if (JNI_CreateJavaVM(pJavaVM, pEnv, &amp;initArgs) &lt; 0) {
        ALOGE(&#34;JNI_CreateJavaVM failed\n&#34;);
        return -1;
    }

    return 0;
}</code></pre> 
<h3>4.1.5 [AndroidRuntime.cpp] startReg()</h3> 
<p>startReg首先是设置了Android创建线程的处理函数&#xff0c;然后创建了一个200容量的局部引用作用域&#xff0c;用于确保不会出现OutOfMemoryException&#xff0c;最后就是调用register_jni_procs进行JNI方法的注册</p> 
<pre class="has"><code>int AndroidRuntime::startReg(JNIEnv* env)
{
    ATRACE_NAME(&#34;RegisterAndroidNatives&#34;);
	//设置Android创建线程的函数javaCreateThreadEtc&#xff0c;这个函数内部是通过Linux的clone来创建线程的
    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);

    ALOGV(&#34;--- registering native functions ---\n&#34;);

	//创建一个200容量的局部引用作用域,这个局部引用其实就是局部变量
    env-&gt;PushLocalFrame(200);

	//注册JNI方法
    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) &lt; 0) {
        env-&gt;PopLocalFrame(NULL);
        return -1;
    }
    env-&gt;PopLocalFrame(NULL); //释放局部引用作用域
    return 0;
}</code></pre> 
<p> </p> 
<h3>4.1.6 [Thread.cpp] androidSetCreateThreadFunc()</h3> 
<p>  虚拟机启动后startReg()过程&#xff0c;会设置线程创建函数指针gCreateThreadFn指向javaCreateThreadEtc.</p> 
<p> </p> 
<pre class="has"><code>void androidSetCreateThreadFunc(android_create_thread_fn func) {
    gCreateThreadFn &#61; func;
}</code></pre> 
<h3><strong>4.1.7 [AndroidRuntime.cpp] register_jni_procs()</strong></h3> 
<p>它的处理是交给RegJNIRec的mProc,RegJNIRec是个很简单的结构体&#xff0c;mProc是个函数指针</p> 
<pre class="has"><code class="language-cpp">static int register_jni_procs(const RegJNIRec array[], size_t count, JNIEnv* env)
{
    for (size_t i &#61; 0; i &lt; count; i&#43;&#43;) {
        if (array[i].mProc(env) &lt; 0) {/ /调用gRegJNI的mProc&#xff0c;参考[4.1.8]
            return -1;
        }
    }
    return 0;
}</code></pre> 
<h3>4.1.8 [AndroidRuntime.cpp] gRegJNI()</h3> 
<p> </p> 
<pre class="has"><code class="language-cpp">static const RegJNIRec gRegJNI[] &#61; {
    REG_JNI(register_com_android_internal_os_RuntimeInit),
    REG_JNI(register_com_android_internal_os_ZygoteInit_nativeZygoteInit),
    REG_JNI(register_android_os_SystemClock),
    REG_JNI(register_android_util_EventLog),
    REG_JNI(register_android_util_Log),
	...
}

    #define REG_JNI(name)      { name, #name }
    struct RegJNIRec {
        int (*mProc)(JNIEnv*);
    };
gRegJNI 中是一堆函数指针&#xff0c;因此循环调用 gRegJNI 的mProc&#xff0c;即等价于调用其所对应的函数指针。
例如调用&#xff1a; register_com_android_internal_os_RuntimeInit
这是一个JNI函数动态注册的标准方法。

int register_com_android_internal_os_RuntimeInit(JNIEnv* env)
{
    const JNINativeMethod methods[] &#61; {
        { &#34;nativeFinishInit&#34;, &#34;()V&#34;,
            (void*) com_android_internal_os_RuntimeInit_nativeFinishInit },
        { &#34;nativeSetExitWithoutCleanup&#34;, &#34;(Z)V&#34;,
            (void*) com_android_internal_os_RuntimeInit_nativeSetExitWithoutCleanup },
    };

    //跟Java侧的com/android/internal/os/RuntimeInit.java 的函数nativeFinishInit() 进行一一对应
    return jniRegisterNativeMethods(env, &#34;com/android/internal/os/RuntimeInit&#34;,
        methods, NELEM(methods));
}</code></pre> 
<h2>4.2 Java世界的Zygote启动主要代码调用流程&#xff1a;</h2> 
<p>上节我们通过JNI调用ZygoteInit的main函数后&#xff0c;Zygote便进入了Java框架层&#xff0c;此前没有任何代码进入过Java框架层&#xff0c;换句换说Zygote开创了Java框架层。</p> 
<p> <img alt="" class="has" height="664" src="https://img-blog.csdnimg.cn/20191215162957869.PNG?x-oss-process&#61,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpcmFuZmVuZw&#61;&#61;,size_16,color_FFFFFF,t_70" width="995" /></p> 
<p> </p> 
<h3>4.2.1 [ZygoteInit.java]main.cpp</h3> 
<p><strong>代码路径:</strong>frameworks\base\core\java\com\android\internal\os\ZygoteInit.java</p> 
<p><strong>main的主要工作&#xff1a;</strong></p> 
<ol><li>调用preload()来预加载类和资源</li><li>调用ZygoteServer()创建两个Server端的Socket--/dev/socket/zygote 和 /dev/socket/zygote_secondary&#xff0c;Socket用来等待ActivityManagerService来请求Zygote来创建新的应用程序进程。</li><li>调用forkSystemServer 来启动SystemServer进程&#xff0c;这样系统的关键服务也会由SystemServer进程启动起来。</li><li>最后调用runSelectLoop函数来等待客户端请求</li></ol>
<p>下面我们主要来分析这四件事。</p> 
<p> </p> 
<pre class="has"><code class="language-cpp">public static void main(String argv[]) {
        // 1.创建ZygoteServer
        ZygoteServer zygoteServer &#61; null;

        // 调用native函数&#xff0c;确保当前没有其它线程在运行
        ZygoteHooks.startZygoteNoThreadCreation();
        
        //设置pid为0&#xff0c;Zygote进入自己的进程组
        Os.setpgid(0, 0);
        ......
        Runnable caller;
        try {
            ......
            //得到systrace的监控TAG
            String bootTimeTag &#61; Process.is64Bit() ? &#34;Zygote64Timing&#34; : &#34;Zygote32Timing&#34;;
            TimingsTraceLog bootTimingsTraceLog &#61; new TimingsTraceLog(bootTimeTag,
                    Trace.TRACE_TAG_DALVIK);
            //通过systradce来追踪 函数ZygoteInit&#xff0c; 可以通过systrace工具来进行分析
            //traceBegin 和 traceEnd 要成对出现&#xff0c;而且需要使用同一个tag
            bootTimingsTraceLog.traceBegin(&#34;ZygoteInit&#34;);

            //开启DDMS(Dalvik Debug Monitor Service)功能
            //注册所有已知的Java VM的处理块的监听器。线程监听、内存监听、native 堆内存监听、debug模式监听等等
            RuntimeInit.enableDdms();

            boolean startSystemServer &#61; false;
            String zygoteSocketName &#61; &#34;zygote&#34;;
            String abiList &#61; null;
            boolean enableLazyPreload &#61; false;
            
            //2. 解析app_main.cpp - start()传入的参数
            for (int i &#61; 1; i &lt; argv.length; i&#43;&#43;) {
                if (&#34;start-system-server&#34;.equals(argv[i])) {
                    startSystemServer &#61; true; //启动zygote时&#xff0c;才会传入参数&#xff1a;start-system-server
                } else if (&#34;--enable-lazy-preload&#34;.equals(argv[i])) {
                    enableLazyPreload &#61; true; //启动zygote_secondary时&#xff0c;才会传入参数&#xff1a;enable-lazy-preload
                } else if (argv[i].startsWith(ABI_LIST_ARG)) { //通过属性ro.product.cpu.abilist64\ro.product.cpu.abilist32 从C空间传来的值
                    abiList &#61; argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    zygoteSocketName &#61; argv[i].substring(SOCKET_NAME_ARG.length()); //会有两种值&#xff1a;zygote和zygote_secondary
                } else {
                    throw new RuntimeException(&#34;Unknown command line argument: &#34; &#43; argv[i]);
                }
            }

            // 根据传入socket name来决定是创建socket还是zygote_secondary
            final boolean isPrimaryZygote &#61; zygoteSocketName.equals(Zygote.PRIMARY_SOCKET_NAME);

            // 在第一次zygote启动时&#xff0c;enableLazyPreload为false&#xff0c;执行preload
            if (!enableLazyPreload) {
                //systrace 追踪 ZygotePreload
                bootTimingsTraceLog.traceBegin(&#34;ZygotePreload&#34;);
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                        SystemClock.uptimeMillis());
                // 3.加载进程的资源和类,参考[4.2.2]
                preload(bootTimingsTraceLog);
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                        SystemClock.uptimeMillis());
                //systrae结束 ZygotePreload的追踪
                bootTimingsTraceLog.traceEnd(); // ZygotePreload
            } else {
                // 延迟预加载&#xff0c; 变更Zygote进程优先级为NORMAL级别&#xff0c;第一次fork时才会preload
                Zygote.resetNicePriority();
            }

            //结束ZygoteInit的systrace追踪
            bootTimingsTraceLog.traceEnd(); // ZygoteInit
            //禁用systrace追踪&#xff0c;以便fork的进程不会从zygote继承过时的跟踪标记
            Trace.setTracingEnabled(false, 0);
            
            // 4.调用ZygoteServer 构造函数&#xff0c;创建socket&#xff0c;会根据传入的参数&#xff0c;
            // 创建两个socket&#xff1a;/dev/socket/zygote 和 /dev/socket/zygote_secondary
            //参考[4.2.3]
            zygoteServer &#61; new ZygoteServer(isPrimaryZygote);

            if (startSystemServer) {
                //5. fork出system server&#xff0c;参考[4.2.4]
                Runnable r &#61; forkSystemServer(abiList, zygoteSocketName, zygoteServer);

                // 启动SystemServer
                if (r !&#61; null) {
                    r.run();
                    return;
                }
            }

            // 6.  zygote进程进入无限循环&#xff0c;处理请求
            caller &#61; zygoteServer.runSelectLoop(abiList);
        } catch (Throwable ex) {
            Log.e(TAG, &#34;System zygote died with exception&#34;, ex);
            throw ex;
        } finally {
            if (zygoteServer !&#61; null) {
                zygoteServer.closeServerSocket();
            }
        }

        // 7.在子进程中退出了选择循环。继续执行命令
        if (caller !&#61; null) {
            caller.run();
        }
    }</code></pre> 
<p><strong>日志&#xff1a;</strong></p> 
<pre class="has"><code>01-10 11:20:32.219   722   722 D Zygote  : begin preload
01-10 11:20:32.219   722   722 I Zygote  : Calling ZygoteHooks.beginPreload()
01-10 11:20:32.249   722   722 I Zygote  : Preloading classes...
01-10 11:20:33.179   722   722 I Zygote  : ...preloaded 7587 classes in 926ms.
01-10 11:20:33.449   722   722 I Zygote  : Preloading resources...
01-10 11:20:33.459   722   722 I Zygote  : ...preloaded 64 resources in 17ms.
01-10 11:20:33.519   722   722 I Zygote  : Preloading shared libraries...
01-10 11:20:33.539   722   722 I Zygote  : Called ZygoteHooks.endPreload()
01-10 11:20:33.539   722   722 I Zygote  : Installed AndroidKeyStoreProvider in 1ms.
01-10 11:20:33.549   722   722 I Zygote  : Warmed up JCA providers in 11ms.
01-10 11:20:33.549   722   722 D Zygote  : end preload
01-10 11:20:33.649   722   722 D Zygote  : Forked child process 1607
01-10 11:20:33.649   722   722 I Zygote  : System server process 1607 has been created
01-10 11:20:33.649   722   722 I Zygote  : Accepting command socket connections
10-15 06:11:07.749   722   722 D Zygote  : Forked child process 2982
10-15 06:11:07.789   722   722 D Zygote  : Forked child process 3004</code></pre> 
<h3>4.2.2 [ZygoteInit.java] preload()</h3> 
<pre class="has"><code class="language-cpp">static void preload(TimingsTraceLog bootTimingsTraceLog) {
        Log.d(TAG, &#34;begin preload&#34;);

        beginPreload(); // Pin ICU Data, 获取字符集转换资源等

        //预加载类的列表---/system/etc/preloaded-classes&#xff0c; 在版本&#xff1a;/frameworks/base/config/preloaded-classes 中&#xff0c;Android10.0中预计有7603左右个类
        //从下面的log看&#xff0c;成功加载了7587个类
        preloadClasses();

        preloadResources(); //加载图片、颜色等资源文件&#xff0c;部分定义在 /frameworks/base/core/res/res/values/arrays.xml中
        ......
        preloadSharedLibraries();   // 加载 android、compiler_rt、jnigraphics等library
        preloadTextResources();    //用于初始化文字资源

        WebViewFactory.prepareWebViewInZygote();    //用于初始化webview;
        endPreload();   //预加载完成&#xff0c;可以查看下面的log
        warmUpJcaProviders();
        Log.d(TAG, &#34;end preload&#34;);

        sPreloadComplete &#61; true;
    }</code></pre> 
<p><strong>什么是预加载&#xff1a;</strong></p> 
<p>预加载是指在zygote进程启动的时候就加载&#xff0c;这样系统只在zygote执行一次加载操作&#xff0c;所有APP用到该资源不需要再重新加载&#xff0c;减少资源加载时间&#xff0c;加快了应用启动速度&#xff0c;一般情况下&#xff0c;系统中App共享的资源会被列为预加载资源。</p> 
<p>zygote fork子进程时&#xff0c;根据fork的copy-on-write机制可知&#xff0c;有些类如果不做改变&#xff0c;甚至都不用复制&#xff0c;子进程可以和父进程共享这部分数据&#xff0c;从而省去不少内存的占用。</p> 
<p> </p> 
<p><strong>预加载的原理&#xff1a;</strong></p> 
<p>zygote进程启动后将资源读取出来&#xff0c;保存到Resources一个全局静态变量中&#xff0c;下次读取系统资源的时候优先从静态变量中查找。</p> 
<p> </p> 
<p>frameworks/base/config/preloaded-classes&#xff1a;</p> 
<p>参考&#xff1a;</p> 
<p><img alt="" class="has" height="333" src="https://img-blog.csdnimg.cn/20191215163250265.png?x-oss-process&#61,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpcmFuZmVuZw&#61;&#61;,size_16,color_FFFFFF,t_70" width="524" /></p> 
<p><strong>相关日志&#xff1a;</strong></p> 
<p>01-10 11:20:32.219 722 722 D Zygote : begin preload<br /> 01-10 11:20:32.219 722 722 I Zygote : Calling ZygoteHooks.beginPreload()<br /> 01-10 11:20:32.249 722 722 I Zygote : Preloading classes...<br /> 01-10 11:20:33.179 722 722 I Zygote : ...preloaded 7587 classes in 926ms.<br /> 01-10 11:20:33.539 722 722 I Zygote : Called ZygoteHooks.endPreload()<br /> 01-10 11:20:33.549 722 722 D Zygote : end preload</p> 
<p> </p> 
<p><strong>4.2.3 [ZygoteServer.java] ZygoteServer()</strong><br /> path: frameworks\base\core\java\com\android\internal\os\ZygoteServer.java<br /> 作用&#xff1a;ZygoteServer 构造函数初始化时&#xff0c;根据传入的参数&#xff0c;利用LocalServerSocket 创建了4个本地服务端的socket&#xff0c;用来建立连接。<br /><strong>      分别是&#xff1a;</strong>zygote、usap_pool_primary、zygote_secondary、usap_pool_secondary</p> 
<pre class="has"><code class="language-java">    private LocalServerSocket mZygoteSocket;
    private LocalServerSocket mUsapPoolSocket;

    //创建zygote的socket
    ZygoteServer(boolean isPrimaryZygote) {
        mUsapPoolEventFD &#61; Zygote.getUsapPoolEventFD();

        if (isPrimaryZygote) {
            //创建socket&#xff0c;并获取socket对象&#xff0c;socketname&#xff1a; zygote
            mZygoteSocket &#61; Zygote.createManagedSocketFromInitSocket(Zygote.PRIMARY_SOCKET_NAME);
            //创建socket&#xff0c;并获取socket对象&#xff0c;socketname&#xff1a;usap_pool_primary
            mUsapPoolSocket &#61;
                    Zygote.createManagedSocketFromInitSocket(
                            Zygote.USAP_POOL_PRIMARY_SOCKET_NAME);
        } else {
            //创建socket&#xff0c;并获取socket对象&#xff0c;socketname&#xff1a; zygote_secondary
            mZygoteSocket &#61; Zygote.createManagedSocketFromInitSocket(Zygote.SECONDARY_SOCKET_NAME);
            //创建socket&#xff0c;并获取socket对象&#xff0c;socketname&#xff1a; usap_pool_secondary
            mUsapPoolSocket &#61;
                    Zygote.createManagedSocketFromInitSocket(
                            Zygote.USAP_POOL_SECONDARY_SOCKET_NAME);
        }
        fetchUsapPoolPolicyProps();
        mUsapPoolSupported &#61; true;
    }

    static LocalServerSocket createManagedSocketFromInitSocket(String socketName) {
        int fileDesc;
        // ANDROID_SOCKET_PREFIX为&#34;ANDROID_SOCKET_&#34; 
        //加入传入参数为zygote&#xff0c;则fullSocketName&#xff1a;ANDROID_SOCKET_zygote
        final String fullSocketName &#61; ANDROID_SOCKET_PREFIX &#43; socketName;

        try {
            //init.zygote64_32.rc启动时&#xff0c;指定了4个socket&#xff1a;
            //分别是zygote、usap_pool_primary、zygote_secondary、usap_pool_secondary
            // 在进程被创建时&#xff0c;就会创建对应的文件描述符&#xff0c;并加入到环境变量中
            // 这里取出对应的环境变量
            String env &#61; System.getenv(fullSocketName);
            fileDesc &#61; Integer.parseInt(env);
        } catch (RuntimeException ex) {
            throw new RuntimeException(&#34;Socket unset or invalid: &#34; &#43; fullSocketName, ex);
        }

        try {
            FileDescriptor fd &#61; new FileDescriptor();
            fd.setInt$(fileDesc);   // 获取zygote socket的文件描述符
            return new LocalServerSocket(fd);   // 创建Socket的本地服务端
        } catch (IOException ex) {
            throw new RuntimeException(
                &#34;Error building socket from file descriptor: &#34; &#43; fileDesc, ex);
        }
    }

    path: \frameworks\base\core\java\android\net\LocalServerSocket.java
    public LocalServerSocket(FileDescriptor fd) throws IOException
    {
        impl &#61; new LocalSocketImpl(fd);
        impl.listen(LISTEN_BACKLOG);
        localAddress &#61; impl.getSockAddress();
    }</code></pre> 
<p> </p> 
<h3>4.2.4 [ZygoteInit.java] forkSystemServer()</h3> 
<pre class="has"><code class="language-java">  private static Runnable forkSystemServer(String abiList, String socketName,
          ZygoteServer zygoteServer) {
      
      long capabilities &#61; posixCapabilitiesAsBits(
              OsConstants.CAP_IPC_LOCK,
              OsConstants.CAP_KILL,
              OsConstants.CAP_NET_ADMIN,
              OsConstants.CAP_NET_BIND_SERVICE,
              OsConstants.CAP_NET_BROADCAST,
              OsConstants.CAP_NET_RAW,
              OsConstants.CAP_SYS_MODULE,
              OsConstants.CAP_SYS_NICE,
              OsConstants.CAP_SYS_PTRACE,
              OsConstants.CAP_SYS_TIME,
              OsConstants.CAP_SYS_TTY_CONFIG,
              OsConstants.CAP_WAKE_ALARM,
              OsConstants.CAP_BLOCK_SUSPEND
      );
      ......
      //参数准备
      /* Hardcoded command line to start the system server */
      String args[] &#61; {
              &#34;--setuid&#61;1000&#34;,
              &#34;--setgid&#61;1000&#34;,
              &#34;--setgroups&#61;1001,1002,1003,1004,1005,1006,1007,1008,1009,1010
,1018,1021,1023,&#34;
                      &#43; &#34;1024,1032,1065,3001,3002,3003,3006,3007,3009,3010&#34;,
              &#34;--capabilities&#61;&#34; &#43; capabilities &#43; &#34;,&#34; &#43; capabilities,
              &#34;--nice-name&#61;system_server&#34;,
              &#34;--runtime-args&#34;,
              &#34;--target-sdk-version&#61;&#34; &#43; VMRuntime.
SDK_VERSION_CUR_DEVELOPMENT,
              &#34;com.android.server.SystemServer&#34;,
      };
      ZygoteArguments parsedArgs &#61; null;

      int pid;

      try {
          //将上面准备的参数&#xff0c;按照ZygoteArguments的风格进行封装
          parsedArgs &#61; new ZygoteArguments(args);
          Zygote.applyDebuggerSystemProperty(parsedArgs);
          Zygote.applyInvokeWithSystemProperty(parsedArgs);

          boolean profileSystemServer &#61; SystemProperties.getBoolean(
                  &#34;dalvik.vm.profilesystemserver&#34;, false);
          if (profileSystemServer) {
              parsedArgs.mRuntimeFlags |&#61; Zygote.PROFILE_SYSTEM_SERVER;
          }

          //通过fork&#34;分裂&#34;出子进程system_server
          /* Request to fork the system server process */
          pid &#61; Zygote.forkSystemServer(
                  parsedArgs.mUid, parsedArgs.mGid,
                  parsedArgs.mGids,
                  parsedArgs.mRuntimeFlags,
                  null,
                  parsedArgs.mPermittedCapabilities,
                  parsedArgs.mEffectiveCapabilities);
      } catch (IllegalArgumentException ex) {
          throw new RuntimeException(ex);
      }

      //进入子进程system_server
      /* For child process */
      if (pid &#61;&#61; 0) {
          // 处理32_64和64_32的情况
          if (hasSecondZygote(abiList)) {
              waitForSecondaryZygote(socketName);
          }

          // fork时会copy socket&#xff0c;system server需要主动关闭
          zygoteServer.closeServerSocket();
          // system server进程处理自己的工作
          return handleSystemServerProcess(parsedArgs);
      }

      return null;
  }


</code></pre> 
<p>ZygoteInit。forkSystemServer()会在新fork出的子进程中调用 handleSystemServerProcess()&#xff0c;</p> 
<p>主要是返回Runtime.java的MethodAndArgsCaller的方法&#xff0c;然后通过r.run() 启动com.android.server.SystemServer的main 方法</p> 
<p>这个当我们后面的SystemServer的章节进行详细讲解。</p> 
<pre class="has"><code class="language-java">handleSystemServerProcess代码流程&#xff1a;
handleSystemServerProcess()
    |
    [ZygoteInit.java]
    zygoteInit()
        |
    [RuntimeInit.java]
    applicationInit()
        |
    findStaticMain()
        |
    MethodAndArgsCaller()</code></pre> 
<p> </p> 
<h3>4.2.5 [ZygoteServer.java] runSelectLoop()</h3> 
<p><strong>代码路径: </strong>frameworks\base\core\java\com\android\internal\os\ZygoteServer.java</p> 
<pre class="has"><code class="language-java"> Runnable runSelectLoop(String abiList) {
        ArrayList&lt;FileDescriptor&gt; socketFDs &#61; new ArrayList&lt;FileDescriptor&gt;();
        ArrayList&lt;ZygoteConnection&gt; peers &#61; new ArrayList&lt;ZygoteConnection&gt;();

        // 首先将server socket加入到fds
        socketFDs.add(mZygoteSocket.getFileDescriptor());
        peers.add(null);

        while (true) {
            fetchUsapPoolPolicyPropsWithMinInterval();

            int[] usapPipeFDs &##61; null;
            StructPollfd[] pollFDs &#61; null;

            // 每次循环&#xff0c;都重新创建需要监听的pollFds
            // Allocate enough space for the poll structs, taking into account
            // the state of the USAP pool for this Zygote (could be a
            // regular Zygote, a WebView Zygote, or an AppZygote).
            if (mUsapPoolEnabled) {
                usapPipeFDs &#61; Zygote.getUsapPipeFDs();
                pollFDs &#61; new StructPollfd[socketFDs.size() &#43; 1 &#43; usapPipeFDs.
length];
            } else {
                pollFDs &#61; new StructPollfd[socketFDs.size()];
            }

            /*
             * For reasons of correctness the USAP pool pipe and event FDs
             * must be processed before the session and server sockets.  This
             * is to ensure that the USAP pool accounting information is
             * accurate when handling other requests like API blacklist
             * exemptions.
             */

            int pollIndex &#61; 0;
            for (FileDescriptor socketFD : socketFDs) {
                 // 关注事件到来
                pollFDs[pollIndex] &#61; new StructPollfd();
                pollFDs[pollIndex].fd &#61; socketFD;
                pollFDs[pollIndex].events &#61; (short) POLLIN;
                &#43;&#43;pollIndex;
            }

            final int usapPoolEventFDIndex &#61; pollIndex;

            if (mUsapPoolEnabled) {
                pollFDs[pollIndex] &#61; new StructPollfd();
                pollFDs[pollIndex].fd &#61; mUsapPoolEventFD;
                pollFDs[pollIndex].events &#61; (short) POLLIN;
                &#43;&#43;pollIndex;

                for (int usapPipeFD : usapPipeFDs) {
                    FileDescriptor managedFd &#61; new FileDescriptor();
                    managedFd.setInt$(usapPipeFD);

                    pollFDs[pollIndex] &#61; new StructPollfd();
                    pollFDs[pollIndex].fd &#61; managedFd;
                    pollFDs[pollIndex].events &#61; (short) POLLIN;
                    &#43;&#43;pollIndex;
                }
            }

            try {
                // 等待事件到来
                Os.poll(pollFDs, -1);
            } catch (ErrnoException ex) {
                throw new RuntimeException(&#34;poll failed&#34;, ex);
            }

            boolean usapPoolFDRead &#61; false;

            //倒序处理&#xff0c;即优先处理已建立链接的信息&#xff0c;后处理新建链接的请求
            while (--pollIndex &gt;&#61; 0) {
                if ((pollFDs[pollIndex].revents &amp; POLLIN) &#61;&#61; 0) {
                    continue;
                }

                // server socket最先加入fds&#xff0c; 因此这里是server socket收到数据
                if (pollIndex &#61;&#61; 0) {
                    // 收到新的建立通信的请求&#xff0c;建立通信连接
                    ZygoteConnection newPeer &#61; acceptCommandPeer(abiList);
                    // 加入到peers和fds, 即下一次也开始监听
                    peers.add(newPeer);
                    socketFDs.add(newPeer.getFileDescriptor());

                } else if (pollIndex &lt; usapPoolEventFDIndex) {
                    //说明接收到AMS发送过来创建应用程序的请求&#xff0c;调用
processOneCommand
                    //来创建新的应用程序进程
                    // Session socket accepted from the Zygote server socket
                    try {
                        //有socket连接&#xff0c;创建ZygoteConnection对象,并添加到fds。
                        ZygoteConnection connection &#61; peers.get(pollIndex);
                        //处理连接&#xff0c;参考[4.2.6]
                        final Runnable command &#61; connection.processOneCommand(
this);

                        // TODO (chriswailes): Is this extra check necessary?
                        if (mIsForkChild) {
                            // We&#39;re in the child. We should always have a 
command to run at this
                            // stage if processOneCommand hasn&#39;t called &#34;exec&#34;.
                            if (command &#61;&#61; null) {
                                throw new IllegalStateException(&#34;command &#61;&#61; 
null&#34;);
                            }

                            return command;
                        } else {
                            // We&#39;re in the server - we should never have any 
commands to run.
                            if (command !&#61; null) {
                                throw new IllegalStateException(&#34;command !&#61; 
null&#34;);
                            }

                            // We don&#39;t know whether the remote side of the 
socket was closed or
                            // not until we attempt to read from it from 
processOneCommand. This
                            // shows up as a regular POLLIN event in our 
regular processing loop.
                            if (connection.isClosedByPeer()) {
                                connection.closeSocket();
                                peers.remove(pollIndex);
                                socketFDs.remove(pollIndex);    //处理完则从
fds中移除该文件描述符
                            }
                        }
                    } catch (Exception e) {
                        ......
                    } finally {
                        mIsForkChild &#61; false;
                    }
                } else {
                    //处理USAP pool的事件
                    // Either the USAP pool event FD or a USAP reporting pipe.

                    // If this is the event FD the payload will be the number 
of USAPs removed.
                    // If this is a reporting pipe FD the payload will be the 
PID of the USAP
                    // that was just specialized.
                    long messagePayload &#61; -1;

                    try {
                        byte[] buffer &#61; new byte[Zygote.
USAP_MANAGEMENT_MESSAGE_BYTES];
                        int readBytes &#61; Os.read(pollFDs[pollIndex].fd, buffer
, 0, buffer.length);

                        if (readBytes &#61;&#61; Zygote.USAP_MANAGEMENT_MESSAGE_BYTES
) {
                            DataInputStream inputStream &#61;
                                    new DataInputStream(new 
ByteArrayInputStream(buffer));

                            messagePayload &#61; inputStream.readLong();
                        } else {
                            Log.e(TAG, &#34;Incomplete read from USAP management 
FD of size &#34;
                                    &#43; readBytes);
                            continue;
                        }
                    } catch (Exception ex) {
                        if (pollIndex &#61;&#61; usapPoolEventFDIndex) {
                            Log.e(TAG, &#34;Failed to read from USAP pool event FD
: &#34;
                                    &#43; ex.getMessage());
                        } else {
                            Log.e(TAG, &#34;Failed to read from USAP reporting 
pipe: &#34;
                                    &#43; ex.getMessage());
                        }

                        continue;
                    }

                    if (pollIndex &gt; usapPoolEventFDIndex) {
                        Zygote.removeUsapTableEntry((int) messagePayload);
                    }

                    usapPoolFDRead &#61; true;
                }
            }

            // Check to see if the USAP pool needs to be refilled.
            if (usapPoolFDRead) {
                int[] sessionSocketRawFDs &#61;
                        socketFDs.subList(1, socketFDs.size())
                                .stream()
                                .mapToInt(fd -&gt; fd.getInt$())
                                .toArray();

                final Runnable command &#61; fillUsapPool(sessionSocketRawFDs);

                if (command !&#61; null) {
                    return command;
                }
            }
        }
}
</code></pre> 
<p> </p> 
<h3>4.2.6 [ZygoteConnection.java] processOneCommand()</h3> 
<pre class="has"><code class="language-java">Runnable processOneCommand(ZygoteServer zygoteServer) {
    ...
    //fork子进程
    pid &#61; Zygote.forkAndSpecialize(parsedArgs.mUid, parsedArgs.mGid, 
parsedArgs.mGids,
            parsedArgs.mRuntimeFlags, rlimits, parsedArgs.mMountExternal, 
parsedArgs.mSeInfo,
            parsedArgs.mNiceName, fdsToClose, fdsToIgnore, parsedArgs.
mStartChildZygote,
            parsedArgs.mInstructionSet, parsedArgs.mAppDataDir, parsedArgs
.mTargetSdkVersion);
    if (pid &#61;&#61; 0) {
        // 子进程执行
        zygoteServer.setForkChild();
        //进入子进程流程&#xff0c;参考[4.2.7]
        return handleChildProc(parsedArgs, descriptors, childPipeFd,
                parsedArgs.mStartChildZygote);
    } else {
        //父进程执行
        // In the parent. A pid &lt; 0 indicates a failure and will be handled in 
        //handleParentProc.
        handleParentProc(pid, descriptors, serverPipeFd);
        return null;
    }
    ...
}

</code></pre> 
<p> </p> 
<h3>4.2.7 [ZygoteConnection.java] handleChildProc()</h3> 
<pre class="has"><code class="language-java">private Runnable handleChildProc(ZygoteArguments parsedArgs, FileDescriptor[] descriptors, FileDescriptor pipeFd, boolean isZygote) {
      ...
      if (parsedArgs.mInvokeWith !&#61; null) {
          ...
          throw new IllegalStateException(&#34;WrapperInit.execApplication 
unexpectedly returned&#34;);
      } else {
          if (!isZygote) {
              // App进程将会调用到这里&#xff0c;执行目标类的main()方法
              return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                      parsedArgs.mRemainingArgs, null /* classLoader */);
          } else {
              return ZygoteInit.childZygoteInit(parsedArgs.mTargetSdkVersion,
                      parsedArgs.mRemainingArgs, null /* classLoader */);
          }
      }
  }

</code></pre> 
<p> </p> 
<h1>5.问题分析</h1> 
<h2>5.1 为什么SystemServer和Zygote之间通信要采用Socket</h2> 
<p>进程间通信我们常用的是binder&#xff0c;为什么这里要采用socket呢。<br /> 主要是为了解决fork的问题&#xff1a;</p> 
<p><br /><span style="color:#f33b45;"><strong>UNIX上C&#43;&#43;程序设计守则3:多线程程序里不准使用fork</strong></span><br /> Binder通讯是需要多线程操作的&#xff0c;代理对象对Binder的调用是在Binder线程&#xff0c;需要再通过Handler调用主线程来操作。<br /> 比如AMS与应用进程通讯&#xff0c;AMS的本地代理IApplicationThread通过调用ScheduleLaunchActivity&#xff0c;调用到的应用进程ApplicationThread的ScheduleLaunchActivity是在Binder线程&#xff0c;<br /> 需要再把参数封装为一个ActivityClientRecord&#xff0c;sendMessage发送给H类&#xff08;主线程Handler&#xff0c;ActivityThread内部类&#xff09;<br /> 主要原因&#xff1a;害怕父进程binder线程有锁&#xff0c;然后子进程的主线程一直在等其子线程(从父进程拷贝过来的子进程)的资源&#xff0c;但是其实父进程的子进程并没有被拷贝过来&#xff0c;造成死锁。</p> 
<p><br /> 所以fork不允许存在多线程。而非常巧的是Binder通讯偏偏就是多线程&#xff0c;所以干脆父进程&#xff08;Zgote&#xff09;这个时候就不使用binder线程</p> 
<p> </p> 
<h2>5.2为什么一个java应用一个虚拟机&#xff1f;</h2> 
<ol><li>android的VM(vm&#61;&#61;Virtual Machine )也是类似JRE的东西&#xff0c;当然&#xff0c;各方面都截然不同&#xff0c;不过有一个作用都是一样的&#xff0c;为app提供了运行环境</li><li>android为每个程序提供一个vm&#xff0c;可以使每个app都运行在独立的运行环境&#xff0c;使稳定性提高。</li><li>vm的设计可以有更好的兼容性。android apk都被编译成字节码(bytecode)&#xff0c;在运行的时候&#xff0c;vm是先将字节码编译真正可执行的代码&#xff0c;否则不同硬件设备的兼容是很大的麻烦。</li><li>android&#xff08;非ROOT&#xff09;没有windows下键盘钩子之类的东西&#xff0c;每个程序一个虚拟机&#xff0c;各个程序之间也不可以随意访问内存&#xff0c;所以此类木马病毒几乎没有。</li></ol>
<p> </p> 
<h2>5.3 什么是Zygote资源预加载</h2> 
<p style="text-indent:33px;">预加载是指在zygote进程启动的时候就加载&#xff0c;这样系统只在zygote执行一次加载操作&#xff0c;所有APP用到该资源不需要再重新加载&#xff0c;减少资源加载时间&#xff0c;加快了应用启动速度&#xff0c;一般情况下&#xff0c;系统中App共享的资源会被列为预加载资源。<br /> zygote fork子进程时&#xff0c;根据fork的copy-on-write机制可知&#xff0c;有些类如果不做改变&#xff0c;甚至都不用复制&#xff0c;子进程可以和父进程共享这部分数据&#xff0c;从而省去不少内存的占用。</p> 
<p> </p> 
<h2>5.4 Zygote为什么要预加载</h2> 
<p style="text-indent:33px;">应用程序都从Zygote孵化出来&#xff0c;应用程序都会继承Zygote的所有内容。<br />        如果在Zygote启动的时候加载这些类和资源&#xff0c;这些孵化的应用程序就继承Zygote的类和资源&#xff0c;这样启动引用程序的时候就不需要加载类和资源了&#xff0c;启动的速度就会快很多。<br />       开机的次数不多&#xff0c;但是启动应用程序的次数非常多。</p> 
<p> </p> 
<h2>5.5 Zygote 预加载的原理是什么&#xff1f;</h2> 
<p>zygote进程启动后将资源读取出来&#xff0c;保存到Resources一个全局静态变量中&#xff0c;下次读取系统资源的时候优先从静态变量中查找。</p> 
<p> </p> 
<p> </p> 
<h1>6.总结</h1> 
<p>至此&#xff0c;Zygote启动流程结束&#xff0c;Zygote进程共做了如下几件事&#xff1a;</p> 
<ol><li>解析init.zygote64_32.rc&#xff0c;创建AppRuntime并调用其start方法&#xff0c;启动Zygote进程。</li><li>创建JavaVM并为JavaVM注册JNI.</li><li>通过JNI调用ZygoteInit的main函数进入Zygote的Java框架层。</li><li>通过ZygoteServer创建服务端Socket&#xff0c;预加载类和资源&#xff0c;并通过runSelectLoop函数等待如ActivityManagerService等的请求。</li><li>启动SystemServer进程。</li></ol>
<p> </p> 