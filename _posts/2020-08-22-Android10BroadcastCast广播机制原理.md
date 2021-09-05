 ---
layout:     post
title:      Android10 BroadcastCast广播机制原理
subtitle:   广播（BroadcastCast）用于进程/线程间的通信，广播有发送广播和接收广播两部分组成
date:       2020-08-22
author:     duguma
header-img: img/article-bg.jpg
top: true
catalog: true
tags:
    - Android10
    - Android
	- 组件学习
 ---

<h2><a id="_3"></a>一、概述</h2> 
<p>广播&#xff08;BroadcastCast&#xff09;用于进程/线程间的通信&#xff0c;广播有发送广播和接收广播两部分组成&#xff0c;其中广播接收者BroadcastReceiver是四大组件之一。</p> 
<p>BroadcastReceiver分为两类&#xff1a;</p> 
<ul><li> <p>静态广播&#xff1a;通过AndroidManifeset.xml的标签来注册BroadcastReceiver。</p> </li><li> <p>动态广播&#xff1a;通过AMS.registeredReceiver方式注册BroadcastReceiver&#xff0c;动态注册相对静态注册更加的灵活&#xff0c;在不需要时通过unregisteredReceiver来取消注册。</p> </li></ul> 
<p>从广播的发送方式分为四种&#xff1a;</p> 
<ul><li> <p>普通广播&#xff1a;完全异步的广播&#xff0c;在广播发出之后&#xff0c;所有的广播接收器几乎在同一时刻接收到这条广播消息&#xff0c;接收的先后顺序是随机的。通过Context.sendBroadcast发送。</p> </li><li> <p>有序广播&#xff1a;同步执行的广播。在广播发出去之后&#xff0c;同一时刻只有一个广播接收器能够收到这条广播消息&#xff0c;当这个广播接收器中的逻辑处理完成后&#xff0c;广播才可以继续传递。这种广播的接收顺序通过优先级(priority)设置&#xff0c;高的优先级先会收到广播。有序广播可以被接收者截断&#xff0c;使得后面的广播无法收到它。通过Context.sendOrderedBroadcast发送。</p> </li><li> <p>粘性广播&#xff1a;这种广播会一直滞留&#xff0c;当有匹配该广播的接收器被注册后&#xff0c;该接收器就会收到这个广播。粘性广播如果被销毁&#xff0c;下一次重建时会重新接收到消息数据。这种方式一般用来确保重要状态改变后的信息被持久的保存&#xff0c;并且能随时广播给新的广播接收器&#xff0c;比如&#xff1a;耗电量的改变。通过Context.sendStickyBroadcast发送。Android系统已经 &#64;Deprecated该广播方式。</p> </li><li> <p>本地广播&#xff1a;发送处理的广播只能够在应用程序的内部进行传递&#xff0c;并且广播接收器也只能接收本应用程序发出的广播。通过LocalBroadcastManager.sendBroadcast发送。&#xff08;基本原理是Handler&#xff0c;所以在AMS中没有本地广播的处理&#xff09;</p> </li></ul> 
<p>广播在系统中通过BroadcastRecord来记录。</p> 
<pre><code>/**
 * An active intent broadcast.
 */
final class BroadcastRecord extends Binder {
    final Intent intent;    // the original intent that generated us
    final ComponentName targetComp; // original component name set on the intent
    final ProcessRecord callerApp; // process that sent this
    final String callerPackage; // who sent this
    final int callingPid;   // the pid of who sent this
    final int callingUid;   // the uid of who sent this
    final boolean callerInstantApp; // caller is an Instant App?
    final boolean ordered;  // serialize the send to receivers?
    final boolean sticky;   // originated from existing sticky data?
    final boolean initialSticky; // initial broadcast from register to sticky?
    final int userId;       // user id this broadcast was for
    final String resolvedType; // the resolved data type
    final String[] requiredPermissions; // permissions the caller has required
    final int appOp;        // an app op that is associated with this broadcast
    final BroadcastOptions options; // BroadcastOptions supplied by caller
    //包括动态注册的BroadcastFilter和静态注册的ResolveInfo
    final List receivers;   // contains BroadcastFilter and ResolveInfo
    //广播的分发状态
    final int[] delivery;   // delivery state of each receiver
    IIntentReceiver resultTo; // who receives final result if non-null
    
    //入队列时间
    long enqueueClockTime;  // the clock time the broadcast was enqueued
    //分发时间
    long dispatchTime;      // when dispatch started on this set of receivers
    //分发时间
    long dispatchClockTime; // the clock time the dispatch started
    //接收时间
    long receiverTime;      // when current receiver started for timeouts.
    //广播完成时间
    long finishTime;        // when we finished the broadcast.
  
}
</code></pre> 
<h2><a id="_65"></a>二、注册广播</h2> 
<p>注册广播一般可以在Activity/Service中调用registerReceiver方法&#xff0c;而这两者间接集成Context&#xff0c;其实现是ContextImpl。</p> 
<h3><a id="21_CIregisterReceiver_69"></a>2.1 CI.registerReceiver</h3> 
<p>[-&gt;ContextImpl.java]</p> 
<pre><code>    &#64;Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
        return registerReceiver(receiver, filter, null, null);
    }
     &#64;Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
            String broadcastPermission, Handler scheduler) {
        return registerReceiverInternal(receiver, getUserId(),
                filter, broadcastPermission, scheduler, getOuterContext(), 0);
    }
</code></pre> 
<p>其中broadcastPermission表示广播的权限&#xff0c;scheduler表示接收广播时onReceive执行的线程&#xff0c;当scheduler&#61;&#61;null时代表在主线程中执行&#xff0c;大部分情况是不会指令scheduler。getOuterContext可以获取最外层调用者。</p> 
<h3><a id="22_CIregisterReceiverInternal_88"></a>2.2 CI.registerReceiverInternal</h3> 
<p>[-&gt;ContextImpl.java]</p> 
<pre><code> private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
            IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context, int flags) {
        IIntentReceiver rd &#61; null;
        if (receiver !&#61; null) {
            if (mPackageInfo !&#61; null &amp;&amp; context !&#61; null) {
                if (scheduler &#61;&#61; null) {
                    //主线程Handler赋予scheduler
                    scheduler &#61; mMainThread.getHandler();
                }
                //见2.3节&#xff0c;获取IIntentReceiver
                rd &#61; mPackageInfo.getReceiverDispatcher(
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);
            } else {
                if (scheduler &#61;&#61; null) {
                    scheduler &#61; mMainThread.getHandler();
                }
                //context为空&#xff0c;新建ReceiverDispatcher对象&#xff08;见2.3.1节&#xff09;&#xff0c;并获取IIntentReceiver
                rd &#61; new LoadedApk.ReceiverDispatcher(
                        receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {
            //通过Binder调用AMS.registerReceiver
            final Intent intent &#61; ActivityManager.getService().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName, rd, filter,
                    broadcastPermission, userId, flags);
            if (intent !&#61; null) {
                intent.setExtrasClassLoader(getClassLoader());
                intent.prepareToEnterProcess();
            }
            return intent;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

</code></pre> 
<h3><a id="23_LAgetReceiverDispatcher_133"></a>2.3 LA.getReceiverDispatcher</h3> 
<p>[-&gt;LoadedApk.java]</p> 
<pre><code> public IIntentReceiver getReceiverDispatcher(BroadcastReceiver r,
            Context context, Handler handler,
            Instrumentation instrumentation, boolean registered) {
        synchronized (mReceivers) {
            LoadedApk.ReceiverDispatcher rd &#61; null;
            ArrayMap&lt;BroadcastReceiver, LoadedApk.ReceiverDispatcher&gt; map &#61; null;
            //已经注册
            if (registered) {
                map &#61; mReceivers.get(context);
                if (map !&#61; null) {
                    rd &#61; map.get(r);
                }
            }
            if (rd &#61;&#61; null) {
                //广播分发者为空&#xff0c;创建ReceiverDispatcher
                rd &#61; new ReceiverDispatcher(r, context, handler,
                        instrumentation, registered);
                if (registered) {
                    if (map &#61;&#61; null) {
                        map &#61; new ArrayMap&lt;BroadcastReceiver, LoadedApk.ReceiverDispatcher&gt;();
                        mReceivers.put(context, map);
                    }
                    map.put(r, rd);
                }
            } else {
                //验证广播分发者的context和handler是否一致
                rd.validate(context, handler);
            }
            rd.mForgotten &#61; false;
            return rd.getIIntentReceiver();
        }
    }
</code></pre> 
<p>mReceivers定义为final ArrayMap&lt;Context, ArrayMap&lt;BroadcastReceiver, ReceiverDispatcher&gt;&gt; mReceivers&#xff1b;</p> 
<p>mReceivers是以Context为key,以ArrayMap为value的ArrayMap。</p> 
<h4><a id="231_ReceiverDispatcher_176"></a>2.3.1 创建ReceiverDispatcher</h4> 
<p>[-&gt;LoadedApk.java]</p> 
<pre><code> ReceiverDispatcher(BroadcastReceiver receiver, Context context,
                Handler activityThread, Instrumentation instrumentation,
                boolean registered) {
            if (activityThread &#61;&#61; null) {
                throw new NullPointerException(&#34;Handler must not be null&#34;);
            }

            mIIntentReceiver &#61; new InnerReceiver(this, !registered);
            mReceiver &#61; receiver;
            mContext &#61; context;
            mActivityThread &#61; activityThread;
            mInstrumentation &#61; instrumentation;
            mRegistered &#61; registered;
            mLocation &#61; new IntentReceiverLeaked(null);
            mLocation.fillInStackTrace();
        }
</code></pre> 
<p>mActivityThread为前面传过来的Handler。</p> 
<h4><a id="232_InnerReceiver_201"></a>2.3.2 创建InnerReceiver</h4> 
<p>[-&gt;LoadedApk.java]</p> 
<pre><code>  final static class InnerReceiver extends IIntentReceiver.Stub {
            final WeakReference&lt;LoadedApk.ReceiverDispatcher&gt; mDispatcher;
            final LoadedApk.ReceiverDispatcher mStrongRef;

            InnerReceiver(LoadedApk.ReceiverDispatcher rd, boolean strong) {
                //弱引用
                mDispatcher &#61; new WeakReference&lt;LoadedApk.ReceiverDispatcher&gt;(rd);
                mStrongRef &#61; strong ? rd : null;
            }
            ...
}
</code></pre> 
<p>InnerReceiver继承于IIntentReceiver.Stub&#xff0c;是一个服务器端&#xff0c;广播分发者通过getReceiverDispatcher可以获取该Binder服务端对象InnerReceiver&#xff0c;用于IPC通信。</p> 
<h3><a id="24_AMSregisterReceiver_221"></a>2.4 AMS.registerReceiver</h3> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code> public Intent registerReceiver(IApplicationThread caller, String callerPackage,
            IIntentReceiver receiver, IntentFilter filter, String permission, int userId,
            int flags) {
        enforceNotIsolatedCaller(&#34;registerReceiver&#34;);
        ArrayList&lt;Intent&gt; stickyIntents &#61; null;
        ProcessRecord callerApp &#61; null;
        final boolean visibleToInstantApps
                &#61; (flags &amp; Context.RECEIVER_VISIBLE_TO_INSTANT_APPS) !&#61; 0;
        int callingUid;
        int callingPid;
        boolean instantApp;
        synchronized(this) {
            if (caller !&#61; null) {
                //查询调用者的进程信息
                callerApp &#61; getRecordForAppLocked(caller);
                if (callerApp &#61;&#61; null) {
                    throw new SecurityException(
                            &#34;Unable to find app for caller &#34; &#43; caller
                            &#43; &#34; (pid&#61;&#34; &#43; Binder.getCallingPid()
                            &#43; &#34;) when registering receiver &#34; &#43; receiver);
                }
                if (callerApp.info.uid !&#61; SYSTEM_UID &amp;&amp;
                        !callerApp.pkgList.containsKey(callerPackage) &amp;&amp;
                        !&#34;android&#34;.equals(callerPackage)) {
                    throw new SecurityException(&#34;Given caller package &#34; &#43; callerPackage
                            &#43; &#34; is not running in process &#34; &#43; callerApp);
                }
                callingUid &#61; callerApp.info.uid;
                callingPid &#61; callerApp.pid;
            } else {
                callerPackage &#61; null;
                callingUid &#61; Binder.getCallingUid();
                callingPid &#61; Binder.getCallingPid();
            }
            //是否是即时应用
            instantApp &#61; isInstantApp(callerApp, callerPackage, callingUid);
            userId &#61; mUserController.handleIncomingUser(callingPid, callingUid, userId, true,
                    ALLOW_FULL_ONLY, &#34;registerReceiver&#34;, callerPackage);
            //获取InterFilter中的Action
            Iterator&lt;String&gt; actions &#61; filter.actionsIterator();
            if (actions &#61;&#61; null) {
                ArrayList&lt;String&gt; noAction &#61; new ArrayList&lt;String&gt;(1);
                noAction.add(null);
                actions &#61; noAction.iterator();
            }

            // Collect stickies of users
            int[] userIds &#61; { UserHandle.USER_ALL, UserHandle.getUserId(callingUid) };
            while (actions.hasNext()) {
                String action &#61; actions.next();
                for (int id : userIds) {
                    //从mStickyBroadcasts中查看用户的sticky intent
                    ArrayMap&lt;String, ArrayList&lt;Intent&gt;&gt; stickies &#61; mStickyBroadcasts.get(id);
                    if (stickies !&#61; null) {
                        ArrayList&lt;Intent&gt; intents &#61; stickies.get(action);
                        if (intents !&#61; null) {
                            if (stickyIntents &#61;&#61; null) {
                                stickyIntents &#61; new ArrayList&lt;Intent&gt;();
                            }
                            //将sticky intent加入队列
                            stickyIntents.addAll(intents);
                        }
                    }
                }
            }
        }

        ArrayList&lt;Intent&gt; allSticky &#61; null;
        if (stickyIntents !&#61; null) {
            final ContentResolver resolver &#61; mContext.getContentResolver();
            // Look for any matching sticky broadcasts...
            for (int i &#61; 0, N &#61; stickyIntents.size(); i &lt; N; i&#43;&#43;) {
                Intent intent &#61; stickyIntents.get(i);
                //即时应用跳过
                // Don&#39;t provided intents that aren&#39;t available to instant apps.
                if (instantApp &amp;&amp;
                        (intent.getFlags() &amp; Intent.FLAG_RECEIVER_VISIBLE_TO_INSTANT_APPS) &#61;&#61; 0) {
                    continue;
                }
                //查询匹配到的sticky广播&#xff0c;见2.4.2节
                // If intent has scheme &#34;content&#34;, it will need to acccess
                // provider that needs to lock mProviderMap in ActivityThread
                // and also it may need to wait application response, so we
                // cannot lock ActivityManagerService here.
                if (filter.match(resolver, intent, true, TAG) &gt;&#61; 0) {
                    if (allSticky &#61;&#61; null) {
                        allSticky &#61; new ArrayList&lt;Intent&gt;();
                    }
                    //匹配成功&#xff0c;则将该intent添加到allSticky队列
                    allSticky.add(intent);
                }
            }
        }
        //返回第一个stick intent
        // The first sticky in the list is returned directly back to the client.
        Intent sticky &#61; allSticky !&#61; null ? allSticky.get(0) : null;
        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, &#34;Register receiver &#34; &#43; filter &#43; &#34;: &#34; &#43; sticky);
        if (receiver &#61;&#61; null) {
            return sticky;
        }

        synchronized (this) {
            if (callerApp !&#61; null &amp;&amp; (callerApp.thread &#61;&#61; null
                    || callerApp.thread.asBinder() !&#61; caller.asBinder())) {
                // Original caller already died
                //调用者已经死亡
                return null;
            }
            ReceiverList rl &#61; mRegisteredReceivers.get(receiver.asBinder());
            if (rl &#61;&#61; null) {
                //对于没有注册的广播&#xff0c;则创建接收者队列
                rl &#61; new ReceiverList(this, callerApp, callingPid, callingUid,
                        userId, receiver);
                if (rl.app !&#61; null) {
                    //每个应用最多只能注册1000个接收者
                    final int totalReceiversForApp &#61; rl.app.receivers.size();
                    if (totalReceiversForApp &gt;&#61; MAX_RECEIVERS_ALLOWED_PER_APP) {
                        throw new IllegalStateException(&#34;Too many receivers, total of &#34;
                                &#43; totalReceiversForApp &#43; &#34;, registered for pid: &#34;
                                &#43; rl.pid &#43; &#34;, callerPackage: &#34; &#43; callerPackage);
                    }
                    //新创建的接收者队列&#xff0c;添加到已注册队列
                    rl.app.receivers.add(rl);
                } else {
                    try {
                        //服务停止后重连
                        receiver.asBinder().linkToDeath(rl, 0);
                    } catch (RemoteException e) {
                        return sticky;
                    }
                    rl.linkedToDeath &#61; true;
                }
                mRegisteredReceivers.put(receiver.asBinder(), rl);
            } else if (rl.uid !&#61; callingUid) {
                throw new IllegalArgumentException(
                        &#34;Receiver requested to register for uid &#34; &#43; callingUid
                        &#43; &#34; was previously registered for uid &#34; &#43; rl.uid
                        &#43; &#34; callerPackage is &#34; &#43; callerPackage);
            } else if (rl.pid !&#61; callingPid) {
                throw new IllegalArgumentException(
                        &#34;Receiver requested to register for pid &#34; &#43; callingPid
                        &#43; &#34; was previously registered for pid &#34; &#43; rl.pid
                        &#43; &#34; callerPackage is &#34; &#43; callerPackage);
            } else if (rl.userId !&#61; userId) {
                throw new IllegalArgumentException(
                        &#34;Receiver requested to register for user &#34; &#43; userId
                        &#43; &#34; was previously registered for user &#34; &#43; rl.userId
                        &#43; &#34; callerPackage is &#34; &#43; callerPackage);
            }
            //创建BroadcastFilter队列&#xff0c;并添加到接收者队列
            BroadcastFilter bf &#61; new BroadcastFilter(filter, rl, callerPackage,
                    permission, callingUid, userId, instantApp, visibleToInstantApps);
            if (rl.containsFilter(filter)) {
                Slog.w(TAG, &#34;Receiver with filter &#34; &#43; filter
                        &#43; &#34; already registered for pid &#34; &#43; rl.pid
                        &#43; &#34;, callerPackage is &#34; &#43; callerPackage);
            } else {
                rl.add(bf);
                if (!bf.debugCheck()) {
                    Slog.w(TAG, &#34;&#61;&#61;&gt; For Dynamic broadcast&#34;);
                }
                //新创建的广播过滤对象&#xff0c;添加到mReceiverResolver队列
                mReceiverResolver.addFilter(bf);
            }
            //所有匹配该filter的stick广播执行入队操作
            // Enqueue broadcasts for all existing stickies that match
            // this filter.
            if (allSticky !&#61; null) {
                ArrayList receivers &#61; new ArrayList();
                receivers.add(bf);

                final int stickyCount &#61; allSticky.size();
                for (int i &#61; 0; i &lt; stickyCount; i&#43;&#43;) {
                    Intent intent &#61; allSticky.get(i);
                    //根据intent返回前台或者后台广播队列&#xff0c;见2.4.3节
                    BroadcastQueue queue &#61; broadcastQueueForIntent(intent);
                    BroadcastRecord r &#61; new BroadcastRecord(queue, intent, null,
                            null, -1, -1, false, null, null, OP_NONE, null, receivers,
                            null, 0, null, null, false, true, true, -1);
                    //该广播加入到并行队列
                    queue.enqueueParallelBroadcastLocked(r);
                    //调度广播&#xff0c;发送BROADCAST_INTENT_MSG消息&#xff0c;触发处理广播
                    queue.scheduleBroadcastsLocked();
                }
            }

            return sticky;
        }
    }

</code></pre> 
<p>mReceiverResolver记录着所有已经注册的广播&#xff0c;是以receiver IBinder为key&#xff0c; ReceiverList为value的ArrayMap。</p> 
<p>在BroadcastQueue中有两个广播队列mParallelBroadcasts、mOrderedBroadcasts&#xff0c;类型为ArrayList</p> 
<ul><li> <p>mParallelBroadcasts&#xff1a;并行广播队列&#xff0c;可以立刻执行&#xff0c;而无需等待另一个广播运行完成&#xff0c;该队列只允许动态已注册的广播&#xff0c;从而避免发生同时拉起大量进程来执行广播&#xff0c;前台和后台的广播分别位于独立的队列。</p> </li><li> <p>mOrderedBroadcasts&#xff1a;有序广播&#xff0c;同一时间只允许执行一个广播&#xff0c;该队列头部的广播就是活动广播&#xff0c;其他广播必须等待该广播结束才能运行&#xff0c;也是独立区别前台和后台的广播。</p> </li></ul> 
<h4><a id="241_AMSgetRecordForAppLocked_426"></a>2.4.1 AMS.getRecordForAppLocked</h4> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code>  ProcessRecord getRecordForAppLocked(IApplicationThread thread) {
        if (thread &#61;&#61; null) {
            return null;
        }
        //从mLruProcesses队列中获取
        int appIndex &#61; getLRURecordIndexForAppLocked(thread);
        if (appIndex &gt;&#61; 0) {
            return mLruProcesses.get(appIndex);
        }
        //mLruProcesses不存在&#xff0c;再检查一遍
        // Validation: if it isn&#39;t in the LRU list, it shouldn&#39;t exist, but let&#39;s
        // double-check that.
        final IBinder threadBinder &#61; thread.asBinder();
        final ArrayMap&lt;String, SparseArray&lt;ProcessRecord&gt;&gt; pmap &#61; mProcessNames.getMap();
        for (int i &#61; pmap.size()-1; i &gt;&#61; 0; i--) {
            final SparseArray&lt;ProcessRecord&gt; procs &#61; pmap.valueAt(i);
            for (int j &#61; procs.size()-1; j &gt;&#61; 0; j--) {
                final ProcessRecord proc &#61; procs.valueAt(j);
                if (proc.thread !&#61; null &amp;&amp; proc.thread.asBinder() &#61;&#61; threadBinder) {
                    Slog.wtf(TAG, &#34;getRecordForApp: exists in name list but not in LRU list: &#34;
                            &#43; proc);
                    return proc;
                }
            }
        }

        return null;
    }
</code></pre> 
<p>mLruProcesses 定义为ArrayList mLruProcesses &#xff0c;ProcessRecord对象中有一个IApplicationThread字段&#xff0c;根据该字段来查找对应的ProcessRecord。</p> 
<h4><a id="243_IntentFiltermatch_463"></a>2.4.3 IntentFilter.match</h4> 
<p>[-&gt;IntentFilter.java]</p> 
<pre><code>    public final int match(ContentResolver resolver, Intent intent,
            boolean resolve, String logTag) {
        String type &#61; resolve ? intent.resolveType(resolver) : intent.getType();
        return match(intent.getAction(), type, intent.getScheme(),
                     intent.getData(), intent.getCategories(), logTag);
    }
     public final int match(String action, String type, String scheme,
            Uri data, Set&lt;String&gt; categories, String logTag) {
        //不存在匹配的action&#xff0c;存在即可以
        if (action !&#61; null &amp;&amp; !matchAction(action)) {
            if (false) Log.v(
                logTag, &#34;No matching action &#34; &#43; action &#43; &#34; for &#34; &#43; this);
            return NO_MATCH_ACTION;
        }
        //不存在匹配的type或data
        int dataMatch &#61; matchData(type, scheme, data);
        if (dataMatch &lt; 0) {
            if (false) {
                if (dataMatch &#61;&#61; NO_MATCH_TYPE) {
                    Log.v(logTag, &#34;No matching type &#34; &#43; type
                          &#43; &#34; for &#34; &#43; this);
                }
                if (dataMatch &#61;&#61; NO_MATCH_DATA) {
                    Log.v(logTag, &#34;No matching scheme/path &#34; &#43; data
                          &#43; &#34; for &#34; &#43; this);
                }
            }
            return dataMatch;
        }
        //不存在匹配的category&#xff0c;需要全部匹配
        String categoryMismatch &#61; matchCategories(categories);
        if (categoryMismatch !&#61; null) {
            if (false) {
                Log.v(logTag, &#34;No matching category &#34; &#43; categoryMismatch &#43; &#34; for &#34; &#43; this);
            }
            return NO_MATCH_CATEGORY;
        }

        // It would be nice to treat container activities as more
        // important than ones that can be embedded, but this is not the way...
        if (false) {
            if (categories !&#61; null) {
                dataMatch -&#61; mCategories.size() - categories.size();
            }
        }

        return dataMatch;
    }
</code></pre> 
<p>该方法用于匹配Intent的数据是否成功&#xff0c;匹配的数据包括action,type,data,category四项&#xff0c;任何一项匹配不成功都会失败。</p> 
<h4><a id="243_AMSbroadcastQueueForIntent_520"></a>2.4.3 AMS.broadcastQueueForIntent</h4> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code>BroadcastQueue broadcastQueueForIntent(Intent intent) {
        final boolean isFg &#61; (intent.getFlags() &amp; Intent.FLAG_RECEIVER_FOREGROUND) !&#61; 0;
        if (DEBUG_BROADCAST_BACKGROUND) Slog.i(TAG_BROADCAST,
                &#34;Broadcast intent &#34; &#43; intent &#43; &#34; on &#34;
                &#43; (isFg ? &#34;foreground&#34; : &#34;background&#34;) &#43; &#34; queue&#34;);
        return (isFg) ? mFgBroadcastQueue : mBgBroadcastQueue;
    }
</code></pre> 
<p>broadcastQueueForIntent通过判断intent中是否包含FLAG_RECEIVER_FOREGROUND来决定是前台广播还是后台广播</p> 
<h3><a id="25___536"></a>2.5 小结</h3> 
<p>注册广播&#xff1a;</p> 
<p>1.注册广播的参数为广播BroadcastReceiver和过滤添加IntentFilter&#xff1b;</p> 
<p>2.创建对象LoadedApk.ReceiverDispatcher.InnerReceiver,该对象集成于IIntentReceiver.Stub&#xff1b;</p> 
<p>3.通过AMS把当前进程的ApplicationThread和InnerReceiver对象的代理类&#xff0c;注册到systemserver进程&#xff1b;</p> 
<p>4.当广播receiver没有注册时&#xff0c;则创建广播接收者队列ReceiverList,该对象集成于ArrayList&#xff0c;并添加到AMS.mRegisteredReceivers(已注册广播队列)&#xff1b;</p> 
<p>5.创建BroadcastFilter队列&#xff0c;并添加到AMS.mReceiverResolver</p> 
<p>6.将BroadcastFilter添加到广播接收者的ReceiverList</p> 
<p>当注册的广播为Sticky广播&#xff1a;</p> 
<p>1.创建BroadcastRecord&#xff0c;并添加到BroadcastQueue中的mParallelBroadcasts&#xff0c;注册后&#xff0c;调用AMS处理该广播</p> 
<p>2.根据注册的intent中是否包含FLAG_RECEIVER_FOREGROUND,包含则是mFgBroadcastQueue队列&#xff0c;否则为mBgBroadcastQueue队列。</p> 
<h2><a id="_558"></a>三、发送广播</h2> 
<p>和注册广播一样&#xff0c;最后调用的是ContextImpl.sendBroadcast</p> 
<h3><a id="31_CIsendBroadcast_562"></a>3.1 CI.sendBroadcast</h3> 
<p>[-&gt;ContextImpl.java]</p> 
<pre><code>  &#64;Override
    public void sendBroadcast(Intent intent) {
        //UID如果是SYSTEM_UID则提示警告信息
        //&#34;Calling a method in the system process without a qualified user:&#34;&#34;
        warnIfCallingFromSystemProcess();
        String resolvedType &#61; intent.resolveTypeIfNeeded(getContentResolver());
        try {
            intent.prepareToLeaveProcess(this);
            //通过binder方式调用AMS.broadcastIntent
            ActivityManager.getService().broadcastIntent(
                    mMainThread.getApplicationThread(), intent, resolvedType, null,
                    Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, false,
                    getUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
</code></pre> 
<h3><a id="32__AMSbroadcastIntent_586"></a>3.2 AMS.broadcastIntent</h3> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code> public final int broadcastIntent(IApplicationThread caller,
            Intent intent, String resolvedType, IIntentReceiver resultTo,
            int resultCode, String resultData, Bundle resultExtras,
            String[] requiredPermissions, int appOp, Bundle bOptions,
            boolean serialized, boolean sticky, int userId) {
        enforceNotIsolatedCaller(&#34;broadcastIntent&#34;);
        synchronized(this) {
            //验证广播intent是否有效
            intent &#61; verifyBroadcastLocked(intent);
            //获取调用者进程记录对象
            final ProcessRecord callerApp &#61; getRecordForAppLocked(caller);
            final int callingPid &#61; Binder.getCallingPid();
            final int callingUid &#61; Binder.getCallingUid();
            final long origId &#61; Binder.clearCallingIdentity();
            //见3.2节
            int res &#61; broadcastIntentLocked(callerApp,
                    callerApp !&#61; null ? callerApp.info.packageName : null,
                    intent, resolvedType, resultTo, resultCode, resultData, resultExtras,
                    requiredPermissions, appOp, bOptions, serialized, sticky,
                    callingPid, callingUid, userId);
            Binder.restoreCallingIdentity(origId);
            return res;
        }
    }
</code></pre> 
<p>broadcastIntent有两个boolean值参数serialized、sticky共同决定是普通广播&#xff0c;有效广播&#xff0c;还是sticky广播。</p> 
<table><thead><tr><th align="left">类型</th><th>serialized</th><th>sticky</th></tr></thead><tbody><tr><td align="left">sendBroadcast</td><td>false</td><td>false</td></tr><tr><td align="left">sendOrderedBroadcast</td><td>true</td><td>false</td></tr><tr><td align="left">sendStickyBroadcast</td><td>false</td><td>true</td></tr></tbody></table>
<h3><a id="33__AMSbroadcastIntentLocked_625"></a>3.3 AMS.broadcastIntentLocked</h3> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code> &#64;GuardedBy(&#34;this&#34;)
    final int broadcastIntentLocked(ProcessRecord callerApp,
            String callerPackage, Intent intent, String resolvedType,
            IIntentReceiver resultTo, int resultCode, String resultData,
            Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle bOptions,
            boolean ordered, boolean sticky, int callingPid, int callingUid, int userId) {
            
         //setp1&#xff1a;设置广播flags
         //setp2&#xff1a;广播权限验证
         //setp3&#xff1a;处理系统相关广播
         //setp4&#xff1a;增加sticky广播
         //setp5&#xff1a;查询receivers和registeredReceivers
         //setp6&#xff1a;处理并行广播
         //setp7&#xff1a;合并registeredReceivers到receivers
         //setp8&#xff1a;处理串行广播
        
        return ActivityManager.BROADCAST_SUCCESS;
    }
</code></pre> 
<p>该方法比较长&#xff0c;这里分为8部分进行分析。</p> 
<h4><a id="331_flags_652"></a>3.3.1 设置广播flags</h4> 
<pre><code>        intent &#61; new Intent(intent);
        //判断是否是即时应用
        final boolean callerInstantApp &#61; isInstantApp(callerApp, callerPackage, callingUid);
        //即时应用如果没有设置FLAG_RECEIVER_VISIBLE_TO_INSTANT_APPS则不会收到广播
        // Instant Apps cannot use FLAG_RECEIVER_VISIBLE_TO_INSTANT_APPS
        if (callerInstantApp) {
            intent.setFlags(intent.getFlags() &amp; ~Intent.FLAG_RECEIVER_VISIBLE_TO_INSTANT_APPS);
        }
        //增加flag,广播不会发送给已经停止的package
        // By default broadcasts do not go to stopped apps.
        intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);
        
        //系统没有启动完成&#xff0c;不允许启动新进程
        // If we have not finished booting, don&#39;t allow this to launch new processes.
        if (!mProcessesReady &amp;&amp; (intent.getFlags()&amp;Intent.FLAG_RECEIVER_BOOT_UPGRADE) &#61;&#61; 0) {
            intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
        }

        if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST,
                (sticky ? &#34;Broadcast sticky: &#34;: &#34;Broadcast: &#34;) &#43; intent
                &#43; &#34; ordered&#61;&#34; &#43; ordered &#43; &#34; userid&#61;&#34; &#43; userId);
        if ((resultTo !&#61; null) &amp;&amp; !ordered) {
            Slog.w(TAG, &#34;Broadcast &#34; &#43; intent &#43; &#34; not ordered but result callback requested!&#34;);
        }

        userId &#61; mUserController.handleIncomingUser(callingPid, callingUid, userId, true,
                ALLOW_NON_FULL, &#34;broadcast&#34;, callerPackage);
        //检查发送广播时的用户状态
        // Make sure that the user who is receiving this broadcast or its parent is running.
        // If not, we will just skip it. Make an exception for shutdown broadcasts, upgrade steps.
        if (userId !&#61; UserHandle.USER_ALL &amp;&amp; !mUserController.isUserOrItsParentRunning(userId)) {
            if ((callingUid !&#61; SYSTEM_UID
                    || (intent.getFlags() &amp; Intent.FLAG_RECEIVER_BOOT_UPGRADE) &#61;&#61; 0)
                    &amp;&amp; !Intent.ACTION_SHUTDOWN.equals(intent.getAction())) {
                Slog.w(TAG, &#34;Skipping broadcast of &#34; &#43; intent
                        &#43; &#34;: user &#34; &#43; userId &#43; &#34; and its parent (if any) are stopped&#34;);
                return ActivityManager.BROADCAST_FAILED_USER_STOPPED;
            }
        }
</code></pre> 
<p>这个过程主要的工作如下&#xff1a;</p> 
<p>1.是否设置FLAG_RECEIVER_VISIBLE_TO_INSTANT_APPS标志&#xff0c;如果设置广播对即时应用可见&#xff1b;</p> 
<p>2.添加flagFLAG_EXCLUDE_STOPPED_PACKAGES,保证已经停止的app不会收到广播&#xff1b;</p> 
<p>3.当系统还没有启动完成&#xff0c;不允许启动新进程&#xff1b;</p> 
<p>4.非USER_ALL广播且接收广播的用户没有处于Running的情况下&#xff0c;除非是系统升级广播和关键广播&#xff0c;否则直接返回。</p> 
<p>BroadcastReceiver还有其他flag&#xff0c;位于Intent.java常量中&#xff1a;</p> 
<pre><code>FLAG_RECEIVER_REGISTERED_ONLY  //只允许已经注册的receiver接收广播
FLAG_RECEIVER_REPLACE_PENDING //新广播会替代相同广播
FLAG_RECEIVER_FOREGROUND  //只允许前台receiver接收广播
FLAG_RECEIVER_NO_ABORT  //对于有序广播&#xff0c;先接收到的receiver无权抛弃广播
FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT  //boot完成之前&#xff0c;只允许已注册的receiver接收广播
FLAG_RECEIVER_BOOT_UPGRADE //升级模式下&#xff0c;允许系统准备就绪前发送广播
FLAG_RECEIVER_INCLUDE_BACKGROUND  //允许后台台receiver接收广播
FLAG_RECEIVER_EXCLUDE_BACKGROUND //不允许后台台receiver接收广播
FLAG_RECEIVER_VISIBLE_TO_INSTANT_APPS //广播对即时应用可见
</code></pre> 
<h4><a id="332__720"></a>3.3.2 广播权限验证</h4> 
<pre><code>   final String action &#61; intent.getAction();
        BroadcastOptions brOptions &#61; null;
        if (bOptions !&#61; null) {
            brOptions &#61; new BroadcastOptions(bOptions);
            if (brOptions.getTemporaryAppWhitelistDuration() &gt; 0) {
                //如果AppWhitelistDuration&gt;0,检查是否有CHANGE_DEVICE_IDLE_TEMP_WHITELIST权限
                // See if the caller is allowed to do this.  Note we are checking against
                // the actual real caller (not whoever provided the operation as say a
                // PendingIntent), because that who is actually supplied the arguments.
                if (checkComponentPermission(
                        android.Manifest.permission.CHANGE_DEVICE_IDLE_TEMP_WHITELIST,
                        Binder.getCallingPid(), Binder.getCallingUid(), -1, true)
                        !&#61; PackageManager.PERMISSION_GRANTED) {
                    String msg &#61; &#34;Permission Denial: &#34; &#43; intent.getAction()
                            &#43; &#34; broadcast from &#34; &#43; callerPackage &#43; &#34; (pid&#61;&#34; &#43; callingPid
                            &#43; &#34;, uid&#61;&#34; &#43; callingUid &#43; &#34;)&#34;
                            &#43; &#34; requires &#34;
                            &#43; android.Manifest.permission.CHANGE_DEVICE_IDLE_TEMP_WHITELIST;
                    Slog.w(TAG, msg);
                    throw new SecurityException(msg);
                }
            }
            //检查是否有后台限制
            if (brOptions.isDontSendToRestrictedApps()
                    &amp;&amp; !isUidActiveLocked(callingUid)
                    &amp;&amp; isBackgroundRestrictedNoCheck(callingUid, callerPackage)) {
                Slog.i(TAG, &#34;Not sending broadcast &#34; &#43; action &#43; &#34; - app &#34; &#43; callerPackage
                        &#43; &#34; has background restrictions&#34;);
                return ActivityManager.START_CANCELED;
            }
        }
        //检查受保护的广播只允许系统使用
        // Verify that protected broadcasts are only being sent by system code,
        // and that system code is only sending protected broadcasts.
        final boolean isProtectedBroadcast;
        try {
            isProtectedBroadcast &#61; AppGlobals.getPackageManager().isProtectedBroadcast(action);
        } catch (RemoteException e) {
            Slog.w(TAG, &#34;Remote exception&#34;, e);
            return ActivityManager.BROADCAST_SUCCESS;
        }

        final boolean isCallerSystem;
        switch (UserHandle.getAppId(callingUid)) {
            case ROOT_UID:
            case SYSTEM_UID:
            case PHONE_UID:
            case BLUETOOTH_UID:
            case NFC_UID:
            case SE_UID:
            case NETWORK_STACK_UID:
                isCallerSystem &#61; true;
                break;
            default:
                isCallerSystem &#61; (callerApp !&#61; null) &amp;&amp; callerApp.persistent;
                break;
        }
        //非系统发送的广播
        // First line security check before anything else: stop non-system apps from
        // sending protected broadcasts.
        if (!isCallerSystem) {
            //是保护广播&#xff0c;则抛出异常
            if (isProtectedBroadcast) {
                String msg &#61; &#34;Permission Denial: not allowed to send broadcast &#34;
                        &#43; action &#43; &#34; from pid&#61;&#34;
                        &#43; callingPid &#43; &#34;, uid&#61;&#34; &#43; callingUid;
                Slog.w(TAG, msg);
                throw new SecurityException(msg);

            } else if (AppWidgetManager.ACTION_APPWIDGET_CONFIGURE.equals(action)
                    || AppWidgetManager.ACTION_APPWIDGET_UPDATE.equals(action)) {
                //限制广播只发送自己    
                // Special case for compatibility: we don&#39;t want apps to send this,
                // but historically it has not been protected and apps may be using it
                // to poke their own app widget.  So, instead of making it protected,
                // just limit it to the caller.
                if (callerPackage &#61;&#61; null) {
                    String msg &#61; &#34;Permission Denial: not allowed to send broadcast &#34;
                            &#43; action &#43; &#34; from unknown caller.&#34;;
                    Slog.w(TAG, msg);
                    throw new SecurityException(msg);
                } else if (intent.getComponent() !&#61; null) {
                    // They are good enough to send to an explicit component...  verify
                    // it is being sent to the calling app.
                    if (!intent.getComponent().getPackageName().equals(
                            callerPackage)) {
                        String msg &#61; &#34;Permission Denial: not allowed to send broadcast &#34;
                                &#43; action &#43; &#34; to &#34;
                                &#43; intent.getComponent().getPackageName() &#43; &#34; from &#34;
                                &#43; callerPackage;
                        Slog.w(TAG, msg);
                        throw new SecurityException(msg);
                    }
                } else {
                    // Limit broadcast to their own package.
                    intent.setPackage(callerPackage);
                }
            }
        }
         if (action !&#61; null) {
            //如果允许后台应用接收广播&#xff0c;则添加FLAG_RECEIVER_INCLUDE_BACKGROUND
            if (getBackgroundLaunchBroadcasts().contains(action)) {
                if (DEBUG_BACKGROUND_CHECK) {
                    Slog.i(TAG, &#34;Broadcast action &#34; &#43; action &#43; &#34; forcing include-background&#34;);
                }
                intent.addFlags(Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND);
         }

</code></pre> 
<p>主要工作如下&#xff1a;</p> 
<p>1.如果TemporaryAppWhitelistDuration&gt;0,检查是否有CHANGE_DEVICE_IDLE_TEMP_WHITELIST权限&#xff0c;没有抛出异常</p> 
<p>2.检查是否后台限制发送广播&#xff0c;如果限制&#xff0c;则后台应用将不能发送广播</p> 
<p>3.callingUid为 ROOT_UID&#xff0c; SYSTEM_UID&#xff0c;PHONE_UID&#xff0c;BLUETOOTH_UID&#xff0c;NFC_UID&#xff0c;SE_UID&#xff0c;NETWORK_STACK_UID和persistent进程时&#xff0c;可以发送受保护广播</p> 
<p>4.为非系统应用发送广播时&#xff1a;当发送的是受保护的广播&#xff0c;则抛出异常&#xff1b;当action为ACTION_APPWIDGET_CONFIGURE或ACTION_APPWIDGET_UPDATE时&#xff0c;限制该广播只发送给自己&#xff0c;否则抛出异常。</p> 
<p>5.如果允许后台应用接收广播&#xff0c;则添加FLAG_RECEIVER_INCLUDE_BACKGROUND</p> 
<h4><a id="333__845"></a>3.3.3 处理系统相关广播</h4> 
<pre><code>   switch (action) {
                case Intent.ACTION_UID_REMOVED://uid移除
                case Intent.ACTION_PACKAGE_REMOVED://package移除
                case Intent.ACTION_PACKAGE_CHANGED://改变package
                case Intent.ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE://外部设备不可用
                case Intent.ACTION_EXTERNAL_APPLICATIONS_AVAILABLE:
                case Intent.ACTION_PACKAGES_SUSPENDED:
                case Intent.ACTION_PACKAGES_UNSUSPENDED:
                    // Handle special intents: if this broadcast is from the package
                    // manager about a package being removed, we need to remove all of
                    // its activities from the history stack.
                    if (checkComponentPermission(
                            android.Manifest.permission.BROADCAST_PACKAGE_REMOVED,
                            callingPid, callingUid, -1, true)
                            !&#61; PackageManager.PERMISSION_GRANTED) {
                        String msg &#61; &#34;Permission Denial: &#34; &#43; intent.getAction()
                                &#43; &#34; broadcast from &#34; &#43; callerPackage &#43; &#34; (pid&#61;&#34; &#43; callingPid
                                &#43; &#34;, uid&#61;&#34; &#43; callingUid &#43; &#34;)&#34;
                                &#43; &#34; requires &#34;
                                &#43; android.Manifest.permission.BROADCAST_PACKAGE_REMOVED;
                        Slog.w(TAG, msg);
                        throw new SecurityException(msg);
                    }
                    switch (action) {
                        case Intent.ACTION_UID_REMOVED:
                            final int uid &#61; getUidFromIntent(intent);
                            if (uid &gt;&#61; 0) {
                                mBatteryStatsService.removeUid(uid);
                                mAppOpsService.uidRemoved(uid);
                            }
                            break;
                        case Intent.ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE:
                            // If resources are unavailable just force stop all those packages
                            // and flush the attribute cache as well.
                            String list[] &#61;
                                    intent.getStringArrayExtra(Intent.EXTRA_CHANGED_PACKAGE_LIST);
                            if (list !&#61; null &amp;&amp; list.length &gt; 0) {
                                for (int i &#61; 0; i &lt; list.length; i&#43;&#43;) {
                                    forceStopPackageLocked(list[i], -1, false, true, true,
                                            false, false, userId, &#34;storage unmount&#34;);
                                }
                                mRecentTasks.cleanupLocked(UserHandle.USER_ALL);
                                sendPackageBroadcastLocked(
                                        ApplicationThreadConstants.EXTERNAL_STORAGE_UNAVAILABLE,
                                        list, userId);
                            }
                            break;
                        case Intent.ACTION_EXTERNAL_APPLICATIONS_AVAILABLE:
                            mRecentTasks.cleanupLocked(UserHandle.USER_ALL);
                            break;
                        case Intent.ACTION_PACKAGE_REMOVED:
                        case Intent.ACTION_PACKAGE_CHANGED:
                            Uri data &#61; intent.getData();
                            String ssp;
                            if (data !&#61; null &amp;&amp; (ssp&#61;data.getSchemeSpecificPart()) !&#61; null) {
                                boolean removed &#61; Intent.ACTION_PACKAGE_REMOVED.equals(action);
                                final boolean replacing &#61;
                                        intent.getBooleanExtra(Intent.EXTRA_REPLACING, false);
                                final boolean killProcess &#61;
                                        !intent.getBooleanExtra(Intent.EXTRA_DONT_KILL_APP, false);
                                final boolean fullUninstall &#61; removed &amp;&amp; !replacing;
                                if (removed) {
                                    if (killProcess) {
                                        forceStopPackageLocked(ssp, UserHandle.getAppId(
                                                intent.getIntExtra(Intent.EXTRA_UID, -1)),
                                                false, true, true, false, fullUninstall, userId,
                                                removed ? &#34;pkg removed&#34; : &#34;pkg changed&#34;);
                                    }
                                    final int cmd &#61; killProcess
                                            ? ApplicationThreadConstants.PACKAGE_REMOVED
                                            : ApplicationThreadConstants.PACKAGE_REMOVED_DONT_KILL;
                                    sendPackageBroadcastLocked(cmd,
                                            new String[] {ssp}, userId);
                                    if (fullUninstall) {
                                        mAppOpsService.packageRemoved(
                                                intent.getIntExtra(Intent.EXTRA_UID, -1), ssp);

                                        // Remove all permissions granted from/to this package
                                        removeUriPermissionsForPackageLocked(ssp, userId, true,
                                                false);

                                        mRecentTasks.removeTasksByPackageName(ssp, userId);

                                        mServices.forceStopPackageLocked(ssp, userId);
                                        mAppWarnings.onPackageUninstalled(ssp);
                                        mCompatModePackages.handlePackageUninstalledLocked(ssp);
                                        mBatteryStatsService.notePackageUninstalled(ssp);
                                    }
                                } else {
                                    if (killProcess) {
                                        killPackageProcessesLocked(ssp, UserHandle.getAppId(
                                                intent.getIntExtra(Intent.EXTRA_UID, -1)),
                                                userId, ProcessList.INVALID_ADJ,
                                                false, true, true, false, &#34;change &#34; &#43; ssp);
                                    }
                                    cleanupDisabledPackageComponentsLocked(ssp, userId, killProcess,
                                            intent.getStringArrayExtra(
                                                    Intent.EXTRA_CHANGED_COMPONENT_NAME_LIST));
                                }
                            }
                            break;
                        case Intent.ACTION_PACKAGES_SUSPENDED:
                        case Intent.ACTION_PACKAGES_UNSUSPENDED:
                            final boolean suspended &#61; Intent.ACTION_PACKAGES_SUSPENDED.equals(
                                    intent.getAction());
                            final String[] packageNames &#61; intent.getStringArrayExtra(
                                    Intent.EXTRA_CHANGED_PACKAGE_LIST);
                            final int userHandle &#61; intent.getIntExtra(
                                    Intent.EXTRA_USER_HANDLE, UserHandle.USER_NULL);

                            synchronized(ActivityManagerService.this) {
                                mRecentTasks.onPackagesSuspendedChanged(
                                        packageNames, suspended, userHandle);
                            }
                            break;
                    }
                    break;
                case Intent.ACTION_PACKAGE_REPLACED:
                {
                    final Uri data &#61; intent.getData();
                    final String ssp;
                    if (data !&#61; null &amp;&amp; (ssp &#61; data.getSchemeSpecificPart()) !&#61; null) {
                        ApplicationInfo aInfo &#61; null;
                        try {
                            aInfo &#61; AppGlobals.getPackageManager()
                                    .getApplicationInfo(ssp, STOCK_PM_FLAGS, userId);
                        } catch (RemoteException ignore) {}
                        if (aInfo &#61;&#61; null) {
                            Slog.w(TAG, &#34;Dropping ACTION_PACKAGE_REPLACED for non-existent pkg:&#34;
                                    &#43; &#34; ssp&#61;&#34; &#43; ssp &#43; &#34; data&#61;&#34; &#43; data);
                            return ActivityManager.BROADCAST_SUCCESS;
                        }
                        mStackSupervisor.updateActivityApplicationInfoLocked(aInfo);
                        mServices.updateServiceApplicationInfoLocked(aInfo);
                        sendPackageBroadcastLocked(ApplicationThreadConstants.PACKAGE_REPLACED,
                                new String[] {ssp}, userId);
                    }
                    break;
                }
                case Intent.ACTION_PACKAGE_ADDED:
                {
                    // Special case for adding a package: by default turn on compatibility mode.
                    Uri data &#61; intent.getData();
                    String ssp;
                    if (data !&#61; null &amp;&amp; (ssp &#61; data.getSchemeSpecificPart()) !&#61; null) {
                        final boolean replacing &#61;
                                intent.getBooleanExtra(Intent.EXTRA_REPLACING, false);
                        mCompatModePackages.handlePackageAddedLocked(ssp, replacing);

                        try {
                            ApplicationInfo ai &#61; AppGlobals.getPackageManager().
                                    getApplicationInfo(ssp, STOCK_PM_FLAGS, 0);
                            mBatteryStatsService.notePackageInstalled(ssp,
                                    ai !&#61; null ? ai.versionCode : 0);
                        } catch (RemoteException e) {
                        }
                    }
                    break;
                }
                case Intent.ACTION_PACKAGE_DATA_CLEARED:
                {
                    Uri data &#61; intent.getData();
                    String ssp;
                    if (data !&#61; null &amp;&amp; (ssp &#61; data.getSchemeSpecificPart()) !&#61; null) {
                        mCompatModePackages.handlePackageDataClearedLocked(ssp);
                        mAppWarnings.onPackageDataCleared(ssp);
                    }
                    break;
                }
                case Intent.ACTION_TIMEZONE_CHANGED:
                    // If this is the time zone changed action, queue up a message that will reset
                    // the timezone of all currently running processes. This message will get
                    // queued up before the broadcast happens.
                    mHandler.sendEmptyMessage(UPDATE_TIME_ZONE);
                    break;
                case Intent.ACTION_TIME_CHANGED:
                    // EXTRA_TIME_PREF_24_HOUR_FORMAT is optional so we must distinguish between
                    // the tri-state value it may contain and &#34;unknown&#34;.
                    // For convenience we re-use the Intent extra values.
                    final int NO_EXTRA_VALUE_FOUND &#61; -1;
                    final int timeFormatPreferenceMsgValue &#61; intent.getIntExtra(
                            Intent.EXTRA_TIME_PREF_24_HOUR_FORMAT,
                            NO_EXTRA_VALUE_FOUND /* defaultValue */);
                    // Only send a message if the time preference is available.
                    if (timeFormatPreferenceMsgValue !&#61; NO_EXTRA_VALUE_FOUND) {
                        Message updateTimePreferenceMsg &#61;
                                mHandler.obtainMessage(UPDATE_TIME_PREFERENCE_MSG,
                                        timeFormatPreferenceMsgValue, 0);
                        mHandler.sendMessage(updateTimePreferenceMsg);
                    }
                    BatteryStatsImpl stats &#61; mBatteryStatsService.getActiveStatistics();
                    synchronized (stats) {
                        stats.noteCurrentTimeChangedLocked();
                    }
                    break;
                case Intent.ACTION_CLEAR_DNS_CACHE:  //清除DNS cache
                    mHandler.sendEmptyMessage(CLEAR_DNS_CACHE_MSG);
                    break;
                case Proxy.PROXY_CHANGE_ACTION: //网络代理改变
                    mHandler.sendMessage(mHandler.obtainMessage(UPDATE_HTTP_PROXY_MSG));
                    break;
                case android.hardware.Camera.ACTION_NEW_PICTURE:
                case android.hardware.Camera.ACTION_NEW_VIDEO:
                    // In N we just turned these off; in O we are turing them back on partly,
                    // only for registered receivers.  This will still address the main problem
                    // (a spam of apps waking up when a picture is taken putting significant
                    // memory pressure on the system at a bad point), while still allowing apps
                    // that are already actively running to know about this happening.
                    intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
                    break;
                case android.security.KeyChain.ACTION_TRUST_STORE_CHANGED:
                    mHandler.sendEmptyMessage(HANDLE_TRUST_STORAGE_UPDATE_MSG);
                    break;
                case &#34;com.android.launcher.action.INSTALL_SHORTCUT&#34;:
                    // As of O, we no longer support this broadcasts, even for pre-O apps.
                    // Apps should now be using ShortcutManager.pinRequestShortcut().
                    Log.w(TAG, &#34;Broadcast &#34; &#43; action
                            &#43; &#34; no longer supported. It will not be delivered.&#34;);
                    return ActivityManager.BROADCAST_SUCCESS;
            }

            if (Intent.ACTION_PACKAGE_ADDED.equals(action) ||
                    Intent.ACTION_PACKAGE_REMOVED.equals(action) ||
                    Intent.ACTION_PACKAGE_REPLACED.equals(action)) {
                final int uid &#61; getUidFromIntent(intent);
                if (uid !&#61; -1) {
                    final UidRecord uidRec &#61; mActiveUids.get(uid);
                    if (uidRec !&#61; null) {
                        uidRec.updateHasInternetPermission();
                    }
                }
            }
        }

</code></pre> 
<p>该过程主要系统广播&#xff0c;主要是package、时间、网络相关的广播&#xff0c;并对这些广播进行相应的处理。</p> 
<h4><a id="334_sticky_1086"></a>3.3.4 增加sticky广播</h4> 
<pre><code>      // Add to the sticky list if requested.
        if (sticky) {
            //检查是否有BROADCAST_STICKY权限
            if (checkPermission(android.Manifest.permission.BROADCAST_STICKY,
                    callingPid, callingUid)
                    !&#61; PackageManager.PERMISSION_GRANTED) {
                String msg &#61; &#34;Permission Denial: broadcastIntent() requesting a sticky broadcast from pid&#61;&#34;
                        &#43; callingPid &#43; &#34;, uid&#61;&#34; &#43; callingUid
                        &#43; &#34; requires &#34; &#43; android.Manifest.permission.BROADCAST_STICKY;
                Slog.w(TAG, msg);
                throw new SecurityException(msg);
            }
            if (requiredPermissions !&#61; null &amp;&amp; requiredPermissions.length &gt; 0) {
                Slog.w(TAG, &#34;Can&#39;t broadcast sticky intent &#34; &#43; intent
                        &#43; &#34; and enforce permissions &#34; &#43; Arrays.toString(requiredPermissions));
                return ActivityManager.BROADCAST_STICKY_CANT_HAVE_PERMISSION;
            }
            //发送指定组件&#xff0c;抛出异常
            if (intent.getComponent() !&#61; null) {
                throw new SecurityException(
                        &#34;Sticky broadcasts can&#39;t target a specific component&#34;);
            }
            //当非USER_ALL广播和USER_ALL冲突
            // We use userId directly here, since the &#34;all&#34; target is maintained
            // as a separate set of sticky broadcasts.
            if (userId !&#61; UserHandle.USER_ALL) {
                // But first, if this is not a broadcast to all users, then
                // make sure it doesn&#39;t conflict with an existing broadcast to
                // all users.
                ArrayMap&lt;String, ArrayList&lt;Intent&gt;&gt; stickies &#61; mStickyBroadcasts.get(
                        UserHandle.USER_ALL);
                if (stickies !&#61; null) {
                    ArrayList&lt;Intent&gt; list &#61; stickies.get(intent.getAction());
                    if (list !&#61; null) {
                        int N &#61; list.size();
                        int i;
                        for (i&#61;0; i&lt;N; i&#43;&#43;) {
                            if (intent.filterEquals(list.get(i))) {
                                throw new IllegalArgumentException(
                                        &#34;Sticky broadcast &#34; &#43; intent &#43; &#34; for user &#34;
                                        &#43; userId &#43; &#34; conflicts with existing global broadcast&#34;);
                            }
                        }
                    }
                }
            }
            ArrayMap&lt;String, ArrayList&lt;Intent&gt;&gt; stickies &#61; mStickyBroadcasts.get(userId);
            if (stickies &#61;&#61; null) {
                stickies &#61; new ArrayMap&lt;&gt;();
                mStickyBroadcasts.put(userId, stickies);
            }
            ArrayList&lt;Intent&gt; list &#61; stickies.get(intent.getAction());
            if (list &#61;&#61; null) {
                list &#61; new ArrayList&lt;&gt;();
                stickies.put(intent.getAction(), list);
            }
            final int stickiesCount &#61; list.size();
            int i;
            for (i &#61; 0; i &lt; stickiesCount; i&#43;&#43;) {
                if (intent.filterEquals(list.get(i))) {
                    // This sticky already exists, replace it.
                    list.set(i, new Intent(intent));
                    break;
                }
            }
            if (i &gt;&#61; stickiesCount) {
                list.add(new Intent(intent));
            }
        }

</code></pre> 
<p>这个过程主要是检查sticky广播&#xff0c;将sticky广播放入到mStickyBroadcasts&#xff0c;并增加到list</p> 
<h4><a id="335_receiversregisteredReceivers_1163"></a>3.3.5 查询receivers和registeredReceivers</h4> 
<pre><code>        //发送广播的user
        int[] users;
        if (userId &#61;&#61; UserHandle.USER_ALL) {
            // Caller wants broadcast to go to all started users.
            users &#61; mUserController.getStartedUserArray();
        } else {
            // Caller wants broadcast to go to one specific user.
            users &#61; new int[] {userId};
        }
        //查询哪些广播将会接受广播
        // Figure out who all will receive this broadcast.
        List receivers &#61; null;
        List&lt;BroadcastFilter&gt; registeredReceivers &#61; null;
        // Need to resolve the intent to interested receivers...
        //当允许静态接收者处理广播时&#xff0c;则通过PKMS根据intent查询静态receivers
        if ((intent.getFlags()&amp;Intent.FLAG_RECEIVER_REGISTERED_ONLY)
                 &#61;&#61; 0) {
            receivers &#61; collectReceiverComponents(intent, resolvedType, callingUid, users);
        }
        if (intent.getComponent() &#61;&#61; null) {
            if (userId &#61;&#61; UserHandle.USER_ALL &amp;&amp; callingUid &#61;&#61; SHELL_UID) {
                // Query one target user at a time, excluding shell-restricted users
                for (int i &#61; 0; i &lt; users.length; i&#43;&#43;) {
                    if (mUserController.hasUserRestriction(
                            UserManager.DISALLOW_DEBUGGING_FEATURES, users[i])) {
                        continue;
                    }
                    //查询动态注册广播
                    List&lt;BroadcastFilter&gt; registeredReceiversForUser &#61;
                            mReceiverResolver.queryIntent(intent,
                                    resolvedType, false /*defaultOnly*/, users[i]);
                    if (registeredReceivers &#61;&#61; null) {
                        registeredReceivers &#61; registeredReceiversForUser;
                    } else if (registeredReceiversForUser !&#61; null) {
                        registeredReceivers.addAll(registeredReceiversForUser);
                    }
                }
            } else {
                registeredReceivers &#61; mReceiverResolver.queryIntent(intent,
                        resolvedType, false /*defaultOnly*/, userId);
            }
        }

</code></pre> 
<p>主要工作如下&#xff1a;</p> 
<p>1.根据userId判断发送的是全部的接收者还是指定的userId</p> 
<p>2.查询广播&#xff0c;并将其放入到两个列表&#xff1a;</p> 
<p>registeredReceivers&#xff1a;来匹配当前intent的所有动态注册的广播接收者&#xff08;mReceiverResolver见2.4节&#xff09;</p> 
<p>receivers&#xff1a;记录当前intent的所有静态注册的广播接收者</p> 
<pre><code>   private List&lt;ResolveInfo&gt; collectReceiverComponents(Intent intent, String resolvedType,
            int callingUid, int[] users) {
       ...
      //调用PKMS的queryIntentReceivers&#xff0c;可以获取AndroidManifeset中注册的接收信息
       List&lt;ResolveInfo&gt; newReceivers &#61; AppGlobals.getPackageManager()
                        .queryIntentReceivers(intent, resolvedType, pmFlags, user).getList();
       ...        
       return receivers;
    }
</code></pre> 
<h4><a id="336__1233"></a>3.3.6 处理并行广播</h4> 
<pre><code>        //用于标识是否需要用新的intent替换旧的intent
        final boolean replacePending &#61;
                (intent.getFlags()&amp;Intent.FLAG_RECEIVER_REPLACE_PENDING) !&#61; 0;

        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, &#34;Enqueueing broadcast: &#34; &#43; intent.getAction()
                &#43; &#34; replacePending&#61;&#34; &#43; replacePending);
        //处理并行广播
        int NR &#61; registeredReceivers !&#61; null ? registeredReceivers.size() : 0;
        //发送的不是有序广播
        if (!ordered &amp;&amp; NR &gt; 0) {
            // If we are not serializing this broadcast, then send the
            // registered receivers separately so they don&#39;t wait for the
            // components to be launched.
            //检查系统发送的广播
            if (isCallerSystem) {
                checkBroadcastFromSystem(intent, callerApp, callerPackage, callingUid,
                        isProtectedBroadcast, registeredReceivers);
            }
            //根据intent的flag来判断前台队列还是后台队列&#xff0c;见2.4.3节
            final BroadcastQueue queue &#61; broadcastQueueForIntent(intent);
            //创建BroadcastRecord
            BroadcastRecord r &#61; new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, callerInstantApp, resolvedType,
                    requiredPermissions, appOp, brOptions, registeredReceivers, resultTo,
                    resultCode, resultData, resultExtras, ordered, sticky, false, userId);
            if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, &#34;Enqueueing parallel broadcast &#34; &#43; r);
            final boolean replaced &#61; replacePending
                    &amp;&amp; (queue.replaceParallelBroadcastLocked(r) !&#61; null);
            // Note: We assume resultTo is null for non-ordered broadcasts.
            if (!replaced) {
                //将BroadcastRecord加入到并行广播队列
                queue.enqueueParallelBroadcastLocked(r);
                //处理广播&#xff0c;见4.1节
                queue.scheduleBroadcastsLocked();
            }
            //动态注册的广播接收者处理完成&#xff0c;则将该变量设置为空
            registeredReceivers &#61; null;
            NR &#61; 0;
        }

</code></pre> 
<p>广播队列中有一个mParallelBroadcasts变量&#xff0c;类型为ArrayList&#xff0c;记录所有的并行广播</p> 
<pre><code>  public void enqueueParallelBroadcastLocked(BroadcastRecord r) {
        mParallelBroadcasts.add(r);
        enqueueBroadcastHelper(r);
  }
</code></pre> 
<h4><a id="337_registeredReceiversreceivers_1287"></a>3.3.7 合并registeredReceivers到receivers</h4> 
<pre><code>        // Merge into one list.
        int ir &#61; 0;
        if (receivers !&#61; null) {
            //防止应用监听广播&#xff0c;在安装时直接运行
            // A special case for PACKAGE_ADDED: do not allow the package
            // being added to see this broadcast.  This prevents them from
            // using this as a back door to get run as soon as they are
            // installed.  Maybe in the future we want to have a special install
            // broadcast or such for apps, but we&#39;d like to deliberately make
            // this decision.
            String skipPackages[] &#61; null;
            if (Intent.ACTION_PACKAGE_ADDED.equals(intent.getAction())
                    || Intent.ACTION_PACKAGE_RESTARTED.equals(intent.getAction())
                    || Intent.ACTION_PACKAGE_DATA_CLEARED.equals(intent.getAction())) {
                Uri data &#61; intent.getData();
                if (data !&#61; null) {
                    String pkgName &#61; data.getSchemeSpecificPart();
                    if (pkgName !&#61; null) {
                        skipPackages &#61; new String[] { pkgName };
                    }
                }
            } else if (Intent.ACTION_EXTERNAL_APPLICATIONS_AVAILABLE.equals(intent.getAction())) {
                skipPackages &#61; intent.getStringArrayExtra(Intent.EXTRA_CHANGED_PACKAGE_LIST);
            }
            //将skipPackages相关的广播接收者从receivers列表中移除。
            if (skipPackages !&#61; null &amp;&amp; (skipPackages.length &gt; 0)) {
                for (String skipPackage : skipPackages) {
                    if (skipPackage !&#61; null) {
                        int NT &#61; receivers.size();
                        for (int it&#61;0; it&lt;NT; it&#43;&#43;) {
                            ResolveInfo curt &#61; (ResolveInfo)receivers.get(it);
                            if (curt.activityInfo.packageName.equals(skipPackage)) {
                                receivers.remove(it);
                                it--;
                                NT--;
                            }
                        }
                    }
                }
            }
            //3.4.6节有处理动态广播的过程&#xff0c;处理完成后再执行将动态注册的registeredReceivers合并到receivers中
            int NT &#61; receivers !&#61; null ? receivers.size() : 0;
            int it &#61; 0;
            ResolveInfo curt &#61; null;
            BroadcastFilter curr &#61; null;
            while (it &lt; NT &amp;&amp; ir &lt; NR) {
                if (curt &#61;&#61; null) {
                    curt &#61; (ResolveInfo)receivers.get(it);
                }
                if (curr &#61;&#61; null) {
                    curr &#61; registeredReceivers.get(ir);
                }
                //优先级大的&#xff0c;则插到前面
                if (curr.getPriority() &gt;&#61; curt.priority) {
                    // Insert this broadcast record into the final list.
                    receivers.add(it, curr);
                    ir&#43;&#43;;
                    curr &#61; null;
                    it&#43;&#43;;
                    NT&#43;&#43;;
                } else {
                    // Skip to the next ResolveInfo in the final list.
                    it&#43;&#43;;
                    curt &#61; null;
                }
            }
        }
        while (ir &lt; NR) {
            if (receivers &#61;&#61; null) {
                receivers &#61; new ArrayList();
            }
            receivers.add(registeredReceivers.get(ir));
            ir&#43;&#43;;
        }
        //检查系统发送的广播
        if (isCallerSystem) {
            checkBroadcastFromSystem(intent, callerApp, callerPackage, callingUid,
                    isProtectedBroadcast, receivers);
        }

</code></pre> 
<p>这里主要是将动态注册的registeredReceivers&#xff08;如果发送的广播不是有序广播则registeredReceivers &#61; null&#xff09;全部合并到receivers&#xff0c;再统一按照串行方式处理。</p> 
<h4><a id="338__1374"></a>3.3.8 处理串行广播</h4> 
<pre><code>   if ((receivers !&#61; null &amp;&amp; receivers.size() &gt; 0)
                || resultTo !&#61; null) {
            //根据intent的flag判断是前台队列还是后台队列&#xff0c;见2.4.3节    
            BroadcastQueue queue &#61; broadcastQueueForIntent(intent);
            //创建BroadcastRecord
            BroadcastRecord r &#61; new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, callerInstantApp, resolvedType,
                    requiredPermissions, appOp, brOptions, receivers, resultTo, resultCode,
                    resultData, resultExtras, ordered, sticky, false, userId);

            if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, &#34;Enqueueing ordered broadcast &#34; &#43; r
                    &#43; &#34;: prev had &#34; &#43; queue.mOrderedBroadcasts.size());
            if (DEBUG_BROADCAST) Slog.i(TAG_BROADCAST,
                    &#34;Enqueueing broadcast &#34; &#43; r.intent.getAction());
          
            final BroadcastRecord oldRecord &#61;
                    replacePending ? queue.replaceOrderedBroadcastLocked(r) : null;
            if (oldRecord !&#61; null) {
                // Replaced, fire the result-to receiver.
                if (oldRecord.resultTo !&#61; null) {
                    final BroadcastQueue oldQueue &#61; broadcastQueueForIntent(oldRecord.intent);
                    try {
                        oldQueue.performReceiveLocked(oldRecord.callerApp, oldRecord.resultTo,
                                oldRecord.intent,
                                Activity.RESULT_CANCELED, null, null,
                                false, false, oldRecord.userId);
                    } catch (RemoteException e) {
                        Slog.w(TAG, &#34;Failure [&#34;
                                &#43; queue.mQueueName &#43; &#34;] sending broadcast result of &#34;
                                &#43; intent, e);

                    }
                }
            } else {
                //将BroadcastRecord加入到有序广播队列
                queue.enqueueOrderedBroadcastLocked(r);
                //处理广播&#xff0c;见4.1节
                queue.scheduleBroadcastsLocked();
            }
        } else {
            // There was nobody interested in the broadcast, but we still want to record
            // that it happened.
            if (intent.getComponent() &#61;&#61; null &amp;&amp; intent.getPackage() &#61;&#61; null
                    &amp;&amp; (intent.getFlags()&amp;Intent.FLAG_RECEIVER_REGISTERED_ONLY) &#61;&#61; 0) {
                // This was an implicit broadcast... let&#39;s record it for posterity.
                addBroadcastStatLocked(intent.getAction(), callerPackage, 0, 0, 0);
            }
        }

</code></pre> 
<p>广播队列中有一个mOrderedBroadcasts变量&#xff0c;类型为ArrayList&#xff0c;记录所有的有序广播</p> 
<pre><code>//串行广播加入到mOrderedBroadcasts队列
public void enqueueOrderedBroadcastLocked(BroadcastRecord r) {
        mOrderedBroadcasts.add(r);
        enqueueBroadcastHelper(r);
  }
</code></pre> 
<h3><a id="34___1438"></a>3.4 小结</h3> 
<p>发送广播的过程&#xff1a;</p> 
<p>1.默认不发送给已停止的&#xff08;FLAG_EXCLUDE_STOPPED_PACKAGES&#xff09;应用和即时应用&#xff08;需要添加该FLAG_RECEIVER_VISIBLE_TO_INSTANT_APPS标记才可以&#xff09;</p> 
<p>2.对广播进行权限验证&#xff0c;是否是受保护的广播&#xff0c;是否允许后台接收广播&#xff0c;是否允许后台发送广播</p> 
<p>3.处理系统广播&#xff0c;主要是package、时间、网络相关的广播</p> 
<p>4.当为粘性广播时&#xff0c;将sticky广播增加到list&#xff0c;并放入mStickyBroadcasts队列</p> 
<p>5.当广播的intent没有设置FLAG_RECEIVER_REGISTERED_ONLY&#xff0c;则允许静态广播接收者来处理该广播&#xff1b;创建BroadcastRecord对象&#xff0c;并将该对象加入到相应的广播队列&#xff0c;然后调用BroadcastQueue的scheduleBroadcastsLocked方法来完成不同广播的处理。</p> 
<p>不同广播的处理方式&#xff1a;</p> 
<p>1.sticky广播&#xff1a;广播注册过程中处理AMS.registerReceiver&#xff0c;开始处理粘性广播&#xff0c;见2.4节</p> 
<ul><li>创建BroadcastRecord对象</li><li>添加到mParallelBroadcasts队列</li><li>然后执行 queue.scheduleBroadcastsLocked()</li></ul> 
<p>2.并行广播&#xff1a;广播发送处理过程见3.3.6节</p> 
<ul><li>只有动态注册的registeredReceivers才会进行并行处理</li><li>创建BroadcastRecord对象</li><li>添加到mParallelBroadcasts队列</li><li>然后执行 queue.scheduleBroadcastsLocked()</li></ul> 
<p>3.串行广播&#xff1a;广播发送处理过程见3.3.8节</p> 
<ul><li>所有静态注册的receivers以及动态注册的registeredReceivers&#xff08;发送的广播是有序广播&#xff09;合并到一张表处理</li><li>创建BroadcastRecord对象</li><li>添加到mOrderedBroadcasts队列</li><li>然后执行 queue.scheduleBroadcastsLocked()</li></ul> 
<p>从上面可以看出&#xff0c;不管哪种广播方式&#xff0c;都是通过broadcastQueueForIntent来根据intent的flag判段是前台队列还是后台队列广播&#xff0c;然后再调用对应广播队列的scheduleBroadcastsLocked方法来处理广播。</p> 
<h2><a id="_1476"></a>四、接收广播</h2> 
<p>在发送广播的过程会执行scheduleBroadcastsLocked方法来处理广播</p> 
<h3><a id="41__BQscheduleBroadcastsLocked_1480"></a>4.1 BQ.scheduleBroadcastsLocked</h3> 
<p>[-&gt;BroadcastQueue.java]</p> 
<pre><code>  public void scheduleBroadcastsLocked() {
        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, &#34;Schedule broadcasts [&#34;
                &#43; mQueueName &#43; &#34;]: current&#61;&#34;
                &#43; mBroadcastsScheduled);
        //正在处理BROADCAST_INTENT_MSG消息
        if (mBroadcastsScheduled) {
            return;
        }
        //发送BROADCAST_INTENT_MSG消息
        mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
        mBroadcastsScheduled &#61; true;
    }
</code></pre> 
<h4><a id="411_BroadcastHandler_1499"></a>4.1.1 BroadcastHandler</h4> 
<p>[-&gt;BroadcastQueue.java]</p> 
<pre><code> private final class BroadcastHandler extends Handler {
        public BroadcastHandler(Looper looper) {
            super(looper, null, true);
        }

        &#64;Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case BROADCAST_INTENT_MSG: {
                    if (DEBUG_BROADCAST) Slog.v(
                            TAG_BROADCAST, &#34;Received BROADCAST_INTENT_MSG&#34;);
                    //见4.2节
                    processNextBroadcast(true);
                } break;
                case BROADCAST_TIMEOUT_MSG: {
                    synchronized (mService) {
                        broadcastTimeoutLocked(true);
                    }
                } break;
            }
        }
    }
</code></pre> 
<p>发送BROADCAST_INTENT_MSG消息后&#xff0c;BroadcastHandler进行处理&#xff0c;其初始化在构造函数中创建&#xff0c;而BroadcastQueue是在AMS初始化&#xff0c;从源码中可以看出handler采用的是&#34;ActivityManagerService&#34;线程的loop。</p> 
<pre><code> BroadcastQueue(ActivityManagerService service, Handler handler,
            String name, long timeoutPeriod, boolean allowDelayBehindServices) {
        mService &#61; service;
        //创建BroadcastHandler
        mHandler &#61; new BroadcastHandler(handler.getLooper());
        mQueueName &#61; name;
        mTimeoutPeriod &#61; timeoutPeriod;
        mDelayBehindServices &#61; allowDelayBehindServices;
 }
 
 public ActivityManagerService(Context systemContext) {
        ...
        //名为ActivityManagerService的线程
        mHandlerThread &#61; new ServiceThread(TAG,
                THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
        mHandlerThread.start();
        mHandler &#61; new MainHandler(mHandlerThread.getLooper());
        //创建BroadcastQueue对象
        mFgBroadcastQueue &#61; new BroadcastQueue(this, mHandler,
                &#34;foreground&#34;, BROADCAST_FG_TIMEOUT, false);
        mBgBroadcastQueue &#61; new BroadcastQueue(this, mHandler,
                &#34;background&#34;, BROADCAST_BG_TIMEOUT, true);
        
        //android10.0新增加的一个分流队列&#xff0c;目前仅处理BOOT_COMPLETED广播
        // Convenient for easy iteration over the queues. Foreground is first
        // so that dispatch of foreground broadcasts gets precedence.
        final BroadcastQueue[] mBroadcastQueues &#61; new BroadcastQueue[2];
        ...
 } 
 
 //广播超时时间定义
 // How long we allow a receiver to run before giving up on it.
 static final int BROADCAST_FG_TIMEOUT &#61; 10*1000;
 static final int BROADCAST_BG_TIMEOUT &#61; 60*1000;
</code></pre> 
<h3><a id="42__BQprocessNextBroadcast_1567"></a>4.2 BQ.processNextBroadcast</h3> 
<p>[-&gt;BroadcastQueue.java]</p> 
<pre><code>    final void processNextBroadcast(boolean fromMsg) {
        //同步mService
        synchronized (mService) {
            processNextBroadcastLocked(fromMsg, false);
        }
    }  
</code></pre> 
<p>此次mService为AMS&#xff0c;整个流程比较长&#xff0c;全程持有AMS锁&#xff0c;所以广播效率低下的情况下&#xff0c;直接会严重影响手机的性能和流畅度&#xff0c;这里是否应考虑细化同步锁的粒度。</p> 
<pre><code> final void processNextBroadcastLocked(boolean fromMsg, boolean skipOomAdj) {
        //setp1&#xff1a;处理并行广播
        //setp2&#xff1a;处理串行广播
        //setp3&#xff1a;获取下条有序广播
        //setp4&#xff1a;处理下条有序广播  
 }
</code></pre> 
<p>整个处理的过程比较长&#xff0c;将分为四个部分进行分析。</p> 
<h4><a id="421__1593"></a>4.2.1 处理并行广播</h4> 
<pre><code>        BroadcastRecord r;
        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, &#34;processNextBroadcast [&#34;
                &#43; mQueueName &#43; &#34;]: &#34;
                &#43; mParallelBroadcasts.size() &#43; &#34; parallel broadcasts, &#34;
                &#43; mOrderedBroadcasts.size() &#43; &#34; ordered broadcasts&#34;);
        //更新cpu统计信息
        mService.updateCpuStats();
        //方法传进来的是true&#xff0c;将mBroadcastsScheduled重置为false
        if (fromMsg) {
            mBroadcastsScheduled &#61; false;
        }
        
        // 处理并行队列
        // First, deliver any non-serialized broadcasts right away.
        while (mParallelBroadcasts.size() &gt; 0) {
            r &#61; mParallelBroadcasts.remove(0);
            r.dispatchTime &#61; SystemClock.uptimeMillis();
            r.dispatchClockTime &#61; System.currentTimeMillis();

            if (Trace.isTagEnabled(Trace.TRACE_TAG_ACTIVITY_MANAGER)) {
                Trace.asyncTraceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                    createBroadcastTraceTitle(r, BroadcastRecord.DELIVERY_PENDING),
                    System.identityHashCode(r));
                Trace.asyncTraceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                    createBroadcastTraceTitle(r, BroadcastRecord.DELIVERY_DELIVERED),
                    System.identityHashCode(r));
            }

            final int N &#61; r.receivers.size();
            if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST, &#34;Processing parallel broadcast [&#34;
                    &#43; mQueueName &#43; &#34;] &#34; &#43; r);
            for (int i&#61;0; i&lt;N; i&#43;&#43;) {
                Object target &#61; r.receivers.get(i);
                if (DEBUG_BROADCAST)  Slog.v(TAG_BROADCAST,
                        &#34;Delivering non-ordered on [&#34; &#43; mQueueName &#43; &#34;] to registered &#34;
                        &#43; target &#43; &#34;: &#34; &#43; r);
                //分发广播给已经注册的receiver&#xff0c;见4.3节
                deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false, i);
            }
            //将广播添加到历史记录
            addBroadcastToHistoryLocked(r); 
            if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST, &#34;Done with parallel broadcast [&#34;
                    &#43; mQueueName &#43; &#34;] &#34; &#43; r);
        }

</code></pre> 
<p>通过while循环&#xff0c;每次取出一个&#xff0c;一次性分发完所有的并发广播后&#xff0c;将分发完成的添加到历史广播队列。</p> 
<h4><a id="422__1645"></a>4.2.2 处理串行广播</h4> 
<pre><code>        // Now take care of the next serialized one...
        // If we are waiting for a process to come up to handle the next
        // broadcast, then do nothing at this point.  Just in case, we
        // check that the process we&#39;re waiting for still exists.
        if (mPendingBroadcast !&#61; null) {
            if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST,
                    &#34;processNextBroadcast [&#34; &#43; mQueueName &#43; &#34;]: waiting for &#34;
                    &#43; mPendingBroadcast.curApp);

            boolean isDead;
            if (mPendingBroadcast.curApp.pid &gt; 0) {
                synchronized (mService.mPidsSelfLocked) {
                    //从mPidsSelfLocked获取正在处理的广播进程&#xff0c;判断进程是否死亡
                    ProcessRecord proc &#61; mService.mPidsSelfLocked.get(
                            mPendingBroadcast.curApp.pid);
                    isDead &#61; proc &#61;&#61; null || proc.crashing;
                }
            } else {
                //从mProcessNames获取正在处理的广播进程&#xff0c;判断进程是否死亡
                final ProcessRecord proc &#61; mService.mProcessNames.get(
                        mPendingBroadcast.curApp.processName, mPendingBroadcast.curApp.uid);
                isDead &#61; proc &#61;&#61; null || !proc.pendingStart;
            }
            if (!isDead) {
                // 如果正在处理广播的进程保持活跃状态&#xff0c;则继续等待其执行完成
                // It&#39;s still alive, so keep waiting
                return;
            } else {
                Slog.w(TAG, &#34;pending app  [&#34;
                        &#43; mQueueName &#43; &#34;]&#34; &#43; mPendingBroadcast.curApp
                        &#43; &#34; died before responding to broadcast&#34;);
                mPendingBroadcast.state &#61; BroadcastRecord.IDLE;
                mPendingBroadcast.nextReceiver &#61; mPendingBroadcastRecvIndex;
                mPendingBroadcast &#61; null;
            }
        }

        boolean looped &#61; false;

        do {
            if (mOrderedBroadcasts.size() &#61;&#61; 0) {
                // No more broadcasts pending, so all done!
                //所有的串行广播处理完成&#xff0c;则调度执行gc
                mService.scheduleAppGcsLocked();
                if (looped) {
                    // If we had finished the last ordered broadcast, then
                    // make sure all processes have correct oom and sched
                    // adjustments.
                    mService.updateOomAdjLocked();
                }
                return;
            }
            r &#61; mOrderedBroadcasts.get(0);
            boolean forceReceive &#61; false;

            // Ensure that even if something goes awry with the timeout
            // detection, we catch &#34;hung&#34; broadcasts here, discard them,
            // and continue to make progress.
            //
            // This is only done if the system is ready so that PRE_BOOT_COMPLETED
            // receivers don&#39;t get executed with timeouts. They&#39;re intended for
            // one time heavy lifting after system upgrades and can take
            // significant amounts of time.
            //所有有序广播的接收者
            int numReceivers &#61; (r.receivers !&#61; null) ? r.receivers.size() : 0;
            //系统进程已经准备好
            if (mService.mProcessesReady &amp;&amp; r.dispatchTime &gt; 0) {
                long now &#61; SystemClock.uptimeMillis();
                if ((numReceivers &gt; 0) &amp;&amp;
                        (now &gt; r.dispatchTime &#43; (2*mTimeoutPeriod*numReceivers))) {
                    Slog.w(TAG, &#34;Hung broadcast [&#34;
                            &#43; mQueueName &#43; &#34;] discarded after timeout failure:&#34;
                            &#43; &#34; now&#61;&#34; &#43; now
                            &#43; &#34; dispatchTime&#61;&#34; &#43; r.dispatchTime
                            &#43; &#34; startTime&#61;&#34; &#43; r.receiverTime
                            &#43; &#34; intent&#61;&#34; &#43; r.intent
                            &#43; &#34; numReceivers&#61;&#34; &#43; numReceivers
                            &#43; &#34; nextReceiver&#61;&#34; &#43; r.nextReceiver
                            &#43; &#34; state&#61;&#34; &#43; r.state);
                    //广播超时&#xff0c;强制结束这条广播        
                    broadcastTimeoutLocked(false); // forcibly finish this broadcast
                    forceReceive &#61; true;
                    r.state &#61; BroadcastRecord.IDLE;
                }
            }
            if (r.state !&#61; BroadcastRecord.IDLE) {
                if (DEBUG_BROADCAST) Slog.d(TAG_BROADCAST,
                        &#34;processNextBroadcast(&#34;
                        &#43; mQueueName &#43; &#34;) called when not idle (state&#61;&#34;
                        &#43; r.state &#43; &#34;)&#34;);
                return;
            }

            if (r.receivers &#61;&#61; null || r.nextReceiver &gt;&#61; numReceivers
                    || r.resultAbort || forceReceive) {
                // No more receivers for this broadcast!  Send the final
                // result if requested...
                if (r.resultTo !&#61; null) {
                    try {
                        if (DEBUG_BROADCAST) Slog.i(TAG_BROADCAST,
                                &#34;Finishing broadcast [&#34; &#43; mQueueName &#43; &#34;] &#34;
                                &#43; r.intent.getAction() &#43; &#34; app&#61;&#34; &#43; r.callerApp);
                        //处理广播消息&#xff0c;调用到onReceive()        
                        performReceiveLocked(r.callerApp, r.resultTo,
                            new Intent(r.intent), r.resultCode,
                            r.resultData, r.resultExtras, false, false, r.userId);
                        // Set this to null so that the reference
                        // (local and remote) isn&#39;t kept in the mBroadcastHistory.
                        r.resultTo &#61; null;
                    } catch (RemoteException e) {
                        r.resultTo &#61; null;
                        Slog.w(TAG, &#34;Failure [&#34;
                                &#43; mQueueName &#43; &#34;] sending broadcast result of &#34;
                                &#43; r.intent, e);

                    }
                }
                
                if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, &#34;Cancelling BROADCAST_TIMEOUT_MSG&#34;);
                //取消BROADCAST_TIMEOUT_MSG消息
                cancelBroadcastTimeoutLocked();

                if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST,
                        &#34;Finished with ordered broadcast &#34; &#43; r);
                
                // 添加到队列消息记录
                // ... and on to the next...
                addBroadcastToHistoryLocked(r);
                if (r.intent.getComponent() &#61;&#61; null &amp;&amp; r.intent.getPackage() &#61;&#61; null
                        &amp;&amp; (r.intent.getFlags()&amp;Intent.FLAG_RECEIVER_REGISTERED_ONLY) &#61;&#61; 0) {
                    // This was an implicit broadcast... let&#39;s record it for posterity.
                    mService.addBroadcastStatLocked(r.intent.getAction(), r.callerPackage,
                            r.manifestCount, r.manifestSkipCount, r.finishTime-r.dispatchTime);
                }
                mOrderedBroadcasts.remove(0);
                r &#61; null;
                looped &#61; true;
                continue;
            }
        } while (r &#61;&#61; null);
</code></pre> 
<p>mTimeoutPeriod&#xff0c;对于前台广播为10s,后台广播为60s。广播超时为2 * mTimeoutPeriod * numReceivers&#xff0c;接收者个数numReceivers越多&#xff0c;则广播超时总时间越大。</p> 
<h4><a id="423__1792"></a>4.2.3 获取下条有序广播</h4> 
<pre><code>        //获取下条广播的index
        // Get the next receiver...
        int recIdx &#61; r.nextReceiver&#43;&#43;;

        // Keep track of when this receiver started, and make sure there
        // is a timeout message pending to kill it if need be.
        //记录开始时间
        r.receiverTime &#61; SystemClock.uptimeMillis();
        if (recIdx &#61;&#61; 0) {
            r.dispatchTime &#61; r.receiverTime;
            r.dispatchClockTime &#61; System.currentTimeMillis();
            if (Trace.isTagEnabled(Trace.TRACE_TAG_ACTIVITY_MANAGER)) {
                Trace.asyncTraceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                    createBroadcastTraceTitle(r, BroadcastRecord.DELIVERY_PENDING),
                    System.identityHashCode(r));
                Trace.asyncTraceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                    createBroadcastTraceTitle(r, BroadcastRecord.DELIVERY_DELIVERED),
                    System.identityHashCode(r));
            }
            if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST, &#34;Processing ordered broadcast [&#34;
                    &#43; mQueueName &#43; &#34;] &#34; &#43; r);
        }
        if (! mPendingBroadcastTimeoutMessage) {
            long timeoutTime &#61; r.receiverTime &#43; mTimeoutPeriod;
            if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST,
                    &#34;Submitting BROADCAST_TIMEOUT_MSG [&#34;
                    &#43; mQueueName &#43; &#34;] for &#34; &#43; r &#43; &#34; at &#34; &#43; timeoutTime);
            //发送广播超时        
            setBroadcastTimeoutLocked(timeoutTime);
        }

        final BroadcastOptions brOptions &#61; r.options;
        final Object nextReceiver &#61; r.receivers.get(recIdx);

        if (nextReceiver instanceof BroadcastFilter) {
            // Simple case: this is a registered receiver who gets
            // a direct call.
            //对于动态注册的广播接收者
            BroadcastFilter filter &#61; (BroadcastFilter)nextReceiver;
            if (DEBUG_BROADCAST)  Slog.v(TAG_BROADCAST,
                    &#34;Delivering ordered [&#34;
                    &#43; mQueueName &#43; &#34;] to registered &#34;
                    &#43; filter &#43; &#34;: &#34; &#43; r);
            //处理广播&#xff0c;见4.3节        
            deliverToRegisteredReceiverLocked(r, filter, r.ordered, recIdx);
            if (r.receiver &#61;&#61; null || !r.ordered) {
                // The receiver has already finished, so schedule to
                // process the next one.
                if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, &#34;Quick finishing [&#34;
                        &#43; mQueueName &#43; &#34;]: ordered&#61;&#34;
                        &#43; r.ordered &#43; &#34; receiver&#61;&#34; &#43; r.receiver);
                r.state &#61; BroadcastRecord.IDLE;
                //处理下一个广播
                scheduleBroadcastsLocked();
            } else {
                if (brOptions !&#61; null &amp;&amp; brOptions.getTemporaryAppWhitelistDuration() &gt; 0) {
                    scheduleTempWhitelistLocked(filter.owningUid,
                            brOptions.getTemporaryAppWhitelistDuration(), r);
                }
            }
            return;
        }

        // Hard case: need to instantiate the receiver, possibly
        // starting its application process to host it.
        //对于静态注册的广播接收者
        ResolveInfo info &#61;
            (ResolveInfo)nextReceiver;
        ComponentName component &#61; new ComponentName(
                info.activityInfo.applicationInfo.packageName,
                info.activityInfo.name);
       
        //下面都是各种检查权限&#xff0c;权限不满足时则跳过
        boolean skip &#61; false;
        if (brOptions !&#61; null &amp;&amp;
                (info.activityInfo.applicationInfo.targetSdkVersion
                        &lt; brOptions.getMinManifestReceiverApiLevel() ||
                info.activityInfo.applicationInfo.targetSdkVersion
                        &gt; brOptions.getMaxManifestReceiverApiLevel())) {
            skip &#61; true;  //sdk版本&#xff0c;Androidmanifest中设置
        }
        int perm &#61; mService.checkComponentPermission(info.activityInfo.permission,
                r.callingPid, r.callingUid, info.activityInfo.applicationInfo.uid,
                info.activityInfo.exported);
        //Component权限
        if (!skip &amp;&amp; perm !&#61; PackageManager.PERMISSION_GRANTED) {
            if (!info.activityInfo.exported) {
                Slog.w(TAG, &#34;Permission Denial: broadcasting &#34;
                        &#43; r.intent.toString()
                        &#43; &#34; from &#34; &#43; r.callerPackage &#43; &#34; (pid&#61;&#34; &#43; r.callingPid
                        &#43; &#34;, uid&#61;&#34; &#43; r.callingUid &#43; &#34;)&#34;
                        &#43; &#34; is not exported from uid &#34; &#43; info.activityInfo.applicationInfo.uid
                        &#43; &#34; due to receiver &#34; &#43; component.flattenToShortString());
            } else {
                Slog.w(TAG, &#34;Permission Denial: broadcasting &#34;
                        &#43; r.intent.toString()
                        &#43; &#34; from &#34; &#43; r.callerPackage &#43; &#34; (pid&#61;&#34; &#43; r.callingPid
                        &#43; &#34;, uid&#61;&#34; &#43; r.callingUid &#43; &#34;)&#34;
                        &#43; &#34; requires &#34; &#43; info.activityInfo.permission
                        &#43; &#34; due to receiver &#34; &#43; component.flattenToShortString());
            }
            skip &#61; true;
        } else if (!skip &amp;&amp; info.activityInfo.permission !&#61; null) {
            final int opCode &#61; AppOpsManager.permissionToOpCode(info.activityInfo.permission);
            if (opCode !&#61; AppOpsManager.OP_NONE
                    &amp;&amp; mService.mAppOpsService.noteOperation(opCode, r.callingUid,
                            r.callerPackage) !&#61; AppOpsManager.MODE_ALLOWED) {
                Slog.w(TAG, &#34;Appop Denial: broadcasting &#34;
                        &#43; r.intent.toString()
                        &#43; &#34; from &#34; &#43; r.callerPackage &#43; &#34; (pid&#61;&#34;
                        &#43; r.callingPid &#43; &#34;, uid&#61;&#34; &#43; r.callingUid &#43; &#34;)&#34;
                        &#43; &#34; requires appop &#34; &#43; AppOpsManager.permissionToOp(
                                info.activityInfo.permission)
                        &#43; &#34; due to registered receiver &#34;
                        &#43; component.flattenToShortString());
                skip &#61; true;
            }
        }
      
        if (!skip &amp;&amp; info.activityInfo.applicationInfo.uid !&#61; Process.SYSTEM_UID &amp;&amp;
            r.requiredPermissions !&#61; null &amp;&amp; r.requiredPermissions.length &gt; 0) {
            for (int i &#61; 0; i &lt; r.requiredPermissions.length; i&#43;&#43;) {
                String requiredPermission &#61; r.requiredPermissions[i];
                try {
                    perm &#61; AppGlobals.getPackageManager().
                            checkPermission(requiredPermission,
                                    info.activityInfo.applicationInfo.packageName,
                                    UserHandle
                                            .getUserId(info.activityInfo.applicationInfo.uid));
                } catch (RemoteException e) {
                    perm &#61; PackageManager.PERMISSION_DENIED;
                }
                if (perm !&#61; PackageManager.PERMISSION_GRANTED) {
                    Slog.w(TAG, &#34;Permission Denial: receiving &#34;
                            &#43; r.intent &#43; &#34; to &#34;
                            &#43; component.flattenToShortString()
                            &#43; &#34; requires &#34; &#43; requiredPermission
                            &#43; &#34; due to sender &#34; &#43; r.callerPackage
                            &#43; &#34; (uid &#34; &#43; r.callingUid &#43; &#34;)&#34;);
                    skip &#61; true;
                    break;
                }
                int appOp &#61; AppOpsManager.permissionToOpCode(requiredPermission);
                if (appOp !&#61; AppOpsManager.OP_NONE &amp;&amp; appOp !&#61; r.appOp
                        &amp;&amp; mService.mAppOpsService.noteOperation(appOp,
                        info.activityInfo.applicationInfo.uid, info.activityInfo.packageName)
                        !&#61; AppOpsManager.MODE_ALLOWED) {
                    Slog.w(TAG, &#34;Appop Denial: receiving &#34;
                            &#43; r.intent &#43; &#34; to &#34;
                            &#43; component.flattenToShortString()
                            &#43; &#34; requires appop &#34; &#43; AppOpsManager.permissionToOp(
                            requiredPermission)
                            &#43; &#34; due to sender &#34; &#43; r.callerPackage
                            &#43; &#34; (uid &#34; &#43; r.callingUid &#43; &#34;)&#34;);
                    skip &#61; true;
                    break;
                }
            }
        }
        if (!skip &amp;&amp; r.appOp !&#61; AppOpsManager.OP_NONE
                &amp;&amp; mService.mAppOpsService.noteOperation(r.appOp,
                info.activityInfo.applicationInfo.uid, info.activityInfo.packageName)
                !&#61; AppOpsManager.MODE_ALLOWED) {
            Slog.w(TAG, &#34;Appop Denial: receiving &#34;
                    &#43; r.intent &#43; &#34; to &#34;
                    &#43; component.flattenToShortString()
                    &#43; &#34; requires appop &#34; &#43; AppOpsManager.opToName(r.appOp)
                    &#43; &#34; due to sender &#34; &#43; r.callerPackage
                    &#43; &#34; (uid &#34; &#43; r.callingUid &#43; &#34;)&#34;);
            skip &#61; true;
        }
        if (!skip) {
            skip &#61; !mService.mIntentFirewall.checkBroadcast(r.intent, r.callingUid,
                    r.callingPid, r.resolvedType, info.activityInfo.applicationInfo.uid);
        }
        boolean isSingleton &#61; false;
        try {
            isSingleton &#61; mService.isSingleton(info.activityInfo.processName,
                    info.activityInfo.applicationInfo,
                    info.activityInfo.name, info.activityInfo.flags);
        } catch (SecurityException e) {
            Slog.w(TAG, e.getMessage());
            skip &#61; true;
        }
        if ((info.activityInfo.flags&amp;ActivityInfo.FLAG_SINGLE_USER) !&#61; 0) {
            if (ActivityManager.checkUidPermission(
                    android.Manifest.permission.INTERACT_ACROSS_USERS,
                    info.activityInfo.applicationInfo.uid)
                            !&#61; PackageManager.PERMISSION_GRANTED) {
                Slog.w(TAG, &#34;Permission Denial: Receiver &#34; &#43; component.flattenToShortString()
                        &#43; &#34; requests FLAG_SINGLE_USER, but app does not hold &#34;
                        &#43; android.Manifest.permission.INTERACT_ACROSS_USERS);
                skip &#61; true;
            }
        }
        if (!skip &amp;&amp; info.activityInfo.applicationInfo.isInstantApp()
                &amp;&amp; r.callingUid !&#61; info.activityInfo.applicationInfo.uid) {
            Slog.w(TAG, &#34;Instant App Denial: receiving &#34;
                    &#43; r.intent
                    &#43; &#34; to &#34; &#43; component.flattenToShortString()
                    &#43; &#34; due to sender &#34; &#43; r.callerPackage
                    &#43; &#34; (uid &#34; &#43; r.callingUid &#43; &#34;)&#34;
                    &#43; &#34; Instant Apps do not support manifest receivers&#34;);
            skip &#61; true;
        }
        if (!skip &amp;&amp; r.callerInstantApp
                &amp;&amp; (info.activityInfo.flags &amp; ActivityInfo.FLAG_VISIBLE_TO_INSTANT_APP) &#61;&#61; 0
                &amp;&amp; r.callingUid !&#61; info.activityInfo.applicationInfo.uid) {
            Slog.w(TAG, &#34;Instant App Denial: receiving &#34;
                    &#43; r.intent
                    &#43; &#34; to &#34; &#43; component.flattenToShortString()
                    &#43; &#34; requires receiver have visibleToInstantApps set&#34;
                    &#43; &#34; due to sender &#34; &#43; r.callerPackage
                    &#43; &#34; (uid &#34; &#43; r.callingUid &#43; &#34;)&#34;);
            skip &#61; true;
        }
        if (r.curApp !&#61; null &amp;&amp; r.curApp.crashing) {
            // If the target process is crashing, just skip it.
            Slog.w(TAG, &#34;Skipping deliver ordered [&#34; &#43; mQueueName &#43; &#34;] &#34; &#43; r
                    &#43; &#34; to &#34; &#43; r.curApp &#43; &#34;: process crashing&#34;);
            skip &#61; true;
        }
        if (!skip) {
            boolean isAvailable &#61; false;
            try {
                isAvailable &#61; AppGlobals.getPackageManager().isPackageAvailable(
                        info.activityInfo.packageName,
                        UserHandle.getUserId(info.activityInfo.applicationInfo.uid));
            } catch (Exception e) {
                // all such failures mean we skip this receiver
                Slog.w(TAG, &#34;Exception getting recipient info for &#34;
                        &#43; info.activityInfo.packageName, e);
            }
            if (!isAvailable) {
                if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST,
                        &#34;Skipping delivery to &#34; &#43; info.activityInfo.packageName &#43; &#34; / &#34;
                        &#43; info.activityInfo.applicationInfo.uid
                        &#43; &#34; : package no longer available&#34;);
                skip &#61; true;
            }
        }

        // If permissions need a review before any of the app components can run, we drop
        // the broadcast and if the calling app is in the foreground and the broadcast is
        // explicit we launch the review UI passing it a pending intent to send the skipped
        // broadcast.
        if (mService.mPermissionReviewRequired &amp;&amp; !skip) {
            if (!requestStartTargetPermissionsReviewIfNeededLocked(r,
                    info.activityInfo.packageName, UserHandle.getUserId(
                            info.activityInfo.applicationInfo.uid))) {
                skip &#61; true;
            }
        }
           // This is safe to do even if we are skipping the broadcast, and we need
        // this information now to evaluate whether it is going to be allowed to run.
        final int receiverUid &#61; info.activityInfo.applicationInfo.uid;
        // If it&#39;s a singleton, it needs to be the same app or a special app
        if (r.callingUid !&#61; Process.SYSTEM_UID &amp;&amp; isSingleton
                &amp;&amp; mService.isValidSingletonCall(r.callingUid, receiverUid)) {
            info.activityInfo &#61; mService.getActivityInfoForUser(info.activityInfo, 0);
        }
        String targetProcess &#61; info.activityInfo.processName;
        ProcessRecord app &#61; mService.getProcessRecordLocked(targetProcess,
                info.activityInfo.applicationInfo.uid, false);

        if (!skip) {
            final int allowed &#61; mService.getAppStartModeLocked(
                    info.activityInfo.applicationInfo.uid, info.activityInfo.packageName,
                    info.activityInfo.applicationInfo.targetSdkVersion, -1, true, false, false);
            if (allowed !&#61; ActivityManager.APP_START_MODE_NORMAL) {
                // We won&#39;t allow this receiver to be launched if the app has been
                // completely disabled from launches, or it was not explicitly sent
                // to it and the app is in a state that should not receive it
                // (depending on how getAppStartModeLocked has determined that).
                if (allowed &#61;&#61; ActivityManager.APP_START_MODE_DISABLED) {
                    Slog.w(TAG, &#34;Background execution disabled: receiving &#34;
                            &#43; r.intent &#43; &#34; to &#34;
                            &#43; component.flattenToShortString());
                    skip &#61; true;
                } else if (((r.intent.getFlags()&amp;Intent.FLAG_RECEIVER_EXCLUDE_BACKGROUND) !&#61; 0)
                        || (r.intent.getComponent() &#61;&#61; null
                            &amp;&amp; r.intent.getPackage() &#61;&#61; null
                            &amp;&amp; ((r.intent.getFlags()
                                    &amp; Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND) &#61;&#61; 0)
                            &amp;&amp; !isSignaturePerm(r.requiredPermissions))) {
                    mService.addBackgroundCheckViolationLocked(r.intent.getAction(),
                            component.getPackageName());
                    Slog.w(TAG, &#34;Background execution not allowed: receiving &#34;
                            &#43; r.intent &#43; &#34; to &#34;
                            &#43; component.flattenToShortString());
                    skip &#61; true;
                }
            }
        }

        if (!skip &amp;&amp; !Intent.ACTION_SHUTDOWN.equals(r.intent.getAction())
                &amp;&amp; !mService.mUserController
                .isUserRunning(UserHandle.getUserId(info.activityInfo.applicationInfo.uid),
                        0 /* flags */)) {
            skip &#61; true;
            Slog.w(TAG,
                    &#34;Skipping delivery to &#34; &#43; info.activityInfo.packageName &#43; &#34; / &#34;
                            &#43; info.activityInfo.applicationInfo.uid &#43; &#34; : user is not running&#34;);
        }
        
        //跳过该广播
        if (skip) {
            if (DEBUG_BROADCAST)  Slog.v(TAG_BROADCAST,
                    &#34;Skipping delivery of ordered [&#34; &#43; mQueueName &#43; &#34;] &#34;
                    &#43; r &#43; &#34; for whatever reason&#34;);
            r.delivery[recIdx] &#61; BroadcastRecord.DELIVERY_SKIPPED;
            r.receiver &#61; null;
            r.curFilter &#61; null;
            r.state &#61; BroadcastRecord.IDLE;
            r.manifestSkipCount&#43;&#43;;
            scheduleBroadcastsLocked();
            return;
        }
        r.manifestCount&#43;&#43;;

        r.delivery[recIdx] &#61; BroadcastRecord.DELIVERY_DELIVERED;
        r.state &#61; BroadcastRecord.APP_RECEIVE;
        r.curComponent &#61; component;
        r.curReceiver &#61; info.activityInfo;
        if (DEBUG_MU &amp;&amp; r.callingUid &gt; UserHandle.PER_USER_RANGE) {
            Slog.v(TAG_MU, &#34;Updated broadcast record activity info for secondary user, &#34;
                    &#43; info.activityInfo &#43; &#34;, callingUid &#61; &#34; &#43; r.callingUid &#43; &#34;, uid &#61; &#34;
                    &#43; receiverUid);
        }

        if (brOptions !&#61; null &amp;&amp; brOptions.getTemporaryAppWhitelistDuration() &gt; 0) {
            scheduleTempWhitelistLocked(receiverUid,
                    brOptions.getTemporaryAppWhitelistDuration(), r);
        }
        // Broadcast正在执行&#xff0c;stopped状态设置成false
        // Broadcast is being executed, its package can&#39;t be stopped.
        try {
            AppGlobals.getPackageManager().setPackageStoppedState(
                    r.curComponent.getPackageName(), false, UserHandle.getUserId(r.callingUid));
        } catch (RemoteException e) {
        } catch (IllegalArgumentException e) {
            Slog.w(TAG, &#34;Failed trying to unstop package &#34;
                    &#43; r.curComponent.getPackageName() &#43; &#34;: &#34; &#43; e);
        }

</code></pre> 
<p>发送广播超时&#xff0c;如果是动态注册的广播则执行deliverToRegisteredReceiverLocked方法&#xff0c;如果是静态注册的广播&#xff0c;则进行一系列的权限检查&#xff0c;不满足则跳过该广播。</p> 
<h4><a id="424__2144"></a>4.2.4 处理下条有序广播</h4> 
<pre><code>         // Is this receiver&#39;s application already running?
        if (app !&#61; null &amp;&amp; app.thread !&#61; null &amp;&amp; !app.killed) {
            // 接收广播的进程还在运行
            try {
                app.addPackage(info.activityInfo.packageName,
                        info.activityInfo.applicationInfo.versionCode, mService.mProcessStats);
                processCurBroadcastLocked(r, app, skipOomAdj);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, &#34;Exception when sending broadcast to &#34;
                      &#43; r.curComponent, e);
            } catch (RuntimeException e) {
                Slog.wtf(TAG, &#34;Failed sending broadcast to &#34;
                        &#43; r.curComponent &#43; &#34; with &#34; &#43; r.intent, e);
                // If some unexpected exception happened, just skip
                // this broadcast.  At this point we are not in the call
                // from a client, so throwing an exception out from here
                // will crash the entire system instead of just whoever
                // sent the broadcast.
                logBroadcastReceiverDiscardLocked(r);
                finishReceiverLocked(r, r.resultCode, r.resultData,
                        r.resultExtras, r.resultAbort, false);
                scheduleBroadcastsLocked();
                //如果启动receiver失败&#xff0c;则重置状态
                // We need to reset the state if we failed to start the receiver.
                r.state &#61; BroadcastRecord.IDLE;
                return;
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }

        // Not running -- get it started, to be executed when the app comes up.
        if (DEBUG_BROADCAST)  Slog.v(TAG_BROADCAST,
                &#34;Need to start app [&#34;
                &#43; mQueueName &#43; &#34;] &#34; &#43; targetProcess &#43; &#34; for broadcast &#34; &#43; r);
        //receiver所对应的进程尚未启动&#xff0c;则创建该进程        
        if ((r.curApp&#61;mService.startProcessLocked(targetProcess,
                info.activityInfo.applicationInfo, true,
                r.intent.getFlags() | Intent.FLAG_FROM_BACKGROUND,
                &#34;broadcast&#34;, r.curComponent,
                (r.intent.getFlags()&amp;Intent.FLAG_RECEIVER_BOOT_UPGRADE) !&#61; 0, false, false))
                        &#61;&#61; null) {
            // Ah, this recipient is unavailable.  Finish it if necessary,
            // and mark the broadcast record as ready for the next.
            Slog.w(TAG, &#34;Unable to launch app &#34;
                    &#43; info.activityInfo.applicationInfo.packageName &#43; &#34;/&#34;
                    &#43; receiverUid &#43; &#34; for broadcast &#34;
                    &#43; r.intent &#43; &#34;: process is bad&#34;);
            //创建失败&#xff0c;直接结束该receiver        
            logBroadcastReceiverDiscardLocked(r);
            finishReceiverLocked(r, r.resultCode, r.resultData,
                    r.resultExtras, r.resultAbort, false);
            //开启下一个广播
            scheduleBroadcastsLocked();
            r.state &#61; BroadcastRecord.IDLE;
            return;
        }

        mPendingBroadcast &#61; r;
        mPendingBroadcastRecvIndex &#61; recIdx;
</code></pre> 
<ul><li> <p>如果是动态广播接收者&#xff0c;则调用deliverToRegisteredReceiverLocked处理</p> </li><li> <p>如果是静态广播接收者&#xff0c;且对应进程已经创建&#xff0c;则调用processCurBroadcastLocked处理</p> </li><li> <p>如果是静态广播接收者&#xff0c;且对应进程未创建&#xff0c;则调用startProcessLocked创建进程</p> </li></ul> 
<h3><a id="43__BQdeliverToRegisteredReceiverLocked_2217"></a>4.3 BQ.deliverToRegisteredReceiverLocked</h3> 
<p>[-&gt;BroadcastQueue.java]</p> 
<pre><code> private void deliverToRegisteredReceiverLocked(BroadcastRecord r,
            BroadcastFilter filter, boolean ordered, int index) {
        boolean skip &#61; false;
        //检查发送者是否有BroadcastFilter所需要的权限
        if (filter.requiredPermission !&#61; null) {
            int perm &#61; mService.checkComponentPermission(filter.requiredPermission,
                    r.callingPid, r.callingUid, -1, true);
            if (perm !&#61; PackageManager.PERMISSION_GRANTED) {
                Slog.w(TAG, &#34;Permission Denial: broadcasting &#34;
                        &#43; r.intent.toString()
                        &#43; &#34; from &#34; &#43; r.callerPackage &#43; &#34; (pid&#61;&#34;
                        &#43; r.callingPid &#43; &#34;, uid&#61;&#34; &#43; r.callingUid &#43; &#34;)&#34;
                        &#43; &#34; requires &#34; &#43; filter.requiredPermission
                        &#43; &#34; due to registered receiver &#34; &#43; filter);
                skip &#61; true;
            } else {
                final int opCode &#61; AppOpsManager.permissionToOpCode(filter.requiredPermission);
                if (opCode !&#61; AppOpsManager.OP_NONE
                        &amp;&amp; mService.mAppOpsService.noteOperation(opCode, r.callingUid,
                                r.callerPackage) !&#61; AppOpsManager.MODE_ALLOWED) {
                    Slog.w(TAG, &#34;Appop Denial: broadcasting &#34;
                            &#43; r.intent.toString()
                            &#43; &#34; from &#34; &#43; r.callerPackage &#43; &#34; (pid&#61;&#34;
                            &#43; r.callingPid &#43; &#34;, uid&#61;&#34; &#43; r.callingUid &#43; &#34;)&#34;
                            &#43; &#34; requires appop &#34; &#43; AppOpsManager.permissionToOp(
                                    filter.requiredPermission)
                            &#43; &#34; due to registered receiver &#34; &#43; filter);
                    skip &#61; true;
                }
            }
        }
        if (!skip &amp;&amp; r.requiredPermissions !&#61; null &amp;&amp; r.requiredPermissions.length &gt; 0) {
            for (int i &#61; 0; i &lt; r.requiredPermissions.length; i&#43;&#43;) {
                String requiredPermission &#61; r.requiredPermissions[i];
                int perm &#61; mService.checkComponentPermission(requiredPermission,
                        filter.receiverList.pid, filter.receiverList.uid, -1, true);
                if (perm !&#61; PackageManager.PERMISSION_GRANTED) {
                    Slog.w(TAG, &#34;Permission Denial: receiving &#34;
                            &#43; r.intent.toString()
                            &#43; &#34; to &#34; &#43; filter.receiverList.app
                            &#43; &#34; (pid&#61;&#34; &#43; filter.receiverList.pid
                            &#43; &#34;, uid&#61;&#34; &#43; filter.receiverList.uid &#43; &#34;)&#34;
                            &#43; &#34; requires &#34; &#43; requiredPermission
                            &#43; &#34; due to sender &#34; &#43; r.callerPackage
                            &#43; &#34; (uid &#34; &#43; r.callingUid &#43; &#34;)&#34;);
                    skip &#61; true;
                    break;
                }
                int appOp &#61; AppOpsManager.permissionToOpCode(requiredPermission);
                if (appOp !&#61; AppOpsManager.OP_NONE &amp;&amp; appOp !&#61; r.appOp
                        &amp;&amp; mService.mAppOpsService.noteOperation(appOp,
                        filter.receiverList.uid, filter.packageName)
                        !&#61; AppOpsManager.MODE_ALLOWED) {
                    Slog.w(TAG, &#34;Appop Denial: receiving &#34;
                            &#43; r.intent.toString()
                            &#43; &#34; to &#34; &#43; filter.receiverList.app
                            &#43; &#34; (pid&#61;&#34; &#43; filter.receiverList.pid
                            &#43; &#34;, uid&#61;&#34; &#43; filter.receiverList.uid &#43; &#34;)&#34;
                            &#43; &#34; requires appop &#34; &#43; AppOpsManager.permissionToOp(
                            requiredPermission)
                            &#43; &#34; due to sender &#34; &#43; r.callerPackage
                            &#43; &#34; (uid &#34; &#43; r.callingUid &#43; &#34;)&#34;);
                    skip &#61; true;
                    break;
                }
            }
        }
        if (!skip &amp;&amp; (r.requiredPermissions &#61;&#61; null || r.requiredPermissions.length &#61;&#61; 0)) {
            int perm &#61; mService.checkComponentPermission(null,
                    filter.receiverList.pid, filter.receiverList.uid, -1, true);
            if (perm !&#61; PackageManager.PERMISSION_GRANTED) {
                Slog.w(TAG, &#34;Permission Denial: security check failed when receiving &#34;
                        &#43; r.intent.toString()
                        &#43; &#34; to &#34; &#43; filter.receiverList.app
                        &#43; &#34; (pid&#61;&#34; &#43; filter.receiverList.pid
                        &#43; &#34;, uid&#61;&#34; &#43; filter.receiverList.uid &#43; &#34;)&#34;
                        &#43; &#34; due to sender &#34; &#43; r.callerPackage
                        &#43; &#34; (uid &#34; &#43; r.callingUid &#43; &#34;)&#34;);
                skip &#61; true;
            }
        }
        if (!skip &amp;&amp; r.appOp !&#61; AppOpsManager.OP_NONE
                &amp;&amp; mService.mAppOpsService.noteOperation(r.appOp,
                filter.receiverList.uid, filter.packageName)
                !&#61; AppOpsManager.MODE_ALLOWED) {
            Slog.w(TAG, &#34;Appop Denial: receiving &#34;
                    &#43; r.intent.toString()
                    &#43; &#34; to &#34; &#43; filter.receiverList.app
                    &#43; &#34; (pid&#61;&#34; &#43; filter.receiverList.pid
                    &#43; &#34;, uid&#61;&#34; &#43; filter.receiverList.uid &#43; &#34;)&#34;
                    &#43; &#34; requires appop &#34; &#43; AppOpsManager.opToName(r.appOp)
                    &#43; &#34; due to sender &#34; &#43; r.callerPackage
                    &#43; &#34; (uid &#34; &#43; r.callingUid &#43; &#34;)&#34;);
            skip &#61; true;
        }

        if (!mService.mIntentFirewall.checkBroadcast(r.intent, r.callingUid,
                r.callingPid, r.resolvedType, filter.receiverList.uid)) {
            skip &#61; true;
        }

        if (!skip &amp;&amp; (filter.receiverList.app &#61;&#61; null || filter.receiverList.app.killed
                || filter.receiverList.app.crashing)) {
            Slog.w(TAG, &#34;Skipping deliver [&#34; &#43; mQueueName &#43; &#34;] &#34; &#43; r
                    &#43; &#34; to &#34; &#43; filter.receiverList &#43; &#34;: process gone or crashing&#34;);
            skip &#61; true;
        }
        //即时应用检查
        // Ensure that broadcasts are only sent to other Instant Apps if they are marked as
        // visible to Instant Apps.
        final boolean visibleToInstantApps &#61;
                (r.intent.getFlags() &amp; Intent.FLAG_RECEIVER_VISIBLE_TO_INSTANT_APPS) !&#61; 0;

        if (!skip &amp;&amp; !visibleToInstantApps &amp;&amp; filter.instantApp
                &amp;&amp; filter.receiverList.uid !&#61; r.callingUid) {
            Slog.w(TAG, &#34;Instant App Denial: receiving &#34;
                    &#43; r.intent.toString()
                    &#43; &#34; to &#34; &#43; filter.receiverList.app
                    &#43; &#34; (pid&#61;&#34; &#43; filter.receiverList.pid
                    &#43; &#34;, uid&#61;&#34; &#43; filter.receiverList.uid &#43; &#34;)&#34;
                    &#43; &#34; due to sender &#34; &#43; r.callerPackage
                    &#43; &#34; (uid &#34; &#43; r.callingUid &#43; &#34;)&#34;
                    &#43; &#34; not specifying FLAG_RECEIVER_VISIBLE_TO_INSTANT_APPS&#34;);
            skip &#61; true;
        }

        if (!skip &amp;&amp; !filter.visibleToInstantApp &amp;&amp; r.callerInstantApp
                &amp;&amp; filter.receiverList.uid !&#61; r.callingUid) {
            Slog.w(TAG, &#34;Instant App Denial: receiving &#34;
                    &#43; r.intent.toString()
                    &#43; &#34; to &#34; &#43; filter.receiverList.app
                    &#43; &#34; (pid&#61;&#34; &#43; filter.receiverList.pid
                    &#43; &#34;, uid&#61;&#34; &#43; filter.receiverList.uid &#43; &#34;)&#34;
                    &#43; &#34; requires receiver be visible to instant apps&#34;
                    &#43; &#34; due to sender &#34; &#43; r.callerPackage
                    &#43; &#34; (uid &#34; &#43; r.callingUid &#43; &#34;)&#34;);
            skip &#61; true;
        }

        if (skip) {
            r.delivery[index] &#61; BroadcastRecord.DELIVERY_SKIPPED;
            return;
        }

        // If permissions need a review before any of the app components can run, we drop
        // the broadcast and if the calling app is in the foreground and the broadcast is
        // explicit we launch the review UI passing it a pending intent to send the skipped
        // broadcast.
        if (mService.mPermissionReviewRequired) {
            if (!requestStartTargetPermissionsReviewIfNeededLocked(r, filter.packageName,
                    filter.owningUserId)) {
                r.delivery[index] &#61; BroadcastRecord.DELIVERY_SKIPPED;
                return;
            }
        }

        r.delivery[index] &#61; BroadcastRecord.DELIVERY_DELIVERED;
        //如果是有序广播
        // If this is not being sent as an ordered broadcast, then we
        // don&#39;t want to touch the fields that keep track of the current
        // state of ordered broadcasts.
        if (ordered) {
            r.receiver &#61; filter.receiverList.receiver.asBinder();
            r.curFilter &#61; filter;
            filter.receiverList.curBroadcast &#61; r;
            r.state &#61; BroadcastRecord.CALL_IN_RECEIVE;
            if (filter.receiverList.app !&#61; null) {
                // Bump hosting application to no longer be in background
                // scheduling class.  Note that we can&#39;t do that if there
                // isn&#39;t an app...  but we can only be in that case for
                // things that directly call the IActivityManager API, which
                // are already core system stuff so don&#39;t matter for this.
                r.curApp &#61; filter.receiverList.app;
                filter.receiverList.app.curReceivers.add(r);
                mService.updateOomAdjLocked(r.curApp, true);
            }
        }
        try {
            if (DEBUG_BROADCAST_LIGHT) Slog.i(TAG_BROADCAST,
                    &#34;Delivering to &#34; &#43; filter &#43; &#34; : &#34; &#43; r);
            if (filter.receiverList.app !&#61; null &amp;&amp; filter.receiverList.app.inFullBackup) {
                // Skip delivery if full backup in progress
                // If it&#39;s an ordered broadcast, we need to continue to the next receiver.
                if (ordered) {
                    //跳过有序广播
                    skipReceiverLocked(r);
                }
            } else {
               //处理广播&#xff0c;见4.4节
                performReceiveLocked(filter.receiverList.app, filter.receiverList.receiver,
                        new Intent(r.intent), r.resultCode, r.resultData,
                        r.resultExtras, r.ordered, r.initialSticky, r.userId);
            }
            if (ordered) {
                r.state &#61; BroadcastRecord.CALL_DONE_RECEIVE;
            }
        } catch (RemoteException e) {
            Slog.w(TAG, &#34;Failure sending broadcast &#34; &#43; r.intent, e);
            if (ordered) {
                r.receiver &#61; null;
                r.curFilter &#61; null;
                filter.receiverList.curBroadcast &#61; null;
                if (filter.receiverList.app !&#61; null) {
                    filter.receiverList.app.curReceivers.remove(r);
                }
            }
        }
    }
</code></pre> 
<p>这里是检查动态注册的广播的相关权限&#xff0c;如果是有序广播则跳过。最后执行performReceiveLocked处理并行广播。</p> 
<h3><a id="44__BQperformReceiveLocked_2434"></a>4.4 BQ.performReceiveLocked</h3> 
<p>[-&gt;BroadcastQueue.java]</p> 
<pre><code>void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
            Intent intent, int resultCode, String data, Bundle extras,
            boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
        // Send the intent to the receiver asynchronously using one-way binder calls.
        //通过binder机制&#xff0c;发送到广播接收者
        if (app !&#61; null) {
            if (app.thread !&#61; null) {
                // If we have an app thread, do the call through that so it is
                // correctly ordered with other one-way calls.
                try {
                    //见4.5节
                    app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                            data, extras, ordered, sticky, sendingUser, app.repProcState);
                // TODO: Uncomment this when (b/28322359) is fixed and we aren&#39;t getting
                // DeadObjectException when the process isn&#39;t actually dead.
                //} catch (DeadObjectException ex) {
                // Failed to call into the process.  It&#39;s dying so just let it die and move on.
                //    throw ex;
                } catch (RemoteException ex) {
                    // Failed to call into the process. It&#39;s either dying or wedged. Kill it gently.
                    synchronized (mService) {
                        Slog.w(TAG, &#34;Can&#39;t deliver broadcast to &#34; &#43; app.processName
                                &#43; &#34; (pid &#34; &#43; app.pid &#43; &#34;). Crashing it.&#34;);
                        app.scheduleCrash(&#34;can&#39;t deliver broadcast&#34;);
                    }
                    throw ex;
                }
            } else {
                //应有进程死亡&#xff0c;则应有不存在
                // Application has died. Receiver doesn&#39;t exist.
                throw new RemoteException(&#34;app.thread must not be null&#34;);
            }
        } else {
            //调用者进程为空
            receiver.performReceive(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);
        }
    }
</code></pre> 
<p>通过app.thread获取到IApplicationThread代理类&#xff0c;通过这个代理类&#xff0c;访问到ApplicationThread中的方法。</p> 
<h3><a id="45__ATscheduleRegisteredReceiver_2481"></a>4.5 AT.scheduleRegisteredReceiver</h3> 
<p>[-&gt;ActivityThread.java::ApplicationThread]</p> 
<pre><code>        // This function exists to make sure all receiver dispatching is
        // correctly ordered, since these are one-way calls and the binder driver
        // applies transaction ordering per object for such calls.
        public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
                int resultCode, String dataStr, Bundle extras, boolean ordered,
                boolean sticky, int sendingUser, int processState) throws RemoteException {
            //更新进程状态
            updateProcessState(processState, false);
            //见4.6节&#xff0c;其中receiver为2.3.2节所创建
            receiver.performReceive(intent, resultCode, dataStr, extras, ordered,
                    sticky, sendingUser);
        }
</code></pre> 
<p>receiver是注册广播时创建的&#xff0c;见2.3.3节 receiver&#61;LoadedApk.ReceiverDispatcher.InnerReceiver</p> 
<h3><a id="46_IRperformReceive_2502"></a>4.6 IR.performReceive</h3> 
<p>[-&gt;LoadedApk.java::InnerReceiver]</p> 
<pre><code>         public void performReceive(Intent intent, int resultCode, String data,
                    Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                final LoadedApk.ReceiverDispatcher rd;
                if (intent &#61;&#61; null) {
                    Log.wtf(TAG, &#34;Null intent received&#34;);
                    rd &#61; null;
                } else {
                    rd &#61; mDispatcher.get();
                }
                if (ActivityThread.DEBUG_BROADCAST) {
                    int seq &#61; intent.getIntExtra(&#34;seq&#34;, -1);
                    Slog.i(ActivityThread.TAG, &#34;Receiving broadcast &#34; &#43; intent.getAction()
                            &#43; &#34; seq&#61;&#34; &#43; seq &#43; &#34; to &#34; &#43; (rd !&#61; null ? rd.mReceiver : null));
                }
                if (rd !&#61; null) {
                    //见4.7节
                    rd.performReceive(intent, resultCode, data, extras,
                            ordered, sticky, sendingUser);
                } else {
                    // The activity manager dispatched a broadcast to a registered
                    // receiver in this process, but before it could be delivered the
                    // receiver was unregistered.  Acknowledge the broadcast on its
                    // behalf so that the system&#39;s broadcast sequence can continue.
                    if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                            &#34;Finishing broadcast to unregistered receiver&#34;);
                    IActivityManager mgr &#61; ActivityManager.getService();
                    try {
                        if (extras !&#61; null) {
                            extras.setAllowFds(false);
                        }
                        mgr.finishReceiver(this, resultCode, data, extras, false, intent.getFlags());
                    } catch (RemoteException e) {
                        throw e.rethrowFromSystemServer();
                    }
                }
            }
</code></pre> 
<h3><a id="47__RDperformReceive_2545"></a>4.7 RD.performReceive</h3> 
<p>[-&gt;LoadedApk.java::ReceiverDispatcher]</p> 
<pre><code>  public void performReceive(Intent intent, int resultCode, String data,
                Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
            final Args args &#61; new Args(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);
            if (intent &#61;&#61; null) {
                Log.wtf(TAG, &#34;Null intent received&#34;);
            } else {
                if (ActivityThread.DEBUG_BROADCAST) {
                    int seq &#61; intent.getIntExtra(&#34;seq&#34;, -1);
                    Slog.i(ActivityThread.TAG, &#34;Enqueueing broadcast &#34; &#43; intent.getAction()
                            &#43; &#34; seq&#61;&#34; &#43; seq &#43; &#34; to &#34; &#43; mReceiver);
                }
            }
            //见4.8节
            if (intent &#61;&#61; null || !mActivityThread.post(args.getRunnable())) {
                 //消息post到主线程&#xff0c;则不会走这里
                if (mRegistered &amp;&amp; ordered) {
                    IActivityManager mgr &#61; ActivityManager.getService();
                    if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                            &#34;Finishing sync broadcast to &#34; &#43; mReceiver);
                    args.sendFinished(mgr);
                }
            }
        }
</code></pre> 
<p>Args继承于BroadcastReceiver.PendingResult&#xff0c;实现了Runnable&#xff0c;其中mActivityThread是当前的主线程&#xff0c;在2.3.1节完成赋值&#xff0c;post消息机制&#xff0c;将消息放入MessageQueue&#xff0c;该消息是Runnable,而后执行该Runnable。</p> 
<h3><a id="48__ReceiverDispatcherArgsgetRunnable_2578"></a>4.8 ReceiverDispatcher.Args.getRunnable</h3> 
<p>[-&gt;LoadedApk.java]</p> 
<pre><code>      public final Runnable getRunnable() {
                return () -&gt; {
                    final BroadcastReceiver receiver &#61; mReceiver;
                    final boolean ordered &#61; mOrdered;

                    if (ActivityThread.DEBUG_BROADCAST) {
                        int seq &#61; mCurIntent.getIntExtra(&#34;seq&#34;, -1);
                        Slog.i(ActivityThread.TAG, &#34;Dispatching broadcast &#34; &#43; mCurIntent.getAction()
                                &#43; &#34; seq&#61;&#34; &#43; seq &#43; &#34; to &#34; &#43; mReceiver);
                        Slog.i(ActivityThread.TAG, &#34;  mRegistered&#61;&#34; &#43; mRegistered
                                &#43; &#34; mOrderedHint&#61;&#34; &#43; ordered);
                    }

                    final IActivityManager mgr &#61; ActivityManager.getService();
                    final Intent intent &#61; mCurIntent;
                    if (intent &#61;&#61; null) {
                        Log.wtf(TAG, &#34;Null intent being dispatched, mDispatched&#61;&#34; &#43; mDispatched
                                &#43; &#34;: run() previously called at &#34;
                                &#43; Log.getStackTraceString(mPreviousRunStacktrace));
                    }

                    mCurIntent &#61; null;
                    mDispatched &#61; true;
                    mPreviousRunStacktrace &#61; new Throwable(&#34;Previous stacktrace&#34;);
                    if (receiver &#61;&#61; null || intent &#61;&#61; null || mForgotten) {
                        if (mRegistered &amp;&amp; ordered) {
                            if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                                    &#34;Finishing null broadcast to &#34; &#43; mReceiver);
                            sendFinished(mgr);
                        }
                        return;
                    }

                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, &#34;broadcastReceiveReg&#34;);
                    try {
                        //获取mReceiver的类加载器
                        ClassLoader cl &#61; mReceiver.getClass().getClassLoader();
                        intent.setExtrasClassLoader(cl);
                        intent.prepareToEnterProcess();
                        setExtrasClassLoader(cl);
                        receiver.setPendingResult(this);
                        //调用receiver的onReceive回调
                        receiver.onReceive(mContext, intent);
                    } catch (Exception e) {
                        if (mRegistered &amp;&amp; ordered) {
                            if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                                    &#34;Finishing failed broadcast to &#34; &#43; mReceiver);
                            sendFinished(mgr);
                        }
                        if (mInstrumentation &#61;&#61; null ||
                                !mInstrumentation.onException(mReceiver, e)) {
                            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                            throw new RuntimeException(
                                    &#34;Error receiving broadcast &#34; &#43; intent
                                            &#43; &#34; in &#34; &#43; mReceiver, e);
                        }
                    }
                  
                    if (receiver.getPendingResult() !&#61; null) {
                       //见4.9节
                        finish();
                    }
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                };
            }
</code></pre> 
<h3><a id="49__PendingResultfinish_2650"></a>4.9 PendingResult.finish</h3> 
<p>[-&gt;BroadcastReceiver.java::PendingResult]</p> 
<pre><code>        /* Finish the broadcast.  The current result will be sent and the
         * next broadcast will proceed.
         */
        public final void finish() {
            if (mType &#61;&#61; TYPE_COMPONENT) {//代表静态注册的广播
                final IActivityManager mgr &#61; ActivityManager.getService();
                if (QueuedWork.hasPendingWork()) {
                    // If this is a broadcast component, we need to make sure any
                    // queued work is complete before telling AM we are done, so
                    // we don&#39;t have our process killed before that.  We now know
                    // there is pending work; put another piece of work at the end
                    // of the list to finish the broadcast, so we don&#39;t block this
                    // thread (which may be the main thread) to have it finished.
                    //
                    // Note that we don&#39;t need to use QueuedWork.addFinisher() with the
                    // runnable, since we know the AM is waiting for us until the
                    // executor gets to it.
                    QueuedWork.queue(new Runnable() {
                        &#64;Override public void run() {
                            if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                                    &#34;Finishing broadcast after work to component &#34; &#43; mToken);
                            sendFinished(mgr);//见4.9.1节
                        }
                    }, false);
                } else {
                    if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                            &#34;Finishing broadcast to component &#34; &#43; mToken);
                    sendFinished(mgr);//见4.9.1节
                }
            } else if (mOrderedHint &amp;&amp; mType !&#61; TYPE_UNREGISTERED) { //动态注册的串行广播
                if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                        &#34;Finishing broadcast to &#34; &#43; mToken);
                final IActivityManager mgr &#61; ActivityManager.getService();
                sendFinished(mgr);//见4.9.1节
            }
        }
</code></pre> 
<p>主要工作&#xff1a;</p> 
<p>1.静态注册的广播接收者:</p> 
<ul><li> <p>当QueuedWork工作未完成时&#xff0c;则等待完成再执行sendFinished</p> </li><li> <p>当QueuedWork工作已经完成&#xff0c;直接调用sendFinished方法</p> </li></ul> 
<p>2.动态注册的广播接收者&#xff1a;当发送的是串行广播&#xff0c;则直接调用sendFinished</p> 
<p>参数说明&#xff1a;</p> 
<p>TYPE_COMPONENT&#xff1a;静态注册</p> 
<p>TYPE_REGISTERED&#xff1a;动态注册</p> 
<p>TYPE_UNREGISTERED&#xff1a;取消注册</p> 
<h4><a id="491__PendingResultsendFinished_2711"></a>4.9.1 PendingResult.sendFinished</h4> 
<p>[-&gt;BroadcastReceiver.java::PendingResult]</p> 
<pre><code>        /** &#64;hide */
        public void sendFinished(IActivityManager am) {
            synchronized (this) {
                if (mFinished) {
                    throw new IllegalStateException(&#34;Broadcast already finished&#34;);
                }
                mFinished &#61; true;

                try {
                    if (mResultExtras !&#61; null) {
                        mResultExtras.setAllowFds(false);
                    }
                    //串行广播&#xff0c;见4.9.2
                    if (mOrderedHint) {
                        am.finishReceiver(mToken, mResultCode, mResultData, mResultExtras,
                                mAbortBroadcast, mFlags);
                    } else {
                        // 并行广播&#xff0c;当属于静态注册的广播&#xff0c;需要告知AMS
                        // This broadcast was sent to a component; it is not ordered,
                        // but we still need to tell the activity manager we are done.
                        am.finishReceiver(mToken, 0, null, null, false, mFlags);
                    }
                } catch (RemoteException ex) {
                }
            }
        }
</code></pre> 
<h4><a id="492__AMSfinishReceiver_2744"></a>4.9.2 AMS.finishReceiver</h4> 
<p>[-&gt;ActivityManagerService.java]</p> 
<p>通过binder机制最后调用到AMS的finishReceiver方法。</p> 
<pre><code>
    public void finishReceiver(IBinder who, int resultCode, String resultData,
            Bundle resultExtras, boolean resultAbort, int flags) {
        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, &#34;Finish receiver: &#34; &#43; who);

        // Refuse possible leaked file descriptors
        if (resultExtras !&#61; null &amp;&amp; resultExtras.hasFileDescriptors()) {
            throw new IllegalArgumentException(&#34;File descriptors passed in Bundle&#34;);
        }

        final long origId &#61; Binder.clearCallingIdentity();
        try {
            boolean doNext &#61; false;
            BroadcastRecord r;

            synchronized(this) {
                //属于哪个队列
                BroadcastQueue queue &#61; (flags &amp; Intent.FLAG_RECEIVER_FOREGROUND) !&#61; 0
                        ? mFgBroadcastQueue : mBgBroadcastQueue;
                r &#61; queue.getMatchingOrderedReceiver(who);
                if (r !&#61; null) {
                    //见4.9.3节
                    doNext &#61; r.queue.finishReceiverLocked(r, resultCode,
                        resultData, resultExtras, resultAbort, true);
                }
                if (doNext) {
                    //处理下一条广播
                    r.queue.processNextBroadcastLocked(/*fromMsg&#61;*/ false, /*skipOomAdj&#61;*/ true);
                }
                // updateOomAdjLocked() will be done here
                trimApplicationsLocked();
            }

        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
</code></pre> 
<h4><a id="493__BQfinishReceiverLocked_2790"></a>4.9.3 BQ.finishReceiverLocked</h4> 
<p>[-&gt;BroadcastQueue.java]</p> 
<pre><code>  public boolean finishReceiverLocked(BroadcastRecord r, int resultCode,
            String resultData, Bundle resultExtras, boolean resultAbort, boolean waitForServices) {
        final int state &#61; r.state;
        final ActivityInfo receiver &#61; r.curReceiver;
        r.state &#61; BroadcastRecord.IDLE;
        if (state &#61;&#61; BroadcastRecord.IDLE) {
            Slog.w(TAG, &#34;finishReceiver [&#34; &#43; mQueueName &#43; &#34;] called but state is IDLE&#34;);
        }
        r.receiver &#61; null;
        r.intent.setComponent(null);
        if (r.curApp !&#61; null &amp;&amp; r.curApp.curReceivers.contains(r)) {
            r.curApp.curReceivers.remove(r);
        }
        if (r.curFilter !&#61; null) {
            r.curFilter.receiverList.curBroadcast &#61; null;
        }
        r.curFilter &#61; null;
        r.curReceiver &#61; null;
        r.curApp &#61; null;
        mPendingBroadcast &#61; null;

        r.resultCode &#61; resultCode;
        r.resultData &#61; resultData;
        r.resultExtras &#61; resultExtras;
        if (resultAbort &amp;&amp; (r.intent.getFlags()&amp;Intent.FLAG_RECEIVER_NO_ABORT) &#61;&#61; 0) {
            r.resultAbort &#61; resultAbort;
        } else {
            r.resultAbort &#61; false;
        }

        if (waitForServices &amp;&amp; r.curComponent !&#61; null &amp;&amp; r.queue.mDelayBehindServices
                &amp;&amp; r.queue.mOrderedBroadcasts.size() &gt; 0
                &amp;&amp; r.queue.mOrderedBroadcasts.get(0) &#61;&#61; r) {
            ActivityInfo nextReceiver;
            if (r.nextReceiver &lt; r.receivers.size()) {
                Object obj &#61; r.receivers.get(r.nextReceiver);
                nextReceiver &#61; (obj instanceof ActivityInfo) ? (ActivityInfo)obj : null;
            } else {
                nextReceiver &#61; null;
            }
            // Don&#39;t do this if the next receive is in the same process as the current one.
            if (receiver &#61;&#61; null || nextReceiver &#61;&#61; null
                    || receiver.applicationInfo.uid !&#61; nextReceiver.applicationInfo.uid
                    || !receiver.processName.equals(nextReceiver.processName)) {
                // In this case, we are ready to process the next receiver for the current broadcast,
                // but are on a queue that would like to wait for services to finish before moving
                // on.  If there are background services currently starting, then we will go into a
                // special state where we hold off on continuing this broadcast until they are done.
                if (mService.mServices.hasBackgroundServicesLocked(r.userId)) {
                    Slog.i(TAG, &#34;Delay finish: &#34; &#43; r.curComponent.flattenToShortString());
                    r.state &#61; BroadcastRecord.WAITING_SERVICES;
                    return false;
                }
            }
        }

        r.curComponent &#61; null;

        // We will process the next receiver right now if this is finishing
        // an app receiver (which is always asynchronous) or after we have
        // come back from calling a receiver.
        return state &#61;&#61; BroadcastRecord.APP_RECEIVE
                || state &#61;&#61; BroadcastRecord.CALL_DONE_RECEIVE;
    }
</code></pre> 
<p>这个过程主要是设置BroadcastRecord各个参数为null&#xff0c;并判断是否还有广播需要处理。</p> 
<h3><a id="410__2863"></a>4.10 小结</h3> 
<p>接收广播的过程&#xff0c;主要是对并行广播队列和串行广播队列的处理&#xff0c;先处理并行广播队列后处理串行广播队列&#xff1a;</p> 
<p>1.处理广播队列&#xff0c;对广播队列进行相应的权限检查处理。无论是并行还是串行&#xff0c;最后调用的都是performReceiveLocked方法。其中串行广播还有时间限制&#xff0c;会调用broadcastTimeoutLocked方法&#xff0c;超时会强制结束广播&#xff1b;</p> 
<p>2.通过Binder机制&#xff0c;将消息传递给接收广播的进程进行处理&#xff0c;如果接收广播的进程没起来&#xff0c;还需要启动其进程&#xff1b;</p> 
<p>3.最后回调receiver.onReceive的方法&#xff0c;对于串行广播还需要通知AMS已经处理完该条广播&#xff0c;并进行下个广播的处理。</p> 
<h2><a id="_2873"></a>五、总结</h2> 
<p>发送广播过程的流程如下&#xff1a;</p> 
<p><img src="https://img-blog.csdnimg.cn/20200105193923486.jpg?x-oss-process&#61;image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Nhbzg2MTU0NDMyNQ&#61;&#61;,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" /><br /> <strong>广播机制</strong></p> 
<p>1.当发送串行广播&#xff08;order&#61; true&#xff09;时</p> 
<ul><li> <p>静态注册的广播接收者&#xff08;receivers&#xff09;&#xff0c;采用串行处理</p> </li><li> <p>动态注册的广播接收者&#xff08;registeredReceivers&#xff09;&#xff0c;采用串行处理</p> </li></ul> 
<p>2.当发送并行广播&#xff08;order&#61; false&#xff09;时</p> 
<ul><li> <p>静态注册的广播接收者&#xff08;receivers&#xff09;&#xff0c;采用串行处理</p> </li><li> <p>动态注册的广播接收者&#xff08;registeredReceivers&#xff09;&#xff0c;采用并行处理</p> </li></ul> 
<p>静态注册的receiver都是采用串行处理&#xff1b;动态注册的registeredReceivers处理方式无论是串行还是并行&#xff0c;取决于广播的发送方式&#xff08;processNextBroadcast&#xff09;&#xff1b;静态注册的广播由于其所在的进程没有创建&#xff0c;而进程的创建需要耗费系统的资源比较多&#xff0c;所以让静态注册的广播串行化&#xff0c;防止瞬间启动大量的进程。</p> 
<p>广播ANR只有在串行广播时才需要考虑&#xff0c;因为接收者是串行处理的&#xff0c;前一个receiver处理慢&#xff0c;会影响后一个receiver&#xff1b;并行广播通过一个循环一次性将所有的receiver分发完&#xff0c;不存在彼此影响的问题&#xff0c;没有广播超时。</p> 
<p>串行超时情况&#xff1a;某个广播处理时间&gt;2<em>receiver总个数</em>mTimeoutPeriod&#xff0c;其中mTimeoutPeriod&#xff0c;前后队列为10s,后台队列为60s&#xff1b;某个receiver的执行时间超过mTimeoutPeriod。</p> 
<h2><a id="_2898"></a>附录</h2> 
<p>源码路径</p> 
<pre><code>frameworks/base/services/core/java/com/android/server/am/BroadcastRecord.java
frameworks/base/core/java/android/app/ContextImpl.java
frameworks/base/core/java/android/app/LoadedApk.java
frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
frameworks/base/core/java/android/content/IntentFilter.java
frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java
frameworks/base/core/java/android/content/BroadcastReceiver.java
</code></pre>