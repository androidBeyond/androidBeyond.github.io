---
layout:     post
title:      Android10 Binder机制9-全面理解通信架构
subtitle:   本文从Java framework到Native，再到Linux Kernel，全面分析Binder通信架构.
date:       2021-04-11
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

<h2 id="一-概述">一. 概述</h2>

<h4 id="11-Binder架构分层">1.1 Binder架构分层</h4>

<p>Android内核是基于Linux系统, 而Linux现存多种进程间IPC方式:管道, 消息队列, 共享内存, 套接字, 信号量, 信号. android选择Binder进行进程间通讯的原因，前面文章有分析过，此处不再赘述 
</p>

<p>Binder采用的是分层架构设计, 每一层都有其不同的功能:</p>

<p><img src="https://img-blog.csdnimg.cn/b313a8fa781b4109bc0ce87260de44ef.jpg?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt="binder_ipc_arch" /></p>

<ul>
  <li><strong>Java应用层:</strong> 对于上层应用通过调用AMP.startService, 完全可以不用关心底层,经过层层调用,最终必然会调用到AMS.startService.</li>
  <li><strong>Java IPC层:</strong> Binder通信是采用C/S架构, Android系统的基础架构便已设计好Binder在Java framework层的Binder客户类BinderProxy和服务类Binder;</li>
  <li><strong>Native IPC层:</strong> 对于Native层,如果需要直接使用Binder(比如media相关), 则可以直接使用BpBinder和BBinder(当然这里还有JavaBBinder)即可, 对于上一层Java IPC的通信也是基于这个层面.</li>
  <li><strong>Kernel物理层:</strong> 这里是Binder Driver, 前面3层都跑在用户空间,对于用户空间的内存资源是不共享的,每个Android的进程只能运行在自己进程所拥有的虚拟地址空间, 而内核空间却是可共享的. 真正通信的核心环节还是在Binder Driver.</li>
</ul>

<h4 id="12-分析起点">1.2 分析起点</h4>

<p>前面一系列文章从源码角度讲解了Binder的各个层面, 但是Binder牵涉颇为广泛, 几乎是整个Android架构的顶梁柱, 虽说用了十几篇文章来阐述Binder的各个过程.
但依然还是没有将Binder IPC(进程间通信)的过程彻底说透.</p>

<p>Binder系统如此庞大, 那么这里需要寻求一个出发点来穿针引线, 一窥视Binder全貌. 那么本文将以startService流程分析为例子来说说Binder所起的作用.
首先在发起方进程调用AMP.startService，经过binder驱动，最终调用系统进程AMS.startService,如下图:</p>

<p><img src="https://img-blog.csdnimg.cn/571c8399df8f4418b4c71d46306e1fbc.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt="start_server_binder" /></p>

<p>AMP和AMN都是实现了IActivityManager接口,AMS继承于AMN.  其中AMP作为Binder的客户端,运行在各个app所在进程, AMN(或AMS)运行在系统进程system_server.</p>

<h4 id="13-binder-ipc原理">1.3 Binder IPC原理</h4>

<p>Binder通信采用C/S架构，从组件视角来说，包含Client、Server、ServiceManager以及binder驱动，其中ServiceManager用于管理系统中的各种服务。下面说说startService过程所涉及的Binder对象的架构图：</p>

<p><img src="https://img-blog.csdnimg.cn/fbcc99baaeb3476a918d4bee73f69574.jpg?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt="ams_ipc" /></p>

<p>可以看出无论是注册服务和获取服务的过程都需要ServiceManager，需要注意的是此处的Service Manager是指Native层的ServiceManager（C++），并非指framework层的ServiceManager(Java)。ServiceManager是整个Binder通信机制的大管家，是Android进程间通信机制Binder的守护进程，Client端和Server端通信时都需要先获取Service Manager接口，才能开始通信服务, 当然查找到目标信息可以缓存起来则不需要每次都向ServiceManager请求。</p>

<p>图中Client/Server/ServiceManage之间的相互通信都是基于Binder机制。既然基于Binder机制通信，那么同样也是C/S架构，则图中的3大步骤都有相应的Client端与Server端。</p>

<ol>
  <li><strong>注册服务</strong>：首先AMS注册到ServiceManager。该过程：AMS所在进程(system_server)是客户端，ServiceManager是服务端。</li>
  <li><strong>获取服务</strong>：Client进程使用AMS前，须先向ServiceManager中获取AMS的代理类AMP。该过程：AMP所在进程(app process)是客户端，ServiceManager是服务端。</li>
  <li><strong>使用服务</strong>： app进程根据得到的代理类AMP,便可以直接与AMS所在进程交互。该过程：AMP所在进程(app process)是客户端，AMS所在进程(system_server)是服务端。</li>
</ol>

<p>图中的Client,Server,Service Manager之间交互都是虚线表示，是由于它们彼此之间不是直接交互的，而是都通过与Binder Driver进行交互的，从而实现IPC通信方式。其中Binder驱动位于内核空间，Client,Server,Service Manager位于用户空间。Binder驱动和Service Manager可以看做是Android平台的基础架构，而Client和Server是Android的应用层.</p>

<p>这3大过程每一次都是一个完整的Binder IPC过程, 接下来从源码角度, 仅介绍<strong>第3过程使用服务</strong>, 即展开<code class="language-plaintext highlighter-rouge">AMP.startService是如何调用到AMS.startService的过程</code>.</p>

<p><strong>Tips:</strong> 如果你只想了解大致过程,并不打算细扣源码, 那么你可以略过通信过程源码分析, 仅看本文第一段落和最后段落也能对Binder所有理解.</p>

<h2 id="二-通信过程">二. 通信过程</h2>

<h3 id="21-ampstartservice">2.1 AMP.startService</h3>
<p>[-&gt; ActivityManagerNative.java  ::ActivityManagerProxy]</p>

<pre ><code>public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType, String callingPackage, int userId) throws RemoteException
{
    //获取或创建Parcel对象【见小节2.2】
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IActivityManager.descriptor);
    data.writeStrongBinder(caller != null ? caller.asBinder() : null);
    service.writeToParcel(data, 0);
    //写入Parcel数据 【见小节2.3】
    data.writeString(resolvedType);
    data.writeString(callingPackage);
    data.writeInt(userId);

    //通过Binder传递数据【见小节2.5】
    mRemote.transact(START_SERVICE_TRANSACTION, data, reply, 0);
    //读取应答消息的异常情况
    reply.readException();
    //根据reply数据来创建ComponentName对象
    ComponentName res = ComponentName.readFromParcel(reply);
    //【见小节2.2.3】
    data.recycle();
    reply.recycle();
    return res;
}
</code></pre>

<p>主要功能:</p>

<ul>
  <li>获取或创建两个Parcel对象,data用于发送数据，reply用于接收应答数据.</li>
  <li>将startService相关数据都封装到Parcel对象data, 其中descriptor = “android.app.IActivityManager”;</li>
  <li>通过Binder传递数据,并将应答消息写入reply;</li>
  <li>读取reply应答消息的异常情况和组件对象;</li>
</ul>

<h3 id="22-parcelobtain">2.2 Parcel.obtain</h3>

<p>[-&gt; Parcel.java]</p>

<pre ><code>public static Parcel obtain() {
    final Parcel[] pool = sOwnedPool;
    synchronized (pool) {
        Parcel p;
        //POOL_SIZE = 6
        for (int i=0; i&lt;POOL_SIZE; i++) {
            p = pool[i];
            if (p != null) {
                pool[i] = null;
                return p;
            }
        }
    }
    //当缓存池没有现成的Parcel对象，则直接创建[见流程2.2.1]
    return new Parcel(0);
}
</code></pre>

<p><code class="language-plaintext highlighter-rouge">sOwnedPool</code>是一个大小为6，存放着parcel对象的缓存池,这样设计的目标是用于节省每次都创建Parcel对象的开销。obtain()方法的作用：</p>

<ol>
  <li>先尝试从缓存池<code class="language-plaintext highlighter-rouge">sOwnedPool</code>中查询是否存在缓存Parcel对象，当存在则直接返回该对象;</li>
  <li>如果没有可用的Parcel对象，则直接创建Parcel对象。</li>
</ol>

<h4 id="221-new-parcel">2.2.1 new Parcel</h4>
<p>[-&gt; Parcel.java]</p>

<pre ><code>private Parcel(long nativePtr) {
    //初始化本地指针
    init(nativePtr);
}

private void init(long nativePtr) {
    if (nativePtr != 0) {
        mNativePtr = nativePtr;
        mOwnsNativeParcelObject = false;
    } else {
        // 首次创建,进入该分支[见流程2.2.2]
        mNativePtr = nativeCreate();
        mOwnsNativeParcelObject = true;
    }
}
</code></pre>

<p>nativeCreate这是native方法,经过JNI进入native层, 调用android_os_Parcel_create()方法.</p>

<h4 id="222--android_os_parcel_create">2.2.2  android_os_Parcel_create</h4>
<p>[-&gt; android_os_Parcel.cpp]</p>

<pre ><code>static jlong android_os_Parcel_create(JNIEnv* env, jclass clazz)
{
    Parcel* parcel = new Parcel();
    return reinterpret_cast&lt;jlong&gt;(parcel);
}
</code></pre>

<p>创建C++层的Parcel对象, 该对象指针强制转换为long型, 并保存到Java层的<code class="language-plaintext highlighter-rouge">mNativePtr</code>对象. 创建完Parcel对象利用Parcel对象写数据. 接下来以writeString为例.</p>

<h4 id="223-parcelrecycle">2.2.3 Parcel.recycle</h4>

<pre ><code>public final void recycle() {
    //释放native parcel对象
    freeBuffer();
    final Parcel[] pool;
    //根据情况来选择加入相应池
    if (mOwnsNativeParcelObject) {
        pool = sOwnedPool;
    } else {
        mNativePtr = 0;
        pool = sHolderPool;
    }
    synchronized (pool) {
        for (int i=0; i&lt;POOL_SIZE; i++) {
            if (pool[i] == null) {
                pool[i] = this;
                return;
            }
        }
    }
}
</code></pre>

<p>将不再使用的Parcel对象放入缓存池，可回收重复利用，当缓存池已满则不再加入缓存池。这里有两个Parcel线程池,<code class="language-plaintext highlighter-rouge">mOwnsNativeParcelObject</code>变量来决定:</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">mOwnsNativeParcelObject</code>=true,  即调用不带参数obtain()方法获取的对象, 回收时会放入<code class="language-plaintext highlighter-rouge">sOwnedPool</code>对象池;</li>
  <li><code class="language-plaintext highlighter-rouge">mOwnsNativeParcelObject</code>=false, 即调用带nativePtr参数的obtain(long)方法获取的对象, 回收时会放入<code class="language-plaintext highlighter-rouge">sHolderPool</code>对象池;</li>
</ul>

<h3 id="23-writestring">2.3 writeString</h3>
<p>[-&gt; Parcel.java]</p>

<pre ><code>public final void writeString(String val) {
    //[见流程2.3.1]
    nativeWriteString(mNativePtr, val);
}
</code></pre>

<h4 id="231-nativewritestring">2.3.1 nativeWriteString</h4>
<p>[-&gt; android_os_Parcel.cpp]</p>

<pre ><code>static void android_os_Parcel_writeString(JNIEnv* env, jclass clazz, jlong nativePtr, jstring val)
{
    Parcel* parcel = reinterpret_cast&lt;Parcel*&gt;(nativePtr);
    if (parcel != NULL) {
        status_t err = NO_MEMORY;
        if (val) {
            const jchar* str = env-&gt;GetStringCritical(val, 0);
            if (str) {
                //[见流程2.3.2]
                err = parcel-&gt;writeString16(
                    reinterpret_cast&lt;const char16_t*&gt;(str),
                    env-&gt;GetStringLength(val));
                env-&gt;ReleaseStringCritical(val, str);
            }
        } else {
            err = parcel-&gt;writeString16(NULL, 0);
        }
        if (err != NO_ERROR) {
            signalExceptionForError(env, clazz, err);
        }
    }
}
</code></pre>

<h4 id="232-writestring16">2.3.2 writeString16</h4>
<p>[-&gt; Parcel.cpp]</p>

<pre ><code>status_t Parcel::writeString16(const char16_t* str, size_t len)
{
    if (str == NULL) return writeInt32(-1);

    status_t err = writeInt32(len);
    if (err == NO_ERROR) {
        len *= sizeof(char16_t);
        uint8_t* data = (uint8_t*)writeInplace(len+sizeof(char16_t));
        if (data) {
            //数据拷贝到data所指向的位置
            memcpy(data, str, len);
            *reinterpret_cast&lt;char16_t*&gt;(data+len) = 0;
            return NO_ERROR;
        }
        err = mError;
    }
    return err;
}
</code></pre>

<p><strong>Tips:</strong> 除了writeString(),在<code class="language-plaintext highlighter-rouge">Parcel.java</code>中大量的native方法, 都是调用<code class="language-plaintext highlighter-rouge">android_os_Parcel.cpp</code>相对应的方法, 该方法再调用<code class="language-plaintext highlighter-rouge">Parcel.cpp</code>中对应的方法.  <br />
调用流程:    Parcel.java –&gt;  android_os_Parcel.cpp  –&gt; Parcel.cpp.</p>

<pre><code class="language-Java">frameworks/base/core/java/android/os/Parcel.java
frameworks/base/core/jni/android_os_Parcel.cpp
frameworks/native/libs/binder/Parcel.cpp
</code></pre>

<p>简单说,就是</p>

<h3 id="24-mremote究竟为何物">2.4 mRemote究竟为何物</h3>

<p>mRemote的出生,要出先说说ActivityManagerProxy对象(简称AMP)创建说起, AMP是通过ActivityManagerNative.getDefault()来获取的.</p>

<h4 id="241-amngetdefault">2.4.1 AMN.getDefault</h4>
<p>[-&gt; ActivityManagerNative.java]</p>

<pre><code class="language-Java">static public IActivityManager getDefault() {
    // [见流程2.4.2]
    return gDefault.get();
}
</code></pre>

<p>gDefault的数据类型为<code class="language-plaintext highlighter-rouge">Singleton&lt;IActivityManager&gt;</code>, 这是一个单例模式, 接下来看看Singleto.get()的过程</p>

<h4 id="242-gdefaultget">2.4.2 gDefault.get</h4>

<pre ><code>public abstract class Singleton&lt;IActivityManager&gt; {
    public final IActivityManager get() {
        synchronized (this) {
            if (mInstance == null) {
                //首次调用create()来获取AMP对象[见流程2.4.3]
                mInstance = create();
            }
            return mInstance;
        }
    }
}
</code></pre>

<p>首次调用时需要创建,创建完之后保持到mInstance对象,之后可直接使用.</p>

<h4 id="243-gdefaultcreate">2.4.3 gDefault.create</h4>

<pre ><code>private static final Singleton&lt;IActivityManager&gt; gDefault = new Singleton&lt;IActivityManager&gt;() {
    protected IActivityManager create() {
        //获取名为"activity"的服务
        IBinder b = ServiceManager.getService("activity");
        //创建AMP对象[见流程2.4.4]
        IActivityManager am = asInterface(b);
        return am;
    }
};
</code></pre>

<p>由之前的framework层分析可知ServiceManager.getService(“activity”)返回的是指向目标服务AMS的代理对象<code class="language-plaintext highlighter-rouge">BinderProxy</code>对象，由该代理对象可以找到目标服务AMS所在进程</p>

<h4 id="244-amnasinterface">2.4.4 AMN.asInterface</h4>
<p>[-&gt; ActivityManagerNative.java]</p>

<pre ><code>public abstract class ActivityManagerNative extends Binder implements IActivityManager
{
    static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        //此处obj = BinderProxy,  descriptor = "android.app.IActivityManager"; [见流程2.4.5]
        IActivityManager in = (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) { //此处为null
            return in;
        }
        //[见流程2.4.6]
        return new ActivityManagerProxy(obj);
    }
    ...
}
</code></pre>

<p>此时obj为BinderProxy对象, 记录着远程进程system_server中AMS服务的binder线程的handle.</p>

<h4 id="245--querylocalinterface">2.4.5  queryLocalInterface</h4>
<p>[Binder.java]</p>

<pre ><code>public class Binder implements IBinder {
    //对于Binder对象的调用,则返回值不为空
    public IInterface queryLocalInterface(String descriptor) {
        //mDescriptor的初始化在attachInterface()过程中赋值
        if (mDescriptor.equals(descriptor)) {
            return mOwner;
        }
        return null;
    }
}

//由上一小节[2.4.4]调用的流程便是此处,返回Null
final class BinderProxy implements IBinder {
    //BinderProxy对象的调用, 则返回值为空
    public IInterface queryLocalInterface(String descriptor) {
        return null;
    }
}
</code></pre>

<p>对于Binder IPC的过程中, 同一个进程的调用则会是asInterface()方法返回的便是本地的Binder对象;对于不同进程的调用则会是远程代理对象BinderProxy.</p>

<h4 id="246-创建amp">2.4.6 创建AMP</h4>
<p>[-&gt; ActivityManagerNative.java :: AMP]</p>

<pre ><code>class ActivityManagerProxy implements IActivityManager
{
    public ActivityManagerProxy(IBinder remote)
    {
        mRemote = remote;
    }
}
</code></pre>

<p>可知<code class="language-plaintext highlighter-rouge">mRemote</code>便是指向AMS服务的<code class="language-plaintext highlighter-rouge">BinderProxy</code>对象。</p>

<h3 id="25-mremotetransact">2.5 mRemote.transact</h3>
<p>[-&gt; Binder.java ::BinderProxy]</p>

<pre ><code>final class BinderProxy implements IBinder {
    public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        //用于检测Parcel大小是否大于800k
        Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");
        //【见2.6】
        return transactNative(code, data, reply, flags);
    }
}
</code></pre>

<p>mRemote.transact()方法中的code=START_SERVICE_TRANSACTION, data保存了<code class="language-plaintext highlighter-rouge">descriptor</code>，<code class="language-plaintext highlighter-rouge">caller</code>, <code class="language-plaintext highlighter-rouge">intent</code>, <code class="language-plaintext highlighter-rouge">resolvedType</code>, <code class="language-plaintext highlighter-rouge">callingPackage</code>, <code class="language-plaintext highlighter-rouge">userId</code>这6项信息。</p>

<p>transactNative是native方法，经过jni调用android_os_BinderProxy_transact方法。</p>

<h3 id="26-android_os_binderproxy_transact">2.6 android_os_BinderProxy_transact</h3>
<p>[-&gt; android_util_Binder.cpp]</p>

<pre ><code>static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
    jint code, jobject dataObj, jobject replyObj, jint flags)
{
    ...
    //将java Parcel转为c++ Parcel
    Parcel* data = parcelForJavaObject(env, dataObj);
    Parcel* reply = parcelForJavaObject(env, replyObj);

    //gBinderProxyOffsets.mObject中保存的是new BpBinder(handle)对象
    IBinder* target = (IBinder*) env-&gt;GetLongField(obj, gBinderProxyOffsets.mObject);
    ...

    //此处便是BpBinder::transact()【见小节2.7】
    status_t err = target-&gt;transact(code, *data, reply, flags);
    ...

    //最后根据transact执行具体情况，抛出相应的Exception
    signalExceptionForError(env, obj, err, true , data-&gt;dataSize());
    return JNI_FALSE;
}
</code></pre>

<p>gBinderProxyOffsets.mObject中保存的是<code class="language-plaintext highlighter-rouge">BpBinder</code>对象, 这是开机时Zygote调用<code class="language-plaintext highlighter-rouge">AndroidRuntime::startReg</code>方法来完成jni方法的注册.</p>

<p>其中register_android_os_Binder()过程就有一个初始并注册BinderProxy的操作,完成gBinderProxyOffsets的赋值过程. 接下来就进入该方法.</p>

<h3 id="27-bpbindertransact">2.7 BpBinder.transact</h3>
<p>[-&gt; BpBinder.cpp]</p>

<pre ><code>status_t BpBinder::transact(
    uint32_t code, const Parcel&amp; data, Parcel* reply, uint32_t flags)
{
    if (mAlive) {
        // 【见小节2.8】
        status_t status = IPCThreadState::self()-&gt;transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }
    return DEAD_OBJECT;
}
</code></pre>

<p>IPCThreadState::self()采用单例模式，保证每个线程只有一个实例对象。</p>

<h3 id="28-ipctransact">2.8 IPC.transact</h3>
<p>[-&gt; IPCThreadState.cpp]</p>

<pre ><code>status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel&amp; data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err = data.errorCheck(); //数据错误检查
    flags |= TF_ACCEPT_FDS;
    ....
    if (err == NO_ERROR) {
         // 传输数据 【见小节2.9】
        err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    }

    if (err != NO_ERROR) {
        if (reply) reply-&gt;setError(err);
        return (mLastError = err);
    }

    // 默认情况下,都是采用非oneway的方式, 也就是需要等待服务端的返回结果
    if ((flags &amp; TF_ONE_WAY) == 0) {
        if (reply) {
            //reply对象不为空 【见小节2.10】
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

<p>transact主要过程:</p>

<ul>
  <li>先执行writeTransactionData()已向Parcel数据类型的<code class="language-plaintext highlighter-rouge">mOut</code>写入数据，此时<code class="language-plaintext highlighter-rouge">mIn</code>还没有数据；</li>
  <li>然后执行waitForResponse()方法，循环执行，直到收到应答消息. 调用talkWithDriver()跟驱动交互，收到应答消息，便会写入<code class="language-plaintext highlighter-rouge">mIn</code>, 则根据收到的不同响应吗，执行相应的操作。</li>
</ul>

<p>此处调用waitForResponse根据是否有设置<code class="language-plaintext highlighter-rouge">TF_ONE_WAY</code>的标记:</p>

<ul>
  <li>当已设置oneway时, 则调用waitForResponse(NULL, NULL);</li>
  <li>当未设置oneway时, 则调用waitForResponse(reply) 或 waitForResponse(&amp;fakeReply)</li>
</ul>

<h3 id="29-ipcwritetransactiondata">2.9 IPC.writeTransactionData</h3>
<p>[-&gt; IPCThreadState.cpp]</p>

<pre ><code>status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel&amp; data, status_t* statusBuffer)
{
    binder_transaction_data tr;

    tr.target.ptr = 0;
    tr.target.handle = handle; // handle指向AMS
    tr.code = code;            // START_SERVICE_TRANSACTION
    tr.flags = binderFlags;    // 0
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        // data为startService相关信息
        tr.data_size = data.ipcDataSize();   // mDataSize
        tr.data.ptr.buffer = data.ipcData(); // mData指针
        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t); //mObjectsSize
        tr.data.ptr.offsets = data.ipcObjects(); //mObjects指针
    }
    ...
    mOut.writeInt32(cmd);         //cmd = BC_TRANSACTION
    mOut.write(&amp;tr, sizeof(tr));  //写入binder_transaction_data数据
    return NO_ERROR;
}
</code></pre>

<p>将数据写入mOut</p>

<h3 id="210-ipcwaitforresponse">2.10 IPC.waitForResponse</h3>

<pre ><code>status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    int32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) &lt; NO_ERROR) break; // 【见小节2.11】
        err = mIn.errorCheck();
        if (err &lt; NO_ERROR) break; //当存在error则退出循环

         //每当跟Driver交互一次，若mIn收到数据则往下执行一次BR命令
        if (mIn.dataAvail() == 0) continue;

        cmd = mIn.readInt32();

        switch (cmd) {
        case BR_TRANSACTION_COMPLETE:
            //只有当不需要reply, 也就是oneway时 才会跳出循环,否则还需要等待.
            if (!reply &amp;&amp; !acquireResult) goto finish; break;

        case BR_DEAD_REPLY:
            err = DEAD_OBJECT;         goto finish;
        case BR_FAILED_REPLY:
            err = FAILED_TRANSACTION;  goto finish;
        case BR_REPLY: ...             goto finish;

        default:
            err = executeCommand(cmd);  //【见小节2.12】
            if (err != NO_ERROR) goto finish;
            break;
        }
    }

finish:
    if (err != NO_ERROR) {
        if (reply) reply-&gt;setError(err); //将发送的错误代码返回给最初的调用者
    }
    return err;
}
</code></pre>

<p>在这个过程中, 收到以下任一BR_命令，处理后便会退出waitForResponse()的状态:</p>

<ul>
  <li>BR_TRANSACTION_COMPLETE: binder驱动收到BC_TRANSACTION事件后的应答消息; 对于oneway transaction,当收到该消息,则完成了本次Binder通信;</li>
  <li>BR_DEAD_REPLY: 回复失败，往往是线程或节点为空. 则结束本次通信Binder;</li>
  <li>BR_FAILED_REPLY:回复失败，往往是transaction出错导致. 则结束本次通信Binder;</li>
  <li>BR_REPLY: Binder驱动向Client端发送回应消息; 对于非oneway transaction时,当收到该消息,则完整地完成本次Binder通信;</li>
</ul>

<p>除了以上命令，其他命令的处理流程【见小节2.12】</p>

<h3 id="211--ipctalkwithdriver">2.11  IPC.talkWithDriver</h3>

<pre ><code>//mOut有数据，mIn还没有数据。doReceive默认值为true
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    binder_write_read bwr;

    const bool needRead = mIn.dataPosition() &gt;= mIn.dataSize();
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();

    if (doReceive &amp;&amp; needRead) {
        //接收数据缓冲区信息的填充。当收到驱动的数据，则写入mIn
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

    // 当同时没有输入和输出数据则直接返回
    if ((bwr.write_size == 0) &amp;&amp; (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        //ioctl执行binder读写操作，经过syscall，进入Binder驱动。调用Binder_ioctl【小节3.1】
        if (ioctl(mProcess-&gt;mDriverFD, BINDER_WRITE_READ, &amp;bwr) &gt;= 0)
            err = NO_ERROR;
        else
            err = -errno;
        ...
    } while (err == -EINTR);

    if (err &gt;= NO_ERROR) {
        if (bwr.write_consumed &gt; 0) {
            if (bwr.write_consumed &lt; mOut.dataSize())
                mOut.remove(0, bwr.write_consumed);
            else
                mOut.setDataSize(0);
        }
        if (bwr.read_consumed &gt; 0) {
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }
        return NO_ERROR;
    }
    return err;
}
</code></pre>

<p>binder_write_read结构体>用来与Binder设备交换数据的结构, 通过ioctl与mDriverFD通信，是真正与Binder驱动进行数据读写交互的过程。</p>

<h3 id="212--ipcexecutecommand">2.12  IPC.executeCommand</h3>

<pre ><code>status_t IPCThreadState::executeCommand(int32_t cmd)
{
    BBinder* obj;
    RefBase::weakref_type* refs;
    status_t result = NO_ERROR;

    switch ((uint32_t)cmd) {
    case BR_ERROR: ...
    case BR_OK: ...
    case BR_ACQUIRE: ...
    case BR_RELEASE: ...
    case BR_INCREFS: ...
    case BR_TRANSACTION: ... //Binder驱动向Server端发送消息
    case BR_DEAD_BINDER: ...
    case BR_CLEAR_DEATH_NOTIFICATION_DONE: ...
    case BR_NOOP: ...
    case BR_SPAWN_LOOPER: ... //创建新binder线程
    default: ...
    }
}
</code></pre>

<p>再回到【小节2.11】，可知ioctl()方法经过syscall最终调用到Binder_ioctl()方法.</p>

<h2 id="三binder-driver">三、Binder driver</h2>

<h4 id="31-binder_ioctl">3.1 binder_ioctl</h4>
<p>[-&gt; Binder.c]</p>

<p>由【小节2.11】传递过出来的参数 cmd=<code class="language-plaintext highlighter-rouge">BINDER_WRITE_READ</code></p>

<pre ><code>static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    int ret;
    struct binder_proc *proc = filp-&gt;private_data;
    struct binder_thread *thread;

    //当binder_stop_on_user_error&gt;=2时，则该线程加入等待队列并进入休眠状态. 该值默认为0
    ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error &lt; 2);
    ...
    binder_lock(__func__);
    //查找或创建binder_thread结构体
    thread = binder_get_thread(proc);
    ...
    switch (cmd) {
        case BINDER_WRITE_READ:
            //【见小节3.2】
            ret = binder_ioctl_write_read(filp, cmd, arg, thread);
            break;
        ...
    }
    ret = 0;

err:
    if (thread)
        thread-&gt;looper &amp;= ~BINDER_LOOPER_STATE_NEED_RETURN;
    binder_unlock(__func__);
    wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error &lt; 2);
    return ret;
}
</code></pre>

<p>首先,根据传递过来的文件句柄指针获取相应的binder_proc结构体, 再从中查找binder_thread,如果当前线程已经加入到proc的线程队列则直接返回，
如果不存在则创建binder_thread，并将当前线程添加到当前的proc.</p>

<ul>
  <li>当返回值为-ENOMEM，则意味着内存不足，往往会出现创建binder_thread对象失败;</li>
  <li>当返回值为-EINVAL，则意味着CMD命令参数无效；</li>
</ul>

<h4 id="32--binder_ioctl_write_read">3.2  binder_ioctl_write_read</h4>

<pre ><code>static int binder_ioctl_write_read(struct file *filp,
                unsigned int cmd, unsigned long arg,
                struct binder_thread *thread)
{
    int ret = 0;
    struct binder_proc *proc = filp-&gt;private_data;
    unsigned int size = _IOC_SIZE(cmd);
    void __user *ubuf = (void __user *)arg;
    struct binder_write_read bwr;
    if (size != sizeof(struct binder_write_read)) {
        ret = -EINVAL;
        goto out;
    }
    //将用户空间bwr结构体拷贝到内核空间
    if (copy_from_user(&amp;bwr, ubuf, sizeof(bwr))) {
        ret = -EFAULT;
        goto out;
    }

    if (bwr.write_size &gt; 0) {
        //将数据放入目标进程【见小节3.3】
        ret = binder_thread_write(proc, thread,
                      bwr.write_buffer,
                      bwr.write_size,
                      &amp;bwr.write_consumed);
        //当执行失败，则直接将内核bwr结构体写回用户空间，并跳出该方法
        if (ret &lt; 0) {
            bwr.read_consumed = 0;
            if (copy_to_user_preempt_disabled(ubuf, &amp;bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }
    if (bwr.read_size &gt; 0) {
        //读取自己队列的数据 【见小节3.5】
        ret = binder_thread_read(proc, thread, bwr.read_buffer,
             bwr.read_size,
             &amp;bwr.read_consumed,
             filp-&gt;f_flags &amp; O_NONBLOCK);
        //当进程的todo队列有数据,则唤醒在该队列等待的进程
        if (!list_empty(&amp;proc-&gt;todo))
            wake_up_interruptible(&amp;proc-&gt;wait);
        //当执行失败，则直接将内核bwr结构体写回用户空间，并跳出该方法
        if (ret &lt; 0) {
            if (copy_to_user_preempt_disabled(ubuf, &amp;bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }

    if (copy_to_user(ubuf, &amp;bwr, sizeof(bwr))) {
        ret = -EFAULT;
        goto out;
    }
out:
    return ret;
}   
</code></pre>

<p>此时arg是一个<code class="language-plaintext highlighter-rouge">binder_write_read</code>结构体，<code class="language-plaintext highlighter-rouge">mOut</code>数据保存在write_buffer，所以write_size&gt;0，但此时read_size=0。首先,将用户空间bwr结构体拷贝到内核空间,然后执行binder_thread_write()操作.</p>

<h4 id="33-binder_thread_write">3.3 binder_thread_write</h4>

<pre ><code>static int binder_thread_write(struct binder_proc *proc,
            struct binder_thread *thread,
            binder_uintptr_t binder_buffer, size_t size,
            binder_size_t *consumed)
{
    uint32_t cmd;
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;
    while (ptr &lt; end &amp;&amp; thread-&gt;return_error == BR_OK) {
        //拷贝用户空间的cmd命令，此时为BC_TRANSACTION
        if (get_user(cmd, (uint32_t __user *)ptr)) -EFAULT;
        ptr += sizeof(uint32_t);
        switch (cmd) {
        case BC_TRANSACTION:
        case BC_REPLY: {
            struct binder_transaction_data tr;
            //拷贝用户空间的binder_transaction_data
            if (copy_from_user(&amp;tr, ptr, sizeof(tr)))   return -EFAULT;
            ptr += sizeof(tr);
            // 见小节3.4】
            binder_transaction(proc, thread, &amp;tr, cmd == BC_REPLY);
            break;
        }
        ...
    }
    *consumed = ptr - buffer;
  }
  return 0;
}
</code></pre>

<p>不断从binder_buffer所指向的地址获取cmd, 当只有<code class="language-plaintext highlighter-rouge">BC_TRANSACTION</code>或者<code class="language-plaintext highlighter-rouge">BC_REPLY</code>时, 则调用binder_transaction()来处理事务.</p>

<h4 id="34-binder_transaction">3.4 binder_transaction</h4>

<p>发送的是BC_TRANSACTION时，此时reply=0。</p>

<pre ><code>static void binder_transaction(struct binder_proc *proc,
               struct binder_thread *thread,
               struct binder_transaction_data *tr, int reply){
     struct binder_transaction *t;
     struct binder_work *tcomplete;
     binder_size_t *offp, *off_end;
     binder_size_t off_min;
     struct binder_proc *target_proc;
     struct binder_thread *target_thread = NULL;
     struct binder_node *target_node = NULL;
     struct list_head *target_list;
     wait_queue_head_t *target_wait;
     struct binder_transaction *in_reply_to = NULL;

    if (reply) {
        ...
    }else {
        if (tr-&gt;target.handle) {
            struct binder_ref *ref;
            // 由handle 找到相应 binder_ref, 由binder_ref 找到相应 binder_node
            ref = binder_get_ref(proc, tr-&gt;target.handle);
            target_node = ref-&gt;node;
        } else {
            target_node = binder_context_mgr_node;
        }
        // 由binder_node 找到相应 binder_proc
        target_proc = target_node-&gt;proc;
    }


    if (target_thread) {
        e-&gt;to_thread = target_thread-&gt;pid;
        target_list = &amp;target_thread-&gt;todo;
        target_wait = &amp;target_thread-&gt;wait;
    } else {
        //首次执行target_thread为空
        target_list = &amp;target_proc-&gt;todo;
        target_wait = &amp;target_proc-&gt;wait;
    }

    t = kzalloc(sizeof(*t), GFP_KERNEL);
    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);

    //非oneway的通信方式，把当前thread保存到transaction的from字段
    if (!reply &amp;&amp; !(tr-&gt;flags &amp; TF_ONE_WAY))
        t-&gt;from = thread;
    else
        t-&gt;from = NULL;

    t-&gt;sender_euid = task_euid(proc-&gt;tsk);
    t-&gt;to_proc = target_proc; //此次通信目标进程为system_server
    t-&gt;to_thread = target_thread;
    t-&gt;code = tr-&gt;code;  //此次通信code = START_SERVICE_TRANSACTION
    t-&gt;flags = tr-&gt;flags;  // 此次通信flags = 0
    t-&gt;priority = task_nice(current);

    //从目标进程target_proc中分配内存空间【3.4.1】
    t-&gt;buffer = binder_alloc_buf(target_proc, tr-&gt;data_size,
        tr-&gt;offsets_size, !reply &amp;&amp; (t-&gt;flags &amp; TF_ONE_WAY));

    t-&gt;buffer-&gt;allow_user_free = 0;
    t-&gt;buffer-&gt;transaction = t;
    t-&gt;buffer-&gt;target_node = target_node;

    if (target_node)
        binder_inc_node(target_node, 1, 0, NULL); //引用计数加1
    //binder对象的偏移量
    offp = (binder_size_t *)(t-&gt;buffer-&gt;data + ALIGN(tr-&gt;data_size, sizeof(void *)));

    //分别拷贝用户空间的binder_transaction_data中ptr.buffer和ptr.offsets到目标进程的binder_buffer
    copy_from_user(t-&gt;buffer-&gt;data,
        (const void __user *)(uintptr_t)tr-&gt;data.ptr.buffer, tr-&gt;data_size);
    copy_from_user(offp,
        (const void __user *)(uintptr_t)tr-&gt;data.ptr.offsets, tr-&gt;offsets_size);

    off_end = (void *)offp + tr-&gt;offsets_size;

    for (; offp &lt; off_end; offp++) {
        struct flat_binder_object *fp;
        fp = (struct flat_binder_object *)(t-&gt;buffer-&gt;data + *offp);
        off_min = *offp + sizeof(struct flat_binder_object);
        switch (fp-&gt;type) {
        ...
        case BINDER_TYPE_HANDLE:
        case BINDER_TYPE_WEAK_HANDLE: {
            //处理引用计数情况
            struct binder_ref *ref = binder_get_ref(proc, fp-&gt;handle);
            if (ref-&gt;node-&gt;proc == target_proc) {
                if (fp-&gt;type == BINDER_TYPE_HANDLE)
                    fp-&gt;type = BINDER_TYPE_BINDER;
                else
                    fp-&gt;type = BINDER_TYPE_WEAK_BINDER;
                fp-&gt;binder = ref-&gt;node-&gt;ptr;
                fp-&gt;cookie = ref-&gt;node-&gt;cookie;
                binder_inc_node(ref-&gt;node, fp-&gt;type == BINDER_TYPE_BINDER, 0, NULL);
            } else {    
                struct binder_ref *new_ref;
                new_ref = binder_get_ref_for_node(target_proc, ref-&gt;node);
                fp-&gt;handle = new_ref-&gt;desc;
                binder_inc_ref(new_ref, fp-&gt;type == BINDER_TYPE_HANDLE, NULL);
            }
        } break;
        ...

        default:
            return_error = BR_FAILED_REPLY;
            goto err_bad_object_type;
        }
    }

    if (reply) {
        //BC_REPLY的过程
        binder_pop_transaction(target_thread, in_reply_to);
    } else if (!(t-&gt;flags &amp; TF_ONE_WAY)) {
        //BC_TRANSACTION 且 非oneway,则设置事务栈信息
        t-&gt;need_reply = 1;
        t-&gt;from_parent = thread-&gt;transaction_stack;
        thread-&gt;transaction_stack = t;
    } else {
        //BC_TRANSACTION 且 oneway,则加入异步todo队列
        if (target_node-&gt;has_async_transaction) {
            target_list = &amp;target_node-&gt;async_todo;
            target_wait = NULL;
        } else
            target_node-&gt;has_async_transaction = 1;
    }

    //将BINDER_WORK_TRANSACTION添加到目标队列,即target_proc-&gt;todo
    t-&gt;work.type = BINDER_WORK_TRANSACTION;
    list_add_tail(&amp;t-&gt;work.entry, target_list);

    //将BINDER_WORK_TRANSACTION_COMPLETE添加到当前线程队列，即thread-&gt;todo
    tcomplete-&gt;type = BINDER_WORK_TRANSACTION_COMPLETE;
    list_add_tail(&amp;tcomplete-&gt;entry, &amp;thread-&gt;todo);

    //唤醒等待队列，本次通信的目标队列为target_proc-&gt;wait
    if (target_wait)
        wake_up_interruptible(target_wait);
    return;
}
</code></pre>

<p>主要功能:</p>

<ol>
  <li>查询目标进程的过程： handle -&gt; binder_ref -&gt; binder_node -&gt; binder_proc</li>
  <li>将<code class="language-plaintext highlighter-rouge">BINDER_WORK_TRANSACTION</code>添加到目标队列target_list:
    <ul>
      <li>call事务， 则目标队列target_list=<code class="language-plaintext highlighter-rouge">target_proc-&gt;todo</code>;</li>
      <li>reply事务，则目标队列target_list=<code class="language-plaintext highlighter-rouge">target_thread-&gt;todo</code>;</li>
      <li>async事务，则目标队列target_list=<code class="language-plaintext highlighter-rouge">target_node-&gt;async_todo</code>.</li>
    </ul>
  </li>
  <li>数据拷贝
    <ul>
      <li>将用户空间binder_transaction_data中ptr.buffer和ptr.offsets拷贝到目标进程的binder_buffer-&gt;data；</li>
      <li>这就是只拷贝一次的真理所在；</li>
    </ul>
  </li>
  <li>设置事务栈信息
    <ul>
      <li>BC_TRANSACTION且非oneway, 则将当前事务添加到thread-&gt;transaction_stack；</li>
    </ul>
  </li>
  <li>事务分发过程：
    <ul>
      <li>将<code class="language-plaintext highlighter-rouge">BINDER_WORK_TRANSACTION</code>添加到目标队列(此时为target_proc-&gt;todo队列);</li>
      <li>将<code class="language-plaintext highlighter-rouge">BINDER_WORK_TRANSACTION_COMPLETE</code>添加到当前线程thread-&gt;todo队列;</li>
    </ul>
  </li>
  <li>唤醒目标进程target_proc开始执行事务。</li>
</ol>

<p>该方法中proc/thread是指当前发起方的进程信息，而binder_proc是指目标接收端进程。
此时当前线程thread的todo队列已经有事务, 接下来便会进入binder_thread_read来处理相关的事务.</p>

<h4 id="341-binder_alloc_buf">3.4.1 binder_alloc_buf</h4>

<pre ><code>static struct binder_buffer *binder_alloc_buf(struct binder_proc *proc,
                          size_t data_size, size_t offsets_size, int is_async)
{
    struct rb_node *n = proc-&gt;free_buffers.rb_node;
    struct binder_buffer *buffer;
    size_t buffer_size;
    struct rb_node *best_fit = NULL;
    void *has_page_addr;
    void *end_page_addr;
    size_t size;
    ..
    size = ALIGN(data_size, sizeof(void *)) + ALIGN(offsets_size, sizeof(void *));
    if (is_async &amp;&amp; proc-&gt;free_async_space &lt; size + sizeof(struct binder_buffer)) {
        return NULL; // 剩余可用的异步空间，小于所需的大小
    }
    while (n) {  //从binder_buffer的红黑树中查找大小相等的buffer块
        buffer = rb_entry(n, struct binder_buffer, rb_node);
        buffer_size = binder_buffer_size(proc, buffer);
        if (size &lt; buffer_size) {
            best_fit = n;
            n = n-&gt;rb_left;
        } else if (size &gt; buffer_size)
            n = n-&gt;rb_right;
        else {
            best_fit = n;
            break;
        }
    }
    ...
    if (n == NULL) {
        buffer = rb_entry(best_fit, struct binder_buffer, rb_node);
        buffer_size = binder_buffer_size(proc, buffer);
    }

    has_page_addr =(void *)(((uintptr_t)buffer-&gt;data + buffer_size) &amp; PAGE_MASK);
    if (n == NULL) {
        if (size + sizeof(struct binder_buffer) + 4 &gt;= buffer_size)
            buffer_size = size;
        else
            buffer_size = size + sizeof(struct binder_buffer);
    }
    //末端地址
    end_page_addr =     (void *)PAGE_ALIGN((uintptr_t)buffer-&gt;data + buffer_size);
    ...
    //分配物理页
    if (binder_update_page_range(proc, 1,
        (void *)PAGE_ALIGN((uintptr_t)buffer-&gt;data), end_page_addr, NULL))
        return NULL;
    rb_erase(best_fit, &amp;proc-&gt;free_buffers);
    buffer-&gt;free = 0;
    binder_insert_allocated_buffer(proc, buffer);
    if (buffer_size != size) {
        struct binder_buffer *new_buffer = (void *)buffer-&gt;data + size;
        list_add(&amp;new_buffer-&gt;entry, &amp;buffer-&gt;entry);
        new_buffer-&gt;free = 1;
        binder_insert_free_buffer(proc, new_buffer);
    }

    buffer-&gt;data_size = data_size;
    buffer-&gt;offsets_size = offsets_size;
    buffer-&gt;async_transaction = is_async;
    if (is_async) { //调整异步可用内存空间大小
        proc-&gt;free_async_space -= size + sizeof(struct binder_buffer);
    }
    return buffer;
}
</code></pre>

<h4 id="35-binder_thread_read">3.5 binder_thread_read</h4>

<pre ><code>binder_thread_read（）{
    //当已使用字节数为0时，将BR_NOOP响应码放入指针ptr
    if (*consumed == 0) {
            if (put_user(BR_NOOP, (uint32_t __user *)ptr))
                return -EFAULT;
            ptr += sizeof(uint32_t);
        }

retry:
    //binder_transaction()已设置transaction_stack不为空，则wait_for_proc_work为false.
    wait_for_proc_work = thread-&gt;transaction_stack == NULL &amp;&amp;
            list_empty(&amp;thread-&gt;todo);

    thread-&gt;looper |= BINDER_LOOPER_STATE_WAITING;
    if (wait_for_proc_work)
      proc-&gt;ready_threads++; //进程中空闲binder线程加1

    //只有当前线程todo队列为空，并且transaction_stack也为空，才会开始处于当前进程的事务
    if (wait_for_proc_work) {
        if (non_block) {
            ...
        } else
            //当进程todo队列没有数据,则进入休眠等待状态
            ret = wait_event_freezable_exclusive(proc-&gt;wait, binder_has_proc_work(proc, thread));
    } else {
        if (non_block) {
            ...
        } else
            //当线程todo队列有数据则执行往下执行；当线程todo队列没有数据，则进入休眠等待状态
            ret = wait_event_freezable(thread-&gt;wait, binder_has_thread_work(thread));
    }

    if (wait_for_proc_work)
      proc-&gt;ready_threads--; //退出等待状态, 则进程中空闲binder线程减1
    thread-&gt;looper &amp;= ~BINDER_LOOPER_STATE_WAITING;
    ...

    while (1) {

        uint32_t cmd;
        struct binder_transaction_data tr;
        struct binder_work *w;
        struct binder_transaction *t = NULL;
        //先从线程todo队列获取事务数据
        if (!list_empty(&amp;thread-&gt;todo)) {
            w = list_first_entry(&amp;thread-&gt;todo, struct binder_work, entry);
        // 线程todo队列没有数据, 则从进程todo对获取事务数据
        } else if (!list_empty(&amp;proc-&gt;todo) &amp;&amp; wait_for_proc_work) {
            w = list_first_entry(&amp;proc-&gt;todo, struct binder_work, entry);
        } else {
            //没有数据,则返回retry
            if (ptr - buffer == 4 &amp;&amp;
                !(thread-&gt;looper &amp; BINDER_LOOPER_STATE_NEED_RETURN))
                goto retry;
            break;
        }

        switch (w-&gt;type) {
            case BINDER_WORK_TRANSACTION:
                //获取transaction数据
                t = container_of(w, struct binder_transaction, work);
                break;

            case BINDER_WORK_TRANSACTION_COMPLETE:
                cmd = BR_TRANSACTION_COMPLETE;
                //将BR_TRANSACTION_COMPLETE写入*ptr，并跳出循环。
                put_user(cmd, (uint32_t __user *)ptr)；
                list_del(&amp;w-&gt;entry);
                kfree(w);
                break;

            case BINDER_WORK_NODE: ...    break;
            case BINDER_WORK_DEAD_BINDER:
            case BINDER_WORK_DEAD_BINDER_AND_CLEAR:
            case BINDER_WORK_CLEAR_DEATH_NOTIFICATION: ...   break;
        }

        //只有BINDER_WORK_TRANSACTION命令才能继续往下执行
        if (!t)
            continue;

        if (t-&gt;buffer-&gt;target_node) {
            //获取目标node
            struct binder_node *target_node = t-&gt;buffer-&gt;target_node;
            tr.target.ptr = target_node-&gt;ptr;
            tr.cookie =  target_node-&gt;cookie;
            t-&gt;saved_priority = task_nice(current);
            ...
            cmd = BR_TRANSACTION;  //设置命令为BR_TRANSACTION
        } else {
            tr.target.ptr = NULL;
            tr.cookie = NULL;
            cmd = BR_REPLY; //设置命令为BR_REPLY
        }
        tr.code = t-&gt;code;
        tr.flags = t-&gt;flags;
        tr.sender_euid = t-&gt;sender_euid;

        if (t-&gt;from) {
            struct task_struct *sender = t-&gt;from-&gt;proc-&gt;tsk;
            //当非oneway的情况下,将调用者进程的pid保存到sender_pid
            tr.sender_pid = task_tgid_nr_ns(sender,
                            current-&gt;nsproxy-&gt;pid_ns);
        } else {
            //当oneway的的情况下,则该值为0
            tr.sender_pid = 0;
        }

        tr.data_size = t-&gt;buffer-&gt;data_size;
        tr.offsets_size = t-&gt;buffer-&gt;offsets_size;
        tr.data.ptr.buffer = (void *)t-&gt;buffer-&gt;data + proc-&gt;user_buffer_offset;
        tr.data.ptr.offsets = tr.data.ptr.buffer +
                    ALIGN(t-&gt;buffer-&gt;data_size, sizeof(void *));

        //将cmd和数据写回用户空间
        if (put_user(cmd, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
        if (copy_to_user(ptr, &amp;tr, sizeof(tr)))
            return -EFAULT;
        ptr += sizeof(tr);

        list_del(&amp;t-&gt;work.entry);
        t-&gt;buffer-&gt;allow_user_free = 1;
        if (cmd == BR_TRANSACTION &amp;&amp; !(t-&gt;flags &amp; TF_ONE_WAY)) {
            t-&gt;to_parent = thread-&gt;transaction_stack;
            t-&gt;to_thread = thread;
            thread-&gt;transaction_stack = t;
        } else {
            t-&gt;buffer-&gt;transaction = NULL;
            kfree(t); //通信完成,则运行释放
        }
        break;
    }
done:
    *consumed = ptr - buffer;
    //当满足请求线程加已准备线程数等于0，已启动线程数小于最大线程数(15)，
    //且looper状态为已注册或已进入时创建新的线程。
    if (proc-&gt;requested_threads + proc-&gt;ready_threads == 0 &amp;&amp;
        proc-&gt;requested_threads_started &lt; proc-&gt;max_threads &amp;&amp;
        (thread-&gt;looper &amp; (BINDER_LOOPER_STATE_REGISTERED |
         BINDER_LOOPER_STATE_ENTERED))) {
        proc-&gt;requested_threads++;
        // 生成BR_SPAWN_LOOPER命令，用于创建新的线程
        put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer)；
    }
    return 0;
}
</code></pre>

<p>该方法功能说明:</p>

<p>此处wait_for_proc_work是指当前线程todo队列为空，并且transaction_stack也为空,该值为true.</p>

<ol>
  <li>当wait_for_proc_work = false, 则进入线程的等待队列thread-&gt;wait, 直到thread-&gt;todo队列有事务才往下执行;
    <ul>
      <li>获取并处理thread-&gt;todo队列中的事务;将相应的cmd和数据写回用户空间.</li>
    </ul>
  </li>
  <li>当wait_for_proc_work = true, 则进入线程的等待队列proc-&gt;wait, 直到proc-&gt;todo队列有事务才往下执行;
    <ul>
      <li>获取并处理proc-&gt;todo队列中的事务;将相应的cmd和数据写回用户空间.</li>
    </ul>
  </li>
</ol>

<p>到这里,可能有人好奇,对于[小节3.4]介绍了target_list有3种, 这里只会处理前2种:thread-&gt;todo, proc-&gt;todo.那么对于
target_node-&gt;async_todo的处理过程时间呢? [见小节5.4]</p>

<h4 id="36-下一步何去何从">3.6 下一步何去何从</h4>

<ol>
  <li>执行完binder_thread_write方法后, 通过binder_transaction()首先写入<code class="language-plaintext highlighter-rouge">BINDER_WORK_TRANSACTION_COMPLETE</code>写入当前线程.</li>
  <li>这时bwr.read_size &gt; 0, 回到binder_ioctl_write_read方法, 便开始执行binder_thread_read();</li>
  <li>在binder_thread_read()方法, 将获取cmd=BR_TRANSACTION_COMPLETE, 再将cmd和数据写回用户空间;</li>
  <li>一次Binder_ioctl完成,接着回调用户空间方法talkWithDriver(),刚才的数据以写入mIn.</li>
  <li>这时mIn有可读数据, 回到【小节2.10】IPC.waitForResponse()方法,完成BR_TRANSACTION_COMPLETE过程.
如果本次transaction采用非oneway方式, 这次Binder通信便完成, 否则还是要等待Binder服务端的返回。</li>
</ol>

<p>对于startService过程, 采用的便是非oneway方式,那么发起者进程还会继续停留在waitForResponse()方法,继续talkWithDriver()，然后休眠在binder_thread_read()的wait_event_freezable()过程，等待当前线程的todo队列有数据的到来，即等待收到BR_REPLY消息.</p>

<p>由于在前面binder_transaction()除了向自己所在线程写入了<code class="language-plaintext highlighter-rouge">BINDER_WORK_TRANSACTION_COMPLETE</code>, 还向目标进程(此处为system_server)写入了<code class="language-plaintext highlighter-rouge">BINDER_WORK_TRANSACTION</code>命令，那么接下里介绍system_server进程的工作。</p>

<h2 id="四-回到用户空间">四. 回到用户空间</h2>

<p><code class="language-plaintext highlighter-rouge">system_server</code>的binder线程是如何运转的，那么就需要从Binder线程的创建开始说起，
Binder线程的创建有两种方式：</p>

<ul>
  <li>ProcessState::self()-&gt;startThreadPool();</li>
  <li>IPCThreadState::self()-&gt;joinThreadPool();</li>
</ul>

<p>调用链如下：
startThreadPool()过程会创建新Binder线程，再经过层层调用也会进入joinThreadPool()方法。
<code class="language-plaintext highlighter-rouge">system_server</code>的binder线程从IPC.joinThreadPool –&gt;  IPC.getAndExecuteCommand() -&gt;  IPC.talkWithDriver() ,但talkWithDriver收到事务之后, 便进入IPC.executeCommand()方法。</p>

<p>接下来从joinThreadPool说起：</p>

<h3 id="41-ipcjointhreadpool">4.1 IPC.joinThreadPool</h3>

<pre ><code>void IPCThreadState::joinThreadPool(bool isMain)
{
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);
    set_sched_policy(mMyThreadId, SP_FOREGROUND);

    status_t result;
    do {
        processPendingDerefs(); //处理对象引用
        result = getAndExecuteCommand();//获取并执行命令【见小节4.2】

        if (result &lt; NO_ERROR &amp;&amp; result != TIMED_OUT &amp;&amp; result != -ECONNREFUSED &amp;&amp; result != -EBADF) {
            ALOGE("getAndExecuteCommand(fd=%d) returned unexpected error %d, aborting",
                  mProcess-&gt;mDriverFD, result);
            abort();
        }

        //对于binder非主线程不再使用，则退出
        if(result == TIMED_OUT &amp;&amp; !isMain) {
            break;
        }
    } while (result != -ECONNREFUSED &amp;&amp; result != -EBADF);

    mOut.writeInt32(BC_EXIT_LOOPER);
    talkWithDriver(false);
}
</code></pre>

<h3 id="42--ipcgetandexecutecommand">4.2  IPC.getAndExecuteCommand</h3>

<pre ><code>status_t IPCThreadState::getAndExecuteCommand()
{
    status_t result;
    int32_t cmd;

    result = talkWithDriver(); //该Binder Driver进行交互
    if (result &gt;= NO_ERROR) {
        size_t IN = mIn.dataAvail();
        if (IN &lt; sizeof(int32_t)) return result;
        cmd = mIn.readInt32(); //读取命令

        pthread_mutex_lock(&amp;mProcess-&gt;mThreadCountLock);
        mProcess-&gt;mExecutingThreadsCount++;
        pthread_mutex_unlock(&amp;mProcess-&gt;mThreadCountLock);

        result = executeCommand(cmd); //【见小节4.3】

        pthread_mutex_lock(&amp;mProcess-&gt;mThreadCountLock);
        mProcess-&gt;mExecutingThreadsCount--;
        pthread_cond_broadcast(&amp;mProcess-&gt;mThreadCountDecrement);
        pthread_mutex_unlock(&amp;mProcess-&gt;mThreadCountLock);

        set_sched_policy(mMyThreadId, SP_FOREGROUND);
    }
    return result;
}
</code></pre>

<p>此时system_server的binder线程空闲便是停留在binder_thread_read()方法来处理进程/线程新的事务。
由【小节3.4】可知收到的是<code class="language-plaintext highlighter-rouge">BINDER_WORK_TRANSACTION</code>命令, 再经过inder_thread_read()后生成命令cmd=<code class="language-plaintext highlighter-rouge">BR_TRANSACTION</code>.再将cmd和数据写回用户空间。</p>

<h3 id="43-ipcexecutecommand">4.3 IPC.executeCommand</h3>

<pre ><code>status_t IPCThreadState::executeCommand(int32_t cmd)
{
    BBinder* obj;
    RefBase::weakref_type* refs;
    status_t result = NO_ERROR;

    switch ((uint32_t)cmd) {
        case BR_TRANSACTION:
        {
            binder_transaction_data tr;
            result = mIn.read(&amp;tr, sizeof(tr)); //读取mIn数据
            if (result != NO_ERROR) break;

            Parcel buffer;
            //当buffer对象回收时，则会调用freeBuffer来回收内存【见小节4.3.1】
            buffer.ipcSetDataReference(
                reinterpret_cast&lt;const uint8_t*&gt;(tr.data.ptr.buffer),
                tr.data_size,
                reinterpret_cast&lt;const binder_size_t*&gt;(tr.data.ptr.offsets),
                tr.offsets_size/sizeof(binder_size_t), freeBuffer, this);

            const pid_t origPid = mCallingPid;
            const uid_t origUid = mCallingUid;
            const int32_t origStrictModePolicy = mStrictModePolicy;
            const int32_t origTransactionBinderFlags = mLastTransactionBinderFlags;

            //设置调用者的pid和uid
            mCallingPid = tr.sender_pid;
            mCallingUid = tr.sender_euid;
            mLastTransactionBinderFlags = tr.flags;

            int curPrio = getpriority(PRIO_PROCESS, mMyThreadId);
            if (gDisableBackgroundScheduling) {
                ... //不进入此分支
            } else {
                if (curPrio &gt;= ANDROID_PRIORITY_BACKGROUND) {
                    set_sched_policy(mMyThreadId, SP_BACKGROUND);
                }
            }

            Parcel reply;
            status_t error;
            if (tr.target.ptr) {
                //尝试通过弱引用获取强引用
                if (reinterpret_cast&lt;RefBase::weakref_type*&gt;(
                        tr.target.ptr)-&gt;attemptIncStrong(this)) {

                    // tr.cookie里存放的是BBinder子类JavaBBinder [见流程4.4]
                    error = reinterpret_cast&lt;BBinder*&gt;(tr.cookie)-&gt;transact(tr.code, buffer,
                            &amp;reply, tr.flags);
                    reinterpret_cast&lt;BBinder*&gt;(tr.cookie)-&gt;decStrong(this);
                } else {
                    error = UNKNOWN_TRANSACTION;
                }

            } else {
                error = the_context_object-&gt;transact(tr.code, buffer, &amp;reply, tr.flags);
            }

            if ((tr.flags &amp; TF_ONE_WAY) == 0) {
                if (error &lt; NO_ERROR) reply.setError(error);
                //对于非oneway, 需要reply通信过程,则向Binder驱动发送BC_REPLY命令【见小节4.3.1】
                sendReply(reply, 0);
            }
            //恢复pid和uid信息
            mCallingPid = origPid;
            mCallingUid = origUid;
            ...
        }
        break;

        case ...

        default:
            result = UNKNOWN_ERROR;
            break;
    }

    if (result != NO_ERROR) {
        mLastError = result;
    }
    return result;
}
</code></pre>

<ul>
  <li>对于oneway的场景, 执行完本次transact()则全部结束.</li>
  <li>对于非oneway, 需要reply的通信过程,则向Binder驱动发送BC_REPLY命令【见小节5.1】</li>
</ul>

<h4 id="431-ipcsetdatareference">4.3.1 ipcSetDataReference</h4>
<p>[-&gt; Parcel.cpp]</p>

<pre ><code>void Parcel::ipcSetDataReference(const uint8_t* data, size_t dataSize,
    const binder_size_t* objects, size_t objectsCount, release_func relFunc, void* relCookie)
{
    binder_size_t minOffset = 0;
    freeDataNoInit(); //【见小节4.3.2】
    mError = NO_ERROR;
    mData = const_cast&lt;uint8_t*&gt;(data);
    mDataSize = mDataCapacity = dataSize;
    mDataPos = 0;
    mObjects = const_cast&lt;binder_size_t*&gt;(objects);
    mObjectsSize = mObjectsCapacity = objectsCount;
    mNextObjectHint = 0;
    mOwner = relFunc;
    mOwnerCookie = relCookie;
    for (size_t i = 0; i &lt; mObjectsSize; i++) {
        binder_size_t offset = mObjects[i];
        if (offset &lt; minOffset) {
            mObjectsSize = 0;
            break;
        }
        minOffset = offset + sizeof(flat_binder_object);
    }
    scanForFds();
}
</code></pre>

<p>该方法的功能，Parcel成员变量说明：</p>

<ul>
  <li>mData：parcel数据起始地址</li>
  <li>mDataSize：parcel数据大小</li>
  <li>mObjects：flat_binder_object地址偏移量</li>
  <li>mObjectsSize：parcel中flat_binder_object个数</li>
  <li>mOwner：释放函数freebuffer</li>
  <li>mOwnerCookie：释放函数所需信息</li>
</ul>

<h4 id="432-freedatanoinit">4.3.2 freeDataNoInit</h4>
<p>[-&gt; Parcel.cpp]</p>

<pre ><code>void Parcel::freeDataNoInit()
{
    if (mOwner) {
        mOwner(this, mData, mDataSize, mObjects, mObjectsSize, mOwnerCookie);
    } else { //mOwner为空， 进入该分支
        releaseObjects(); //【见小节4.3.3】
        if (mData) {
            pthread_mutex_lock(&amp;gParcelGlobalAllocSizeLock);
            if (mDataCapacity &lt;= gParcelGlobalAllocSize) {
              gParcelGlobalAllocSize = gParcelGlobalAllocSize - mDataCapacity;
            } else {
              gParcelGlobalAllocSize = 0;
            }
            if (gParcelGlobalAllocCount &gt; 0) {
              gParcelGlobalAllocCount--;
            }
            pthread_mutex_unlock(&amp;gParcelGlobalAllocSizeLock);
            free(mData);
        }
        if (mObjects) free(mObjects);
    }
}
</code></pre>

<h4 id="433--releaseobjects">4.3.3  releaseObjects</h4>

<pre ><code>void Parcel::releaseObjects()
{
    const sp&lt;ProcessState&gt; proc(ProcessState::self());
    size_t i = mObjectsSize;
    uint8_t* const data = mData;
    binder_size_t* const objects = mObjects;
    while (i &gt; 0) {
        i--;
        const flat_binder_object* flat
            = reinterpret_cast&lt;flat_binder_object*&gt;(data+objects[i]);
        //【见小节4.3.4】
        release_object(proc, *flat, this, &amp;mOpenAshmemSize);
    }
}
</code></pre>

<h4 id="434-release_object">4.3.4 release_object</h4>

<pre ><code>static void release_object(const sp&lt;ProcessState&gt;&amp; proc,
    const flat_binder_object&amp; obj, const void* who, size_t* outAshmemSize)
{
    switch (obj.type) {
        case BINDER_TYPE_BINDER:
            if (obj.binder) {
                reinterpret_cast&lt;IBinder*&gt;(obj.cookie)-&gt;decStrong(who);
            }
            return;
        case BINDER_TYPE_WEAK_BINDER:
            if (obj.binder)
                reinterpret_cast&lt;RefBase::weakref_type*&gt;(obj.binder)-&gt;decWeak(who);
            return;
        case BINDER_TYPE_HANDLE: {
            const sp&lt;IBinder&gt; b = proc-&gt;getStrongProxyForHandle(obj.handle);
            if (b != NULL) {
                b-&gt;decStrong(who);
            }
            return;
        }
        case BINDER_TYPE_WEAK_HANDLE: {
            const wp&lt;IBinder&gt; b = proc-&gt;getWeakProxyForHandle(obj.handle);
            if (b != NULL) b.get_refs()-&gt;decWeak(who);
            return;
        }
        case BINDER_TYPE_FD: {
            ...
            return;
        }
    }
}
</code></pre>

<p>根据flat_binder_object的类型，来决定减少相应的强弱引用。</p>

<h4 id="435-parcel">4.3.5 ~Parcel</h4>
<p>[-&gt; Parcel.cpp]</p>

<p>当[小节4.3]executeCommand执行完成后， 便会释放局部变量Parcel buffer，则会析构Parcel。</p>

<pre ><code>Parcel::~Parcel()
{
    freeDataNoInit();
}

void Parcel::freeDataNoInit()
{
    if (mOwner) { //此处mOwner等于freeBuffer 【见小节4.3.6】
        mOwner(this, mData, mDataSize, mObjects, mObjectsSize, mOwnerCookie);
    } else { 
        ...
    }
}
</code></pre>

<p>接下来，进入IPC的freeBuffer过程。</p>

<h4 id="436-freebuffer">4.3.6 freeBuffer</h4>
<p>[-&gt; IPCThreadState.cpp]</p>

<pre ><code>void IPCThreadState::freeBuffer(Parcel* parcel, const uint8_t* data,
                                size_t /*dataSize*/,
                                const binder_size_t* /*objects*/,
                                size_t /*objectsSize*/, void* /*cookie*/)
{
    if (parcel != NULL) parcel-&gt;closeFileDescriptors();
    IPCThreadState* state = self();
    state-&gt;mOut.writeInt32(BC_FREE_BUFFER);
    state-&gt;mOut.writePointer((uintptr_t)data);
}
</code></pre>

<p>向Binder驱动写入BC_FREE_BUFFER命令。</p>

<h3 id="44---bbindertransact">4.4   BBinder.transact</h3>
<p>[-&gt; Binder.cpp ::BBinder ]</p>

<pre ><code>status_t BBinder::transact(
    uint32_t code, const Parcel&amp; data, Parcel* reply, uint32_t flags)
{
    data.setDataPosition(0);

    status_t err = NO_ERROR;
    switch (code) {
        case PING_TRANSACTION:
            reply-&gt;writeInt32(pingBinder());
            break;
        default:
            err = onTransact(code, data, reply, flags); //【见流程4.5】
            break;
    }

    if (reply != NULL) {
        reply-&gt;setDataPosition(0);
    }

    return err;
}
</code></pre>

<h3 id="45-javabbinderontransact">4.5 JavaBBinder.onTransact</h3>
<p>[-&gt; android_util_Binder.cpp]</p>

<pre ><code>virtual status_t onTransact(
    uint32_t code, const Parcel&amp; data, Parcel* reply, uint32_t flags = 0)
{
    JNIEnv* env = javavm_to_jnienv(mVM);

    IPCThreadState* thread_state = IPCThreadState::self();

    //调用Binder.execTransact [见流程4.6]
    jboolean res = env-&gt;CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
        code, reinterpret_cast&lt;jlong&gt;(&amp;data), reinterpret_cast&lt;jlong&gt;(reply), flags);

    jthrowable excep = env-&gt;ExceptionOccurred();
    if (excep) {
        res = JNI_FALSE;
        //发生异常, 则清理JNI本地引用
        env-&gt;DeleteLocalRef(excep);
    }
    ...
    return res != JNI_FALSE ? NO_ERROR : UNKNOWN_TRANSACTION;
}
</code></pre>

<p>还记得AndroidRuntime::startReg过程吗, 其中有一个过程便是register_android_os_Binder(),该过程会把gBinderOffsets.mExecTransact便是Binder.java中的execTransact()方法.</p>

<p>另外,此处mObject是在服务注册addService过程,会调用writeStrongBinder方法, 将Binder对象传入了JavaBBinder构造函数的参数, 最终赋值给mObject. 在本次通信过程中Object为ActivityManagerNative对象.</p>

<p>此处斗转星移, 从C++代码回到了Java代码. 进入AMN.execTransact, 由于AMN继续于Binder对象, 接下来进入Binder.execTransact</p>

<h3 id="46-binderexectransact">4.6 Binder.execTransact</h3>
<p>[Binder.java]</p>

<pre ><code>private boolean execTransact(int code, long dataObj, long replyObj,
        int flags) {
    Parcel data = Parcel.obtain(dataObj);
    Parcel reply = Parcel.obtain(replyObj);

    boolean res;
    try {
        // 调用子类AMN.onTransact方法 [见流程4.7]
        res = onTransact(code, data, reply, flags);
    } catch (RemoteException e) {
        if ((flags &amp; FLAG_ONEWAY) != 0) {
            ...
        } else {
            //非oneway的方式,则会将异常写回reply
            reply.setDataPosition(0);
            reply.writeException(e);
        }
        res = true;
    } catch (RuntimeException e) {
        if ((flags &amp; FLAG_ONEWAY) != 0) {
            ...
        } else {
            reply.setDataPosition(0);
            reply.writeException(e);
        }
        res = true;
    } catch (OutOfMemoryError e) {
        RuntimeException re = new RuntimeException("Out of memory", e);
        reply.setDataPosition(0);
        reply.writeException(re);
        res = true;
    }
    reply.recycle();
    data.recycle();
    return res;
}
</code></pre>

<p>当发生RemoteException, RuntimeException, OutOfMemoryError, 对于非oneway的情况下都会把异常传递给调用者.</p>

<h3 id="47-amnontransact">4.7 AMN.onTransact</h3>
<p>[-&gt; ActivityManagerNative.java]</p>

<pre ><code>public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
        throws RemoteException {
    switch (code) {
    ...
     case START_SERVICE_TRANSACTION: {
        data.enforceInterface(IActivityManager.descriptor);
        IBinder b = data.readStrongBinder();
        //生成ApplicationThreadNative的代理对象，即ApplicationThreadProxy对象
        IApplicationThread app = ApplicationThreadNative.asInterface(b);
        Intent service = Intent.CREATOR.createFromParcel(data);
        String resolvedType = data.readString();
        String callingPackage = data.readString();
        int userId = data.readInt();
        //调用ActivityManagerService的startService()方法【见流程4.8】
        ComponentName cn = startService(app, service, resolvedType, callingPackage, userId);
        reply.writeNoException();
        ComponentName.writeToParcel(cn, reply);
        return true;
    }
}
</code></pre>

<h3 id="48-amsstartservice">4.8 AMS.startService</h3>

<pre ><code>public ComponentName startService(IApplicationThread caller, Intent service,
        String resolvedType, String callingPackage, int userId)
        throws TransactionTooLargeException {

    synchronized(this) {
        ...
        ComponentName res = mServices.startServiceLocked(caller, service,
                resolvedType, callingPid, callingUid, callingPackage, userId);
        Binder.restoreCallingIdentity(origId);
        return res;
    }
}
</code></pre>

<p>历经千山万水, 总算是进入了AMS.startService. 当system_server收到BR_TRANSACTION的过程后，通信并没有完全结束，还需将服务启动完成的回应消息
告诉给发起端进程。</p>

<h2 id="五-reply流程">五. Reply流程</h2>

<p>还记得前面【小节2.10】IPC.waitForResponse()过程，对于非oneway的方式，还仍在一直等待system_server这边的响应呢，只有收到BR_REPLY，或者BR_DEAD_REPLY，或者BR_FAILED_REPLY，再或许其他BR_命令执行出错的情况下，该waitForResponse()才会退出。</p>

<p>BR_REPLY命令是如何来的呢？【小节4.3】IPC.executeCommand()过程处理完BR_TRANSACTION命令的同时，还会通过sendReply()向Binder Driver发送BC_REPLY消息，接下来从该方法说起。</p>

<h4 id="51-ipcsendreply">5.1 IPC.sendReply</h4>

<pre ><code>status_t IPCThreadState::sendReply(const Parcel&amp; reply, uint32_t flags)
{
    status_t err;
    status_t statusBuffer;
    //[见小节2.10]
    err = writeTransactionData(BC_REPLY, flags, -1, 0, reply, &amp;statusBuffer);
    if (err &lt; NO_ERROR) return err;
    //[见小节5.3]
    return waitForResponse(NULL, NULL);
}
</code></pre>

<p>先将数据写入mOut；再进waitForResponse，等待应答，此时同理也是等待BR_TRANSACTION_COMPLETE。
同理经过IPC.talkWithDriver -&gt; binder_ioctl -&gt; binder_ioctl_write_read -&gt; binder_thread_write，
再就是进入binder_transaction方法。</p>

<h4 id="52--bc_reply">5.2  BC_REPLY</h4>

<pre ><code>// reply =true
static void binder_transaction(struct binder_proc *proc,
             struct binder_thread *thread,
             struct binder_transaction_data *tr, int reply)
{
  ...
  if (reply) {
    in_reply_to = thread-&gt;transaction_stack; //接收端的事务栈
    ...
    thread-&gt;transaction_stack = in_reply_to-&gt;to_parent;
    target_thread = in_reply_to-&gt;from; //发起端的线程

        //发起端线程不能为空
    if (target_thread == NULL) {
      return_error = BR_DEAD_REPLY;
      goto err_dead_binder;
    }

        //发起端线程的事务栈 要等于 接收端的事务栈
    if (target_thread-&gt;transaction_stack != in_reply_to) {
      return_error = BR_FAILED_REPLY;
      in_reply_to = NULL;
      target_thread = NULL;
      goto err_dead_binder;
    }
    target_proc = target_thread-&gt;proc; //发起端的进程
  } else {
    ...
  }

  if (target_thread) {
      //发起端的线程
      target_list = &amp;target_thread-&gt;todo;
      target_wait = &amp;target_thread-&gt;wait;
    } else {
      ...
    }

    t = kzalloc(sizeof(*t), GFP_KERNEL);
    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
    ...

    if (!reply &amp;&amp; !(tr-&gt;flags &amp; TF_ONE_WAY))
      t-&gt;from = thread;
    else
      t-&gt;from = NULL; //进入该分支
    t-&gt;sender_euid = task_euid(proc-&gt;tsk);
    t-&gt;to_proc = target_proc;
    t-&gt;to_thread = target_thread;
    t-&gt;code = tr-&gt;code;
    t-&gt;flags = tr-&gt;flags;
    t-&gt;priority = task_nice(current);

    // 发起端进程分配buffer
    t-&gt;buffer = binder_alloc_buf(target_proc, tr-&gt;data_size,
      tr-&gt;offsets_size, !reply &amp;&amp; (t-&gt;flags &amp; TF_ONE_WAY));
    ...
    t-&gt;buffer-&gt;allow_user_free = 0;
    t-&gt;buffer-&gt;transaction = t;
    t-&gt;buffer-&gt;target_node = target_node;
    if (target_node)
      binder_inc_node(target_node, 1, 0, NULL);

    //分别拷贝用户空间的binder_transaction_data中ptr.buffer和ptr.offsets到内核
    copy_from_user(t-&gt;buffer-&gt;data,
       (const void __user *)(uintptr_t)tr-&gt;data.ptr.buffer, tr-&gt;data_size);
    copy_from_user(offp,
       (const void __user *)(uintptr_t)tr-&gt;data.ptr.offsets, tr-&gt;offsets_size);
    ...

    if (reply) {
      binder_pop_transaction(target_thread, in_reply_to);
    } else if (!(t-&gt;flags &amp; TF_ONE_WAY)) {
      ...
    } else {
      ...
    }

    //将BINDER_WORK_TRANSACTION添加到目标队列，本次通信的目标队列为target_thread-&gt;todo
    t-&gt;work.type = BINDER_WORK_TRANSACTION;
    list_add_tail(&amp;t-&gt;work.entry, target_list);

    //将BINDER_WORK_TRANSACTION_COMPLETE添加到当前线程的todo队列
    tcomplete-&gt;type = BINDER_WORK_TRANSACTION_COMPLETE;
    list_add_tail(&amp;tcomplete-&gt;entry, &amp;thread-&gt;todo);

    //唤醒等待队列，本次通信的目标队列为target_thread-&gt;wait
    if (target_wait)
        wake_up_interruptible(target_wait);
    return;
</code></pre>

<p>binder_transaction -&gt; binder_thread_read -&gt; IPC.waitForResponse，收到BR_REPLY来回收buffer.</p>

<h4 id="53-br_reply">5.3 BR_REPLY</h4>

<pre ><code>status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    int32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) &lt; NO_ERROR) break; // 【见小节2.11】
        if (mIn.dataAvail() == 0) continue;
        ...
        cmd = mIn.readInt32();
        switch (cmd) {
          ...
          case BR_REPLY:
           {
               binder_transaction_data tr;
               err = mIn.read(&amp;tr, sizeof(tr));
               if (err != NO_ERROR) goto finish;

               if (reply) {
                   ...
               } else {
                   // 释放buffer[见小节5.4]
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
            ...
            break;
        }
    }
    ...
}
</code></pre>

<h4 id="54-ipcfreebuffer">5.4 IPC.freeBuffer</h4>

<pre ><code>void IPCThreadState::freeBuffer(Parcel* parcel, const uint8_t* data,
                                size_t /*dataSize*/,
                                const binder_size_t* /*objects*/,
                                size_t /*objectsSize*/, void* /*cookie*/)
{
    if (parcel != NULL) parcel-&gt;closeFileDescriptors();
    IPCThreadState* state = self();
    state-&gt;mOut.writeInt32(BC_FREE_BUFFER);
    state-&gt;mOut.writePointer((uintptr_t)data);
}
</code></pre>

<p>将BC_FREE_BUFFER写入mOut,再talkWithDriver()</p>

<h5 id="55-bc_free_buffer">5.5 BC_FREE_BUFFER</h5>

<pre ><code>static int binder_thread_write(struct binder_proc *proc,
            struct binder_thread *thread,
            binder_uintptr_t binder_buffer, size_t size,
            binder_size_t *consumed)
{
    uint32_t cmd;
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;
    while (ptr &lt; end &amp;&amp; thread-&gt;return_error == BR_OK) {
        //拷贝用户空间的cmd命令，此时为BC_FREE_BUFFER
        if (get_user(cmd, (uint32_t __user *)ptr)) -EFAULT;
        ptr += sizeof(uint32_t);
        switch (cmd) {
        case BC_TRANSACTION:
        case BC_REPLY: ...
        case BC_FREE_BUFFER: {
            void __user *data_ptr;
            struct binder_buffer *buffer;

            if (get_user(data_ptr, (void * __user *)ptr)) return -EFAULT;
            ptr += sizeof(void *);

            buffer = binder_buffer_lookup(proc, data_ptr);
            ...

            if (buffer-&gt;transaction) {
                buffer-&gt;transaction-&gt;buffer = NULL;
                buffer-&gt;transaction = NULL;
            }
            // binder_buffer存在异步事务,且binder_node不为空
            if (buffer-&gt;async_transaction &amp;&amp; buffer-&gt;target_node) {
                if (list_empty(&amp;buffer-&gt;target_node-&gt;async_todo))
                    buffer-&gt;target_node-&gt;has_async_transaction = 0;
                else
                    //当异步队列async_todo也不为空,则事务追加到该线程todo队列.
                    list_move_tail(buffer-&gt;target_node-&gt;async_todo.next, &amp;thread-&gt;todo);
            }

            binder_transaction_buffer_release(proc, buffer, NULL);
            binder_free_buf(proc, buffer);
            break;
        }
    }
    *consumed = ptr - buffer;
  }
  return 0;
}
</code></pre>

<p>接收端线程处理BC_FREE_BUFFER命令:</p>

<ul>
  <li>当binder_buffer存在异步事务,当异步队列async_todo也不为空,则事务追加到该线程todo队列.</li>
  <li>释放当前的buffer.</li>
</ul>

<h5 id="56--binder_thread_read">5.6  binder_thread_read</h5>

<pre ><code>binder_thread_read（）{
    ...
    while (1) {

        uint32_t cmd;
        struct binder_transaction_data tr;
        struct binder_work *w;
        struct binder_transaction *t = NULL;

        //从线程todo队列获取事务数据
        if (!list_empty(&amp;thread-&gt;todo)) {
            w = list_first_entry(&amp;thread-&gt;todo, struct binder_work, entry);
        } else if (!list_empty(&amp;proc-&gt;todo) &amp;&amp; wait_for_proc_work) {
            ...
        } else {
            ...
        }

        switch (w-&gt;type) {
            case BINDER_WORK_TRANSACTION:
                //获取transaction数据
                t = container_of(w, struct binder_transaction, work);
                break;

            ...
        }

        ...
        if (t-&gt;buffer-&gt;target_node) {
            //获取目标node
            struct binder_node *target_node = t-&gt;buffer-&gt;target_node;
            tr.target.ptr = target_node-&gt;ptr;
            tr.cookie =  target_node-&gt;cookie;
            t-&gt;saved_priority = task_nice(current);
            ...
            cmd = BR_TRANSACTION;  //设置命令为BR_TRANSACTION
        } else {
            tr.target.ptr = NULL;
            tr.cookie = NULL;
            cmd = BR_REPLY; //设置命令为BR_REPLY
        }

        tr.code = t-&gt;code;
        tr.flags = t-&gt;flags;
        tr.sender_euid = t-&gt;sender_euid;

        ...
        //将cmd和数据写回用户空间
        if (put_user(cmd, (uint32_t __user *)ptr)) return -EFAULT;
        ptr += sizeof(uint32_t);
        if (copy_to_user(ptr, &amp;tr, sizeof(tr)))  return -EFAULT;
        ptr += sizeof(tr);

        list_del(&amp;t-&gt;work.entry);
        t-&gt;buffer-&gt;allow_user_free = 1;
        if (cmd == BR_TRANSACTION &amp;&amp; !(t-&gt;flags &amp; TF_ONE_WAY)) {
            t-&gt;to_parent = thread-&gt;transaction_stack;
            t-&gt;to_thread = thread;
            thread-&gt;transaction_stack = t;
        } else {
            t-&gt;buffer-&gt;transaction = NULL;
            kfree(t); //通信完成,则运行释放
        }
        break;
    }

    ...
    return 0;
}
</code></pre>

<h2 id="六-总结">六. 总结</h2>

<p>本文详细地介绍如何从AMP.startService是如何通过Binder一步步调用进入到system_server进程的AMS.startService. 整个过程涉及Java framework, native, kernel driver各个层面知识. 仅仅一个Binder IPC调用, 就花费了如此大篇幅来讲解, 可见系统之庞大. 整个过程的调用流程:</p>

<h3 id="61-通信流程">6.1 通信流程</h3>

<p>从通信流程角度来看整个过程:
<img src="https://img-blog.csdnimg.cn/8c6689d795314c38a2ba8dd97fc1b3c2.jpg?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt="binder_ipc_process" /></p>

<p>图解:</p>

<ol>
  <li>发起端线程向Binder Driver发起binder ioctl请求后, 便采用环不断talkWithDriver,此时该线程处于阻塞状态, 直到收到如下BR_XXX命令才会结束该过程.
    <ul>
      <li>BR_TRANSACTION_COMPLETE: oneway模式下,收到该命令则退出</li>
      <li>BR_REPLY: 非oneway模式下,收到该命令才退出;</li>
      <li>BR_DEAD_REPLY: 目标进程/线程/binder实体为空, 以及释放正在等待reply的binder thread或者binder buffer;</li>
      <li>BR_FAILED_REPLY: 情况较多,比如非法handle, 错误事务栈, security, 内存不足, buffer不足, 数据拷贝失败, 节点创建失败, 各种不匹配等问题</li>
      <li>BR_ACQUIRE_RESULT: 目前未使用的协议;</li>
    </ul>
  </li>
  <li>左图中waitForResponse收到BR_TRANSACTION_COMPLETE,则直接退出循环, 则没有机会执行executeCommand()方法, 故将其颜色画为灰色. 除以上5种BR_XXX命令, 当收到其他BR命令,则都会执行executeCommand过程.</li>
  <li>目标Binder线程创建后, 便进入joinThreadPool()方法, 采用循环不断地循环执行getAndExecuteCommand()方法, 当bwr的读写buffer都没有数据时,则阻塞在binder_thread_read的wait_event过程.  另外,正常情况下binder线程一旦创建则不会退出.</li>
</ol>

<h3 id="62-通信协议">6.2 通信协议</h3>

<p>从通信协议的角度来看这个过程:</p>

<p><img src="https://img-blog.csdnimg.cn/4e6fb58ffff24b669787247d7d3b76a2.jpg?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16)" alt="binder_transaction" /></p>

<ul>
  <li>Binder客户端或者服务端向Binder Driver发送的命令都是以BC_开头,例如本文的<code class="language-plaintext highlighter-rouge">BC_TRANSACTION</code>和<code class="language-plaintext highlighter-rouge">BC_REPLY</code>, 所有Binder Driver向Binder客户端或者服务端发送的命令则都是以BR_开头, 例如本文中的<code class="language-plaintext highlighter-rouge">BR_TRANSACTION</code>和<code class="language-plaintext highlighter-rouge">BR_REPLY</code>.</li>
  <li>只有当<code class="language-plaintext highlighter-rouge">BC_TRANSACTION</code>或者<code class="language-plaintext highlighter-rouge">BC_REPLY</code>时, 才调用binder_transaction()来处理事务. 并且都会回应调用者一个<code class="language-plaintext highlighter-rouge">BINDER_WORK_TRANSACTION_COMPLETE</code>事务, 经过binder_thread_read()会转变成<code class="language-plaintext highlighter-rouge">BR_TRANSACTION_COMPLETE</code>.</li>
  <li>startService过程便是一个非oneway的过程, 那么oneway的通信过程如下所述.</li>
</ul>

<h3 id="63-说一说oneway">6.3 说一说oneway</h3>

<p>上图是非oneway通信过程的协议图, 下图则是对于oneway场景下的通信协议图:</p>

<p><img src="https://img-blog.csdnimg.cn/d5efdcec6bb94bf8b2194c4e006892fb.jpg?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt="binder_transaction_oneway" /></p>

<p>当收到BR_TRANSACTION_COMPLETE则程序返回,有人可能觉得好奇,为何oneway怎么还要等待回应消息? 我举个例子,你就明白了.</p>

<p>你(app进程)要给远方的家人(system_server进程)邮寄一封信(transaction), 你需要通过邮寄员(Binder Driver)来完成.整个过程如下:</p>

<ol>
  <li>你把信交给邮寄员(<code class="language-plaintext highlighter-rouge">BC_TRANSACTION</code>);</li>
  <li>邮寄员收到信后, 填一张单子给你作为一份回执(<code class="language-plaintext highlighter-rouge">BR_TRANSACTION_COMPLETE</code>). 这样你才放心知道邮递员已确定接收信, 否则就这样走了,信到底有没有交到邮递员手里都不知道,这样的通信实在太让人不省心, 长时间收不到远方家人的回信, 无法得知是在路的中途信件丢失呢,还是压根就没有交到邮递员的手里. 所以说oneway也得知道信是投递状态是否成功.</li>
  <li>邮递员利用交通工具(Binder Driver),将信交给了你的家人(<code class="language-plaintext highlighter-rouge">BR_TRANSACTION</code>);</li>
</ol>

<p>当你收到回执(BR_TRANSACTION_COMPLETE)时心里也不期待家人回信, 那么这便是一次oneway的通信过程.</p>

<p>如果你希望家人回信, 那便是非oneway的过程,在上述步骤2后并不是直接返回,而是继续等待着收到家人的回信, 经历前3个步骤之后继续执行:</p>

<ol>
  <li>家人收到信后, 立马写了个回信交给邮递员<code class="language-plaintext highlighter-rouge">BC_REPLY</code>;</li>
  <li>同样,邮递员要写一个回执(<code class="language-plaintext highlighter-rouge">BR_TRANSACTION_COMPLETE</code>)给你家人;</li>
  <li>邮递员再次利用交通工具(Binder Driver), 将回信成功交到你的手上(<code class="language-plaintext highlighter-rouge">BR_REPLY</code>)</li>
</ol>

<p>这便是一次完成的非oneway通信过程.</p>

<p>oneway与非oneway: 都是需要等待Binder Driver的回应消息BR_TRANSACTION_COMPLETE. 主要区别在于oneway的通信收到BR_TRANSACTION_COMPLETE则返回,而不会再等待BR_REPLY消息的到来. 另外，oneway的binder IPC则接收端无法获取对方的pid.</p>

<h3 id="64-小规律">6.4 小规律</h3>

<ul>
  <li>BC_TRANSACTION +  BC_REPLY =  BR_TRANSACTION_COMPLETE +  BR_DEAD_REPLY +  BR_FAILED_REPLY</li>
  <li>Binder线程只有当本线程的thread-&gt;todo队列为空，并且thread-&gt;transaction_stack也为空，才会去处理当前进程的事务，
否则会继续处理或等待当前线程的todo队列事务。换句话说，就是只有当前线程的事务;</li>
  <li>binder_thread_write: 添加成员到todo队列;</li>
  <li>binder_thread_read: 消耗todo队列;</li>
  <li>对于处于空闲可用的,或者Ready的binder线程是指停在binder_thread_read()的wait_event地方的Binder线程;</li>
  <li>每一次BR_TRANSACTION或者BR_REPLY结束之后都会调用freeBuffer().</li>
  <li>ProcessState.mHandleToObject记录着handle与对应的BpBinder信息。</li>
</ul>

<p>整个过程copy once便是指binder_transaction()过程把binder_transaction_data-&gt;data拷贝到目标进程的buffer。</p>

<h3 id="65-数据流">6.5 数据流</h3>

<p><img src="https://img-blog.csdnimg.cn/c47873eed18f42d89c3dd7b8af7b454e.jpg?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt="binder_transaction_data" /></p>

<ul>
  <li>[2.1]AMP.startService：组装flat_binder_object对象等组成的Parcel data；</li>
  <li>[2.9]IPC.writeTransactionData：组装BC_TRANSACTION和binder_transaction_data结构体，写入mOut;</li>
  <li>[2.11]IPC.talkWithDriver: 组装BINDER_WRITE_READ和binder_write_read结构体，通过ioctl传输到驱动层。</li>
</ul>

<p>进入驱动后</p>

<ul>
  <li>[3.3]binder_thread_write: 处理binder_write_read.write_buffer数据</li>
  <li>[3.4]binder_transaction: 处理write_buffer.binder_transaction_data数据；
    <ul>
      <li>创建binder_transaction结构体，记录事务通信的线程来源以及事务链条等相关信息；</li>
      <li>分配binder_buffer结构体，拷贝当前线程binder_transaction_data的data数据到binder_buffer-&gt;data；</li>
    </ul>
  </li>
  <li>[3.5]binder_thread_read:  处理binder_transaction结构体数据
    <ul>
      <li>组装cmd=BR_TRANSACTION和binder_transaction_data结构体，写入binder_write_read.read_buffer数据</li>
    </ul>
  </li>
</ul>

<p>回到用户空间</p>

<ul>
  <li>[4.3]IPC.executeCommand：处理BR_TRANSACTION命令, 将binder_transaction_data数据解析成BBinder.transact()所需的参数</li>
  <li>[4.7] AMN.onTransact: 层层回调，进入该方法，反序列化数据后，调用startService()方法。</li>
</ul>
