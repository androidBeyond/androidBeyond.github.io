---
layout:     post
title:      Android10 Binder机制3-注册服务
subtitle:   ServiceManger是Binder IPC通信过程中的守护进程，是一个具体的服务，其功能主要是查询和注册服务。
date:       2021-02-26
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android10
    - Android
    - Binder
    - 进程间通信
---
 
 <h2 id="一、概述"><a href="#一、概述" class="headerlink" title="一、概述"></a>一、概述</h2><p>本文将分析系统服务的注册流程，如注册ActivityManagerService时，通过ServiceManager中的静态方法addService注册具体的服务。ServiceManger是Binder IPC通信过程中的守护进程，是一个具体的服务，其功能主要是查询和注册服务。</p>
<h2 id="二、ServiceManager启动"><a href="#二、ServiceManager启动" class="headerlink" title="二、ServiceManager启动"></a>二、ServiceManager启动</h2><p>ServiceManager是由init进程通过servicemanager.rc文件而创建，其所在的可执行文件在system/bin/servicemanager，对应的源文件是service_manager.c。</p>

<pre><code>
service servicemanager /system/bin/servicemanager
    class core animation
    user system
    group system readproc
    critical
    onrestart restart healthd
    onrestart restart zygote
    onrestart restart audioserver
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart inputflinger
    onrestart restart drm
    onrestart restart cameraserver
    onrestart restart keystore
    onrestart restart gatekeeperd
    writepid /dev/cpuset/system-background/tasks
    shutdown critical
</code></pre>

<p>启动ServiceManager的入口时文件中的main方法.</p>
<h3 id="2-1-mian"><a href="#2-1-mian" class="headerlink" title="2.1 mian"></a>2.1 mian</h3><p>[-&gt;service_manager.c]</p>

<pre><code>
int main(int argc, char** argv)
{
    struct binder_state *bs;
    union selinux_callback cb;
    char *driver;

    if (argc &gt; 1) {
        driver = argv[1];
    } else {
        driver = "/dev/binder";
    }
    //见2.2节，打开binder驱动，申请128K字节大小的内存空间
    bs = binder_open(driver, 128*1024);
    if (!bs) {
#ifdef VENDORSERVICEMANAGER
        ALOGW("failed to open binder driver %s/n", driver);
        while (true) {
            sleep(UINT_MAX);
        }
#else
        ALOGE("failed to open binder driver %s/n", driver);
#endif
        return -1;
    }
    //见2.3节，成为上下文管理者
    if (binder_become_context_manager(bs)) {
        ALOGE("cannot become context manager (%s)/n", strerror(errno));
        return -1;
    }

    cb.func_audit = audit_callback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);
    cb.func_log = selinux_log_callback;
    selinux_set_callback(SELINUX_CB_LOG, cb);

#ifdef VENDORSERVICEMANAGER
    sehandle = selinux_android_vendor_service_context_handle();
#else
    sehandle = selinux_android_service_context_handle();
#endif
    selinux_status_open(true);

    if (sehandle == NULL) {
        ALOGE("SELinux: Failed to acquire sehandle. Aborting./n");
        abort();
    }

    if (getcon(&service_manager_context) != 0) {
        ALOGE("SELinux: Failed to acquire service_manager context. Aborting./n");
        abort();
    }

    //见2.4节，进入无限循环，处理client发送过来的消息
    binder_loop(bs, svcmgr_handler);

    return 0;
}</code></pre>

<h3 id="2-2-binder-open"><a href="#2-2-binder-open" class="headerlink" title="2.2 binder_open"></a>2.2 binder_open</h3><p>[-&gt;servicemanager/binder.c]</p>

<pre><code>
struct binder_state *binder_open(const char* driver, size_t mapsize)
{
    struct binder_state *bs;
    struct binder_version vers;

    bs = malloc(sizeof(*bs));
    if (!bs) {
        errno = ENOMEM;
        return NULL;
    }
    //通过系统调用陷入内核，打开Binder设备驱动
    bs-&gt;fd = open(driver, O_RDWR | O_CLOEXEC);
    if (bs-&gt;fd &lt; 0) {
        fprintf(stderr,"binder: cannot open %s (%s)/n",
                driver, strerror(errno));
        goto fail_open;
    }
    //通过系统调用，ioctl获取binder版本信息
    if ((ioctl(bs-&gt;fd, BINDER_VERSION, &vers) == -1) ||
        (vers.protocol_version != BINDER_CURRENT_PROTOCOL_VERSION)) {
        fprintf(stderr,
                "binder: kernel driver version (%d) differs from user space version (%d)/n",
                vers.protocol_version, BINDER_CURRENT_PROTOCOL_VERSION);
        goto fail_open; //内核空间与用户空间的binder版本不是同一个版本
    }

    bs-&gt;mapsize = mapsize;
    //通过系统调用，mmap内存映射，必须是page的整数倍
    bs-&gt;mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs-&gt;fd, 0);
    if (bs-&gt;mapped == MAP_FAILED) {
        fprintf(stderr,"binder: cannot map device (%s)/n",
                strerror(errno));
        goto fail_map;
    }

    return bs;

fail_map:
    close(bs-&gt;fd);
fail_open:
    free(bs);
    return NULL;
}</code></pre>

<ul>
<li><p>调用open打开binder驱动设备，open方法经过系统调用，进入binder驱动，然后调用binder_open方法，该方法会在binder驱动层创建一个binder_proc对象，再将binder_proc对象赋值给fd-&gt;private_data，同时放入全局链表binder_procs。</p>
</li>
<li><p>再通过ioctl检查当前binder版本与binder驱动层的版本是否一致。</p>
</li>
<li><p>调用mmap进行内存映射，通过系统调用，对应于binder驱动层的binder_mmap方法，该方法会在binder驱动层创建Binder_buffer对象，并放入当前的binder_proc的proc-&gt;buffers链表。</p>
</li>
</ul>
<h3 id="2-3-binder-become-context-manager"><a href="#2-3-binder-become-context-manager" class="headerlink" title="2.3 binder_become_context_manager"></a>2.3 binder_become_context_manager</h3><p>[-&gt;servicemanager/binder.c]</p>

<pre><code>
int binder_become_context_manager(struct binder_state *bs)
{
    return ioctl(bs-&gt;fd, BINDER_SET_CONTEXT_MGR, 0);
}</code></pre>

<p>成为上下文管理者，整个系统只有一个这样的管理者，经过系统调用binder驱动层的binder_ioctl方法。</p>
<h3 id="2-4-binder-loop"><a href="#2-4-binder-loop" class="headerlink" title="2.4 binder_loop"></a>2.4 binder_loop</h3><p>[-&gt;servicemanager/binder.c]</p>

<pre><code>
void binder_loop(struct binder_state *bs, binder_handler func)
{
    int res;
    struct binder_write_read bwr;
    uint32_t readbuf[32];

    bwr.write_size = 0;
    bwr.write_consumed = 0;
    bwr.write_buffer = 0;

    readbuf[0] = BC_ENTER_LOOPER;
    //将BC_ENTER_LOOPER，发送给binder驱动，让ServiceManger进入循环
    binder_write(bs, readbuf, sizeof(uint32_t));

    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (uintptr_t) readbuf;
        //进入循环，不断的binder读写
        res = ioctl(bs-&gt;fd, BINDER_WRITE_READ, &bwr);

        if (res &lt; 0) {
            ALOGE("binder_loop: ioctl failed (%s)/n", strerror(errno));
            break;
        }
        //解析binder信息
        res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
        if (res == 0) {
            ALOGE("binder_loop: unexpected reply?!/n");
            break;
        }
        if (res &lt; 0) {
            ALOGE("binder_loop: io error %d %s/n", res, strerror(errno));
            break;
        }
    }
}</code></pre>

<p>进入循环读写操作，main方法传递过来的参数fun指向svcmgr_handler。</p>
<h2 id="三、注册服务"><a href="#三、注册服务" class="headerlink" title="三、注册服务"></a>三、注册服务</h2><p>ServiceManager.addService(String name, IBinder service)过程</p>
<h3 id="3-1-SM-addService"><a href="#3-1-SM-addService" class="headerlink" title="3.1 SM.addService"></a>3.1 SM.addService</h3><p>[-&gt;ServiceManager.java]</p>

<pre><code>
public static void addService(String name, IBinder service) {
    addService(name, service, false, IServiceManager.DUMP_FLAG_PRIORITY_DEFAULT);
}

public static void addService(String name, IBinder service, boolean allowIsolated,
        int dumpPriority) {
    try {
        getIServiceManager().addService(name, service, allowIsolated, dumpPriority);
    } catch (RemoteException e) {
        Log.e(TAG, "error in addService", e);
    }
}</code></pre>

<h3 id="3-2-getIServiceManager"><a href="#3-2-getIServiceManager" class="headerlink" title="3.2  getIServiceManager"></a>3.2  getIServiceManager</h3><p>[-&gt;ServiceManager.java]</p>

<pre><code>
private static IServiceManager getIServiceManager() {
       if (sServiceManager != null) {
           return sServiceManager;
       }

       // Find the service manager
       sServiceManager = ServiceManagerNative
               .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
       return sServiceManager;
   }
   
    public static final native IBinder getContextObject();
	</code></pre>

<p>getContextObject通过Native方式调用。</p>
<h4 id="3-2-1-Binder-allowBlocking"><a href="#3-2-1-Binder-allowBlocking" class="headerlink" title="3.2.1 Binder.allowBlocking"></a>3.2.1 Binder.allowBlocking</h4>
<pre><code>
public static IBinder allowBlocking(IBinder binder) {
       try {
           //是BinderProxy类
           if (binder instanceof BinderProxy) {
               ((BinderProxy) binder).mWarnOnBlocking = false;
           } else if (binder != null && binder.getInterfaceDescriptor() != null
                   && binder.queryLocalInterface(binder.getInterfaceDescriptor()) == null) {
               Log.w(TAG, "Unable to allow blocking on interface " + binder);
           }
       } catch (RemoteException ignored) {
       }
       return binder;
   }</code></pre>

<p>这个方法用来判断IBinder类，根据不同情况添加标志，具体可见3.7节</p>
<h4 id="3-2-2-getContextObject"><a href="#3-2-2-getContextObject" class="headerlink" title="3.2.2 getContextObject"></a>3.2.2 getContextObject</h4><p>[-&gt;android_util_Binder.cpp]</p>

<pre><code>
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    sp&lt;IBinder&gt; b = ProcessState::self()-&gt;getContextObject(NULL);
    return javaObjectForIBinder(env, b);
}</code></pre>

<h5 id="3-2-2-1-ProcessState-self"><a href="#3-2-2-1-ProcessState-self" class="headerlink" title="3.2.2.1 ProcessState::self"></a>3.2.2.1 ProcessState::self</h5><p>[-&gt;ProcessState.cpp]</p>

<pre><code>
sp&lt;ProcessState&gt; ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != NULL) {
        return gProcess;
    }
    //创建ProcessState对象
    gProcess = new ProcessState("/dev/binder");
    return gProcess;
}</code></pre>

<p>单例模式创建ProcessState对象，其初始化如下：</p>

<pre><code>
ProcessState::ProcessState(const char *driver)
    : mDriverName(String8(driver))
    , mDriverFD(open_driver(driver)) //打开驱动
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mStarvationStartTimeMs(0)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
{
    if (mDriverFD &gt;= 0) {
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        // 采用内存映射函数，给binder分配一块虚拟内存空间，用来接收事务
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            // *sigh*
            ALOGE("Using %s failed: unable to mmap transaction memory./n", mDriverName.c_str());
            close(mDriverFD);
            mDriverFD = -1;
            mDriverName.clear();
        }
    }

    LOG_ALWAYS_FATAL_IF(mDriverFD &lt; 0, "Binder driver could not be opened.  Terminating.");
}</code></pre>

<p>ProcessState一个进程只打开一个binder设置一次，其中mDriverFD，记录binder驱动的fd，用于访问binder设置；</p>
<p>BINDER_VM_SIZE = 1 <em> 1024 </em> 1024 - 4096 * 2)，binder分配的内存大小为1M-8K；</p>
<p>DEFAULT_MAX_BINDER_THREADS =15，binder默认最大可并发访问的线程数为16。</p>
<p>打开驱动的过程如下，这里就只介绍到这里。</p>

<pre><code>
static int open_driver(const char *driver)
{
    //打开/dev/binder设置
    int fd = open(driver, O_RDWR | O_CLOEXEC);
    if (fd &gt;= 0) {
        int vers = 0;
        status_t result = ioctl(fd, BINDER_VERSION, &vers);
        if (result == -1) {
            ALOGE("Binder ioctl to obtain version failed: %s", strerror(errno));
            close(fd);
            fd = -1;
        }
        if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {
          ALOGE("Binder driver protocol(%d) does not match user space protocol(%d)! ioctl() return value: %d",
                vers, BINDER_CURRENT_PROTOCOL_VERSION, result);
            close(fd);
            fd = -1;
        }
        //设定最大的binder支持的最大线程数
        size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
        //通过ioctl设置binder驱动
        result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
        if (result == -1) {
            ALOGE("Binder ioctl to set max threads failed: %s", strerror(errno));
        }
    } else {
        ALOGW("Opening '%s' failed: %s/n", driver, strerror(errno));
    }
    return fd;
}</code></pre>

<h5 id="3-2-2-2-PS-getContextObject"><a href="#3-2-2-2-PS-getContextObject" class="headerlink" title="3.2.2.2  PS.getContextObject"></a>3.2.2.2  PS.getContextObject</h5><p>[-&gt;ProcessState.cpp]</p>

<pre><code>
sp&lt;IBinder&gt; ProcessState::getContextObject(const sp&lt;IBinder&gt;& /*caller*/)
{
    return getStrongProxyForHandle(0);
}</code></pre>

<h5 id="3-2-2-3-PS-getStrongProxyForHandle"><a href="#3-2-2-3-PS-getStrongProxyForHandle" class="headerlink" title="3.2.2.3 PS.getStrongProxyForHandle"></a>3.2.2.3 PS.getStrongProxyForHandle</h5><p>[-&gt;ProcessState.cpp]</p>

<pre><code>
sp&lt;IBinder&gt; ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp&lt;IBinder&gt; result;

    AutoMutex _l(mLock);
    //查找handle对应的资源项
    handle_entry* e = lookupHandleLocked(handle);

    if (e != NULL) {
        // We need to create a new BpBinder if there isn't currently one, OR we
        // are unable to acquire a weak reference on this current one.  See comment
        // in getWeakProxyForHandle() for more info about this.
        IBinder* b = e-&gt;binder;
        if (b == NULL || !e-&gt;refs-&gt;attemptIncWeak(this)) {
            if (handle == 0) {
                // Special case for context manager...
                // The context manager is the only object for which we create
                // a BpBinder proxy without already holding a reference.
                // Perform a dummy transaction to ensure the context manager
                // is registered before we create the first local reference
                // to it (which will occur when creating the BpBinder).
                // If a local reference is created for the BpBinder when the
                // context manager is not present, the driver will fail to
                // provide a reference to the context manager, but the
                // driver API does not return status.
                //
                // Note that this is not race-free if the context manager
                // dies while this code runs.
                //
                // TODO: add a driver API to wait for context manager, or
                // stop special casing handle 0 for context manager and add
                // a driver API to get a handle to the context manager with
                // proper reference counting.

                Parcel data;
                status_t status = IPCThreadState::self()-&gt;transact(
                        0, IBinder::PING_TRANSACTION, data, NULL, 0);
                if (status == DEAD_OBJECT)
                   return NULL;
            }
            //创建BpBinder
            b = BpBinder::create(handle);
            e-&gt;binder = b;
            if (b) e-&gt;refs = b-&gt;getWeakRefs();
            result = b;
        } else {
            // This little bit of nastyness is to allow us to add a primary
            // reference to the remote proxy when this team doesn't have one
            // but another team is sending the handle to us.
            result.force_set(b);
            e-&gt;refs-&gt;decWeak(this);
        }
    }

    return result;
}</code></pre>

<p>当handler值所对应的Binder不存在或弱引用无效时会创建BpBinder，否则直接获取。对应handler等于0的情况，通过PING_TRANSACTION来判断是否准备就绪。如果context manager还未生效前，一个BpBinder的本地已经被创建，那么驱动将无法提供context manager的引用。</p>
<h4 id="3-2-3-javaObjectForIBinder"><a href="#3-2-3-javaObjectForIBinder" class="headerlink" title="3.2.3 javaObjectForIBinder"></a>3.2.3 javaObjectForIBinder</h4><p>[-&gt;android_util_Binder.cpp]</p>

<pre><code>
jobject javaObjectForIBinder(JNIEnv* env, const sp&lt;IBinder&gt;& val)
{
    if (val == NULL) return NULL;

    if (val-&gt;checkSubclass(&gBinderOffsets)) {
        // It's a JavaBBinder created by ibinderForJavaObject. Already has Java object.
        jobject object = static_cast&lt;JavaBBinder*&gt;(val.get())-&gt;object();
        LOGDEATH("objectForBinder %p: it's our own %p!/n", val.get(), object);
        return object;
    }

    // For the rest of the function we will hold this lock, to serialize
    // looking/creation/destruction of Java proxies for native Binder proxies.
    AutoMutex _l(gProxyLock);

    BinderProxyNativeData* nativeData = gNativeDataCache;
    if (nativeData == nullptr) {
        nativeData = new BinderProxyNativeData();
    }
    // gNativeDataCache is now logically empty.
    //创建BinderProxy对象
    jobject object = env-&gt;CallStaticObjectMethod(gBinderProxyOffsets.mClass,
            gBinderProxyOffsets.mGetInstance, (jlong) nativeData, (jlong) val.get());
    if (env-&gt;ExceptionCheck()) {
        // In the exception case, getInstance still took ownership of nativeData.
        gNativeDataCache = nullptr;
        return NULL;
    }
    BinderProxyNativeData* actualNativeData = getBPNativeData(env, object);
    if (actualNativeData == nativeData) {
        // New BinderProxy; we still have exclusive access.
        nativeData-&gt;mOrgue = new DeathRecipientList;
        nativeData-&gt;mObject = val;
        gNativeDataCache = nullptr;
        ++gNumProxies;
        if (gNumProxies &gt;= gProxiesWarned + PROXY_WARN_INTERVAL) {
            ALOGW("Unexpectedly many live BinderProxies: %d/n", gNumProxies);
            gProxiesWarned = gNumProxies;
        }
    } else {
        // nativeData wasn't used. Reuse it the next time.
        gNativeDataCache = nativeData;
    }

    return object;
}
</code></pre>

<p>根据BpBinder（C++）创建BinderProxy（java）对象，主要工作是创建BinderProxy对象。</p>
<p>ServiceManagerNative.asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));相当于</p>
<p>ServiceManagerNative.asInterface(new BinderProxy())</p>
<h3 id="3-3-SMN-asInterface"><a href="#3-3-SMN-asInterface" class="headerlink" title="3.3 SMN.asInterface"></a>3.3 SMN.asInterface</h3>
<pre><code>
static public IServiceManager asInterface(IBinder obj)
   {
       if (obj == null) {
           return null;
       }
       //由于obj为
       IServiceManager in = (IServiceManager)obj.queryLocalInterface(descriptor);
       if (in != null) {
           return in;
       }

       return new ServiceManagerProxy(obj);
   }</code></pre>

<p>由3.2节可知ServiceManagerNative.asInterface(new BinderProxy()),等价于</p>
<p>new ServiceManagerProxy(new BinderProxy());</p>
<h4 id="3-3-1-ServiceManagerProxy"><a href="#3-3-1-ServiceManagerProxy" class="headerlink" title="3.3.1 ServiceManagerProxy"></a>3.3.1 ServiceManagerProxy</h4><p>[-&gt;ServiceManagerNative::ServiceManagerProxy]</p>

<pre><code>
class ServiceManagerProxy implements IServiceManager {
    public ServiceManagerProxy(IBinder remote) {
        mRemote = remote;
    }

    public IBinder asBinder() {
        return mRemote;
    }
   ...
}
</code></pre>

<p><strong>mRemote为BinderProxy对象</strong>，该对象对应于BpBinder(0)，作为binder代理类，执行native层ServiceManager</p>
<p> getIServiceManager()等价于new ServiceManagerProxy(new BinderProxy());</p>
<p> getIServiceManager().addService等价于ServiceManagerProxy.addService</p>
<p>framework层的ServiceManager的调用实际交给了ServiceManagerProxy成员变量BinderProxy，而BinderProxy通过JNI方式，最终调用的BpBinder对象，可以看出binder架构的核心功能依赖于native架构的服务来完成。</p>
<h3 id="3-4-SMP-addService"><a href="#3-4-SMP-addService" class="headerlink" title="3.4 SMP.addService"></a>3.4 SMP.addService</h3><p>[-&gt;ServiceManagerNative::ServiceManagerProxy]</p>

<pre><code>
public void addService(String name, IBinder service, boolean allowIsolated, int dumpPriority)
        throws RemoteException {
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IServiceManager.descriptor);
    data.writeString(name);
    //见3.5节
    data.writeStrongBinder(service);
    data.writeInt(allowIsolated ? 1 : 0);
    data.writeInt(dumpPriority);
    //见3.7节
    mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);
    reply.recycle();
    data.recycle();
}
</code></pre>

<h3 id="3-5-writeStrongBinder-Java"><a href="#3-5-writeStrongBinder-Java" class="headerlink" title="3.5 writeStrongBinder(Java)"></a>3.5 writeStrongBinder(Java)</h3><p>[-&gt;Parcel.java]</p>

<pre><code>
public final void writeStrongBinder(IBinder val) {
    //native调用
    nativeWriteStrongBinder(mNativePtr, val);
}
</code></pre>

<h4 id="3-5-1-android-os-Parcel-writeStrongBinder"><a href="#3-5-1-android-os-Parcel-writeStrongBinder" class="headerlink" title="3.5.1 android_os_Parcel_writeStrongBinder"></a>3.5.1 android_os_Parcel_writeStrongBinder</h4><p>[-&gt;android_os_Parcel.cpp]</p>

<pre><code>
static void android_os_Parcel_writeStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr, jobject object)
{
    //将java层的Parce转换为native层的Parcel
    Parcel* parcel = reinterpret_cast&lt;Parcel*&gt;(nativePtr);
    if (parcel != NULL) {
        const status_t err = parcel-&gt;writeStrongBinder(ibinderForJavaObject(env, object));
        if (err != NO_ERROR) {
            signalExceptionForError(env, clazz, err);
        }
    }
}
</code></pre>

<p>nativePtr为native层创建Parcel对象后返回的指针地址。</p>
<h4 id="3-5-2-ibinderForJavaObject"><a href="#3-5-2-ibinderForJavaObject" class="headerlink" title="3.5.2 ibinderForJavaObject"></a>3.5.2 ibinderForJavaObject</h4><p>[-&gt;android_util_Binder.cpp]</p>

<pre><code>
sp&lt;IBinder&gt; ibinderForJavaObject(JNIEnv* env, jobject obj)
{
    if (obj == NULL) return NULL;

    // Instance of Binder?
    // java层Binder对象？
    if (env-&gt;IsInstanceOf(obj, gBinderOffsets.mClass)) {
        JavaBBinderHolder* jbh = (JavaBBinderHolder*)
            env-&gt;GetLongField(obj, gBinderOffsets.mObject);
        return jbh-&gt;get(env, obj);
    }

    // Instance of BinderProxy?
    //java层的BinderProxy对象
    if (env-&gt;IsInstanceOf(obj, gBinderProxyOffsets.mClass)) {
        return getBPNativeData(env, obj)-&gt;mObject;
    }

    ALOGW("ibinderForJavaObject: %p is not a Binder object", obj);
    return NULL;
}</code></pre>

<p>根据Binder(Java)生成JavaBBinderHolder(C++)对象，并把JavaBBinderHolder对象地址保存到Binder.mObject成员变量</p>
<p>如果是BinderProxy(Java)返回的是Binder Native代理BpBinder。</p>
<h4 id="3-5-3-JavaBBinderHolder"><a href="#3-5-3-JavaBBinderHolder" class="headerlink" title="3.5.3 JavaBBinderHolder"></a>3.5.3 JavaBBinderHolder</h4><p>[-&gt;android_util_Binder.cpp]</p>

<pre><code>
class JavaBBinderHolder
{
public:
    sp&lt;JavaBBinder&gt; get(JNIEnv* env, jobject obj)
    {
        AutoMutex _l(mLock);
        sp&lt;JavaBBinder&gt; b = mBinder.promote();
        if (b == NULL) {
            //创建JavaBBinder对象
            b = new JavaBBinder(env, obj);
            mBinder = b;
            ALOGV("Creating JavaBinder %p (refs %p) for Object %p, weakCount=%" PRId32 "/n",
                 b.get(), b-&gt;getWeakRefs(), obj, b-&gt;getWeakRefs()-&gt;getWeakCount());
        }

        return b;
    }

    sp&lt;JavaBBinder&gt; getExisting()
    {
        AutoMutex _l(mLock);
        return mBinder.promote();
    }

private:
    Mutex           mLock;
    wp&lt;JavaBBinder&gt; mBinder;
};</code></pre>

<h4 id="3-5-4-new-JavaBBinder"><a href="#3-5-4-new-JavaBBinder" class="headerlink" title="3.5.4 new JavaBBinder"></a>3.5.4 new JavaBBinder</h4><p>[-&gt;android_util_Binder.cpp]</p>

<pre><code>
class JavaBBinder : public BBinder
{
 public:
    JavaBBinder(JNIEnv* env, jobject /* Java Binder */ object)
        : mVM(jnienv_to_javavm(env)), mObject(env-&gt;NewGlobalRef(object))
    {
        ALOGV("Creating JavaBBinder %p/n", this);
        gNumLocalRefsCreated.fetch_add(1, std::memory_order_relaxed);
        gcIfManyNewRefs(env);
    }

    bool    checkSubclass(const void* subclassID) const
    {
        return subclassID == &gBinderOffsets;
    }

    jobject object() const
    {
        return mObject;
    }
  }</code></pre>

<p>创建JavaBBinder，该类继承于BBinder， data.writeStrongBinder(service);等价于parcel-&gt;writeStrongBinder(new JavaBBinder(env, obj))</p>
<h3 id="3-6-writeStrongBinder-C"><a href="#3-6-writeStrongBinder-C" class="headerlink" title="3.6 writeStrongBinder(C++)"></a>3.6 writeStrongBinder(C++)</h3><p>[-&gt;Parcel.cpp]</p>

<pre><code>
status_t Parcel::writeStrongBinder(const sp&lt;IBinder&gt;& val)
{
    return flatten_binder(ProcessState::self(), val, this);
}
</code></pre>

<h4 id="3-6-1-flatten-binder"><a href="#3-6-1-flatten-binder" class="headerlink" title="3.6.1 flatten_binder"></a>3.6.1 flatten_binder</h4><p>[-&gt;Parcel.cpp]</p>

<pre><code>
status_t flatten_binder(const sp&lt;ProcessState&gt;& /*proc*/,
    const sp&lt;IBinder&gt;& binder, Parcel* out)
{
    flat_binder_object obj;

    if (IPCThreadState::self()-&gt;backgroundSchedulingDisabled()) {
        /* minimum priority for all nodes is nice 0 */
        obj.flags = FLAT_BINDER_FLAG_ACCEPTS_FDS;
    } else {
        /* minimum priority for all nodes is MAX_NICE(19) */
        obj.flags = 0x13 | FLAT_BINDER_FLAG_ACCEPTS_FDS;
    }

    if (binder != NULL) {
        IBinder *local = binder-&gt;localBinder();
        if (!local) { //远程Binder
            BpBinder *proxy = binder-&gt;remoteBinder();
            if (proxy == NULL) {
                ALOGE("null proxy");
            }
            const int32_t handle = proxy ? proxy-&gt;handle() : 0;
            obj.hdr.type = BINDER_TYPE_HANDLE;
            obj.binder = 0; /* Don't pass uninitialized stack data to a remote process */
            obj.handle = handle;
            obj.cookie = 0;
        } else {  //本地Binder
            obj.hdr.type = BINDER_TYPE_BINDER;
            obj.binder = reinterpret_cast&lt;uintptr_t&gt;(local-&gt;getWeakRefs());
            obj.cookie = reinterpret_cast&lt;uintptr_t&gt;(local);
        }
    } else {
        obj.hdr.type = BINDER_TYPE_BINDER;
        obj.binder = 0;
        obj.cookie = 0;
    }

    return finish_flatten_binder(binder, obj, out);
}</code></pre>

<p>主要是将Binder对象扁平化，转换成flat_binder_object。</p>
<p>对于Binder实体，cookie记录Binder实体的指针</p>
<p>对于Binder代理，则用handle记录Binder代理的句柄。</p>
<h4 id="3-6-2-finish-flatten-binder"><a href="#3-6-2-finish-flatten-binder" class="headerlink" title="3.6.2 finish_flatten_binder"></a>3.6.2 finish_flatten_binder</h4><p>[-&gt;Parcel.cpp]</p>

<pre><code>
inline static status_t finish_flatten_binder(
    const sp&lt;IBinder&gt;& /*binder*/, const flat_binder_object& flat, Parcel* out)
{
    return out-&gt;writeObject(flat, false);
}
</code></pre>

<p>回到3.4的addService过程，则进入transact过程</p>
<h3 id="3-7-BP-transact"><a href="#3-7-BP-transact" class="headerlink" title="3.7 BP.transact"></a>3.7 BP.transact</h3><p>[-&gt;BinderProxy.java]</p>

<pre><code>
public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
       Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");

       if (mWarnOnBlocking && ((flags & FLAG_ONEWAY) == 0)) {
           // For now, avoid spamming the log by disabling after we've logged
           // about this interface at least once
           mWarnOnBlocking = false;
           Log.w(Binder.TAG, "Outgoing transactions from this process must be FLAG_ONEWAY",
                   new Throwable());
       }

       final boolean tracingEnabled = Binder.isTracingEnabled();
       if (tracingEnabled) {
           final Throwable tr = new Throwable();
           Binder.getTransactionTracker().addTrace(tr);
           StackTraceElement stackTraceElement = tr.getStackTrace()[1];
           Trace.traceBegin(Trace.TRACE_TAG_ALWAYS,
                   stackTraceElement.getClassName() + "." + stackTraceElement.getMethodName());
       }
       try {
           //native层调用
           return transactNative(code, data, reply, flags);
       } finally {
           if (tracingEnabled) {
               Trace.traceEnd(Trace.TRACE_TAG_ALWAYS);
           }
       }
   }</code></pre>

<h3 id="3-8-android-os-BinderProxy-transact"><a href="#3-8-android-os-BinderProxy-transact" class="headerlink" title="3.8  android_os_BinderProxy_transact"></a>3.8  android_os_BinderProxy_transact</h3><p>[-&gt;android_util_Binder.cpp]</p>

<pre><code>
static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
        jint code, jobject dataObj, jobject replyObj, jint flags) // throws RemoteException
{
    if (dataObj == NULL) {
        jniThrowNullPointerException(env, NULL);
        return JNI_FALSE;
    }
   
    //将java parcel转换为native parcel
    Parcel* data = parcelForJavaObject(env, dataObj);
    if (data == NULL) {
        return JNI_FALSE;
    }
    Parcel* reply = parcelForJavaObject(env, replyObj);
    if (reply == NULL && replyObj != NULL) {
        return JNI_FALSE;
    }
    
    //获取binder native代理
    IBinder* target = getBPNativeData(env, obj)-&gt;mObject.get();
    if (target == NULL) {
        jniThrowException(env, "java/lang/IllegalStateException", "Binder has been finalized!");
        return JNI_FALSE;
    }

    ALOGV("Java code calling transact on %p in Java object %p with code %" PRId32 "/n",
            target, obj, code);


    bool time_binder_calls;
    int64_t start_millis;
    if (kEnableBinderSample) {
        // Only log the binder call duration for things on the Java-level main thread.
        // But if we don't
        time_binder_calls = should_time_binder_calls();

        if (time_binder_calls) {
            start_millis = uptimeMillis();
        }
    }

    //printf("Transact from Java code to %p sending: ", target); data-&gt;print();
    //相当于BpBinder.transact
    status_t err = target-&gt;transact(code, *data, reply, flags);
    //if (reply) printf("Transact from Java code to %p received: ", target); reply-&gt;print();

    if (kEnableBinderSample) {
        if (time_binder_calls) {
            conditionally_log_binder_call(start_millis, target, code);
        }
    }

    if (err == NO_ERROR) {
        return JNI_TRUE;
    } else if (err == UNKNOWN_TRANSACTION) {
        return JNI_FALSE;
    }

    signalExceptionForError(env, obj, err, true /*canThrowRemoteException*/, data-&gt;dataSize());
    return JNI_FALSE;
}</code></pre>

<h3 id="3-9-BpBinder-transact"><a href="#3-9-BpBinder-transact" class="headerlink" title="3.9 BpBinder::transact"></a>3.9 BpBinder::transact</h3><p>[-&gt;BpBinder.cpp]</p>

<pre><code>
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        //见3.10节
        status_t status = IPCThreadState::self()-&gt;transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }

    return DEAD_OBJECT;
}
</code></pre>

<h4 id="3-9-1-BpBinder-transact"><a href="#3-9-1-BpBinder-transact" class="headerlink" title="3.9.1 BpBinder::transact"></a>3.9.1 BpBinder::transact</h4><p>[-&gt;IPCThreadState.cpp]</p>

<pre><code>
IPCThreadState* IPCThreadState::self()
{
    if (gHaveTLS) {
restart:
        const pthread_key_t k = gTLS;
        IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
        if (st) return st;
        return new IPCThreadState; //创建IPCThreadState，见3.9.2
    }

    if (gShutdown) {
        ALOGW("Calling IPCThreadState::self() during shutdown is dangerous, expect a crash./n");
        return NULL;
    }

    //加锁
    pthread_mutex_lock(&gTLSMutex);
    if (!gHaveTLS) {
        //创建线程TLS
        int key_create_value = pthread_key_create(&gTLS, threadDestructor);
        if (key_create_value != 0) {
            pthread_mutex_unlock(&gTLSMutex);
            ALOGW("IPCThreadState::self() unable to create TLS key, expect a crash: %s/n",
                    strerror(key_create_value));
            return NULL;
        }
        gHaveTLS = true;
    }
    //释放锁
    pthread_mutex_unlock(&gTLSMutex);
    goto restart;
}</code></pre>

<p>TLS(Thread local storage),线程本地存储空间，每个线程都有自己私有的TLS,线程之间不能共享。</p>
<p>pthread_setspecific/pthread_getspecific可以设置和获取这些空间中的内容。</p>
<h4 id="3-9-2-new-IPCThreadState"><a href="#3-9-2-new-IPCThreadState" class="headerlink" title="3.9.2 new IPCThreadState"></a>3.9.2 new IPCThreadState</h4><p>[-&gt;IPCThreadState.cpp]</p>

<pre><code>
IPCThreadState::IPCThreadState()
    : mProcess(ProcessState::self()),
      mStrictModePolicy(0),
      mLastTransactionBinderFlags(0)
{
    pthread_setspecific(gTLS, this);
    clearCaller();
    mIn.setDataCapacity(256);
    mOut.setDataCapacity(256);
}
</code></pre>

<p>每个线程都有一个IPCThreadState，每个IPCThreadState都有一个mIn，mOut，成员变量mProcess保存了ProcessState变量，每个进程只有一个。</p>
<p>mInt用来接收来自Binder设备的数据，默认大小事256</p>
<p>mOut用来存储发往Binder设备的数据，默认大小事256</p>
<h3 id="3-10-IPC-transact"><a href="#3-10-IPC-transact" class="headerlink" title="3.10 IPC::transact"></a>3.10 IPC::transact</h3><p>[-&gt;IPCThreadState.cpp]</p>

<pre><code>
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err;

    flags |= TF_ACCEPT_FDS;

    IF_LOG_TRANSACTIONS() {
        TextOutput::Bundle _b(alog);
        alog &lt;&lt; "BC_TRANSACTION thr " &lt;&lt; (void*)pthread_self() &lt;&lt; " / hand "
            &lt;&lt; handle &lt;&lt; " / code " &lt;&lt; TypeCode(code) &lt;&lt; ": "
            &lt;&lt; indent &lt;&lt; data &lt;&lt; dedent &lt;&lt; endl;
    }

    LOG_ONEWAY("&gt;&gt;&gt;&gt; SEND from pid %d uid %d %s", getpid(), getuid(),
        (flags & TF_ONE_WAY) == 0 ? "READ REPLY" : "ONE WAY");
    //传输数据，见3.11    
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);

    if (err != NO_ERROR) {
        //错误则返回
        if (reply) reply-&gt;setError(err);
        return (mLastError = err);
    }

    if ((flags & TF_ONE_WAY) == 0) {
        #if 0
        if (code == 4) { // relayout
            ALOGI("&gt;&gt;&gt;&gt;&gt;&gt; CALLING transaction 4");
        } else {
            ALOGI("&gt;&gt;&gt;&gt;&gt;&gt; CALLING transaction %d", code);
        }
        #endif
        if (reply) {
            //等待回应
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
        #if 0
        if (code == 4) { // relayout
            ALOGI("&lt;&lt;&lt;&lt;&lt;&lt; RETURNING transaction 4");
        } else {
            ALOGI("&lt;&lt;&lt;&lt;&lt;&lt; RETURNING transaction %d", code);
        }
        #endif

        IF_LOG_TRANSACTIONS() {
            TextOutput::Bundle _b(alog);
            alog &lt;&lt; "BR_REPLY thr " &lt;&lt; (void*)pthread_self() &lt;&lt; " / hand "
                &lt;&lt; handle &lt;&lt; ": ";
            if (reply) alog &lt;&lt; indent &lt;&lt; *reply &lt;&lt; dedent &lt;&lt; endl;
            else alog &lt;&lt; "(none requested)" &lt;&lt; endl;
        }
    } else {
        //oneway，不需要等待reply的场景
        err = waitForResponse(NULL, NULL);
    }

    return err;
}</code></pre>

<p>transact主要工作：</p>
<ul>
<li><p>errorCheck错误检查</p>
</li>
<li><p>writeTransactionData传输数据</p>
</li>
<li><p>waitForResponse等待响应</p>
</li>
</ul>
<h3 id="3-11-IPC-writeTransactionData"><a href="#3-11-IPC-writeTransactionData" class="headerlink" title="3.11  IPC::writeTransactionData"></a>3.11  IPC::writeTransactionData</h3><p>[-&gt;IPCThreadState.cpp]</p>

<pre><code>
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;

    tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */
    tr.target.handle = handle;  //handle = 0
    tr.code = code;             //code = ADD_SERVICE_TRANSACTION
    tr.flags = binderFlags;     // binderFlags = 0
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    //data记录服务的Parcel对象
    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        tr.data_size = data.ipcDataSize();
        tr.data.ptr.buffer = data.ipcData();
        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
        tr.data.ptr.offsets = data.ipcObjects();
    } else if (statusBuffer) {
        tr.flags |= TF_STATUS_CODE;
        *statusBuffer = err;
        tr.data_size = sizeof(status_t);
        tr.data.ptr.buffer = reinterpret_cast&lt;uintptr_t&gt;(statusBuffer);
        tr.offsets_size = 0;
        tr.data.ptr.offsets = 0;
    } else {
        return (mLastError = err);
    }

    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));

    return NO_ERROR;
}
</code></pre>

<p>handler对象用来标记目的端，注册服务的目的端为ServiceManager,这里handle = 0所对应的是binder实体对象。</p>
<p>binder_transaction_data是binder驱动通信的数据结构，该过程吧Binder请求码BC_TRANSACTION和binder_transaction_data结构体写入mOut，写完后执行waitForResponse方法。</p>
<p>binder_transaction_data中重要的成员变量</p>
<ul>
<li><p>data_size，binder_transaction的数据大小</p>
</li>
<li><p>data.ptr.buffer，binder_transaction数据的起始地址</p>
</li>
<li><p>offsets_size，记录flat_binder_object结构体的个数</p>
</li>
<li><p>data.ptr.offsets，记录flat_binder_object结构体的数据偏移量</p>
</li>
</ul>
<h3 id="3-12-IPC-waitForResponse"><a href="#3-12-IPC-waitForResponse" class="headerlink" title="3.12 IPC::waitForResponse"></a>3.12 IPC::waitForResponse</h3><p>[-&gt;IPCThreadState.cpp]</p>

<pre><code>
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        //见3.13小节
        if ((err=talkWithDriver()) &lt; NO_ERROR) break;
        err = mIn.errorCheck();
        if (err &lt; NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;

        cmd = (uint32_t)mIn.readInt32();

        IF_LOG_COMMANDS() {
            alog &lt;&lt; "Processing waitForResponse Command: "
                &lt;&lt; getReturnString(cmd) &lt;&lt; endl;
        }

        switch (cmd) {
        case BR_TRANSACTION_COMPLETE:
            if (!reply && !acquireResult) goto finish;
            break;

        case BR_DEAD_REPLY:
            err = DEAD_OBJECT;
            goto finish;

        case BR_FAILED_REPLY:
            err = FAILED_TRANSACTION;
            goto finish;

        case BR_ACQUIRE_RESULT:
            {
                ALOG_ASSERT(acquireResult != NULL, "Unexpected brACQUIRE_RESULT");
                const int32_t result = mIn.readInt32();
                if (!acquireResult) continue;
                *acquireResult = result ? NO_ERROR : INVALID_OPERATION;
            }
            goto finish;

        case BR_REPLY:
            {
                binder_transaction_data tr;
                err = mIn.read(&tr, sizeof(tr));
                ALOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");
                if (err != NO_ERROR) goto finish;

                if (reply) {
                    if ((tr.flags & TF_STATUS_CODE) == 0) {
                        reply-&gt;ipcSetDataReference(
                            reinterpret_cast&lt;const uint8_t*&gt;(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast&lt;const binder_size_t*&gt;(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t),
                            freeBuffer, this);
                    } else {
                        err = *reinterpret_cast&lt;const status_t*&gt;(tr.data.ptr.buffer);
                        freeBuffer(NULL,
                            reinterpret_cast&lt;const uint8_t*&gt;(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast&lt;const binder_size_t*&gt;(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t), this);
                    }
                } else {
                    freeBuffer(NULL,
                        reinterpret_cast&lt;const uint8_t*&gt;(tr.data.ptr.buffer),
                        tr.data_size,
                        reinterpret_cast&lt;const binder_size_t*&gt;(tr.data.ptr.offsets),
                        tr.offsets_size/sizeof(binder_size_t), this);
                    continue;
                }
            }
            goto finish;

        default:
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }

finish:
    if (err != NO_ERROR) {
        if (acquireResult) *acquireResult = err;
        if (reply) reply-&gt;setError(err);
        mLastError = err;
    }

    return err;
}</code></pre>

<p>talkWithDriver过程是和binder驱动通信过程，Binder驱动收到BC_TRANSACTION后，会回应BR_TRANSACTION_COMPLETE,然后等待目标进程的BR_REPLY.</p>
<h3 id="3-13-IPC-talkWithDriver"><a href="#3-13-IPC-talkWithDriver" class="headerlink" title="3.13 IPC::talkWithDriver"></a>3.13 IPC::talkWithDriver</h3><p>[-&gt;IPCThreadState.cpp]</p>

<pre><code>
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    if (mProcess-&gt;mDriverFD &lt;= 0) {
        return -EBADF;
    }

    binder_write_read bwr;

    // Is the read buffer empty?
    // 读缓冲是否为空
    const bool needRead = mIn.dataPosition() &gt;= mIn.dataSize();

    // We don't want to write anything if we are still reading
    // from data left in the input buffer and the caller
    // has requested to read the next data.
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();

    // This is what we'll read.
    if (doReceive && needRead) {
        //接收数据缓冲区信息，如果收到数据，就直接填在mInt中
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

    IF_LOG_COMMANDS() {
        TextOutput::Bundle _b(alog);
        if (outAvail != 0) {
            alog &lt;&lt; "Sending commands to driver: " &lt;&lt; indent;
            const void* cmds = (const void*)bwr.write_buffer;
            const void* end = ((const uint8_t*)cmds)+bwr.write_size;
            alog &lt;&lt; HexDump(cmds, bwr.write_size) &lt;&lt; endl;
            while (cmds &lt; end) cmds = printCommand(alog, cmds);
            alog &lt;&lt; dedent;
        }
        alog &lt;&lt; "Size of receive buffer: " &lt;&lt; bwr.read_size
            &lt;&lt; ", needRead: " &lt;&lt; needRead &lt;&lt; ", doReceive: " &lt;&lt; doReceive &lt;&lt; endl;
    }

    // Return immediately if there is nothing to do.
    //读缓冲和写缓冲为空，直接返回
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    //先清零
    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        IF_LOG_COMMANDS() {
            alog &lt;&lt; "About to read/write, write size = " &lt;&lt; mOut.dataSize() &lt;&lt; endl;
        }
#if defined(__ANDROID__)
        //通过ioctl和Binder Driver进行通信，进行读写操作
        if (ioctl(mProcess-&gt;mDriverFD, BINDER_WRITE_READ, &bwr) &gt;= 0)
            err = NO_ERROR;
        else
            err = -errno;
#else
        err = INVALID_OPERATION;
#endif
        if (mProcess-&gt;mDriverFD &lt;= 0) {
            err = -EBADF;
        }
        IF_LOG_COMMANDS() {
            alog &lt;&lt; "Finished read/write, write size = " &lt;&lt; mOut.dataSize() &lt;&lt; endl;
        }
    } while (err == -EINTR); //当被中断，则继续执行

    IF_LOG_COMMANDS() {
        alog &lt;&lt; "Our err: " &lt;&lt; (void*)(intptr_t)err &lt;&lt; ", write consumed: "
            &lt;&lt; bwr.write_consumed &lt;&lt; " (of " &lt;&lt; mOut.dataSize()
                        &lt;&lt; "), read consumed: " &lt;&lt; bwr.read_consumed &lt;&lt; endl;
    }

    if (err &gt;= NO_ERROR) {
        if (bwr.write_consumed &gt; 0) {
            //写的数据大于0
            if (bwr.write_consumed &lt; mOut.dataSize())
                mOut.remove(0, bwr.write_consumed);
            else {
                mOut.setDataSize(0);
                //清除引用
                processPostWriteDerefs();
            }
        }
        if (bwr.read_consumed &gt; 0) {
            //读取的数据大于0，设置mIn数据大小和将指针位置置为开始位置
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }
        IF_LOG_COMMANDS() {
            TextOutput::Bundle _b(alog);
            alog &lt;&lt; "Remaining data size: " &lt;&lt; mOut.dataSize() &lt;&lt; endl;
            alog &lt;&lt; "Received commands from driver: " &lt;&lt; indent;
            const void* cmds = mIn.data();
            const void* end = mIn.data() + mIn.dataSize();
            alog &lt;&lt; HexDump(cmds, mIn.dataSize()) &lt;&lt; endl;
            while (cmds &lt; end) cmds = printReturnCommand(alog, cmds);
            alog &lt;&lt; dedent;
        }
        return NO_ERROR;
    }

    return err;
}</code></pre>

<p>binder_write_read结构体用来与Binder驱动进行数据交换，通过ioctl与mDriverFD通信，主要操作的是mInt和mOut.</p>
<p>ioctl通过系统调用进入到Binder Driver。</p>
<h2 id="四、Binder-Driver"><a href="#四、Binder-Driver" class="headerlink" title="四、Binder Driver"></a>四、Binder Driver</h2><p>用户态的程序调用Kernel层驱动需陷入内核态，进行系统调用（syscall），其定义在kernel/include/linux/syscalls.h文件中</p>
<p>打开Binder驱动方法调用流程：ioctl-&gt;__ioctl-&gt;binder_ioctl，ioctl为用户空间的方法，</p>
<p>__ioctl是系统调用中相应的处理方法，通过查找找到内核binder驱动的binder_ioctl的方法。</p>
<p>上面传过来的是BINDER_WRITE_READ命令</p>
<h3 id="4-1-binder-ioctl"><a href="#4-1-binder-ioctl" class="headerlink" title="4.1 binder_ioctl"></a>4.1 binder_ioctl</h3><p>[-&gt;android/binder.c]</p>

<pre><code>
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	int ret;
	struct binder_proc *proc = filp-&gt;private_data;  //binder进程
	struct binder_thread *thread;      //binder线程
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;

	/*pr_info("binder_ioctl: %d:%d %x %lx/n",
			proc-&gt;pid, current-&gt;pid, cmd, arg);*/

	binder_selftest_alloc(&proc-&gt;alloc);

	trace_binder_ioctl(cmd, arg);
    //进入休眠状态，直到中断唤醒
	ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error &lt; 2);
	if (ret)
		goto err_unlocked;
    //获取binder_thread，见4.1.1节
	thread = binder_get_thread(proc);
	if (thread == NULL) {
		ret = -ENOMEM;
		goto err;
	}

	switch (cmd) {
	case BINDER_WRITE_READ:  //binder读写操作
	    //见4.2节
		ret = binder_ioctl_write_read(filp, cmd, arg, thread);
		if (ret)
			goto err;
		break;
	case BINDER_SET_MAX_THREADS: { //Binder线程最大个数
		int max_threads;

		if (copy_from_user(&max_threads, ubuf,
				   sizeof(max_threads))) {
			ret = -EINVAL;
			goto err;
		}
		binder_inner_proc_lock(proc);
		proc-&gt;max_threads = max_threads;
		binder_inner_proc_unlock(proc);
		break;
	} 
	case BINDER_SET_CONTEXT_MGR:  //binder上下文管理者，ServiceManager成为守护进程
		ret = binder_ioctl_set_ctx_mgr(filp);
		if (ret)
			goto err;
		break; 
	case BINDER_THREAD_EXIT:  //退出，释放Binder线程
		binder_debug(BINDER_DEBUG_THREADS, "%d:%d exit/n",
			     proc-&gt;pid, thread-&gt;pid);
		binder_thread_release(proc, thread);
		thread = NULL;
		break;
	case BINDER_VERSION: {  //Binder版本信息
		struct binder_version __user *ver = ubuf;

		if (size != sizeof(struct binder_version)) {
			ret = -EINVAL;
			goto err;
		}
		if (put_user(BINDER_CURRENT_PROTOCOL_VERSION,
			     &ver-&gt;protocol_version)) {
			ret = -EINVAL;
			goto err;
		}
		break;
	}
	case BINDER_GET_NODE_DEBUG_INFO: {  //binder debug信息
		struct binder_node_debug_info info;

		if (copy_from_user(&info, ubuf, sizeof(info))) {
			ret = -EFAULT;
			goto err;
		}

		ret = binder_ioctl_get_node_debug_info(proc, &info);
		if (ret &lt; 0)
			goto err;

		if (copy_to_user(ubuf, &info, sizeof(info))) {
			ret = -EFAULT;
			goto err;
		}
		break;
	}
	default:
		ret = -EINVAL;
		goto err;
	}
	ret = 0;
err:
	if (thread)
		thread-&gt;looper_need_return = false;
	wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error &lt; 2);
	if (ret && ret != -ERESTARTSYS)
		pr_info("%d:%d ioctl %x %lx returned %d/n", proc-&gt;pid, current-&gt;pid, cmd, arg, ret);
err_unlocked:
	trace_binder_ioctl_done(ret);
	return ret;
}</code></pre>

<p> binder_ioctl负责两个进程间收发IPC数据和IPC reply数据</p>
<blockquote>
<p> binder_ioctl(文件描述符，ioctl命令，数据类型)</p>
</blockquote>
<p>文件描述符，是通过binder_open方法打开Binder Driver后的返回值</p>
<p>ioctl命令和数据类型是一体的，不同的命令对应不同的数据类型</p>
<table>
<thead>
<tr>
<th>ioctl命令</th>
<th>数据类型</th>
<th>操作</th>
</tr>
</thead>
<tbody>
<tr>
<td>BINDER_WRITE_READ</td>
<td>struct binder_write_read</td>
<td>收发Binder IPC数据</td>
</tr>
<tr>
<td>BINDER_SET_MAX_THREADS</td>
<td>__u32</td>
<td>Binder线程最大个数</td>
</tr>
<tr>
<td>BINDER_SET_CONTEXT_MGR</td>
<td>__s32</td>
<td>Service Manager节点</td>
</tr>
<tr>
<td>BINDER_THREAD_EXIT</td>
<td>__s32</td>
<td>释放Binder线程</td>
</tr>
<tr>
<td>BINDER_VERSION</td>
<td>struct binder_version</td>
<td>Binder版本信息</td>
</tr>
<tr>
<td>BINDER_GET_NODE_DEBUG_INFO</td>
<td>struct binder_node_debug_info</td>
<td>binder debug信息</td>
</tr>
</tbody>
</table>
<h4 id="4-1-1-binder-get-thread"><a href="#4-1-1-binder-get-thread" class="headerlink" title="4.1.1 binder_get_thread"></a>4.1.1 binder_get_thread</h4><p>[-&gt;android/binder.c]</p>

<pre><code>
static struct binder_thread *binder_get_thread(struct binder_proc *proc)
{
	struct binder_thread *thread;
	struct binder_thread *new_thread;
    //获取锁
	binder_inner_proc_lock(proc);
	//见4.1.2节,获取binder线程
	thread = binder_get_thread_ilocked(proc, NULL);
	//释放锁
	binder_inner_proc_unlock(proc);
	//未获取到线程
	if (!thread) {
	    //新建binder_thread结构体
		new_thread = kzalloc(sizeof(*thread), GFP_KERNEL);
		if (new_thread == NULL)
			return NULL;
		binder_inner_proc_lock(proc);
		thread = binder_get_thread_ilocked(proc, new_thread);
		binder_inner_proc_unlock(proc);
		if (thread != new_thread)
			kfree(new_thread);
	}
	return thread;
}
</code></pre>

<h4 id="4-1-2-binder-get-thread-ilocked"><a href="#4-1-2-binder-get-thread-ilocked" class="headerlink" title="4.1.2 binder_get_thread_ilocked"></a>4.1.2 binder_get_thread_ilocked</h4><p>[-&gt;android/binder.c]</p>

<pre><code>
static struct binder_thread *binder_get_thread_ilocked(
		struct binder_proc *proc, struct binder_thread *new_thread)
{
	struct binder_thread *thread = NULL;
	//红黑树数据结构存储结点
	struct rb_node *parent = NULL;
	struct rb_node **p = &proc-&gt;threads.rb_node;
    //根据当前线程的pid,从binder_proc中查找相应的binder_thread
	while (*p) {
		parent = *p;
		thread = rb_entry(parent, struct binder_thread, rb_node);

		if (current-&gt;pid &lt; thread-&gt;pid)
			p = &(*p)-&gt;rb_left;
		else if (current-&gt;pid &gt; thread-&gt;pid)
			p = &(*p)-&gt;rb_right;
		else
			return thread;
	}
	if (!new_thread)
		return NULL;
	thread = new_thread;
	binder_stats_created(BINDER_STAT_THREAD);
	thread-&gt;proc = proc;
	thread-&gt;pid = current-&gt;pid; //保存当前线程的pid
	get_task_struct(current);
	thread-&gt;task = current;
	atomic_set(&thread-&gt;tmp_ref, 0);
	init_waitqueue_head(&thread-&gt;wait);
	INIT_LIST_HEAD(&thread-&gt;todo);
	rb_link_node(&thread-&gt;rb_node, parent, p);
	rb_insert_color(&thread-&gt;rb_node, &proc-&gt;threads);
	thread-&gt;looper_need_return = true;
	thread-&gt;return_error.work.type = BINDER_WORK_RETURN_ERROR;
	thread-&gt;return_error.cmd = BR_OK;
	thread-&gt;reply_error.work.type = BINDER_WORK_RETURN_ERROR;
	thread-&gt;reply_error.cmd = BR_OK;
	INIT_LIST_HEAD(&new_thread-&gt;waiting_thread_node);
	return thread;
}</code></pre>

<p>从binder_proc中查找binder_thread,如果当前线程已经加入到proc的线程队列则直接返回，如果不存在则创建binder_thread，并将当前线程添加到当期的proc。</p>
<h3 id="4-2-binder-ioctl-write-read"><a href="#4-2-binder-ioctl-write-read" class="headerlink" title="4.2 binder_ioctl_write_read"></a>4.2 binder_ioctl_write_read</h3><p>[-&gt;android/binder.c]</p>

<pre><code>
static int binder_ioctl_write_read(struct file *filp,
				unsigned int cmd, unsigned long arg,
				struct binder_thread *thread)
{
	int ret = 0;
	struct binder_proc *proc = filp-&gt;private_data;
	unsigned int size = _IOC_SIZE(cmd);
	//用户空间binder_write_read
	void __user *ubuf = (void __user *)arg;
	struct binder_write_read bwr;

	if (size != sizeof(struct binder_write_read)) {
		ret = -EINVAL;
		goto out;
	}
	//拷贝用户空间的binder_write_read拷贝到内核
	if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
		ret = -EFAULT;
		goto out;
	}
	binder_debug(BINDER_DEBUG_READ_WRITE,
		     "%d:%d write %lld at %016llx, read %lld at %016llx/n",
		     proc-&gt;pid, thread-&gt;pid,
		     (u64)bwr.write_size, (u64)bwr.write_buffer,
		     (u64)bwr.read_size, (u64)bwr.read_buffer);

	if (bwr.write_size &gt; 0) {
	    //将数据放入目标进程
		ret = binder_thread_write(proc, thread,
					  bwr.write_buffer,
					  bwr.write_size,
					  &bwr.write_consumed);
		trace_binder_write_done(ret);
		if (ret &lt; 0) {
			bwr.read_consumed = 0;
			if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
				ret = -EFAULT;
			goto out;
		}
	}
	if (bwr.read_size &gt; 0) {
	    //读取自己队列数据
		ret = binder_thread_read(proc, thread, bwr.read_buffer,
					 bwr.read_size,
					 &bwr.read_consumed,
					 filp-&gt;f_flags & O_NONBLOCK);
		trace_binder_read_done(ret);
		binder_inner_proc_lock(proc);
		if (!binder_worklist_empty_ilocked(&proc-&gt;todo))
			binder_wakeup_proc_ilocked(proc);
		binder_inner_proc_unlock(proc);
		if (ret &lt; 0) {
			if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
				ret = -EFAULT;
			goto out;
		}
	}
	binder_debug(BINDER_DEBUG_READ_WRITE,
		     "%d:%d wrote %lld of %lld, read return %lld of %lld/n",
		     proc-&gt;pid, thread-&gt;pid,
		     (u64)bwr.write_consumed, (u64)bwr.write_size,
		     (u64)bwr.read_consumed, (u64)bwr.read_size);
	//将内核空间的bwr结构体，拷贝到用户空间     
	if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
		ret = -EFAULT;
		goto out;
	}
out:
	return ret;
}
</code></pre>

<h3 id="4-3-binder-thread-write"><a href="#4-3-binder-thread-write" class="headerlink" title="4.3  binder_thread_write"></a>4.3  binder_thread_write</h3><p>[-&gt;android/binder.c]</p>

<pre><code>
static int binder_thread_write(struct binder_proc *proc,
			struct binder_thread *thread,
			binder_uintptr_t binder_buffer, size_t size,
			binder_size_t *consumed)
{
	uint32_t cmd;
	struct binder_context *context = proc-&gt;context;
	void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
	void __user *ptr = buffer + *consumed;
	void __user *end = buffer + size;

	while (ptr &lt; end && thread-&gt;return_error.cmd == BR_OK) {
		int ret;
         //获取用户空间的cmd命令，此时为BC_TRANSACTION
		if (get_user(cmd, (uint32_t __user *)ptr))
			return -EFAULT;
		ptr += sizeof(uint32_t);
		trace_binder_command(cmd);
		if (_IOC_NR(cmd) &lt; ARRAY_SIZE(binder_stats.bc)) {
			atomic_inc(&binder_stats.bc[_IOC_NR(cmd)]);
			atomic_inc(&proc-&gt;stats.bc[_IOC_NR(cmd)]);
			atomic_inc(&thread-&gt;stats.bc[_IOC_NR(cmd)]);
		}
		switch (cmd) {
		    ...
		    case BC_TRANSACTION:
		    case BC_REPLY: {
			    struct binder_transaction_data tr;
                  //从用户空间获取binder_transaction_data
			    if (copy_from_user(&tr, ptr, sizeof(tr)))
				   return -EFAULT;
			    ptr += sizeof(tr);
			    //见4.3节
			    binder_transaction(proc, thread, &tr,
					   cmd == BC_REPLY, 0);
			    break;
		    }
		    ...
		}
	}
   }</code></pre>

<h3 id="4-4-binder-transaction"><a href="#4-4-binder-transaction" class="headerlink" title="4.4 binder_transaction"></a>4.4 binder_transaction</h3><p>[-&gt;android/binder.c]</p>

<pre><code>
static void binder_transaction(struct binder_proc *proc,
			       struct binder_thread *thread,
			       struct binder_transaction_data *tr, int reply,
			       binder_size_t extra_buffers_size)
{
	int ret;
	struct binder_transaction *t;
	struct binder_work *tcomplete;
	binder_size_t *offp, *off_end, *off_start;
	binder_size_t off_min;
	u8 *sg_bufp, *sg_buf_end;
	struct binder_proc *target_proc = NULL;
	struct binder_thread *target_thread = NULL;
	struct binder_node *target_node = NULL;
	struct binder_transaction *in_reply_to = NULL;
	struct binder_transaction_log_entry *e;
	uint32_t return_error = 0;
	uint32_t return_error_param = 0;
	uint32_t return_error_line = 0;
	struct binder_buffer_object *last_fixup_obj = NULL;
	binder_size_t last_fixup_min_off = 0;
	struct binder_context *context = proc-&gt;context;
	int t_debug_id = atomic_inc_return(&binder_last_id);

	e = binder_transaction_log_add(&binder_transaction_log);
	e-&gt;debug_id = t_debug_id;
	e-&gt;call_type = reply ? 2 : !!(tr-&gt;flags & TF_ONE_WAY);
	e-&gt;from_proc = proc-&gt;pid;
	e-&gt;from_thread = thread-&gt;pid;
	e-&gt;target_handle = tr-&gt;target.handle;
	e-&gt;data_size = tr-&gt;data_size;
	e-&gt;offsets_size = tr-&gt;offsets_size;
	e-&gt;context_name = proc-&gt;context-&gt;name;
     //此处上面传过来reply为false
	if (reply) {
		...
	} else {
	    //上面传过来的handle是0
		if (tr-&gt;target.handle) {
		   ...
		} else {
			mutex_lock(&context-&gt;context_mgr_node_lock);
			//handle为0，找到servicemanager实体
			target_node = context-&gt;binder_context_mgr_node;
			if (target_node)
				target_node = binder_get_node_refs_for_txn(
						target_node, &target_proc,
						&return_error);
			else
				return_error = BR_DEAD_REPLY;
			mutex_unlock(&context-&gt;context_mgr_node_lock);
			if (target_node && target_proc == proc) {
				binder_user_error("%d:%d got transaction to context manager from process owning it/n",
						  proc-&gt;pid, thread-&gt;pid);
				return_error = BR_FAILED_REPLY;
				return_error_param = -EINVAL;
				return_error_line = __LINE__;
				goto err_invalid_target_handle;
			}
		}
		//target_node为空转到err_dead_binder
		if (!target_node) {
			/*
			 * return_error is set above
			 */
			return_error_param = -EINVAL;
			return_error_line = __LINE__;
			goto err_dead_binder;
		}
		e-&gt;to_node = target_node-&gt;debug_id;
		if (security_binder_transaction(proc-&gt;tsk,
						target_proc-&gt;tsk) &lt; 0) {
			return_error = BR_FAILED_REPLY;
			return_error_param = -EPERM;
			return_error_line = __LINE__;
			goto err_invalid_target_handle;
		}
		binder_inner_proc_lock(proc);
		if (!(tr-&gt;flags & TF_ONE_WAY) && thread-&gt;transaction_stack) {
			struct binder_transaction *tmp;

			tmp = thread-&gt;transaction_stack;
			if (tmp-&gt;to_thread != thread) {
				spin_lock(&tmp-&gt;lock);
				binder_user_error("%d:%d got new transaction with bad transaction stack, transaction %d has target %d:%d/n",
					proc-&gt;pid, thread-&gt;pid, tmp-&gt;debug_id,
					tmp-&gt;to_proc ? tmp-&gt;to_proc-&gt;pid : 0,
					tmp-&gt;to_thread ?
					tmp-&gt;to_thread-&gt;pid : 0);
				spin_unlock(&tmp-&gt;lock);
				binder_inner_proc_unlock(proc);
				return_error = BR_FAILED_REPLY;
				return_error_param = -EPROTO;
				return_error_line = __LINE__;
				goto err_bad_call_stack;
			}
			while (tmp) {
				struct binder_thread *from;

				spin_lock(&tmp-&gt;lock);
				from = tmp-&gt;from;
				if (from && from-&gt;proc == target_proc) {
					atomic_inc(&from-&gt;tmp_ref);
					target_thread = from;
					spin_unlock(&tmp-&gt;lock);
					break;
				}
				spin_unlock(&tmp-&gt;lock);
				tmp = tmp-&gt;from_parent;
			}
		}
		binder_inner_proc_unlock(proc);
	}
	if (target_thread)
		e-&gt;to_thread = target_thread-&gt;pid;
	e-&gt;to_proc = target_proc-&gt;pid;

	/* TODO: reuse incoming transaction for reply */
	t = kzalloc(sizeof(*t), GFP_KERNEL);
	if (t == NULL) {
		return_error = BR_FAILED_REPLY;
		return_error_param = -ENOMEM;
		return_error_line = __LINE__;
		goto err_alloc_t_failed;
	}
	binder_stats_created(BINDER_STAT_TRANSACTION);
	spin_lock_init(&t-&gt;lock);

	tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
	if (tcomplete == NULL) {
		return_error = BR_FAILED_REPLY;
		return_error_param = -ENOMEM;
		return_error_line = __LINE__;
		goto err_alloc_tcomplete_failed;
	}
	binder_stats_created(BINDER_STAT_TRANSACTION_COMPLETE);
	t-&gt;debug_id = t_debug_id;
	...
	//非oneway通信方式，把当前thread保存到transaction的from字段
	if (!reply && !(tr-&gt;flags & TF_ONE_WAY))
		t-&gt;from = thread;
	else
		t-&gt;from = NULL;
	t-&gt;sender_euid = task_euid(proc-&gt;tsk);
	t-&gt;to_proc = target_proc;//此次通信目标进程为servicemanager进程
	t-&gt;to_thread = target_thread; //此次通信的目标线程为servicemanager线程
	t-&gt;code = tr-&gt;code; //此次通信code=ADD_SERVICE_TRANSACTION
	t-&gt;flags = tr-&gt;flags; //此次flag = 0
	if (!(t-&gt;flags & TF_ONE_WAY) &&
	    binder_supported_policy(current-&gt;policy)) {
		/* Inherit supported policies for synchronous transactions */
		t-&gt;priority.sched_policy = current-&gt;policy;
		t-&gt;priority.prio = current-&gt;normal_prio;
	} else {
		/* Otherwise, fall back to the default priority */
		t-&gt;priority = target_proc-&gt;default_priority;
	}

	trace_binder_transaction(reply, t, target_node);
    //从servicemanage进程中分配buffer
	t-&gt;buffer = binder_alloc_new_buf(&target_proc-&gt;alloc, tr-&gt;data_size,
		tr-&gt;offsets_size, extra_buffers_size,
		!reply && (t-&gt;flags & TF_ONE_WAY));
	if (IS_ERR(t-&gt;buffer)) {
		/*
		 * -ESRCH indicates VMA cleared. The target is dying.
		 */
		return_error_param = PTR_ERR(t-&gt;buffer);
		return_error = return_error_param == -ESRCH ?
			BR_DEAD_REPLY : BR_FAILED_REPLY;
		return_error_line = __LINE__;
		t-&gt;buffer = NULL;
		goto err_binder_alloc_buf_failed;
	}
	t-&gt;buffer-&gt;allow_user_free = 0;
	t-&gt;buffer-&gt;debug_id = t-&gt;debug_id;
	t-&gt;buffer-&gt;transaction = t;
	t-&gt;buffer-&gt;target_node = target_node;
	trace_binder_transaction_alloc_buf(t-&gt;buffer);
	off_start = (binder_size_t *)(t-&gt;buffer-&gt;data +
				      ALIGN(tr-&gt;data_size, sizeof(void *)));
	offp = off_start;
    //分别拷贝用户空间的binder_transaction_data中的ptr.buffer和ptr.offsets
	if (copy_from_user(t-&gt;buffer-&gt;data, (const void __user *)(uintptr_t)
			   tr-&gt;data.ptr.buffer, tr-&gt;data_size)) {
		binder_user_error("%d:%d got transaction with invalid data ptr/n",
				proc-&gt;pid, thread-&gt;pid);
		return_error = BR_FAILED_REPLY;
		return_error_param = -EFAULT;
		return_error_line = __LINE__;
		goto err_copy_data_failed;
	}
	if (copy_from_user(offp, (const void __user *)(uintptr_t)
			   tr-&gt;data.ptr.offsets, tr-&gt;offsets_size)) {
		binder_user_error("%d:%d got transaction with invalid offsets ptr/n",
				proc-&gt;pid, thread-&gt;pid);
		return_error = BR_FAILED_REPLY;
		return_error_param = -EFAULT;
		return_error_line = __LINE__;
		goto err_copy_data_failed;
	}
	if (!IS_ALIGNED(tr-&gt;offsets_size, sizeof(binder_size_t))) {
		binder_user_error("%d:%d got transaction with invalid offsets size, %lld/n",
				proc-&gt;pid, thread-&gt;pid, (u64)tr-&gt;offsets_size);
		return_error = BR_FAILED_REPLY;
		return_error_param = -EINVAL;
		return_error_line = __LINE__;
		goto err_bad_offset;
	}
	if (!IS_ALIGNED(extra_buffers_size, sizeof(u64))) {
		binder_user_error("%d:%d got transaction with unaligned buffers size, %lld/n",
				  proc-&gt;pid, thread-&gt;pid,
				  (u64)extra_buffers_size);
		return_error = BR_FAILED_REPLY;
		return_error_param = -EINVAL;
		return_error_line = __LINE__;
		goto err_bad_offset;
	}
	off_end = (void *)off_start + tr-&gt;offsets_size;
	sg_bufp = (u8 *)(PTR_ALIGN(off_end, sizeof(void *)));
	sg_buf_end = sg_bufp + extra_buffers_size;
	off_min = 0;
	for (; offp &lt; off_end; offp++) {
		struct binder_object_header *hdr;
		size_t object_size = binder_validate_object(t-&gt;buffer, *offp);

		if (object_size == 0 || *offp &lt; off_min) {
			binder_user_error("%d:%d got transaction with invalid offset (%lld, min %lld max %lld) or object./n",
					  proc-&gt;pid, thread-&gt;pid, (u64)*offp,
					  (u64)off_min,
					  (u64)t-&gt;buffer-&gt;data_size);
			return_error = BR_FAILED_REPLY;
			return_error_param = -EINVAL;
			return_error_line = __LINE__;
			goto err_bad_offset;
		}

		hdr = (struct binder_object_header *)(t-&gt;buffer-&gt;data + *offp);
		off_min = *offp + object_size;
		switch (hdr-&gt;type) {
		case BINDER_TYPE_BINDER:
		case BINDER_TYPE_WEAK_BINDER: {
			struct flat_binder_object *fp;
			fp = to_flat_binder_object(hdr);
			//见4.4.1节
			ret = binder_translate_binder(fp, t, thread);
			if (ret &lt; 0) {
				return_error = BR_FAILED_REPLY;
				return_error_param = ret;
				return_error_line = __LINE__;
				goto err_translate_failed;
			}
		} break;
		...
	}
	//将当前线程的type为BINDER_WORK_TRANSACTION_COMPLETE
	tcomplete-&gt;type = BINDER_WORK_TRANSACTION_COMPLETE;
	//设置目标线程type为BINDER_WORK_TRANSACTION
	t-&gt;work.type = BINDER_WORK_TRANSACTION;

	if (reply) {
		...
	} else if (!(t-&gt;flags & TF_ONE_WAY)) {
	    //BC_TRANSACTION且oneway，则设置事务栈信息
		BUG_ON(t-&gt;buffer-&gt;async_transaction != 0);
		binder_inner_proc_lock(proc);
		/*
		 * Defer the TRANSACTION_COMPLETE, so we don't return to
		 * userspace immediately; this allows the target process to
		 * immediately start processing this transaction, reducing
		 * latency. We will then return the TRANSACTION_COMPLETE when
		 * the target replies (or there is an error).
		 */
		 //BINDER_WORK_TRANSACTION_COMPLETE加入到当前线程todo队列
		binder_enqueue_deferred_thread_work_ilocked(thread, tcomplete);
		t-&gt;need_reply = 1;
		t-&gt;from_parent = thread-&gt;transaction_stack;
		thread-&gt;transaction_stack = t;
		binder_inner_proc_unlock(proc);
		//见4.4.2节，BINDER_WORK_TRANSACTION加入到目标线程todo队列
		if (!binder_proc_transaction(t, target_proc, target_thread)) {
			binder_inner_proc_lock(proc);
			binder_pop_transaction_ilocked(thread, t);
			binder_inner_proc_unlock(proc);
			goto err_dead_proc_or_thread;
		}
	} else {
		...
	}
	if (target_thread)
		binder_thread_dec_tmpref(target_thread);
	binder_proc_dec_tmpref(target_proc);
	if (target_node)
		binder_dec_node_tmpref(target_node);
	/*
	 * write barrier to synchronize with initialization
	 * of log entry
	 */
	smp_wmb();
	WRITE_ONCE(e-&gt;debug_id_done, t_debug_id);
	return;
     ...
}
</code></pre>

<p>binder_transaction主要的工作如下</p>
<ul>
<li><p>注册服务的过程，传递的是BBinder对象，所以hdr-&gt;type为BINDER_TYPE_BINDER；</p>
</li>
<li><p>服务注册过程中是在服务所在的进程创建binder_node，在Servicemanager进程创建binder_ref,对应一个binder_node，每个进程只会创建一个binder_ref对象；</p>
</li>
<li><p>向当前线程的binder_proc-&gt;todo添加BINDER_WORK_TRANSACTION_COMPLETE事务，回复服务注册进程收到BC_TRANSACTION；</p>
</li>
<li><p>向servicemanager的binder_proc-&gt;todo添加BINDER_WORK_TRANSACTION事务，接下来进入到servicemanager进程中。</p>
</li>
</ul>
<h4 id="4-4-1-binder-translate-binder"><a href="#4-4-1-binder-translate-binder" class="headerlink" title="4.4.1  binder_translate_binder"></a>4.4.1  binder_translate_binder</h4><p>[-&gt;android/binder.c]</p>

<pre><code>
static int binder_translate_binder(struct flat_binder_object *fp,
				   struct binder_transaction *t,
				   struct binder_thread *thread)
{
	struct binder_node *node;
	struct binder_proc *proc = thread-&gt;proc;
	struct binder_proc *target_proc = t-&gt;to_proc;
	struct binder_ref_data rdata;
	int ret = 0;
    //获取binder实体
	node = binder_get_node(proc, fp-&gt;binder);
	if (!node) {
	    //创建binder实体
		node = binder_new_node(proc, fp);
		if (!node)
			return -ENOMEM;
	}
	if (fp-&gt;cookie != node-&gt;cookie) {
		binder_user_error("%d:%d sending u%016llx node %d, cookie mismatch %016llx != %016llx/n",
				  proc-&gt;pid, thread-&gt;pid, (u64)fp-&gt;binder,
				  node-&gt;debug_id, (u64)fp-&gt;cookie,
				  (u64)node-&gt;cookie);
		ret = -EINVAL;
		goto done;
	}
	if (security_binder_transfer_binder(proc-&gt;tsk, target_proc-&gt;tsk)) {
		ret = -EPERM;
		goto done;
	}
    //servicemanager进程的binder_ref
	ret = binder_inc_ref_for_node(target_proc, node,
			fp-&gt;hdr.type == BINDER_TYPE_BINDER,
			&thread-&gt;todo, &rdata);
	if (ret)
		goto done;
     //调整type为HANDLE类型
	if (fp-&gt;hdr.type == BINDER_TYPE_BINDER)
		fp-&gt;hdr.type = BINDER_TYPE_HANDLE;
	else
		fp-&gt;hdr.type = BINDER_TYPE_WEAK_HANDLE;
	fp-&gt;binder = 0;
	fp-&gt;handle = rdata.desc;  //设置handler值
	fp-&gt;cookie = 0;

	trace_binder_transaction_node_to_ref(t, node, &rdata);
	binder_debug(BINDER_DEBUG_TRANSACTION,
		     "        node %d u%016llx -&gt; ref %d desc %d/n",
		     node-&gt;debug_id, (u64)node-&gt;ptr,
		     rdata.debug_id, rdata.desc);
done:
	binder_put_node(node);
	return ret;
}</code></pre>

<h4 id="4-4-2-binder-proc-transaction"><a href="#4-4-2-binder-proc-transaction" class="headerlink" title="4.4.2  binder_proc_transaction"></a>4.4.2  binder_proc_transaction</h4><p>[-&gt;android/binder.c]</p>

<pre><code>
static bool binder_proc_transaction(struct binder_transaction *t,
				    struct binder_proc *proc,
				    struct binder_thread *thread)
{
	struct binder_node *node = t-&gt;buffer-&gt;target_node;
	struct binder_priority node_prio;
	bool oneway = !!(t-&gt;flags & TF_ONE_WAY);
	bool pending_async = false;

	BUG_ON(!node);
	binder_node_lock(node);
	node_prio.prio = node-&gt;min_priority;
	node_prio.sched_policy = node-&gt;sched_policy;

	if (oneway) {
		BUG_ON(thread);
		if (node-&gt;has_async_transaction) {
			pending_async = true;
		} else {
			node-&gt;has_async_transaction = 1;
		}
	}

	binder_inner_proc_lock(proc);

	if (proc-&gt;is_dead || (thread && thread-&gt;is_dead)) {
		binder_inner_proc_unlock(proc);
		binder_node_unlock(node);
		return false;
	}

	if (!thread && !pending_async)
		thread = binder_select_thread_ilocked(proc);

	if (thread) {
		binder_transaction_priority(thread-&gt;task, t, node_prio,
					    node-&gt;inherit_rt);
		//BINDER_WORK_TRANSACTION加入到目标线程todo队列
		binder_enqueue_thread_work_ilocked(thread, &t-&gt;work);
	} else if (!pending_async) {
		binder_enqueue_work_ilocked(&t-&gt;work, &proc-&gt;todo);
	} else {
		binder_enqueue_work_ilocked(&t-&gt;work, &node-&gt;async_todo);
	}

	if (!pending_async)
	    //唤醒等待队列，本次通信的目标队列为target_proc-&gt;wait
		binder_wakeup_thread_ilocked(proc, thread, !oneway /* sync */);

	binder_inner_proc_unlock(proc);
	binder_node_unlock(node);

	return true;
}
</code></pre>

<p>4.2节中执行完binder_thread_write，再执行binder_thread_read操作,见4.5节</p>
<h4 id="4-4-3-binder-inc-ref-for-node"><a href="#4-4-3-binder-inc-ref-for-node" class="headerlink" title="4.4.3 binder_inc_ref_for_node"></a>4.4.3 binder_inc_ref_for_node</h4><p>[-&gt;android/binder.c]</p>

<pre><code>
static int binder_inc_ref_for_node(struct binder_proc *proc,
			struct binder_node *node,
			bool strong,
			struct list_head *target_list,
			struct binder_ref_data *rdata)
{
	struct binder_ref *ref;
	struct binder_ref *new_ref = NULL;
	int ret = 0;

	binder_proc_lock(proc);
	//获取binder_ref,见4.4.4节
	ref = binder_get_ref_for_node_olocked(proc, node, NULL);
	if (!ref) {
		binder_proc_unlock(proc);
		new_ref = kzalloc(sizeof(*ref), GFP_KERNEL);
		if (!new_ref)
			return -ENOMEM;
		binder_proc_lock(proc);
		ref = binder_get_ref_for_node_olocked(proc, node, new_ref);
	}
	ret = binder_inc_ref_olocked(ref, strong, target_list);
	*rdata = ref-&gt;data;
	binder_proc_unlock(proc);
	if (new_ref && ref != new_ref)
		/*
		 * Another thread created the ref first so
		 * free the one we allocated
		 */
		kfree(new_ref);
	return ret;
}</code></pre>

<h4 id="4-4-4-binder-get-ref-for-node-olocked"><a href="#4-4-4-binder-get-ref-for-node-olocked" class="headerlink" title="4.4.4 binder_get_ref_for_node_olocked"></a>4.4.4 binder_get_ref_for_node_olocked</h4><p>[-&gt;android/binder.c]</p>

<pre><code>
static struct binder_ref *binder_get_ref_for_node_olocked(
					struct binder_proc *proc,
					struct binder_node *node,
					struct binder_ref *new_ref)
{
	struct binder_context *context = proc-&gt;context;
	struct rb_node **p = &proc-&gt;refs_by_node.rb_node;
	struct rb_node *parent = NULL;
	struct binder_ref *ref;
	struct rb_node *n;
    //从refs_by_node红黑树，找到binder_ref则直接返回
	while (*p) {
		parent = *p;
		ref = rb_entry(parent, struct binder_ref, rb_node_node);

		if (node &lt; ref-&gt;node)
			p = &(*p)-&gt;rb_left;
		else if (node &gt; ref-&gt;node)
			p = &(*p)-&gt;rb_right;
		else
			return ref;
	}
	if (!new_ref)
		return NULL;

	binder_stats_created(BINDER_STAT_REF);
	new_ref-&gt;data.debug_id = atomic_inc_return(&binder_last_id);
	new_ref-&gt;proc = proc;
	new_ref-&gt;node = node;
	rb_link_node(&new_ref-&gt;rb_node_node, parent, p);
	rb_insert_color(&new_ref-&gt;rb_node_node, &proc-&gt;refs_by_node);
    
    //计算binder引用的handle值
	new_ref-&gt;data.desc = (node == context-&gt;binder_context_mgr_node) ? 0 : 1;
	//从红黑树最左边的handle对比，依次递增，直到红黑树结束或者找到更大的handle结束
	for (n = rb_first(&proc-&gt;refs_by_desc); n != NULL; n = rb_next(n)) {
	//根据binder_ref的成员变量rb_node_desc地址指针，来获取binder_ref首地址
		ref = rb_entry(n, struct binder_ref, rb_node_desc);
		if (ref-&gt;data.desc &gt; new_ref-&gt;data.desc)
			break;
		new_ref-&gt;data.desc = ref-&gt;data.desc + 1;
	}

	p = &proc-&gt;refs_by_desc.rb_node;
	while (*p) {
		parent = *p;
		ref = rb_entry(parent, struct binder_ref, rb_node_desc);

		if (new_ref-&gt;data.desc &lt; ref-&gt;data.desc)
			p = &(*p)-&gt;rb_left;
		else if (new_ref-&gt;data.desc &gt; ref-&gt;data.desc)
			p = &(*p)-&gt;rb_right;
		else
			BUG();
	}
	rb_link_node(&new_ref-&gt;rb_node_desc, parent, p);
	rb_insert_color(&new_ref-&gt;rb_node_desc, &proc-&gt;refs_by_desc);

	binder_node_lock(node);
	hlist_add_head(&new_ref-&gt;node_entry, &node-&gt;refs);

	binder_debug(BINDER_DEBUG_INTERNAL_REFS,
		     "%d new ref %d desc %d for node %d/n",
		      proc-&gt;pid, new_ref-&gt;data.debug_id, new_ref-&gt;data.desc,
		      node-&gt;debug_id);
	binder_node_unlock(node);
	return new_ref;
}
</code></pre>

<p>handle值计算方法规律：</p>
<ul>
<li>每个进程binder_proc记录的binder_ref的handle值是从1开始递增的</li>
<li>所有进程的binder_proc所记录的handler=0的binder_ref都指向service manage</li>
<li>同一个服务的binder_node在不同进程的binder_ref的handle值可以不同</li>
</ul>
<h3 id="4-5-binder-thread-read"><a href="#4-5-binder-thread-read" class="headerlink" title="4.5 binder_thread_read"></a>4.5 binder_thread_read</h3><p>[-&gt;android/binder.c]</p>

<pre><code>
static int binder_thread_read(struct binder_proc *proc,
			      struct binder_thread *thread,
			      binder_uintptr_t binder_buffer, size_t size,
			      binder_size_t *consumed, int non_block)
{
	void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
	void __user *ptr = buffer + *consumed;
	void __user *end = buffer + size;

	int ret = 0;
	int wait_for_proc_work;

	if (*consumed == 0) {
		if (put_user(BR_NOOP, (uint32_t __user *)ptr))
			return -EFAULT;
		ptr += sizeof(uint32_t);
	}

retry:
	binder_inner_proc_lock(proc);
	wait_for_proc_work = binder_available_for_proc_work_ilocked(thread);
	binder_inner_proc_unlock(proc);

	thread-&gt;looper |= BINDER_LOOPER_STATE_WAITING;

	trace_binder_wait_for_work(wait_for_proc_work,
				   !!thread-&gt;transaction_stack,
				   !binder_worklist_empty(proc, &thread-&gt;todo));
	if (wait_for_proc_work) {
		if (!(thread-&gt;looper & (BINDER_LOOPER_STATE_REGISTERED |
					BINDER_LOOPER_STATE_ENTERED))) {
			binder_user_error("%d:%d ERROR: Thread waiting for process work before calling BC_REGISTER_LOOPER or BC_ENTER_LOOPER (state %x)/n",
				proc-&gt;pid, thread-&gt;pid, thread-&gt;looper);
			wait_event_interruptible(binder_user_error_wait,
						 binder_stop_on_user_error &lt; 2);
		}
		binder_restore_priority(current, proc-&gt;default_priority);
	}

	if (non_block) {
		if (!binder_has_work(thread, wait_for_proc_work))
			ret = -EAGAIN;
	} else {
		ret = binder_wait_for_work(thread, wait_for_proc_work);
	}

	thread-&gt;looper &= ~BINDER_LOOPER_STATE_WAITING;

	if (ret)
		return ret;

	while (1) {
		uint32_t cmd;
		struct binder_transaction_data tr;
		struct binder_work *w = NULL;
		struct list_head *list = NULL;
		struct binder_transaction *t = NULL;
		struct binder_thread *t_from;

		binder_inner_proc_lock(proc);
		if (!binder_worklist_empty_ilocked(&thread-&gt;todo))
			list = &thread-&gt;todo;
		else if (!binder_worklist_empty_ilocked(&proc-&gt;todo) &&
			   wait_for_proc_work)
			list = &proc-&gt;todo;
		else {
			binder_inner_proc_unlock(proc);

			/* no data added */
			if (ptr - buffer == 4 && !thread-&gt;looper_need_return)
				goto retry;
			break;
		}

		if (end - ptr &lt; sizeof(tr) + 4) {
			binder_inner_proc_unlock(proc);
			break;
		}
		//上面binder_transaction中将BINDER_WORK_TRANSACTION_COMPLETE事务添加到了todo队列
		w = binder_dequeue_work_head_ilocked(list);
		if (binder_worklist_empty_ilocked(&thread-&gt;todo))
			thread-&gt;process_todo = false;
         //这里是BINDER_WORK_TRANSACTION_COMPLETE
		switch (w-&gt;type) {
		...
		case BINDER_WORK_TRANSACTION_COMPLETE: {
			binder_inner_proc_unlock(proc);
			//cmd赋值，回复客服端
			cmd = BR_TRANSACTION_COMPLETE;
			//将cmd和数据写回用户空间
			if (put_user(cmd, (uint32_t __user *)ptr))
				return -EFAULT;
			ptr += sizeof(uint32_t);

			binder_stat_br(proc, thread, cmd);
			binder_debug(BINDER_DEBUG_TRANSACTION_COMPLETE,
				     "%d:%d BR_TRANSACTION_COMPLETE/n",
				     proc-&gt;pid, thread-&gt;pid);
			kfree(w);
			binder_stats_deleted(BINDER_STAT_TRANSACTION_COMPLETE);
		} break;
		...
		}
         //binder_transaction为空继续
		if (!t)
			continue;

		BUG_ON(t-&gt;buffer == NULL);
		if (t-&gt;buffer-&gt;target_node) {
		     //获取目标node
			struct binder_node *target_node = t-&gt;buffer-&gt;target_node;
			struct binder_priority node_prio;

			tr.target.ptr = target_node-&gt;ptr;
			tr.cookie =  target_node-&gt;cookie;
			node_prio.sched_policy = target_node-&gt;sched_policy;
			node_prio.prio = target_node-&gt;min_priority;
			binder_transaction_priority(current, t, node_prio,
						    target_node-&gt;inherit_rt);
			cmd = BR_TRANSACTION; //设置目标cmd为BR_TRANSACTION
		} else {
			tr.target.ptr = 0;
			tr.cookie = 0;
			cmd = BR_REPLY; //设置目标cmd为BR_REPLY
		}
		tr.code = t-&gt;code;
		tr.flags = t-&gt;flags;
		tr.sender_euid = from_kuid(current_user_ns(), t-&gt;sender_euid);

		t_from = binder_get_txn_from(t);
		if (t_from) {
			struct task_struct *sender = t_from-&gt;proc-&gt;tsk;

			tr.sender_pid = task_tgid_nr_ns(sender,
							task_active_pid_ns(current));
		} else {
			tr.sender_pid = 0;
		}

		tr.data_size = t-&gt;buffer-&gt;data_size;
		tr.offsets_size = t-&gt;buffer-&gt;offsets_size;
		tr.data.ptr.buffer = (binder_uintptr_t)
			((uintptr_t)t-&gt;buffer-&gt;data +
			binder_alloc_get_user_buffer_offset(&proc-&gt;alloc));
		tr.data.ptr.offsets = tr.data.ptr.buffer +
					ALIGN(t-&gt;buffer-&gt;data_size,
					    sizeof(void *));

         //将cmd和数据写回用户空间
		if (put_user(cmd, (uint32_t __user *)ptr)) {
			if (t_from)
				binder_thread_dec_tmpref(t_from);

			binder_cleanup_transaction(t, "put_user failed",
						   BR_FAILED_REPLY);

			return -EFAULT;
		}
		ptr += sizeof(uint32_t);
		//将内核空间的binder_transaction_data，拷贝到用户空间   
		if (copy_to_user(ptr, &tr, sizeof(tr))) {
			if (t_from)
				binder_thread_dec_tmpref(t_from);

			binder_cleanup_transaction(t, "copy_to_user failed",
						   BR_FAILED_REPLY);

			return -EFAULT;
		}
		ptr += sizeof(tr);

		trace_binder_transaction_received(t);
		binder_stat_br(proc, thread, cmd);
		binder_debug(BINDER_DEBUG_TRANSACTION,
			     "%d:%d %s %d %d:%d, cmd %d size %zd-%zd ptr %016llx-%016llx/n",
			     proc-&gt;pid, thread-&gt;pid,
			     (cmd == BR_TRANSACTION) ? "BR_TRANSACTION" :
			     "BR_REPLY",
			     t-&gt;debug_id, t_from ? t_from-&gt;proc-&gt;pid : 0,
			     t_from ? t_from-&gt;pid : 0, cmd,
			     t-&gt;buffer-&gt;data_size, t-&gt;buffer-&gt;offsets_size,
			     (u64)tr.data.ptr.buffer, (u64)tr.data.ptr.offsets);

		if (t_from)
			binder_thread_dec_tmpref(t_from);
		t-&gt;buffer-&gt;allow_user_free = 1;
		if (cmd == BR_TRANSACTION && !(t-&gt;flags & TF_ONE_WAY)) {
			binder_inner_proc_lock(thread-&gt;proc);
			t-&gt;to_parent = thread-&gt;transaction_stack;
			t-&gt;to_thread = thread;
			thread-&gt;transaction_stack = t;
			binder_inner_proc_unlock(thread-&gt;proc);
		} else {
			binder_free_transaction(t);
		}
		break;
	}

done:

	*consumed = ptr - buffer;
	binder_inner_proc_lock(proc);
	if (proc-&gt;requested_threads == 0 &&
	    list_empty(&thread-&gt;proc-&gt;waiting_threads) &&
	    proc-&gt;requested_threads_started &lt; proc-&gt;max_threads &&
	    (thread-&gt;looper & (BINDER_LOOPER_STATE_REGISTERED |
	     BINDER_LOOPER_STATE_ENTERED)) /* the user-space code fails to */
	     /*spawn a new thread if we leave this out */) {
		proc-&gt;requested_threads++;
		binder_inner_proc_unlock(proc);
		binder_debug(BINDER_DEBUG_THREADS,
			     "%d:%d BR_SPAWN_LOOPER/n",
			     proc-&gt;pid, thread-&gt;pid);
		if (put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer))
			return -EFAULT;
		binder_stat_br(proc, thread, BR_SPAWN_LOOPER);
	} elseB
		binder_inner_proc_unlock(proc);
	return 0;
}</code></pre>

<p>这里binder_thread_read主要是读取队列中的事务，将BR_TRANSACTION_COMPLETE命令写回客户端的用户空间，并且将BR_TRANSACTION命令写回服务端的用户空间。</p>
<p>下面看下服务端进程servicemanager。</p>
<h2 id="五、ServiceManager进程"><a href="#五、ServiceManager进程" class="headerlink" title="五、ServiceManager进程"></a>五、ServiceManager进程</h2><p>从第二大节中可以看到servicemanager启动后，循环再binder_loop过程，会调用binder_parse方法</p>
<h3 id="5-1-binder-parse"><a href="#5-1-binder-parse" class="headerlink" title="5.1 binder_parse"></a>5.1 binder_parse</h3><p>[-&gt;servicemanager/binder.c]</p>

<pre><code>
int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uintptr_t ptr, size_t size, binder_handler func)
{
    int r = 1;
    uintptr_t end = ptr + (uintptr_t) size;

    while (ptr &lt; end) {
        uint32_t cmd = *(uint32_t *) ptr;
        ptr += sizeof(uint32_t);
#if TRACE
        fprintf(stderr,"%s:/n", cmd_name(cmd));
#endif
        switch(cmd) {
        ...
        //从上面传过来的是BR_TRANSACTION
        case BR_TRANSACTION: {
            struct binder_transaction_data *txn = (struct binder_transaction_data *) ptr;
            if ((end - ptr) &lt; sizeof(*txn)) {
                ALOGE("parse: txn too small!/n");
                return -1;
            }
            binder_dump_txn(txn);
            if (func) {
                unsigned rdata[256/4];
                struct binder_io msg;
                struct binder_io reply;
                int res;

                bio_init(&reply, rdata, sizeof(rdata), 4);
                //从txn解析出binder_io信息
                bio_init_from_txn(&msg, txn);
                //收到Binder事务
                res = func(bs, txn, &msg, &reply);
                //这里走的是非oneway
                if (txn-&gt;flags & TF_ONE_WAY) {
                    binder_free_buffer(bs, txn-&gt;data.ptr.buffer);
                } else {
                    //发送reply事件
                    binder_send_reply(bs, &reply, txn-&gt;data.ptr.buffer, res);
                }
            }
            ptr += sizeof(*txn);
            break;
        }
        ...
        }
    }

    return r;
}</code></pre>

<h3 id="5-2-svcmgr-handler"><a href="#5-2-svcmgr-handler" class="headerlink" title="5.2 svcmgr_handler"></a>5.2 svcmgr_handler</h3><p>[-&gt;service_manager.c]</p>

<pre><code>
int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data *txn,
                   struct binder_io *msg,
                   struct binder_io *reply)
{
    struct svcinfo *si;
    uint16_t *s;
    size_t len;
    uint32_t handle;
    uint32_t strict_policy;
    int allow_isolated;
    uint32_t dumpsys_priority;

    //ALOGI("target=%p code=%d pid=%d uid=%d/n",
    //      (void*) txn-&gt;target.ptr, txn-&gt;code, txn-&gt;sender_pid, txn-&gt;sender_euid);

    if (txn-&gt;target.ptr != BINDER_SERVICE_MANAGER)
        return -1;

    if (txn-&gt;code == PING_TRANSACTION)
        return 0;

    // Equivalent to Parcel::enforceInterface(), reading the RPC
    // header with the strict mode policy mask and the interface name.
    // Note that we ignore the strict_policy and don't propagate it
    // further (since we do no outbound RPCs anyway).
    strict_policy = bio_get_uint32(msg);
    s = bio_get_string16(msg, &len);
    if (s == NULL) {
        return -1;
    }

    if ((len != (sizeof(svcmgr_id) / 2)) ||
        memcmp(svcmgr_id, s, sizeof(svcmgr_id))) {
        fprintf(stderr,"invalid id %s/n", str8(s, len));
        return -1;
    }

    if (sehandle && selinux_status_updated() &gt; 0) {
#ifdef VENDORSERVICEMANAGER
        struct selabel_handle *tmp_sehandle = selinux_android_vendor_service_context_handle();
#else
        struct selabel_handle *tmp_sehandle = selinux_android_service_context_handle();
#endif
        if (tmp_sehandle) {
            selabel_close(sehandle);
            sehandle = tmp_sehandle;
        }
    }

    switch(txn-&gt;code) {
    ...
    //这里是addservice
    case SVC_MGR_ADD_SERVICE:
        s = bio_get_string16(msg, &len);
        if (s == NULL) {
            return -1;
        }
        //获取handle
        handle = bio_get_ref(msg);
        allow_isolated = bio_get_uint32(msg) ? 1 : 0;
        dumpsys_priority = bio_get_uint32(msg);
        //注册服务，见5.3节
        if (do_add_service(bs, s, len, handle, txn-&gt;sender_euid, allow_isolated, dumpsys_priority,
                           txn-&gt;sender_pid))
            return -1;
        break;

  
    }

    bio_put_uint32(reply, 0);
    return 0;
}</code></pre>

<h3 id="5-3-do-add-service"><a href="#5-3-do-add-service" class="headerlink" title="5.3 do_add_service"></a>5.3 do_add_service</h3><p>[-&gt;service_manager.c]</p>

<pre><code>
int do_add_service(struct binder_state *bs, const uint16_t *s, size_t len, uint32_t handle,
                   uid_t uid, int allow_isolated, uint32_t dumpsys_priority, pid_t spid) {
    struct svcinfo *si;

    //ALOGI("add_service('%s',%x,%s) uid=%d/n", str8(s, len), handle,
    //        allow_isolated ? "allow_isolated" : "!allow_isolated", uid);

    if (!handle || (len == 0) || (len &gt; 127))
        return -1;

    //权限检查
    if (!svc_can_register(s, len, spid, uid)) {
        ALOGE("add_service('%s',%x) uid=%d - PERMISSION DENIED/n",
             str8(s, len), handle, uid);
        return -1;
    }
    //服务检索
    si = find_svc(s, len);
    if (si) {
        if (si-&gt;handle) {
            ALOGE("add_service('%s',%x) uid=%d - ALREADY REGISTERED, OVERRIDE/n",
                 str8(s, len), handle, uid);
            svcinfo_death(bs, si);//服务已经注册时，释放相应的服务
        }
        si-&gt;handle = handle;
    } else {
        si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));
        if (!si) { //内存不足，无法分配足够内存
            ALOGE("add_service('%s',%x) uid=%d - OUT OF MEMORY/n",
                 str8(s, len), handle, uid);
            return -1;
        }
        si-&gt;handle = handle;
        si-&gt;len = len;
        //拷贝内存服务信息
        memcpy(si-&gt;name, s, (len + 1) * sizeof(uint16_t));
        si-&gt;name[len] = '/0';
        si-&gt;death.func = (void*) svcinfo_death;
        si-&gt;death.ptr = si;
        si-&gt;allow_isolated = allow_isolated;
        si-&gt;dumpsys_priority = dumpsys_priority;
        si-&gt;next = svclist; //svclist保存所有已经注册的服务
        svclist = si;
    }
    //BC_ACQUIRE为命令，handle为目标的信息通过ioct发送给binder驱动
    binder_acquire(bs, handle);
    //BC_REQUEST_DEATH_NOTIFICATION为命令的信息通过ioct发送给binder驱动
    //主要用于清理内存等收尾工作
    binder_link_to_death(bs, handle, &si-&gt;death);
    return 0;
}</code></pre>

<h3 id="5-4-binder-send-reply"><a href="#5-4-binder-send-reply" class="headerlink" title="5.4 binder_send_reply"></a>5.4 binder_send_reply</h3><p>[-&gt;servicemanager/binder.c]</p>

<pre><code>
void binder_send_reply(struct binder_state *bs,
                       struct binder_io *reply,
                       binder_uintptr_t buffer_to_free,
                       int status)
{
    struct {
        uint32_t cmd_free;
        binder_uintptr_t buffer;
        uint32_t cmd_reply;
        struct binder_transaction_data txn;
    } __attribute__((packed)) data;

    data.cmd_free = BC_FREE_BUFFER;
    data.buffer = buffer_to_free;
    data.cmd_reply = BC_REPLY;
    data.txn.target.ptr = 0;
    data.txn.cookie = 0;
    data.txn.code = 0;
    if (status) {
        data.txn.flags = TF_STATUS_CODE;
        data.txn.data_size = sizeof(int);
        data.txn.offsets_size = 0;
        data.txn.data.ptr.buffer = (uintptr_t)&status;
        data.txn.data.ptr.offsets = 0;
    } else {
        data.txn.flags = 0;
        data.txn.data_size = reply-&gt;data - reply-&gt;data0;
        data.txn.offsets_size = ((char*) reply-&gt;offs) - ((char*) reply-&gt;offs0);
        data.txn.data.ptr.buffer = (uintptr_t)reply-&gt;data0;
        data.txn.data.ptr.offsets = (uintptr_t)reply-&gt;offs0;
    }
    //向Binder驱动写数据，见5.5
    binder_write(bs, &data, sizeof(data));
}</code></pre>

<p>将BC_FREE_BUFFER和BC_REPLY发送给binder驱动，向client发送reply。</p>
<h3 id="5-5-binder-write"><a href="#5-5-binder-write" class="headerlink" title="5.5 binder_write"></a>5.5 binder_write</h3><p>[-&gt;servicemanager/binder.c]</p>

<pre><code>
int binder_write(struct binder_state *bs, void *data, size_t len)
{
    struct binder_write_read bwr;
    int res;

    bwr.write_size = len;
    bwr.write_consumed = 0;
    bwr.write_buffer = (uintptr_t) data;
    bwr.read_size = 0;
    bwr.read_consumed = 0;
    bwr.read_buffer = 0;
    res = ioctl(bs-&gt;fd, BINDER_WRITE_READ, &bwr);
    if (res &lt; 0) {
        fprintf(stderr,"binder_write: ioctl failed (%s)/n",
                strerror(errno));
    }
    return res;
}</code></pre>

<p>这里ioctl操作又回到驱动层，见第四节，流程和上面的一样，这里不再详细分析。</p>
<h2 id="六、总结"><a href="#六、总结" class="headerlink" title="六、总结"></a>六、总结</h2><h3 id="6-1-servicemanager启动流程"><a href="#6-1-servicemanager启动流程" class="headerlink" title="6.1 servicemanager启动流程"></a>6.1 servicemanager启动流程</h3><p>1.打开Binder启动，调用mmap方法分配128K的内存映射空间：binder_open</p>
<p>2.通知binder驱动，使其成为上下文管理者，整个系统只有一个：binder_become_context_manager</p>
<p>3.进入循环等待，等待Client请求：binder_loop</p>
<h3 id="6-2-addService核心过程"><a href="#6-2-addService核心过程" class="headerlink" title="6.2 addService核心过程"></a>6.2 addService核心过程</h3>
<pre><code>
public static void addService(String name, IBinder service) {
      //此处需要将Java层Parcel转换为Native层的Parcel
      Parcel data = Parcel.obtain();
      Parcel reply = Parcel.obtain();
      data.writeInterfaceToken(IServiceManager.descriptor);
      data.writeString(name);
      //见3.5节
      data.writeStrongBinder(new JavaBBinder(env, obj));
      data.writeInt(allowIsolated ? 1 : 0);
      data.writeInt(dumpPriority);
      //见3.7节
      mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);
      reply.recycle();
      data.recycle();
}
</code></pre>

<p>通过创建JavaBBinder对象，将服务添加到ServiceManager中，和binder驱动的交互通过BpBinder.transact发送ADD_SERVICE_TRANSACTION实现添加，svclist保存所有已经注册的服务（包括服务名和handle信息）</p>
<h3 id="6-3-通信流程"><a href="#6-3-通信流程" class="headerlink" title="6.3 通信流程"></a>6.3 通信流程</h3><p>1.发起端进程向Binder Driver发送binder_ioct请求后，采用不断的循环talkWithDriver，此时线程处于阻塞状态，直到收到BR_TRANSACTION_COMPLETE命令才会结束该流程；</p>
<p>2.waitForResponse收到BR_TRANSACTION_COMPLETE命令后，则直接退出循环，不会再执行executeCommand方法，除六种其他命令外会执行该方法；</p>
<p>3.由于ServiceManager进程已经启动，并且有binder_loop一直在循环查询命令，当收到BR_TRANSACTION命令后，就开始处理addService过程。</p>
<p><img src="https://img-blog.csdnimg.cn/5dd1300ea9394c76a04c63f760e542b2.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_14,color_FFFFFF,t_70,g_se,x_16" alt="binder_addservice"></p>
<h2 id="附录"><a href="#附录" class="headerlink" title="附录"></a>附录</h2><p>源码路径</p>

<pre><code>
frameworks/base/core/java/android/os/ServiceManager.java
frameworks/base/core/java/android/os/ServiceManagerNative.java
frameworks/base/core/java/android/os/Parcel.java
frameworks/base/core/java/android/os/BinderProxy.java

frameworks/base/core/jni/android_os_Parcel.cpp
frameworks/base/core/jni/android_util_Binder.cpp
native/libs/binder/Parcel.cpp
native/libs/binder/ProcessState.cpp
native/libs/binder/BpBinder.cpp
native/libs/binder/IPCThreadState.cpp
native/cmds/servicemanager/binder.c
native/cmds/servicemanager/service_manager.c
native/cmds/servicemanager/servicemanager.rc

kernel/include/linux/syscalls.h
kernel/drivers/android/binder.c</code></pre>

      