---
layout:     post
title:      Android10 Binder机制6-Framework层分析
subtitle:   binder在framework层，采用JNI技术来调用native(C/C++)层的binder架构，从而为上层应用程序提供服务
date:       2021-03-17
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

<h2 id="一概述">一、概述</h2>

<h3 id="11-binder架构">1.1 Binder架构</h3>

<p>binder在framework层，采用JNI技术来调用native(C/C++)层的binder架构，从而为上层应用程序提供服务。 看过binder系列之前的文章，我们知道native层中，binder是C/S架构，分为Bn端(Server)和Bp端(Client)。对于java层在命名与架构上非常相近，同样实现了一套IPC通信架构。</p>

<p>framework Binder架构图</p>

<p><img src="https://img-blog.csdnimg.cn/5ea1454fb4a543ad9fb2eb38aa82ae22.jpg?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt="java_binder" /></p>

<p><strong>图解：</strong></p>

<ul>
  <li>图中红色代表整个framework层 binder架构相关组件；
    <ul>
      <li>Binder类代表Server端，BinderProxy类代码Client端；</li>
    </ul>
  </li>
  <li>图中蓝色代表Native层Binder架构相关组件；</li>
  <li>上层framework层的Binder逻辑是建立在Native层架构基础之上的，核心逻辑都是交予Native层方法来处理。</li>
  <li>framework层的ServiceManager类与Native层的功能并不完全对应，framework层的ServiceManager类的实现最终是通过BinderProxy传递给Native层来完成的，后面会详细说明。</li>
</ul>

<h3 id="12-binder类图">1.2 Binder类图</h3>

<p>下面列举framework的binder类关系图</p>

<p><img src="https://img-blog.csdnimg.cn/919c51e7a6224a46ae7fd9e9ae296174.jpg?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt="class_java_binder" /></p>

<p>图解：(图中浅蓝色都是Interface，其余都是Class)</p>

<ol>
  <li><strong>ServiceManager：</strong>通过getIServiceManager方法获取的是ServiceManagerProxy对象； ServiceManager的addService, getService实际工作都交由ServiceManagerProxy的相应方法来处理；</li>
  <li><strong>ServiceManagerProxy：</strong>其成员变量mRemote指向BinderProxy对象，ServiceManagerProxy的addService, getService方法最终是交由mRemote来完成。</li>
  <li><strong>ServiceManagerNative</strong>：其方法asInterface()返回的是ServiceManagerProxy对象，ServiceManager便是借助ServiceManagerNative类来找到ServiceManagerProxy；</li>
  <li><strong>Binder：</strong>其成员变量mObject和方法execTransact()用于native方法</li>
  <li><strong>BinderInternal：</strong>内部有一个GcWatcher类，用于处理和调试与Binder相关的垃圾回收。</li>
  <li><strong>IBinder：</strong>接口中常量FLAG_ONEWAY：客户端利用binder跟服务端通信是阻塞式的，但如果设置了FLAG_ONEWAY，这成为非阻塞的调用方式，客户端能立即返回，服务端采用回调方式来通知客户端完成情况。另外IBinder接口有一个内部接口DeathDecipient(死亡通告)。</li>
</ol>

<h3 id="13-binder类分层">1.3 Binder类分层</h3>

<p>整个Binder从kernel至，native，JNI，Framework层所涉及的全部类</p>

<p><img src="https://img-blog.csdnimg.cn/899d87caef484e5c8ba433dacd9717e3.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt="java_binder_framework" /></p>

<h2 id="二初始化">二、初始化</h2>

<p>在Android系统开机过程中，Zygote启动时会有一个<a href="http://gityuan.com/2016/02/13/android-zygote/#jnistartreg">虚拟机注册过程</a>，该过程调用AndroidRuntime::<code class="language-plaintext highlighter-rouge">startReg</code>方法来完成jni方法的注册。</p>

<h3 id="21-startreg">2.1 startReg</h3>

<p> AndroidRuntime.cpp</p>

<pre><code>int AndroidRuntime::startReg(JNIEnv* env)
{
    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);

    env-&gt;PushLocalFrame(200);

    //注册jni方法【见2.2】
    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) &lt; 0) {
        env-&gt;PopLocalFrame(NULL);
        return -1;
    }
    env-&gt;PopLocalFrame(NULL);

    return 0;
}
</code></pre>

<p>注册JNI方法，其中<code class="language-plaintext highlighter-rouge">gRegJNI</code>是一个数组，记录所有需要注册的jni方法，其中有一项便是REG_JNI(register_android_os_Binder)，下面说说<code class="language-plaintext highlighter-rouge">register_android_os_Binder</code>过程。</p>

<h3 id="22-register_android_os_binder">2.2 register_android_os_Binder</h3>

<p> android_util_Binder.cpp</p>

<pre><code>int register_android_os_Binder(JNIEnv* env)
{
    // 注册Binder类的jni方法【见2.3】
    if (int_register_android_os_Binder(env) &lt; 0)
        return -1;

    // 注册BinderInternal类的jni方法【见2.4】
    if (int_register_android_os_BinderInternal(env) &lt; 0)
        return -1;

    // 注册BinderProxy类的jni方法【见2.5】
    if (int_register_android_os_BinderProxy(env) &lt; 0)
        return -1;
    ...
    return 0;
}
</code></pre>

<h3 id="23-注册binder">2.3 注册Binder</h3>

<p> android_util_Binder.cpp</p>

<pre><code>static int int_register_android_os_Binder(JNIEnv* env)
{
    //其中kBinderPathName = "android/os/Binder";查找kBinderPathName路径所属类
    jclass clazz = FindClassOrDie(env, kBinderPathName);

    //将Java层Binder类保存到mClass变量；
    gBinderOffsets.mClass = MakeGlobalRefOrDie(env, clazz);
    //将Java层execTransact()方法保存到mExecTransact变量；
    gBinderOffsets.mExecTransact = GetMethodIDOrDie(env, clazz, "execTransact", "(IJJI)Z");
    //将Java层mObject属性保存到mObject变量
    gBinderOffsets.mObject = GetFieldIDOrDie(env, clazz, "mObject", "J");

    //注册JNI方法
    return RegisterMethodsOrDie(env, kBinderPathName, gBinderMethods,
        NELEM(gBinderMethods));
}
</code></pre>

<p>注册    Binder类的jni方法，其中：</p>

<ul>
  <li>FindClassOrDie(env, kBinderPathName) 基本等价于 env-&gt;FindClass(kBinderPathName)</li>
  <li>MakeGlobalRefOrDie() 等价于 env-&gt;NewGlobalRef()</li>
  <li>GetMethodIDOrDie() 等价于 env-&gt;GetMethodID()</li>
  <li>GetFieldIDOrDie() 等价于 env-&gt;GeFieldID()</li>
  <li>RegisterMethodsOrDie() 等价于 Android::registerNativeMethods();</li>
</ul>

<p><strong>(1)<code class="language-plaintext highlighter-rouge">gBinderOffsets</code></strong></p>

<p><code class="language-plaintext highlighter-rouge">gBinderOffsets</code>是全局静态结构体(struct)，定义如下：</p>

<pre><code>static struct bindernative_offsets_t
{
    jclass mClass; //记录Binder类
    jmethodID mExecTransact; //记录execTransact()方法
    jfieldID mObject; //记录mObject属性

} gBinderOffsets;
</code></pre>

<p><code class="language-plaintext highlighter-rouge">gBinderOffsets</code>保存了<code class="language-plaintext highlighter-rouge">Binder.java</code>类本身以及其成员方法<code class="language-plaintext highlighter-rouge">execTransact()</code>和成员属性<code class="language-plaintext highlighter-rouge">mObject</code>，这为JNI层访问Java层提供通道。另外通过查询获取Java层 binder信息后保存到<code class="language-plaintext highlighter-rouge">gBinderOffsets</code>，而不再需要每次查找binder类信息的方式能大幅度提高效率，是由于每次查询需要花费较多的CPU时间，尤其是频繁访问时，但用额外的结构体来保存这些信息，是以空间换时间的方法。</p>

<p><strong>(2)gBinderMethods</strong></p>

<pre><code>static const JNINativeMethod gBinderMethods[] = {
     /* 名称, 签名, 函数指针 */
    { "getCallingPid", "()I", (void*)android_os_Binder_getCallingPid },
    { "getCallingUid", "()I", (void*)android_os_Binder_getCallingUid },
    { "clearCallingIdentity", "()J", (void*)android_os_Binder_clearCallingIdentity },
    { "restoreCallingIdentity", "(J)V", (void*)android_os_Binder_restoreCallingIdentity },
    { "setThreadStrictModePolicy", "(I)V", (void*)android_os_Binder_setThreadStrictModePolicy },
    { "getThreadStrictModePolicy", "()I", (void*)android_os_Binder_getThreadStrictModePolicy },
    { "flushPendingCommands", "()V", (void*)android_os_Binder_flushPendingCommands },
    { "init", "()V", (void*)android_os_Binder_init },
    { "destroy", "()V", (void*)android_os_Binder_destroy },
    { "blockUntilThreadAvailable", "()V", (void*)android_os_Binder_blockUntilThreadAvailable }
};
</code></pre>

<p>通过RegisterMethodsOrDie()，将为gBinderMethods数组中的方法建立了一一映射关系，从而为Java层访问JNI层提供通道。</p>

<p>总之，<code class="language-plaintext highlighter-rouge">int_register_android_os_Binder</code>方法的主要功能：</p>

<ul>
  <li>通过gBinderOffsets，保存Java层Binder类的信息，为JNI层访问Java层提供通道；</li>
  <li>通过RegisterMethodsOrDie，将gBinderMethods数组完成映射关系，从而为Java层访问JNI层提供通道。</li>
</ul>

<p>也就是说该过程建立了Binder类在Native层与framework层之间的相互调用的桥梁。</p>

<h3 id="24-注册binderinternal">2.4 注册BinderInternal</h3>

<p> android_util_Binder.cpp</p>

<pre><code>static int int_register_android_os_BinderInternal(JNIEnv* env)
{
    //其中kBinderInternalPathName = "com/android/internal/os/BinderInternal"
    jclass clazz = FindClassOrDie(env, kBinderInternalPathName);

    gBinderInternalOffsets.mClass = MakeGlobalRefOrDie(env, clazz);
    gBinderInternalOffsets.mForceGc = GetStaticMethodIDOrDie(env, clazz, "forceBinderGc", "()V");

    return RegisterMethodsOrDie(
        env, kBinderInternalPathName,
        gBinderInternalMethods, NELEM(gBinderInternalMethods));
}
</code></pre>

<p>注册BinderInternal类的jni方法，<code class="language-plaintext highlighter-rouge">gBinderInternalOffsets</code>保存了BinderInternal的<code class="language-plaintext highlighter-rouge">forceBinderGc()</code>方法。</p>

<p>下面是BinderInternal类的JNI方法注册：</p>

<pre><code>static const JNINativeMethod gBinderInternalMethods[] = {
    { "getContextObject", "()Landroid/os/IBinder;", (void*)android_os_BinderInternal_getContextObject },
    { "joinThreadPool", "()V", (void*)android_os_BinderInternal_joinThreadPool },
    { "disableBackgroundScheduling", "(Z)V", (void*)android_os_BinderInternal_disableBackgroundScheduling },
    { "handleGc", "()V", (void*)android_os_BinderInternal_handleGc }
};
</code></pre>

<p>该过程其【2.3】非常类似，也就是说该过程建立了是BinderInternal类在Native层与framework层之间的相互调用的桥梁。</p>

<h3 id="25-注册binderproxy">2.5 注册BinderProxy</h3>

<p> android_util_Binder.cpp</p>

<pre><code>static int int_register_android_os_BinderProxy(JNIEnv* env)
{
    //gErrorOffsets保存了Error类信息
    jclass clazz = FindClassOrDie(env, "java/lang/Error");
    gErrorOffsets.mClass = MakeGlobalRefOrDie(env, clazz);

    //gBinderProxyOffsets保存了BinderProxy类的信息
    //其中kBinderProxyPathName = "android/os/BinderProxy"
    clazz = FindClassOrDie(env, kBinderProxyPathName);
    gBinderProxyOffsets.mClass = MakeGlobalRefOrDie(env, clazz);
    gBinderProxyOffsets.mConstructor = GetMethodIDOrDie(env, clazz, "&lt;init&gt;", "()V");
    gBinderProxyOffsets.mSendDeathNotice = GetStaticMethodIDOrDie(env, clazz, "sendDeathNotice", "(Landroid/os/IBinder$DeathRecipient;)V");
    gBinderProxyOffsets.mObject = GetFieldIDOrDie(env, clazz, "mObject", "J");
    gBinderProxyOffsets.mSelf = GetFieldIDOrDie(env, clazz, "mSelf", "Ljava/lang/ref/WeakReference;");
    gBinderProxyOffsets.mOrgue = GetFieldIDOrDie(env, clazz, "mOrgue", "J");

    //gClassOffsets保存了Class.getName()方法
    clazz = FindClassOrDie(env, "java/lang/Class");
    gClassOffsets.mGetName = GetMethodIDOrDie(env, clazz, "getName", "()Ljava/lang/String;");

    return RegisterMethodsOrDie(
        env, kBinderProxyPathName,
        gBinderProxyMethods, NELEM(gBinderProxyMethods));
}
</code></pre>

<p>注册BinderProxy类的jni方法，<code class="language-plaintext highlighter-rouge">gBinderProxyOffsets</code>保存了BinderProxy的<init>构造方法，sendDeathNotice(), mObject, mSelf, mOrgue信息。</init></p>

<p>下面BinderProxy类的JNI方法注册：</p>

<pre><code>static const JNINativeMethod gBinderProxyMethods[] = {
     /* 名称, 签名, 函数指针 */
    {"pingBinder",          "()Z", (void*)android_os_BinderProxy_pingBinder},
    {"isBinderAlive",       "()Z", (void*)android_os_BinderProxy_isBinderAlive},
    {"getInterfaceDescriptor", "()Ljava/lang/String;", (void*)android_os_BinderProxy_getInterfaceDescriptor},
    {"transactNative",      "(ILandroid/os/Parcel;Landroid/os/Parcel;I)Z", (void*)android_os_BinderProxy_transact},
    {"linkToDeath",         "(Landroid/os/IBinder$DeathRecipient;I)V", (void*)android_os_BinderProxy_linkToDeath},
    {"unlinkToDeath",       "(Landroid/os/IBinder$DeathRecipient;I)Z", (void*)android_os_BinderProxy_unlinkToDeath},
    {"destroy",             "()V", (void*)android_os_BinderProxy_destroy},
};
</code></pre>

<p>该过程其【2.3】非常类似，也就是说该过程建立了是BinderProxy类在Native层与framework层之间的相互调用的桥梁。</p>

<h2 id="三注册服务">三、注册服务</h2>

<h3 id="31-smaddservice">3.1 SM.addService</h3>
<p>[-&gt; ServiceManager.java]</p>

<pre><code>public static void addService(String name, IBinder service, boolean allowIsolated) {
    try {
        //先获取SMP对象，则执行注册服务操作【见小节3.2/3.4】
        getIServiceManager().addService(name, service, allowIsolated);
    } catch (RemoteException e) {
        Log.e(TAG, "error in addService", e);
    }
}
</code></pre>

<p>先来看看getIServiceManager()过程，如下：</p>

<h3 id="32-getiservicemanager">3.2 getIServiceManager</h3>
<p>[-&gt; ServiceManager.java]</p>

<pre><code>private static IServiceManager getIServiceManager() {
    if (sServiceManager != null) {
        return sServiceManager;
    }
    //【分别见3.2.1和3.3】
    sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
    return sServiceManager;
}
</code></pre>

<p>采用了单例模式获取ServiceManager  getIServiceManager()返回的是ServiceManagerProxy(简称SMP)对象</p>

<h4 id="321-getcontextobject">3.2.1 getContextObject()</h4>
<p>[-&gt; android_util_binder.cpp]</p>

<pre><code>static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    sp&lt;IBinder&gt; b = ProcessState::self()-&gt;getContextObject(NULL);
    return javaObjectForIBinder(env, b);  //【见3.2.2】
}
</code></pre>

<p>BinderInternal.java中有一个native方法getContextObject()，JNI调用执行上述方法。</p>

<p>对于ProcessState::self()-&gt;getContextObject()，在<a href="http://gityuan.com/2015/11/08/binder-get-sm/">获取ServiceManager</a>的第3节已详细解决，即<code class="language-plaintext highlighter-rouge">ProcessState::self()-&gt;getContextObject()</code>等价于 <code class="language-plaintext highlighter-rouge">new BpBinder(0)</code>;</p>

<h4 id="322-javaobjectforibinder">3.2.2 javaObjectForIBinder</h4>
<p>[-&gt; android_util_binder.cpp]</p>

<pre><code>jobject javaObjectForIBinder(JNIEnv* env, const sp&lt;IBinder&gt;&amp; val)
{
    if (val == NULL) return NULL;

    if (val-&gt;checkSubclass(&amp;gBinderOffsets)) { //返回false
        jobject object = static_cast&lt;JavaBBinder*&gt;(val.get())-&gt;object();
        return object;
    }

    AutoMutex _l(mProxyLock);

    jobject object = (jobject)val-&gt;findObject(&amp;gBinderProxyOffsets);
    if (object != NULL) { //第一次object为null
        jobject res = jniGetReferent(env, object);
        if (res != NULL) {
            return res;
        }
        android_atomic_dec(&amp;gNumProxyRefs);
        val-&gt;detachObject(&amp;gBinderProxyOffsets);
        env-&gt;DeleteGlobalRef(object);
    }

    //创建BinderProxy对象
    object = env-&gt;NewObject(gBinderProxyOffsets.mClass, gBinderProxyOffsets.mConstructor);
    if (object != NULL) {
        //BinderProxy.mObject成员变量记录BpBinder对象
        env-&gt;SetLongField(object, gBinderProxyOffsets.mObject, (jlong)val.get());
        val-&gt;incStrong((void*)javaObjectForIBinder);

        jobject refObject = env-&gt;NewGlobalRef(
                env-&gt;GetObjectField(object, gBinderProxyOffsets.mSelf));
        //将BinderProxy对象信息附加到BpBinder的成员变量mObjects中
        val-&gt;attachObject(&amp;gBinderProxyOffsets, refObject,
                jnienv_to_javavm(env), proxy_cleanup);

        sp&lt;DeathRecipientList&gt; drl = new DeathRecipientList;
        drl-&gt;incStrong((void*)javaObjectForIBinder);
        //BinderProxy.mOrgue成员变量记录死亡通知对象
        env-&gt;SetLongField(object, gBinderProxyOffsets.mOrgue, reinterpret_cast&lt;jlong&gt;(drl.get()));

        android_atomic_inc(&amp;gNumProxyRefs);
        incRefsCreated(env);
    }
    return object;
}
</code></pre>

<p>根据BpBinder(C++)生成BinderProxy(Java)对象. 主要工作是创建BinderProxy对象,并把BpBinder对象地址保存到BinderProxy.mObject成员变量.
到此，可知ServiceManagerNative.asInterface(BinderInternal.getContextObject()) 等价于</p>

<pre><code>ServiceManagerNative.asInterface(new BinderProxy())
</code></pre>

<h3 id="33--smnasinterface">3.3  SMN.asInterface</h3>
<p>[-&gt; ServiceManagerNative.java]</p>

<pre><code class="language-Java"> static public IServiceManager asInterface(IBinder obj)
{
    if (obj == null) { //obj为BpBinder
        return null;
    }
    //由于obj为BpBinder，该方法默认返回null
    IServiceManager in = (IServiceManager)obj.queryLocalInterface(descriptor);
    if (in != null) {
        return in;
    }
    return new ServiceManagerProxy(obj); //【见小节3.3.1】
}
</code></pre>

<p>由此，可知ServiceManagerNative.asInterface(new BinderProxy()) 等价于<code class="language-plaintext highlighter-rouge">new ServiceManagerProxy(new BinderProxy())</code>. 为了方便，ServiceManagerProxy简称为SMP。</p>

<h4 id="331-servicemanagerproxy初始化">3.3.1 ServiceManagerProxy初始化</h4>
<p>[-&gt; ServiceManagerNative.java  ::ServiceManagerProxy]</p>

<pre><code>class ServiceManagerProxy implements IServiceManager {
    public ServiceManagerProxy(IBinder remote) {
        mRemote = remote;
    }
}
</code></pre>

<p>mRemote为BinderProxy对象，该BinderProxy对象对应于BpBinder(0)，其作为binder代理端，指向native层大管家service Manager。</p>

<p><code class="language-plaintext highlighter-rouge">ServiceManager.getIServiceManager</code>最终等价于<code class="language-plaintext highlighter-rouge">new ServiceManagerProxy(new BinderProxy())</code>,意味着【3.1】中的getIServiceManager().addService()，等价于SMP.addService().</p>

<p>framework层的ServiceManager的调用实际的工作确实交给SMP的成员变量BinderProxy；而BinderProxy通过jni方式，最终会调用BpBinder对象；可见上层binder架构的核心功能依赖native架构的服务来完成的。</p>

<h3 id="34-smpaddservice">3.4 SMP.addService</h3>
<p>[-&gt; ServiceManagerNative.java  ::ServiceManagerProxy]</p>

<pre><code>public void addService(String name, IBinder service, boolean allowIsolated)
        throws RemoteException {
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IServiceManager.descriptor);
    data.writeString(name);
    //【见小节3.5】
    data.writeStrongBinder(service);
    data.writeInt(allowIsolated ? 1 : 0);
    //mRemote为BinderProxy【见小节3.7】
    mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);
    reply.recycle();
    data.recycle();
}
</code></pre>

<h3 id="35-writestrongbinderjava">3.5 writeStrongBinder(Java)</h3>
<p>[-&gt; Parcel.java]</p>

<pre><code>public writeStrongBinder(IBinder val){
    //此处为Native调用【见3.5.1】
    nativewriteStrongBinder(mNativePtr, val);
}
</code></pre>

<h4 id="351-android_os_parcel_writestrongbinder">3.5.1 android_os_Parcel_writeStrongBinder</h4>
<p>[-&gt; android_os_Parcel.cpp]</p>

<pre><code>static void android_os_Parcel_writeStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr, jobject object)
{
    //将java层Parcel转换为native层Parcel
    Parcel* parcel = reinterpret_cast&lt;Parcel*&gt;(nativePtr);
    if (parcel != NULL) {
        //【见3.5.2】
        const status_t err = parcel-&gt;writeStrongBinder(ibinderForJavaObject(env, object));
        if (err != NO_ERROR) {
            signalExceptionForError(env, clazz, err);
        }
    }
}
</code></pre>

<h4 id="352-ibinderforjavaobject">3.5.2 ibinderForJavaObject</h4>
<p>[-&gt; android_util_Binder.cpp]</p>

<pre><code>sp&lt;IBinder&gt; ibinderForJavaObject(JNIEnv* env, jobject obj)
{
    if (obj == NULL) return NULL;

    //Java层的Binder对象
    if (env-&gt;IsInstanceOf(obj, gBinderOffsets.mClass)) {
        JavaBBinderHolder* jbh = (JavaBBinderHolder*)
            env-&gt;GetLongField(obj, gBinderOffsets.mObject);
        return jbh != NULL ? jbh-&gt;get(env, obj) : NULL; //【见3.5.3】
    }
    //Java层的BinderProxy对象
    if (env-&gt;IsInstanceOf(obj, gBinderProxyOffsets.mClass)) {
        return (IBinder*)env-&gt;GetLongField(obj, gBinderProxyOffsets.mObject);
    }
    return NULL;
}
</code></pre>

<p>根据Binde(Java)生成JavaBBinderHolder(C++)对象. 主要工作是创建JavaBBinderHolder对象,并把JavaBBinderHolder对象地址保存到Binder.mObject成员变量.</p>

<h4 id="353-javabbinderholderget">3.5.3 JavaBBinderHolder.get()</h4>
<p>[-&gt; android_util_Binder.cpp]</p>

<pre><code class="language-Java">sp&lt;JavaBBinder&gt; get(JNIEnv* env, jobject obj)
{
    AutoMutex _l(mLock);
    sp&lt;JavaBBinder&gt; b = mBinder.promote();
    if (b == NULL) {
        //首次进来，创建JavaBBinder对象【见3.5.4】
        b = new JavaBBinder(env, obj);
        mBinder = b;
    }
    return b;
}
</code></pre>

<p>JavaBBinderHolder有一个成员变量mBinder，保存当前创建的JavaBBinder对象，这是一个wp类型的，可能会被垃圾回收器给回收，所以每次使用前，都需要先判断是否存在。</p>

<h4 id="354-javabbinder初始化">3.5.4 JavaBBinder初始化</h4>
<p> [-&gt; android_util_Binder.cpp]</p>

<pre><code>JavaBBinder(JNIEnv* env, jobject object)
    : mVM(jnienv_to_javavm(env)), mObject(env-&gt;NewGlobalRef(object))
{
    android_atomic_inc(&amp;gNumLocalRefs);
    incRefsCreated(env);
}
</code></pre>

<p>创建JavaBBinder，该对象继承于BBinder对象。</p>

<p>data.writeStrongBinder(service)最终等价于<code class="language-plaintext highlighter-rouge">parcel-&gt;writeStrongBinder(new JavaBBinder(env, obj))</code>;</p>

<h3 id="36-writestrongbinderc">3.6 writeStrongBinder(C++)</h3>
<p>[-&gt; parcel.cpp]</p>

<pre><code>status_t Parcel::writeStrongBinder(const sp&lt;IBinder&gt;&amp; val)
{
    return flatten_binder(ProcessState::self(), val, this);
}
</code></pre>

<h4 id="361-flatten_binder">3.6.1 flatten_binder</h4>
<p>[-&gt; parcel.cpp]</p>

<pre><code>status_t flatten_binder(const sp&lt;ProcessState&gt;&amp; /*proc*/,
    const sp&lt;IBinder&gt;&amp; binder, Parcel* out)
{
    flat_binder_object obj;

    obj.flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
    if (binder != NULL) {
        IBinder *local = binder-&gt;localBinder();
        if (!local) {
            BpBinder *proxy = binder-&gt;remoteBinder();
            const int32_t handle = proxy ? proxy-&gt;handle() : 0;
            obj.type = BINDER_TYPE_HANDLE; //远程Binder
            obj.binder = 0;
            obj.handle = handle;
            obj.cookie = 0;
        } else {
            obj.type = BINDER_TYPE_BINDER; //本地Binder，进入该分支
            obj.binder = reinterpret_cast&lt;uintptr_t&gt;(local-&gt;getWeakRefs());
            obj.cookie = reinterpret_cast&lt;uintptr_t&gt;(local);
        }
    } else {
        obj.type = BINDER_TYPE_BINDER;  //本地Binder
        obj.binder = 0;
        obj.cookie = 0;
    }
    //【见小节3.6.2】
    return finish_flatten_binder(binder, obj, out);
}
</code></pre>

<p>将Binder对象扁平化，转换成flat_binder_object对象。</p>

<ul>
  <li>对于Binder实体，则cookie记录Binder实体的指针；</li>
  <li>对于Binder代理，则用handle记录Binder代理的句柄；</li>
</ul>

<p>关于localBinder，代码见Binder.cpp。</p>

<pre><code>BBinder* BBinder::localBinder()
{
    return this;
}

BBinder* IBinder::localBinder()
{
    return NULL;
}
</code></pre>

<h4 id="362-finish_flatten_binder">3.6.2 finish_flatten_binder</h4>

<pre><code>inline static status_t finish_flatten_binder(
    const sp&lt;IBinder&gt;&amp; , const flat_binder_object&amp; flat, Parcel* out)
{
    return out-&gt;writeObject(flat, false);
}
</code></pre>

<p>再回到小节3.4的addService过程，则接下来进入transact。</p>

<h3 id="37-binderproxytransact">3.7 BinderProxy.transact</h3>
<p>[-&gt; Binder.java ::BinderProxy]</p>

<pre><code>public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    //用于检测Parcel大小是否大于800k
    Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");
    return transactNative(code, data, reply, flags); //【见3.8】
}
</code></pre>

<p>回到ServiceManagerProxy.addService，其成员变量mRemote是BinderProxy。transactNative经过jni调用，进入下面的方法</p>

<h3 id="38-android_os_binderproxy_transact">3.8 android_os_BinderProxy_transact</h3>
<p>[-&gt; android_util_Binder.cpp]</p>

<pre><code>static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
    jint code, jobject dataObj, jobject replyObj, jint flags)
{
    ...
    //java Parcel转为native Parcel
    Parcel* data = parcelForJavaObject(env, dataObj);
    Parcel* reply = parcelForJavaObject(env, replyObj);
    ...

    //gBinderProxyOffsets.mObject中保存的是new BpBinder(0)对象
    IBinder* target = (IBinder*)
        env-&gt;GetLongField(obj, gBinderProxyOffsets.mObject);
    ...

    //此处便是BpBinder::transact(), 经过native层，进入Binder驱动程序
    status_t err = target-&gt;transact(code, *data, reply, flags);
    ...
    return JNI_FALSE;
}
</code></pre>

<p>Java层的BinderProxy.transact()最终交由Native层的BpBinder::transact()完成。Native Binder的<a href="http://gityuan.com/2015/11/14/binder-add-service/">注册服务(addService)</a>中有详细说明BpBinder执行过程。另外，该方法可抛出RemoteException。</p>

<h3 id="39-小结">3.9 小结</h3>

<p>addService的核心过程：</p>

<pre><code>public void addService(String name, IBinder service, boolean allowIsolated)
        throws RemoteException {
    ...
    Parcel data = Parcel.obtain(); //此处还需要将java层的Parcel转为Native层的Parcel
    data-&gt;writeStrongBinder(new JavaBBinder(env, obj));
    BpBinder::transact(ADD_SERVICE_TRANSACTION, *data, reply, 0); //与Binder驱动交互
    ...
}
</code></pre>

<p>注册服务过程就是通过BpBinder来发送<code class="language-plaintext highlighter-rouge">ADD_SERVICE_TRANSACTION</code>命令，与实现与binder驱动进行数据交互。</p>

<h2 id="四获取服务">四、获取服务</h2>

<h3 id="41-smgetservice">4.1 SM.getService</h3>
<p>[-&gt; ServiceManager.java]</p>

<pre><code class="language-Java">public static IBinder getService(String name) {
    try {
        IBinder service = sCache.get(name); //先从缓存中查看
        if (service != null) {
            return service;
        } else {
            return getIServiceManager().getService(name); 【见4.2】
        }
    } catch (RemoteException e) {
        Log.e(TAG, "error in getService", e);
    }
    return null;
}
</code></pre>

<p>关于getIServiceManager()，在前面<a href="http://gityuan.com/2015/11/21/binder-framework/#getiservicemanager">小节3.2</a>已经讲述了，等价于new ServiceManagerProxy(new BinderProxy())。
其中sCache = new HashMap&lt;String, IBinder&gt;()以hashmap格式缓存已组成的名称。请求获取服务过程中，先从缓存中查询是否存在，如果缓存中不存在的话，再通过binder交互来查询相应的服务。</p>

<h3 id="42-smpgetservice">4.2 SMP.getService</h3>
<p>[-&gt; ServiceManagerNative.java ::ServiceManagerProxy]</p>

<pre><code>class ServiceManagerProxy implements IServiceManager {
    public IBinder getService(String name) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IServiceManager.descriptor);
        data.writeString(name);
        //mRemote为BinderProxy 【见4.3】
        mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0);
        //从reply里面解析出获取的IBinder对象【见4.8】
        IBinder binder = reply.readStrongBinder();
        reply.recycle();
        data.recycle();
        return binder;
    }
}
</code></pre>

<h3 id="43--binderproxytransact">4.3  BinderProxy.transact</h3>
<p>[-&gt; Binder.java]</p>

<pre><code>final class BinderProxy implements IBinder {
    public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");
        return transactNative(code, data, reply, flags);
    }
}
</code></pre>

<h3 id="44-android_os_binderproxy_transact">4.4 android_os_BinderProxy_transact</h3>
<p>[-&gt; android_util_Binder.cpp]</p>

<pre><code>static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
    jint code, jobject dataObj, jobject replyObj, jint flags)
{
    ...
    //java Parcel转为native Parcel
    Parcel* data = parcelForJavaObject(env, dataObj);
    Parcel* reply = parcelForJavaObject(env, replyObj);
    ...

    //gBinderProxyOffsets.mObject中保存的是new BpBinder(0)对象
    IBinder* target = (IBinder*)
        env-&gt;GetLongField(obj, gBinderProxyOffsets.mObject);
    ...

    //此处便是BpBinder::transact(), 经过native层[见小节4.5]
    status_t err = target-&gt;transact(code, *data, reply, flags);
    ...
    return JNI_FALSE;
}
</code></pre>

<h3 id="45--bpbindertransact">4.5  BpBinder.transact</h3>
<p>[-&gt; BpBinder.cpp]</p>

<pre><code>status_t BpBinder::transact(
    uint32_t code, const Parcel&amp; data, Parcel* reply, uint32_t flags)
{
    if (mAlive) {
        // [见小节4.6]
        status_t status = IPCThreadState::self()-&gt;transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }

    return DEAD_OBJECT;
}
</code></pre>

<h3 id="46-ipctransact">4.6 IPC.transact</h3>
<p>[-&gt; IPCThreadState.cpp]</p>

<pre><code>status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel&amp; data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err = data.errorCheck(); //数据错误检查
    flags |= TF_ACCEPT_FDS;
    ....
    if (err == NO_ERROR) {
         // 传输数据
        err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    }
    ...

    // 默认情况下,都是采用非oneway的方式, 也就是需要等待服务端的返回结果
    if ((flags &amp; TF_ONE_WAY) == 0) {
        if (reply) {
            //等待回应事件
            err = waitForResponse(reply);
        }else {
            Parcel fakeReply;
            err = waitForResponse(&amp;fakeReply);
        }
    } else {
        err = waitForResponse(NULL, NULL);
    }
    return err;
}
</code></pre>

<h3 id="47-ipcwaitforresponse">4.7 IPC.waitForResponse</h3>

<pre><code>status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    int32_t cmd;
    int32_t err;
    while (1) {
        if ((err=talkWithDriver()) &lt; NO_ERROR) break;
        ...
        cmd = mIn.readInt32();
        switch (cmd) {
          case BR_REPLY:
          {
            binder_transaction_data tr;
            err = mIn.read(&amp;tr, sizeof(tr));
            if (reply) {
                if ((tr.flags &amp; TF_STATUS_CODE) == 0) {
                    //当reply对象回收时，则会调用freeBuffer来回收内存
                    reply-&gt;ipcSetDataReference(
                        reinterpret_cast&lt;const uint8_t*&gt;(tr.data.ptr.buffer),
                        tr.data_size,
                        reinterpret_cast&lt;const binder_size_t*&gt;(tr.data.ptr.offsets),
                        tr.offsets_size/sizeof(binder_size_t),
                        freeBuffer, this);
                } else {
                    ...
                }
            }
          }
          case :...
        }
    }
    ...
    return err;
}
</code></pre>

<p>那么这个reply是哪来的呢，在文章<a href="http://gityuan.com/2015/11/07/binder-start-sm/">Binder系列3—启动ServiceManager</a></p>

<h4 id="471-binder_send_reply">4.7.1 binder_send_reply</h4>
<p>[-&gt; servicemanager/binder.c]</p>

<pre><code>void binder_send_reply(struct binder_state *bs,
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

    data.cmd_free = BC_FREE_BUFFER; //free buffer命令
    data.buffer = buffer_to_free;
    data.cmd_reply = BC_REPLY; // reply命令
    data.txn.target.ptr = 0;
    data.txn.cookie = 0;
    data.txn.code = 0;
    if (status) {
        ...
    } else {=

        data.txn.flags = 0;
        data.txn.data_size = reply-&gt;data - reply-&gt;data0;
        data.txn.offsets_size = ((char*) reply-&gt;offs) - ((char*) reply-&gt;offs0);
        data.txn.data.ptr.buffer = (uintptr_t)reply-&gt;data0;
        data.txn.data.ptr.offsets = (uintptr_t)reply-&gt;offs0;
    }
    //向Binder驱动通信
    binder_write(bs, &amp;data, sizeof(data));
}
</code></pre>

<p>binder_write将BC_FREE_BUFFER和BC_REPLY命令协议发送给驱动，进入驱动。binder_ioctl -&gt;
binder_ioctl_write_read -&gt; binder_thread_write，由于是BC_REPLY命令协议，则进入binder_transaction，
该方法会向请求服务的线程Todo队列插入事务。</p>

<p>接下来，请求服务的进程在执行talkWithDriver的过程执行到binder_thread_read()，处理Todo队列的事务。</p>

<h3 id="48-readstrongbinder">4.8 readStrongBinder</h3>
<p>[-&gt; Parcel.java]</p>

<p>readStrongBinder的过程基本是writeStrongBinder逆过程。</p>

<pre><code>static jobject android_os_Parcel_readStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr)
{
    Parcel* parcel = reinterpret_cast&lt;Parcel*&gt;(nativePtr);
    if (parcel != NULL) {
        //【见小节4.8.1】
        return javaObjectForIBinder(env, parcel-&gt;readStrongBinder());
    }
    return NULL;
}
</code></pre>

<p>javaObjectForIBinder 将native层BpBinder对象转换为Java层BinderProxy对象。</p>

<h4 id="481-readstrongbinderc">4.8.1 readStrongBinder(C++)</h4>
<p>[-&gt; Parcel.cpp]</p>

<pre><code>sp&lt;IBinder&gt; Parcel::readStrongBinder() const
{
    sp&lt;IBinder&gt; val;
    //【见小节4.8.2】
    unflatten_binder(ProcessState::self(), *this, &amp;val);
    return val;
}
</code></pre>

<h4 id="482-unflatten_binder">4.8.2 unflatten_binder</h4>
<p>[-&gt; Parcel.cpp]</p>

<pre><code class="language-Java">status_t unflatten_binder(const sp&lt;ProcessState&gt;&amp; proc,
    const Parcel&amp; in, sp&lt;IBinder&gt;* out)
{
    const flat_binder_object* flat = in.readObject(false);
    if (flat) {
        switch (flat-&gt;type) {
            case BINDER_TYPE_BINDER:
                *out = reinterpret_cast&lt;IBinder*&gt;(flat-&gt;cookie);
                return finish_unflatten_binder(NULL, *flat, in);
            case BINDER_TYPE_HANDLE:
                //进入该分支【见4.8.3】
                *out = proc-&gt;getStrongProxyForHandle(flat-&gt;handle);
                //创建BpBinder对象
                return finish_unflatten_binder(
                    static_cast&lt;BpBinder*&gt;(out-&gt;get()), *flat, in);
        }
    }
    return BAD_TYPE;
}
</code></pre>

<h4 id="483-getstrongproxyforhandle">4.8.3 getStrongProxyForHandle</h4>
<p>[-&gt; ProcessState.cpp]</p>

<pre><code>sp&lt;IBinder&gt; ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp&lt;IBinder&gt; result;

    AutoMutex _l(mLock);
    //查找handle对应的资源项
    handle_entry* e = lookupHandleLocked(handle);

    if (e != NULL) {
        IBinder* b = e-&gt;binder;
        if (b == NULL || !e-&gt;refs-&gt;attemptIncWeak(this)) {
            ...
            //当handle值所对应的IBinder不存在或弱引用无效时，则创建BpBinder对象
            b = new BpBinder(handle);
            e-&gt;binder = b;
            if (b) e-&gt;refs = b-&gt;getWeakRefs();
            result = b;
        } else {
            result.force_set(b);
            e-&gt;refs-&gt;decWeak(this);
        }
    }
    return result;
}
</code></pre>

<p>经过该方法，最终创建了指向Binder服务端的BpBinder代理对象。回到[小节4.8] 经过javaObjectForIBinder将native层BpBinder对象转换为Java层BinderProxy对象。 也就是说通过getService()最终获取了指向目标Binder服务端的代理对象BinderProxy。</p>

<h3 id="49-小结">4.9 小结</h3>

<p>getService的核心过程：</p>

<pre><code>public static IBinder getService(String name) {
    ...
    Parcel reply = Parcel.obtain(); //此处还需要将java层的Parcel转为Native层的Parcel
    BpBinder::transact(GET_SERVICE_TRANSACTION, *data, reply, 0);  //与Binder驱动交互
    IBinder binder = javaObjectForIBinder(env, new BpBinder(handle));
    ...
}
</code></pre>

<p>javaObjectForIBinder作用是创建BinderProxy对象，并将BpBinder对象的地址保存到BinderProxy对象的mObjects中。
获取服务过程就是通过BpBinder来发送<code class="language-plaintext highlighter-rouge">GET_SERVICE_TRANSACTION</code>命令，与实现与binder驱动进行数据交互。</p>

<h2 id="五-实例">五. 实例</h2>

<p>以IWindowManager为例</p>

<pre><code>public interface IWindowManager extends android.os.IInterface {

    public static abstract class Stub extends android.os.Binder implements android.view.IWindowManager {
        private static final java.lang.String DESCRIPTOR = "android.view.IWindowManager";

        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        public static android.view.IWindowManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) &amp;&amp; (iin instanceof android.view.IWindowManager))) {
                return ((android.view.IWindowManager) iin);
            }
            return new android.view.IWindowManager.Stub.Proxy(obj);
        }

        public android.os.IBinder asBinder() {
            return this;
        }

        private static class Proxy implements android.view.IWindowManager {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            public android.os.IBinder asBinder() {
                return mRemote;
            }
        }
        ...
    }
}
</code></pre>

<h3 id="51-binder">5.1 Binder</h3>
<p>[-&gt; Binder.java]</p>

<pre><code>public class Binder implements IBinder {
    public void attachInterface(IInterface owner, String descriptor) {
        mOwner = owner;
        mDescriptor = descriptor;
    }

    public IInterface queryLocalInterface(String descriptor) {
        if (mDescriptor.equals(descriptor)) {
            return mOwner;
        }
        return null;
    }
}
</code></pre>

<h3 id="52-binderproxy">5.2 BinderProxy</h3>

<pre><code>final class BinderProxy implements IBinder {
    public IInterface queryLocalInterface(String descriptor) {
        return null;
    }
}
</code></pre>
