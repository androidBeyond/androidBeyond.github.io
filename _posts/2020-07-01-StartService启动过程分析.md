 ---
layout:     post
title:      Android10 StartService启动过程分析
subtitle:   这篇文章分析一下StartService 的整体启动流程
date:       2020-07-01
author:     duguma
header-img: img/article-bg.jpg
top: true
catalog: true
tags:
    - Android10
    - android
	- 组件学习
 ---

<h2><a id="_3"></a>一、概述</h2> 
<p>前面已经介绍了详细介绍了管理Android四大剑客Activity、Service、Broadcast、ContentProvider的ActivityManagerService启动的详细流程&#xff0c;这里讲从应用startService的启动过程来分析AMS。</p> 
<p>ActivityManagerService相关的类图如下&#xff1a;</p> 
<p><img src="https://img-blog.csdnimg.cn/20200105192643489.jpg?x-oss-process&#61;image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Nhbzg2MTU0NDMyNQ&#61;&#61;,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" /></p> 
<p>启动服务通过startServie或者bindService即可&#xff0c;该过程如下&#xff1a;</p> 
<p>当应用调用Andorid API方法startServie或者bindService来启动服务的过程&#xff0c;主要是AMS来完成的。</p> 
<p>1.AMS通过socket通信方式向zygote进程请求创建用于承载服务进程的ActivityThread。如果启动服务运行在本地服务则不需要再次创建进程。</p> 
<p>2.zygote通过fork的方法&#xff0c;将zygote进程复制升级新的进程&#xff0c;并将ActivityThread相关的资源加载到新进程。</p> 
<p>3.AMS向新生成的ActivityThread进程&#xff0c;通过Binder方式发送创建服务的请求</p> 
<p>4.ActivityThread启动本地运行服务。</p> 
<p>启动服务的流程如下&#xff1a;</p> 
<p><img src="https://img-blog.csdnimg.cn/20200105192657238.jpg?x-oss-process&#61;image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Nhbzg2MTU0NDMyNQ&#61;&#61;,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" /></p> 
<h2><a id="_26"></a>二、启动服务进程端</h2> 
<p>在app中调用startService&#xff0c;调用的是ContextWrapper中的startService</p> 
<h3><a id="1_CWstartService_30"></a>1. CW.startService</h3> 
<p>[-&gt;ContextWrapper.java]</p> 
<pre><code> &#64;Override
 public ComponentName startService(Intent service) {
      //mBase为ContextImpl对象
     return mBase.startService(service);
 }
</code></pre> 
<h3><a id="2_ClstartService_42"></a>2. Cl.startService</h3> 
<p>[-&gt;ContextImpl.java]</p> 
<pre><code> &#64;Override
 public ComponentName startService(Intent service) {
       //system进程调用此方法时输出warn信息
        warnIfCallingFromSystemProcess();
        return startServiceCommon(service, false, mUser);
  }
</code></pre> 
<h3><a id="3__CIstartServiceCommon_55"></a>3. CI.startServiceCommon</h3> 
<p>[-&gt;ContextImpl.java]</p> 
<pre><code> private ComponentName startServiceCommon(Intent service, boolean requireForeground,
            UserHandle user) {
        try {
            //校验service&#xff0c;sdk大于等于21时&#xff0c;service中必须带Component和Package
            validateServiceIntent(service);
            service.prepareToLeaveProcess(this);
            //通过binder调用startService,见1.4节
            ComponentName cn &#61; ActivityManager.getService().startService(
                mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                            getContentResolver()), requireForeground,
                            getOpPackageName(), user.getIdentifier());
            if (cn !&#61; null) {
                if (cn.getPackageName().equals(&#34;!&#34;)) {
                    throw new SecurityException(
                            &#34;Not allowed to start service &#34; &#43; service
                            &#43; &#34; without permission &#34; &#43; cn.getClassName());
                } else if (cn.getPackageName().equals(&#34;!!&#34;)) {
                    throw new SecurityException(
                            &#34;Unable to start service &#34; &#43; service
                            &#43; &#34;: &#34; &#43; cn.getClassName());
                } else if (cn.getPackageName().equals(&#34;?&#34;)) {
                    throw new IllegalStateException(
                            &#34;Not allowed to start service &#34; &#43; service &#43; &#34;: &#34; &#43; cn.getClassName());
                }
            }
            return cn;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
</code></pre> 
<h3><a id="4_AMgetService_92"></a>4. AM.getService</h3> 
<p>这个方法和Android6.0不一样&#xff0c;没有了ActivityManagerNative和ActivityManagerProxy&#xff0c;直接通过IActivityManager.aidl生成的接口获得ActivityManagerService的代理。startService通过Binder机制&#xff0c;调用了服务器端AMS的startService方法。</p> 
<pre><code>    /**
     * &#64;hide
     */
    &#64;UnsupportedAppUsage
    public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

    &#64;UnsupportedAppUsage
    private static final Singleton&lt;IActivityManager&gt; IActivityManagerSingleton &#61;
            new Singleton&lt;IActivityManager&gt;() {
                &#64;Override
                protected IActivityManager create() {
                    final IBinder b &#61; ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    //IAcitivityManager.aidl编译会生成相应的代理类和实现类
                    final IActivityManager am &#61; IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };
</code></pre> 
<h2><a id="SystemServer_118"></a>三、SystemServer端</h2> 
<h3><a id="5__AMSstartService_120"></a>5. AMS.startService</h3> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code>  &#64;Override
    public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType, boolean requireForeground, String callingPackage, int userId)
            throws TransactionTooLargeException {
        //当调用进程是孤立进程时抛出异常&#xff0c;孤立进程uid为99000~99999
        enforceNotIsolatedCaller(&#34;startService&#34;);
        // Refuse possible leaked file descriptors
        if (service !&#61; null &amp;&amp; service.hasFileDescriptors() &#61;&#61; true) {
            throw new IllegalArgumentException(&#34;File descriptors passed in Intent&#34;);
        }

        if (callingPackage &#61;&#61; null) {
            throw new IllegalArgumentException(&#34;callingPackage cannot be null&#34;);
        }

        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE,
                &#34;*** startService: &#34; &#43; service &#43; &#34; type&#61;&#34; &#43; resolvedType &#43; &#34; fg&#61;&#34; &#43; requireForeground);
        synchronized(this) {
            //调用者pid
            final int callingPid &#61; Binder.getCallingPid();
            //调用者uid
            final int callingUid &#61; Binder.getCallingUid();
            final long origId &#61; Binder.clearCallingIdentity();
            ComponentName res;
            try {
                //mServices为ActiveServices对象
                res &#61; mServices.startServiceLocked(caller, service,
                        resolvedType, callingPid, callingUid,
                        requireForeground, callingPackage, userId);
            } finally {
                Binder.restoreCallingIdentity(origId);
            }
            return res;
        }
    }
</code></pre> 
<p>startServiceLocked方法参数说明&#xff1a;</p> 
<p>caller&#xff1a;IApplicationThread类型</p> 
<p>service&#xff1a;Intent类型&#xff0c;包含运行的Service信息</p> 
<p>resolvedType&#xff1a;String类型</p> 
<p>callingPid&#xff1a;调用者pid</p> 
<p>callingUid&#xff1a;调用者uid</p> 
<p>requireForeground&#xff1a;是否需要前台运行&#xff0c;前面传的是false</p> 
<p>callingPackage&#xff1a;调用该方法的包名</p> 
<p>userId&#xff1a;用户id</p> 
<h3><a id="6__ASstartServiceLocked_180"></a>6. AS.startServiceLocked</h3> 
<p>[-&gt;ActiveServices.java]</p> 
<pre><code>  ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
            int callingPid, int callingUid, boolean fgRequired, String callingPackage, final int userId)
            throws TransactionTooLargeException {
        if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, &#34;startService: &#34; &#43; service
                &#43; &#34; type&#61;&#34; &#43; resolvedType &#43; &#34; args&#61;&#34; &#43; service.getExtras());

        final boolean callerFg;
        if (caller !&#61; null) {
            //进程不存在抛出异常
            final ProcessRecord callerApp &#61; mAm.getRecordForAppLocked(caller);
            if (callerApp &#61;&#61; null) {
                throw new SecurityException(
                        &#34;Unable to find app for caller &#34; &#43; caller
                        &#43; &#34; (pid&#61;&#34; &#43; callingPid
                        &#43; &#34;) when starting service &#34; &#43; service);
            }
            callerFg &#61; callerApp.setSchedGroup !&#61; ProcessList.SCHED_GROUP_BACKGROUND;
        } else {
            callerFg &#61; true;
        }
        //检查服务信息
        ServiceLookupResult res &#61;
            retrieveServiceLocked(service, resolvedType, callingPackage,
                    callingPid, callingUid, userId, true, callerFg, false, false);
        if (res &#61;&#61; null) {
            return null;
        }
        if (res.record &#61;&#61; null) {
            return new ComponentName(&#34;!&#34;, res.permission !&#61; null
                    ? res.permission : &#34;private to package&#34;);
        }

        ServiceRecord r &#61; res.record;
        //检查是否存在启动服务的user
        if (!mAm.mUserController.exists(r.userId)) {
            Slog.w(TAG, &#34;Trying to start service with non-existent user! &#34; &#43; r.userId);
            return null;
        }

        //是否运行后台启动服务
        // If we&#39;re starting indirectly (e.g. from PendingIntent), figure out whether
        // we&#39;re launching into an app in a background state.  This keys off of the same
        // idleness state tracking as e.g. O&#43; background service start policy.
        final boolean bgLaunch &#61; !mAm.isUidActiveLocked(r.appInfo.uid);

        // If the app has strict background restrictions, we treat any bg service
        // start analogously to the legacy-app forced-restrictions case, regardless
        // of its target SDK version.
        boolean forcedStandby &#61; false;
        if (bgLaunch &amp;&amp; appRestrictedAnyInBackground(r.appInfo.uid, r.packageName)) {
            if (DEBUG_FOREGROUND_SERVICE) {
                Slog.d(TAG, &#34;Forcing bg-only service start only for &#34; &#43; r.shortName
                        &#43; &#34; : bgLaunch&#61;&#34; &#43; bgLaunch &#43; &#34; callerFg&#61;&#34; &#43; callerFg);
            }
            forcedStandby &#61; true;
        }
        //如果是要求前台启动&#xff0c;则允许每个应用操作
        // If this is a direct-to-foreground start, make sure it is allowed as per the app op.
        boolean forceSilentAbort &#61; false;
        if (fgRequired) {
            final int mode &#61; mAm.mAppOpsService.checkOperation(
                    AppOpsManager.OP_START_FOREGROUND, r.appInfo.uid, r.packageName);
            switch (mode) {
                case AppOpsManager.MODE_ALLOWED:
                case AppOpsManager.MODE_DEFAULT:
                    // All okay.
                    break;
                case AppOpsManager.MODE_IGNORED:
                    // Not allowed, fall back to normal start service, failing siliently
                    // if background check restricts that.
                    Slog.w(TAG, &#34;startForegroundService not allowed due to app op: service &#34;
                            &#43; service &#43; &#34; to &#34; &#43; r.name.flattenToShortString()
                            &#43; &#34; from pid&#61;&#34; &#43; callingPid &#43; &#34; uid&#61;&#34; &#43; callingUid
                            &#43; &#34; pkg&#61;&#34; &#43; callingPackage);
                    fgRequired &#61; false;
                    forceSilentAbort &#61; true;
                    break;
                default:
                    return new ComponentName(&#34;!!&#34;, &#34;foreground not allowed as per app op&#34;);
            }
        }
      
        //如果不是前台启动&#xff0c;则检查服务是否可以启动
        // If this isn&#39;t a direct-to-foreground start, check our ability to kick off an
        // arbitrary service
        if (forcedStandby || (!r.startRequested &amp;&amp; !fgRequired)) {
            // Before going further -- if this app is not allowed to start services in the
            // background, then at this point we aren&#39;t going to let it period.
            final int allowed &#61; mAm.getAppStartModeLocked(r.appInfo.uid, r.packageName,
                    r.appInfo.targetSdkVersion, callingPid, false, false, forcedStandby);
            if (allowed !&#61; ActivityManager.APP_START_MODE_NORMAL) {
                Slog.w(TAG, &#34;Background start not allowed: service &#34;
                        &#43; service &#43; &#34; to &#34; &#43; r.name.flattenToShortString()
                        &#43; &#34; from pid&#61;&#34; &#43; callingPid &#43; &#34; uid&#61;&#34; &#43; callingUid
                        &#43; &#34; pkg&#61;&#34; &#43; callingPackage &#43; &#34; startFg?&#61;&#34; &#43; fgRequired);
                if (allowed &#61;&#61; ActivityManager.APP_START_MODE_DELAYED || forceSilentAbort) {
                    // In this case we are silently disabling the app, to disrupt as
                    // little as possible existing apps.
                    return null;
                }
                if (forcedStandby) {
                    // This is an O&#43; app, but we might be here because the user has placed
                    // it under strict background restrictions.  Don&#39;t punish the app if it&#39;s
                    // trying to do the right thing but we&#39;re denying it for that reason.
                    if (fgRequired) {
                        if (DEBUG_BACKGROUND_CHECK) {
                            Slog.v(TAG, &#34;Silently dropping foreground service launch due to FAS&#34;);
                        }
                        return null;
                    }
                }
                // This app knows it is in the new model where this operation is not
                // allowed, so tell it what has happened.
                UidRecord uidRec &#61; mAm.mActiveUids.get(r.appInfo.uid);
                return new ComponentName(&#34;?&#34;, &#34;app is in background uid &#34; &#43; uidRec);
            }
        }
       
        //前台服务判断&#xff0c;小于android10.0 则fgRequired为false
        // At this point we&#39;ve applied allowed-to-start policy based on whether this was
        // an ordinary startService() or a startForegroundService().  Now, only require that
        // the app follow through on the startForegroundService() -&gt; startForeground()
        // contract if it actually targets O&#43;.
        if (r.appInfo.targetSdkVersion &lt; Build.VERSION_CODES.O &amp;&amp; fgRequired) {
            if (DEBUG_BACKGROUND_CHECK || DEBUG_FOREGROUND_SERVICE) {
                Slog.i(TAG, &#34;startForegroundService() but host targets &#34;
                        &#43; r.appInfo.targetSdkVersion &#43; &#34; - not requiring startForeground()&#34;);
            }
            fgRequired &#61; false;
        }

        NeededUriGrants neededGrants &#61; mAm.checkGrantUriPermissionFromIntentLocked(
                callingUid, r.packageName, service, service.getFlags(), null, r.userId);
        //如果权限需要授权&#xff0c;则不启动服务
        // If permissions need a review before any of the app components can run,
        // we do not start the service and launch a review activity if the calling app
        // is in the foreground passing it a pending intent to start the service when
        // review is completed.
        if (mAm.mPermissionReviewRequired) {
            // XXX This is not dealing with fgRequired!
            if (!requestStartTargetPermissionsReviewIfNeededLocked(r, callingPackage,
                    callingUid, service, callerFg, userId)) {
                return null;
            }
        }

        if (unscheduleServiceRestartLocked(r, callingUid, false)) {
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, &#34;START SERVICE WHILE RESTART PENDING: &#34; &#43; r);
        }
        r.lastActivity &#61; SystemClock.uptimeMillis();
        r.startRequested &#61; true;
        r.delayedStop &#61; false;
        r.fgRequired &#61; fgRequired;
        r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                service, neededGrants, callingUid));

        if (fgRequired) {
            // We are now effectively running a foreground service.
            mAm.mAppOpsService.startOperation(AppOpsManager.getToken(mAm.mAppOpsService),
                    AppOpsManager.OP_START_FOREGROUND, r.appInfo.uid, r.packageName, true);
        }

        final ServiceMap smap &#61; getServiceMapLocked(r.userId);
        boolean addToStarting &#61; false;
        //非前台服务的管理
        if (!callerFg &amp;&amp; !fgRequired &amp;&amp; r.app &#61;&#61; null
                &amp;&amp; mAm.mUserController.hasStartedUserState(r.userId)) {
            ProcessRecord proc &#61; mAm.getProcessRecordLocked(r.processName, r.appInfo.uid, false);
            if (proc &#61;&#61; null || proc.curProcState &gt; ActivityManager.PROCESS_STATE_RECEIVER) {
                // If this is not coming from a foreground caller, then we may want
                // to delay the start if there are already other background services
                // that are starting.  This is to avoid process start spam when lots
                // of applications are all handling things like connectivity broadcasts.
                // We only do this for cached processes, because otherwise an application
                // can have assumptions about calling startService() for a service to run
                // in its own process, and for that process to not be killed before the
                // service is started.  This is especially the case for receivers, which
                // may start a service in onReceive() to do some additional work and have
                // initialized some global state as part of that.
                if (DEBUG_DELAYED_SERVICE) Slog.v(TAG_SERVICE, &#34;Potential start delay of &#34;
                        &#43; r &#43; &#34; in &#34; &#43; proc);
                //如果延迟启动
                if (r.delayed) {
                    // This service is already scheduled for a delayed start; just leave
                    // it still waiting.
                    if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, &#34;Continuing to delay: &#34; &#43; r);
                    return r.name;
                }
                //如果后台启动的服务数大于同一时间内启动的最大服务数&#xff0c;则加入延迟启动队列
                if (smap.mStartingBackground.size() &gt;&#61; mMaxStartingBackground) {
                    // Something else is starting, delay!
                    Slog.i(TAG_SERVICE, &#34;Delaying start of: &#34; &#43; r);
                    smap.mDelayedStartList.add(r);
                    r.delayed &#61; true;
                    return r.name;
                }
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, &#34;Not delaying: &#34; &#43; r);
                addToStarting &#61; true;
            } else if (proc.curProcState &gt;&#61; ActivityManager.PROCESS_STATE_SERVICE) {
                // We slightly loosen when we will enqueue this new service as a background
                // starting service we are waiting for, to also include processes that are
                // currently running other services or receivers.
                //将服务加入到后台启动队列
                addToStarting &#61; true;
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                        &#34;Not delaying, but counting as bg: &#34; &#43; r);
            } else if (DEBUG_DELAYED_STARTS) {
                StringBuilder sb &#61; new StringBuilder(128);
                sb.append(&#34;Not potential delay (state&#61;&#34;).append(proc.curProcState)
                        .append(&#39; &#39;).append(proc.adjType);
                String reason &#61; proc.makeAdjReason();
                if (reason !&#61; null) {
                    sb.append(&#39; &#39;);
                    sb.append(reason);
                }
                sb.append(&#34;): &#34;);
                sb.append(r.toString());
                Slog.v(TAG_SERVICE, sb.toString());
            }
        } else if (DEBUG_DELAYED_STARTS) {
            if (callerFg || fgRequired) {
                Slog.v(TAG_SERVICE, &#34;Not potential delay (callerFg&#61;&#34; &#43; callerFg &#43; &#34; uid&#61;&#34;
                        &#43; callingUid &#43; &#34; pid&#61;&#34; &#43; callingPid &#43; &#34; fgRequired&#61;&#34; &#43; fgRequired &#43; &#34;): &#34; &#43; r);
            } else if (r.app !&#61; null) {
                Slog.v(TAG_SERVICE, &#34;Not potential delay (cur app&#61;&#34; &#43; r.app &#43; &#34;): &#34; &#43; r);
            } else {
                Slog.v(TAG_SERVICE,
                        &#34;Not potential delay (user &#34; &#43; r.userId &#43; &#34; not started): &#34; &#43; r);
            }
        }

        ComponentName cmp &#61; startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
        return cmp;
    }
</code></pre> 
<p>可以看到&#xff0c;Android10.0对于后台服务的启动&#xff0c;要求更加的严格。如果allowed不等于APP_START_MODE_NORMAL&#xff0c;则后台服务将不允许被启动。</p> 
<p>callerFg对用于标记前台还是后台&#xff0c;当发起方进程不等于SCHED_GROUP_BACKGROUND或者发起方为空&#xff0c;则callerFg&#61; true&#xff0c;否则为false。</p> 
<h4><a id="61_AMSgetAppStartModeLocked_425"></a>6.1 AMS.getAppStartModeLocked</h4> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code>  int getAppStartModeLocked(int uid, String packageName, int packageTargetSdk,
            int callingPid, boolean alwaysRestrict, boolean disabledOnly, boolean forcedStandby) {
        UidRecord uidRec &#61; mActiveUids.get(uid);
        if (DEBUG_BACKGROUND_CHECK) Slog.d(TAG, &#34;checkAllowBackground: uid&#61;&#34; &#43; uid &#43; &#34; pkg&#61;&#34;
                &#43; packageName &#43; &#34; rec&#61;&#34; &#43; uidRec &#43; &#34; always&#61;&#34; &#43; alwaysRestrict &#43; &#34; idle&#61;&#34;
                &#43; (uidRec !&#61; null ? uidRec.idle : false));
        //不进入这里则会被允许启动后台服务&#xff0c;否则将会进入下一步的检查
        if (uidRec &#61;&#61; null || alwaysRestrict || forcedStandby || uidRec.idle) {
            boolean ephemeral;
            if (uidRec &#61;&#61; null) {
                ephemeral &#61; getPackageManagerInternalLocked().isPackageEphemeral(
                        UserHandle.getUserId(uid), packageName);
            } else {
                ephemeral &#61; uidRec.ephemeral;
            }

            if (ephemeral) {
                // We are hard-core about ephemeral apps not running in the background.
                return ActivityManager.APP_START_MODE_DISABLED;
            } else {
                if (disabledOnly) {
                    // The caller is only interested in whether app starts are completely
                    // disabled for the given package (that is, it is an instant app).  So
                    // we don&#39;t need to go further, which is all just seeing if we should
                    // apply a &#34;delayed&#34; mode for a regular app.
                    return ActivityManager.APP_START_MODE_NORMAL;
                }
                //alwaysRestrict为上面传过来的callerFg
                final int startMode &#61; (alwaysRestrict)
                        ? appRestrictedInBackgroundLocked(uid, packageName, packageTargetSdk)
                        : appServicesRestrictedInBackgroundLocked(uid, packageName,
                                packageTargetSdk);
                if (DEBUG_BACKGROUND_CHECK) {
                    Slog.d(TAG, &#34;checkAllowBackground: uid&#61;&#34; &#43; uid
                            &#43; &#34; pkg&#61;&#34; &#43; packageName &#43; &#34; startMode&#61;&#34; &#43; startMode
                            &#43; &#34; onwhitelist&#61;&#34; &#43; isOnDeviceIdleWhitelistLocked(uid, false)
                            &#43; &#34; onwhitelist(ei)&#61;&#34; &#43; isOnDeviceIdleWhitelistLocked(uid, true));
                }
                //延时模式下如果发起方的进行存在则还是可以启动
                if (startMode &#61;&#61; ActivityManager.APP_START_MODE_DELAYED) {
                    // This is an old app that has been forced into a &#34;compatible as possible&#34;
                    // mode of background check.  To increase compatibility, we will allow other
                    // foreground apps to cause its services to start.
                    if (callingPid &gt;&#61; 0) {
                        ProcessRecord proc;
                        synchronized (mPidsSelfLocked) {
                            proc &#61; mPidsSelfLocked.get(callingPid);
                        }
                        if (proc !&#61; null &amp;&amp;
                                !ActivityManager.isProcStateBackground(proc.curProcState)) {
                            // Whoever is instigating this is in the foreground, so we will allow it
                            // to go through.
                            return ActivityManager.APP_START_MODE_NORMAL;
                        }
                    }
                }
                return startMode;
            }
        }
        return ActivityManager.APP_START_MODE_NORMAL;
    }

</code></pre> 
<p>getAppStartModeLocked方法获取是否允许是否后台启动&#xff0c;一般callerFg为false&#xff0c;从而会进行服务再一次判断&#xff0c;普通的服务会进入appServicesRestrictedInBackgroundLocked方法进行判断如下&#xff1a;</p> 
<h4><a id="62_AMSappServicesRestrictedInBackgroundLocked_496"></a>6.2 AMS.appServicesRestrictedInBackgroundLocked</h4> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code>    // Service launch is available to apps with run-in-background exemptions but
    // some other background operations are not.  If we&#39;re doing a check
    // of service-launch policy, allow those callers to proceed unrestricted.
    int appServicesRestrictedInBackgroundLocked(int uid, String packageName, int packageTargetSdk)    {
       // Persistent进程
       // Persistent app?
        if (mPackageManagerInt.isPackagePersistent(packageName)) {
            if (DEBUG_BACKGROUND_CHECK) {
                Slog.i(TAG, &#34;App &#34; &#43; uid &#43; &#34;/&#34; &#43; packageName
                        &#43; &#34; is persistent; not restricted in background&#34;);
            }
            return ActivityManager.APP_START_MODE_NORMAL;
        }
        //uid白名单
        // Non-persistent but background whitelisted?
        if (uidOnBackgroundWhitelist(uid)) {
            if (DEBUG_BACKGROUND_CHECK) {
                Slog.i(TAG, &#34;App &#34; &#43; uid &#43; &#34;/&#34; &#43; packageName
                        &#43; &#34; on background whitelist; not restricted in background&#34;);
            }
            return ActivityManager.APP_START_MODE_NORMAL;
        }
        //电池白名单
        // Is this app on the battery whitelist?
        if (isOnDeviceIdleWhitelistLocked(uid, /*allowExceptIdleToo&#61;*/ false)) {
            if (DEBUG_BACKGROUND_CHECK) {
                Slog.i(TAG, &#34;App &#34; &#43; uid &#43; &#34;/&#34; &#43; packageName
                        &#43; &#34; on idle whitelist; not restricted in background&#34;);
            }
            return ActivityManager.APP_START_MODE_NORMAL;
        }

        // None of the service-policy criteria apply, so we apply the common criteria
        return appRestrictedInBackgroundLocked(uid, packageName, packageTargetSdk);
    }

</code></pre> 
<p>可以看到对于persistent app、uid后台服务白名单、电池白名单里面都是可以启动后台服务的。</p> 
<h3><a id="7_ASstartServiceInnerLocked_541"></a>7. AS.startServiceInnerLocked</h3> 
<p>[-&gt;ActiveServices.java]</p> 
<pre><code> ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
            boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
        ServiceState stracker &#61; r.getTracker();
        if (stracker !&#61; null) {
            stracker.setStarted(true, mAm.mProcessStats.getMemFactorLocked(), r.lastActivity);
        }
        r.callStart &#61; false;
        synchronized (r.stats.getBatteryStats()) {
            //用于耗电统计&#xff0c;开启允许状态
            r.stats.startRunningLocked();
        }
        String error &#61; bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);
        if (error !&#61; null) {
            return new ComponentName(&#34;!!&#34;, error);
        }

        if (r.startRequested &amp;&amp; addToStarting) {
            boolean first &#61; smap.mStartingBackground.size() &#61;&#61; 0;
            smap.mStartingBackground.add(r);
            r.startingBgTimeout &#61; SystemClock.uptimeMillis() &#43; mAm.mConstants.BG_START_TIMEOUT;
            if (DEBUG_DELAYED_SERVICE) {
                RuntimeException here &#61; new RuntimeException(&#34;here&#34;);
                here.fillInStackTrace();
                Slog.v(TAG_SERVICE, &#34;Starting background (first&#61;&#34; &#43; first &#43; &#34;): &#34; &#43; r, here);
            } else if (DEBUG_DELAYED_STARTS) {
                Slog.v(TAG_SERVICE, &#34;Starting background (first&#61;&#34; &#43; first &#43; &#34;): &#34; &#43; r);
            }
            if (first) {
                smap.rescheduleDelayedStartsLocked();
            }
        } else if (callerFg || r.fgRequired) {
            smap.ensureNotStartingBackgroundLocked(r);
        }

        return r.name;
    }
</code></pre> 
<h3><a id="8ASbringUpServiceLocked_584"></a>8.AS.bringUpServiceLocked</h3> 
<p>[-&gt;ActiveServices.java]</p> 
<pre><code> private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
            boolean whileRestarting, boolean permissionsReviewRequired)
            throws TransactionTooLargeException {
        //Slog.i(TAG, &#34;Bring up service:&#34;);
        //r.dump(&#34;  &#34;);

        if (r.app !&#61; null &amp;&amp; r.app.thread !&#61; null) {
            //调用service.onStartCommand过程
            sendServiceArgsLocked(r, execInFg, false);
            return null;
        }

        if (!whileRestarting &amp;&amp; mRestartingServices.contains(r)) {
            //等待延迟重启的过程则直接返回
            // If waiting for a restart, then do nothing.
            return null;
        }

        if (DEBUG_SERVICE) {
            Slog.v(TAG_SERVICE, &#34;Bringing up &#34; &#43; r &#43; &#34; &#34; &#43; r.intent &#43; &#34; fg&#61;&#34; &#43; r.fgRequired);
        }
        //启动service前&#xff0c;把service从启动服务队列中移除
        // We are now bringing the service up, so no longer in the
        // restarting state.
        if (mRestartingServices.remove(r)) {
            clearRestartingIfNeededLocked(r);
        }
        //service正在启动&#xff0c;将delayed设置为false
        // Make sure this service is no longer considered delayed, we are starting it now.
        if (r.delayed) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, &#34;REM FR DELAY LIST (bring up): &#34; &#43; r);
            getServiceMapLocked(r.userId).mDelayedStartList.remove(r);
            r.delayed &#61; false;
        }
        //确保拥有服务的user已经启动&#xff0c;否则停止服务
        // Make sure that the user who owns this service is started.  If not,
        // we don&#39;t want to allow it to run.
        if (!mAm.mUserController.hasStartedUserState(r.userId)) {
            String msg &#61; &#34;Unable to launch app &#34;
                    &#43; r.appInfo.packageName &#43; &#34;/&#34;
                    &#43; r.appInfo.uid &#43; &#34; for service &#34;
                    &#43; r.intent.getIntent() &#43; &#34;: user &#34; &#43; r.userId &#43; &#34; is stopped&#34;;
            Slog.w(TAG, msg);
            bringDownServiceLocked(r);
            return msg;
        }
        //服务正在启动&#xff0c;设置package停止状态为false
        // Service is now being launched, its package can&#39;t be stopped.
        try {
            AppGlobals.getPackageManager().setPackageStoppedState(
                    r.packageName, false, r.userId);
        } catch (RemoteException e) {
        } catch (IllegalArgumentException e) {
            Slog.w(TAG, &#34;Failed trying to unstop package &#34;
                    &#43; r.packageName &#43; &#34;: &#34; &#43; e);
        }
        
        final boolean isolated &#61; (r.serviceInfo.flags&amp;ServiceInfo.FLAG_ISOLATED_PROCESS) !&#61; 0;
        final String procName &#61; r.processName;
        String hostingType &#61; &#34;service&#34;;
        ProcessRecord app;
        //如果不是孤立进程
        if (!isolated) {
            //根据uid和pid查询ProcessRecord
            app &#61; mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
            if (DEBUG_MU) Slog.v(TAG_MU, &#34;bringUpServiceLocked: appInfo.uid&#61;&#34; &#43; r.appInfo.uid
                        &#43; &#34; app&#61;&#34; &#43; app);
            if (app !&#61; null &amp;&amp; app.thread !&#61; null) {
                try {
                    app.addPackage(r.appInfo.packageName, r.appInfo.longVersionCode, mAm.mProcessStats);
                    //启动服务
                    realStartServiceLocked(r, app, execInFg);
                    return null;
                } catch (TransactionTooLargeException e) {
                    throw e;
                } catch (RemoteException e) {
                    Slog.w(TAG, &#34;Exception when starting service &#34; &#43; r.shortName, e);
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
            app &#61; r.isolatedProc;
            if (WebViewZygote.isMultiprocessEnabled()
                    &amp;&amp; r.serviceInfo.packageName.equals(WebViewZygote.getPackageName())) {
                hostingType &#61; &#34;webview_service&#34;;
            }
        }
        //对于服务进程没有启动的情况
        // Not running -- get it started, and enqueue this service record
        // to be executed when the app comes up.
        if (app &#61;&#61; null &amp;&amp; !permissionsReviewRequired) {
            //启动服务所需要的进程
            if ((app&#61;mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    hostingType, r.name, false, isolated, false)) &#61;&#61; null) {
                String msg &#61; &#34;Unable to launch app &#34;
                        &#43; r.appInfo.packageName &#43; &#34;/&#34;
                        &#43; r.appInfo.uid &#43; &#34; for service &#34;
                        &#43; r.intent.getIntent() &#43; &#34;: process is bad&#34;;
                Slog.w(TAG, msg);
                //进程启动失败
                bringDownServiceLocked(r);
                return msg;
            }
            if (isolated) {
                r.isolatedProc &#61; app;
            }
        }

        if (r.fgRequired) {
            if (DEBUG_FOREGROUND_SERVICE) {
                Slog.v(TAG, &#34;Whitelisting &#34; &#43; UserHandle.formatUid(r.appInfo.uid)
                        &#43; &#34; for fg-service launch&#34;);
            }
            mAm.tempWhitelistUidLocked(r.appInfo.uid,
                    SERVICE_START_FOREGROUND_TIMEOUT, &#34;fg-service-launch&#34;);
        }

        if (!mPendingServices.contains(r)) {
            mPendingServices.add(r);
        }

        if (r.delayedStop) {
            // Oh and hey we&#39;ve already been asked to stop!
            r.delayedStop &#61; false;
            if (r.startRequested) {
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                        &#34;Applying delayed stop (in bring up): &#34; &#43; r);
                //停止服务
                stopServiceLocked(r);
            }
        }

        return null;
    }
</code></pre> 
<p>当目标进程已经存在&#xff0c;则直接执行realStartServiceLocked</p> 
<p>当目标进程不存在&#xff0c;则先执行startProcessLocked创建进程&#xff0c;经过层层调用最后会调用到AMS.attachApplicationLocked,然后再执行realStartServiceLocked。</p> 
<p>对于非前台进程调用而需要启动的服务&#xff0c;如果已经有其他的后台服务正在启动&#xff0c;则可能希望延迟其启动&#xff0c;从而避免同时启动过多的进程&#xff08;非必须&#xff09;。</p> 
<h4><a id="81_AMSattachApplicationLocked_739"></a>8.1 AMS.attachApplicationLocked</h4> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code> &#64;GuardedBy(&#34;this&#34;)
    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid, int callingUid, long startSeq) {
        ...
        thread.bindApplication(processName, appInfo, providers,
                        app.instr.mClass,
                        profilerInfo, app.instr.mArguments,
                        app.instr.mWatcher,
                        app.instr.mUiAutomationConnection, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.persistent,
                        new Configuration(getGlobalConfiguration()), app.compat,
                        getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, isAutofillCompatEnabled);
        
        //寻找所有需要在该进程中运行的服务
        // Find any services that should be running in this process...
        if (!badApp) {
            try {
                didSomething |&#61; mServices.attachApplicationLocked(app, processName);
                checkTime(startTime, &#34;attachApplicationLocked: after mServices.attachApplicationLocked&#34;);
            } catch (Exception e) {
                Slog.wtf(TAG, &#34;Exception thrown starting services in &#34; &#43; app, e);
                badApp &#61; true;
            }
        }
        ...
        return true;
    }
</code></pre> 
<h4><a id="82_ASattachApplicationLocked_776"></a>8.2 AS.attachApplicationLocked</h4> 
<p>[-&gt;ActiveServices.java]</p> 
<pre><code>  boolean attachApplicationLocked(ProcessRecord proc, String processName)
            throws RemoteException {
        boolean didSomething &#61; false;
        //启动mPendingServices队列中&#xff0c;等待在该进程启动的服务
        // Collect any services that are waiting for this process to come up.
        if (mPendingServices.size() &gt; 0) {
            ServiceRecord sr &#61; null;
            try {
                for (int i&#61;0; i&lt;mPendingServices.size(); i&#43;&#43;) {
                    sr &#61; mPendingServices.get(i);
                    if (proc !&#61; sr.isolatedProc &amp;&amp; (proc.uid !&#61; sr.appInfo.uid
                            || !processName.equals(sr.processName))) {
                        continue;
                    }

                    mPendingServices.remove(i);
                    i--;
                    //将当前服务的包信息加入到proc
                    proc.addPackage(sr.appInfo.packageName, sr.appInfo.longVersionCode,
                            mAm.mProcessStats);
                    //启动服务        
                    realStartServiceLocked(sr, proc, sr.createdFromFg);
                    didSomething &#61; true;
                    if (!isServiceNeededLocked(sr, false, false)) {
                        // We were waiting for this service to start, but it is actually no
                        // longer needed.  This could happen because bringDownServiceIfNeeded
                        // won&#39;t bring down a service that is pending...  so now the pending
                        // is done, so let&#39;s drop it.
                        bringDownServiceLocked(sr);
                    }
                }
            } catch (RemoteException e) {
                Slog.w(TAG, &#34;Exception in new application when starting service &#34;
                        &#43; sr.shortName, e);
                throw e;
            }
        }
        //对于正在等待重启并需要运行在该进程中的服务&#xff0c;现在是启动的好时机
        // Also, if there are any services that are waiting to restart and
        // would run in this process, now is a good time to start them.  It would
        // be weird to bring up the process but arbitrarily not let the services
        // run at this point just because their restart time hasn&#39;t come up.
        if (mRestartingServices.size() &gt; 0) {
            ServiceRecord sr;
            for (int i&#61;0; i&lt;mRestartingServices.size(); i&#43;&#43;) {
                sr &#61; mRestartingServices.get(i);
                if (proc !&#61; sr.isolatedProc &amp;&amp; (proc.uid !&#61; sr.appInfo.uid
                        || !processName.equals(sr.processName))) {
                    continue;
                }
                mAm.mHandler.removeCallbacks(sr.restarter);
                mAm.mHandler.post(sr.restarter);
            }
        }
        return didSomething;
    }
</code></pre> 
<h3><a id="9ASrealStartServiceLocked_839"></a>9.AS.realStartServiceLocked</h3> 
<p>[-&gt;ActiveServices.java]</p> 
<pre><code>private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        if (app.thread &#61;&#61; null) {
            throw new RemoteException();
        }
        if (DEBUG_MU)
            Slog.v(TAG_MU, &#34;realStartServiceLocked, ServiceRecord.uid &#61; &#34; &#43; r.appInfo.uid
                    &#43; &#34;, ProcessRecord.uid &#61; &#34; &#43; app.uid);
        r.app &#61; app;
        r.restartTime &#61; r.lastActivity &#61; SystemClock.uptimeMillis();

        final boolean newService &#61; app.services.add(r);
        //发送delay消息
        bumpServiceExecutingLocked(r, execInFg, &#34;create&#34;);
        mAm.updateLruProcessLocked(app, false, null);
        updateServiceForegroundLocked(r.app, /* oomAdj&#61; */ false);
        mAm.updateOomAdjLocked();

        boolean created &#61; false;
        try {
            if (LOG_SERVICE_START_STOP) {
                String nameTerm;
                int lastPeriod &#61; r.shortName.lastIndexOf(&#39;.&#39;);
                nameTerm &#61; lastPeriod &gt;&#61; 0 ? r.shortName.substring(lastPeriod) : r.shortName;
                EventLogTags.writeAmCreateService(
                        r.userId, System.identityHashCode(r), nameTerm, r.app.uid, r.app.pid);
            }
            synchronized (r.stats.getBatteryStats()) {
                r.stats.startLaunchedLocked();
            }
            mAm.notifyPackageUse(r.serviceInfo.packageName,
                                 PackageManager.NOTIFY_PACKAGE_USE_SERVICE);
            app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
            //服务进入onCreate
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            r.postNotification();
            created &#61; true;
        } catch (DeadObjectException e) {
            //应用死亡处理
            Slog.w(TAG, &#34;Application dead when creating service &#34; &#43; r);
            mAm.appDiedLocked(app);
            throw e;
        } finally {
            if (!created) {
                // Keep the executeNesting count accurate.
                final boolean inDestroying &#61; mDestroyingServices.contains(r);
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);
                // Cleanup.
                if (newService) {
                    app.services.remove(r);
                    r.app &#61; null;
                }

                // Retry.
                if (!inDestroying) {
                    scheduleServiceRestartLocked(r, false);
                }
            }
        }

        if (r.whitelistManager) {
            app.whitelistManager &#61; true;
        }
        requestServiceBindingsLocked(r, execInFg);
        updateServiceClientActivitiesLocked(app, null, true);
        // If the service is in the started state, and there are no
        // pending arguments, then fake up one so its onStartCommand() will
        // be called.
        if (r.startRequested &amp;&amp; r.callStart &amp;&amp; r.pendingStarts.size() &#61;&#61; 0) {
            r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                    null, null, 0));
        }
        //服务进入onStartCommand
        sendServiceArgsLocked(r, execInFg, true);

        if (r.delayed) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, &#34;REM FR DELAY LIST (new proc): &#34; &#43; r);
            getServiceMapLocked(r.userId).mDelayedStartList.remove(r);
            r.delayed &#61; false;
        }

        if (r.delayedStop) {
            // Oh and hey we&#39;ve already been asked to stop!
            r.delayedStop &#61; false;
            if (r.startRequested) {
                //停止服务
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                        &#34;Applying delayed stop (from start): &#34; &#43; r);
                stopServiceLocked(r);
            }
        }
    }
</code></pre> 
<p>bumpServiceExecutingLocked会发送一个延迟处理的消息SERVICE_TIMEOUT_MSG。在方法scheduleCreateService执行完成&#xff0c;如果onCreate回调执行完成后&#xff0c;便会remove掉该消息。但如果没有在延时的时间内移除掉消息&#xff0c;则会进入到service timeout流程</p> 
<h4><a id="91_ASbumpServiceExecutingLocked_942"></a>9.1 AS.bumpServiceExecutingLocked</h4> 
<p>[-&gt;ActiveServices.java]</p> 
<pre><code> private final void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why) {
        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, &#34;&gt;&gt;&gt; EXECUTING &#34;
                &#43; why &#43; &#34; of &#34; &#43; r &#43; &#34; in app &#34; &#43; r.app);
        else if (DEBUG_SERVICE_EXECUTING) Slog.v(TAG_SERVICE_EXECUTING, &#34;&gt;&gt;&gt; EXECUTING &#34;
                &#43; why &#43; &#34; of &#34; &#43; r.shortName);

        // For b/34123235: Services within the system server won&#39;t start until SystemServer
        // does Looper.loop(), so we shouldn&#39;t try to start/bind to them too early in the boot
        // process. However, since there&#39;s a little point of showing the ANR dialog in that case,
        // let&#39;s suppress the timeout until PHASE_THIRD_PARTY_APPS_CAN_START.
        //
        // (Note there are multiple services start at PHASE_THIRD_PARTY_APPS_CAN_START too,
        // which technically could also trigger this timeout if there&#39;s a system server
        // that takes a long time to handle PHASE_THIRD_PARTY_APPS_CAN_START, but that shouldn&#39;t
        // happen.)
        boolean timeoutNeeded &#61; true;
        if ((mAm.mBootPhase &lt; SystemService.PHASE_THIRD_PARTY_APPS_CAN_START)
                &amp;&amp; (r.app !&#61; null) &amp;&amp; (r.app.pid &#61;&#61; android.os.Process.myPid())) {
            //SystemServer还未进入到PHASE_THIRD_PARTY_APPS_CAN_START状态&#xff0c;则不能启动服务
            Slog.w(TAG, &#34;Too early to start/bind service in system_server: Phase&#61;&#34; &#43; mAm.mBootPhase
                    &#43; &#34; &#34; &#43; r.getComponentName());
            timeoutNeeded &#61; false;
        }

        long now &#61; SystemClock.uptimeMillis();
        if (r.executeNesting &#61;&#61; 0) {
            r.executeFg &#61; fg;
            ServiceState stracker &#61; r.getTracker();
            if (stracker !&#61; null) {
                stracker.setExecuting(true, mAm.mProcessStats.getMemFactorLocked(), now);
            }
            if (r.app !&#61; null) {
                r.app.executingServices.add(r);
                r.app.execServicesFg |&#61; fg;
                if (timeoutNeeded &amp;&amp; r.app.executingServices.size() &#61;&#61; 1) {
                    scheduleServiceTimeoutLocked(r.app);
                }
            }
        } else if (r.app !&#61; null &amp;&amp; fg &amp;&amp; !r.app.execServicesFg) {
            r.app.execServicesFg &#61; true;
            if (timeoutNeeded) {
                scheduleServiceTimeoutLocked(r.app);
            }
        }
        r.executeFg |&#61; fg;
        r.executeNesting&#43;&#43;;
        r.executingStart &#61; now;
    }
</code></pre> 
<h4><a id="92_ASscheduleServiceTimeoutLocked_997"></a>9.2 AS.scheduleServiceTimeoutLocked</h4> 
<p>[-&gt;ActiveServices.java]</p> 
<pre><code> void scheduleServiceTimeoutLocked(ProcessRecord proc) {
        if (proc.executingServices.size() &#61;&#61; 0 || proc.thread &#61;&#61; null) {
            return;
        }
        Message msg &#61; mAm.mHandler.obtainMessage(
                ActivityManagerService.SERVICE_TIMEOUT_MSG);
        msg.obj &#61; proc;
        //超时后还没有移除SERVICE_TIMEOUT_MSG消息&#xff0c;则执行SERVICE_TIMEOUT流程
        mAm.mHandler.sendMessageDelayed(msg,
                proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
    }
</code></pre> 
<p>发送延迟消息SERVICE_TIMEOUT_MSG&#xff1a;</p> 
<ul><li> <p>对于前台服务&#xff0c;则超时为SERVICE_TIMEOUT&#61;20s</p> </li><li> <p>对于后台服务&#xff0c;则超时为SERVICE_BACKGROUND_TIMEOUT&#61;SERVICE_TIMEOUT * 10</p> </li></ul> 
<p>app.thread.scheduleCreateService通过Binder机制调用&#xff0c;IApplicationThread.aidl在编译时会生成代理类和实现类。通过IApplicationThread代理&#xff0c;调用到目标进程端的scheduleCreateService具体实现。</p> 
<h2><a id="_1023"></a>四、目标进程端</h2> 
<h3><a id="10ATscheduleCreateService_1025"></a>10.AT.scheduleCreateService</h3> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code>  public final void scheduleCreateService(IBinder token,
                ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
            updateProcessState(processState, false);
            CreateServiceData s &#61; new CreateServiceData();
            s.token &#61; token;
            s.info &#61; info;
            s.compatInfo &#61; compatInfo;
            sendMessage(H.CREATE_SERVICE, s);
   }
</code></pre> 
<p>该方法执行在ActivityThread线程。</p> 
<h4><a id="101_AThandleMessage_1043"></a>10.1 AT.handleMessage</h4> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code> public void handleMessage(Message msg) {
      ...
               case CREATE_SERVICE:
                     handleCreateService((CreateServiceData)msg.obj);
                     break;
                case BIND_SERVICE:
                    handleBindService((BindServiceData)msg.obj);
                    break;
                case UNBIND_SERVICE:
                    handleUnbindService((BindServiceData)msg.obj);
                    schedulePurgeIdler();
                    break;
                case SERVICE_ARGS:
                    handleServiceArgs((ServiceArgsData)msg.obj);//onStartCommand
                    break;
                case STOP_SERVICE:
                    handleStopService((IBinder)msg.obj);
                    schedulePurgeIdler();
                    break;
       ...    
   }
</code></pre> 
<h3><a id="11AThandleCreateService_1071"></a>11.AT.handleCreateService</h3> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code>    &#64;UnsupportedAppUsage
    private void handleCreateService(CreateServiceData data) {
        //当应用处于后台即将进行gc,而此时回调到活动状态&#xff0c;则跳过本次gc
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();
        LoadedApk packageInfo &#61; getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service &#61; null;
        //通过反射创建目标服务对象
        try {
            java.lang.ClassLoader cl &#61; packageInfo.getClassLoader();
            service &#61; packageInfo.getAppFactory()
                    .instantiateService(cl, data.info.name, data.intent);
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    &#34;Unable to instantiate service &#34; &#43; data.info.name
                    &#43; &#34;: &#34; &#43; e.toString(), e);
            }
        }

        try {
            if (localLOGV) Slog.v(TAG, &#34;Creating service &#34; &#43; data.info.name);
            //创建ContextImpl
            ContextImpl context &#61; ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);
           
            //创建Application
            Application app &#61; packageInfo.makeApplication(false, mInstrumentation);
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManager.getService());
            //调用服务onCreate方法       
            service.onCreate();
            mServices.put(data.token, service);
            //调用服务创建完成
            try {
                ActivityManager.getService().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    &#34;Unable to create service &#34; &#43; data.info.name
                    &#43; &#34;: &#34; &#43; e.toString(), e);
            }
        }
    }
</code></pre> 
<h4><a id="111_ServiceonCreate_1128"></a>11.1 Service.onCreate</h4> 
<p>[-&gt;Service.java]</p> 
<pre><code> public abstract class Service extends ContextWrapper implements ComponentCallbacks2 {
    /**
     * Called by the system when the service is first created.  Do not call this method directly.
     */
    public void onCreate() {
    }
  }  
</code></pre> 
<p>最终调用到了Service.onCreate方法&#xff0c;对于目标服务一般都是继承Service&#xff0c;并且覆写onCreate方法&#xff0c;到此终于进入到了Service的生命周期。</p> 
<h3><a id="12AMSserviceDoneExecuting_1144"></a>12.AMS.serviceDoneExecuting</h3> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code> public void serviceDoneExecuting(IBinder token, int type, int startId, int res) {
        synchronized(this) {
            if (!(token instanceof ServiceRecord)) {
                Slog.e(TAG, &#34;serviceDoneExecuting: Invalid service token&#61;&#34; &#43; token);
                throw new IllegalArgumentException(&#34;Invalid service token&#34;);
            }
            mServices.serviceDoneExecutingLocked((ServiceRecord)token, type, startId, res);
        }
    }
</code></pre> 
<p>由流程9.1 bumpServiceExecutingLocked方法发送了一个延时消息SERVICE_TIMEOUT_MSG&#xff0c;现在onCreate执行完成&#xff0c;那么就需要移除该消息&#xff0c;否则会报超时。</p> 
<h4><a id="121_ASserviceDoneExecutingLocked_1162"></a>12.1 AS.serviceDoneExecutingLocked</h4> 
<p>[-&gt;ActiveServices.java]</p> 
<pre><code> void serviceDoneExecutingLocked(ServiceRecord r, int type, int startId, int res) {
        boolean inDestroying &#61; mDestroyingServices.contains(r);
        if (r !&#61; null) {
            if (type &#61;&#61; ActivityThread.SERVICE_DONE_EXECUTING_START) {
                // This is a call from a service start...  take care of
                // book-keeping.
                r.callStart &#61; true;
                //onStartCommand返回值的处理
                switch (res) {
                    case Service.START_STICKY_COMPATIBILITY:
                    case Service.START_STICKY: {
                        // We are done with the associated start arguments.
                        r.findDeliveredStart(startId, false, true);
                        // Don&#39;t stop if killed.
                        r.stopIfKilled &#61; false;
                        break;
                    }
                    case Service.START_NOT_STICKY: {
                        // We are done with the associated start arguments.
                        r.findDeliveredStart(startId, false, true);
                        if (r.getLastStartId() &#61;&#61; startId) {
                            // There is no more work, and this service
                            // doesn&#39;t want to hang around if killed.
                            r.stopIfKilled &#61; true;
                        }
                        break;
                    }
                    case Service.START_REDELIVER_INTENT: {
                        // We&#39;ll keep this item until they explicitly
                        // call stop for it, but keep track of the fact
                        // that it was delivered.
                        ServiceRecord.StartItem si &#61; r.findDeliveredStart(startId, false, false);
                        if (si !&#61; null) {
                            si.deliveryCount &#61; 0;
                            si.doneExecutingCount&#43;&#43;;
                            // Don&#39;t stop if killed.
                            r.stopIfKilled &#61; true;
                        }
                        break;
                    }
                    case Service.START_TASK_REMOVED_COMPLETE: {
                        // Special processing for onTaskRemoved().  Don&#39;t
                        // impact normal onStartCommand() processing.
                        r.findDeliveredStart(startId, true, true);
                        break;
                    }
                    default:
                        throw new IllegalArgumentException(
                                &#34;Unknown service start result: &#34; &#43; res);
                }
                if (res &#61;&#61; Service.START_STICKY_COMPATIBILITY) {
                    r.callStart &#61; false;
                }
            } else if (type &#61;&#61; ActivityThread.SERVICE_DONE_EXECUTING_STOP) {
                // This is the final call from destroying the service...  we should
                // actually be getting rid of the service at this point.  Do some
                // validation of its state, and ensure it will be fully removed.
                if (!inDestroying) {
                    // Not sure what else to do with this...  if it is not actually in the
                    // destroying list, we don&#39;t need to make sure to remove it from it.
                    // If the app is null, then it was probably removed because the process died,
                    // otherwise wtf
                    if (r.app !&#61; null) {
                        Slog.w(TAG, &#34;Service done with onDestroy, but not inDestroying: &#34;
                                &#43; r &#43; &#34;, app&#61;&#34; &#43; r.app);
                    }
                } else if (r.executeNesting !&#61; 1) {
                    Slog.w(TAG, &#34;Service done with onDestroy, but executeNesting&#61;&#34;
                            &#43; r.executeNesting &#43; &#34;: &#34; &#43; r);
                    // Fake it to keep from ANR due to orphaned entry.
                    r.executeNesting &#61; 1;
                }
            }
            final long origId &#61; Binder.clearCallingIdentity();
            //如下
            serviceDoneExecutingLocked(r, inDestroying, inDestroying);
            Binder.restoreCallingIdentity(origId);
        } else {
            Slog.w(TAG, &#34;Done executing unknown service from pid &#34;
                    &#43; Binder.getCallingPid());
        }
    }
</code></pre> 
<h4><a id="122_ASserviceDoneExecutingLocked_1251"></a>12.2 AS.serviceDoneExecutingLocked</h4> 
<p>[-&gt;ActiveServices.java]</p> 
<pre><code> private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying,
            boolean finishing) {
        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, &#34;&lt;&lt;&lt; DONE EXECUTING &#34; &#43; r
                &#43; &#34;: nesting&#61;&#34; &#43; r.executeNesting
                &#43; &#34;, inDestroying&#61;&#34; &#43; inDestroying &#43; &#34;, app&#61;&#34; &#43; r.app);
        else if (DEBUG_SERVICE_EXECUTING) Slog.v(TAG_SERVICE_EXECUTING,
                &#34;&lt;&lt;&lt; DONE EXECUTING &#34; &#43; r.shortName);
        r.executeNesting--;
        if (r.executeNesting &lt;&#61; 0) {
            if (r.app !&#61; null) {
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE,
                        &#34;Nesting at 0 of &#34; &#43; r.shortName);
                r.app.execServicesFg &#61; false;
                r.app.executingServices.remove(r);
                if (r.app.executingServices.size() &#61;&#61; 0) {
                    //移除SERVICE_TIMEOUT_MSG消息
                    if (DEBUG_SERVICE || DEBUG_SERVICE_EXECUTING) Slog.v(TAG_SERVICE_EXECUTING,
                            &#34;No more executingServices of &#34; &#43; r.shortName);
                    mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
                } else if (r.executeFg) {
                    // Need to re-evaluate whether the app still needs to be in the foreground.
                    for (int i&#61;r.app.executingServices.size()-1; i&gt;&#61;0; i--) {
                        if (r.app.executingServices.valueAt(i).executeFg) {
                            r.app.execServicesFg &#61; true;
                            break;
                        }
                    }
                }
                if (inDestroying) {
                    if (DEBUG_SERVICE) Slog.v(TAG_SERVICE,
                            &#34;doneExecuting remove destroying &#34; &#43; r);
                    mDestroyingServices.remove(r);
                    r.bindings.clear();
                }
                //更新adj
                mAm.updateOomAdjLocked(r.app, true);
            }
            r.executeFg &#61; false;
            if (r.tracker !&#61; null) {
                r.tracker.setExecuting(false, mAm.mProcessStats.getMemFactorLocked(),
                        SystemClock.uptimeMillis());
                if (finishing) {
                    r.tracker.clearCurrentOwner(r, false);
                    r.tracker &#61; null;
                }
            }
            if (finishing) {
                if (r.app !&#61; null &amp;&amp; !r.app.persistent) {
                    r.app.services.remove(r);
                    if (r.whitelistManager) {
                        updateWhitelistManagerLocked(r.app);
                    }
                }
                r.app &#61; null;
            }
        }
    }
</code></pre> 
<p>该方法将会移除SERVICE_TIMEOUT_MSG消息。Service启动过程出现ANR&#xff0c;会发送超时serviceRecord消息&#xff0c;这通常是onCreate的回调方法过长导致。</p> 
<p>realStartServiceLocked方法&#xff0c;在完成onCreate操作时&#xff0c;后面进入到了onStartCoomnad方法.</p> 
<h3><a id="13ASsendServiceArgsLocked_1319"></a>13.AS.sendServiceArgsLocked</h3> 
<p>[-&gt;ActiveServices.java]</p> 
<pre><code> private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
            boolean oomAdjusted) throws TransactionTooLargeException {
        final int N &#61; r.pendingStarts.size();
        if (N &#61;&#61; 0) {
            return;
        }
        ArrayList&lt;ServiceStartArgs&gt; args &#61; new ArrayList&lt;&gt;();
        while (r.pendingStarts.size() &gt; 0) {
            ServiceRecord.StartItem si &#61; r.pendingStarts.remove(0);
            if (DEBUG_SERVICE) {
                Slog.v(TAG_SERVICE, &#34;Sending arguments to: &#34;
                        &#43; r &#43; &#34; &#34; &#43; r.intent &#43; &#34; args&#61;&#34; &#43; si.intent);
            }
            if (si.intent &#61;&#61; null &amp;&amp; N &gt; 1) {
                // If somehow we got a dummy null intent in the middle,
                // then skip it.  DO NOT skip a null intent when it is
                // the only one in the list -- this is to support the
                // onStartCommand(null) case.
                continue;
            }
            si.deliveredTime &#61; SystemClock.uptimeMillis();
            r.deliveredStarts.add(si);
            si.deliveryCount&#43;&#43;;
            if (si.neededGrants !&#61; null) {
                mAm.grantUriPermissionUncheckedFromIntentLocked(si.neededGrants,
                        si.getUriPermissionsLocked());
            }
            mAm.grantEphemeralAccessLocked(r.userId, si.intent,
                    r.appInfo.uid, UserHandle.getAppId(si.callingId));
            //标记启动开始&#xff0c;同上
            bumpServiceExecutingLocked(r, execInFg, &#34;start&#34;);
            if (!oomAdjusted) {
                oomAdjusted &#61; true;
                mAm.updateOomAdjLocked(r.app, true);
            }
            if (r.fgRequired &amp;&amp; !r.fgWaiting) {
                if (!r.isForeground) {
                    if (DEBUG_BACKGROUND_CHECK) {
                        Slog.i(TAG, &#34;Launched service must call startForeground() within timeout: &#34; &#43; r);
                    }
                    scheduleServiceForegroundTransitionTimeoutLocked(r);
                } else {
                    if (DEBUG_BACKGROUND_CHECK) {
                        Slog.i(TAG, &#34;Service already foreground; no new timeout: &#34; &#43; r);
                    }
                    r.fgRequired &#61; false;
                }
            }
            int flags &#61; 0;
            if (si.deliveryCount &gt; 1) {
                flags |&#61; Service.START_FLAG_RETRY;
            }
            if (si.doneExecutingCount &gt; 0) {
                flags |&#61; Service.START_FLAG_REDELIVERY;
            }
            args.add(new ServiceStartArgs(si.taskRemoved, si.id, flags, si.intent));
        }

        ParceledListSlice&lt;ServiceStartArgs&gt; slice &#61; new ParceledListSlice&lt;&gt;(args);
        slice.setInlineCountLimit(4);
        Exception caughtException &#61; null;
        try {
            //流程同onCreate&#xff0c;最后回调onStartCommand
            r.app.thread.scheduleServiceArgs(r, slice);
        } catch (TransactionTooLargeException e) {
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, &#34;Transaction too large for &#34; &#43; args.size()
                    &#43; &#34; args, first: &#34; &#43; args.get(0).args);
            Slog.w(TAG, &#34;Failed delivering service starts&#34;, e);
            caughtException &#61; e;
        } catch (RemoteException e) {
            // Remote process gone...  we&#39;ll let the normal cleanup take care of this.
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, &#34;Crashed while sending args: &#34; &#43; r);
            Slog.w(TAG, &#34;Failed delivering service starts&#34;, e);
            caughtException &#61; e;
        } catch (Exception e) {
            Slog.w(TAG, &#34;Unexpected exception&#34;, e);
            caughtException &#61; e;
        }

        if (caughtException !&#61; null) {
            // Keep nesting count correct
            final boolean inDestroying &#61; mDestroyingServices.contains(r);
            for (int i &#61; 0; i &lt; args.size(); i&#43;&#43;) {
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);
            }
            if (caughtException instanceof TransactionTooLargeException) {
                throw (TransactionTooLargeException)caughtException;
            }
        }
    }
</code></pre> 
<p>realStartServiceLocked先后执行的方法如下&#xff1a;</p> 
<p>执行scheduleCreateService方法&#xff0c;通过层层回调到Service.onCreate;</p> 
<p>执行scheduleServiceArgs方法&#xff0c;通过层层回调到Service.onStartCommand&#xff0c;流程同onCreate.</p> 
<h2><a id="_1422"></a>五、总结</h2> 
<p>整个startService过程&#xff0c;从进程角度来看服务启动的过程</p> 
<p>process A进程&#xff1a;是指调用startService方法所在的进程&#xff0c;也就是启动服务的发起端进程&#xff0c;如&#xff1a;桌面上点击图标&#xff0c;此处Process A就是Launcher所在的进程。</p> 
<p>system_server进程&#xff1a;里面运行着大量的系统服务&#xff0c;如AMS,是运行在system_server不同的线程中&#xff0c;基于Binder接口&#xff0c;binder线程的创建与销毁都是由Binder驱动来决定的&#xff0c;每个进程的binder线程个数上线为16</p> 
<p>Zygote进程&#xff1a;是由init进程孵化而来&#xff0c;用于创建java层进程的母体&#xff0c;所有的java层进程都是由Zygote进程孵化而来。</p> 
<p>Remote Service进程&#xff1a;远程服务所在的进程&#xff0c;是由zygote进程孵化而来&#xff0c;用于运行Remote服务进程。主线程主要是负责Activity/Service等组件的生命周期以及UI相关的操作&#xff1b;另外&#xff0c;每个app进程至少会有两个binder线程&#xff0c;ApplicationThread和ActivityManagerService的代理。</p> 
<p>上面涉及到IPC通信的三种方式&#xff0c;Binder、Socket、Handler。一般来说&#xff0c;同一进程内的线程间通信采用Handler消息队列机制&#xff0c;不同进程间的通信采用Binder机制&#xff0c;另外与Zygote通信采用Socket。</p> 
<p>启动流程&#xff1a;</p> 
<p>1.Process A进程采用Binder IPC向system_server进程发起startService请求</p> 
<p>2.system_server进程接收到请求后&#xff0c;向zygote进程发送创建进程的请求</p> 
<p>3.zygote进程fork出新的子进程Remote Service进程</p> 
<p>4.RemoteService进程通过Binder IPC向system_server进程发起attachApplication请求</p> 
<p>5.system_server进程收到请求后&#xff0c;进行一序列准备工作后&#xff0c;再通过Binder IPC向remote sevice进程发送scheduleCreateService请求</p> 
<p>6.Remote Service进程的binder线程收到请求后&#xff0c;通过handler向主线程发送CREATE_SERVICE消息</p> 
<p>7.主线程收到消息后&#xff0c;通过反射机制创建目标Service&#xff0c;并回调Service.onCreate方法。</p> 
<p>到此&#xff0c;服务就正式启动了。当创建的是本地服务或者服务所属进程已经创建&#xff0c;则无需经过步骤2,3直接创建服务即可。</p> 
<h2><a id="_1454"></a>附录</h2> 
<p>源码路径</p> 
<pre><code>frameworks/base/core/java/android/app/IActivityManager.aidl
frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
frameworks/base/core/java/android/app/ActivityThread.java
frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
frameworks/base/core/java/android/app/IApplicationThread.aidl
</code></pre>