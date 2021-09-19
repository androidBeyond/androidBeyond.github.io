---
layout:     post
title:      Android10 Binder机制5-绑定服务
subtitle:   本篇文章分析下bindService流程，从客户端调用bindService到服务器端通过ServiceConnected对象返回代理类给客户端
date:       2021-03-10
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


<h2 id="一、概述"><a href="#一、概述" class="headerlink" title="一、概述"></a>一、概述</h2>
<h3 id="1-1-Binder-IPC原理"><a href="#1-1-Binder-IPC原理" class="headerlink" title="1.1 Binder IPC原理"></a>1.1 Binder IPC原理</h3><p>Binder通信采用C/S架构，包含Client，Server，ServiceManager以及binder驱动，其中ServiceManager用于管理系统中的各种服务，下面是以AMS服务为例的架构图：</p>
<p><img src="https://img-blog.csdnimg.cn/82fd1ecd1c4140a88badb932049bd961.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_17,color_FFFFFF,t_70,g_se,x_16" alt="bindservice_binder_frame" style="zoom:67%;"></p>
<p>无论是注册服务还是获取服务的过程都需要ServiceManager，此处的ServiceManager是指Native层的ServiceManager(C++)，并非指framework层的ServiceManager（Java）。ServiceManager是整个Binder通信机制的大管家，是Android进程间通信机制Binder的守护进程。Client端和Server端通信时都需要先获取ServiceManager接口，才能开始通信服务，查找到目标信息可以缓存起来则不需要每次都向ServiceManager请求。</p>
<p>图中Client/Server/ServiceManager之间的相互通信都是基于Binder机制，其主要分为三个过程：</p>
<p>1.注册服务：AMS注册到ServiceManager。这个过程AMS所在的进程(system_server)是客户端，ServiceManager是服务端。</p>
<p>2.获取服务：Client进程使用AMS前，必须向ServiceManager中获取AMS的代理类。这个过程：AMS的代理类是客户端，ServiceManager是服务端。</p>
<p>3.使用服务：app进程根据得到的代理类，便可以直接与AMS所在进程交互。这个过程：代理类所在进程是客户端，AMS所在进程(system_server)是服务端。</p>
<p>Client,Server，ServiceManager之间不是直接交互的，都是通过与Binder Driver进行交互的，从而实现IPC通信方式。Binder驱动位于内核层，Client,Server,ServiceManager位于用户空间。Binder驱动和ServiceManager可以看做是Android平台的基础架构，而Client和Server是Android应用层。</p>
<p>前面已经分析过第一第二个过程注册服务和获取服务，本文主要介绍第三个过程使用服务，以bindService过程为例。</p>
<h3 id="1-2-bindService流程"><a href="#1-2-bindService流程" class="headerlink" title="1.2 bindService流程"></a>1.2 bindService流程</h3><p>bindService流程如下图，从客户端调用bindService到服务器端通过ServiceConnected对象返回代理类给客户端，下面将从源码的角度分析这个过程。</p>
<p><img src="https://img-blog.csdnimg.cn/6bbbd8465bc04156a5efe18ba274d36f.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt="bindservice" style="zoom: 67%;"></p>
<h2 id="二、客户端进程"><a href="#二、客户端进程" class="headerlink" title="二、客户端进程"></a>二、客户端进程</h2><h3 id="2-1-CL-bindService"><a href="#2-1-CL-bindService" class="headerlink" title="2.1 CL.bindService"></a>2.1 CL.bindService</h3><p>[-&gt;ContextImpl.java]</p>

<pre><code>
@Override
 public boolean bindService(Intent service, ServiceConnection conn,
         int flags) {
     warnIfCallingFromSystemProcess();
     return bindServiceCommon(service, conn, flags, mMainThread.getHandler(), getUser());
 }
</code></pre>

<h3 id="2-2-CL-bindServiceCommon"><a href="#2-2-CL-bindServiceCommon" class="headerlink" title="2.2 CL.bindServiceCommon"></a>2.2 CL.bindServiceCommon</h3><p>[-&gt;ContextImpl.java]</p>

<pre><code>
private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags, Handler
          handler, UserHandle user) {
      // Keep this in sync with DevicePolicyManager.bindDeviceAdminServiceAsUser.
      IServiceConnection sd;
      if (conn == null) {
          throw new IllegalArgumentException("connection is null");
      }
      if (mPackageInfo != null) {
          //获取的内部静态类InnerConnection
          sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
      } else {
          throw new RuntimeException("Not supported in system context");
      }
      validateServiceIntent(service);
      try {
          IBinder token = getActivityToken();
          if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null
                  && mPackageInfo.getApplicationInfo().targetSdkVersion
                  < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
              flags |= BIND_WAIVE_PRIORITY;
          }
          service.prepareToLeaveProcess(this);
          //bindservice见2.3节
          int res = ActivityManager.getService().bindService(
              mMainThread.getApplicationThread(), getActivityToken(), service,
              service.resolveTypeIfNeeded(getContentResolver()),
              sd, flags, getOpPackageName(), user.getIdentifier());
          if (res < 0) {
              throw new SecurityException(
                      "Not allowed to bind to service " + service);
          }
          return res != 0;
      } catch (RemoteException e) {
          throw e.rethrowFromSystemServer();
      }
  }</code></pre>

<p>主要的工作如下：</p>
<ul>
<li><p>创建对象LoadedApk.ServiceDispatcher对象</p>
</li>
<li><p>向AMS发送bindservice请求</p>
</li>
</ul>
<h4 id="2-2-1-getServiceDispatcher"><a href="#2-2-1-getServiceDispatcher" class="headerlink" title="2.2.1 getServiceDispatcher"></a>2.2.1 getServiceDispatcher</h4><p>[-&gt;LoadedApk.java]</p>

<pre><code>
public final IServiceConnection getServiceDispatcher(ServiceConnection c,
           Context context, Handler handler, int flags) {
       synchronized (mServices) {
           LoadedApk.ServiceDispatcher sd = null;
           ArrayMap&lt;ServiceConnection, LoadedApk.ServiceDispatcher&gt; map = mServices.get(context);
           if (map != null) {
               if (DEBUG) Slog.d(TAG, "Returning existing dispatcher " + sd + " for conn " + c);
               sd = map.get(c);
           }
           if (sd == null) {
               //创建服务分发对象
               sd = new ServiceDispatcher(c, context, handler, flags);
               if (DEBUG) Slog.d(TAG, "Creating new dispatcher " + sd + " for conn " + c);
               if (map == null) {
                   map = new ArrayMap&lt;&gt;();
                   mServices.put(context, map);
               }
               //ServiceConnection为key，ServiceDispatcher为value保存到map
               map.put(c, sd);
           } else {
               sd.validate(context, handler);
           }
           return sd.getIServiceConnection();
       }
   }
</code></pre>

<ul>
<li>mServices记录所有context里面每个ServiceConnection以及所对应的所对应的LoadedApk.ServiceDispatcher对象；同一个ServiceConnection只会创建一次，</li>
</ul>
<ul>
<li>返回的对象是ServiceConnection.InnerConnection,该对象继承于IServiceConnection.Stub。</li>
</ul>
<h4 id="2-2-2-ServiceDispatcher"><a href="#2-2-2-ServiceDispatcher" class="headerlink" title="2.2.2 ServiceDispatcher"></a>2.2.2 ServiceDispatcher</h4><p>[-&gt;LoadedApk.java]</p>

<pre><code>
static final class ServiceDispatcher {
        private final ServiceDispatcher.InnerConnection mIServiceConnection;
        @UnsupportedAppUsage
        private final ServiceConnection mConnection;
        @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.P, trackingBug = 115609023)
        private final Context mContext;
        private final Handler mActivityThread;
        private final ServiceConnectionLeaked mLocation;
        private final int mFlags;

        private RuntimeException mUnbindLocation;

        private boolean mForgotten;

        private static class ConnectionInfo {
            IBinder binder;
            IBinder.DeathRecipient deathMonitor;
        }

        private static class InnerConnection extends IServiceConnection.Stub {
            @UnsupportedAppUsage
            final WeakReference&lt;LoadedApk.ServiceDispatcher&gt; mDispatcher;

            InnerConnection(LoadedApk.ServiceDispatcher sd) {
                //创建ServiceDispatcher弱引用
                mDispatcher = new WeakReference&lt;LoadedApk.ServiceDispatcher&gt;(sd);
            }

            public void connected(ComponentName name, IBinder service, boolean dead)
                    throws RemoteException {
                LoadedApk.ServiceDispatcher sd = mDispatcher.get();
                if (sd != null) {
                    sd.connected(name, service, dead);
                }
            }
        }

        private final ArrayMap&lt;ComponentName, ServiceDispatcher.ConnectionInfo&gt; mActiveConnections
            = new ArrayMap&lt;ComponentName, ServiceDispatcher.ConnectionInfo&gt;();

        @UnsupportedAppUsage
        ServiceDispatcher(ServiceConnection conn,
                Context context, Handler activityThread, int flags) {
            //新建InnerConnection
            mIServiceConnection = new InnerConnection(this);
            mConnection = conn;
            mContext = context;
            mActivityThread = activityThread;
            mLocation = new ServiceConnectionLeaked(null);
            mLocation.fillInStackTrace();
            mFlags = flags;
        }
        ....
         @UnsupportedAppUsage
        IServiceConnection getIServiceConnection() {
            //返回上面初始化的 mIServiceConnection
            return mIServiceConnection;
        }
        ....
   }</code></pre>

<p> getServiceDispatcher返回的是构造方法中的InnerConnection对象。</p>
<h4 id="2-2-3-AM-getService"><a href="#2-2-3-AM-getService" class="headerlink" title="2.2.3 AM.getService"></a>2.2.3 AM.getService</h4><p>[-&gt;ActivityManager.java]</p>

<pre><code>
public static IActivityManager getService() {
     return IActivityManagerSingleton.get();
 }
 
 @UnsupportedAppUsage
 private static final Singleton&lt;IActivityManager&gt; IActivityManagerSingleton =
      new Singleton&lt;IActivityManager&gt;() {
             @Override
             protected IActivityManager create() {
                 //获取IBinder
                 final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                 //见2.2.5节，获取代理
                 final IActivityManager am = IActivityManager.Stub.asInterface(b);
                 return am;
             }
  };</code></pre>

<p>在前面获取服务那篇文章中可以看出ServiceManager.getService(Context.ACTIVITY_SERVICE);等价于new BinderProxy(nativeData)，这里的b相当于BinderProxy对象。</p>
<h4 id="2-2-4-asInterface"><a href="#2-2-4-asInterface" class="headerlink" title="2.2.4 asInterface"></a>2.2.4 asInterface</h4><p>[-&gt;IActivityManager.java]</p>

<pre><code>
public static android.app.IActivityManager asInterface(android.os.IBinder obj) {
         if ((obj == null)) {
             return null;
         }
         android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
         //如果不是代理，这里不会走这里
         if (((iin != null) && (iin instanceof android.app.IActivityManager))) {
             return ((android.app.IActivityManager) iin);
         }
         return new android.app.IActivityManager.Stub.Proxy(obj);
   }</code></pre>

<h4 id="2-2-5-创建Proxy"><a href="#2-2-5-创建Proxy" class="headerlink" title="2.2.5 创建Proxy"></a>2.2.5 创建Proxy</h4><p>[-&gt;IActivityManager.java]</p>

<pre><code>
private static class Proxy implements android.app.IActivityManager {
     private android.os.IBinder mRemote;
      Proxy(android.os.IBinder remote) {
             mRemote = remote;
      } 
      ...
        @Override
  public int bindService(android.app.IApplicationThread caller, android.os.IBinder token, android.content.Intent service, String resolvedType, android.app.IServiceConnection connection, int flags, String callingPackage, int userId) throws android.os.RemoteException {
             android.os.Parcel _data = android.os.Parcel.obtain();
             android.os.Parcel _reply = android.os.Parcel.obtain();
             int _result;
             try {
                 _data.writeInterfaceToken(DESCRIPTOR);
                 //将ApplicationThread对象传递给systemserver
                 _data.writeStrongBinder((((caller != null)) ? (caller.asBinder()) : (null)));
                 _data.writeStrongBinder(token);
                 if ((service != null)) {
                     _data.writeInt(1);
                     service.writeToParcel(_data, 0);
                 } else {
                     _data.writeInt(0);
                 }
                 _data.writeString(resolvedType);
                 //将InnerConnection对象传递给systemserver
                 _data.writeStrongBinder((((connection != null)) ? (connection.asBinder()) : (null)));
                 _data.writeInt(flags);
                 _data.writeString(callingPackage);
                 _data.writeInt(userId);
                 //通过bind调用，进入到systemserver
                 mRemote.transact(Stub.TRANSACTION_bindService, _data, _reply, 0);
                 _reply.readException();
                 _result = _reply.readInt();
             } finally {
                 _reply.recycle();
                 _data.recycle();
             }
             return _result;
     }
     ...
 }</code></pre>

<p>这里mRemote为BinderProxy对象，通过mRemote向服务端传输数据。</p>
<p>writeStrongBinder、transact操作在注册服务那篇文章有详细的介绍，这里不再分析，向目标进程写入BINDER_WORK_TRANSACTION命令，下面进入服务端systemserver进程。</p>
<h2 id="三、system-server进程"><a href="#三、system-server进程" class="headerlink" title="三、system_server进程"></a>三、system_server进程</h2><p>在进程的启动那篇文章15.2节中，systemserver进程启动时会启动binder线程</p>
<h3 id="3-1-onZygoteInit"><a href="#3-1-onZygoteInit" class="headerlink" title="3.1 onZygoteInit()"></a>3.1 onZygoteInit()</h3><p>[-&gt;app_main.cpp]</p>

<pre><code>
virtual void onZygoteInit()
  {
      sp&lt;ProcessState&gt; proc = ProcessState::self();
      ALOGV("App process: starting thread pool./n");
      proc-&gt;startThreadPool(); //启动新的binder线程
  }</code></pre>

<h4 id="3-1-1-startThreadPool"><a href="#3-1-1-startThreadPool" class="headerlink" title="3.1.1  startThreadPool"></a>3.1.1  startThreadPool</h4><p>[-&gt;ProcessState.cpp]</p>

<pre><code>
void ProcessState::startThreadPool()
{
    AutoMutex _l(mLock);
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true);
    }
}</code></pre>

<h4 id="3-1-2-spawnPooledThread"><a href="#3-1-2-spawnPooledThread" class="headerlink" title="3.1.2  spawnPooledThread"></a>3.1.2  spawnPooledThread</h4><p>[-&gt;ProcessState.cpp]</p>

<pre><code>
void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName();
        ALOGV("Spawning new pooled thread, name=%s/n", name.string());
        //创建线程池
        sp&lt;Thread&gt; t = new PoolThread(isMain);
        //执行threadLoop方法
        t-&gt;run(name.string());
    }
}</code></pre>

<h4 id="3-1-3-new-PoolThread"><a href="#3-1-3-new-PoolThread" class="headerlink" title="3.1.3  new PoolThread"></a>3.1.3  new PoolThread</h4><p>[-&gt;ProcessState.cpp]</p>

<pre><code>
class PoolThread : public Thread
{
    public:
    explicit PoolThread(bool isMain)
        : mIsMain(isMain)
    {
    }
    
    protected:
    virtual bool threadLoop()
    {
        //见3.2节
        IPCThreadState::self()-&gt;joinThreadPool(mIsMain);
        return false;
    }
    
    const bool mIsMain;
};</code></pre>

<h3 id="3-2-joinThreadPool"><a href="#3-2-joinThreadPool" class="headerlink" title="3.2 joinThreadPool"></a>3.2 joinThreadPool</h3><p>[-&gt;IPCThreadState.cpp]</p>

<pre><code>
void IPCThreadState::joinThreadPool(bool isMain)
{
    LOG_THREADPOOL("**** THREAD %p (PID %d) IS JOINING THE THREAD POOL/n", (void*)pthread_self(), getpid());

    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);

    status_t result;
    do {
        //处理对象引用
        processPendingDerefs();
        // now get the next command to be processed, waiting if necessary
        //获取并执行命令，见2.3节
        result = getAndExecuteCommand();

        if (result &lt; NO_ERROR && result != TIMED_OUT && result != -ECONNREFUSED && result != -EBADF)        {
            ALOGE("getAndExecuteCommand(fd=%d) returned unexpected error %d, aborting",
                  mProcess-&gt;mDriverFD, result);
            abort();
        }

        // Let this thread exit the thread pool if it is no longer
        // needed and it is not the main process thread.
        //对于binder非主线程，不再使用则退出
        if(result == TIMED_OUT && !isMain) {
            break;
        }
    } while (result != -ECONNREFUSED && result != -EBADF);

    LOG_THREADPOOL("**** THREAD %p (PID %d) IS LEAVING THE THREAD POOL err=%d/n",
        (void*)pthread_self(), getpid(), result);

    mOut.writeInt32(BC_EXIT_LOOPER);
    //和binder驱动交互
    talkWithDriver(false);
}</code></pre>

<p>system_server进程，通过这个while循环来获取并执行binder命令。</p>
<h3 id="3-3-IPC-getAndExecuteCommand"><a href="#3-3-IPC-getAndExecuteCommand" class="headerlink" title="3.3 IPC.getAndExecuteCommand"></a>3.3 IPC.getAndExecuteCommand</h3><p>[-&gt;IPCThreadState.cpp]</p>

<pre><code>
status_t IPCThreadState::getAndExecuteCommand()
{
    status_t result;
    int32_t cmd;
    //和Binder Driver交互
    result = talkWithDriver();
    if (result &gt;= NO_ERROR) {
        size_t IN = mIn.dataAvail();
        if (IN &lt; sizeof(int32_t)) return result;
        //读取命令
        cmd = mIn.readInt32();
        IF_LOG_COMMANDS() {
            alog &lt;&lt; "Processing top-level Command: "
                 &lt;&lt; getReturnString(cmd) &lt;&lt; endl;
        }

        pthread_mutex_lock(&mProcess-&gt;mThreadCountLock);
        mProcess-&gt;mExecutingThreadsCount++;
        if (mProcess-&gt;mExecutingThreadsCount &gt;= mProcess-&gt;mMaxThreads &&
                mProcess-&gt;mStarvationStartTimeMs == 0) {
            mProcess-&gt;mStarvationStartTimeMs = uptimeMillis();
        }
        pthread_mutex_unlock(&mProcess-&gt;mThreadCountLock);
        //见3.4节
        result = executeCommand(cmd);

        pthread_mutex_lock(&mProcess-&gt;mThreadCountLock);
        mProcess-&gt;mExecutingThreadsCount--;
        if (mProcess-&gt;mExecutingThreadsCount &lt; mProcess-&gt;mMaxThreads &&
                mProcess-&gt;mStarvationStartTimeMs != 0) {
            int64_t starvationTimeMs = uptimeMillis() - mProcess-&gt;mStarvationStartTimeMs;
            if (starvationTimeMs &gt; 100) {
                ALOGE("binder thread pool (%zu threads) starved for %" PRId64 " ms",
                      mProcess-&gt;mMaxThreads, starvationTimeMs);
            }
            mProcess-&gt;mStarvationStartTimeMs = 0;
        }
        pthread_cond_broadcast(&mProcess-&gt;mThreadCountDecrement);
        pthread_mutex_unlock(&mProcess-&gt;mThreadCountLock);
    }

    return result;
}</code></pre>

<p>这里system_server的binder线程空闲，停在binder_thread_read方法来处理进程/线程的事务。前面收到BINDER_WORK_TRANSACTION命令，经过binder_thread_read后生成命令cmd=BR_TRANSACTION，再将cmd和数据写回用户空间。</p>
<h3 id="3-4-IPC-executeCommand"><a href="#3-4-IPC-executeCommand" class="headerlink" title="3.4 IPC.executeCommand"></a>3.4 IPC.executeCommand</h3><p>[-&gt;IPCThreadState.cpp]</p>

<pre><code>
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    BBinder* obj;
    RefBase::weakref_type* refs;
    status_t result = NO_ERROR;

    switch ((uint32_t)cmd) {
     ...
     case BR_TRANSACTION:
        {
            binder_transaction_data tr;
            result = mIn.read(&tr, sizeof(tr));
            ALOG_ASSERT(result == NO_ERROR,
                "Not enough command data for brTRANSACTION");
            if (result != NO_ERROR) break;

            Parcel buffer;
            //当buffer对象回收时，通过调用freeBuffer来回收内存
            buffer.ipcSetDataReference(
                reinterpret_cast&lt;const uint8_t*&gt;(tr.data.ptr.buffer),
                tr.data_size,
                reinterpret_cast&lt;const binder_size_t*&gt;(tr.data.ptr.offsets),
                tr.offsets_size/sizeof(binder_size_t), freeBuffer, this);

            const pid_t origPid = mCallingPid;
            const uid_t origUid = mCallingUid;
            const int32_t origStrictModePolicy = mStrictModePolicy;
            const int32_t origTransactionBinderFlags = mLastTransactionBinderFlags;
            //设置调用者的pid、uid
            mCallingPid = tr.sender_pid;
            mCallingUid = tr.sender_euid;
            mLastTransactionBinderFlags = tr.flags;

            //ALOGI("&gt;&gt;&gt;&gt; TRANSACT from pid %d uid %d/n", mCallingPid, mCallingUid);

            Parcel reply;
            status_t error;
            IF_LOG_TRANSACTIONS() {
                TextOutput::Bundle _b(alog);
                alog &lt;&lt; "BR_TRANSACTION thr " &lt;&lt; (void*)pthread_self()
                    &lt;&lt; " / obj " &lt;&lt; tr.target.ptr &lt;&lt; " / code "
                    &lt;&lt; TypeCode(tr.code) &lt;&lt; ": " &lt;&lt; indent &lt;&lt; buffer
                    &lt;&lt; dedent &lt;&lt; endl
                    &lt;&lt; "Data addr = "
                    &lt;&lt; reinterpret_cast&lt;const uint8_t*&gt;(tr.data.ptr.buffer)
                    &lt;&lt; ", offsets addr="
                    &lt;&lt; reinterpret_cast&lt;const size_t*&gt;(tr.data.ptr.offsets) &lt;&lt; endl;
            }
            if (tr.target.ptr) {
                // We only have a weak reference on the target object, so we must first try to
                // safely acquire a strong reference before doing anything else with it.
                // 尝试通过弱引用获取强引用
                if (reinterpret_cast&lt;RefBase::weakref_type*&gt;(
                        tr.target.ptr)-&gt;attemptIncStrong(this)) {
                    //tr.cookie存放的是BBinder子类的JavaBBinder    
                    error = reinterpret_cast&lt;BBinder*&gt;(tr.cookie)-&gt;transact(tr.code, buffer,
                            &reply, tr.flags);
                    reinterpret_cast&lt;BBinder*&gt;(tr.cookie)-&gt;decStrong(this);
                } else {
                    error = UNKNOWN_TRANSACTION;
                }

            } else {
                error = the_context_object-&gt;transact(tr.code, buffer, &reply, tr.flags);
            }

            //ALOGI("&lt;&lt;&lt;&lt; TRANSACT from pid %d restore pid %d uid %d/n",
            //     mCallingPid, origPid, origUid);

            if ((tr.flags & TF_ONE_WAY) == 0) {
                //对于非oneway，需要通过reply通信过程，向binder驱动发送BC_REPLY命令
                LOG_ONEWAY("Sending reply to %d!", mCallingPid);
                if (error &lt; NO_ERROR) reply.setError(error);
                sendReply(reply, 0);
            } else {
                LOG_ONEWAY("NOT sending reply to %d!", mCallingPid);
            }
            //恢复pid和uid信息
            mCallingPid = origPid;
            mCallingUid = origUid;
            mStrictModePolicy = origStrictModePolicy;
            mLastTransactionBinderFlags = origTransactionBinderFlags;

            IF_LOG_TRANSACTIONS() {
                TextOutput::Bundle _b(alog);
                alog &lt;&lt; "BC_REPLY thr " &lt;&lt; (void*)pthread_self() &lt;&lt; " / obj "
                    &lt;&lt; tr.target.ptr &lt;&lt; ": " &lt;&lt; indent &lt;&lt; reply &lt;&lt; dedent &lt;&lt; endl;
            }

        }
        break;
        ...
    }

    if (result != NO_ERROR) {
        mLastError = result;
    }

    return result;
}</code></pre>

<ul>
<li><p>对于oneway的情况，执行完本次transact则全部结束</p>
</li>
<li><p>对于非oneway的情况，需要reply的通信过程，则向Binder驱动发送RC_REPLY命令</p>
</li>
</ul>
<h4 id="3-4-1-ipcSetDataReference"><a href="#3-4-1-ipcSetDataReference" class="headerlink" title="3.4.1 ipcSetDataReference"></a>3.4.1 ipcSetDataReference</h4><p>[-&gt;Parcel.cpp]</p>

<pre><code>
void Parcel::ipcSetDataReference(const uint8_t* data, size_t dataSize,
    const binder_size_t* objects, size_t objectsCount, release_func relFunc, void* relCookie)
{
    binder_size_t minOffset = 0;
    //见3.4.2小节
    freeDataNoInit();
    mError = NO_ERROR;
    mData = const_cast&lt;uint8_t*&gt;(data);
    mDataSize = mDataCapacity = dataSize;
    //ALOGI("setDataReference Setting data size of %p to %lu (pid=%d)", this, mDataSize, getpid());
    mDataPos = 0;
    ALOGV("setDataReference Setting data pos of %p to %zu", this, mDataPos);
    mObjects = const_cast&lt;binder_size_t*&gt;(objects);
    mObjectsSize = mObjectsCapacity = objectsCount;
    mNextObjectHint = 0;
    mObjectsSorted = false;
    mOwner = relFunc;
    mOwnerCookie = relCookie;
    for (size_t i = 0; i &lt; mObjectsSize; i++) {
        binder_size_t offset = mObjects[i];
        if (offset &lt; minOffset) {
            ALOGE("%s: bad object offset %" PRIu64 " &lt; %" PRIu64 "/n",
                  __func__, (uint64_t)offset, (uint64_t)minOffset);
            mObjectsSize = 0;
            break;
        }
        minOffset = offset + sizeof(flat_binder_object);
    }
    scanForFds();
}</code></pre>

<p>Parcel成员变量说明：</p>
<p>mData：parcel数据起始地址</p>
<p>mDataSize：parcel数据大小</p>
<p>mObjects：flat_binder_object地址偏移量</p>
<p>mObjectsSize：parcel中flat_binder_object个数</p>
<p>mOwner：释放函数freeBuffer</p>
<p>mOwnerCookie：释放函数所需信息</p>
<h4 id="3-4-2-freeDataNoInit"><a href="#3-4-2-freeDataNoInit" class="headerlink" title="3.4.2 freeDataNoInit"></a>3.4.2 freeDataNoInit</h4><p>[-&gt;Parcel.cpp]</p>

<pre><code>
void Parcel::freeDataNoInit()
{
    if (mOwner) {
        LOG_ALLOC("Parcel %p: freeing other owner data", this);
        //ALOGI("Freeing data ref of %p (pid=%d)", this, getpid());
        mOwner(this, mData, mDataSize, mObjects, mObjectsSize, mOwnerCookie);
    } else {
        //为空进入这里
        LOG_ALLOC("Parcel %p: freeing allocated data", this);
        releaseObjects();
        if (mData) {
            LOG_ALLOC("Parcel %p: freeing with %zu capacity", this, mDataCapacity);
            pthread_mutex_lock(&gParcelGlobalAllocSizeLock);
            if (mDataCapacity &lt;= gParcelGlobalAllocSize) {
              gParcelGlobalAllocSize = gParcelGlobalAllocSize - mDataCapacity;
            } else {
              gParcelGlobalAllocSize = 0;
            }
            if (gParcelGlobalAllocCount &gt; 0) {
              gParcelGlobalAllocCount--;
            }
            pthread_mutex_unlock(&gParcelGlobalAllocSizeLock);
            free(mData);
        }
        if (mObjects) free(mObjects);
    }
}</code></pre>

<h4 id="3-4-3-releaseObjects"><a href="#3-4-3-releaseObjects" class="headerlink" title="3.4.3 releaseObjects"></a>3.4.3 releaseObjects</h4><p>[-&gt;Parcel.cpp]</p>

<pre><code>
void Parcel::releaseObjects()
{
    const sp&lt;ProcessState&gt; proc(ProcessState::self());
    size_t i = mObjectsSize;
    uint8_t* const data = mData;
    binder_size_t* const objects = mObjects;
    while (i &gt; 0) {
        i--;
        const flat_binder_object* flat
            = reinterpret_cast&lt;flat_binder_object*&gt;(data+objects[i]);
        //见3.4.3小节
        release_object(proc, *flat, this, &mOpenAshmemSize);
    }
}</code></pre>

<h4 id="3-4-4-release-object"><a href="#3-4-4-release-object" class="headerlink" title="3.4.4 release_object"></a>3.4.4 release_object</h4><p>[-&gt;Parcel.cpp]</p>

<pre><code>
static void release_object(const sp&lt;ProcessState&gt;& proc,
    const flat_binder_object& obj, const void* who, size_t* outAshmemSize)
{
    switch (obj.hdr.type) {
        case BINDER_TYPE_BINDER:
            if (obj.binder) {
                LOG_REFS("Parcel %p releasing reference on local %p", who, obj.cookie);
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
                LOG_REFS("Parcel %p releasing reference on remote %p", who, b.get());
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
            if (obj.cookie != 0) { // owned
                if ((outAshmemSize != NULL) && ashmem_valid(obj.handle)) {
                    int size = ashmem_get_size_region(obj.handle);
                    if (size &gt; 0) {
                        *outAshmemSize -= size;
                    }
                }

                close(obj.handle);
            }
            return;
        }
    }

    ALOGE("Invalid object type 0x%08x", obj.hdr.type);
}</code></pre>

<p>根据flat_binder_object的类型，来减少相应的强弱引用。</p>
<h4 id="3-4-5-Parcel"><a href="#3-4-5-Parcel" class="headerlink" title="3.4.5 ~Parcel"></a>3.4.5 ~Parcel</h4><p>[-&gt;Parcel.cpp]</p>

<pre><code>
Parcel::~Parcel()
{
    freeDataNoInit();
    LOG_ALLOC("Parcel %p: destroyed", this);
}
void Parcel::freeDataNoInit()
{
    if (mOwner) {
        LOG_ALLOC("Parcel %p: freeing other owner data", this);
        //ALOGI("Freeing data ref of %p (pid=%d)", this, getpid());
        mOwner(this, mData, mDataSize, mObjects, mObjectsSize, mOwnerCookie);
    } else {
    ...
    }
 }</code></pre>

<p>执行完executeCommand方法后，会释放局部变量Parcelbuffer，则会析构Parcel。接下来，则会执行freeBuffer方法</p>
<h4 id="3-4-6-freeBuffer"><a href="#3-4-6-freeBuffer" class="headerlink" title="3.4.6  freeBuffer"></a>3.4.6  freeBuffer</h4><p>[-&gt;IPCThreadState.cpp]</p>

<pre><code>
void IPCThreadState::freeBuffer(Parcel* parcel, const uint8_t* data,
                                size_t /*dataSize*/,
                                const binder_size_t* /*objects*/,
                                size_t /*objectsSize*/, void* /*cookie*/)
{
    //ALOGI("Freeing parcel %p", &parcel);
    IF_LOG_COMMANDS() {
        alog &lt;&lt; "Writing BC_FREE_BUFFER for " &lt;&lt; data &lt;&lt; endl;
    }
    ALOG_ASSERT(data != NULL, "Called with NULL data");
    if (parcel != NULL) parcel-&gt;closeFileDescriptors();
    IPCThreadState* state = self();
    state-&gt;mOut.writeInt32(BC_FREE_BUFFER);
    state-&gt;mOut.writePointer((uintptr_t)data);
}</code></pre>

<p>向binder驱动写入BC_FREE_BUFFER命令。</p>
<h3 id="3-5-BBbinder-transact"><a href="#3-5-BBbinder-transact" class="headerlink" title="3.5 BBbinder.transact"></a>3.5 BBbinder.transact</h3><p>[-&gt;binder/Binder.cpp]</p>

<pre><code>
status_t BBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    data.setDataPosition(0);

    status_t err = NO_ERROR;
    switch (code) {
        case PING_TRANSACTION:
            reply-&gt;writeInt32(pingBinder());
            break;
        default:
            //见3.5.1节
            err = onTransact(code, data, reply, flags);
            break;
    }

    if (reply != NULL) {
        reply-&gt;setDataPosition(0);
    }

    return err;
}
</code></pre>

<h4 id="3-5-1-onTransact"><a href="#3-5-1-onTransact" class="headerlink" title="3.5.1 onTransact"></a>3.5.1 onTransact</h4><p>[-&gt;android_util_Binder.cpp]</p>

<pre><code>
status_t onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0) override
    {
        JNIEnv* env = javavm_to_jnienv(mVM);

        ALOGV("onTransact() on %p calling object %p in env %p vm %p/n", this, mObject, env, mVM);

        IPCThreadState* thread_state = IPCThreadState::self();
        const int32_t strict_policy_before = thread_state-&gt;getStrictModePolicy();

        //printf("Transact from %p to Java code sending: ", this);
        //data.print();
        //printf("/n");
        //调用Binder.execTransact方法
        jboolean res = env-&gt;CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
            code, reinterpret_cast&lt;jlong&gt;(&data), reinterpret_cast&lt;jlong&gt;(reply), flags);

        if (env-&gt;ExceptionCheck()) {
            //发生异常，清理JNI本地引用
            ScopedLocalRef&lt;jthrowable&gt; excep(env, env-&gt;ExceptionOccurred());
            report_exception(env, excep.get(),
                "*** Uncaught remote exception!  "
                "(Exceptions are not yet supported across processes.)");
            res = JNI_FALSE;
        }

        // Check if the strict mode state changed while processing the
        // call.  The Binder state will be restored by the underlying
        // Binder system in IPCThreadState, however we need to take care
        // of the parallel Java state as well.
        if (thread_state-&gt;getStrictModePolicy() != strict_policy_before) {
            set_dalvik_blockguard_policy(env, strict_policy_before);
        }

        if (env-&gt;ExceptionCheck()) {
            //发生异常，清理JNI本地引用
            ScopedLocalRef&lt;jthrowable&gt; excep(env, env-&gt;ExceptionOccurred());
            report_exception(env, excep.get(),
                "*** Uncaught exception in onBinderStrictModePolicyChange");
        }

        // Need to always call through the native implementation of
        // SYSPROPS_TRANSACTION.
        if (code == SYSPROPS_TRANSACTION) {
            BBinder::onTransact(code, data, reply, flags);
        }

        //aout &lt;&lt; "onTransact to Java code; result=" &lt;&lt; res &lt;&lt; endl
        //    &lt;&lt; "Transact from " &lt;&lt; this &lt;&lt; " to Java code returning "
        //    &lt;&lt; reply &lt;&lt; ": " &lt;&lt; *reply &lt;&lt; endl;
        return res != JNI_FALSE ? NO_ERROR : UNKNOWN_TRANSACTION;
    }</code></pre>

<p>mObject是在服务注册addService过程中，会调用WriteStrongBinder方法，将Binder传入JavaBBinder构造函数的参数，最终赋值给mObject。</p>
<p>gBinderOffsets在int_register_android_os_Binder函数中进行的初始化。</p>
<p>这样通过JNI的方式，从C++回到Java代码，进入IActivityManager.execTransact方法。</p>
<p>[-&gt;android_util_Binder.cpp]</p>

<pre><code>
static int int_register_android_os_Binder(JNIEnv* env)
{
    jclass clazz = FindClassOrDie(env, kBinderPathName);

    gBinderOffsets.mClass = MakeGlobalRefOrDie(env, clazz);
    gBinderOffsets.mExecTransact = GetMethodIDOrDie(env, clazz, "execTransact", "(IJJI)Z");
    gBinderOffsets.mGetInterfaceDescriptor = GetMethodIDOrDie(env, clazz, "getInterfaceDescriptor",
        "()Ljava/lang/String;");
    gBinderOffsets.mObject = GetFieldIDOrDie(env, clazz, "mObject", "J");

    return RegisterMethodsOrDie(
        env, kBinderPathName,
        gBinderMethods, NELEM(gBinderMethods));
}
</code></pre>

<h4 id="3-5-2-execTransact"><a href="#3-5-2-execTransact" class="headerlink" title="3.5.2 execTransact"></a>3.5.2 execTransact</h4><p>[-&gt;Binder.java]</p>

<pre><code>
// Entry point from android_util_Binder.cpp's onTransact
 @UnsupportedAppUsage
 private boolean execTransact(int code, long dataObj, long replyObj,
         int flags) {
     BinderCallsStats binderCallsStats = BinderCallsStats.getInstance();
     BinderCallsStats.CallSession callSession = binderCallsStats.callStarted(this, code);
     Parcel data = Parcel.obtain(dataObj);
     Parcel reply = Parcel.obtain(replyObj);
     // theoretically, we should call transact, which will call onTransact,
     // but all that does is rewind it, and we just got these from an IPC,
     // so we'll just call it directly.
     boolean res;
     // Log any exceptions as warnings, don't silently suppress them.
     // If the call was FLAG_ONEWAY then these exceptions disappear into the ether.
     final boolean tracingEnabled = Binder.isTracingEnabled();
     try {
         if (tracingEnabled) {
             Trace.traceBegin(Trace.TRACE_TAG_ALWAYS, getClass().getName() + ":" + code);
         }
         //调用子类IActivityManager的execTransact方法
         res = onTransact(code, data, reply, flags);
     } catch (RemoteException|RuntimeException e) {
         if (LOG_RUNTIME_EXCEPTION) {
             Log.w(TAG, "Caught a RuntimeException from the binder stub implementation.", e);
         }
         if ((flags & FLAG_ONEWAY) != 0) {
             if (e instanceof RemoteException) {
                 Log.w(TAG, "Binder call failed.", e);
             } else {
                 Log.w(TAG, "Caught a RuntimeException from the binder stub implementation.", e);
             }
         } else {
             //非oneway方式，则会将异常写回reply
             reply.setDataPosition(0);
             reply.writeException(e);
         }
         res = true;
     } finally {
         if (tracingEnabled) {
             Trace.traceEnd(Trace.TRACE_TAG_ALWAYS);
         }
     }
     checkParcel(this, code, reply, "Unreasonably large binder reply buffer");
     reply.recycle();
     data.recycle();

     // Just in case -- we are done with the IPC, so there should be no more strict
     // mode violations that have gathered for this thread.  Either they have been
     // parceled and are now in transport off to the caller, or we are returning back
     // to the main transaction loop to wait for another incoming transaction.  Either
     // way, strict mode begone!
     StrictMode.clearGatheredViolations();
     binderCallsStats.callEnded(callSession);

     return res;
 }</code></pre>

<h3 id="3-6-IActivityManager-onTransact"><a href="#3-6-IActivityManager-onTransact" class="headerlink" title="3.6 IActivityManager.onTransact"></a>3.6 IActivityManager.onTransact</h3><p>[-&gt;IActivityManager.java]</p>

<pre><code>
public static abstract class Stub extends android.os.Binder implements android.app.IActivityManager {
  ...
 @Override
 public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
        String descriptor = DESCRIPTOR;
        switch (code) {
           ...
            //在2.2.5小节传过来的command
           case TRANSACTION_bindService: {
               return this.onTransact$bindService$(data, reply);
           }
           ...
        }
  }
}</code></pre>

<h3 id="3-7-onTransact-bindService"><a href="#3-7-onTransact-bindService" class="headerlink" title="3.7 onTransact$bindService$"></a>3.7 onTransact$bindService$</h3><p>[-&gt;IActivityManager.java]</p>

<pre><code>
private boolean onTransact$bindService$(android.os.Parcel data, android.os.Parcel reply) throws  android.os.RemoteException {
           data.enforceInterface(DESCRIPTOR);
           android.app.IApplicationThread _arg0;
            //获取ApplicationThread的代理对象
           _arg0 = android.app.IApplicationThread.Stub.asInterface(data.readStrongBinder());
           android.os.IBinder _arg1;
           _arg1 = data.readStrongBinder();
           android.content.Intent _arg2;
           if ((0 != data.readInt())) {
               _arg2 = android.content.Intent.CREATOR.createFromParcel(data);
           } else {
               _arg2 = null;
           }
           String _arg3;
           _arg3 = data.readString();
           android.app.IServiceConnection _arg4;
           //获取InnerConnection的代理对象
           _arg4 = android.app.IServiceConnection.Stub.asInterface(data.readStrongBinder());
           int _arg5;
           _arg5 = data.readInt();
           String _arg6;
           _arg6 = data.readString();
           int _arg7;
           _arg7 = data.readInt();
           //见3.3节，调用AMS.bindService
           int _result = this.bindService(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5, _arg6, _arg7);
           reply.writeNoException();
           reply.writeInt(_result);
           return true;
       }</code></pre>

<p>IPC::waitForResponse对于非oneway方式，还在等待system_server这边的响应，只有收到BR_REPLY或者BR_DEAD_REPLY等其他出错的情况下，才会退出waitForResponse。</p>
<p>当bindService完成后，还需要将bindservice完成的回应消息告诉发起端的进程。在3.4节中IPC.executeCommand过程中处理完成BR_TRANSACTION命令的同时，还会通过 sendReply(reply, 0);向Binder Driver发送BC_RELY消息。这里Rely流程不再详细介绍，还是和进入之前相应的流程类似。</p>
<h3 id="3-8-AMS-bindService"><a href="#3-8-AMS-bindService" class="headerlink" title="3.8 AMS.bindService"></a>3.8 AMS.bindService</h3><p>[-&gt;ActivityManagerService.java]</p>

<pre><code>
public int bindService(IApplicationThread caller, IBinder token, Intent service,
           String resolvedType, IServiceConnection connection, int flags, String callingPackage,
           int userId) throws TransactionTooLargeException {
       enforceNotIsolatedCaller("bindService");

       // Refuse possible leaked file descriptors
       if (service != null && service.hasFileDescriptors() == true) {
           throw new IllegalArgumentException("File descriptors passed in Intent");
       }

       if (callingPackage == null) {
           throw new IllegalArgumentException("callingPackage cannot be null");
       }

       synchronized(this) {
           //见3.9节
           return mServices.bindServiceLocked(caller, token, service,
                   resolvedType, connection, flags, callingPackage, userId);
       }
   }</code></pre>

<h3 id="3-9-AS-bindServiceLocked"><a href="#3-9-AS-bindServiceLocked" class="headerlink" title="3.9 AS.bindServiceLocked"></a>3.9 AS.bindServiceLocked</h3><p>[-&gt;ActiveServices.java]</p>

<pre><code>
int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
           String resolvedType, final IServiceConnection connection, int flags,
           String callingPackage, final int userId) throws TransactionTooLargeException {
       if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "bindService: " + service
               + " type=" + resolvedType + " conn=" + connection.asBinder()
               + " flags=0x" + Integer.toHexString(flags));
       //查询发起端所对应的进程记录
       final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
       if (callerApp == null) {
           throw new SecurityException(
                   "Unable to find app for caller " + caller
                   + " (pid=" + Binder.getCallingPid()
                   + ") when binding service " + service);
       }

       ActivityRecord activity = null;
       //token不为空，代表发起方具有activity的上下文
       if (token != null) {
           activity = ActivityRecord.isInStackLocked(token);
           if (activity == null) {
               Slog.w(TAG, "Binding with unknown activity: " + token);
               return 0;  //存在token,却找不到activity为空，则直接返回
           }
       }

       int clientLabel = 0;
       PendingIntent clientIntent = null;
       final boolean isCallerSystem = callerApp.info.uid == Process.SYSTEM_UID;
       //发起端是system进程
       if (isCallerSystem) {
           // Hacky kind of thing -- allow system stuff to tell us
           // what they are, so we can report this elsewhere for
           // others to know why certain services are running.
           service.setDefusable(true);
           clientIntent = service.getParcelableExtra(Intent.EXTRA_CLIENT_INTENT);
           if (clientIntent != null) {
               clientLabel = service.getIntExtra(Intent.EXTRA_CLIENT_LABEL, 0);
               if (clientLabel != 0) {
                   // There are no useful extras in the intent, trash them.
                   // System code calling with this stuff just needs to know
                   // this will happen.
                   service = service.cloneFilter();
               }
           }
       }

       if ((flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
           mAm.enforceCallingPermission(android.Manifest.permission.MANAGE_ACTIVITY_STACKS,
                   "BIND_TREAT_LIKE_ACTIVITY");
       }
       //不在白名单则抛出异常
       if ((flags & Context.BIND_ALLOW_WHITELIST_MANAGEMENT) != 0 && !isCallerSystem) {
           throw new SecurityException(
                   "Non-system caller " + caller + " (pid=" + Binder.getCallingPid()
                   + ") set BIND_ALLOW_WHITELIST_MANAGEMENT when binding service " + service);
       }

       if ((flags & Context.BIND_ALLOW_INSTANT) != 0 && !isCallerSystem) {
           throw new SecurityException(
                   "Non-system caller " + caller + " (pid=" + Binder.getCallingPid()
                           + ") set BIND_ALLOW_INSTANT when binding service " + service);
       }
       //根据发送端所在进程的SchedGroup来决定是否为前台服务
       final boolean callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND;
       final boolean isBindExternal = (flags & Context.BIND_EXTERNAL_SERVICE) != 0;
       final boolean allowInstant = (flags & Context.BIND_ALLOW_INSTANT) != 0;
       //根据用户端传递进来的intent来检索对应的服务
       ServiceLookupResult res =
           retrieveServiceLocked(service, resolvedType, callingPackage, Binder.getCallingPid(),
                   Binder.getCallingUid(), userId, true, callerFg, isBindExternal, allowInstant);
       if (res == null) {
           return 0;
       }
       if (res.record == null) {
           return -1;
       }
       //查询到相应的服务
       ServiceRecord s = res.record;

       boolean permissionsReviewRequired = false;

       // If permissions need a review before any of the app components can run,
       // we schedule binding to the service but do not start its process, then
       // we launch a review activity to which is passed a callback to invoke
       // when done to start the bound service's process to completing the binding.
       //如果需要权限，启动activity通过callback启动服务进程
       if (mAm.mPermissionReviewRequired) {
           if (mAm.getPackageManagerInternalLocked().isPermissionsReviewRequired(
                   s.packageName, s.userId)) {

               permissionsReviewRequired = true;

               // Show a permission review UI only for binding from a foreground app
               if (!callerFg) {
                   Slog.w(TAG, "u" + s.userId + " Binding to a service in package"
                           + s.packageName + " requires a permissions review");
                   return 0;
               }

               final ServiceRecord serviceRecord = s;
               final Intent serviceIntent = service;

               RemoteCallback callback = new RemoteCallback(
                       new RemoteCallback.OnResultListener() {
                   @Override
                   public void onResult(Bundle result) {
                       synchronized(mAm) {
                           final long identity = Binder.clearCallingIdentity();
                           try {
                               if (!mPendingServices.contains(serviceRecord)) {
                                   return;
                               }
                               // If there is still a pending record, then the service
                               // binding request is still valid, so hook them up. We
                               // proceed only if the caller cleared the review requirement
                               // otherwise we unbind because the user didn't approve.
                               if (!mAm.getPackageManagerInternalLocked()
                                       .isPermissionsReviewRequired(
                                               serviceRecord.packageName,
                                               serviceRecord.userId)) {
                                   try {
                                       bringUpServiceLocked(serviceRecord,
                                               serviceIntent.getFlags(),
                                               callerFg, false, false);
                                   } catch (RemoteException e) {
                                       /* ignore - local call */
                                   }
                               } else {
                                   unbindServiceLocked(connection);
                               }
                           } finally {
                               Binder.restoreCallingIdentity(identity);
                           }
                       }
                   }
               });

               final Intent intent = new Intent(Intent.ACTION_REVIEW_PERMISSIONS);
               intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK
                       | Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS);
               intent.putExtra(Intent.EXTRA_PACKAGE_NAME, s.packageName);
               intent.putExtra(Intent.EXTRA_REMOTE_CALLBACK, callback);

               if (DEBUG_PERMISSIONS_REVIEW) {
                   Slog.i(TAG, "u" + s.userId + " Launching permission review for package "
                           + s.packageName);
               }

               mAm.mHandler.post(new Runnable() {
                   @Override
                   public void run() {
                       //启动activity
                       mAm.mContext.startActivityAsUser(intent, new UserHandle(userId));
                   }
               });
           }
       }

       final long origId = Binder.clearCallingIdentity();

       try {
           if (unscheduleServiceRestartLocked(s, callerApp.info.uid, false)) {
               if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "BIND SERVICE WHILE RESTART PENDING: "
                       + s);
           }

           if ((flags&Context.BIND_AUTO_CREATE) != 0) {
               s.lastActivity = SystemClock.uptimeMillis();
               if (!s.hasAutoCreateConnections()) {
                   // This is the first binding, let the tracker know.
                   ServiceState stracker = s.getTracker();
                   if (stracker != null) {
                       stracker.setBound(true, mAm.mProcessStats.getMemFactorLocked(),
                               s.lastActivity);
                   }
               }
           }

           mAm.startAssociationLocked(callerApp.uid, callerApp.processName, callerApp.curProcState,
                   s.appInfo.uid, s.name, s.processName);
           // Once the apps have become associated, if one of them is caller is ephemeral
           // the target app should now be able to see the calling app
           mAm.grantEphemeralAccessLocked(callerApp.userId, service,
                   s.appInfo.uid, UserHandle.getAppId(callerApp.uid));

           AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
           //创建对象ConnectionRecord，此处connection来自发起方
           ConnectionRecord c = new ConnectionRecord(b, activity,
                   connection, flags, clientLabel, clientIntent);

           IBinder binder = connection.asBinder();
           ArrayList&lt;ConnectionRecord&gt; clist = s.connections.get(binder);
           if (clist == null) {
               clist = new ArrayList&lt;ConnectionRecord&gt;();
               s.connections.put(binder, clist);
           }
           //clist是ServiceRecord.connection的成员变量
           clist.add(c);  
           b.connections.add(c);//b是指AppBindRecord
           if (activity != null) {
               if (activity.connections == null) {
                   activity.connections = new HashSet&lt;ConnectionRecord&gt;();
               }
               activity.connections.add(c);
           }
           b.client.connections.add(c);
           if ((c.flags&Context.BIND_ABOVE_CLIENT) != 0) {
               b.client.hasAboveClient = true;
           }
           if ((c.flags&Context.BIND_ALLOW_WHITELIST_MANAGEMENT) != 0) {
               s.whitelistManager = true;
           }
           if (s.app != null) {
               updateServiceClientActivitiesLocked(s.app, c, true);
           }
           clist = mServiceConnections.get(binder);
           if (clist == null) {
               clist = new ArrayList&lt;ConnectionRecord&gt;();
               mServiceConnections.put(binder, clist);
           }
           clist.add(c);

           if ((flags&Context.BIND_AUTO_CREATE) != 0) {
               s.lastActivity = SystemClock.uptimeMillis();
               //启动service，这个过程和startService过程一致，见3.10节
               if (bringUpServiceLocked(s, service.getFlags(), callerFg, false,
                       permissionsReviewRequired) != null) {
                   return 0;
               }
           }

           if (s.app != null) {
               if ((flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
                   s.app.treatLikeActivity = true;
               }
               if (s.whitelistManager) {
                   s.app.whitelistManager = true;
               }
               // This could have made the service more important.
               //更新service所在进程的优先级
               mAm.updateLruProcessLocked(s.app, s.app.hasClientActivities
                       || s.app.treatLikeActivity, b.client);
               mAm.updateOomAdjLocked(s.app, true);
           }

           if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Bind " + s + " with " + b
                   + ": received=" + b.intent.received
                   + " apps=" + b.intent.apps.size()
                   + " doRebind=" + b.intent.doRebind);

           if (s.app != null && b.intent.received) {
               // Service is already running, so we can immediately
               // publish the connection.
               try {
                   //service已经正在运行，则调用InnerConnection的代理对象，见7.1小节
                   c.conn.connected(s.name, b.intent.binder, false);
               } catch (Exception e) {
                   Slog.w(TAG, "Failure sending service " + s.shortName
                           + " to connection " + c.conn.asBinder()
                           + " (in " + c.binding.client.processName + ")", e);
               }

               // If this is the first app connected back to this binding,
               // and the service had previously asked to be told when
               // rebound, then do so.
               //当第一个app连接到该binding，且之前已经被bind过，则回调onRebind方法
               if (b.intent.apps.size() == 1 && b.intent.doRebind) {
                   requestServiceBindingLocked(s, b.intent, callerFg, true);
               }
           } else if (!b.intent.requested) {
               //最终回调onBind方法
               requestServiceBindingLocked(s, b.intent, callerFg, false);
           }

           getServiceMapLocked(s.userId).ensureNotStartingBackgroundLocked(s);

       } finally {
           Binder.restoreCallingIdentity(origId);
       }

       return 1;
   }</code></pre>

<h3 id="3-10-AS-bringUpServiceLocked"><a href="#3-10-AS-bringUpServiceLocked" class="headerlink" title="3.10 AS.bringUpServiceLocked"></a>3.10 AS.bringUpServiceLocked</h3><p>[-&gt;ActiveServices.java]</p>

<pre><code>
private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
            boolean whileRestarting, boolean permissionsReviewRequired)
            throws TransactionTooLargeException {
        //Slog.i(TAG, "Bring up service:");
        //r.dump("  ");
        //进程已经存在的情况
        if (r.app != null && r.app.thread != null) {
            //调用onStartCommand过程
            sendServiceArgsLocked(r, execInFg, false);
            return null;
        }

        if (!whileRestarting && mRestartingServices.contains(r)) {
            // If waiting for a restart, then do nothing.
            return null;
        }

        if (DEBUG_SERVICE) {
            Slog.v(TAG_SERVICE, "Bringing up " + r + " " + r.intent + " fg=" + r.fgRequired);
        }

        // We are now bringing the service up, so no longer in the
        // restarting state.
        if (mRestartingServices.remove(r)) {
            clearRestartingIfNeededLocked(r);
        }

        // Make sure this service is no longer considered delayed, we are starting it now.
        if (r.delayed) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "REM FR DELAY LIST (bring up): " + r);
            getServiceMapLocked(r.userId).mDelayedStartList.remove(r);
            r.delayed = false;
        }

        // Make sure that the user who owns this service is started.  If not,
        // we don't want to allow it to run.
        if (!mAm.mUserController.hasStartedUserState(r.userId)) {
            String msg = "Unable to launch app "
                    + r.appInfo.packageName + "/"
                    + r.appInfo.uid + " for service "
                    + r.intent.getIntent() + ": user " + r.userId + " is stopped";
            Slog.w(TAG, msg);
            bringDownServiceLocked(r);
            return msg;
        }

        // Service is now being launched, its package can't be stopped.
        //服务正在启动，设置package停止状态为false
        try {
            AppGlobals.getPackageManager().setPackageStoppedState(
                    r.packageName, false, r.userId);
        } catch (RemoteException e) {
        } catch (IllegalArgumentException e) {
            Slog.w(TAG, "Failed trying to unstop package "
                    + r.packageName + ": " + e);
        }

        final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
        final String procName = r.processName;
        String hostingType = "service";
        ProcessRecord app;

        if (!isolated) {
            app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
            if (DEBUG_MU) Slog.v(TAG_MU, "bringUpServiceLocked: appInfo.uid=" + r.appInfo.uid
                        + " app=" + app);
            if (app != null && app.thread != null) {
                try {
                    app.addPackage(r.appInfo.packageName, r.appInfo.longVersionCode, mAm.mProcessStats);
                    //启动服务，见3.11节
                    realStartServiceLocked(r, app, execInFg);
                    return null;
                } catch (TransactionTooLargeException e) {
                    throw e;
                } catch (RemoteException e) {
                    Slog.w(TAG, "Exception when starting service " + r.shortName, e);
                }

                // If a dead object exception was thrown -- fall through to
                // restart the application.
            }
        } else {
            // If this service runs in an isolated process, then each time
            // we call startProcessLocked() we will get a new isolated
            // process, starting another process if we are currently waiting
            // for a previous process to come up.  To deal with this, we store
            // in the service any current isolated process it is running in or
            // waiting to have come up.
            app = r.isolatedProc;
            if (WebViewZygote.isMultiprocessEnabled()
                    && r.serviceInfo.packageName.equals(WebViewZygote.getPackageName())) {
                hostingType = "webview_service";
            }
        }

        // Not running -- get it started, and enqueue this service record
        // to be executed when the app comes up.
        //对于进程没有启动的情况
        if (app == null && !permissionsReviewRequired) {
            //启动service所要运行的进程，最终会调用3.11小节
            if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    hostingType, r.name, false, isolated, false)) == null) {
                String msg = "Unable to launch app "
                        + r.appInfo.packageName + "/"
                        + r.appInfo.uid + " for service "
                        + r.intent.getIntent() + ": process is bad";
                Slog.w(TAG, msg);
                bringDownServiceLocked(r);
                return msg;
            }
            if (isolated) {
                r.isolatedProc = app;
            }
        }

        if (r.fgRequired) {
            if (DEBUG_FOREGROUND_SERVICE) {
                Slog.v(TAG, "Whitelisting " + UserHandle.formatUid(r.appInfo.uid)
                        + " for fg-service launch");
            }
            mAm.tempWhitelistUidLocked(r.appInfo.uid,
                    SERVICE_START_FOREGROUND_TIMEOUT, "fg-service-launch");
        }

        if (!mPendingServices.contains(r)) {
            mPendingServices.add(r);
        }

        if (r.delayedStop) {
            // Oh and hey we've already been asked to stop!
            // 要求停止服务
            r.delayedStop = false;
            if (r.startRequested) {
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                        "Applying delayed stop (in bring up): " + r);
                stopServiceLocked(r);
            }
        }

        return null;
    }</code></pre>

<h3 id="3-11-AS-realStartServiceLocked"><a href="#3-11-AS-realStartServiceLocked" class="headerlink" title="3.11 AS.realStartServiceLocked"></a>3.11 AS.realStartServiceLocked</h3><p>[-&gt;ActiveServices.java]</p>

<pre><code>
private final void realStartServiceLocked(ServiceRecord r,
           ProcessRecord app, boolean execInFg) throws RemoteException {
       if (app.thread == null) {
           throw new RemoteException();
       }
       if (DEBUG_MU)
           Slog.v(TAG_MU, "realStartServiceLocked, ServiceRecord.uid = " + r.appInfo.uid
                   + ", ProcessRecord.uid = " + app.uid);
       r.app = app;
       r.restartTime = r.lastActivity = SystemClock.uptimeMillis();

       final boolean newService = app.services.add(r);
       bumpServiceExecutingLocked(r, execInFg, "create");
       mAm.updateLruProcessLocked(app, false, null);
       updateServiceForegroundLocked(r.app, /* oomAdj= */ false);
       mAm.updateOomAdjLocked();

       boolean created = false;
       try {
           if (LOG_SERVICE_START_STOP) {
               String nameTerm;
               int lastPeriod = r.shortName.lastIndexOf('.');
               nameTerm = lastPeriod &gt;= 0 ? r.shortName.substring(lastPeriod) : r.shortName;
               EventLogTags.writeAmCreateService(
                       r.userId, System.identityHashCode(r), nameTerm, r.app.uid, r.app.pid);
           }
           synchronized (r.stats.getBatteryStats()) {
               r.stats.startLaunchedLocked();
           }
           mAm.notifyPackageUse(r.serviceInfo.packageName,
                                PackageManager.NOTIFY_PACKAGE_USE_SERVICE);
           app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
           //服务进入onCreate方法，见流程4.1小节
           app.thread.scheduleCreateService(r, r.serviceInfo,
                   mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                   app.repProcState);
           r.postNotification();
           created = true;
       } catch (DeadObjectException e) {
           Slog.w(TAG, "Application dead when creating service " + r);
           mAm.appDiedLocked(app); //应用死亡
           throw e;
       } finally {
           if (!created) {
               // Keep the executeNesting count accurate.
               final boolean inDestroying = mDestroyingServices.contains(r);
               serviceDoneExecutingLocked(r, inDestroying, inDestroying);

               // Cleanup.
               if (newService) {
                   app.services.remove(r);
                   r.app = null;
               }

               // Retry.
               //尝试重新启动服务
               if (!inDestroying) {
                   scheduleServiceRestartLocked(r, false);
               }
           }
       }

       if (r.whitelistManager) {
           app.whitelistManager = true;
       }
       //见流程5.1小节
       requestServiceBindingsLocked(r, execInFg);

       updateServiceClientActivitiesLocked(app, null, true);

       // If the service is in the started state, and there are no
       // pending arguments, then fake up one so its onStartCommand() will
       // be called.
       if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
           r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                   null, null, 0));
       }
       //onStartCommand
       sendServiceArgsLocked(r, execInFg, true);

       if (r.delayed) {
           if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "REM FR DELAY LIST (new proc): " + r);
           getServiceMapLocked(r.userId).mDelayedStartList.remove(r);
           r.delayed = false;
       }

       if (r.delayedStop) {
           // Oh and hey we've already been asked to stop!
           r.delayedStop = false;
           if (r.startRequested) {
               if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                       "Applying delayed stop (from start): " + r);
               stopServiceLocked(r);
           }
       }
   }
</code></pre>

<h2 id="四、服务端进程"><a href="#四、服务端进程" class="headerlink" title="四、服务端进程"></a>四、服务端进程</h2><h3 id="4-1-AT-scheduleCreateService"><a href="#4-1-AT-scheduleCreateService" class="headerlink" title="4.1 AT.scheduleCreateService"></a>4.1 AT.scheduleCreateService</h3><p>[-&gt;ActivityThread.java]</p>

<pre><code>
public final void scheduleCreateService(IBinder token,
               ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
           updateProcessState(processState, false);
           //准备服务创建所需要的数据
           CreateServiceData s = new CreateServiceData();
           s.token = token;
           s.info = info;
           s.compatInfo = compatInfo;

           sendMessage(H.CREATE_SERVICE, s);
    }
</code></pre>

<p>通过handler机制，发送消息给服务端进程的主线程的handler处理。</p>
<h3 id="4-2-AT-handleMessage"><a href="#4-2-AT-handleMessage" class="headerlink" title="4.2 AT.handleMessage"></a>4.2 AT.handleMessage</h3><p>[-&gt;ActivityThread.java]</p>

<pre><code>
public void handleMessage(Message msg) {
      if (DEBUG_MESSAGES) Slog.v(TAG, "&gt;&gt;&gt; handling: " + codeToString(msg.what));
          switch (msg.what) {
              case CREATE_SERVICE:
                  Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, ("serviceCreate: " + String.valueOf(msg.obj)));
                  //见4.3小节
                  handleCreateService((CreateServiceData)msg.obj);
                  Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                  break;
                  ....
            }
            
        }</code></pre>

<h3 id="4-3-AT-handleCreateService"><a href="#4-3-AT-handleCreateService" class="headerlink" title="4.3 AT.handleCreateService"></a>4.3 AT.handleCreateService</h3><p>[-&gt;ActivityThread.java]</p>

<pre><code>
private void handleCreateService(CreateServiceData data) {
       // If we are getting ready to gc after going to the background, well
       // we are back active so skip it.
       //当应用处于后台即将进行gc,而此时被调回到活动，则此时跳过本次gc
       unscheduleGcIdler();

       LoadedApk packageInfo = getPackageInfoNoCheck(
               data.info.applicationInfo, data.compatInfo);
       Service service = null;
       try {
           //通过反射创建目标服务对象
           java.lang.ClassLoader cl = packageInfo.getClassLoader();
           service = packageInfo.getAppFactory()
                   .instantiateService(cl, data.info.name, data.intent);
       } catch (Exception e) {
           if (!mInstrumentation.onException(service, e)) {
               throw new RuntimeException(
                   "Unable to instantiate service " + data.info.name
                   + ": " + e.toString(), e);
           }
       }

       try {
           if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);
           //创建ContextImpl对象
           ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
           context.setOuterContext(service);
           //创建Application对象
           Application app = packageInfo.makeApplication(false, mInstrumentation);
           service.attach(context, this, data.info.name, data.token, app,
                   ActivityManager.getService());
           //调用服务的onCreate方法
           service.onCreate();
           mServices.put(data.token, service);
           try {
               //调用服务创建完成
               ActivityManager.getService().serviceDoneExecuting(
                       data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
           } catch (RemoteException e) {
               throw e.rethrowFromSystemServer();
           }
       } catch (Exception e) {
           if (!mInstrumentation.onException(service, e)) {
               throw new RuntimeException(
                   "Unable to create service " + data.info.name
                   + ": " + e.toString(), e);
           }
       }
   }</code></pre>

<p>前面3.11中realStartServiceLocked过程，执行完成scheduleCreateService操作后，接下来，继续回到system_server进程，开始执行requestServiceBindingsLocked过程。</p>
<h2 id="五、system-server进程"><a href="#五、system-server进程" class="headerlink" title="五、system_server进程"></a>五、system_server进程</h2><h3 id="5-1-AS-requestServiceBindingsLocked"><a href="#5-1-AS-requestServiceBindingsLocked" class="headerlink" title="5.1 AS.requestServiceBindingsLocked"></a>5.1 AS.requestServiceBindingsLocked</h3><p>[-&gt;ActiveServices.java]</p>

<pre><code>
private final void requestServiceBindingsLocked(ServiceRecord r, boolean execInFg)
           throws TransactionTooLargeException {
       for (int i=r.bindings.size()-1; i&gt;=0; i--) {
           IntentBindRecord ibr = r.bindings.valueAt(i);
           if (!requestServiceBindingLocked(r, ibr, execInFg, false)) {
               break;
           }
       }
   }</code></pre>

<p>通过bindService方式启动服务，那么ServiceRecord的bindings肯定不会为空。</p>
<h3 id="5-2-AS-requestServiceBindingLocked"><a href="#5-2-AS-requestServiceBindingLocked" class="headerlink" title="5.2 AS.requestServiceBindingLocked"></a>5.2 AS.requestServiceBindingLocked</h3><p>[-&gt;ActiveServices.java]</p>

<pre><code>
private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
            boolean execInFg, boolean rebind) throws TransactionTooLargeException {
        if (r.app == null || r.app.thread == null) {
            // If service is not currently running, can't yet bind.
            return false;
        }
        if (DEBUG_SERVICE) Slog.d(TAG_SERVICE, "requestBind " + i + ": requested=" + i.requested
                + " rebind=" + rebind);
        if ((!i.requested || rebind) && i.apps.size() &gt; 0) {
            try {
               //发送bind开始的消息
                bumpServiceExecutingLocked(r, execInFg, "bind");
                r.app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
                //服务进入onBind流程，见流程6.1小节
                r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
                        r.app.repProcState);
                if (!rebind) {
                    i.requested = true;
                }
                i.hasBound = true;
                i.doRebind = false;
            } catch (TransactionTooLargeException e) {
                // Keep the executeNesting count accurate.
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Crashed while binding " + r, e);
                final boolean inDestroying = mDestroyingServices.contains(r);
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);
                throw e;
            } catch (RemoteException e) {
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Crashed while binding " + r);
                // Keep the executeNesting count accurate.
                final boolean inDestroying = mDestroyingServices.contains(r);
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);
                return false;
            }
        }
        return true;
    }</code></pre>

<p>scheduleBindService通过Binder代理的方式，调用AT的scheduleBindService，其代理对象由IApplicationThread.aidl生成和AMS类似。</p>
<h2 id="六、服务端进程"><a href="#六、服务端进程" class="headerlink" title="六、服务端进程"></a>六、服务端进程</h2><h3 id="6-1-AT-scheduleBindService"><a href="#6-1-AT-scheduleBindService" class="headerlink" title="6.1 AT.scheduleBindService"></a>6.1 AT.scheduleBindService</h3><p>[-&gt;ActivityThread.java]</p>

<pre><code>
public final void scheduleBindService(IBinder token, Intent intent,
             boolean rebind, int processState) {
         updateProcessState(processState, false);
         BindServiceData s = new BindServiceData();
         s.token = token;
         s.intent = intent;
         s.rebind = rebind;

         if (DEBUG_SERVICE)
             Slog.v(TAG, "scheduleBindService token=" + token + " intent=" + intent + " uid="
                     + Binder.getCallingUid() + " pid=" + Binder.getCallingPid());
         sendMessage(H.BIND_SERVICE, s);
     }</code></pre>

<p>发送消息到服务端进程的主线程处理。</p>
<h3 id="6-2-AT-handleMessage"><a href="#6-2-AT-handleMessage" class="headerlink" title="6.2 AT.handleMessage"></a>6.2 AT.handleMessage</h3><p>[-&gt;ActivityThread.java]</p>

<pre><code>
public void handleMessage(Message msg) {
          if (DEBUG_MESSAGES) Slog.v(TAG, "&gt;&gt;&gt; handling: " + codeToString(msg.what));
          switch (msg.what) {
             ...
              case BIND_SERVICE:
                  //见6.3小节
                  Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceBind");
                  handleBindService((BindServiceData)msg.obj);
                  Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                  break;
               ...
               }
        }</code></pre>

<h3 id="6-3-AT-handleBindService"><a href="#6-3-AT-handleBindService" class="headerlink" title="6.3 AT.handleBindService"></a>6.3 AT.handleBindService</h3><p>[-&gt;ActivityThread.java]</p>

<pre><code>
private void handleBindService(BindServiceData data) {
       Service s = mServices.get(data.token);
       if (DEBUG_SERVICE)
           Slog.v(TAG, "handleBindService s=" + s + " rebind=" + data.rebind);
       if (s != null) {
           try {
               data.intent.setExtrasClassLoader(s.getClassLoader());
               data.intent.prepareToEnterProcess();
               try {
                   if (!data.rebind) {
                       //执行onBind回调方法
                       IBinder binder = s.onBind(data.intent);
                       //将onBind回来的对象传递回去，见流程6.4小节
                       ActivityManager.getService().publishService(
                               data.token, data.intent, binder);
                   } else {
                      //执行onRebind方法
                       s.onRebind(data.intent);
                       ActivityManager.getService().serviceDoneExecuting(
                               data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                   }
               } catch (RemoteException ex) {
                   throw ex.rethrowFromSystemServer();
               }
           } catch (Exception e) {
               if (!mInstrumentation.onException(s, e)) {
                   throw new RuntimeException(
                           "Unable to bind to service " + s
                           + " with " + data.intent + ": " + e.toString(), e);
               }
           }
       }
   }</code></pre>

<p>经过Binder IPC进入到system_server进程，并将binder传回到system_server进程。</p>
<h2 id="七、system-server进程"><a href="#七、system-server进程" class="headerlink" title="七、system_server进程"></a>七、system_server进程</h2><h3 id="7-1-AMS-publishService"><a href="#7-1-AMS-publishService" class="headerlink" title="7.1 AMS.publishService"></a>7.1 AMS.publishService</h3><p>[-&gt;ActivityManagerService.java]</p>

<pre><code>
public void publishService(IBinder token, Intent intent, IBinder service) {
       // Refuse possible leaked file descriptors
       if (intent != null && intent.hasFileDescriptors() == true) {
           throw new IllegalArgumentException("File descriptors passed in Intent");
       }

       synchronized(this) {
           if (!(token instanceof ServiceRecord)) {
               throw new IllegalArgumentException("Invalid service token");
           }
           mServices.publishServiceLocked((ServiceRecord)token, intent, service);
       }
   }</code></pre>

<p>服务端的onBind返回的binder对象，在经过writeStrongBinder传递到底层，再回到system_server进程，经过readStrongBinder获取代理对象。</p>
<h3 id="7-2-AMS-publishServiceLocked"><a href="#7-2-AMS-publishServiceLocked" class="headerlink" title="7.2 AMS.publishServiceLocked"></a>7.2 AMS.publishServiceLocked</h3><p>[-&gt;ActiveServices.java]</p>

<pre><code>
void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
       final long origId = Binder.clearCallingIdentity();
       try {
           if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "PUBLISHING " + r
                   + " " + intent + ": " + service);
           if (r != null) {
               Intent.FilterComparison filter
                       = new Intent.FilterComparison(intent);
               IntentBindRecord b = r.bindings.get(filter);
               if (b != null && !b.received) {
                   b.binder = service;
                   b.requested = true;
                   b.received = true;
                   for (int conni=r.connections.size()-1; conni&gt;=0; conni--) {
                       ArrayList&lt;ConnectionRecord&gt; clist = r.connections.valueAt(conni);
                       for (int i=0; i&lt;clist.size(); i++) {
                           ConnectionRecord c = clist.get(i);
                           if (!filter.equals(c.binding.intent.intent)) {
                               if (DEBUG_SERVICE) Slog.v(
                                       TAG_SERVICE, "Not publishing to: " + c);
                               if (DEBUG_SERVICE) Slog.v(
                                       TAG_SERVICE, "Bound intent: " + c.binding.intent.intent);
                               if (DEBUG_SERVICE) Slog.v(
                                       TAG_SERVICE, "Published intent: " + intent);
                               continue;
                           }
                           if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Publishing to: " + c);
                           try {
                               //见流程8.1
                               c.conn.connected(r.name, service, false);
                           } catch (Exception e) {
                               Slog.w(TAG, "Failure sending service " + r.name +
                                     " to connection " + c.conn.asBinder() +
                                     " (in " + c.binding.client.processName + ")", e);
                           }
                       }
                   }
               }

               serviceDoneExecutingLocked(r, mDestroyingServices.contains(r), false);
           }
       } finally {
           Binder.restoreCallingIdentity(origId);
       }
   }</code></pre>

<p>c.conn是指客户端进程的IServiceConnection.Stub.Proxy代理对象，通过BinderIPC调用，进入客户端的IServiceConnection.Stub对象，回到客户端进程中的InnerConnection对象。</p>
<h2 id="八、客户端进程"><a href="#八、客户端进程" class="headerlink" title="八、客户端进程"></a>八、客户端进程</h2><h3 id="8-1-InnerConnection-connected"><a href="#8-1-InnerConnection-connected" class="headerlink" title="8.1  InnerConnection.connected"></a>8.1  InnerConnection.connected</h3><p>[-&gt;LoadedApk.java]</p>

<pre><code>
private static class InnerConnection extends IServiceConnection.Stub {
            @UnsupportedAppUsage
            final WeakReference&lt;LoadedApk.ServiceDispatcher&gt; mDispatcher;

            InnerConnection(LoadedApk.ServiceDispatcher sd) {
                mDispatcher = new WeakReference&lt;LoadedApk.ServiceDispatcher&gt;(sd);
            }

            public void connected(ComponentName name, IBinder service, boolean dead)
                    throws RemoteException {
                LoadedApk.ServiceDispatcher sd = mDispatcher.get();
                if (sd != null) {
                    //见流程8.2
                    sd.connected(name, service, dead);
                }
            }
        }
</code></pre>

<h3 id="8-2-SD-connected"><a href="#8-2-SD-connected" class="headerlink" title="8.2  SD.connected"></a>8.2  SD.connected</h3><p>[-&gt;LoadedApk.java]</p>

<pre><code>
public void connected(ComponentName name, IBinder service, boolean dead) {
          if (mActivityThread != null) {
              //这里是主线程的handler
              mActivityThread.post(new RunConnection(name, service, 0, dead));
          } else {
              doConnected(name, service, dead);
          }
      }</code></pre>

<h3 id="8-3-new-RunConnection"><a href="#8-3-new-RunConnection" class="headerlink" title="8.3  new RunConnection"></a>8.3  new RunConnection</h3><p>[-&gt;LoadedApk.java]</p>

<pre><code>
private final class RunConnection implements Runnable {
          RunConnection(ComponentName name, IBinder service, int command, boolean dead) {
              mName = name;
              mService = service;
              mCommand = command;  //此时为0
              mDead = dead;
          }

          public void run() {
              if (mCommand == 0) {
                  //见流程8.4小节
                  doConnected(mName, mService, mDead);
              } else if (mCommand == 1) {
                  doDeath(mName, mService);
              }
          }

          final ComponentName mName;
          final IBinder mService; //onBinder返回的代理对象
          final int mCommand;
          final boolean mDead;
      }</code></pre>

<h3 id="8-4-doConnected"><a href="#8-4-doConnected" class="headerlink" title="8.4 doConnected"></a>8.4 doConnected</h3><p>[-&gt;LoadedApk.java]</p>

<pre><code>
public void doConnected(ComponentName name, IBinder service, boolean dead) {
           ServiceDispatcher.ConnectionInfo old;
           ServiceDispatcher.ConnectionInfo info;

           synchronized (this) {
               if (mForgotten) {
                   // We unbound before receiving the connection; ignore
                   // any connection received.
                   return;
               }
               old = mActiveConnections.get(name);
               if (old != null && old.binder == service) {
                   // Huh, already have this one.  Oh well!
                   return;
               }

               if (service != null) {
                   // A new service is being connected... set it all up.
                   info = new ConnectionInfo();
                   info.binder = service;
                   //创建死亡监听对象
                   info.deathMonitor = new DeathMonitor(name, service);
                   try {
                       //建立死亡通知
                       service.linkToDeath(info.deathMonitor, 0);
                       mActiveConnections.put(name, info);
                   } catch (RemoteException e) {
                       // This service was dead before we got it...  just
                       // don't do anything with it.
                       mActiveConnections.remove(name);
                       return;
                   }

               } else {
                   // The named service is being disconnected... clean up.
                   mActiveConnections.remove(name);
               }

               if (old != null) {
                   old.binder.unlinkToDeath(old.deathMonitor, 0);
               }
           }

           // If there was an old service, it is now disconnected.
           if (old != null) {
               mConnection.onServiceDisconnected(name);
           }
           if (dead) {
               mConnection.onBindingDied(name);
           }
           // If there is a new viable service, it is now connected.
           if (service != null) {
               //回调用户自定义的ServiceConnected对象方法
               mConnection.onServiceConnected(name, service);
           } else {
               // The binding machinery worked, but the remote returned null from onBind().
               mConnection.onNullBinding(name);
           }
       }</code></pre>

<h2 id="九、总结"><a href="#九、总结" class="headerlink" title="九、总结"></a>九、总结</h2><h3 id="9-1-通信流程"><a href="#9-1-通信流程" class="headerlink" title="9.1 通信流程"></a>9.1 通信流程</h3><p>1.发起端线程向Binder驱动发起binder_ioctl请求后，waitForResponse进入while循环，不断进行talkWithDriver,此时该线程处理阻塞状态，直到收到BR_XX命令才会结束该过程。</p>
<ul>
<li>BR_TRANSACTION_COMPLETE: oneway模式下,收到该命令则退出；</li>
<li>BR_DEAD_REPLY: 目标进程/线程/binder实体为空, 以及释放正在等待reply的binder thread或者binder buffer;</li>
<li>BR_FAILED_REPLY: 情况较多,比如非法handle, 错误事务栈, security, 内存不足, buffer不足, 数据拷贝失败, 节点创建失败, 各种不匹配等问题；</li>
<li>BR_ACQUIRE_RESULT: 目前未使用的协议;</li>
<li>BR_REPLY: 非oneway模式下,收到该命令才退出;</li>
</ul>
<p>2.waitForResponse收到BR_TRANSACTION_COMPLETE，则直接退出循环，不会执行executeCommand方法，除上述五种BR_XXX命令，当收到其他BR命令，则会执行executeCommand方法。</p>
<p>3.目标Binder线程创建之后，便进入joinThreadPool方法，不断循环执行getAndExecuteCommand方法，当bwr的读写buffer没有数据时，则阻塞在binder_thread_read的wait_event过程。正常情况下binder线程一旦创建就不会退出。</p>
<p><img src="https://img-blog.csdnimg.cn/5dd1300ea9394c76a04c63f760e542b2.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_14,color_FFFFFF,t_70,g_se,x_16" alt="bindservice_cm_frame" style="zoom:80%;"></p>
<h3 id="9-2-通信协议"><a href="#9-2-通信协议" class="headerlink" title="9.2 通信协议"></a>9.2 通信协议</h3><p>1.Binder客户端和服务端向Binder驱动发送的命令都是以BC_开头，Binder驱动向服务端或客户端发送的命令都是以 BR _开头；</p>
<p>2.只有当BC_TRANSACTION或BC_REPLY时，才会调用binder_transaction来处理事务，并且都会回应调用者BINDER_WORK_TRANSACTION_COMPLETE，经过binder_thread_read转变成BR_TRANSACTION_COMPLETE；</p>
<p>3.bindServie是一个非oneway过程，oneway过程没有BC_REPLY。</p>
<p><img src="https://img-blog.csdnimg.cn/b0655680b5814257bf4b66f7d735daad.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_18,color_FFFFFF,t_70,g_se,x_16" alt="binderservice_cm_pro" style="zoom:75%;"></p>
<h3 id="9-3-数据流"><a href="#9-3-数据流" class="headerlink" title="9.3 数据流"></a>9.3 数据流</h3><p><strong>用户空间</strong></p>
<p>1.bindService：组装flat_binder_object等对象组成Parcel data;</p>
<p>2.IPC.writeTransactionData：组装BC_TRANSACTION和binder_transaction_data结构体，写入mOut;</p>
<p>3.IPC.talkWithDriver：组装BINDER_WRITE_READ和binder_writer_read结构体，通过ioctl传输到驱动层。</p>
<p><strong>进入驱动后</strong></p>
<p>4.binder_thread_write:处理binder_write_read.write_buffer数据</p>
<p>5.binder_transaction:处理write_buffer.binder_transaction_data数据</p>
<ul>
<li>创建binder_transaction结构体，记录事务通信的线程来源以及事务链条等相关信息；</li>
<li>分配binder_buffer结构体，拷贝当前线程binder_transaction_data的data数据到binder_buffer-&gt;data;</li>
</ul>
<p>6.binder_thread_read:处理binder_transaction结构体数据</p>
<ul>
<li>组装cmd= BR_TRANSACTION和binder_transaction_data结构体，写入binder_write_read.read_buffer数据。</li>
</ul>
<p><strong>回到用户空间</strong></p>
<p>7.IPC.executeCommand:处理BR_TRANSACIOTN命令，将binder_transaction_data数据解析成BBinder.transact所需的参数</p>
<p>8.onTransact:层层回调，进入该方法，反序列化数据后，调用bindService方法。</p>
<p><img src="https://img-blog.csdnimg.cn/803f08a8a0bb46c6a6638427d4606952.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_17,color_FFFFFF,t_70,g_se,x_16" alt="bindservice_data"></p>
<h2 id="附录"><a href="#附录" class="headerlink" title="附录"></a>附录</h2><p>源码路径</p>

<pre><code>
frameworks/base/core/java/android/app/ContextImpl.java
frameworks/base/core/java/android/app/LoadedApk.java
frameworks/base/core/java/android/app/ActivityManager.java
frameworks/base/core/java/android/os/Binder.java
frameworks/base/core/java/android/os/BinderProxy.java

frameworks/base/core/java/android/app/ActivityThread.java
frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
frameworks/base/core/java/android/os/Binder.java
frameworks/base/core/jni/android_util_Binder.cpp

native/libs/binder/ProcessState.cpp
native/libs/binder/Parcel.cpp
native/libs/binder/IPCThreadState.cpp
native/libs/binder/Binder.cpp
</code></pre>

      