---
layout:     post
title:      Android10 Binder机制4-获取服务
subtitle:   获取服务是通过ServiceManager中的getService静态方法获取具体的服务，这个流程经历Java层，Native层，Kernel层。
date:       2021-03-05
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

<h2 id="一、概述"><a href="#一、概述" class="headerlink" title="一、概述"></a>一、概述</h2><p>本文将介绍系统服务获取的具体流程，如获取ActivityManagerService时，通过ServiceManager中的getService静态方法获取具体的服务，这个流程经历Java层，Native层，Kernel层，其通信流程如下：</p>
<p><img src="/2020/深入理解Binder机制3-获取服务getService/bind_getService.PNG" alt="bind_getService"></p>
<p>1.发起端进程向Binder Driver发送binder_ioct请求后，采用不断的循环talkWithDriver，此时线程处于阻塞状态，直到收到BR_RPLEY命令才会结束该流程；</p>
<p>2.waitForResponse收到BR_RPLEY命令后，则直接退出循环，不会再执行executeCommand方法，除六种其他命令外(BR_REPLY、BR_TRANSACTION_COMPLETE、BR_DEAD_REPLY、BR_FAILED_REPLY、BR_ACQUIRE_RESULT<br>、BR_REPLY)会执行executeCommand方法；</p>
<p>3.由于ServiceManager进程已经启动，并且有binder_loop一直在循环查询命令，当收到BR_TRANSACTION命令后，就开始处理findService过程。</p>
<p>4.找到服务之后，通过binder驱动再传回到发起端进程，waitForResponse处理驱动发过来的BR_RPLEY，将服务的代理类返回给发起端进程。</p>
<h2 id="二、获取服务进程"><a href="#二、获取服务进程" class="headerlink" title="二、获取服务进程"></a>二、获取服务进程</h2><h3 id="2-1-SM-getService"><a href="#2-1-SM-getService" class="headerlink" title="2.1 SM.getService"></a>2.1 SM.getService</h3><p>[-&gt;ServiceManager.java]</p>
<pre><code>public static IBinder getService(String name) {
      try {
          IBinder service = sCache.get(name);
          if (service != null) {
              return service;
          } else {
              return Binder.allowBlocking(rawGetService(name));
          }
      } catch (RemoteException e) {
          Log.e(TAG, "error in getService", e);
      }
      return null;
  }</code></pre>
<h4 id="2-1-1-rawGetService"><a href="#2-1-1-rawGetService" class="headerlink" title="2.1.1 rawGetService"></a>2.1.1 rawGetService</h4><p>[-&gt;ServiceManager.java]</p>
<pre><code>private static IBinder rawGetService(String name) throws RemoteException {
       final long start = sStatLogger.getTime();

       final IBinder binder = getIServiceManager().getService(name);
       
       //记录获取服务时间
       final int time = (int) sStatLogger.logDurationStat(Stats.GET_SERVICE, start);

       final int myUid = Process.myUid();
       final boolean isCore = UserHandle.isCore(myUid);

       final long slowThreshold = isCore
               ? GET_SERVICE_SLOW_THRESHOLD_US_CORE
               : GET_SERVICE_SLOW_THRESHOLD_US_NON_CORE;
       //记录时间
       synchronized (sLock) {
           sGetServiceAccumulatedUs += time;
           sGetServiceAccumulatedCallCount++;

           final long nowUptime = SystemClock.uptimeMillis();

           // Was a slow call?
           if (time &gt;= slowThreshold) {
               // We do a slow log:
               // - At most once in every SLOW_LOG_INTERVAL_MS
               // - OR it was slower than the previously logged slow call.
               if ((nowUptime &gt; (sLastSlowLogUptime + SLOW_LOG_INTERVAL_MS))
                       || (sLastSlowLogActualTime &lt; time)) {
                   EventLogTags.writeServiceManagerSlow(time / 1000, name);

                   sLastSlowLogUptime = nowUptime;
                   sLastSlowLogActualTime = time;
               }
           }

           // Every GET_SERVICE_LOG_EVERY_CALLS calls, log the total time spent in getService().

           final int logInterval = isCore
                   ? GET_SERVICE_LOG_EVERY_CALLS_CORE
                   : GET_SERVICE_LOG_EVERY_CALLS_NON_CORE;

           if ((sGetServiceAccumulatedCallCount &gt;= logInterval)
                   && (nowUptime &gt;= (sLastStatsLogUptime + STATS_LOG_INTERVAL_MS))) {

               EventLogTags.writeServiceManagerStats(
                       sGetServiceAccumulatedCallCount, // Total # of getService() calls.
                       sGetServiceAccumulatedUs / 1000, // Total time spent in getService() calls.
                       (int) (nowUptime - sLastStatsLogUptime)); // Uptime duration since last log.
               sGetServiceAccumulatedCallCount = 0;
               sGetServiceAccumulatedUs = 0;
               sLastStatsLogUptime = nowUptime;
           }
       }
       return binder;
   }
</code></pre>
<p>由前面<a href="https://skytoby.github.io/2020/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Binder%E6%9C%BA%E5%88%B62-%E6%B3%A8%E5%86%8C%E6%9C%8D%E5%8A%A1addService/" target="_blank" rel="noopener">ServiceManager注册服务</a>中3.3.1节可知getIServiceManager()相当于ServiceManagerProxy</p>
<h4 id="2-1-2-allowBlocking"><a href="#2-1-2-allowBlocking" class="headerlink" title="2.1.2 allowBlocking"></a>2.1.2 allowBlocking</h4><p>[-&gt;Binder.java]</p>
<pre><code>public static IBinder allowBlocking(IBinder binder) {
       try {
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
<p>这个方法用来判断IBinder类，根据不同情况添加标志。</p>
<h3 id="2-2-SMP-getService"><a href="#2-2-SMP-getService" class="headerlink" title="2.2 SMP.getService"></a>2.2 SMP.getService</h3><p>[-&gt;ServiceManagerNative::ServiceManagerProxy]</p>
<pre><code>public IBinder getService(String name) throws RemoteException {
       Parcel data = Parcel.obtain();
       Parcel reply = Parcel.obtain();
       data.writeInterfaceToken(IServiceManager.descriptor);
       data.writeString(name);
       mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0);
       IBinder binder = reply.readStrongBinder();
       reply.recycle();
       data.recycle();
       return binder;
   }
</code></pre>
<p>由前面<a href="https://skytoby.github.io/2020/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Binder%E6%9C%BA%E5%88%B62-%E6%B3%A8%E5%86%8C%E6%9C%8D%E5%8A%A1addService/" target="_blank" rel="noopener">ServiceManager注册服务</a>中3.3.1节</p>
<p>mRemote为BinderProxy对象，该对象对应于BpBinder(0)，作为binder代理类，执行native层ServiceManager</p>
<h3 id="2-3-BP-transact"><a href="#2-3-BP-transact" class="headerlink" title="2.3 BP.transact"></a>2.3 BP.transact</h3><p>[-&gt;BinderProxy.java]</p>
<pre><code>public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
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
   }
</code></pre>
<h3 id="2-4-android-os-BinderProxy-transact"><a href="#2-4-android-os-BinderProxy-transact" class="headerlink" title="2.4  android_os_BinderProxy_transact"></a>2.4  android_os_BinderProxy_transact</h3><p>[-&gt;android_util_Binder.cpp]</p>
<pre><code>static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
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
<h3 id="2-5-BpBinder-transact"><a href="#2-5-BpBinder-transact" class="headerlink" title="2.5 BpBinder::transact"></a>2.5 BpBinder::transact</h3><p>[-&gt;BpBinder.cpp]</p>
<pre><code>status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        //见2.6节
        status_t status = IPCThreadState::self()-&gt;transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }

    return DEAD_OBJECT;
}</code></pre>
<h4 id="2-5-1-BpBinder-transact"><a href="#2-5-1-BpBinder-transact" class="headerlink" title="2.5.1 BpBinder::transact"></a>2.5.1 BpBinder::transact</h4><p>[-&gt;IPCThreadState.cpp]</p>
<pre><code>IPCThreadState* IPCThreadState::self()
{
    if (gHaveTLS) {
restart:
        const pthread_key_t k = gTLS;
        IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
        if (st) return st;
        return new IPCThreadState; //创建IPCThreadState，见2.5.2
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
<h4 id="2-5-2-new-IPCThreadState"><a href="#2-5-2-new-IPCThreadState" class="headerlink" title="2.5.2 new IPCThreadState"></a>2.5.2 new IPCThreadState</h4><p>[-&gt;IPCThreadState.cpp]</p>
<pre><code>IPCThreadState::IPCThreadState()
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
<h3 id="2-6-IPC-transact"><a href="#2-6-IPC-transact" class="headerlink" title="2.6 IPC::transact"></a>2.6 IPC::transact</h3><p>[-&gt;IPCThreadState.cpp]</p>
<pre><code>status_t IPCThreadState::transact(int32_t handle,
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
    //传输数据，见2.7    
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
<li>errorCheck错误检查</li>
<li>writeTransactionData传输数据</li>
<li>waitForResponse等待响应</li>
</ul>
<h3 id="2-7-IPC-writeTransactionData"><a href="#2-7-IPC-writeTransactionData" class="headerlink" title="2.7  IPC::writeTransactionData"></a>2.7  IPC::writeTransactionData</h3><p>[-&gt;IPCThreadState.cpp]</p>
<pre><code>status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
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
<li>data_size，binder_transaction的数据大小</li>
<li>data.ptr.buffer，binder_transaction数据的起始地址</li>
<li>offsets_size，记录flat_binder_object结构体的个数</li>
<li>data.ptr.offsets，记录flat_binder_object结构体的数据偏移量</li>
</ul>
<h3 id="2-8-IPC-waitForResponse"><a href="#2-8-IPC-waitForResponse" class="headerlink" title="2.8 IPC::waitForResponse"></a>2.8 IPC::waitForResponse</h3><p>[-&gt;IPCThreadState.cpp]</p>
<pre><code>status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
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
<h3 id="2-9-IPC-talkWithDriver"><a href="#2-9-IPC-talkWithDriver" class="headerlink" title="2.9 IPC::talkWithDriver"></a>2.9 IPC::talkWithDriver</h3><p>[-&gt;IPCThreadState.cpp]</p>
<pre><code>status_t IPCThreadState::talkWithDriver(bool doReceive)
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
<p>进入到Binder Driver在前面ServiceManager注册服务中第四章有详细的描述，这里直接到ServiceManager进程、</p>
<h2 id="三、ServiceManager进程"><a href="#三、ServiceManager进程" class="headerlink" title="三、ServiceManager进程"></a>三、ServiceManager进程</h2><p>从在前面<a href="https://skytoby.github.io/2020/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Binder%E6%9C%BA%E5%88%B62-%E6%B3%A8%E5%86%8C%E6%9C%8D%E5%8A%A1addService/" target="_blank" rel="noopener">ServiceManager注册服务</a>中第二章可以看到servicemanager启动后，循环再binder_loop过程，会调用binder_parse方法</p>
<h3 id="3-1-binder-parse"><a href="#3-1-binder-parse" class="headerlink" title="3.1 binder_parse"></a>3.1 binder_parse</h3><p>[-&gt;servicemanager/binder.c]</p>
<pre><code>int binder_parse(struct binder_state *bs, struct binder_io *bio,
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
<h3 id="3-2-svcmgr-handler"><a href="#3-2-svcmgr-handler" class="headerlink" title="3.2 svcmgr_handler"></a>3.2 svcmgr_handler</h3><p>[-&gt;service_manager.c]</p>
<pre><code>int svcmgr_handler(struct binder_state *bs,
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
    case SVC_MGR_GET_SERVICE:
    case SVC_MGR_CHECK_SERVICE:
        s = bio_get_string16(msg, &len);
        if (s == NULL) {
            return -1;
        }
        handle = do_find_service(s, len, txn-&gt;sender_euid, txn-&gt;sender_pid);
        if (!handle)
            break;
        bio_put_ref(reply, handle);
        return 0;
      ...  
    }

    bio_put_uint32(reply, 0);
    return 0;
}</code></pre>
<h3 id="3-3-do-find-service"><a href="#3-3-do-find-service" class="headerlink" title="3.3 do_find_service"></a>3.3 do_find_service</h3><p>[-&gt;service_manager.c]</p>
<pre><code>uint32_t do_find_service(const uint16_t *s, size_t len, uid_t uid, pid_t spid)
{
    //查找列表
    struct svcinfo *si = find_svc(s, len);

    if (!si || !si-&gt;handle) {
        return 0;
    }

    if (!si-&gt;allow_isolated) {
        // If this service doesn't allow access from isolated processes,
        // then check the uid to see if it is isolated.
        uid_t appid = uid % AID_USER;
        if (appid &gt;= AID_ISOLATED_START && appid &lt;= AID_ISOLATED_END) {
            return 0;
        }
    }

    if (!svc_can_find(s, len, spid, uid)) {
        return 0;
    }

    return si-&gt;handle;
}</code></pre>
<p>这里返回已经注册的服务。</p>
<h3 id="3-4-binder-send-reply"><a href="#3-4-binder-send-reply" class="headerlink" title="3.4 binder_send_reply"></a>3.4 binder_send_reply</h3><p>[-&gt;servicemanager/binder.c]</p>
<pre><code>void binder_send_reply(struct binder_state *bs,
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
}
</code></pre>
<p>将BC_FREE_BUFFER和BC_REPLY发送给binder驱动，向client发送reply。</p>
<h2 id="四、获取服务进程"><a href="#四、获取服务进程" class="headerlink" title="四、获取服务进程"></a>四、获取服务进程</h2><p>上面2.2节中transact之后，执行reply的readStrongBinder操作。</p>
<h3 id="4-1-readStrongBinder"><a href="#4-1-readStrongBinder" class="headerlink" title="4.1 readStrongBinder"></a>4.1 readStrongBinder</h3><p>[-&gt;Parcel.java]</p>
<pre><code>/**
 * Read an object from the parcel at the current dataPosition().
 */
public final IBinder readStrongBinder() {
    //native调用，见4.2节
    return nativeReadStrongBinder(mNativePtr);
}</code></pre>
<h3 id="4-2-android-os-Parcel-readStrongBinder"><a href="#4-2-android-os-Parcel-readStrongBinder" class="headerlink" title="4.2 android_os_Parcel_readStrongBinder"></a>4.2 android_os_Parcel_readStrongBinder</h3><p>[-&gt;android_os_Parcel.cpp]</p>
<pre><code>static jobject android_os_Parcel_readStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr)
{
    Parcel* parcel = reinterpret_cast&lt;Parcel*&gt;(nativePtr);
    if (parcel != NULL) {
        //返回IBinder
        return javaObjectForIBinder(env, parcel-&gt;readStrongBinder());
    }
    return NULL;
}</code></pre>
<h3 id="4-3-readStrongBinder-Java"><a href="#4-3-readStrongBinder-Java" class="headerlink" title="4.3 readStrongBinder(Java)"></a>4.3 readStrongBinder(Java)</h3><p>[-&gt;Parcel.java]</p>
<pre><code>public final IBinder readStrongBinder() {
     return nativeReadStrongBinder(mNativePtr);
 }
 </code></pre>
<p>[-&gt;android_os_Parcel.cpp]</p>
<pre><code>static jobject android_os_Parcel_readStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr)
{
    //指向Parcel内存
    Parcel* parcel = reinterpret_cast&lt;Parcel*&gt;(nativePtr);
    if (parcel != NULL) {
        return javaObjectForIBinder(env, parcel-&gt;readStrongBinder());
    }
    return NULL;
}</code></pre>
<h3 id="4-4-readStrongBinder-C"><a href="#4-4-readStrongBinder-C" class="headerlink" title="4.4 readStrongBinder(C++)"></a>4.4 readStrongBinder(C++)</h3><p>[-&gt;Parcel.cpp]</p>
<pre><code>sp&lt;IBinder&gt; Parcel::readStrongBinder() const
{
    sp&lt;IBinder&gt; val; //sp为智能指针，strong point
    // Note that a lot of code in Android reads binders by hand with this
    // method, and that code has historically been ok with getting nullptr
    // back (while ignoring error codes).
    readNullableStrongBinder(&val);
    return val;
}

status_t Parcel::readNullableStrongBinder(sp&lt;IBinder&gt;* val) const
{
    //见4.4.1节
    return unflatten_binder(ProcessState::self(), *this, val);
}</code></pre>
<h4 id="4-4-1-unflatten-binder"><a href="#4-4-1-unflatten-binder" class="headerlink" title="4.4.1 unflatten_binder"></a>4.4.1 unflatten_binder</h4><p>[-&gt;Parcel.cpp]</p>
<pre><code>status_t unflatten_binder(const sp&lt;ProcessState&gt;& proc,
    const Parcel& in, sp&lt;IBinder&gt;* out)
{
    const flat_binder_object* flat = in.readObject&lt;flat_binder_object&gt;();

    if (flat) {
        switch (flat-&gt;hdr.type) {
            //当请求服务的进程和服务进程属于同一个进程
            case BINDER_TYPE_BINDER:
                *out = reinterpret_cast&lt;IBinder*&gt;(flat-&gt;cookie);
                return finish_unflatten_binder(NULL, *flat, in);
            //当请求服务的进程与服务属于不同的进程
            case BINDER_TYPE_HANDLE:
                *out = proc-&gt;getStrongProxyForHandle(flat-&gt;handle);
                return finish_unflatten_binder(
                    static_cast&lt;BpHwBinder*&gt;(out-&gt;get()), *flat, in);
        }
    }
    return BAD_TYPE;
}</code></pre>
<h4 id="4-4-2-getStrongProxyForHandle"><a href="#4-4-2-getStrongProxyForHandle" class="headerlink" title="4.4.2 getStrongProxyForHandle"></a>4.4.2 getStrongProxyForHandle</h4><p>[-&gt;ProcessState.cpp]</p>
<pre><code>sp&lt;IBinder&gt; ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp&lt;IBinder&gt; result;

    AutoMutex _l(mLock);

    handle_entry* e = lookupHandleLocked(handle);

    if (e != NULL) {
        // We need to create a new BpHwBinder if there isn't currently one, OR we
        // are unable to acquire a weak reference on this current one.  See comment
        // in getWeakProxyForHandle() for more info about this.
        IBinder* b = e-&gt;binder;
        if (b == NULL || !e-&gt;refs-&gt;attemptIncWeak(this)) {
            //当handle值所对应的IBinder不存在或者弱引用无效时，则创建BpHwBinder对象
            b = new BpHwBinder(handle);
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
<p>返回BpHwBinder代理对象。</p>
<h4 id="4-4-3-lookupHandleLocked"><a href="#4-4-3-lookupHandleLocked" class="headerlink" title="4.4.3 lookupHandleLocked"></a>4.4.3 lookupHandleLocked</h4><p>[-&gt;ProcessState.cpp]</p>
<pre><code>ProcessState::handle_entry* ProcessState::lookupHandleLocked(int32_t handle)
{
    const size_t N=mHandleToObject.size();
     //当handle值大于mHandleToObject时才进入
    if (N &lt;= (size_t)handle) {
        handle_entry e;
        e.binder = NULL;
        e.refs = NULL;
        //从mHandleToObject的第N个位置开始插入handler+1-n e到队列中
        status_t err = mHandleToObject.insertAt(e, N, handle+1-N);
        if (err &lt; NO_ERROR) return NULL;
    }
    return &mHandleToObject.editItemAt(handle);
}</code></pre>
<h4 id="4-4-4-finish-unflatten-binder"><a href="#4-4-4-finish-unflatten-binder" class="headerlink" title="4.4.4 finish_unflatten_binder"></a>4.4.4 finish_unflatten_binder</h4><p>[-&gt;Parcel.cpp]</p>
<pre><code>inline static status_t finish_unflatten_binder(
    BpHwBinder* /*proxy*/, const flat_binder_object& /*flat*/,
    const Parcel& /*in*/)
{
    return NO_ERROR;
}</code></pre>
<h3 id="4-5-javaObjectForIBinder"><a href="#4-5-javaObjectForIBinder" class="headerlink" title="4.5  javaObjectForIBinder"></a>4.5  javaObjectForIBinder</h3><p>[-&gt;android_util_Binder.cpp]</p>
<pre><code>jobject javaObjectForIBinder(JNIEnv* env, const sp&lt;IBinder&gt;& val)
{
    if (val == NULL) return NULL;
    //如果是JavaBBinder则直接返回
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
    //返回服务的代理,见4.6节
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
}</code></pre>
<p>gBinderProxyOffsets作为全局变量，其初始化在int_register_android_os_BinderProxy方法中。</p>
<pre><code>static int int_register_android_os_BinderProxy(JNIEnv* env)
{
    jclass clazz = FindClassOrDie(env, "java/lang/Error");
    gErrorOffsets.mClass = MakeGlobalRefOrDie(env, clazz);

    clazz = FindClassOrDie(env, kBinderProxyPathName);
    //初始化gBinderProxyOffsets值
    gBinderProxyOffsets.mClass = MakeGlobalRefOrDie(env, clazz);
    gBinderProxyOffsets.mGetInstance = GetStaticMethodIDOrDie(env, clazz, "getInstance",
            "(JJ)Landroid/os/BinderProxy;");
    gBinderProxyOffsets.mSendDeathNotice = GetStaticMethodIDOrDie(env, clazz, "sendDeathNotice",
            "(Landroid/os/IBinder$DeathRecipient;)V");
    gBinderProxyOffsets.mNativeData = GetFieldIDOrDie(env, clazz, "mNativeData", "J");

    clazz = FindClassOrDie(env, "java/lang/Class");
    gClassOffsets.mGetName = GetMethodIDOrDie(env, clazz, "getName", "()Ljava/lang/String;");

    return RegisterMethodsOrDie(
        env, kBinderProxyPathName,
        gBinderProxyMethods, NELEM(gBinderProxyMethods));
}</code></pre>
<p>env-&gt;CallStaticObjectMethod，jni调用BinderProxy中的getInstance方法。</p>
<h3 id="4-6-getInstance"><a href="#4-6-getInstance" class="headerlink" title="4.6 getInstance"></a>4.6 getInstance</h3><p>[-&gt;BinderProxy.java]</p>
<pre><code>private static BinderProxy getInstance(long nativeData, long iBinder) {
       BinderProxy result;
       synchronized (sProxyMap) {
           try {
               result = sProxyMap.get(iBinder);
               //如果已经存在则直接返回，否则新建BinderProxy
               if (result != null) {
                   return result;
               }
               result = new BinderProxy(nativeData);
           } catch (Throwable e) {
               // We're throwing an exception (probably OOME); don't drop nativeData.
               NativeAllocationRegistry.applyFreeFunction(NoImagePreloadHolder.sNativeFinalizer,
                       nativeData);
               throw e;
           }
           NoImagePreloadHolder.sRegistry.registerNativeAllocation(result, nativeData);
           // The registry now owns nativeData, even if registration threw an exception.
           sProxyMap.set(iBinder, result);
       }
       return result;
   }</code></pre>
<p>由上面分析， <strong>reply.readStrongBinder();等价于new BinderProxy</strong>即SMP.getService等价于new BinderProxy</p>
<h2 id="五、总结"><a href="#五、总结" class="headerlink" title="五、总结"></a>五、总结</h2><p><strong>getService的核心过程</strong></p>
<pre><code>public IBinder getService(String name) throws RemoteException {
       //这里需要将Java层的Parcel转换成Native层的Parcel
       Parcel data = Parcel.obtain();
       Parcel reply = Parcel.obtain();
       data.writeInterfaceToken(IServiceManager.descriptor);
       data.writeString(name);
       //与binder驱动交互
       mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0);
       //返回代理对象，为new BinderProxy(nativeData)
       IBinder binder = javaObjectForIBinder(env, new BpHwBinder(handle));
       reply.recycle();
       data.recycle();
       return binder;
   }</code></pre>
<p>获取服务通过BpBinder发送GET_SERVICE_TRANSACTION命令和Binder驱动层进行数据交互，javaObjectForIBinder的作用是创建 BinderProxy对象，并将BpHwBinder对象的地址保存到BinderProxy的mObjects中,最后返回代理对象。</p>
<h2 id="附录"><a href="#附录" class="headerlink" title="附录"></a>附录</h2><p>源码路径</p>
<pre><code>frameworks/base/core/java/android/os/ServiceManager.java
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
kernel/drivers/android/binder.c
</code></pre>
      