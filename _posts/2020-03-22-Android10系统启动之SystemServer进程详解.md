---
layout:     post
title:      Android10系统启动之SystemServer进程详解
subtitle:   这篇文章我们来详细学习下Android10系统启动中SystemServer进程的启动过程
date:       2020-03-22
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android10
    - 系统启动
    - SystemServer
--- 

<h1>1.概述</h1> 
<p style="text-indent:33px;">上一节讲解了Zygote进程的整个启动流程。Zygote是所有应用的鼻祖。SystemServer和其他所有Dalivik虚拟机进程都是由Zygote fork而来。Zygote fork的第一个进程就是SystemServer&#xff0c;其在手机中的进程名为 system_server。</p> 
<p style="text-indent:33px;">system_server 进程承载着整个framework的核心服务&#xff0c;例如创建 ActivityManagerService、PowerManagerService、DisplayManagerService、PackageManagerService、WindowManagerService、LauncherAppsService等80多个核心系统服务。这些服务以不同的线程方式存在于system_server这个进程中。</p> 
<p style="text-indent:33px;">接下来&#xff0c;就让我们透过Android系统源码一起来分析一下SystemServer的整个启动过程。</p> 
<p> </p> 
<h1>2.核心源码</h1> 
<pre class="has"><code class="language-cpp">/frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
/frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
/frameworks/base/core/java/com/android/internal/os/Zygote.java

/frameworks/base/services/java/com/android/server/SystemServer.java
/frameworks/base/services/core/java/com/android/serverSystemServiceManager.java
/frameworks/base/services/core/java/com/android/ServiceThread.java
/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

/frameworks/base/core/java/android/app/ActivityThread.java
/frameworks/base/core/java/android/app/LoadedApk.java
/frameworks/base/core/java/android/app/ContextImpl.java

/frameworks/base/core/jni/AndroidRuntime.cpp
/frameworks/base/core/jni/com_android_internal_os_ZygoteInit.cpp
/frameworks/base/cmds/app_process/app_main.cpp

</code></pre> 
<h1>3.架构</h1> 
<h2>3.1 架构图</h2> 
<p>    SystemServer 被Zygote进程fork出来后&#xff0c;用来创建ActivityManagerService、PowerManagerService、DisplayManagerService、PackageManagerService、WindowManagerService、LauncherAppsService等90多个核心系统服务</p> 
<p> <img alt="" class="has" height="230" src="https://img-blog.csdnimg.cn/fb78b52cb2834ac2ac85d9752a036547.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_9,color_FFFFFF,t_70,g_se,x_16" width="360" /></p> 
<h2>3.2 服务启动</h2> 
<p> SystemServer思维导图</p> 
<p><img alt="" class="has" height="592" src="https://img-blog.csdnimg.cn/b26fbec953cb40e4afebc2370ddca17c.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_18,color_FFFFFF,t_70,g_se,x_16" width="724" /></p> 
<h1>4. 源码分析</h1> 
<h2>4.1 SystemServer fork流程分析</h2> 
<p> <img alt="" class="has" height="1177" src="https://img-blog.csdnimg.cn/20191215164941628.png?x-oss-process&#61,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpcmFuZmVuZw&#61;&#61;,size_16,color_FFFFFF,t_70" width="1200" /></p> 
<h3>4.1.1 [ZygoteInit.java] main()</h3> 
<p><strong>说明&#xff1a;</strong>Zygote进程&#xff0c;通过fork()函数&#xff0c;最终孵化出system_server的进程&#xff0c;通过反射的方法启动</p> 
<p>SystemServer.java的main()方法</p> 
<p><strong>源码&#xff1a;</strong></p> 
<pre class="has"><code>public static void main(String argv[]) {
	ZygoteServer zygoteServer &#61; null;
    ...
	try {
		zygoteServer &#61; new ZygoteServer(isPrimaryZygote);
		if (startSystemServer) {
			//fork system_server
			Runnable r &#61; forkSystemServer(abiList, zygoteSocketName, zygoteServer);

			// {&#64;code r &#61;&#61; null} in the parent (zygote) process, and {&#64;code r !&#61; null} in the
			// child (system_server) process.
			if (r !&#61; null) {
				r.run(); //启动SystemServer.java的main()
				return; //Android 8.0之前是通过抛异常的方式来启动&#xff0c;这里是直接return出去&#xff0c;用来清空栈&#xff0c;提高栈帧利用率
			}
		}
        caller &#61; zygoteServer.runSelectLoop(abiList);
    } catch (Throwable ex) {
        Log.e(TAG, &#34;System zygote died with exception&#34;, ex);
        throw ex;
    } finally {
        if (zygoteServer !&#61; null) {
            zygoteServer.closeServerSocket();
        }
    }
	if (caller !&#61; null) {
        caller.run();
    }
	...
}</code></pre> 
<p> </p> 
<h3>4.1.2[ZygoteInit.java] forkSystemServer()</h3> 
<p><strong>说明&#xff1a;</strong>准备参数&#xff0c;用来进行system_server的fork&#xff0c;从参数可知&#xff0c;pid&#61;1000&#xff0c;gid&#61;1000&#xff0c;进程名nick-name&#61;system_server</p> 
<p>当有两个Zygote进程时&#xff0c;需要等待第二个Zygote创建完成。由于fork时会拷贝socket&#xff0c;因此&#xff0c;在fork出system_server进程后&#xff0c;</p> 
<p>需要关闭Zygote原有的socket</p> 
<p><strong>源码&#xff1a;</strong></p> 
<pre class="has"><code class="language-java">private static Runnable forkSystemServer(String abiList, String socketName,
        ZygoteServer zygoteServer) {
    ......
    //参数准备&#xff0c;uid和gid都是为1000
    String args[] &#61; {
            &#34;--setuid&#61;1000&#34;,
            &#34;--setgid&#61;1000&#34;,
            &#34;--setgroups&#61;1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,&#34;
                    &#43; &#34;1024,1032,1065,3001,3002,3003,3006,3007,3009,3010&#34;,
            &#34;--capabilities&#61;&#34; &#43; capabilities &#43; &#34;,&#34; &#43; capabilities,
            &#34;--nice-name&#61;system_server&#34;,
            &#34;--runtime-args&#34;,
            &#34;--target-sdk-version&#61;&#34; &#43; VMRuntime.SDK_VERSION_CUR_DEVELOPMENT,
            &#34;com.android.server.SystemServer&#34;,
    };
    ZygoteArguments parsedArgs &#61; null;
    int pid;
    try {
        //将上面准备的参数&#xff0c;按照ZygoteArguments的风格进行封装
        parsedArgs &#61; new ZygoteArguments(args);
        Zygote.applyDebuggerSystemProperty(parsedArgs);
        Zygote.applyInvokeWithSystemProperty(parsedArgs);

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
    if (pid &#61;&#61; 0) {
        // 处理32_64和64_32的情况
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);  //需要等待第二个Zygote创建完成
        }

        // fork时会copy socket&#xff0c;Zygote原有的socket需要关闭
        zygoteServer.closeServerSocket();
        // system server进程处理自己的工作
        return handleSystemServerProcess(parsedArgs);
    }
    return null;
}</code></pre> 
<h3>4.1.3 [Zygote.java] forkSystemServer()</h3> 
<p><strong>说明&#xff1a;</strong>这里的nativeForkSystemServer()最终是通过JNI&#xff0c;调用Nativate C空间的com_android_internal_os_Zygote_nativeForkSystemServer()</p> 
<p>来fork system_server</p> 
<p><strong>源码&#xff1a;</strong></p> 
<p> </p> 
<pre class="has"><code class="language-java">public static int forkSystemServer(int uid, int gid, int[] gids, int runtimeFlags,
        int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) {
    ZygoteHooks.preFork();
    // Resets nice priority for zygote process.
    resetNicePriority();
    //调用native的方法来fork system_server
    //最终调用native的方法:com_android_internal_os_Zygote_nativeForkSystemServer
    int pid &#61; nativeForkSystemServer(
            uid, gid, gids, runtimeFlags, rlimits,
            permittedCapabilities, effectiveCapabilities);
    // Enable tracing as soon as we enter the system_server.
    if (pid &#61;&#61; 0) {
        Trace.setTracingEnabled(true, runtimeFlags);
    }
    ZygoteHooks.postForkCommon();
    return pid;
}</code></pre> 
<p><strong>[com_android_internal_os_Zygote.cpp]</strong></p> 
<p><strong>说明&#xff1a;</strong>JNI注册的映射关系</p> 
<pre class="has"><code class="language-java">static const JNINativeMethod gMethods[] &#61; {
    { &#34;nativeForkSystemServer&#34;, &#34;(II[II[[IJJ)I&#34;,
      (void *) com_android_internal_os_Zygote_nativeForkSystemServer },
}</code></pre> 
<p> </p> 
<h3>4.1.4[com_android_internal_os_Zygote.cpp]</h3> 
<p><strong>com_android_internal_os_Zygote_nativeForkSystemServer()</strong></p> 
<p><strong>说明&#xff1a;</strong>通过 SpecializeCommon进行fork&#xff0c;pid返回0时&#xff0c;表示当前为system_server子进程</p> 
<p>当pid &gt;0 时&#xff0c;是进入父进程&#xff0c;即Zygote进程&#xff0c;通过waitpid 的WNOHANG 非阻塞方式来监控</p> 
<p>system_server进程挂掉&#xff0c;如果挂掉后重启Zygote进程。</p> 
<p>现在使用的Android系统大部分情况下是64位的&#xff0c;会存在两个Zygote&#xff0c;当system_server挂掉后&#xff0c;</p> 
<p>只启动Zygote64这个父进程</p> 
<p><strong>源码&#xff1a;</strong></p> 
<pre class="has"><code class="language-java">static jint com_android_internal_os_Zygote_nativeForkSystemServer(
        JNIEnv* env, jclass, uid_t uid, gid_t gid, jintArray gids,
        jint runtime_flags, jobjectArray rlimits, jlong permitted_capabilities,
        jlong effective_capabilities) {


  pid_t pid &#61; ForkCommon(env, true,
                         fds_to_close,
                         fds_to_ignore);
  if (pid &#61;&#61; 0) {
      //进入子进程
      SpecializeCommon(env, uid, gid, gids, runtime_flags, rlimits,
                       permitted_capabilities, effective_capabilities,
                       MOUNT_EXTERNAL_DEFAULT, nullptr, nullptr, true,
                       false, nullptr, nullptr);
  } else if (pid &gt; 0) {
      //进入父进程&#xff0c;即zygote进程
      ALOGI(&#34;System server process %d has been created&#34;, pid);

      int status;
	  //用waitpid函数获取状态发生变化的子进程pid
	  //waitpid的标记为WNOHANG&#xff0c;即非阻塞&#xff0c;返回为正值就说明有进程挂掉了
      if (waitpid(pid, &amp;status, WNOHANG) &#61;&#61; pid) {
		  //当system_server进程死亡后&#xff0c;重启zygote进程
          ALOGE(&#34;System server process %d has died. Restarting Zygote!&#34;, pid);
          RuntimeAbort(env, __LINE__, &#34;System server process has died. Restarting Zygote!&#34;);
      }
	  ...
  }
  return pid;
}</code></pre> 
<h3><strong>4.1.5 [com_android_internal_os_Zygote.cpp] ForkCommon</strong></h3> 
<p><strong>说明&#xff1a;</strong>从Zygote孵化出一个进程的使用程序</p> 
<p><strong>源码&#xff1a;</strong></p> 
<p> </p> 
<pre class="has"><code class="language-java">static pid_t ForkCommon(JNIEnv* env, bool is_system_server,
                        const std::vector&lt;int&gt;&amp; fds_to_close,
                        const std::vector&lt;int&gt;&amp; fds_to_ignore) {
  //设置子进程的signal
  SetSignalHandlers();

  //在fork的过程中&#xff0c;临时锁住SIGCHLD
  BlockSignal(SIGCHLD, fail_fn);

  //fork子进程,采用copy on write方式&#xff0c;这里执行一次&#xff0c;会返回两次
  //pid&#61;0 表示Zygote  fork SystemServer这个子进程成功
  //pid &gt; 0 表示SystemServer 的真正的PID
  pid_t pid &#61; fork();

  if (pid &#61;&#61; 0) {
     //进入子进程
    // The child process.
    PreApplicationInit();

    // 关闭并清除文件描述符
    // Clean up any descriptors which must be closed immediately
    DetachDescriptors(env, fds_to_close, fail_fn);
	...
  } else {
    ALOGD(&#34;Forked child process %d&#34;, pid);
  }

  //fork结束&#xff0c;解锁
  UnblockSignal(SIGCHLD, fail_fn);

  return pid;
}</code></pre> 
<h3>4.1.6 [Zygcom_android_internal_os_Zygoteote.cpp] SpecializeCommon</h3> 
<p><strong>说明&#xff1a;</strong>system_server进程的一些调度配置</p> 
<p><strong>源码&#xff1a;</strong></p> 
<p> </p> 
<pre class="has"><code class="language-java">static void SpecializeCommon(JNIEnv* env, uid_t uid, gid_t gid, jintArray gids,
                             jint runtime_flags, jobjectArray rlimits,
                             jlong permitted_capabilities, jlong effective_capabilities,
                             jint mount_external, jstring managed_se_info,
                             jstring managed_nice_name, bool is_system_server,
                             bool is_child_zygote, jstring managed_instruction_set,
                             jstring managed_app_data_dir) {
  ...
  bool use_native_bridge &#61; !is_system_server &amp;&amp;
                           instruction_set.has_value() &amp;&amp;
                           android::NativeBridgeAvailable() &amp;&amp;
                           android::NeedsNativeBridge(instruction_set.value().c_str());

  if (!is_system_server &amp;&amp; getuid() &#61;&#61; 0) {
    //对于非system_server子进程&#xff0c;则创建进程组
    const int rc &#61; createProcessGroup(uid, getpid());
    if (rc &#61;&#61; -EROFS) {
      ALOGW(&#34;createProcessGroup failed, kernel missing CONFIG_CGROUP_CPUACCT?&#34;);
    } else if (rc !&#61; 0) {
      ALOGE(&#34;createProcessGroup(%d, %d) failed: %s&#34;, uid, /* pid&#61; */ 0, strerror(-rc));
    }
  }

  SetGids(env, gids, fail_fn);  //设置设置group
  SetRLimits(env, rlimits, fail_fn); //设置资源limit

  if (use_native_bridge) {
    // Due to the logic behind use_native_bridge we know that both app_data_dir
    // and instruction_set contain values.
    android::PreInitializeNativeBridge(app_data_dir.value().c_str(),
                                       instruction_set.value().c_str());
  }

  if (setresgid(gid, gid, gid) &#61;&#61; -1) {
    fail_fn(CREATE_ERROR(&#34;setresgid(%d) failed: %s&#34;, gid, strerror(errno)));
  }
   ...
   //selinux上下文
  if (selinux_android_setcontext(uid, is_system_server, se_info_ptr, nice_name_ptr) &#61;&#61; -1) {
    fail_fn(CREATE_ERROR(&#34;selinux_android_setcontext(%d, %d, \&#34;%s\&#34;, \&#34;%s\&#34;) failed&#34;,
                         uid, is_system_server, se_info_ptr, nice_name_ptr));
  }

  //设置线程名为system_server&#xff0c;方便调试
  if (nice_name.has_value()) {
    SetThreadName(nice_name.value());
  } else if (is_system_server) {
    SetThreadName(&#34;system_server&#34;);
  }

  // Unset the SIGCHLD handler, but keep ignoring SIGHUP (rationale in SetSignalHandlers).
  //设置子进程的signal信号处理函数为默认函数
  UnsetChldSignalHandler();

  if (is_system_server) {
   
    //对应  Zygote.java 的callPostForkSystemServerHooks()
    env-&gt;CallStaticVoidMethod(gZygoteClass, gCallPostForkSystemServerHooks);
    if (env-&gt;ExceptionCheck()) {
      fail_fn(&#34;Error calling post fork system server hooks.&#34;);
    }

	//对应ZygoteInit.java 的 createSystemServerClassLoader()
	//预取系统服务器的类加载器。这样做是为了尽早地绑定适当的系统服务器selinux域。
    env-&gt;CallStaticVoidMethod(gZygoteInitClass, gCreateSystemServerClassLoader);
    if (env-&gt;ExceptionCheck()) {
      // Be robust here. The Java code will attempt to create the classloader
      // at a later point (but may not have rights to use AoT artifacts).
      env-&gt;ExceptionClear();
    }
	...
  }

   //等价于调用zygote.java 的callPostForkChildHooks()
  env-&gt;CallStaticVoidMethod(gZygoteClass, gCallPostForkChildHooks, runtime_flags,
                            is_system_server, is_child_zygote, managed_instruction_set);

  if (env-&gt;ExceptionCheck()) {
    fail_fn(&#34;Error calling post fork hooks.&#34;);
  }
}</code></pre> 
<h3>4.1.7 [ZygoteInit.java] handleSystemServerProcess</h3> 
<p><strong>说明&#xff1a;</strong>创建类加载器&#xff0c;并赋予当前线程&#xff0c;其中环境变量SYSTEMSERVERCLASSPATH&#xff0c;主要是<strong>service.jar、ethernet-service.jar和wifi-service.jar</strong>这三个jar包</p> 
<pre class="has"><code>export SYSTEMSERVERCLASSPATH&#61;/system/framework/services.jar:/system/framework/ethernet-service.jar:/system/framework/wifi-service.jar</code></pre> 
<p><strong>源码&#xff1a;</strong></p> 
<p> </p> 
<pre class="has"><code class="language-java">private static Runnable handleSystemServerProcess(ZygoteArguments parsedArgs) {

    if (parsedArgs.mNiceName !&#61; null) {
        Process.setArgV0(parsedArgs.mNiceName); //设置当前进程名为&#34;system_server&#34;
    }

    final String systemServerClasspath &#61; Os.getenv(&#34;SYSTEMSERVERCLASSPATH&#34;);
    if (systemServerClasspath !&#61; null) {
       //执行dex优化操作
        if (performSystemServerDexOpt(systemServerClasspath)) {
            sCachedSystemServerClassLoader &#61; null;
        }
		...
    }

    if (parsedArgs.mInvokeWith !&#61; null) {
        String[] args &#61; parsedArgs.mRemainingArgs;
		//如果我们有一个非空系统服务器类路径&#xff0c;我们将不得不复制现有的参数并将类路径附加到它。
		//当我们执行一个新进程时&#xff0c;ART将正确地处理类路径。
        if (systemServerClasspath !&#61; null) {
            String[] amendedArgs &#61; new String[args.length &#43; 2];
            amendedArgs[0] &#61; &#34;-cp&#34;;
            amendedArgs[1] &#61; systemServerClasspath;
            System.arraycopy(args, 0, amendedArgs, 2, args.length);
            args &#61; amendedArgs;
        }

        //启动应用进程
        WrapperInit.execApplication(parsedArgs.mInvokeWith,
                parsedArgs.mNiceName, parsedArgs.mTargetSdkVersion,
                VMRuntime.getCurrentInstructionSet(), null, args);

        throw new IllegalStateException(&#34;Unexpected return from WrapperInit.execApplication&#34;);
    } else {
        // 创建类加载器&#xff0c;并赋予当前线程
        createSystemServerClassLoader();
        ClassLoader cl &#61; sCachedSystemServerClassLoader;
        if (cl !&#61; null) {
            Thread.currentThread().setContextClassLoader(cl);
        }

        //system_server进入此分支
        return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                parsedArgs.mRemainingArgs, cl);
    }
}
</code></pre> 
<h3>4.1.8[ZygoteInit.java] zygoteInit</h3> 
<p><strong>说明&#xff1a;</strong>基础配置&#xff0c;并进行应用初始化&#xff0c;返回对象</p> 
<p><strong>源码&#xff1a;</strong></p> 
<pre class="has"><code class="language-java">public static final Runnable zygoteInit(int targetSdkVersion, String[] argv,
        ClassLoader classLoader) {

    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, &#34;ZygoteInit&#34;);
    RuntimeInit.redirectLogStreams();  //重定向log输出

    RuntimeInit.commonInit(); //通用的一些初始化
    ZygoteInit.nativeZygoteInit(); // zygote初始化
    // 应用初始化
    return RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
}</code></pre> 
<h3>4.1.9[RuntimeInit.java] commonInit</h3> 
<p><strong>说明&#xff1a;</strong>配置log、时区、http userAgent等基础信息</p> 
<p><strong>源码&#xff1a;</strong></p> 
<p> </p> 
<pre class="has"><code class="language-java">protected static final void commonInit() {
	LoggingHandler loggingHandler &#61; new LoggingHandler();
    
    // 设置默认的未捕捉异常处理方法
    RuntimeHooks.setUncaughtExceptionPreHandler(loggingHandler);
    Thread.setDefaultUncaughtExceptionHandler(new KillApplicationHandler(loggingHandler));

    // 设置时区&#xff0c;通过属性读出中国时区为&#34;Asia/Shanghai&#34;
    RuntimeHooks.setTimeZoneIdSupplier(() -&gt; SystemProperties.get(&#34;persist.sys.timezone&#34;));

    //重置log配置
    LogManager.getLogManager().reset();
    new AndroidConfig();

    //设置默认的HTTP User-agent格式,用于 HttpURLConnection
    String userAgent &#61; getDefaultUserAgent();
    System.setProperty(&#34;http.agent&#34;, userAgent);

    /*
     * Wire socket tagging to traffic stats.
     */
    //设置socket的tag&#xff0c;用于网络流量统计
    NetworkManagementSocketTagger.install();
	...
}</code></pre> 
<h3>4.1.10[ZygoteInit.java] nativeZygoteInit</h3> 
<p><strong>说明&#xff1a;</strong>nativeZygoteInit 通过反射&#xff0c;进入com_android_internal_os_ZygoteInit_nativeZygoteInit</p> 
<p><strong>源码&#xff1a;</strong></p> 
<p><strong>[AndroidRuntime.cpp]</strong></p> 
<pre class="has"><code class="language-java">int register_com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env)
{
    const JNINativeMethod methods[] &#61; {
        { &#34;nativeZygoteInit&#34;, &#34;()V&#34;,
            (void*) com_android_internal_os_ZygoteInit_nativeZygoteInit },
    };
    return jniRegisterNativeMethods(env, &#34;com/android/internal/os/ZygoteInit&#34;,
        methods, NELEM(methods));
}

gCurRuntime &#61; this;
static void com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    //此处的gCurRuntime为AppRuntime&#xff0c;是在AndroidRuntime.cpp中定义的
    gCurRuntime-&gt;onZygoteInit();
}

[app_main.cpp]
virtual void onZygoteInit()
{
    sp&lt;ProcessState&gt; proc &#61; ProcessState::self();
    ALOGV(&#34;App process: starting thread pool.\n&#34;);
    proc-&gt;startThreadPool(); //启动新binder线程
}</code></pre> 
<p> </p> 
<h3>4.1.11[RuntimeInit.java] applicationInit</h3> 
<p><strong>说明&#xff1a;</strong>通过参数解析&#xff0c;得到args.startClass &#61; com.android.server.SystemServer</p> 
<p><strong>源码&#xff1a;</strong></p> 
<pre class="has"><code>protected static Runnable applicationInit(int targetSdkVersion, String[] argv,
        ClassLoader classLoader) {

    //true代表应用程序退出时不调用AppRuntime.onExit()&#xff0c;否则会在退出前调用
    nativeSetExitWithoutCleanup(true);

    // We want to be fairly aggressive about heap utilization, to avoid
    // holding on to a lot of memory that isn&#39;t needed.
    
    //设置虚拟机的内存利用率参数值为0.75
    VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
    VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);

    final Arguments args &#61; new Arguments(argv);  //解析参数
	...
    // Remaining arguments are passed to the start class&#39;s static main
    //调用startClass的static方法 main() 
    return findStaticMain(args.startClass, args.startArgs, classLoader);
}</code></pre> 
<h3>4.1.12 [RuntimeInit.java] findStaticMain</h3> 
<p><strong>说明&#xff1a;</strong>拿到SystemServer的main()方法&#xff0c;并返回 MethodAndArgsCaller()对象</p> 
<p><strong>源码&#xff1a;</strong></p> 
<p> </p> 
<pre class="has"><code class="language-java">protected static Runnable findStaticMain(String className, String[] argv,
        ClassLoader classLoader) {
    Class&lt;?&gt; cl;

    try {
		//拿到com.android.server.SystemServer 的类对象
        cl &#61; Class.forName(className, true, classLoader);
    } catch (ClassNotFoundException ex) {
        throw new RuntimeException(
                &#34;Missing class when invoking static main &#34; &#43; className,
                ex);
    }

    Method m;
    try {
        //得到SystemServer的main()方法&#xff0c;
        m &#61; cl.getMethod(&#34;main&#34;, new Class[] { String[].class });
    } catch (NoSuchMethodException ex) {
        throw new RuntimeException(
                &#34;Missing static main on &#34; &#43; className, ex);
    } catch (SecurityException ex) {
        throw new RuntimeException(
                &#34;Problem getting static main on &#34; &#43; className, ex);
    }

    int modifiers &#61; m.getModifiers();
    if (! (Modifier.isStatic(modifiers) &amp;&amp; Modifier.isPublic(modifiers))) {
        throw new RuntimeException(
                &#34;Main method is not public and static on &#34; &#43; className);
    }

    //把MethodAndArgsCaller的对象返回给ZygoteInit.main()。这样做好处是能清空栈帧&#xff0c;提高栈帧利用率
	//清除了设置进程所需的所有堆栈帧
    return new MethodAndArgsCaller(m, argv);
}</code></pre> 
<h3>4.1.13[RuntimeInit.java] MethodAndArgsCaller</h3> 
<p><strong>说明&#xff1a;</strong>最终在ZygoteInit.java的main()&#xff0c;调用这里的run()来启动SystemServer.java的main()&#xff0c;真正进入SystemServer进程</p> 
<p><strong>源码&#xff1a;</strong></p> 
<p> </p> 
<pre class="has"><code class="language-java">static class MethodAndArgsCaller implements Runnable {
    /** method to call */
    private final Method mMethod;

    /** argument array */
    private final String[] mArgs;

    public MethodAndArgsCaller(Method method, String[] args) {
        mMethod &#61; method;
        mArgs &#61; args;
    }

    public void run() {
        try {
          //根据传递过来的参数&#xff0c;可知此处通过反射机制调用的是SystemServer.main()方法
            mMethod.invoke(null, new Object[] { mArgs });
        } catch (IllegalAccessException ex) {
            throw new RuntimeException(ex);
        } catch (InvocationTargetException ex) {
            Throwable cause &#61; ex.getCause();
            if (cause instanceof RuntimeException) {
                throw (RuntimeException) cause;
            } else if (cause instanceof Error) {
                throw (Error) cause;
            }
            throw new RuntimeException(ex);
        }
    }
}</code></pre> 
<p>4.2 SystemServer 启动后的流程</p> 
<p> <img alt="" class="has" height="341" src="https://img-blog.csdnimg.cn/335b4d51f1d34da599afacf94b34288b.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_1" width="799" /></p> 
<h3> </h3> 
<h3>4.2.1[SystemServer.java] main</h3> 
<p><strong>说明&#xff1a;</strong>main函数由Zygote进程 fork后运行&#xff0c;作用是new 一个SystemServer对象&#xff0c;再调用该对象的run()方法</p> 
<p><strong>源码&#xff1a;</strong></p> 
<pre class="has"><code class="language-java">public static void main(String[] args) {
    //new 一个SystemServer对象&#xff0c;再调用该对象的run()方法
    new SystemServer().run();
}</code></pre> 
<h3>4.2.2 [SystemServer.java] run</h3> 
<p><strong>说明&#xff1a;</strong>先初始化一些系统变量&#xff0c;加载类库&#xff0c;创建Context对象&#xff0c;创建SystemServiceManager对象等候再启动服务,启动引导服务、核心服务和其他服务</p> 
<p><strong>源码&#xff1a;</strong></p> 
<p>  </p> 
<pre class="has"><code class="language-java">private void run() {
    try {
        traceBeginAndSlog(&#34;InitBeforeStartServices&#34;);

        // Record the process start information in sys props.
        //从属性中读取system_server进程的一些信息
        SystemProperties.set(SYSPROP_START_COUNT, String.valueOf(mStartCount));
        SystemProperties.set(SYSPROP_START_ELAPSED, String.valueOf(mRuntimeStartElapsedTime));
        SystemProperties.set(SYSPROP_START_UPTIME, String.valueOf(mRuntimeStartUptime));

        EventLog.writeEvent(EventLogTags.SYSTEM_SERVER_START,
                mStartCount, mRuntimeStartUptime, mRuntimeStartElapsedTime);


        //如果一个设备的时钟是在1970年之前(0年之前)&#xff0c;
        //那么很多api 都会因为处理负数而崩溃&#xff0c;尤其是java.io.File#setLastModified
        //我把把时间设置为1970
        if (System.currentTimeMillis() &lt; EARLIEST_SUPPORTED_TIME) {
            Slog.w(TAG, &#34;System clock is before 1970; setting to 1970.&#34;);
            SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
        }

        //如果时区不存在&#xff0c;设置时区为GMT
        String timezoneProperty &#61; SystemProperties.get(&#34;persist.sys.timezone&#34;);
        if (timezoneProperty &#61;&#61; null || timezoneProperty.isEmpty()) {
            Slog.w(TAG, &#34;Timezone not set; setting to GMT.&#34;);
            SystemProperties.set(&#34;persist.sys.timezone&#34;, &#34;GMT&#34;);
        }

        //变更虚拟机的库文件&#xff0c;对于Android 10.0默认采用的是libart.so
        SystemProperties.set(&#34;persist.sys.dalvik.vm.lib.2&#34;, VMRuntime.getRuntime().vmLibrary());

        // Mmmmmm... more memory!
        //清除vm内存增长上限&#xff0c;由于启动过程需要较多的虚拟机内存空间
        VMRuntime.getRuntime().clearGrowthLimit();
		...
        //系统服务器必须一直运行&#xff0c;所以它需要尽可能高效地使用内存
        //设置内存的可能有效使用率为0.8
        VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);


        //一些设备依赖于运行时指纹生成&#xff0c;所以在进一步启动之前&#xff0c;请确保我们已经定义了它。
        Build.ensureFingerprintProperty();

        //访问环境变量前&#xff0c;需要明确地指定用户
        //在system_server中&#xff0c;任何传入的包都应该被解除&#xff0c;以避免抛出BadParcelableException。
        BaseBundle.setShouldDefuse(true);

        //在system_server中&#xff0c;当打包异常时&#xff0c;信息需要包含堆栈跟踪
        Parcel.setStackTraceParceling(true);

        //确保当前系统进程的binder调用&#xff0c;总是运行在前台优先级(foreground priority)
        BinderInternal.disableBackgroundScheduling(true);

        //设置system_server中binder线程的最大数量,最大值为31
        BinderInternal.setMaxThreads(sMaxBinderThreads);

        //准备主线程lopper&#xff0c;即在当前线程运行
        android.os.Process.setThreadPriority(
                android.os.Process.THREAD_PRIORITY_FOREGROUND);
        android.os.Process.setCanSelfBackground(false);
        Looper.prepareMainLooper();
        Looper.getMainLooper().setSlowLogThresholdMs(
                SLOW_DISPATCH_THRESHOLD_MS, SLOW_DELIVERY_THRESHOLD_MS);

        //加载android_servers.so库&#xff0c;初始化native service
        System.loadLibrary(&#34;android_servers&#34;);

        // Debug builds - allow heap profiling.
        //如果是Debug版本&#xff0c;允许堆内存分析
        if (Build.IS_DEBUGGABLE) {
            initZygoteChildHeapProfiling();
        }

        //检测上次关机过程是否失败&#xff0c;这个调用可能不会返回
        performPendingShutdown();

        //初始化系统上下文
        createSystemContext();

        //创建系统服务管理--SystemServiceManager
        mSystemServiceManager &#61; new SystemServiceManager(mSystemContext);
        mSystemServiceManager.setStartInfo(mRuntimeRestart,
                mRuntimeStartElapsedTime, mRuntimeStartUptime);
        //将mSystemServiceManager添加到本地服务的成员sLocalServiceObjects
        LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
        // Prepare the thread pool for init tasks that can be parallelized
        //为可以并行化的init任务准备线程池
        SystemServerInitThreadPool.get();
    } finally {
        traceEnd();  // InitBeforeStartServices
    }

    // Start services.
    //启动服务
    try {
        traceBeginAndSlog(&#34;StartServices&#34;);
        startBootstrapServices();   // 启动引导服务
        startCoreServices();        // 启动核心服务
        startOtherServices();       // 启动其他服务
        SystemServerInitThreadPool.shutdown(); //停止线程池
    } catch (Throwable ex) {
        Slog.e(&#34;System&#34;, &#34;******************************************&#34;);
        Slog.e(&#34;System&#34;, &#34;************ Failure starting system services&#34;, ex);
        throw ex;
    } finally {
        traceEnd();
    }

    //为当前的虚拟机初始化VmPolicy
    StrictMode.initVmDefaults(null);
	...
    // Loop forever.
    //死循环执行
    Looper.loop();
    throw new RuntimeException(&#34;Main thread loop unexpectedly exited&#34;);
}
</code></pre> 
<h3>4.2.3[SystemServer.java] performPendingShutdown</h3> 
<p><strong>说明&#xff1a;</strong>检测上次关机过程是否失败&#xff0c;这个调用可能不会返回</p> 
<p><strong>源码&#xff1a;</strong></p> 
<p> </p> 
<pre class="has"><code class="language-java">private void performPendingShutdown() {
    final String shutdownAction &#61; SystemProperties.get(
            ShutdownThread.SHUTDOWN_ACTION_PROPERTY, &#34;&#34;);
    if (shutdownAction !&#61; null &amp;&amp; shutdownAction.length() &gt; 0) {
        boolean reboot &#61; (shutdownAction.charAt(0) &#61;&#61; &#39;1&#39;);

        final String reason;
        if (shutdownAction.length() &gt; 1) {
            reason &#61; shutdownAction.substring(1, shutdownAction.length());
        } else {
            reason &#61; null;
        }

        //如果需要重新启动才能应用更新&#xff0c;一定要确保uncrypt在需要时正确执行。
        //如果&#39;/cache/recovery/block.map&#39;还没有创建&#xff0c;停止重新启动&#xff0c;它肯定会失败&#xff0c;
        //并有机会捕获一个bugreport时&#xff0c;这仍然是可行的。
        if (reason !&#61; null &amp;&amp; reason.startsWith(PowerManager.REBOOT_RECOVERY_UPDATE)) {
            File packageFile &#61; new File(UNCRYPT_PACKAGE_FILE);
            if (packageFile.exists()) {
                String filename &#61; null;
                try {
                    filename &#61; FileUtils.readTextFile(packageFile, 0, null);
                } catch (IOException e) {
                    Slog.e(TAG, &#34;Error reading uncrypt package file&#34;, e);
                }

                if (filename !&#61; null &amp;&amp; filename.startsWith(&#34;/data&#34;)) {
                    if (!new File(BLOCK_MAP_FILE).exists()) {
                        Slog.e(TAG, &#34;Can&#39;t find block map file, uncrypt failed or &#34; &#43;
                                &#34;unexpected runtime restart?&#34;);
                        return;
                    }
                }
            }
        }
        Runnable runnable &#61; new Runnable() {
            &#64;Override
            public void run() {
                synchronized (this) {
                    //当属性sys.shutdown.requested的值为1时&#xff0c;会重启
                    //当属性的值不为空&#xff0c;且不为1时&#xff0c;会关机
                    ShutdownThread.rebootOrShutdown(null, reboot, reason);
                }
            }
        };

        // ShutdownThread must run on a looper capable of displaying the UI.
        //ShutdownThread必须在一个能够显示UI的looper上运行
        //即UI线程启动ShutdownThread的rebootOrShutdown
        Message msg &#61; Message.obtain(UiThread.getHandler(), runnable);
        msg.setAsynchronous(true);
        UiThread.getHandler().sendMessage(msg);

    }
}</code></pre> 
<h3>4.2.4[SystemServer.java] createSystemContext</h3> 
<p><strong>说明&#xff1a;</strong>初始化系统上下文&#xff0c; 该过程会创建对象有ActivityThread&#xff0c;Instrumentation, ContextImpl&#xff0c;LoadedApk&#xff0c;Application</p> 
<p><strong>源码&#xff1a;</strong></p> 
<pre class="has"><code class="language-java">private void createSystemContext() {
    //创建system_server进程的上下文信息
    ActivityThread activityThread &#61; ActivityThread.systemMain();
    mSystemContext &#61; activityThread.getSystemContext();
    //设置主题
    mSystemContext.setTheme(DEFAULT_SYSTEM_THEME);

    //获取systemui上下文信息&#xff0c;并设置主题
    final Context systemUiContext &#61; activityThread.getSystemUiContext();
    systemUiContext.setTheme(DEFAULT_SYSTEM_THEME);
}</code></pre> 
<h3>4.2.5[SystemServer.java] startBootstrapServices</h3> 
<p><strong>说明&#xff1a;</strong>用于启动系统Boot级服务&#xff0c;有ActivityManagerService, PowerManagerService, LightsService, DisplayManagerService&#xff0c; PackageManagerService&#xff0c; UserManagerService&#xff0c; sensor服务.</p> 
<p><strong>源码&#xff1a;</strong></p> 
<p> </p> 
<pre class="has"><code class="language-java">private void startBootstrapServices() {
    traceBeginAndSlog(&#34;StartWatchdog&#34;);
    //启动watchdog
    //尽早启动watchdog&#xff0c;如果在早起启动时发生死锁&#xff0c;我们可以让system_server
    //崩溃&#xff0c;从而进行详细分析
    final Watchdog watchdog &#61; Watchdog.getInstance();
    watchdog.start();
    traceEnd();

...
    //添加PLATFORM_COMPAT_SERVICE&#xff0c;Platform compat服务被ActivityManagerService、PackageManagerService
    //以及将来可能出现的其他服务使用。
    traceBeginAndSlog(&#34;PlatformCompat&#34;);
    ServiceManager.addService(Context.PLATFORM_COMPAT_SERVICE,
            new PlatformCompat(mSystemContext));
    traceEnd();

    //阻塞等待installd完成启动&#xff0c;以便有机会创建具有适当权限的关键目录&#xff0c;如/data/user。
    //我们需要在初始化其他服务之前完成此任务。
    traceBeginAndSlog(&#34;StartInstaller&#34;);
    Installer installer &#61; mSystemServiceManager.startService(Installer.class);
    traceEnd();
...
    //启动服务ActivityManagerService,并为其设置mSystemServiceManager和installer
    traceBeginAndSlog(&#34;StartActivityManager&#34;);
    ActivityTaskManagerService atm &#61; mSystemServiceManager.startService(
            ActivityTaskManagerService.Lifecycle.class).getService();
    mActivityManagerService &#61; ActivityManagerService.Lifecycle.startService(mSystemServiceManager, atm);
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    mActivityManagerService.setInstaller(installer);
    mWindowManagerGlobalLock &#61; atm.getGlobalLock();
    traceEnd();

    //启动服务PowerManagerService
    //Power manager需要尽早启动&#xff0c;因为其他服务需要它。
    //本机守护进程可能正在监视它的注册&#xff0c;
    //因此它必须准备好立即处理传入的绑定器调用(包括能够验证这些调用的权限)
。
    traceBeginAndSlog(&#34;StartPowerManager&#34;);
    mPowerManagerService &#61; mSystemServiceManager.startService(
PowerManagerService.class);
    traceEnd();

...
    //初始化power management
    traceBeginAndSlog(&#34;InitPowerManagement&#34;);
    mActivityManagerService.initPowerManagement();
    traceEnd();

    //启动recovery system&#xff0c;以防需要重新启动
    traceBeginAndSlog(&#34;StartRecoverySystemService&#34;);
    mSystemServiceManager.startService(RecoverySystemService.class);
    traceEnd();
...
    //启动服务LightsService
    //管理led和显示背光&#xff0c;所以我们需要它来打开显示
    traceBeginAndSlog(&#34;StartLightsService&#34;);
    mSystemServiceManager.startService(LightsService.class);
    traceEnd();
...
    //启动服务DisplayManagerService
    //显示管理器需要在包管理器之前提供显示指标
    traceBeginAndSlog(&#34;StartDisplayManager&#34;);
    mDisplayManagerService &#61; mSystemServiceManager.startService(DisplayManagerService.class);
    traceEnd();

    // Boot Phases: Phase100: 在初始化package manager之前&#xff0c;需要默认的显示.
    traceBeginAndSlog(&#34;WaitForDisplay&#34;);
    mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);
    traceEnd();

    //当设备正在加密时&#xff0c;仅运行核心
    String cryptState &#61; VoldProperties.decrypt().orElse(&#34;&#34;);
    if (ENCRYPTING_STATE.equals(cryptState)) {
        Slog.w(TAG, &#34;Detected encryption in progress - only parsing core apps&#34;);
        mOnlyCore &#61; true;
    } else if (ENCRYPTED_STATE.equals(cryptState)) {
        Slog.w(TAG, &#34;Device encrypted - only parsing core apps&#34;);
        mOnlyCore &#61; true;
    }
...
    //启动服务PackageManagerService
    traceBeginAndSlog(&#34;StartPackageManagerService&#34;);
    try {
        Watchdog.getInstance().pauseWatchingCurrentThread(&#34;packagemanagermain&#34;);
        mPackageManagerService &#61; PackageManagerService.main(mSystemContext, installer,
                mFactoryTestMode !&#61; FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
    } finally {
        Watchdog.getInstance().resumeWatchingCurrentThread(&#34;packagemanagermain&#34;);
    }
...
    //启动服务UserManagerService&#xff0c;新建目录/data/user/
    traceBeginAndSlog(&#34;StartUserManagerService&#34;);
    mSystemServiceManager.startService(UserManagerService.LifeCycle.class);
    traceEnd();

    // Set up the Application instance for the system process and get  started.
    //为系统进程设置应用程序实例并开始。
    //设置AMS
    traceBeginAndSlog(&#34;SetSystemProcess&#34;);
    mActivityManagerService.setSystemProcess();
    traceEnd();

    //使用一个ActivityManager实例完成watchdog设置并监听重启&#xff0c;
//只有在ActivityManagerService作为一个系统进程正确启动后才能这样做
    traceBeginAndSlog(&#34;InitWatchdog&#34;);
    watchdog.init(mSystemContext, mActivityManagerService);
    traceEnd();

     //传感器服务需要访问包管理器服务、app ops服务和权限服务&#xff0c;
    //因此我们在它们之后启动它。
    //在单独的线程中启动传感器服务。在使用它之前应该检查完成情况。
    mSensorServiceStart &#61; SystemServerInitThreadPool.get().submit(() -&gt; {
        TimingsTraceLog traceLog &#61; new TimingsTraceLog(
                SYSTEM_SERVER_TIMING_ASYNC_TAG, Trace.
TRACE_TAG_SYSTEM_SERVER);
        traceLog.traceBegin(START_SENSOR_SERVICE);
        startSensorService(); //启动传感器服务
        traceLog.traceEnd();
    }, START_SENSOR_SERVICE);
}

</code></pre> 
<h3><strong>4.2.6[SystemServer.java] startCoreServices</strong></h3> 
<p><strong>说明&#xff1a;</strong>启动核心服务BatteryService&#xff0c;UsageStatsService&#xff0c;WebViewUpdateService、BugreportManagerService、GpuService等</p> 
<p><strong>源码&#xff1a;</strong></p> 
<pre class="has"><code class="language-java">private void startCoreServices() {
    //启动服务BatteryService&#xff0c;用于统计电池电量&#xff0c;需要LightService.
    mSystemServiceManager.startService(BatteryService.class);

    //启动服务UsageStatsService&#xff0c;用于统计应用使用情况
    mSystemServiceManager.startService(UsageStatsService.class);
    mActivityManagerService.setUsageStatsManager(
            LocalServices.getService(UsageStatsManagerInternal.class));

    //启动服务WebViewUpdateService
    //跟踪可更新的WebView是否处于就绪状态&#xff0c;并监视更新安装
    if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_WEBVIEW)) {
        mWebViewUpdateService &#61; mSystemServiceManager.startService(WebViewUpdateService.class);
    }

    //启动CachedDeviceStateService&#xff0c;跟踪和缓存设备状态
    mSystemServiceManager.startService(CachedDeviceStateService.class);

    //启动BinderCallsStatsService, 跟踪在绑定器调用中花费的cpu时间
    traceBeginAndSlog(&#34;StartBinderCallsStatsService&#34;);
    mSystemServiceManager.startService(BinderCallsStatsService.LifeCycle.class);
    traceEnd();

    //启动LooperStatsService&#xff0c;跟踪处理程序中处理消息所花费的时间。
    traceBeginAndSlog(&#34;StartLooperStatsService&#34;);
    mSystemServiceManager.startService(LooperStatsService.Lifecycle.class);
    traceEnd();

    //启动RollbackManagerService&#xff0c;管理apk回滚
    mSystemServiceManager.startService(RollbackManagerService.class);

    //启动BugreportManagerService&#xff0c;捕获bugreports的服务
    mSystemServiceManager.startService(BugreportManagerService.class);

    //启动GpuService&#xff0c;为GPU和GPU驱动程序提供服务。
    mSystemServiceManager.startService(GpuService.class);
}

</code></pre> 
<p> </p> 
<h3>4.2.7[SystemServer.java] startOtherServices</h3> 
<p><strong>说明&#xff1a;</strong>启动其他的服务&#xff0c;开始处理一大堆尚未重构和整理的东西,这里的服务太多&#xff0c;大体启动过程类似&#xff0c;就不详细说明</p> 
<p><strong>源码&#xff1a;</strong></p> 
<pre class="has"><code class="language-java">private void startOtherServices() {
	...
    //启动TelecomLoaderService&#xff0c;通话相关核心服务
    mSystemServiceManager.startService(TelecomLoaderService.class);

    //启动TelephonyRegistry
    telephonyRegistry &#61; new TelephonyRegistry(context);
    ServiceManager.addService(&#34;telephony.registry&#34;, telephonyRegistry);
	...
	//启动AlarmManagerService&#xff0c;时钟管理
	mSystemServiceManager.startService(new AlarmManagerService(context));
	...
	//启动InputManagerService
	inputManager &#61; new InputManagerService(context);
	ServiceManager.addService(Context.INPUT_SERVICE, inputManager,
            /* allowIsolated&#61; */ false, DUMP_FLAG_PRIORITY_CRITICAL);
	...
	inputManager.setWindowManagerCallbacks(wm.getInputManagerCallback());
    inputManager.start();
	...
	//Phase480:在接收到此启动阶段后&#xff0c;服务可以获得锁设置数据
    mSystemServiceManager.startBootPhase(SystemService.PHASE_LOCK_SETTINGS_READY);

    //Phase500:在接收到这个启动阶段之后&#xff0c;服务可以安全地调用核心系统服务&#xff0c;
    //如PowerManager或PackageManager。
    mSystemServiceManager.startBootPhase(SystemService.PHASE_SYSTEM_SERVICES_READY);
	
	mActivityManagerService.systemReady(() -&gt; {
        //Phase550:在接收到此引导阶段后&#xff0c;服务可以广播意图。
        mSystemServiceManager.startBootPhase(
                SystemService.PHASE_ACTIVITY_MANAGER_READY);

		//Phase600:在接收到这个启动阶段后&#xff0c;服务可以启动/绑定到第三方应用程序。
        //此时&#xff0c;应用程序将能够对服务进行绑定调用。
        mSystemServiceManager.startBootPhase(
        SystemService.PHASE_THIRD_PARTY_APPS_CAN_START);
	}
}</code></pre> 
<p> </p> 
<h1>5.服务启动分析</h1> 
<p>  服务启动流程如下&#xff0c;从阶段0到阶段1000&#xff0c;一共8个阶段。</p> 
<p> <img alt="" class="has" height="535" src="https://img-blog.csdnimg.cn/20191215170638875.jpg?x-oss-process&#61,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpcmFuZmVuZw&#61;&#61;,size_16,color_FFFFFF,t_70" width="859" /></p> 
<p>其中PHASE_BOOT_COMPLETED&#61;1000&#xff0c;该阶段是发生在Boot完成和home应用启动完毕。系统服务更倾向于监听该阶段&#xff0c;而不是注册广播ACTION_BOOT_COMPLETED&#xff0c;从而降低系统延迟。</p> 
<p> </p> 
<h2>5.1 PHASE 0:</h2> 
<p><strong>  说明&#xff1a;</strong>startBootstrapServices() 启动引导级服务</p> 
<p>       主要启动以下10个服务&#xff1a;</p> 
<ul><li>Installer</li><li>DeviceIdentifiersPolicyService</li><li>UriGrantsManagerService</li><li>ActivityTaskManagerService</li><li>ActivityManagerService</li><li>PowerManagerService</li><li>ThermalManagerService</li><li>RecoverySystemService</li><li>LightsService</li><li>DisplayManagerService</li></ul>
<p> </p> 
<p>启动完后&#xff0c;进入PHASE_WAIT_FOR_DEFAULT_DISPLAY&#61;100&#xff0c; 即Phase100阶段</p> 
<p><strong>源码&#xff1a;</strong></p> 
<pre class="has"><code class="language-java">       ...
    //1.启动DeviceIdentifiersPolicyService
    mSystemServiceManager.startService(DeviceIdentifiersPolicyService.class);

    //2.启动UriGrantsManagerService
    mSystemServiceManager.startService(UriGrantsManagerService.Lifecycle.class);

    //3.启动ActivityTaskManagerService
    atm &#61; mSystemServiceManager.startService(
                ActivityTaskManagerService.Lifecycle.class).getService();

    //4.启动PowerManagerService
    mPowerManagerService &#61; mSystemServiceManager.startService(PowerManagerService.class);

    //5.启动ThermalManagerService
    mSystemServiceManager.startService(ThermalManagerService.class);

    //6.启动RecoverySystemService
    mSystemServiceManager.startService(RecoverySystemService.class);

    //7.启动LightsService
    mSystemServiceManager.startService(LightsService.class);

    //8.启动DisplayManagerService
    mDisplayManagerService &#61; mSystemServiceManager.startService(DisplayManagerService.class);
    
    //执行回调函数 onBootPhase&#xff0c;把PHASE_WAIT_FOR_DEFAULT_DISPLAY&#61;100&#xff0c; 传入各个service的 onBootPhase
    mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);
       ...
}
</code></pre> 
<h2>5.2 PHASE 100  (阶段100)&#xff1a;</h2> 
<p><strong>定义&#xff1a;</strong>public static final int PHASE_WAIT_FOR_DEFAULT_DISPLAY &#61; 100;</p> 
<p><strong>说明&#xff1a;</strong> 启动阶段-Boot Phase, 该阶段需要等待Display有默认显示</p> 
<p>             进入阶段PHASE_WAIT_FOR_DEFAULT_DISPLAY&#61;100回调服务: onBootPhase(100)</p> 
<p><strong>流程&#xff1a;</strong>startBootPhase(100) -&gt; onBootPhase(100)</p> 
<p>从以下源码可以看到这里遍历了一下服务列表&#xff0c;然后回调到各服务的 onBootPhase() 方法中了。每个服务的onBootPhase()处理都不相同&#xff0c;这里不详细分析</p> 
<p><strong>源码&#xff1a;</strong></p> 
<pre class="has"><code class="language-java">public void startBootPhase(final int phase) {
        ...
        mCurrentPhase &#61; phase;
        ...
        final int serviceLen &#61; mServices.size();
        for (int i &#61; 0; i &lt; serviceLen; i&#43;&#43;) {
            final SystemService service &#61; mServices.get(i);
            ...
            try {
                service.onBootPhase(mCurrentPhase); // 轮训前面加过的service&#xff0c;把phase加入服务回调
            } catch (Exception ex) {
                ...
            }
            ...
        }
        ...
    }</code></pre> 
<p> </p> 
<p><strong>创建以下80多个服务</strong>&#xff1a;</p> 
<ul><li>BatteryService</li><li>UsageStatsService</li><li>WebViewUpdateService</li><li>CachedDeviceStateService</li><li>BinderCallsStatsService</li><li>LooperStatsService</li><li>RollbackManagerService</li><li>BugreportManagerService</li><li>GpuService</li><li>....</li></ul>
<h2>5.3 PHASE 480  (阶段480)&#xff1a;</h2> 
<p><strong>定义&#xff1a;</strong>public static final int PHASE_LOCK_SETTINGS_READY &#61; 480;</p> 
<p><strong>说明&#xff1a;</strong> 该阶段后&#xff0c; 服务可以获取到锁屏设置的数据了</p> 
<p>               480到500之间没有任何操作&#xff0c;直接进入500</p> 
<p> </p> 
<h2>5.4 PHASE 500  (阶段500)&#xff1a;</h2> 
<p><strong>定义&#xff1a;</strong>public static final int PHASE_SYSTEM_SERVICES_READY &#61; 500;</p> 
<p><strong>说明&#xff1a;</strong>该阶段后&#xff0c;服务可以安全地调用核心系统服务&#xff0c;比如PowerManager或PackageManager。</p> 
<p>        启动以下两个服务&#xff1a;</p> 
<ul><li> PermissionPolicyService</li><li> eviceSpecificServices</li></ul>
<p> </p> 
<h2>5.5 PHASE 520  (阶段520)&#xff1a;</h2> 
<p><strong>定义&#xff1a;</strong>public static final int PHASE_DEVICE_SPECIFIC_SERVICES_READY &#61; 520;</p> 
<p><strong>说明&#xff1a;</strong>在接收到这个引导阶段之后&#xff0c;服务可以安全地调用特定于设备的服务。</p> 
<p>       告诉AMS可以运行第三方代码,Making services ready</p> 
<p>       mActivityManagerService.systemReady()</p> 
<p> </p> 
<h2>5.6 PHASE 550  (阶段550)&#xff1a;</h2> 
<p><strong>定义&#xff1a;</strong>public static final int PHASE_ACTIVITY_MANAGER_READY &#61; 550;</p> 
<p><strong>说明&#xff1a;</strong>该阶段后&#xff0c;服务可以接收到广播Intents</p> 
<p>       AMS启动native crash监控&#xff0c;启动SystemUI&#xff0c;其余服务调用systemReady()</p> 
<p>    <strong>   1) AMS启动native crash监控&#xff1a;</strong></p> 
<pre class="has"><code>mActivityManagerService.startObservingNativeCrashes();</code></pre> 
<p><strong>         2)  启动systemUI&#xff1a;</strong></p> 
<p>              startSystemUi()</p> 
<p>       <strong> 3) 其余服务调用systemReady()&#xff1a;</strong></p> 
<ul><li>       networkManagementF.systemReady()</li><li>       ipSecServiceF.systemReady();</li><li>       networkStatsF.systemReady();</li><li>       connectivityF.systemReady();</li><li>       networkPolicyF.systemReady(networkPolicyInitReadySignal);</li></ul>
<h2> </h2> 
<h2><strong>5.7 PHASE 600  (阶段600)&#xff1a;</strong></h2> 
<p><strong>定义&#xff1a;</strong>public static final int PHASE_THIRD_PARTY_APPS_CAN_START &#61; 600;</p> 
<p><strong>说明&#xff1a;</strong>该阶段后&#xff0c;服务可以启动/绑定到第三方应用程序。此时&#xff0c;应用程序将能够对服务进行绑定调用。</p> 
<p>               各种服务调用systemRunning方法&#xff1a;</p> 
<ul><li>               locationF.systemRunning();</li><li>               countryDetectorF.systemRunning();</li><li>               networkTimeUpdaterF.systemRunning();</li><li>               inputManagerF.systemRunning();</li><li>               telephonyRegistryF.systemRunning();</li><li>               mediaRouterF.systemRunning();</li><li>               mmsServiceF.systemRunning();</li><li>               incident.systemRunning();</li><li>               touchEventDispatchServiceF.systemRunning();</li></ul>
<p> </p> 
<h2>5.8 PHASE 1000 (阶段1000)&#xff1a;</h2> 
<p><strong>定义&#xff1a;</strong>public static final int PHASE_BOOT_COMPLETED &#61; 1000;</p> 
<p><strong>说明&#xff1a; </strong>该阶段后&#xff0c;服务可以允许用户与设备交互。此阶段在引导完成且主应用程序启动时发生。</p> 
<p>             系统服务可能更倾向于监听此阶段&#xff0c;而不是为完成的操作注册广播接收器&#xff0c;以减少总体延迟。</p> 
<p>               在经过一系列流程&#xff0c;再调用AMS.finishBooting()时&#xff0c;则进入阶段Phase1000。</p> 
<p>               到此&#xff0c;系统服务启动阶段完成就绪&#xff0c;system_server进程启动完成则进入Looper.loop()状态&#xff0c;随时待命&#xff0c;等待消息队列MessageQueue中的消息到来&#xff0c;则马上进入执行状态。</p> 
<p> </p> 
<h1><strong>6.服务分类</strong></h1> 
<p>system_server进程启动的服务&#xff0c;从源码角度划分为引导服务、核心服务、其他服务3类。</p> 
<p><strong>引导服务 Boot Service (10个)&#xff1a;</strong></p> 
<p><img alt="" class="has" height="100" src="https://img-blog.csdnimg.cn/20191215171222121.jpg" width="883" /></p> 
<p><strong>核心服务 Core Service(9个)&#xff1a;</strong></p> 
<p> <img alt="" class="has" height="100" src="https://img-blog.csdnimg.cn/20191215171249723.jpg" width="882" /></p> 
<p><strong>其他服务 Other Service(70个&#43;)&#xff1a;</strong></p> 
<p> <img alt="" class="has" height="661" src="https://img-blog.csdnimg.cn/20191215171313873.jpg?x-oss-process&#61,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpcmFuZmVuZw&#61;&#61;,size_16,color_FFFFFF,t_70" width="885" /></p> 
<p>7.总结</p> 
<ul><li>Zygote启动后fork的第一个进程为SystemServer,在手机中的进程别名为&#34;system_server&#34;&#xff0c;主要用来启动系统中的服务</li><li>.Zygote fork后&#xff0c;进入SystemServer的main()</li><li>SystemServer在启动过程中&#xff0c;先初始化一些系统变量&#xff0c;加载类库&#xff0c;创建Context对象&#xff0c;创建SystemServiceManager对象等候再启动服务</li><li>启动的服务分为 引导服务(Boot Service)、核心服务(Core Service)和其他服务(Other Service)三大类&#xff0c;共90多个服务</li><li>SystemServer在启动服务前&#xff0c;会尝试与Zygote建立Socket通信&#xff0c;通信成功后才去启动服务</li><li>启动的服务都单独运行在SystemServer的各自线程中&#xff0c;同属于SystemServer进程</li></ul>
<p> </p>