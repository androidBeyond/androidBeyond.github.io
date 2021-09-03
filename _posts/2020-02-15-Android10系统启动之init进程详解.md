---
layout:     post
title:      Android10系统启动之init进程详解
subtitle:   这篇文章我们来详细学习下Android10系统启动中init进程的启动过程
date:       2020-02-15
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android10
    - 系统启动
    - init进程
--- 

<h2>1.概述&#xff1a;</h2> 
<p>init进程是linux系统中用户空间的第一个进程&#xff0c;进程号为1.</p> 
<p>当bootloader启动后&#xff0c;启动kernel&#xff0c;kernel启动完后&#xff0c;在用户空间启动init进程&#xff0c;再通过init进程&#xff0c;来读取init.rc中的相关配置&#xff0c;从而来启动其他相关进程以及其他操作。</p> 
<p> </p> 
<p>init进程被赋予了很多重要工作&#xff0c;init进程启动主要分为两个阶段&#xff1a;</p> 
<p>第一个阶段完成以下内容&#xff1a;</p> 
<ul><li>ueventd/watchdogd跳转及环境变量设置</li><li>挂载文件系统并创建目录</li><li>初始化日志输出、挂载分区设备</li><li>启用SELinux安全策略</li><li>开始第二阶段前的准备</li></ul>
<p>第二个阶段完成以下内容&#xff1a;</p> 
<ul><li>初始化属性系统</li><li>执行SELinux第二阶段并恢复一些文件安全上下文</li><li>新建epoll并初始化子进程终止信号处理函数</li><li>设置其他系统属性并开启属性服务</li></ul>
<h2>2.架构</h2> 
<h3>2.1 Init进程如何被启动&#xff1f;</h3> 
<p><img alt="" class="has" height="364" src="https://img-blog.csdnimg.cn/20191215154631193.png?x-oss-process&#61,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpcmFuZmVuZw&#61;&#61;,size_16,color_FFFFFF,t_70" width="633" /></p> 
<p>Init进程是在Kernel启动后&#xff0c;启动的第一个用户空间进程&#xff0c;PID为1。</p> 
<p>kernel_init启动后&#xff0c;完成一些init的初始化操作&#xff0c;然后去系统根目录下依次找ramdisk_execute_command和execute_command设置的应用程序&#xff0c;如果这两个目录都找不到&#xff0c;就依次去根目录下找 /sbin/init&#xff0c;/etc/init&#xff0c;/bin/init,/bin/sh 这四个应用程序进行启动&#xff0c;只要这些应用程序有一个启动了&#xff0c;其他就不启动了。</p> 
<p>Android系统一般会在根目录下放一个init的可执行文件&#xff0c;也就是说Linux系统的init进程在内核初始化完成后&#xff0c;就直接执行init这个文件。</p> 
<p> </p> 
<h3>2.2Init进程启动后&#xff0c;做了哪些事&#xff1f;</h3> 
<p><img alt="" class="has" height="299" src="https://img-blog.csdnimg.cn/2019121515470388.png?x-oss-process&#61,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpcmFuZmVuZw&#61;&#61;,size_16,color_FFFFFF,t_70" width="731" /></p> 
<p>Init进程启动后&#xff0c;首先挂载文件系统、再挂载相应的分区&#xff0c;启动SELinux安全策略&#xff0c;启动属性服务&#xff0c;解析rc文件&#xff0c;并启动相应属性服务进程&#xff0c;初始化epoll&#xff0c;依次设置signal、property、keychord这3个fd可读时相对应的回调函数。进入无线循环&#xff0c;用来响应各个进程的变化与重建。</p> 
<p> </p> 
<p> </p> 
<h2>3.kernel启动init进程 源码分析</h2> 
<p> </p> 
<h3>3.1  kernel_init</h3> 
<p>kernel/msm-4.19/init/main.c</p> 
<pre class="has"><code>kernel/msm-4.19/init/main.c
kernel_init()
  |
run_init_process(ramdisk_execute_command)  //运行可执行文件&#xff0c;启动init进程</code></pre> 
<pre class="has"><code class="language-cpp">static int __ref kernel_init(void *unused)
{
	kernel_init_freeable(); //进行init进程的一些初始化操作
	/* need to finish all async __init code before freeing the memory */
	async_synchronize_full();// 等待所有异步调用执行完成,&#xff0c;在释放内存前&#xff0c;必须完成所有的异步 __init 代码
	free_initmem();// 释放所有init.* 段中的内存
	mark_rodata_ro(); //arm64空实现
	system_state &#61; SYSTEM_RUNNING;// 设置系统状态为运行状态
	numa_default_policy(); // 设定NUMA系统的默认内存访问策略

	flush_delayed_fput(); // 释放所有延时的struct file结构体

	if (ramdisk_execute_command) { //ramdisk_execute_command的值为&#34;/init&#34;
		if (!run_init_process(ramdisk_execute_command)) //运行根目录下的init程序
			return 0;
		pr_err(&#34;Failed to execute %s\n&#34;, ramdisk_execute_command);
	}

	/*
	 * We try each of these until one succeeds.
	 *
	 * The Bourne shell can be used instead of init if we are
	 * trying to recover a really broken machine.
	 */
	if (execute_command) { //execute_command的值如果有定义就去根目录下找对应的应用程序,然后启动
		if (!run_init_process(execute_command))
			return 0;
		pr_err(&#34;Failed to execute %s.  Attempting defaults...\n&#34;,
			execute_command);
	}
	if (!run_init_process(&#34;/sbin/init&#34;) || //如果ramdisk_execute_command和execute_command定义的应用程序都没有找到&#xff0c;
	//就到根目录下找 /sbin/init&#xff0c;/etc/init&#xff0c;/bin/init,/bin/sh 这四个应用程序进行启动

	    !run_init_process(&#34;/etc/init&#34;) ||
	    !run_init_process(&#34;/bin/init&#34;) ||
	    !run_init_process(&#34;/bin/sh&#34;))
		return 0;

	panic(&#34;No init found.  Try passing init&#61; option to kernel. &#34;
	      &#34;See Linux Documentation/init.txt for guidance.&#34;);
}</code></pre> 
<h3>3.2  do_basic_setup</h3> 
<pre class="has"><code class="language-cpp">kernel_init_freeable&#xff08;&#xff09;
|
do_basic_setup&#xff08;&#xff09;</code></pre> 
<p> </p> 
<p> </p> 
<p> </p> 
<pre class="has"><code class="language-cpp">static void __init do_basic_setup(void)
{
	cpuset_init_smp();//针对SMP系统&#xff0c;初始化内核control group的cpuset子系统。
	usermodehelper_init();// 创建khelper单线程工作队列&#xff0c;用于协助新建和运行用户空间程序
	shmem_init();// 初始化共享内存
	driver_init();// 初始化设备驱动
	init_irq_proc();//创建/proc/irq目录, 并初始化系统中所有中断对应的子目录
	do_ctors();// 执行内核的构造函数
	usermodehelper_enable();// 启用usermodehelper
	do_initcalls();//遍历initcall_levels数组&#xff0c;调用里面的initcall函数&#xff0c;这里主要是对设备、驱动、文件系统进行初始化&#xff0c;
	//之所有将函数封装到数组进行遍历&#xff0c;主要是为了好扩展

	random_int_secret_init();//初始化随机数生成池
}</code></pre> 
<p> </p> 
<p> </p> 
<p> </p> 
<p> </p> 
<h2>4. Init 进程启动源码分析</h2> 
<p>我们主要是分析Android Q(10.0) 的init的代码。</p> 
<p> </p> 
<p>涉及源码文件&#xff1a;</p> 
<pre class="has"><code class="language-cpp">platform/system/core/init/main.cpp
platform/system/core/init/init.cpp
platform/system/core/init/ueventd.cpp
platform/system/core/init/selinux.cpp
platform/system/core/init/subcontext.cpp
platform/system/core/base/logging.cpp
platform/system/core/init/first_stage_init.cpp
platform/system/core/init/first_stage_main.cpp
platform/system/core/init/first_stage_mount.cpp
platform/system/core/init/keyutils.h
platform/system/core/init/property_service.cpp
platform/external/selinux/libselinux/src/label.c
platform/system/core/init/signal_handler.cpp
platform/system/core/init/service.cpp</code></pre> 
<h3>4.1 Init 进程入口</h3> 
<p>前面已经通过kernel_init,启动了init进程&#xff0c;init进程属于一个守护进程&#xff0c;准确的说&#xff0c;它是Linux系统中用户控制的第一个进程&#xff0c;它的进程号为1。它的生命周期贯穿整个Linux内核运行的始终。Android中所有其它的进程共同的鼻祖均为init进程。</p> 
<p>可以通过<strong>&#34;adb shell ps |grep init&#34;</strong> 的命令来查看init的进程号。</p> 
<p>Android Q(10.0) 的init入口函数由原先的init.cpp 调整到了main.cpp&#xff0c;把各个阶段的操作分离开来&#xff0c;使代码更加简洁命令&#xff0c;接下来我们就从main函数开始学习。</p> 
<p><strong> [system/core/init/main.cpp]</strong></p> 
<pre class="has"><code class="language-cpp">/*
 * 1.第一个参数argc表示参数个数&#xff0c;第二个参数是参数列表&#xff0c;也就是具体的参数
 * 2.main函数有四个参数入口&#xff0c;
 *一是参数中有ueventd&#xff0c;进入ueventd_main
 *二是参数中有subcontext&#xff0c;进入InitLogging 和SubcontextMain
 *三是参数中有selinux_setup&#xff0c;进入SetupSelinux
 *四是参数中有second_stage&#xff0c;进入SecondStageMain
 *3.main的执行顺序如下&#xff1a;
   *  (1)ueventd_main    init进程创建子进程ueventd&#xff0c;
   *      并将创建设备节点文件的工作托付给ueventd&#xff0c;ueventd通过两种方式创建设备节点文件
   *  (2)FirstStageMain  启动第一阶段
   *  (3)SetupSelinux     加载selinux规则&#xff0c;并设置selinux日志,完成SELinux相关工作
   *  (4)SecondStageMain  启动第二阶段
 */
int main(int argc, char** argv) {
    //当argv[0]的内容为ueventd时&#xff0c;strcmp的值为0,&#xff01;strcmp为1
    //1表示true&#xff0c;也就执行ueventd_main,ueventd主要是负责设备节点的创建、权限设定等一些列工作
    if (!strcmp(basename(argv[0]), &#34;ueventd&#34;)) {
        return ueventd_main(argc, argv);
    }

   //当传入的参数个数大于1时&#xff0c;执行下面的几个操作
    if (argc &gt; 1) {
        //参数为subcontext&#xff0c;初始化日志系统&#xff0c;
        if (!strcmp(argv[1], &#34;subcontext&#34;)) {
            android::base::InitLogging(argv, &amp;android::base::KernelLogger);
            const BuiltinFunctionMap function_map;
            return SubcontextMain(argc, argv, &amp;function_map);
        }

      //参数为“selinux_setup”,启动Selinux安全策略
        if (!strcmp(argv[1], &#34;selinux_setup&#34;)) {
            return SetupSelinux(argv);
        }
       //参数为“second_stage”,启动init进程第二阶段
        if (!strcmp(argv[1], &#34;second_stage&#34;)) {
            return SecondStageMain(argc, argv);
        }
    }
 // 默认启动init进程第一阶段
    return FirstStageMain(argc, argv);
}
</code></pre> 
<h3>4.2 ueventd_main</h3> 
<p><strong>代码路径&#xff1a;</strong>platform/system/core/init/ueventd.cpp</p> 
<p>Android根文件系统的镜像中不存在“/dev”目录&#xff0c;该目录是init进程启动后动态创建的。</p> 
<p>因此&#xff0c;建立Android中设备节点文件的重任&#xff0c;也落在了init进程身上。为此&#xff0c;init进程创建子进程ueventd&#xff0c;并将创建设备节点文件的工作托付给ueventd。</p> 
<p>ueventd通过两种方式创建设备节点文件。</p> 
<p>第一种方式对应“冷插拔”&#xff08;Cold Plug&#xff09;&#xff0c;即以预先定义的设备信息为基础&#xff0c;当ueventd启动后&#xff0c;统一创建设备节点文件。这一类设备节点文件也被称为静态节点文件。</p> 
<p>第二种方式对应“热插拔”&#xff08;Hot Plug&#xff09;&#xff0c;即在系统运行中&#xff0c;当有设备插入USB端口时&#xff0c;ueventd就会接收到这一事件&#xff0c;为插入的设备动态创建设备节点文件。这一类设备节点文件也被称为动态节点文件。</p> 
<p> </p> 
<pre class="has"><code class="language-cpp">int ueventd_main(int argc, char** argv) {
    //设置新建文件的默认值,这个与chmod相反,这里相当于新建文件后的权限为666
    umask(000); 

    //初始化内核日志&#xff0c;位于节点/dev/kmsg, 此时logd、logcat进程还没有起来&#xff0c;
    //采用kernel的log系统&#xff0c;打开的设备节点/dev/kmsg&#xff0c; 那么可通过cat /dev/kmsg来获取内核log。
    android::base::InitLogging(argv, &amp;android::base::KernelLogger);

    //注册selinux相关的用于打印log的回调函数
    SelinuxSetupKernelLogging(); 
    SelabelInitialize();

    //解析xml&#xff0c;根据不同SOC厂商获取不同的hardware rc文件
    auto ueventd_configuration &#61; ParseConfig({&#34;/ueventd.rc&#34;, &#34;/vendor/ueventd.rc&#34;,
                                              &#34;/odm/ueventd.rc&#34;, &#34;/ueventd.&#34; &#43; hardware &#43; &#34;.rc&#34;});

    //冷启动
    if (access(COLDBOOT_DONE, F_OK) !&#61; 0) {
        ColdBoot cold_boot(uevent_listener, uevent_handlers);
        cold_boot.Run();
    }
    for (auto&amp; uevent_handler : uevent_handlers) {
        uevent_handler-&gt;ColdbootDone();
    }

    //忽略子进程终止信号
    signal(SIGCHLD, SIG_IGN);
    // Reap and pending children that exited between the last call to waitpid() and setting SIG_IGN
    // for SIGCHLD above.
       //在最后一次调用waitpid&#xff08;&#xff09;和为上面的sigchld设置SIG_IGN之间退出的获取和挂起的子级
    while (waitpid(-1, nullptr, WNOHANG) &gt; 0) {
    }

    //监听来自驱动的uevent,进行“热插拔”处理
    uevent_listener.Poll([&amp;uevent_handlers](const Uevent&amp; uevent) {
        for (auto&amp; uevent_handler : uevent_handlers) {
            uevent_handler-&gt;HandleUevent(uevent); //热启动&#xff0c;创建设备
        }
        return ListenerAction::kContinue;
    });
    return 0;
}
</code></pre> 
<h3>4.3 init 进程启动第一阶段</h3> 
<p><strong>代码路径&#xff1a;</strong>platform\system\core\init\first_stage_init.cpp</p> 
<p>init进程第一阶段做的主要工作是挂载分区,创建设备节点和一些关键目录,初始化日志输出系统,启用SELinux安全策略</p> 
<p>第一阶段完成以下内容&#xff1a;</p> 
<p>/* 01. 创建文件系统目录并挂载相关的文件系统 */</p> 
<p>/* 02. 屏蔽标准的输入输出/初始化内核log系统 */</p> 
<p> </p> 
<p><strong>4.3.1 FirstStageMain</strong></p> 
<p> </p> 
<pre class="has"><code class="language-cpp">int FirstStageMain(int argc, char** argv) {
    //init crash时重启引导加载程序
    //这个函数主要作用将各种信号量&#xff0c;如SIGABRT,SIGBUS等的行为设置为SA_RESTART,一旦监听到这些信号即执行重启系统
    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }
    //清空文件权限
    umask(0);

    CHECKCALL(clearenv());
    CHECKCALL(setenv(&#34;PATH&#34;, _PATH_DEFPATH, 1));

    //在RAM内存上获取基本的文件系统&#xff0c;剩余的被rc文件所用
    CHECKCALL(mount(&#34;tmpfs&#34;, &#34;/dev&#34;, &#34;tmpfs&#34;, MS_NOSUID, &#34;mode&#61;0755&#34;));
    CHECKCALL(mkdir(&#34;/dev/pts&#34;, 0755));
    CHECKCALL(mkdir(&#34;/dev/socket&#34;, 0755));
    CHECKCALL(mount(&#34;devpts&#34;, &#34;/dev/pts&#34;, &#34;devpts&#34;, 0, NULL));
#define MAKE_STR(x) __STRING(x)
    CHECKCALL(mount(&#34;proc&#34;, &#34;/proc&#34;, &#34;proc&#34;, 0, &#34;hidepid&#61;2,gid&#61;&#34; MAKE_STR(AID_READPROC)));
#undef MAKE_STR

    // 非特权应用不能使用Andrlid cmdline
    CHECKCALL(chmod(&#34;/proc/cmdline&#34;, 0440));
    gid_t groups[] &#61; {AID_READPROC};
    CHECKCALL(setgroups(arraysize(groups), groups));
    CHECKCALL(mount(&#34;sysfs&#34;, &#34;/sys&#34;, &#34;sysfs&#34;, 0, NULL));
    CHECKCALL(mount(&#34;selinuxfs&#34;, &#34;/sys/fs/selinux&#34;, &#34;selinuxfs&#34;, 0, NULL));

    CHECKCALL(mknod(&#34;/dev/kmsg&#34;, S_IFCHR | 0600, makedev(1, 11)));

    if constexpr (WORLD_WRITABLE_KMSG) {
        CHECKCALL(mknod(&#34;/dev/kmsg_debug&#34;, S_IFCHR | 0622, makedev(1, 11)));
    }

    CHECKCALL(mknod(&#34;/dev/random&#34;, S_IFCHR | 0666, makedev(1, 8)));
    CHECKCALL(mknod(&#34;/dev/urandom&#34;, S_IFCHR | 0666, makedev(1, 9)));


    //这对于日志包装器是必需的&#xff0c;它在ueventd运行之前被调用
    CHECKCALL(mknod(&#34;/dev/ptmx&#34;, S_IFCHR | 0666, makedev(5, 2)));
    CHECKCALL(mknod(&#34;/dev/null&#34;, S_IFCHR | 0666, makedev(1, 3)));


    //在第一阶段挂在tmpfs、mnt/vendor、mount/product分区。其他的分区不需要在第一阶段加载&#xff0c;
    //只需要在第二阶段通过rc文件解析来加载。
    CHECKCALL(mount(&#34;tmpfs&#34;, &#34;/mnt&#34;, &#34;tmpfs&#34;, MS_NOEXEC | MS_NOSUID | MS_NODEV,
                    &#34;mode&#61;0755,uid&#61;0,gid&#61;1000&#34;));
    
    //创建可供读写的vendor目录
    CHECKCALL(mkdir(&#34;/mnt/vendor&#34;, 0755));
    // /mnt/product is used to mount product-specific partitions that can not be
    // part of the product partition, e.g. because they are mounted read-write.
    CHECKCALL(mkdir(&#34;/mnt/product&#34;, 0755));

    // 挂载APEX&#xff0c;这在Android 10.0中特殊引入&#xff0c;用来解决碎片化问题&#xff0c;类似一种组件方式&#xff0c;对Treble的增强&#xff0c;
    // 不写谷歌特殊更新不需要完整升级整个系统版本&#xff0c;只需要像升级APK一样&#xff0c;进行APEX组件升级
    CHECKCALL(mount(&#34;tmpfs&#34;, &#34;/apex&#34;, &#34;tmpfs&#34;, MS_NOEXEC | MS_NOSUID | MS_NODEV,
                    &#34;mode&#61;0755,uid&#61;0,gid&#61;0&#34;));

    // /debug_ramdisk is used to preserve additional files from the debug ramdisk
    CHECKCALL(mount(&#34;tmpfs&#34;, &#34;/debug_ramdisk&#34;, &#34;tmpfs&#34;, MS_NOEXEC | MS_NOSUID | MS_NODEV,
                    &#34;mode&#61;0755,uid&#61;0,gid&#61;0&#34;));
#undef CHECKCALL

    //把标准输入、标准输出和标准错误重定向到空设备文件&#34;/dev/null&#34;
    SetStdioToDevNull(argv);
    //在/dev目录下挂载好 tmpfs 以及 kmsg 
    //这样就可以初始化 /kernel Log 系统&#xff0c;供用户打印log
    InitKernelLogging(argv);

    ...

    /* 初始化一些必须的分区
     *主要作用是去解析/proc/device-tree/firmware/android/fstab,
     * 然后得到&#34;/system&#34;, &#34;/vendor&#34;, &#34;/odm&#34;三个目录的挂载信息
     */
    if (!DoFirstStageMount()) {
        LOG(FATAL) &lt;&lt; &#34;Failed to mount required partitions early ...&#34;;
    }

    struct stat new_root_info;
    if (stat(&#34;/&#34;, &amp;new_root_info) !&#61; 0) {
        PLOG(ERROR) &lt;&lt; &#34;Could not stat(\&#34;/\&#34;), not freeing ramdisk&#34;;
        old_root_dir.reset();
    }

    if (old_root_dir &amp;&amp; old_root_info.st_dev !&#61; new_root_info.st_dev) {
        FreeRamdisk(old_root_dir.get(), old_root_info.st_dev);
    }

    SetInitAvbVersionInRecovery();

    static constexpr uint32_t kNanosecondsPerMillisecond &#61; 1e6;
    uint64_t start_ms &#61; start_time.time_since_epoch().count() / kNanosecondsPerMillisecond;
    setenv(&#34;INIT_STARTED_AT&#34;, std::to_string(start_ms).c_str(), 1);

    //启动init进程&#xff0c;传入参数selinux_steup
    // 执行命令&#xff1a; /system/bin/init selinux_setup
    const char* path &#61; &#34;/system/bin/init&#34;;
    const char* args[] &#61; {path, &#34;selinux_setup&#34;, nullptr};
    execv(path, const_cast&lt;char**&gt;(args));
    PLOG(FATAL) &lt;&lt; &#34;execv(\&#34;&#34; &lt;&lt; path &lt;&lt; &#34;\&#34;) failed&#34;;

    return 1;
}
</code></pre> 
<h3>4.4 加载SELinux规则</h3> 
<p>SELinux是「Security-Enhanced Linux」的简称&#xff0c;是美国国家安全局「NSA&#61;The National Security Agency」</p> 
<p>和SCC&#xff08;Secure Computing Corporation&#xff09;开发的 Linux的一个扩张强制访问控制安全模块。</p> 
<p>在这种访问控制体系的限制下&#xff0c;进程只能访问那些在他的任务中所需要文件。</p> 
<p><strong>selinux有两种工作模式&#xff1a;</strong></p> 
<ol><li>permissive&#xff0c;所有的操作都被允许&#xff08;即没有MAC&#xff09;&#xff0c;但是如果违反权限的话&#xff0c;会记录日志,一般eng模式用</li><li>enforcing&#xff0c;所有操作都会进行权限检查。一般user和user-debug模式用</li></ol>
<p>不管是security_setenforce还是security_getenforce都是去操作/sys/fs/selinux/enforce 文件, <strong>0表示permissive 1表示enforcing</strong></p> 
<p> </p> 
<p><strong>4.4.1 SetupSelinux</strong></p> 
<p><strong>说明:</strong>初始化selinux&#xff0c;加载SELinux规则&#xff0c;配置SELinux相关log输出&#xff0c;并启动第二阶段</p> 
<p><strong>代码路径&#xff1a; </strong>platform\system\core\init\selinux.cpp</p> 
<pre class="has"><code class="language-cpp">/*此函数初始化selinux&#xff0c;然后执行init以在init selinux中运行*/
int SetupSelinux(char** argv) {
       //初始化Kernel日志
    InitKernelLogging(argv);

       // Debug版本init crash时重启引导加载程序
    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }

    //注册回调&#xff0c;用来设置需要写入kmsg的selinux日志
    SelinuxSetupKernelLogging();
   
     //加载SELinux规则
    SelinuxInitialize();

    /*
       *我们在内核域中&#xff0c;希望转换到init域。在其xattrs中存储selabel的文件系统&#xff08;如ext4&#xff09;不需要显式restorecon&#xff0c;
       *但其他文件系统需要。尤其是对于ramdisk&#xff0c;如对于a/b设备的恢复映像&#xff0c;这是必需要做的一步。
       *其实就是当前在内核域中&#xff0c;在加载Seliux后&#xff0c;需要重新执行init切换到C空间的用户态
       */
    if (selinux_android_restorecon(&#34;/system/bin/init&#34;, 0) &#61;&#61; -1) {
        PLOG(FATAL) &lt;&lt; &#34;restorecon failed of /system/bin/init failed&#34;;
    }

  //准备启动innit进程&#xff0c;传入参数second_stage
    const char* path &#61; &#34;/system/bin/init&#34;;
    const char* args[] &#61; {path, &#34;second_stage&#34;, nullptr};
    execv(path, const_cast&lt;char**&gt;(args));

    /*
       *执行 /system/bin/init second_stage, 进入第二阶段
       */
    PLOG(FATAL) &lt;&lt; &#34;execv(\&#34;&#34; &lt;&lt; path &lt;&lt; &#34;\&#34;) failed&#34;;

    return 1;
}
</code></pre> 
<p><strong>4.4.2 SelinuxInitialize()</strong></p> 
<p> </p> 
<p> </p> 
<pre class="has"><code class="language-cpp">/*加载selinux 规则*/
void SelinuxInitialize() {
    LOG(INFO) &lt;&lt; &#34;Loading SELinux policy&#34;;
    if (!LoadPolicy()) {
        LOG(FATAL) &lt;&lt; &#34;Unable to load SELinux policy&#34;;
    }

    //获取当前Kernel的工作模式
    bool kernel_enforcing &#61; (security_getenforce() &#61;&#61; 1);

    //获取工作模式的配置
    bool is_enforcing &#61; IsEnforcing();

    //如果当前的工作模式与配置的不同&#xff0c;就将当前的工作模式改掉
    if (kernel_enforcing !&#61; is_enforcing) {
        if (security_setenforce(is_enforcing)) {
            PLOG(FATAL) &lt;&lt; &#34;security_setenforce(&#34; &lt;&lt; (is_enforcing ? &#34;true&#34; : &#34;false&#34;)
                        &lt;&lt; &#34;) failed&#34;;
        }
    }

    if (auto result &#61; WriteFile(&#34;/sys/fs/selinux/checkreqprot&#34;, &#34;0&#34;); !result) {
        LOG(FATAL) &lt;&lt; &#34;Unable to write to /sys/fs/selinux/checkreqprot: &#34; &lt;&lt; result.error();
    }
}</code></pre> 
<pre class="has"><code class="language-cpp">/*
 *加载SELinux规则
 *这里区分了两种情况,这两种情况只是区分从哪里加载安全策略文件,
 *第一个是从 /vendor/etc/selinux/precompiled_sepolicy 读取,
 *第二个是从 /sepolicy 读取,他们最终都是调用selinux_android_load_policy_from_fd方法
 */
bool LoadPolicy() {
    return IsSplitPolicyDevice() ? LoadSplitPolicy() : LoadMonolithicPolicy();
}
</code></pre> 
<h2>4.5 init进程启动第二阶段</h2> 
<p>第二阶段主要内容&#xff1a;</p> 
<ol><li>创建进程会话密钥并初始化属性系统</li><li>进行SELinux第二阶段并恢复一些文件安全上下文</li><li>新建epoll并初始化子进程终止信号处理函数&#xff0c;详细看第五节-信号处理</li><li>启动匹配属性的服务端&#xff0c; 详细查看第六节-属性服务</li><li>解析init.rc等文件&#xff0c;建立rc文件的action 、service&#xff0c;启动其他进程&#xff0c;详细查看第七节-rc文件解析</li></ol>
<p> </p> 
<p><img alt="" class="has" height="286" src="https://img-blog.csdnimg.cn/20191215155613672.png?x-oss-process&#61,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpcmFuZmVuZw&#61;&#61;,size_16,color_FFFFFF,t_70" width="759" /></p> 
<h3>4.5.1 SecondStageMain</h3> 
<p> </p> 
<pre class="has"><code class="language-cpp">int SecondStageMain(int argc, char** argv) {
    /* 01. 创建进程会话密钥并初始化属性系统 */
    keyctl_get_keyring_ID(KEY_SPEC_SESSION_KEYRING, 1);

    //创建 /dev/.booting 文件&#xff0c;就是个标记&#xff0c;表示booting进行中
    close(open(&#34;/dev/.booting&#34;, O_WRONLY | O_CREAT | O_CLOEXEC, 0000));

    // 初始化属性系统&#xff0c;并从指定文件读取属性
    property_init();

    /* 02. 进行SELinux第二阶段并恢复一些文件安全上下文 */
    	SelinuxRestoreContext();

    /* 03. 新建epoll并初始化子进程终止信号处理函数 */
    Epoll epoll;
    if (auto result &#61; epoll.Open(); !result) {
        PLOG(FATAL) &lt;&lt; result.error();
    }

    	     InstallSignalFdHandler(&amp;epoll);

    /* 04. 设置其他系统属性并开启系统属性服务*/
    StartPropertyService(&amp;epoll);

    	   /* 05 解析init.rc等文件&#xff0c;建立rc文件的action 、service&#xff0c;启动其他进程*/
    ActionManager&amp; am &#61; ActionManager::GetInstance();
    ServiceList&amp; sm &#61; ServiceList::GetInstance();
    LoadBootScripts(am, sm);
}
</code></pre> 
<p><strong>代码流程详细解析&#xff1a;</strong></p> 
<pre class="has"><code class="language-cpp">int SecondStageMain(int argc, char** argv) {
    /* 
    *init crash时重启引导加载程序
    *这个函数主要作用将各种信号量&#xff0c;如SIGABRT,SIGBUS等的行为设置为SA_RESTART,一旦监听到这些信号即执行重启系统
    */
    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }

    //把标准输入、标准输出和标准错误重定向到空设备文件&#34;/dev/null&#34;
    SetStdioToDevNull(argv);
    //在/dev目录下挂载好 tmpfs 以及 kmsg 
    //这样就可以初始化 /kernel Log 系统&#xff0c;供用户打印log
    InitKernelLogging(argv);
    LOG(INFO) &lt;&lt; &#34;init second stage started!&#34;;

    // 01. 创建进程会话密钥并初始化属性系统
    keyctl_get_keyring_ID(KEY_SPEC_SESSION_KEYRING, 1);

    //创建 /dev/.booting 文件&#xff0c;就是个标记&#xff0c;表示booting进行中
    close(open(&#34;/dev/.booting&#34;, O_WRONLY | O_CREAT | O_CLOEXEC, 0000));

    // 初始化属性系统&#xff0c;并从指定文件读取属性
    property_init();

    /*
     * 1.如果参数同时从命令行和DT传过来&#xff0c;DT的优先级总是大于命令行的
     * 2.DT即device-tree&#xff0c;中文意思是设备树&#xff0c;这里面记录自己的硬件配置和系统运行参数&#xff0c;
     */
    process_kernel_dt(); // 处理 DT属性
    process_kernel_cmdline(); // 处理命令行属性

    // 处理一些其他的属性
    export_kernel_boot_props();

    // Make the time that init started available for bootstat to log.
    property_set(&#34;ro.boottime.init&#34;, getenv(&#34;INIT_STARTED_AT&#34;));
    property_set(&#34;ro.boottime.init.selinux&#34;, getenv(&#34;INIT_SELINUX_TOOK&#34;));

    // Set libavb version for Framework-only OTA match in Treble build.
    const char* avb_version &#61; getenv(&#34;INIT_AVB_VERSION&#34;);
    if (avb_version) property_set(&#34;ro.boot.avb_version&#34;, avb_version);

    // See if need to load debug props to allow adb root, when the device is unlocked.
    const char* force_debuggable_env &#61; getenv(&#34;INIT_FORCE_DEBUGGABLE&#34;);
    if (force_debuggable_env &amp;&amp; AvbHandle::IsDeviceUnlocked()) {
        load_debug_prop &#61; &#34;true&#34;s &#61;&#61; force_debuggable_env;
    }

    // 基于cmdline设置memcg属性
    bool memcg_enabled &#61; android::base::GetBoolProperty(&#34;ro.boot.memcg&#34;,false);
    if (memcg_enabled) {
       // root memory control cgroup
       mkdir(&#34;/dev/memcg&#34;, 0700);
       chown(&#34;/dev/memcg&#34;,AID_ROOT,AID_SYSTEM);
       mount(&#34;none&#34;, &#34;/dev/memcg&#34;, &#34;cgroup&#34;, 0, &#34;memory&#34;);
       // app mem cgroups, used by activity manager, lmkd and zygote
       mkdir(&#34;/dev/memcg/apps/&#34;,0755);
       chown(&#34;/dev/memcg/apps/&#34;,AID_SYSTEM,AID_SYSTEM);
       mkdir(&#34;/dev/memcg/system&#34;,0550);
       chown(&#34;/dev/memcg/system&#34;,AID_SYSTEM,AID_SYSTEM);
    }

    // 清空这些环境变量&#xff0c;之前已经存到了系统属性中去了
    unsetenv(&#34;INIT_STARTED_AT&#34;);
    unsetenv(&#34;INIT_SELINUX_TOOK&#34;);
    unsetenv(&#34;INIT_AVB_VERSION&#34;);
    unsetenv(&#34;INIT_FORCE_DEBUGGABLE&#34;);

    // Now set up SELinux for second stage.
    SelinuxSetupKernelLogging();
    SelabelInitialize();

    /*
     * 02. 进行SELinux第二阶段并恢复一些文件安全上下文 
     * 恢复相关文件的安全上下文,因为这些文件是在SELinux安全机制初始化前创建的&#xff0c;
     * 所以需要重新恢复上下文
     */
    SelinuxRestoreContext();

   /*
    * 03. 新建epoll并初始化子进程终止信号处理函数
    *  创建epoll实例&#xff0c;并返回epoll的文件描述符
    */
    Epoll epoll;
    if (auto result &#61; epoll.Open(); !result) {
        PLOG(FATAL) &lt;&lt; result.error();
    }

    /* 
     *主要是创建handler处理子进程终止信号&#xff0c;注册一个signal到epoll进行监听
     *进行子继承处理
     */
    InstallSignalFdHandler(&amp;epoll);

    // 进行默认属性配置相关的工作
    property_load_boot_defaults(load_debug_prop);
    UmountDebugRamdisk();
    fs_mgr_vendor_overlay_mount_all();
    export_oem_lock_status();

    /*
     *04. 设置其他系统属性并开启系统属性服务
     */
    StartPropertyService(&amp;epoll);
    MountHandler mount_handler(&amp;epoll);

    //为USB存储设置udc Contorller, sys/class/udc
    set_usb_controller();

    // 匹配命令和函数之间的对应关系
    const BuiltinFunctionMap function_map;
    Action::set_function_map(&amp;function_map);

    if (!SetupMountNamespaces()) {
        PLOG(FATAL) &lt;&lt; &#34;SetupMountNamespaces failed&#34;;
    }

    // 初始化文件上下文
    subcontexts &#61; InitializeSubcontexts();

   /*
     *05 解析init.rc等文件&#xff0c;建立rc文件的action 、service&#xff0c;启动其他进程
     */
    ActionManager&amp; am &#61; ActionManager::GetInstance();
    ServiceList&amp; sm &#61; ServiceList::GetInstance();

    LoadBootScripts(am, sm);

    // Turning this on and letting the INFO logging be discarded adds 0.2s to
    // Nexus 9 boot time, so it&#39;s disabled by default.
    if (false) DumpState();

    // 当GSI脚本running时&#xff0c;确保GSI状态可用.
    if (android::gsi::IsGsiRunning()) {
        property_set(&#34;ro.gsid.image_running&#34;, &#34;1&#34;);
    } else {
        property_set(&#34;ro.gsid.image_running&#34;, &#34;0&#34;);
    }


    am.QueueBuiltinAction(SetupCgroupsAction, &#34;SetupCgroups&#34;);

    // 执行rc文件中触发器为 on early-init 的语句
    am.QueueEventTrigger(&#34;early-init&#34;);

    // 等冷插拔设备初始化完成
    am.QueueBuiltinAction(wait_for_coldboot_done_action, &#34;wait_for_coldboot_done&#34;);

    // 开始查询来自 /dev的 action
    am.QueueBuiltinAction(MixHwrngIntoLinuxRngAction, &#34;MixHwrngIntoLinuxRng&#34;);
    am.QueueBuiltinAction(SetMmapRndBitsAction, &#34;SetMmapRndBits&#34;);
    am.QueueBuiltinAction(SetKptrRestrictAction, &#34;SetKptrRestrict&#34;);

    // 设备组合键的初始化操作
    Keychords keychords;
    am.QueueBuiltinAction(
        [&amp;epoll, &amp;keychords](const BuiltinArguments&amp; args) -&gt; Result&lt;Success&gt; {
            for (const auto&amp; svc : ServiceList::GetInstance()) {
                keychords.Register(svc-&gt;keycodes());
            }
            keychords.Start(&amp;epoll, HandleKeychord);
            return Success();
        },
        &#34;KeychordInit&#34;);

    //在屏幕上显示Android 静态LOGO
    am.QueueBuiltinAction(console_init_action, &#34;console_init&#34;);

    // 执行rc文件中触发器为on init的语句
    am.QueueEventTrigger(&#34;init&#34;);

    // Starting the BoringSSL self test, for NIAP certification compliance.
    am.QueueBuiltinAction(StartBoringSslSelfTest, &#34;StartBoringSslSelfTest&#34;);

    // Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random
    // wasn&#39;t ready immediately after wait_for_coldboot_done
    am.QueueBuiltinAction(MixHwrngIntoLinuxRngAction, &#34;MixHwrngIntoLinuxRng&#34;);

    // Initialize binder before bringing up other system services
    am.QueueBuiltinAction(InitBinder, &#34;InitBinder&#34;);

    // 当设备处于充电模式时&#xff0c;不需要mount文件系统或者启动系统服务
    // 充电模式下&#xff0c;将charger假如执行队列&#xff0c;否则把late-init假如执行队列
    std::string bootmode &#61; GetProperty(&#34;ro.bootmode&#34;, &#34;&#34;);
    if (bootmode &#61;&#61; &#34;charger&#34;) {
        am.QueueEventTrigger(&#34;charger&#34;);
    } else {
        am.QueueEventTrigger(&#34;late-init&#34;);
    }

    // 基于属性当前状态 运行所有的属性触发器.
    am.QueueBuiltinAction(queue_property_triggers_action, &#34;queue_property_triggers&#34;);

    while (true) {
        // By default, sleep until something happens.
        auto epoll_timeout &#61; std::optional&lt;std::chrono::milliseconds&gt;{};

        if (do_shutdown &amp;&amp; !shutting_down) {
            do_shutdown &#61; false;
            if (HandlePowerctlMessage(shutdown_command)) {
                shutting_down &#61; true;
            }
        }

//依次执行每个action中携带command对应的执行函数
        if (!(waiting_for_prop || Service::is_exec_service_running())) {
            am.ExecuteOneCommand();
        }
        if (!(waiting_for_prop || Service::is_exec_service_running())) {
            if (!shutting_down) {
                auto next_process_action_time &#61; HandleProcessActions();

                // If there&#39;s a process that needs restarting, wake up in time for that.
                if (next_process_action_time) {
                    epoll_timeout &#61; std::chrono::ceil&lt;std::chrono::milliseconds&gt;(
                            *next_process_action_time - boot_clock::now());
                    if (*epoll_timeout &lt; 0ms) epoll_timeout &#61; 0ms;
                }
            }

            // If there&#39;s more work to do, wake up again immediately.
            if (am.HasMoreCommands()) epoll_timeout &#61; 0ms;
        }

        // 循环等待事件发生
        if (auto result &#61; epoll.Wait(epoll_timeout); !result) {
            LOG(ERROR) &lt;&lt; result.error();
        }
    }

    return 0;
}</code></pre> 
<p> </p> 
<h2>5. 信号处理</h2> 
<p style="text-indent:33px;">init是一个守护进程&#xff0c;为了防止init的子进程成为僵尸进程(zombie process)&#xff0c;需要init在子进程在结束时获取子进程的结束码&#xff0c;通过结束码将程序表中的子进程移除&#xff0c;防止成为僵尸进程的子进程占用程序表的空间&#xff08;程序表的空间达到上限时&#xff0c;系统就不能再启动新的进程了&#xff0c;会引起严重的系统问题&#xff09;。</p> 
<p style="text-indent:33px;"><strong>子进程重启流程如下图所示&#xff1a;</strong></p> 
<p> </p> 
<p><img alt="" class="has" height="214" src="https://img-blog.csdnimg.cn/20191215155741981.png?x-oss-process&#61,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpcmFuZmVuZw&#61;&#61;,size_16,color_FFFFFF,t_70" width="579" /></p> 
<p><strong>信号处理主要工作&#xff1a;</strong></p> 
<ul><li>初始化信号signal句柄</li><li>循环处理子进程</li><li>注册epoll句柄</li><li>处理子进程终止</li></ul>
<p><strong>注: </strong> EPOLL类似于POLL&#xff0c;是Linux中用来做事件触发的&#xff0c;跟EventBus功能差不多。linux很长的时间都在使用select来做事件触发&#xff0c;它是通过轮询来处理的&#xff0c;轮询的fd数目越多&#xff0c;自然耗时越多&#xff0c;对于大量的描述符处理&#xff0c;EPOLL更有优势</p> 
<p> </p> 
<h3>5.1 InstallSignalFdHandler</h3> 
<p style="text-indent:33px;">在linux当中&#xff0c;父进程是通过捕捉SIGCHLD信号来得知子进程运行结束的情况&#xff0c;SIGCHLD信号会在子进程终止的时候发出&#xff0c;了解这些背景后&#xff0c;我们来看看init进程如何处理这个信号。</p> 
<ul><li>首先&#xff0c;新建一个sigaction结构体&#xff0c;sa_handler是信号处理函数&#xff0c;指向内核指定的函数指针SIG_DFL和Android 9.0及之前的版本不同&#xff0c;这里不再通过socket的读写句柄进行接收信号&#xff0c;改成了内核的信号处理函数SIG_DFL。</li><li>然后&#xff0c;sigaction(SIGCHLD, &amp;act, nullptr) 这个是建立信号绑定关系&#xff0c;也就是说当监听到SIGCHLD信号时&#xff0c;由act这个sigaction结构体处理</li><li>最后&#xff0c;RegisterHandler 的作用就是signal_read_fd&#xff08;之前的s[1]&#xff09;收到信号&#xff0c;触发handle_signal</li></ul>
<p>终上所述&#xff0c;InstallSignalFdHandler函数的作用就是&#xff0c;接收到SIGCHLD信号时触发HandleSignalFd进行信号处理</p> 
<p>                   <strong>                信号处理示意图&#xff1a;</strong></p> 
<p> </p> 
<p><img alt="" class="has" height="213" src="https://img-blog.csdnimg.cn/20191215155935532.png?x-oss-process&#61,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpcmFuZmVuZw&#61;&#61;,size_16,color_FFFFFF,t_70" width="500" /></p> 
<p> </p> 
<p><strong>代码路径&#xff1a;</strong>platform/system/core/init.cpp</p> 
<p><strong>说明&#xff1a;</strong>该函数主要的作用是初始化子进程终止信号处理过程</p> 
<pre class="has"><code class="language-cpp">static void InstallSignalFdHandler(Epoll* epoll) {
 
    // SA_NOCLDSTOP使init进程只有在其子进程终止时才会受到SIGCHLD信号
    const struct sigaction act { .sa_handler &#61; SIG_DFL, .sa_flags &#61; SA_NOCLDSTOP };
    sigaction(SIGCHLD, &amp;act, nullptr);

    sigset_t mask;
    sigemptyset(&amp;mask);
    sigaddset(&amp;mask, SIGCHLD); 

    if (!IsRebootCapable()) {
        // 如果init不具有 CAP_SYS_BOOT的能力&#xff0c;则它此时正值容器中运行
        // 在这种场景下&#xff0c;接收SIGTERM 将会导致系统关闭
        sigaddset(&amp;mask, SIGTERM);
    }

    if (sigprocmask(SIG_BLOCK, &amp;mask, nullptr) &#61;&#61; -1) {
        PLOG(FATAL) &lt;&lt; &#34;failed to block signals&#34;;
    }

    // 注册处理程序以解除对子进程中的信号的阻止
    const int result &#61; pthread_atfork(nullptr, nullptr, &amp;UnblockSignals);
    if (result !&#61; 0) {
        LOG(FATAL) &lt;&lt; &#34;Failed to register a fork handler: &#34; &lt;&lt; strerror(result);
    }

    //创建信号句柄
    signal_fd &#61; signalfd(-1, &amp;mask, SFD_CLOEXEC);
    if (signal_fd &#61;&#61; -1) {
        PLOG(FATAL) &lt;&lt; &#34;failed to create signalfd&#34;;
    }

    //信号注册&#xff0c;当signal_fd收到信号时&#xff0c;触发HandleSignalFd
    if (auto result &#61; epoll-&gt;RegisterHandler(signal_fd, HandleSignalFd); !result) {
        LOG(FATAL) &lt;&lt; result.error();
    }
}</code></pre> 
<h3>5.2 RegisterHandler</h3> 
<p> </p> 
<p><strong>代码路径&#xff1a;</strong>/platform/system/core/epoll.cpp</p> 
<p><strong>说明&#xff1a;</strong>信号注册,把fd句柄加入到 epoll_fd_的监听队列中</p> 
<pre class="has"><code class="language-cpp">Result&lt;void&gt; Epoll::RegisterHandler(int fd, std::function&lt;void()&gt; handler, uint32_t events) {
    if (!events) {
        return Error() &lt;&lt; &#34;Must specify events&#34;;
    }
    auto [it, inserted] &#61; epoll_handlers_.emplace(fd, std::move(handler));
    if (!inserted) {
        return Error() &lt;&lt; &#34;Cannot specify two epoll handlers for a given FD&#34;;
    }
    epoll_event ev;
    ev.events &#61; events;
    // std::map&#39;s iterators do not get invalidated until erased, so we use the
    // pointer to the std::function in the map directly for epoll_ctl.
    ev.data.ptr &#61; reinterpret_cast&lt;void*&gt;(&amp;it-&gt;second);
    // 将fd的可读事件加入到epoll_fd_的监听队列中
    if (epoll_ctl(epoll_fd_, EPOLL_CTL_ADD, fd, &amp;ev) &#61;&#61; -1) {
        Result&lt;void&gt; result &#61; ErrnoError() &lt;&lt; &#34;epoll_ctl failed to add fd&#34;;
        epoll_handlers_.erase(fd);
        return result;
    }
    return {};
}
</code></pre> 
<p> </p> 
<h3>5.3 HandleSignalFd</h3> 
<p> </p> 
<p><strong>代码路径&#xff1a;</strong>platform/system/core/init.cpp</p> 
<p><strong>说明&#xff1a;</strong>监控SIGCHLD信号&#xff0c;调用 ReapAnyOutstandingChildren 来 终止出现问题的子进程</p> 
<pre class="has"><code class="language-cpp">static void HandleSignalFd() {
    signalfd_siginfo siginfo;
    ssize_t bytes_read &#61; TEMP_FAILURE_RETRY(read(signal_fd, &amp;siginfo, sizeof(siginfo)));
    if (bytes_read !&#61; sizeof(siginfo)) {
        PLOG(ERROR) &lt;&lt; &#34;Failed to read siginfo from signal_fd&#34;;
        return;
    }

   //监控SIGCHLD信号
    switch (siginfo.ssi_signo) {
        case SIGCHLD:
            ReapAnyOutstandingChildren();
            break;
        case SIGTERM:
            HandleSigtermSignal(siginfo);
            break;
        default:
            PLOG(ERROR) &lt;&lt; &#34;signal_fd: received unexpected signal &#34; &lt;&lt; siginfo.ssi_signo;
            break;
    }
}</code></pre> 
<p> </p> 
<h3>5.4 ReapOneProcess</h3> 
<p> </p> 
<p><strong>代码路径&#xff1a;</strong>/platform/system/core/sigchld_handle.cpp</p> 
<p><strong>说明&#xff1a;</strong>ReapOneProcess是最终的处理函数了&#xff0c;这个函数先用waitpid找出挂掉进程的pid,然后根据pid找到对应Service&#xff0c;</p> 
<p>最后调用Service的Reap方法清除资源,根据进程对应的类型&#xff0c;决定是否重启机器或重启进程</p> 
<pre class="has"><code class="language-cpp">void ReapAnyOutstandingChildren() {
    while (ReapOneProcess()) {
    }
}

static bool ReapOneProcess() {
    siginfo_t siginfo &#61; {};
    //用waitpid函数获取状态发生变化的子进程pid
    //waitpid的标记为WNOHANG&#xff0c;即非阻塞&#xff0c;返回为正值就说明有进程挂掉了
    if (TEMP_FAILURE_RETRY(waitid(P_ALL, 0, &amp;siginfo, WEXITED | WNOHANG | WNOWAIT)) !&#61; 0) {
        PLOG(ERROR) &lt;&lt; &#34;waitid failed&#34;;
        return false;
    }

    auto pid &#61; siginfo.si_pid;
    if (pid &#61;&#61; 0) return false;

    // 当我们知道当前有一个僵尸pid&#xff0c;我们使用scopeguard来清楚该pid
    auto reaper &#61; make_scope_guard([pid] { TEMP_FAILURE_RETRY(waitpid(pid, nullptr, WNOHANG)); });

    std::string name;
    std::string wait_string;
    Service* service &#61; nullptr;

    if (SubcontextChildReap(pid)) {
        name &#61; &#34;Subcontext&#34;;
    } else {
        //通过pid找到对应的service
        service &#61; ServiceList::GetInstance().FindService(pid, &amp;Service::pid);

        if (service) {
            name &#61; StringPrintf(&#34;Service &#39;%s&#39; (pid %d)&#34;, service-&gt;name().c_str(), pid);
            if (service-&gt;flags() &amp; SVC_EXEC) {
                auto exec_duration &#61; boot_clock::now() - service-&gt;time_started();
                auto exec_duration_ms &#61;
                    std::chrono::duration_cast&lt;std::chrono::milliseconds&gt;(exec_duration).count();
                wait_string &#61; StringPrintf(&#34; waiting took %f seconds&#34;, exec_duration_ms / 1000.0f);
            } else if (service-&gt;flags() &amp; SVC_ONESHOT) {
                auto exec_duration &#61; boot_clock::now() - service-&gt;time_started();
                auto exec_duration_ms &#61;
                        std::chrono::duration_cast&lt;std::chrono::milliseconds&gt;(exec_duration)
                                .count();
                wait_string &#61; StringPrintf(&#34; oneshot service took %f seconds in background&#34;,exec_duration_ms / 1000.0f);
            }
        } else {
            name &#61; StringPrintf(&#34;Untracked pid %d&#34;, pid);
        }
    }

    if (siginfo.si_code &#61;&#61; CLD_EXITED) {
        LOG(INFO) &lt;&lt; name &lt;&lt; &#34; exited with status &#34; &lt;&lt; siginfo.si_status &lt;&lt; wait_string;
    } else {
        LOG(INFO) &lt;&lt; name &lt;&lt; &#34; received signal &#34; &lt;&lt; siginfo.si_status &lt;&lt; wait_string;
    }

    //没有找到service&#xff0c;说明已经结束了&#xff0c;退出
    if (!service) return true;

    service-&gt;Reap(siginfo);//清除子进程相关的资源

    if (service-&gt;flags() &amp; SVC_TEMPORARY) {
        ServiceList::GetInstance().RemoveService(*service); //移除该service
    }

    return true;
}</code></pre> 
<p> </p> 
<h2>6.属性服务</h2> 
<p style="text-indent:33px;">我们在开发和调试过程中看到通过property_set可以轻松设置系统属性&#xff0c;那干嘛这里还要启动一个属性服务呢&#xff1f;这里其实涉及到一些权限的问题&#xff0c;不是所有进程都可以随意修改任何的系统属性&#xff0c;</p> 
<p style="text-indent:33px;">Android将属性的设置统一交由init进程管理&#xff0c;其他进程不能直接修改属性&#xff0c;而只能通知init进程来修改&#xff0c;而在这过程中&#xff0c;init进程可以进行权限控制&#xff0c;我们来看看具体的流程是什么</p> 
<p> </p> 
<h3>6.1 property_init</h3> 
<p><strong>代码路径&#xff1a;</strong>platform/system/core/property_service.cpp</p> 
<p><strong>说明&#xff1a;</strong>初始化属性系统&#xff0c;并从指定文件读取属性&#xff0c;并进行SELinux注册&#xff0c;进行属性权限控制</p> 
<p>清除缓存&#xff0c;这里主要是清除几个链表以及在内存中的映射&#xff0c;新建property_filename目录&#xff0c;这个目录的值为 /dev/_properties_</p> 
<p>然后就是调用CreateSerializedPropertyInfo加载一些系统属性的类别信息&#xff0c;最后将加载的链表写入文件并映射到内存</p> 
<pre class="has"><code class="language-cpp">void property_init() {

    //设置SELinux回调&#xff0c;进行权限控制
    selinux_callback cb;
    cb.func_audit &#61; PropertyAuditCallback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);

    mkdir(&#34;/dev/__properties__&#34;, S_IRWXU | S_IXGRP | S_IXOTH);
    CreateSerializedPropertyInfo();
    if (__system_property_area_init()) {
        LOG(FATAL) &lt;&lt; &#34;Failed to initialize property area&#34;;
    }
    if (!property_info_area.LoadDefaultPath()) {
        LOG(FATAL) &lt;&lt; &#34;Failed to load serialized property info file&#34;;
    }
}</code></pre> 
<p> </p> 
<p><strong>通过CreateSerializedPropertyInfo 来加载以下目录的contexts</strong>&#xff1a;</p> 
<p><strong>1)与SELinux相关</strong></p> 
<pre class="has"><code>/system/etc/selinux/plat_property_contexts

/vendor/etc/selinux/vendor_property_contexts

/vendor/etc/selinux/nonplat_property_contexts

/product/etc/selinux/product_property_contexts

/odm/etc/selinux/odm_property_contexts</code></pre> 
<p><strong>2)与SELinux无关</strong></p> 
<pre class="has"><code>/plat_property_contexts

/vendor_property_contexts

/nonplat_property_contexts

/product_property_contexts

/odm_property_contexts</code></pre> 
<p> </p> 
<h3>6.2 StartPropertyService</h3> 
<p><strong>代码路径&#xff1a; </strong>platform/system/core/init.cpp</p> 
<p><strong>说明&#xff1a;</strong>启动属性服务</p> 
<p>首先创建一个socket并返回文件描述符&#xff0c;然后设置最大并发数为8&#xff0c;其他进程可以通过这个socket通知init进程修改系统属性&#xff0c;</p> 
<p>最后注册epoll事件&#xff0c;也就是当监听到property_set_fd改变时调用handle_property_set_fd</p> 
<pre class="has"><code class="language-cpp">void StartPropertyService(Epoll* epoll) {
    property_set(&#34;ro.property_service.version&#34;, &#34;2&#34;);

    //建立socket连接
    if (auto result &#61; CreateSocket(PROP_SERVICE_NAME, SOCK_STREAM | SOCK_CLOEXEC | SOCK_NONBLOCK,
                                   false, 0666, 0, 0, {})) {
        property_set_fd &#61; *result;
    } else {
        PLOG(FATAL) &lt;&lt; &#34;start_property_service socket creation failed: &#34; &lt;&lt; result.error();
    }

    // 最大监听8个并发
    listen(property_set_fd, 8);

    // 注册property_set_fd&#xff0c;当收到句柄改变时&#xff0c;通过handle_property_set_fd来处理
    if (auto result &#61; epoll-&gt;RegisterHandler(property_set_fd, handle_property_set_fd); !result) {
        PLOG(FATAL) &lt;&lt; result.error();
    }
}</code></pre> 
<h3>6.3 handle_property_set_fd</h3> 
<p><strong>代码路径&#xff1a;</strong>platform/system/core/property_service.cpp</p> 
<p><strong>说明&#xff1a;</strong>建立socket连接&#xff0c;然后从socket中读取操作信息&#xff0c;根据不同的操作类型&#xff0c;调用HandlePropertySet做具体的操作</p> 
<p>HandlePropertySet是最终的处理函数&#xff0c;以&#34;ctl&#34;开头的key就做一些Service的Start,Stop,Restart操作&#xff0c;其他的就是调用property_set进行属性设置&#xff0c;不管是前者还是后者&#xff0c;都要进行SELinux安全性检查&#xff0c;只有该进程有操作权限才能执行相应操作</p> 
<p> </p> 
<pre class="has"><code class="language-cpp">static void handle_property_set_fd() {
    static constexpr uint32_t kDefaultSocketTimeout &#61; 2000; /* ms */

    // 等待客户端连接
    int s &#61; accept4(property_set_fd, nullptr, nullptr, SOCK_CLOEXEC);
    if (s &#61;&#61; -1) {
        return;
    }

    ucred cr;
    socklen_t cr_size &#61; sizeof(cr);
    // 获取连接到此socket的进程的凭据
    if (getsockopt(s, SOL_SOCKET, SO_PEERCRED, &amp;cr, &amp;cr_size) &lt; 0) {
        close(s);
        PLOG(ERROR) &lt;&lt; &#34;sys_prop: unable to get SO_PEERCRED&#34;;
        return;
    }

    // 建立socket连接
    SocketConnection socket(s, cr);
    uint32_t timeout_ms &#61; kDefaultSocketTimeout;

    uint32_t cmd &#61; 0;
    // 读取socket中的操作信息
    if (!socket.RecvUint32(&amp;cmd, &amp;timeout_ms)) {
        PLOG(ERROR) &lt;&lt; &#34;sys_prop: error while reading command from the socket&#34;;
        socket.SendUint32(PROP_ERROR_READ_CMD);
        return;
    }

    // 根据操作信息&#xff0c;执行对应处理,两者区别一个是以char形式读取&#xff0c;一个以String形式读取
    switch (cmd) {
    case PROP_MSG_SETPROP: {
        char prop_name[PROP_NAME_MAX];
        char prop_value[PROP_VALUE_MAX];

        if (!socket.RecvChars(prop_name, PROP_NAME_MAX, &amp;timeout_ms) ||
            !socket.RecvChars(prop_value, PROP_VALUE_MAX, &amp;timeout_ms)) {
          PLOG(ERROR) &lt;&lt; &#34;sys_prop(PROP_MSG_SETPROP): error while reading name/value from the socket&#34;;
          return;
        }

        prop_name[PROP_NAME_MAX-1] &#61; 0;
        prop_value[PROP_VALUE_MAX-1] &#61; 0;

        std::string source_context;
        if (!socket.GetSourceContext(&amp;source_context)) {
            PLOG(ERROR) &lt;&lt; &#34;Unable to set property &#39;&#34; &lt;&lt; prop_name &lt;&lt; &#34;&#39;: getpeercon() failed&#34;;
            return;
        }

        const auto&amp; cr &#61; socket.cred();
        std::string error;
        uint32_t result &#61; HandlePropertySet(prop_name, prop_value, source_context, cr, &amp;error);
        if (result !&#61; PROP_SUCCESS) {
            LOG(ERROR) &lt;&lt; &#34;Unable to set property &#39;&#34; &lt;&lt; prop_name &lt;&lt; &#34;&#39; from uid:&#34; &lt;&lt; cr.uid
                       &lt;&lt; &#34; gid:&#34; &lt;&lt; cr.gid &lt;&lt; &#34; pid:&#34; &lt;&lt; cr.pid &lt;&lt; &#34;: &#34; &lt;&lt; error;
        }

        break;
      }

    case PROP_MSG_SETPROP2: {
        std::string name;
        std::string value;
        if (!socket.RecvString(&amp;name, &amp;timeout_ms) ||
            !socket.RecvString(&amp;value, &amp;timeout_ms)) {
          PLOG(ERROR) &lt;&lt; &#34;sys_prop(PROP_MSG_SETPROP2): error while reading name/value from the socket&#34;;
          socket.SendUint32(PROP_ERROR_READ_DATA);
          return;
        }

        std::string source_context;
        if (!socket.GetSourceContext(&amp;source_context)) {
            PLOG(ERROR) &lt;&lt; &#34;Unable to set property &#39;&#34; &lt;&lt; name &lt;&lt; &#34;&#39;: getpeercon() failed&#34;;
            socket.SendUint32(PROP_ERROR_PERMISSION_DENIED);
            return;
        }

        const auto&amp; cr &#61; socket.cred();
        std::string error;
        uint32_t result &#61; HandlePropertySet(name, value, source_context, cr, &amp;error);
        if (result !&#61; PROP_SUCCESS) {
            LOG(ERROR) &lt;&lt; &#34;Unable to set property &#39;&#34; &lt;&lt; name &lt;&lt; &#34;&#39; from uid:&#34; &lt;&lt; cr.uid
                       &lt;&lt; &#34; gid:&#34; &lt;&lt; cr.gid &lt;&lt; &#34; pid:&#34; &lt;&lt; cr.pid &lt;&lt; &#34;: &#34; &lt;&lt; error;
        }
        socket.SendUint32(result);
        break;
      }

    default:
        LOG(ERROR) &lt;&lt; &#34;sys_prop: invalid command &#34; &lt;&lt; cmd;
        socket.SendUint32(PROP_ERROR_INVALID_CMD);
        break;
    }
}
</code></pre> 
<h2>7.第三阶段init.rc</h2> 
<p style="text-indent:33px;">当属性服务建立完成后&#xff0c;init的自身功能基本就告一段落&#xff0c;接下来需要来启动其他的进程。但是init进程如何其他其他进程呢&#xff1f;其他进程都是一个二进制文件&#xff0c;我们可以直接通过exec的命令方式来启动&#xff0c;例如 ./system/bin/init second_stage&#xff0c;来启动init进程的第二阶段。但是Android系统有那么多的Native进程&#xff0c;如果都通过传exec在代码中一个个的来执行进程&#xff0c;那无疑是一个灾难性的设计。</p> 
<p style="text-indent:33px;">在这个基础上Android推出了一个init.rc的机制&#xff0c;即类似通过读取配置文件的方式&#xff0c;来启动不同的进程。接下来我们就来看看init.rc是如何工作的。</p> 
<p style="text-indent:33px;">init.rc是一个配置文件&#xff0c;内部由Android初始化语言编写&#xff08;Android Init Language&#xff09;编写的脚本。</p> 
<p style="text-indent:33px;">init.rc在手机的目录&#xff1a;./init.rc</p> 
<p style="text-indent:33px;">init.rc主要包含五种类型语句&#xff1a;</p> 
<ul><li>Action</li><li>Command</li><li>Service</li><li>Option</li><li>Import</li></ul>
<p> <img alt="" class="has" height="452" src="https://img-blog.csdnimg.cn/20191215160534809.png?x-oss-process&#61,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpcmFuZmVuZw&#61;&#61;,size_16,color_FFFFFF,t_70" width="433" /></p> 
<h2>7.1 Action</h2> 
<p style="text-indent:33px;">动作表示了一组命令(commands)组成.动作包括一个触发器&#xff0c;决定了何时运行这个动作</p> 
<p>Action&#xff1a; 通过触发器trigger&#xff0c;即以on开头的语句来决定执行相应的service的时机&#xff0c;具体有如下时机&#xff1a;</p> 
<ul><li>on early-init; 在初始化早期阶段触发&#xff1b;</li><li>on init; 在初始化阶段触发&#xff1b;</li><li>on late-init; 在初始化晚期阶段触发&#xff1b;</li><li>on boot/charger&#xff1a; 当系统启动/充电时触发&#xff1b;</li><li>on property:&lt;key&gt;&#61;&lt;value&gt;: 当属性值满足条件时触发&#xff1b;</li></ul>
<p> </p> 
<h2>7.2 Command</h2> 
<p style="text-indent:33px;">command是action的命令列表中的命令&#xff0c;或者是service中的选项 onrestart 的参数命令,命令将在所属事件发生时被一个个地执行.</p> 
<p><strong>下面列举常用的命令</strong></p> 
<ul><li>class_start &lt;service_class_name&gt;&#xff1a; 启动属于同一个class的所有服务&#xff1b;</li><li>class_stop &lt;service_class_name&gt; : 停止指定类的服务</li><li>start &lt;service_name&gt;&#xff1a; 启动指定的服务&#xff0c;若已启动则跳过&#xff1b;</li><li>stop &lt;service_name&gt;&#xff1a; 停止正在运行的服务</li><li>setprop &lt;name&gt; &lt;value&gt;&#xff1a;设置属性值</li><li>mkdir &lt;path&gt;&#xff1a;创建指定目录</li><li>symlink &lt;target&gt; &lt;sym_link&gt;&#xff1a; 创建连接到&lt;target&gt;的&lt;sym_link&gt;符号链接&#xff1b;</li><li>write &lt;path&gt; &lt;string&gt;&#xff1a; 向文件path中写入字符串&#xff1b;</li><li>exec&#xff1a; fork并执行&#xff0c;会阻塞init进程直到程序完毕&#xff1b;</li><li>exprot &lt;name&gt; &lt;name&gt;&#xff1a;设定环境变量&#xff1b;</li><li>loglevel &lt;level&gt;&#xff1a;设置log级别</li><li>hostname &lt;name&gt; : 设置主机名</li><li>import &lt;filename&gt; &#xff1a;导入一个额外的init配置文件</li></ul>
<p> </p> 
<h3>7.3 Service</h3> 
<p style="text-indent:33px;">服务Service&#xff0c;以 service开头&#xff0c;由init进程启动&#xff0c;一般运行在init的一个子进程&#xff0c;所以启动service前需要判断对应的可执行文件是否存在。</p> 
<p style="text-indent:33px;"><strong>命令&#xff1a;</strong>service &lt;name&gt;&lt;pathname&gt; [ &lt;argument&gt; ]* &lt;option&gt; &lt;option&gt;</p> 
<p style="text-indent:33px;"> </p> 
<table><tbody><tr><td> <p>参数</p> </td><td> <p>含义</p> </td></tr><tr><td> <p>&lt;name&gt;</p> </td><td> <p>表示此服务的名称</p> </td></tr><tr><td> <p>&lt;pathname&gt;</p> </td><td> <p>此服务所在路径因为是可执行文件&#xff0c;所以一定有存储路径。</p> </td></tr><tr><td> <p>&lt;argument&gt;</p> </td><td> <p>启动服务所带的参数</p> </td></tr><tr><td> <p>&lt;option&gt;</p> </td><td> <p>对此服务的约束选项</p> </td></tr></tbody></table>
<p style="text-indent:33px;">init生成的子进程&#xff0c;定义在rc文件&#xff0c;其中每一个service在启动时会通过fork方式生成子进程。</p> 
<p><strong>例如&#xff1a;</strong> service servicemanager /system/bin/servicemanager代表的是服务名为servicemanager&#xff0c;服务执行的路径为/system/bin/servicemanager。</p> 
<p> </p> 
<p> </p> 
<h3>7.4 Options</h3> 
<p>Options是Service的可选项&#xff0c;与service配合使用</p> 
<ul><li>disabled: 不随class自动启动&#xff0c;只有根据service名才启动&#xff1b;</li><li>oneshot: service退出后不再重启&#xff1b;</li><li>user/group&#xff1a; 设置执行服务的用户/用户组&#xff0c;默认都是root&#xff1b;</li><li>class&#xff1a;设置所属的类名&#xff0c;当所属类启动/退出时&#xff0c;服务也启动/停止&#xff0c;默认为default&#xff1b;</li><li>onrestart:当服务重启时执行相应命令&#xff1b;</li><li>socket: 创建名为/dev/socket/&lt;name&gt;的socket</li><li>critical: 在规定时间内该service不断重启&#xff0c;则系统会重启并进入恢复模式</li></ul>
<p>default: 意味着disabled&#61;false&#xff0c;oneshot&#61;false&#xff0c;critical&#61;false。</p> 
<p> </p> 
<h3>7.5 import</h3> 
<p><strong>用来导入其他的rc文件</strong></p> 
<p><strong>命令&#xff1a;</strong>import &lt;filename&gt;</p> 
<p> </p> 
<h3>7.6 init.rc 解析过程</h3> 
<p><strong>7.6.1 LoadBootScripts</strong></p> 
<p><strong>代码路径&#xff1a;</strong>platform\system\core\init\init.cpp</p> 
<p><strong>说明&#xff1a;</strong>如果没有特殊配置ro.boot.init_rc&#xff0c;则解析./init.rc</p> 
<p>把/system/etc/init,/product/etc/init,/product_services/etc/init,/odm/etc/init,</p> 
<p>/vendor/etc/init 这几个路径加入init.rc之后解析的路径&#xff0c;在init.rc解析完成后&#xff0c;解析这些目录里的rc文件</p> 
<p> </p> 
<pre class="has"><code class="language-cpp">static void LoadBootScripts(ActionManager&amp; action_manager, ServiceList&amp; service_list) {
    Parser parser &#61; CreateParser(action_manager, service_list);

    std::string bootscript &#61; GetProperty(&#34;ro.boot.init_rc&#34;, &#34;&#34;);
    if (bootscript.empty()) {
        parser.ParseConfig(&#34;/init.rc&#34;);
        if (!parser.ParseConfig(&#34;/system/etc/init&#34;)) {
            late_import_paths.emplace_back(&#34;/system/etc/init&#34;);
        }
        if (!parser.ParseConfig(&#34;/product/etc/init&#34;)) {
            late_import_paths.emplace_back(&#34;/product/etc/init&#34;);
        }
        if (!parser.ParseConfig(&#34;/product_services/etc/init&#34;)) {
            late_import_paths.emplace_back(&#34;/product_services/etc/init&#34;);
        }
        if (!parser.ParseConfig(&#34;/odm/etc/init&#34;)) {
            late_import_paths.emplace_back(&#34;/odm/etc/init&#34;);
        }
        if (!parser.ParseConfig(&#34;/vendor/etc/init&#34;)) {
            late_import_paths.emplace_back(&#34;/vendor/etc/init&#34;);
        }
    } else {
        parser.ParseConfig(bootscript);
    }
}</code></pre> 
<p> </p> 
<p>Android7.0后&#xff0c;init.rc进行了拆分&#xff0c;每个服务都有自己的rc文件&#xff0c;他们基本上都被加载到/system/etc/init&#xff0c;/vendor/etc/init, /odm/etc/init等目录&#xff0c;等init.rc解析完成后&#xff0c;会来解析这些目录中的rc文件&#xff0c;用来执行相关的动作。</p> 
<p> </p> 
<p><strong>代码路径&#xff1a;</strong>platform\system\core\init\init.cpp</p> 
<p><strong>说明&#xff1a;</strong>创建Parser解析对象&#xff0c;例如service、on、import对象</p> 
<p> </p> 
<pre class="has"><code class="language-cpp">Parser CreateParser(ActionManager&amp; action_manager, ServiceList&amp; service_list) {
    Parser parser;

    parser.AddSectionParser(
            &#34;service&#34;, std::make_unique&lt;ServiceParser&gt;(&amp;service_list, subcontexts, std::nullopt));
    parser.AddSectionParser(&#34;on&#34;, std::make_unique&lt;ActionParser&gt;(&amp;action_manager, subcontexts));
    parser.AddSectionParser(&#34;import&#34;, std::make_unique&lt;ImportParser&gt;(&amp;parser));

    return parser;
}</code></pre> 
<p><strong>7.6.2 执行Action动作</strong></p> 
<p>按顺序把相关Action加入触发器队列&#xff0c;按顺序为 early-init -&gt; init -&gt; late-init. 然后在循环中&#xff0c;执行所有触发器队列中Action带Command的执行函数。</p> 
<p> </p> 
<pre class="has"><code class="language-cpp">am.QueueEventTrigger(&#34;early-init&#34;);
am.QueueEventTrigger(&#34;init&#34;);
am.QueueEventTrigger(&#34;late-init&#34;);
...
while (true) {
if (!(waiting_for_prop || Service::is_exec_service_running())) {
            am.ExecuteOneCommand();
        }
}</code></pre> 
<p><strong>7.6.2 Zygote启动</strong></p> 
<p>从Android 5.0的版本开始&#xff0c;Android支持64位的编译&#xff0c;因此zygote本身也支持32位和64位。通过属性ro.zygote来控制不同版本的zygote进程启动。</p> 
<p>在init.rc的import段我们看到如下代码&#xff1a;</p> 
<pre class="has"><code class="language-cpp">import /init.${ro.zygote}.rc // 可以看出init.rc不再直接引入一个固定的文件&#xff0c;而是根据属性ro.zygote的内容来引入不同的文件
</code></pre> 
<p><br />  init.rc位于/system/core/rootdir下。在这个路径下还包括四个关于zygote的rc文件。</p> 
<p>分别是init.zygote32.rc&#xff0c;init.zygote32_64.rc&#xff0c;init.zygote64.rc&#xff0c;init.zygote64_32.rc&#xff0c;由硬件决定调用哪个文件。</p> 
<p>这里拿64位处理器为例&#xff0c;<strong>init.zygote64.rc的代码如下所示</strong>&#xff1a;</p> 
<pre class="has"><code class="language-cpp">service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main		# class是一个option&#xff0c;指定zygote服务的类型为main
    	    priority -20
            user root
   	    group root readproc reserved_disk
    	    socket zygote stream 660 root system  # socket关键字表示一个option&#xff0c;创建一个名为dev/socket/zygote&#xff0c;类型为stream&#xff0c;权限为660的socket
   	    socket usap_pool_primary stream 660 root system
            onrestart write /sys/android_power/request_state wake # onrestart是一个option&#xff0c;说明在zygote重启时需要执行的command
    	    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    onrestart restart wificond
    writepid /dev/cpuset/foreground/tasks</code></pre> 
<p> </p> 
<p>service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server 解析&#xff1a;</p> 
<p><strong>service zygote &#xff1a;</strong>init.zygote64.rc 中定义了一个zygote服务。 init进程就是通过这个service名称来创建zygote进程</p> 
<p>/system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server解析&#xff1a;</p> 
<p>zygote这个服务&#xff0c;通过执行进行/system/bin/app_process64 并传入4个参数进行运行&#xff1a;</p> 
<ul><li>参数1&#xff1a;-Xzygote 该参数将作为虚拟机启动时所需的参数</li><li>参数2&#xff1a;/system/bin 代表虚拟机程序所在目录</li><li>参数3&#xff1a;--zygote 指明以ZygoteInit.java类中的main函数作为虚拟机执行入口</li><li>参数4&#xff1a;--start-system-server 告诉Zygote进程启动systemServer进程</li></ul>
<h2>8.总结</h2> 
<p style="text-indent:33px;">init进程第一阶段做的主要工作是挂载分区,创建设备节点和一些关键目录,初始化日志输出系统,启用SELinux安全策略。</p> 
<p style="text-indent:33px;">init进程第二阶段主要工作是初始化属性系统&#xff0c;解析SELinux的匹配规则&#xff0c;处理子进程终止信号&#xff0c;启动系统属性服务&#xff0c;可以说每一项都很关键&#xff0c;如果说第一阶段是为属性系统&#xff0c;SELinux做准备&#xff0c;那么第二阶段就是真正去把这些功能落实。</p> 
<p style="text-indent:33px;">init进行第三阶段主要是解析init.rc 来启动其他进程&#xff0c;进入无限循环&#xff0c;进行子进程实时监控。</p> 