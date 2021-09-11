---
layout:     post
title:      Android10 Installd守护进程
subtitle:   PackageManagerService底层真正干活的是installd，通过Native Binder调用
date:       2020-06-05
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android10
    - Android
    - PKMS
    - 系统服务
---

<p>PackageManagerService真正干活的是installd，通过Native Binder调用。</p>
<p>为什么需要installd守护进程？因为权限问题，PKMS只有system权限，installd却具有root权限。</p>
<p>在SystemServer中installd服务启动</p>
<h4 id="1、客服端实现"><a href="#1、客服端实现" class="headerlink" title="1、客服端实现"></a>1、客服端实现</h4>
<figure >
//启动installer服务，PKMS相关任务的执行者
// Wait for installd to finish starting up so that it has a chance to
// create critical directories such as /data/user with the appropriate
// permissions.  We need this to complete before we initialize other services.
Installer installer = mSystemServiceManager.startService(Installer.class);

 </figure>
<p>Installer代码比较简洁，主要为一些创建、删除文件等操作。</p>
<figure >
@Override
 public void onStart() {
     if (mIsolated) {
         mInstalld = null;
     } else {
         connect();
     }
 }

 private void connect() {
     IBinder binder = ServiceManager.getService("installd");
     if (binder != null) {
         try {
             //断开连接则重新连接
             binder.linkToDeath(new DeathRecipient() {
                 @Override
                 public void binderDied() {
                     Slog.w(TAG, "installd died; reconnecting");
                     connect();
                 }
             }, 0);
         } catch (RemoteException e) {
             binder = null;
         }
     }

     if (binder != null) {
         mInstalld = IInstalld.Stub.asInterface(binder);
         try {
             invalidateMounts();
         } catch (InstallerException ignored) {
         }
     } else {
         Slog.w(TAG, "installd not found; trying again");
         BackgroundThread.getHandler().postDelayed(() -&gt; {
             connect();
         }, DateUtils.SECOND_IN_MILLIS);
     }
 }
 </figure>
<h4 id="2、服务端实现"><a href="#2、服务端实现" class="headerlink" title="2、服务端实现"></a>2、服务端实现</h4><p>Android7.0后单一的init.rc文件被拆分，放在对应分区的etc/init目录中，每个服务一个rc文件，与该服务相关的触发器，操作等也定义在同一rc文件中。</p>
<p>frameworks/native/cmds/installd/Android.bp编译文件中</p>
<figure >
cc_binary {
    name: "installd",
    defaults: ["installd_defaults"],
    srcs: ["installd.cpp"],

    static_libs: ["libdiskusage"],

    init_rc: ["installd.rc"],
}

 </figure>
<p>frameworks/native/cmds/installd/installd.rc中启动installd进程</p>
<figure >
service installd /system/bin/installd
    class main
 </figure>
<p>installd.cpp主函数如下</p>
<figure >
static int installd_main(const int argc ATTRIBUTE_UNUSED, char *argv[]) {
    int ret;
    int selinux_enabled = (is_selinux_enabled() &gt; 0);

    setenv("ANDROID_LOG_TAGS", "*:v", 1);
    android::base::InitLogging(argv);

    SLOGI("installd firing up");

    union selinux_callback cb;
    cb.func_log = log_callback;
    selinux_set_callback(SELINUX_CB_LOG, cb);

    if (!initialize_globals()) {
        SLOGE("Could not initialize globals; exiting./n");
        exit(1);
    }

    if (initialize_directories() &lt; 0) {
        SLOGE("Could not create directories; exiting./n");
        exit(1);
    }

    if (selinux_enabled && selinux_status_open(true) &lt; 0) {
        SLOGE("Could not open selinux status; exiting./n");
        exit(1);
    }

    if ((ret = InstalldNativeService::start()) != android::OK) {
        SLOGE("Unable to start InstalldNativeService: %d", ret);
        exit(1);
    }

    IPCThreadState::self()-&gt;joinThreadPool();

    LOG(INFO) &lt;&lt; "installd shutting down";

    return 0;
}
 </figure>
<p>InstalldNativeService::start()将该服务发布到native层的servicemanager中</p>
<figure >
status_t InstalldNativeService::start() {
    IPCThreadState::self()-&gt;disableBackgroundScheduling(true);
    status_t ret = BinderService&lt;InstalldNativeService&gt;::publish();
    if (ret != android::OK) {
        return ret;
    }
    sp&lt;ProcessState&gt; ps(ProcessState::self());
    ps-&gt;startThreadPool();
    ps-&gt;giveThreadPoolName();
    return android::OK;
}
 </figure>
<p>BinderService<installdnativeservice>::publish()将服务添加到ServiceManager中</installdnativeservice></p>
<p>/framework/native/libs/binder/include/binder/BinderService.h</p>
<figure >
static status_t publish(bool allowIsolated = false) {
        sp&lt;IServiceManager&gt; sm(defaultServiceManager());
        将服务加入到ServiceManager中
        return sm-&gt;addService(
                String16(SERVICE::getServiceName()),
                new SERVICE(), allowIsolated);
    }
 </figure>
<p>所有的install.java中定义的功能都是通过binder调用native层的InstalldNativeService来实现</p>
<p>install.java中dexopt方法，最后是执行Native层的dexopt方法。</p>
<figure >
binder::Status InstalldNativeService::dexopt(const std::string& apkPath, int32_t uid,
        const std::unique_ptr&lt;std::string&gt;& packageName, const std::string& instructionSet,
        int32_t dexoptNeeded, const std::unique_ptr&lt;std::string&gt;& outputPath, int32_t dexFlags,
        const std::string& compilerFilter, const std::unique_ptr&lt;std::string&gt;& uuid,
        const std::unique_ptr&lt;std::string&gt;& classLoaderContext,
        const std::unique_ptr&lt;std::string&gt;& seInfo, bool downgrade, int32_t targetSdkVersion,
        const std::unique_ptr&lt;std::string&gt;& profileName,
        const std::unique_ptr&lt;std::string&gt;& dexMetadataPath,
        const std::unique_ptr&lt;std::string&gt;& compilationReason) {
    ENFORCE_UID(AID_SYSTEM);
    CHECK_ARGUMENT_UUID(uuid);
    CHECK_ARGUMENT_PATH(apkPath);
    if (packageName && *packageName != "*") {
        CHECK_ARGUMENT_PACKAGE_NAME(*packageName);
    }
    CHECK_ARGUMENT_PATH(outputPath);
    CHECK_ARGUMENT_PATH(dexMetadataPath);
    std::lock_guard&lt;std::recursive_mutex&gt; lock(mLock);

    const char* apk_path = apkPath.c_str();
    const char* pkgname = getCStr(packageName, "*");
    const char* instruction_set = instructionSet.c_str();
    const char* oat_dir = getCStr(outputPath);
    const char* compiler_filter = compilerFilter.c_str();
    const char* volume_uuid = getCStr(uuid);
    const char* class_loader_context = getCStr(classLoaderContext);
    const char* se_info = getCStr(seInfo);
    const char* profile_name = getCStr(profileName);
    const char* dm_path = getCStr(dexMetadataPath);
    const char* compilation_reason = getCStr(compilationReason);
    std::string error_msg;
    int res = android::installd::dexopt(apk_path, uid, pkgname, instruction_set, dexoptNeeded,
            oat_dir, dexFlags, compiler_filter, volume_uuid, class_loader_context, se_info,
            downgrade, targetSdkVersion, profile_name, dm_path, compilation_reason, &error_msg);
    return res ? error(res, error_msg) : ok();
}
 </figure>
<p>android::installd::dexopt操作在dexopt.cpp文件中</p>
<pre><code>int dexopt(const char* dex_path, uid_t uid, const char* pkgname, const char* instruction_set,
        int dexopt_needed, const char* oat_dir, int dexopt_flags, const char* compiler_filter,
        const char* volume_uuid, const char* class_loader_context, const char* se_info,
        bool downgrade, int target_sdk_version, const char* profile_name,
        const char* dex_metadata_path, const char* compilation_reason, std::string* error_msg) {
    CHECK(pkgname != nullptr);
    CHECK(pkgname[0] != 0);
    CHECK(error_msg != nullptr);
    CHECK_EQ(dexopt_flags &amp; ~DEXOPT_MASK, 0)
        &lt;&lt; &quot;dexopt flags contains unknown fields: &quot; &lt;&lt; dexopt_flags;
if (!validate_dex_path_size(dex_path)) {
    *error_msg = StringPrintf(&quot;Failed to validate %s&quot;, dex_path);
    return -1;
}

if (class_loader_context != nullptr &amp;&amp; strlen(class_loader_context) &gt; PKG_PATH_MAX) {
    *error_msg = StringPrintf(&quot;Class loader context exceeds the allowed size: %s&quot;,
                              class_loader_context);
    LOG(ERROR) &lt;&lt; *error_msg;
    return -1;
}

bool is_public = (dexopt_flags &amp; DEXOPT_PUBLIC) != 0;
bool debuggable = (dexopt_flags &amp; DEXOPT_DEBUGGABLE) != 0;
bool boot_complete = (dexopt_flags &amp; DEXOPT_BOOTCOMPLETE) != 0;
bool profile_guided = (dexopt_flags &amp; DEXOPT_PROFILE_GUIDED) != 0;
bool is_secondary_dex = (dexopt_flags &amp; DEXOPT_SECONDARY_DEX) != 0;
bool background_job_compile = (dexopt_flags &amp; DEXOPT_IDLE_BACKGROUND_JOB) != 0;
bool enable_hidden_api_checks = (dexopt_flags &amp; DEXOPT_ENABLE_HIDDEN_API_CHECKS) != 0;
bool generate_compact_dex = (dexopt_flags &amp; DEXOPT_GENERATE_COMPACT_DEX) != 0;
bool generate_app_image = (dexopt_flags &amp; DEXOPT_GENERATE_APP_IMAGE) != 0;

// Check if we&apos;re dealing with a secondary dex file and if we need to compile it.
std::string oat_dir_str;
if (is_secondary_dex) {
    if (process_secondary_dex_dexopt(dex_path, pkgname, dexopt_flags, volume_uuid, uid,
            instruction_set, compiler_filter, &amp;is_public, &amp;dexopt_needed, &amp;oat_dir_str,
            downgrade, class_loader_context, error_msg)) {
        oat_dir = oat_dir_str.c_str();
        if (dexopt_needed == NO_DEXOPT_NEEDED) {
            return 0;  // Nothing to do, report success.
        }
    } else {
        if (error_msg-&gt;empty()) {  // TODO: Make this a CHECK.
            *error_msg = &quot;Failed processing secondary.&quot;;
        }
        return -1;  // We had an error, logged in the process method.
    }
} else {
    // Currently these flags are only use for secondary dex files.
    // Verify that they are not set for primary apks.
    CHECK((dexopt_flags &amp; DEXOPT_STORAGE_CE) == 0);
    CHECK((dexopt_flags &amp; DEXOPT_STORAGE_DE) == 0);
}

// Open the input file.
unique_fd input_fd(open(dex_path, O_RDONLY, 0));
if (input_fd.get() &lt; 0) {
    *error_msg = StringPrintf(&quot;installd cannot open &apos;%s&apos; for input during dexopt&quot;, dex_path);
    LOG(ERROR) &lt;&lt; *error_msg;
    return -1;
}

// Create the output OAT file.
char out_oat_path[PKG_PATH_MAX];
Dex2oatFileWrapper out_oat_fd = open_oat_out_file(dex_path, oat_dir, is_public, uid,
        instruction_set, is_secondary_dex, out_oat_path);
if (out_oat_fd.get() &lt; 0) {
    *error_msg = &quot;Could not open out oat file.&quot;;
    return -1;
}

// Open vdex files.
Dex2oatFileWrapper in_vdex_fd;
Dex2oatFileWrapper out_vdex_fd;
if (!open_vdex_files_for_dex2oat(dex_path, out_oat_path, dexopt_needed, instruction_set,
        is_public, uid, is_secondary_dex, profile_guided, &amp;in_vdex_fd, &amp;out_vdex_fd)) {
    *error_msg = &quot;Could not open vdex files.&quot;;
    return -1;
}

// Ensure that the oat dir and the compiler artifacts of secondary dex files have the correct
// selinux context (we generate them on the fly during the dexopt invocation and they don&apos;t
// fully inherit their parent context).
// Note that for primary apk the oat files are created before, in a separate installd
// call which also does the restorecon. TODO(calin): unify the paths.
if (is_secondary_dex) {
    if (selinux_android_restorecon_pkgdir(oat_dir, se_info, uid,
            SELINUX_ANDROID_RESTORECON_RECURSE)) {
        *error_msg = std::string(&quot;Failed to restorecon &quot;).append(oat_dir);
        LOG(ERROR) &lt;&lt; *error_msg;
        return -1;
    }
}

// Create a swap file if necessary.
unique_fd swap_fd = maybe_open_dexopt_swap_file(out_oat_path);

// Create the app image file if needed.
Dex2oatFileWrapper image_fd = maybe_open_app_image(
        out_oat_path, generate_app_image, is_public, uid, is_secondary_dex);

// Open the reference profile if needed.
Dex2oatFileWrapper reference_profile_fd = maybe_open_reference_profile(
        pkgname, dex_path, profile_name, profile_guided, is_public, uid, is_secondary_dex);

unique_fd dex_metadata_fd;
if (dex_metadata_path != nullptr) {
    dex_metadata_fd.reset(TEMP_FAILURE_RETRY(open(dex_metadata_path, O_RDONLY | O_NOFOLLOW)));
    if (dex_metadata_fd.get() &lt; 0) {
        PLOG(ERROR) &lt;&lt; &quot;Failed to open dex metadata file &quot; &lt;&lt; dex_metadata_path;
    }
}

LOG(VERBOSE) &lt;&lt; &quot;DexInv: --- BEGIN &apos;&quot; &lt;&lt; dex_path &lt;&lt; &quot;&apos; ---&quot;;

pid_t pid = fork();
if (pid == 0) {
    /* child -- drop privileges before continuing */
    drop_capabilities(uid);

    SetDex2OatScheduling(boot_complete);
    if (flock(out_oat_fd.get(), LOCK_EX | LOCK_NB) != 0) {
        PLOG(ERROR) &lt;&lt; &quot;flock(&quot; &lt;&lt; out_oat_path &lt;&lt; &quot;) failed&quot;;
        _exit(DexoptReturnCodes::kFlock);
    }

    run_dex2oat(input_fd.get(),
                out_oat_fd.get(),
                in_vdex_fd.get(),
                out_vdex_fd.get(),
                image_fd.get(),
                dex_path,
                out_oat_path,
                swap_fd.get(),
                instruction_set,
                compiler_filter,
                debuggable,
                boot_complete,
                background_job_compile,
                reference_profile_fd.get(),
                class_loader_context,
                target_sdk_version,
                enable_hidden_api_checks,
                generate_compact_dex,
                dex_metadata_fd.get(),
                compilation_reason);
} else {
    int res = wait_child(pid);
    if (res == 0) {
        LOG(VERBOSE) &lt;&lt; &quot;DexInv: --- END &apos;&quot; &lt;&lt; dex_path &lt;&lt; &quot;&apos; (success) ---&quot;;
    } else {
        LOG(VERBOSE) &lt;&lt; &quot;DexInv: --- END &apos;&quot; &lt;&lt; dex_path &lt;&lt; &quot;&apos; --- status=0x&quot;
                     &lt;&lt; std::hex &lt;&lt; std::setw(4) &lt;&lt; res &lt;&lt; &quot;, process failed&quot;;
        *error_msg = format_dexopt_error(res, dex_path);
        return res;
    }
}

update_out_oat_access_times(dex_path, out_oat_path);

// We&apos;ve been successful, don&apos;t delete output.
out_oat_fd.SetCleanup(false);
out_vdex_fd.SetCleanup(false);
image_fd.SetCleanup(false);
reference_profile_fd.SetCleanup(false);

return 0;
}
</code></pre><h3 id="dexopt"><a href="#dexopt" class="headerlink" title="dexopt"></a>dexopt</h3><p><img src="/2019/installd守护进程/dexopt.png" alt=""></p>
<p>dexopt 针对 Dalvik 虚拟机，dex2oat 后者针对 Art 虚拟机(Android4.4)。</p>
<ul>
<li>dexopt 是对 dex 文件 进行 verification 和 optimization 的操作，其对 dex 文件的优化结果变成了 odex 文件，这个文件和 dex 文件很像，只是使用了一些优化操作码（譬如优化调用虚拟指令等）。</li>
<li>dex2oat 是对 dex 文件的 AOT 提前编译操作，其需要一个 dex 文件，然后对其进行编译，结果是一个本地可执行的 ELF 文件，可以直接被本地处理器执行。</li>
</ul>
<p>除此之外在上图还可以看到 Dalvik 虚拟机中有使用 JIT 编译器，也就是说其也能将程序运行的热点 java 字节码编译成本地 code 执行，所以其与 Art 虚拟机还是有区别的。Art 虚拟机的 dex2oat 是提前编译所有 dex 字节码，而 Dalvik 虚拟机只编译使用启发式检测中最频繁执行的热点字节码。</p>
<p>在系统首次启动的场景中，系统会对/system/app、/system/priv-app、/data/app目录下的所有APK进行dex字节码到本地机器码的翻译，同样也会对/system/framework目录下的APK或者JAR文件，以及这些APK所引用的外部JAR，进行dex字节码到本地机器码的翻译。这样可以保证除了应用之外，系统中使用java来开发的系统服务，也会统一地从dex字节码翻译成本地机器码。详细内容请移步老罗的博客<a href="https://blog.csdn.net/luoshengyang/article/details/18006645" target="_blank" rel="noopener">Android ART运行时无缝替换Dalvik虚拟机的过程分析</a>。</p>
<h4 id="1、JVM、DVM、ART虚拟机了解"><a href="#1、JVM、DVM、ART虚拟机了解" class="headerlink" title="1、JVM、DVM、ART虚拟机了解"></a>1、JVM、DVM、ART虚拟机了解</h4><p>JVM虚拟机运行的是java字节码： java-&gt;java bytecode（class）-&gt;java bytecode（jar） 注！java虚拟机基于栈，基于栈的机器必须使用指令来载入和操作栈上的数据，所需指令相对来说比较多。</p>
<p>Dalvik虚拟机解释执行的dex字节码： java-&gt;java bytecode（class）-&gt;dalvik bytecode（dex） 注：相对JVM，Dalvik基于寄存器，且经过优化并允许有限的内存中同时运行多个虚拟机实例，每个Dalvik应用作为一个独立的Linxu进程执行。</p>
<p>如果一个应用中有很多类，编译后会相应生成很多class文件，class文件之间也会有不少冗余信息，dex格式文件把所有classs文件内容整合到一个文件，这样可以减少整体文件占用，IO操作，同时也提高了类的查找速度。此外，dex格式文件增加了新的操作码支持，文件结构也相对简洁，使用等长的指令来提高解析速度。而且dex文件会尽量扩大只读结构的大小，来提高进程间数据共享的速度。</p>
<p>ART虚拟机执行的本地机器码： java-&gt;java bytecode（class）-&gt;dalvik bytecode（dex）-&gt;optimized android runtime machine code（oat） 注：ART所使用的AOT（Ahead-Of-Time）编译，在应用首次安装时，字节码预编译成机器码存储在本地，也就是说在程序运行前编译。而Dalvik是典型的JIT（Just_In_Time），此模式下，应用<strong>每次</strong>运行的时候，字节码都需要即时编译器转换为机器码再执行，也就是在程序运行时编译。因此在App运行时，ART模式相对于Dalvik省去了解释字节码的过程，占用内存也相应减少，进而提高App的运行效率。</p>
<h4 id="2、Odex"><a href="#2、Odex" class="headerlink" title="2、Odex"></a>2、Odex</h4><p>从上面一节中我们知道，在编译打包APK时，Java类会被编译成一个或者多个字节码文件（.class），通过dx工具CLASS文件转换成一个DEX（Dalvik Executable）文件。 通常情况下，我们看到的Android应用程序实际上是一个以.apk为后缀名的压缩文件。我们可以通过压缩工具对apk进行解压，解压出来的内容中有一个名为classes.dex的文件。那么我们首次开机的时候系统需要将其从apk中解压出来保存在data/app目录中。 </p>
<p>如果当前运行在Dalvik虚拟机下，Dalvik会对classes.dex进行一次“翻译”，“翻译”的过程也就是守护进程installd的函数dexopt来对dex字节码进行优化，实际上也就是由dex文件生成odex文件，最终odex文件被保存在手机的VM缓存目录data/dalvik-cache下（注意！这里所生成的odex文件依旧是以dex为后缀名，格式如：system@priv-app@<a href="mailto:Settings@Settings.apk" target="_blank" rel="noopener">Settings@Settings.apk</a>@classes.dex）。 </p>
<p>如果当前运行于Art模式下， Art同样会在首次进入系统的时候调用/system/bin/dexopt工具来将dex字节码翻译成本地机器码，保存在data/dalvik-cache下。 那么这里需要注意的是，无论是对dex字节码进行优化，还是将dex字节码翻译成本地机器码，最终得到的结果都是保存在相同名称的一个odex文件里面的，但是前者对应的是一个dey文件（表示这是一个优化过的dex），后者对应的是一个oat文件（实际上是一个自定义的elf文件，里面包含的都是本地机器指令）。通过这种方式，原来任何通过绝对路径引用了该odex文件的代码就都不需要修改了。</p>
<p>由于在系统首次启动时会对应用进行安装，那么在预置APK比较多的情况下，将会大大增加系统首次启动的时间。从前面的描述可知，既然无论是DVM还是ART，对DEX的优化结果都是保存在一个相同名称的odex文件，那么如果我们把这两个过程在ROM编译的时候预处理提取Odex文件将会大大优化系统首次启动的时间。</p>
<h4 id="3、预编译提取Odex"><a href="#3、预编译提取Odex" class="headerlink" title="3、预编译提取Odex"></a>3、预编译提取Odex</h4><p>在目录/build/core/dex_preopt_odex_install.mk中的代码：</p>
<figure >
ifeq ($(LOCAL_MODULE),helloworld)
LOCAL_DEX_PREOPT:=
endif
build_odex:=
installed_odex:=
 </figure>
<p>helloworld可替换为需要跳过提取odex的apk的LOCAL_MODULE名字，如Settings等。</p>
<h3 id="odex优化的地方"><a href="#odex优化的地方" class="headerlink" title="odex优化的地方"></a>odex优化的地方</h3><h4 id="1-首次开机或者升级"><a href="#1-首次开机或者升级" class="headerlink" title="1.首次开机或者升级"></a>1.首次开机或者升级</h4><p>在SystemServer.java 中有mPackageManagerService.updatePackagesIfNeeded()</p>
<figure >
updatePackagesIfNeeded-&gt;performDexOptUpgrade-&gt;performDexOptTraced-&gt;performDexOptInternal-&gt;performDexOptInternalWithDependenciesLI-&gt;PackageDexOptimizer.performDexOpt-&gt;performDexOptLI-&gt;dexOptPath-&gt;Installer.dexopt-&gt;InstalldNativeService.dexopt-&gt;dexopt.dexopt

 </figure>
<h4 id="2-安装应用"><a href="#2-安装应用" class="headerlink" title="2.安装应用"></a>2.安装应用</h4><p>在PKMS.installPackageLI函数中有：</p>
<figure >
mPackageDexOptimizer.performDexOpt(pkg, pkg.usesLibraryFiles,
null /* instructionSets /, false / checkProfiles */,
getCompilerFilterForReason(REASON_INSTALL),
getOrCreateCompilerPackageStats(pkg),
mDexManager.isUsedByOtherApps(pkg.packageName));

 </figure>
<h4 id="3-IPackageManager-aidl提供了performDexOpt方法"><a href="#3-IPackageManager-aidl提供了performDexOpt方法" class="headerlink" title="3.IPackageManager.aidl提供了performDexOpt方法"></a>3.IPackageManager.aidl提供了performDexOpt方法</h4><p>在PKMS中有实现的地方，但是没找到调用的地方</p>
<h4 id="4-IPackageManager-aidl提供了performDexOptMode方法"><a href="#4-IPackageManager-aidl提供了performDexOptMode方法" class="headerlink" title="4.IPackageManager.aidl提供了performDexOptMode方法"></a>4.IPackageManager.aidl提供了performDexOptMode方法</h4><p>在PKMS中有实现的地方，在PackageManagerShellCommand中会被调用，应该是提供给shell命令调用</p>
<h4 id="5-OTA升级后"><a href="#5-OTA升级后" class="headerlink" title="5.OTA升级后"></a>5.OTA升级后</h4><p>在SystemServer.java 中有OtaDexoptService.main(mSystemContext, mPackageManagerService);</p>
<pre><code>public static OtaDexoptService main(Context context,
        PackageManagerService packageManagerService) {
    OtaDexoptService ota = new OtaDexoptService(context, packageManagerService);
    ServiceManager.addService(&quot;otadexopt&quot;, ota);
// Now it&apos;s time to check whether we need to move any A/B artifacts.
ota.moveAbArtifacts(packageManagerService.mInstaller);
return ota;

private void moveAbArtifacts(Installer installer) {
    if (mDexoptCommands != null) {
        throw new IllegalStateException(&quot;Should not be ota-dexopting when trying to move.&quot;);
    }
    //如果不是升级上来的，就return掉
    if (!mPackageManagerService.isUpgrade()) {
        Slog.d(TAG, &quot;No upgrade, skipping A/B artifacts check.&quot;);
        return;
    }
installer.moveAb(path, dexCodeInstructionSet, oatDir);
</code></pre><p>moveAbArtifacts函数的逻辑：<br>1.判断是否升级<br>2.判断扫描过的package是否有code，没有则跳过<br>3.判断package的code路径是否为空，为空则跳过<br>4.如果package的code在system或者vendor目录下，跳过<br>5.满足上述条件，调用Installer.java中的moveAb方法<br>最终是调用dexopt.cpp的move_ab方法</p>
<p>OtaDexoptService也提供给shell命令一些方法来调用</p>
<h4 id="6-在系统空闲的时候"><a href="#6-在系统空闲的时候" class="headerlink" title="6.在系统空闲的时候"></a>6.在系统空闲的时候</h4><p>是通过BackgroundDexOptService来实现的，BackgroundDexOptService继承了JobService<br>这里启动了两个任务<br>1.开机的时候执行odex优化 JOB_POST_BOOT_UPDATE<br>执行条件：开机一分钟内<br>2.在系统休眠的时候执行优化 JOB_IDLE_OPTIMIZE<br>执行条件：设备处于空闲，插入充电器，且每隔一分钟或者一天就检查一次（根据debug开关控制）</p>
<figure >
private static final long IDLE_OPTIMIZATION_PERIOD = DEBUG
          ? TimeUnit.MINUTES.toMillis(1)
          : TimeUnit.DAYS.toMillis(1);
          
public static void schedule(Context context) {
      if (isBackgroundDexoptDisabled()) {
          return;
      }

      JobScheduler js = (JobScheduler) context.getSystemService(Context.JOB_SCHEDULER_SERVICE);

      // Schedule a one-off job which scans installed packages and updates
      // out-of-date oat files.
      js.schedule(new JobInfo.Builder(JOB_POST_BOOT_UPDATE, sDexoptServiceName)
                  .setMinimumLatency(TimeUnit.MINUTES.toMillis(1))
                  .setOverrideDeadline(TimeUnit.MINUTES.toMillis(1))
                  .build());

      // Schedule a daily job which scans installed packages and compiles
      // those with fresh profiling data.
      js.schedule(new JobInfo.Builder(JOB_IDLE_OPTIMIZE, sDexoptServiceName)
                  .setRequiresDeviceIdle(true)
                  .setRequiresCharging(true)
                  .setPeriodic(IDLE_OPTIMIZATION_PERIOD)
                  .build());

      if (DEBUG_DEXOPT) {
          Log.i(TAG, "Jobs scheduled");
      }
  } </figure>
<h3 id="判断是否需要做dex2oat的逻辑："><a href="#判断是否需要做dex2oat的逻辑：" class="headerlink" title="判断是否需要做dex2oat的逻辑："></a>判断是否需要做dex2oat的逻辑：</h3><h4 id="1）是否需要编译的类型分类："><a href="#1）是否需要编译的类型分类：" class="headerlink" title="1）是否需要编译的类型分类："></a>1）是否需要编译的类型分类：</h4>
<figure >
class OatFileAssistant {
//是否需要编译
enum DexOptNeeded {
 kNoDexOptNeeded = 0, //已经编译过，不需要再编译
 kDex2OatFromScratch = 1, //有dex文件，但还没编过
 kDex2OatForBootImage = 2,//oat文件不能匹配boot image（系统升级 boot image会变化）
 kDex2OatForFilter = 3,//oat文件不能匹配compiler filter
 kDex2OatForRelocation = 4, //还是oat文件与boot image不匹配，但是没有深刻理解relocation是什么场景
 }

 //对应的几种状态
enum OatStatus {
 kOatCannotOpen, //oat文件不存在
 kOatDexOutOfDate, //oat文件过期，与dex文件不匹配
 kOatBootImageOutOfDate, //对应kDex2OatForBootImage，oat文件与boot image不匹配
 kOatRelocationOutOfDate,//对应kDex2OatForRelocation oat文件与boot image不匹配
 kOatUpToDate,//oat文件与dex文件和 boot image都匹配
 };
} </figure>
<p>核心逻辑：</p>
<figure >
art/runtime/oat_file_assistant.cc
 
int OatFileAssistant::GetDexOptNeeded(CompilerFilter::Filter target,
                                      bool profile_changed,
                                      bool downgrade,
                                      ClassLoaderContext* class_loader_context) {
  OatFileInfo& info = GetBestInfo();//获取OatFileInfo对应实例
  DexOptNeeded dexopt_needed = info.GetDexOptNeeded(target,
                                                    profile_changed,
                                                    downgrade,
                                                    class_loader_context);
  if (info.IsOatLocation() || dexopt_needed == kDex2OatFromScratch) {
    return dexopt_needed;
  }
  return -dexopt_needed;
} </figure>
<p>这里有两个概念需要了解：</p>
<ul>
<li>oat location 与odex location 分别是什么？<br>app的安装系统目录data/app和system/app，这个路径下每个应用都会生成一个类似包名+乱码的一个文件夹，里面存放主apk以及编译文件。<br>oat location对应的是oat文件夹路径<br>odex location对应的是oat/arm or arm64/odex文件路径<br> 如果有odex优先用odex。</li>
<li>正负数是指的什么？<br> 正数对应in_odex_path ，负数对应out_oat_path</li>
</ul>
<h4 id="2）DexOptNeeded各类型赋值"><a href="#2）DexOptNeeded各类型赋值" class="headerlink" title="2）DexOptNeeded各类型赋值"></a>2）DexOptNeeded各类型赋值</h4><p>这里主要是看看这几个判断类型是在哪赋值的，这样就知道编译的触发条件有哪些了</p>
<figure >
if (!oat_file_assistant.IsUpToDate()) { 
 switch (oat_file_assistant.MakeUpToDate(/*profile_changed*/false, /*out*/ &error_msg)) {
...
} </figure>
<p>过期逻辑一般是先IsUpToDate判断是否过期，然后MakeUpToDate做过期操作，很明显这个部分还是在oat_file_assistant.cc做的</p>
<figure >
art/runtime/oat_file_assistant.cc

bool OatFileAssistant::IsUpToDate() {
  return GetBestInfo().Status() == kOatUpToDate;//是不是已经编过了
} </figure>
<p>没有编过就通过MakeUpToDate来置DexOptNeeded编译类型</p>
<figure >
OatFileAssistant::MakeUpToDate(bool profile_changed, std::string* error_msg) {
  CompilerFilter::Filter target;
  if (!GetRuntimeCompilerFilterOption(&target, error_msg)) {
    return kUpdateNotAttempted; //We wanted to update the code, but determined we should not make the attempt.
  }
  OatFileInfo& info = GetBestInfo();
  switch (info.GetDexOptNeeded(target, profile_changed)) { //这里有各种条件来赋值DexOptNeeded，条件跟之前的描述差不多

    case kNoDexOptNeeded:
      return kUpdateSucceeded;//We successfully made the code up to date (possibly by doing nothing).

    // TODO: For now, don't bother with all the different ways we can call
 // dex2oat to generate the oat file. Always generate the oat file as if it
 // were kDex2OatFromScratch.
    case kDex2OatFromScratch:
    case kDex2OatForBootImage:
    case kDex2OatForRelocation:
    case kDex2OatForFilter:
      return GenerateOatFileNoChecks(info, target, error_msg);//mark the odex file has changed and we should try to reload.

  }
  UNREACHABLE();
} </figure>
<p>主要赋值在GetBestInfo()</p>
<figure >
OatFileAssistant::OatFileInfo& OatFileAssistant::GetBestInfo() {
  // TODO(calin): Document the side effects of class loading when
  // running dalvikvm command line.
  if (dex_parent_writable_) {
    // If the parent of the dex file is writable it means that we can
    // create the odex file. In this case we unconditionally pick the odex
    // as the best oat file. This corresponds to the regular use case when
    // apps gets installed or when they load private, secondary dex file.
    // For apps on the system partition the odex location will not be
    // writable and thus the oat location might be more up to date.
    return odex_;
  }

  // We cannot write to the odex location. This must be a system app.

  // If the oat location is usable take it.
  if (oat_.IsUseable()) {
    return oat_;
  }

  // The oat file is not usable but the odex file might be up to date.
  // This is an indication that we are dealing with an up to date prebuilt
  // (that doesn't need relocation).
  if (odex_.Status() == kOatUpToDate) {
    return odex_;
  }

  // The oat file is not usable and the odex file is not up to date.
  // However we have access to the original dex file which means we can make
  // the oat location up to date.
  if (HasOriginalDexFiles()) {
    return oat_;
  }

  // We got into the worst situation here:
  // - the oat location is not usable
  // - the prebuild odex location is not up to date
  // - and we don't have the original dex file anymore (stripped).
  // Pick the odex if it exists, or the oat if not.
  return (odex_.Status() == kOatCannotOpen) ? oat_ : odex_;
} </figure>
<h3 id="ART-如何编译-DEX"><a href="#ART-如何编译-DEX" class="headerlink" title="ART 如何编译 DEX"></a>ART 如何编译 DEX</h3><p>ART 如何编译 DEX 代码还有个compile filter以参数的形式来决定：从 Android O 开始，有四个官方支持的过滤器：</p>
<ul>
<li><strong>verify</strong>：只运行 DEX 代码验证。</li>
<li><strong>quicken</strong>：运行 DEX 代码验证，并优化一些 DEX 指令，以获得更好的解释器性能。</li>
<li><strong>speed-profile</strong>：运行 DEX 代码验证，并对配置文件中列出的方法进行 AOT 编译。</li>
<li><strong>speed</strong>：运行 DEX 代码验证，并对所有方法进行 AOT 编译。</li>
</ul>
<p>verify 和quicken 他俩都没执行编译，之后代码执行需要跑解释器。而speed-profile 和 speed 都执行了编译，区别是speed-profile根据profile记录的热点函数来编译，属于部分编译，而speed属于全编。</p>
<p>执行效率上：<br>verify &lt; quicken &lt; speed-profile &lt; speed</p>
<p>编译速度上：<br>verify &gt; quicken &gt; speed-profile &gt; speed</p>
<figure >
[pm.dexopt.ab-ota]: [speed-profile]
[pm.dexopt.bg-dexopt]: [speed-profile]
[pm.dexopt.boot]: [verify]
[pm.dexopt.first-boot]: [quicken]
[pm.dexopt.inactive]: [verify]
[pm.dexopt.install]: [speed-profile]
[pm.dexopt.priv-apps-oob]: [false]
[pm.dexopt.priv-apps-oob-list]: [ALL]
[pm.dexopt.shared]: [speed]
 </figure>
<p>启动时间相关<br> 主要还是看执行模式</p>
<ul>
<li>Android大版本之间相同场景的执行模式是否有区别， dex2oat编译的时候会耗时，并且是多线程的，cpu占用率也会比较高。</li>
<li>方法执行是走的解释模式还是执行机器码，执行时间会有差别。</li>
</ul>
<p>处理方法<br> 如果应用running时间差距比较大，则可以将应用强制按speed进行编译，对比执行效率差异。<br>adb shell cmd package compile -c -f -m speed &lt;包名&gt;<br>应用编译成speed模式后，理论上同平台的机器对比，相同阶段（比如同一个inflate）耗时应该接近。若按speed编译后还存在差异，那就得看是否是其他方面的问题了。</p>
