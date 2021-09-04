---
layout:     post
title:      Android10 StartActivity启动过程分析
subtitle:   startActivity的整体流程和startService相近，启动后都是通过AMS来完成的。但相比service启动更加复杂
date:       2020-06-17
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android10
    - Android
    - framework
    - 组件学习
---
 
<blockquote> 
 <p>基于Android10.0&#xff0c;分析startActivity的启动过程</p> 
</blockquote> 
<h2><a id="_4"></a>一、概述</h2> 
<p>startActivity的整体流程和startService相近&#xff0c;启动后都是通过AMS来完成的。但相比service启动更加复杂&#xff0c;多了任务栈、UI、生命周期。其启动流程如下&#xff1a;<br /> <img src="https://img-blog.csdnimg.cn/20200105191423838.jpg?x-oss-process&#61;image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Nhbzg2MTU0NDMyNQ&#61;&#61;,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" /></p> 
<h2><a id="_9"></a>二、启动流程</h2> 
<p>启动Activity&#xff0c;一般是用startActivity。</p> 
<h3><a id="21_ActivitystartActivity_13"></a>2.1 Activity.startActivity</h3> 
<p>[-&gt;Activity.java]</p> 
<pre><code>   &#64;Override
    public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }
     &#64;Override
    public void startActivity(Intent intent, &#64;Nullable Bundle options) {
        if (options !&#61; null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }
</code></pre> 
<h3><a id="22__startActivityForResult_31"></a>2.2 startActivityForResult</h3> 
<p>[-&gt;ContextImpl.java]</p> 
<pre><code>  /**
     * &#64;hide
     */
    &#64;Override
    &#64;UnsupportedAppUsage
    public void startActivityForResult(
            String who, Intent intent, int requestCode, &#64;Nullable Bundle options) {
        Uri referrer &#61; onProvideReferrer();
        if (referrer !&#61; null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        options &#61; transferSpringboardActivityOptions(options);
        Instrumentation.ActivityResult ar &#61;
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, who,
                intent, requestCode, options);
        if (ar !&#61; null) {
            mMainThread.sendActivityResult(
                mToken, who, requestCode,
                ar.getResultCode(), ar.getResultData());
        }
        cancelInputsAndStartExitTransition(options);
    }
</code></pre> 
<p>execStartActivity方法参数&#xff1a;</p> 
<p>mAppThread:类型为ApplicationThread,通过mMainThread.getApplicationThread()获取。</p> 
<p>mToken&#xff1a;为Binder类型</p> 
<h3><a id="23__execStartActivity_67"></a>2.3 execStartActivity</h3> 
<p>[-&gt;Instrumentation.java]</p> 
<pre><code>  public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, String target,
        Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread &#61; (IApplicationThread) contextThread;
        if (mActivityMonitors !&#61; null) {
            synchronized (mSync) {
                final int N &#61; mActivityMonitors.size();
                for (int i&#61;0; i&lt;N; i&#43;&#43;) {
                    final ActivityMonitor am &#61; mActivityMonitors.get(i);
                    ActivityResult result &#61; null;
                    if (am.ignoreMatchingSpecificIntents()) {
                        result &#61; am.onStartActivity(intent);
                    }
                    if (result !&#61; null) {
                        am.mHits&#43;&#43;;
                        return result;
                    } else if (am.match(who, null, intent)) {
                        am.mHits&#43;&#43;;
                        //如果am阻塞activity启动&#xff0c;则返回
                        if (am.isBlocking()) {
                            return requestCode &gt;&#61; 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            int result &#61; ActivityManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target, requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException(&#34;Failure from system&#34;, e);
        }
        return null;
    }
</code></pre> 
<h4><a id="231__AMgetService_114"></a>2.3.1 AM.getService</h4> 
<p>和Android6.0不同的是Android10.0直接通过AIDL的方式生成了AMS的代理。</p> 
<pre><code> /**
     * &#64;hide
     */
    &#64;UnsupportedAppUsage
    public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }
    public abstract class Singleton&lt;T&gt; {
    &#64;UnsupportedAppUsage
    private T mInstance;

    protected abstract T create();

    &#64;UnsupportedAppUsage
    public final T get() {
        synchronized (this) {
            if (mInstance &#61;&#61; null) {
                mInstance &#61; create();
            }
            return mInstance;
        }
    }
  }  
    &#64;UnsupportedAppUsage
    private static final Singleton&lt;IActivityManager&gt; IActivityManagerSingleton &#61;
            new Singleton&lt;IActivityManager&gt;() {
                &#64;Override
                protected IActivityManager create() {
                    final IBinder b &#61; ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    //获取AMS的代理
                    final IActivityManager am &#61; IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };
</code></pre> 
<h3><a id="24__IActivityManagerstartActivity_155"></a>2.4 IActivityManager.startActivity</h3> 
<p>通过AIDL生成的代理类调用AMS的startActivity&#xff0c;其代理类在编译的时候&#xff0c;会自动生成。</p> 
<p>startActivity共有10个参数&#xff0c;参数对应值如下&#xff1a;</p> 
<ul><li>caller&#xff1a;当前应用的Application对象mAppThread</li><li>callingPackage:当前Activity所在的包名</li><li>intent&#xff1a;启动Activity传过来的参数</li><li>resolvedType&#xff1a;调用intent.resolveTypeIfNeeded获取</li><li>resultTo&#xff1a;来自当前Activity.mToken</li><li>resultWho&#xff1a; 来自当前Activity.mEmbeddedID</li><li>requestCode:-1</li><li>startFlags:0</li><li>profilerInfo&#xff1a;null</li><li>bOptions:null</li></ul> 
<h3><a id="25_AMSstartActivity_172"></a>2.5 AMS.startActivity</h3> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code>   &#64;Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }
     &#64;Override
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
                true /*validateIncomingUser*/);
    }
</code></pre> 
<h3><a id="26_AMSstartActivityAsUser_195"></a>2.6 AMS.startActivityAsUser</h3> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code> public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
            boolean validateIncomingUser) {
        enforceNotIsolatedCaller(&#34;startActivity&#34;);

        userId &#61; mActivityStartController.checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), &#34;startActivityAsUser&#34;);

        // TODO: Switch to user app stacks here.
        return mActivityStartController.obtainStarter(intent, &#34;startActivityAsUser&#34;)
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setMayWait(userId)
                .execute();

    }
</code></pre> 
<p>通过建造者模式&#xff0c;来设置参数&#xff0c;其参数在2.4节有介绍&#xff0c;通过execute方法最后执行。</p> 
<h4><a id="261__ASCobtainStarter_228"></a>2.6.1 ASC.obtainStarter</h4> 
<p>[-&gt;ActivityStartController.java]</p> 
<pre><code> /**
     * &#64;return A starter to configure and execute starting an activity. It is valid until after
     *         {&#64;link ActivityStarter#execute} is invoked. At that point, the starter should be
     *         considered invalid and no longer modified or used.
     */
    ActivityStarter obtainStarter(Intent intent, String reason) {
        return mFactory.obtain().setIntent(intent).setReason(reason);
    }
</code></pre> 
<h4><a id="262__ASsetMayWait_243"></a>2.6.2 AS.setMayWait</h4> 
<p>这个方法在2.6.3节中用到&#xff0c;可以看到这里将mRequest.mayWait设置为true</p> 
<pre><code>   ActivityStarter setMayWait(int userId) {
        mRequest.mayWait &#61; true;
        mRequest.userId &#61; userId;
        return this;
    }
</code></pre> 
<h4><a id="263__ASexecute_255"></a>2.6.3 AS.execute</h4> 
<p>[-&gt;ActivityStarter.java]</p> 
<pre><code> /**
     * Starts an activity based on the request parameters provided earlier.
     * &#64;return The starter result.
     */
    int execute() {
        try {
            // TODO(b/64750076): Look into passing request directly to these methods to allow
            // for transactional diffs and preprocessing.
            //经过该方法&#xff0c;前面已经设置为true
            if (mRequest.mayWait) {
                return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                        mRequest.callingPackage, mRequest.intent, mRequest.resolvedType,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                        mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                        mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                        mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup,
                        mRequest.originatingPendingIntent);
            } else {
                return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                        mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                        mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                        mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                        mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                        mRequest.outActivity, mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup,
                        mRequest.originatingPendingIntent);
            }
        } finally {
            onExecutionComplete();
        }
    }
</code></pre> 
<h3><a id="27__ASstartActivityMayWait_297"></a>2.7 AS.startActivityMayWait</h3> 
<p>这个方法的参数有21个&#xff0c;具体重要的几个参数2.4节已经介绍过, inTask &#61; null。</p> 
<p>[-&gt;ActivityStarter.java]</p> 
<pre><code> private int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult,
            Configuration globalConfig, SafeActivityOptions options, boolean ignoreTargetSecurity,
            int userId, TaskRecord inTask, String reason,
            boolean allowPendingRemoteAnimationRegistryLookup,
            PendingIntentRecord originatingPendingIntent) {
        // Refuse possible leaked file descriptors
        if (intent !&#61; null &amp;&amp; intent.hasFileDescriptors()) {
            throw new IllegalArgumentException(&#34;File descriptors passed in Intent&#34;);
        }
        //activity开始启动日志
        mSupervisor.getActivityMetricsLogger().notifyActivityLaunching();
        boolean componentSpecified &#61; intent.getComponent() !&#61; null;

        final int realCallingPid &#61; Binder.getCallingPid();
        final int realCallingUid &#61; Binder.getCallingUid();

        int callingPid;
        if (callingUid &gt;&#61; 0) {
            callingPid &#61; -1;
        } else if (caller &#61;&#61; null) {
            callingPid &#61; realCallingPid;
            callingUid &#61; realCallingUid;
        } else {
            callingPid &#61; callingUid &#61; -1;
        }

        // Save a copy in case ephemeral needs it
        final Intent ephemeralIntent &#61; new Intent(intent);
        // Don&#39;t modify the client&#39;s object!
        intent &#61; new Intent(intent);
        //对一些特殊的intent做处理
        if (componentSpecified
                &amp;&amp; !(Intent.ACTION_VIEW.equals(intent.getAction()) &amp;&amp; intent.getData() &#61;&#61; null)
                &amp;&amp; !Intent.ACTION_INSTALL_INSTANT_APP_PACKAGE.equals(intent.getAction())
                &amp;&amp; !Intent.ACTION_RESOLVE_INSTANT_APP_PACKAGE.equals(intent.getAction())
                &amp;&amp; mService.getPackageManagerInternalLocked()
                        .isInstantAppInstallerComponent(intent.getComponent())) {
            // intercept intents targeted directly to the ephemeral installer the
            // ephemeral installer should never be started with a raw Intent; instead
            // adjust the intent so it looks like a &#34;normal&#34; instant app launch
            intent.setComponent(null /*component*/);
            componentSpecified &#61; false;
        }
        //处理intent信息&#xff0c;当存在多个activity时&#xff0c;弹出resolverAcitvity
        ResolveInfo rInfo &#61; mSupervisor.resolveIntent(intent, resolvedType, userId,
                0 /* matchFlags */,
                        computeResolveFilterUid(
                                callingUid, realCallingUid, mRequest.filterCallingUid));
        if (rInfo &#61;&#61; null) {
            UserInfo userInfo &#61; mSupervisor.getUserInfo(userId);
            if (userInfo !&#61; null &amp;&amp; userInfo.isManagedProfile()) {
                // Special case for managed profiles, if attempting to launch non-cryto aware
                // app in a locked managed profile from an unlocked parent allow it to resolve
                // as user will be sent via confirm credentials to unlock the profile.
                UserManager userManager &#61; UserManager.get(mService.mContext);
                boolean profileLockedAndParentUnlockingOrUnlocked &#61; false;
                long token &#61; Binder.clearCallingIdentity();
                try {
                    UserInfo parent &#61; userManager.getProfileParent(userId);
                    profileLockedAndParentUnlockingOrUnlocked &#61; (parent !&#61; null)
                            &amp;&amp; userManager.isUserUnlockingOrUnlocked(parent.id)
                            &amp;&amp; !userManager.isUserUnlockingOrUnlocked(userId);
                } finally {
                    Binder.restoreCallingIdentity(token);
                }
                if (profileLockedAndParentUnlockingOrUnlocked) {
                    rInfo &#61; mSupervisor.resolveIntent(intent, resolvedType, userId,
                            PackageManager.MATCH_DIRECT_BOOT_AWARE
                                    | PackageManager.MATCH_DIRECT_BOOT_UNAWARE,
                            computeResolveFilterUid(
                                    callingUid, realCallingUid, mRequest.filterCallingUid));
                }
            }
        }
        //收集intent所指向的activity信息
        // Collect information about the target of the Intent.
        ActivityInfo aInfo &#61; mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);

        synchronized (mService) {
            final ActivityStack stack &#61; mSupervisor.mFocusedStack;
            stack.mConfigWillChange &#61; globalConfig !&#61; null
                    &amp;&amp; mService.getGlobalConfiguration().diff(globalConfig) !&#61; 0;
            if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION,
                    &#34;Starting activity when config will change &#61; &#34; &#43; stack.mConfigWillChange);

            final long origId &#61; Binder.clearCallingIdentity();

            if (aInfo !&#61; null &amp;&amp;
                    (aInfo.applicationInfo.privateFlags
                            &amp; ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) !&#61; 0 &amp;&amp;
                    mService.mHasHeavyWeightFeature) {
                //heavy-weight进程处理流程
                // This may be a heavy-weight process!  Check to see if we already
                // have another, different heavy-weight process running.
                if (aInfo.processName.equals(aInfo.applicationInfo.packageName)) {
                    final ProcessRecord heavy &#61; mService.mHeavyWeightProcess;
                    if (heavy !&#61; null &amp;&amp; (heavy.info.uid !&#61; aInfo.applicationInfo.uid
                            || !heavy.processName.equals(aInfo.processName))) {
                        int appCallingUid &#61; callingUid;
                        if (caller !&#61; null) {
                            ProcessRecord callerApp &#61; mService.getRecordForAppLocked(caller);
                            if (callerApp !&#61; null) {
                                appCallingUid &#61; callerApp.info.uid;
                            } else {
                                Slog.w(TAG, &#34;Unable to find app for caller &#34; &#43; caller
                                        &#43; &#34; (pid&#61;&#34; &#43; callingPid &#43; &#34;) when starting: &#34;
                                        &#43; intent.toString());
                                SafeActivityOptions.abort(options);
                                return ActivityManager.START_PERMISSION_DENIED;
                            }
                        }

                        IIntentSender target &#61; mService.getIntentSenderLocked(
                                ActivityManager.INTENT_SENDER_ACTIVITY, &#34;android&#34;,
                                appCallingUid, userId, null, null, 0, new Intent[] { intent },
                                new String[] { resolvedType }, PendingIntent.FLAG_CANCEL_CURRENT
                                        | PendingIntent.FLAG_ONE_SHOT, null);

                        Intent newIntent &#61; new Intent();
                        if (requestCode &gt;&#61; 0) {
                            // Caller is requesting a result.
                            newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_HAS_RESULT, true);
                        }
                        newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_INTENT,
                                new IntentSender(target));
                        if (heavy.activities.size() &gt; 0) {
                            ActivityRecord hist &#61; heavy.activities.get(0);
                            newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_CUR_APP,
                                    hist.packageName);
                            newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_CUR_TASK,
                                    hist.getTask().taskId);
                        }
                        newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_NEW_APP,
                                aInfo.packageName);
                        newIntent.setFlags(intent.getFlags());
                        newIntent.setClassName(&#34;android&#34;,
                                HeavyWeightSwitcherActivity.class.getName());
                        intent &#61; newIntent;
                        resolvedType &#61; null;
                        caller &#61; null;
                        callingUid &#61; Binder.getCallingUid();
                        callingPid &#61; Binder.getCallingPid();
                        componentSpecified &#61; true;
                        rInfo &#61; mSupervisor.resolveIntent(intent, null /*resolvedType*/, userId,
                                0 /* matchFlags */, computeResolveFilterUid(
                                        callingUid, realCallingUid, mRequest.filterCallingUid));
                        aInfo &#61; rInfo !&#61; null ? rInfo.activityInfo : null;
                        if (aInfo !&#61; null) {
                            aInfo &#61; mService.getActivityInfoForUser(aInfo, userId);
                        }
                    }
                }
            }

            final ActivityRecord[] outRecord &#61; new ActivityRecord[1];
            //见2.8节
            int res &#61; startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo,
                    voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid,
                    callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options,
                    ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason,
                    allowPendingRemoteAnimationRegistryLookup, originatingPendingIntent);

            Binder.restoreCallingIdentity(origId);

            if (stack.mConfigWillChange) {
                // If the caller also wants to switch to a new configuration,
                // do so now.  This allows a clean switch, as we are waiting
                // for the current activity to pause (so we will not destroy
                // it), and have not yet started the next activity.
                mService.enforceCallingPermission(android.Manifest.permission.CHANGE_CONFIGURATION,
                        &#34;updateConfiguration()&#34;);
                stack.mConfigWillChange &#61; false;
                if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION,
                        &#34;Updating to new configuration after starting activity.&#34;);
                mService.updateConfigurationLocked(globalConfig, null, false);
            }

            // Notify ActivityMetricsLogger that the activity has launched. ActivityMetricsLogger
            // will then wait for the windows to be drawn and populate WaitResult.
            mSupervisor.getActivityMetricsLogger().notifyActivityLaunched(res, outRecord[0]);
            if (outResult !&#61; null) {
                outResult.result &#61; res;

                final ActivityRecord r &#61; outRecord[0];

                switch(res) {
                    case START_SUCCESS: {
                        mSupervisor.mWaitingActivityLaunched.add(outResult);
                        do {
                            try {
                                mService.wait();
                            } catch (InterruptedException e) {
                            }
                        } while (outResult.result !&#61; START_TASK_TO_FRONT
                                &amp;&amp; !outResult.timeout &amp;&amp; outResult.who &#61;&#61; null);
                        if (outResult.result &#61;&#61; START_TASK_TO_FRONT) {
                            res &#61; START_TASK_TO_FRONT;
                        }
                        break;
                    }
                    case START_DELIVERED_TO_TOP: {
                        outResult.timeout &#61; false;
                        outResult.who &#61; r.realActivity;
                        outResult.totalTime &#61; 0;
                        break;
                    }
                    case START_TASK_TO_FRONT: {
                        // ActivityRecord may represent a different activity, but it should not be
                        // in the resumed state.
                        if (r.nowVisible &amp;&amp; r.isState(RESUMED)) {
                            outResult.timeout &#61; false;
                            outResult.who &#61; r.realActivity;
                            outResult.totalTime &#61; 0;
                        } else {
                            final long startTimeMs &#61; SystemClock.uptimeMillis();
                            mSupervisor.waitActivityVisible(r.realActivity, outResult, startTimeMs);
                            // Note: the timeout variable is not currently not ever set.
                            do {
                                try {
                                    mService.wait();
                                } catch (InterruptedException e) {
                                }
                            } while (!outResult.timeout &amp;&amp; outResult.who &#61;&#61; null);
                        }
                        break;
                    }
                }
            }

            return res;
        }
    }

</code></pre> 
<h4><a id="271_PKMSresolveIntent_543"></a>2.7.1 PKMS.resolveIntent</h4> 
<p>[-&gt;PackageManagerService.java]</p> 
<p>mSupervisor.resolveInten经过层层调用&#xff0c;通过IPC最后会调用PKMS对象中的resolveIntent。</p> 
<pre><code> /**
     * Normally instant apps can only be resolved when they&#39;re visible to the caller.
     * However, if {&#64;code resolveForStart} is {&#64;code true}, all instant apps are visible
     * since we need to allow the system to start any installed application.
     */
    private ResolveInfo resolveIntentInternal(Intent intent, String resolvedType,
            int flags, int userId, boolean resolveForStart, int filterCallingUid) {
        try {
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, &#34;resolveIntent&#34;);

            if (!sUserManager.exists(userId)) return null;
            final int callingUid &#61; Binder.getCallingUid();
            flags &#61; updateFlagsForResolve(flags, userId, intent, filterCallingUid, resolveForStart);
            mPermissionManager.enforceCrossUserPermission(callingUid, userId,
                    false /*requireFullPermission*/, false /*checkShell*/, &#34;resolve intent&#34;);
 
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, &#34;queryIntentActivities&#34;);
            //找到相应的activity组件&#xff0c;并保存intent对象
            final List&lt;ResolveInfo&gt; query &#61; queryIntentActivitiesInternal(intent, resolvedType,
                    flags, filterCallingUid, userId, resolveForStart, true /*allowDynamicSplits*/);
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
            //根据priority选择最佳的activity
            final ResolveInfo bestChoice &#61;
                    chooseBestActivity(intent, resolvedType, flags, query, userId);
            return bestChoice;
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
    }
</code></pre> 
<h4><a id="272_ASSresolveActivity_581"></a>2.7.2 ASS.resolveActivity</h4> 
<p>[-&gt;ActivityStackSupervisor.java]</p> 
<pre><code> ActivityInfo resolveActivity(Intent intent, ResolveInfo rInfo, int startFlags,
            ProfilerInfo profilerInfo) {
        final ActivityInfo aInfo &#61; rInfo !&#61; null ? rInfo.activityInfo : null;
        if (aInfo !&#61; null) {
            // Store the found target back into the intent, because now that
            // we have it we never want to do this again.  For example, if the
            // user navigates back to this point in the history, we should
            // always restart the exact same activity.
            intent.setComponent(new ComponentName(
                    aInfo.applicationInfo.packageName, aInfo.name));

            // Don&#39;t debug things in the system process
            if (!aInfo.processName.equals(&#34;system&#34;)) {
                if ((startFlags &amp; ActivityManager.START_FLAG_DEBUG) !&#61; 0) {
                    mService.setDebugApp(aInfo.processName, true, false);
                }

                if ((startFlags &amp; ActivityManager.START_FLAG_NATIVE_DEBUGGING) !&#61; 0) {
                    mService.setNativeDebuggingAppLocked(aInfo.applicationInfo, aInfo.processName);
                }

                if ((startFlags &amp; ActivityManager.START_FLAG_TRACK_ALLOCATION) !&#61; 0) {
                    mService.setTrackAllocationApp(aInfo.applicationInfo, aInfo.processName);
                }

                if (profilerInfo !&#61; null) {
                    mService.setProfileApp(aInfo.applicationInfo, aInfo.processName, profilerInfo);
                }
            }
            final String intentLaunchToken &#61; intent.getLaunchToken();
            if (aInfo.launchToken &#61;&#61; null &amp;&amp; intentLaunchToken !&#61; null) {
                aInfo.launchToken &#61; intentLaunchToken;
            }
        }
        return aInfo;
    }
</code></pre> 
<p>Activity类有3个flags用于调试</p> 
<ul><li>START_FLAG_DEBUG&#xff1a;用于调试debug app</li><li>START_FLAG_NATIVE_DEBUGGING&#xff1a;用于调试native</li><li>START_FLAG_TRACK_ALLOCATION&#xff1a;用于调试allocation tracking</li></ul> 
<h3><a id="28__ASstartActivity_630"></a>2.8 AS.startActivity</h3> 
<p>[-&gt;ActivityStarter.java]</p> 
<pre><code>  private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            SafeActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
            ActivityRecord[] outActivity, TaskRecord inTask, String reason,
            boolean allowPendingRemoteAnimationRegistryLookup,
            PendingIntentRecord originatingPendingIntent) {

        if (TextUtils.isEmpty(reason)) {
            throw new IllegalArgumentException(&#34;Need to specify a reason.&#34;);
        }
        mLastStartReason &#61; reason;
        mLastStartActivityTimeMs &#61; System.currentTimeMillis();
        mLastStartActivityRecord[0] &#61; null;

        mLastStartActivityResult &#61; startActivity(caller, intent, ephemeralIntent, resolvedType,
                aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode,
                callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                options, ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
                inTask, allowPendingRemoteAnimationRegistryLookup, originatingPendingIntent);

        if (outActivity !&#61; null) {
            // mLastStartActivityRecord[0] is set in the call to startActivity above.
            outActivity[0] &#61; mLastStartActivityRecord[0];
        }

        return getExternalResult(mLastStartActivityResult);
    }
</code></pre> 
<p>下面才正式进入startActivity具体内容</p> 
<pre><code> private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            SafeActivityOptions options,
            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
            TaskRecord inTask, boolean allowPendingRemoteAnimationRegistryLookup,
            PendingIntentRecord originatingPendingIntent) {
        int err &#61; ActivityManager.START_SUCCESS;
        // Pull the optional Ephemeral Installer-only bundle out of the options early.
        final Bundle verificationBundle
                &#61; options !&#61; null ? options.popAppVerificationBundle() : null;
        //获取调用者的进程记录对象
        ProcessRecord callerApp &#61; null;
        if (caller !&#61; null) {
            callerApp &#61; mService.getRecordForAppLocked(caller);
            if (callerApp !&#61; null) {
                callingPid &#61; callerApp.pid;
                callingUid &#61; callerApp.info.uid;
            } else {
                Slog.w(TAG, &#34;Unable to find app for caller &#34; &#43; caller
                        &#43; &#34; (pid&#61;&#34; &#43; callingPid &#43; &#34;) when starting: &#34;
                        &#43; intent.toString());
                err &#61; ActivityManager.START_PERMISSION_DENIED;
            }
        }

        final int userId &#61; aInfo !&#61; null &amp;&amp; aInfo.applicationInfo !&#61; null
                ? UserHandle.getUserId(aInfo.applicationInfo.uid) : 0;

        if (err &#61;&#61; ActivityManager.START_SUCCESS) {
            Slog.i(TAG, &#34;START u&#34; &#43; userId &#43; &#34; {&#34; &#43; intent.toShortString(true, true, true, false)
                    &#43; &#34;} from uid &#34; &#43; callingUid);
        }

        //获取调用者所在的activity
        ActivityRecord sourceRecord &#61; null;
        ActivityRecord resultRecord &#61; null;
        if (resultTo !&#61; null) {
            sourceRecord &#61; mSupervisor.isInAnyStackLocked(resultTo);
            if (DEBUG_RESULTS) Slog.v(TAG_RESULTS,
                    &#34;Will send result to &#34; &#43; resultTo &#43; &#34; &#34; &#43; sourceRecord);
            if (sourceRecord !&#61; null) {
                //requestCode &#61; -1 不会进入
                if (requestCode &gt;&#61; 0 &amp;&amp; !sourceRecord.finishing) {
                    resultRecord &#61; sourceRecord;
                }
            }
        }

        final int launchFlags &#61; intent.getFlags();

        if ((launchFlags &amp; Intent.FLAG_ACTIVITY_FORWARD_RESULT) !&#61; 0 &amp;&amp; sourceRecord !&#61; null) {
            //activity执行结果的返回由源activity切换到新activity&#xff0c;不需要返回结果则不会进该分支  
            // Transfer the result target from the source activity to the new
            // one being started, including any failures.
            if (requestCode &gt;&#61; 0) {
                SafeActivityOptions.abort(options);
                return ActivityManager.START_FORWARD_AND_REQUEST_CONFLICT;
            }
            resultRecord &#61; sourceRecord.resultTo;
            if (resultRecord !&#61; null &amp;&amp; !resultRecord.isInStackLocked()) {
                resultRecord &#61; null;
            }
            resultWho &#61; sourceRecord.resultWho;
            requestCode &#61; sourceRecord.requestCode;
            sourceRecord.resultTo &#61; null;
            if (resultRecord !&#61; null) {
                resultRecord.removeResultsLocked(sourceRecord, resultWho, requestCode);
            }
            if (sourceRecord.launchedFromUid &#61;&#61; callingUid) {
                // The new activity is being launched from the same uid as the previous
                // activity in the flow, and asking to forward its result back to the
                // previous.  In this case the activity is serving as a trampoline between
                // the two, so we also want to update its launchedFromPackage to be the
                // same as the previous activity.  Note that this is safe, since we know
                // these two packages come from the same uid; the caller could just as
                // well have supplied that same package name itself.  This specifially
                // deals with the case of an intent picker/chooser being launched in the app
                // flow to redirect to an activity picked by the user, where we want the final
                // activity to consider it to have been launched by the previous app activity.
                callingPackage &#61; sourceRecord.launchedFromPackage;
            }
        }
  
        if (err &#61;&#61; ActivityManager.START_SUCCESS &amp;&amp; intent.getComponent() &#61;&#61; null) {
            //从intent中无法找到相应的component
            // We couldn&#39;t find a class that can handle the given Intent.
            // That&#39;s the end of that!
            err &#61; ActivityManager.START_INTENT_NOT_RESOLVED;
        }

        if (err &#61;&#61; ActivityManager.START_SUCCESS &amp;&amp; aInfo &#61;&#61; null) {
            //从intent中无法找到相应的ActivityInfo
            // We couldn&#39;t find the specific class specified in the Intent.
            // Also the end of the line.
            err &#61; ActivityManager.START_CLASS_NOT_FOUND;
        }

        if (err &#61;&#61; ActivityManager.START_SUCCESS &amp;&amp; sourceRecord !&#61; null
                &amp;&amp; sourceRecord.getTask().voiceSession !&#61; null) {
             //启动的activity是voice session一部分
            // If this activity is being launched as part of a voice session, we need
            // to ensure that it is safe to do so.  If the upcoming activity will also
            // be part of the voice session, we can only launch it if it has explicitly
            // said it supports the VOICE category, or it is a part of the calling app.
            if ((launchFlags &amp; FLAG_ACTIVITY_NEW_TASK) &#61;&#61; 0
                    &amp;&amp; sourceRecord.info.applicationInfo.uid !&#61; aInfo.applicationInfo.uid) {
                try {
                    intent.addCategory(Intent.CATEGORY_VOICE);
                    if (!mService.getPackageManager().activitySupportsIntent(
                            intent.getComponent(), intent, resolvedType)) {
                        Slog.w(TAG,
                                &#34;Activity being started in current voice task does not support voice: &#34;
                                        &#43; intent);
                        err &#61; ActivityManager.START_NOT_VOICE_COMPATIBLE;
                    }
                } catch (RemoteException e) {
                    Slog.w(TAG, &#34;Failure checking voice capabilities&#34;, e);
                    err &#61; ActivityManager.START_NOT_VOICE_COMPATIBLE;
                }
            }
        }

        if (err &#61;&#61; ActivityManager.START_SUCCESS &amp;&amp; voiceSession !&#61; null) {
            //启动是是voice session
            // If the caller is starting a new voice session, just make sure the target
            // is actually allowing it to run this way.
            try {
                if (!mService.getPackageManager().activitySupportsIntent(intent.getComponent(),
                        intent, resolvedType)) {
                    Slog.w(TAG,
                            &#34;Activity being started in new voice task does not support: &#34;
                                    &#43; intent);
                    err &#61; ActivityManager.START_NOT_VOICE_COMPATIBLE;
                }
            } catch (RemoteException e) {
                Slog.w(TAG, &#34;Failure checking voice capabilities&#34;, e);
                err &#61; ActivityManager.START_NOT_VOICE_COMPATIBLE;
            }
        }

        final ActivityStack resultStack &#61; resultRecord &#61;&#61; null ? null : resultRecord.getStack();
        //错误则返回
        if (err !&#61; START_SUCCESS) {
            if (resultRecord !&#61; null) {
                resultStack.sendActivityResultLocked(
                        -1, resultRecord, resultWho, requestCode, RESULT_CANCELED, null);
            }
            SafeActivityOptions.abort(options);
            return err;
        }

        //检查权限
        boolean abort &#61; !mSupervisor.checkStartAnyActivityPermission(intent, aInfo, resultWho,
                requestCode, callingPid, callingUid, callingPackage, ignoreTargetSecurity,
                inTask !&#61; null, callerApp, resultRecord, resultStack);
        abort |&#61; !mService.mIntentFirewall.checkStartActivity(intent, callingUid,
                callingPid, resolvedType, aInfo.applicationInfo);

        // Merge the two options bundles, while realCallerOptions takes precedence.
        ActivityOptions checkedOptions &#61; options !&#61; null
                ? options.getOptions(intent, aInfo, callerApp, mSupervisor)
                : null;
        if (allowPendingRemoteAnimationRegistryLookup) {
            checkedOptions &#61; mService.getActivityStartController()
                    .getPendingRemoteAnimationRegistry()
                    .overrideOptionsIfNeeded(callingPackage, checkedOptions);
        }
        //ActivityController不为空的情况&#xff0c;比如monkey测试过程
        if (mService.mController !&#61; null) {
            try {
                // The Intent we give to the watcher has the extra data
                // stripped off, since it can contain private information.
                Intent watchIntent &#61; intent.cloneFilter();
                abort |&#61; !mService.mController.activityStarting(watchIntent,
                        aInfo.applicationInfo.packageName);
            } catch (RemoteException e) {
                mService.mController &#61; null;
            }
        }

        mInterceptor.setStates(userId, realCallingPid, realCallingUid, startFlags, callingPackage);
        if (mInterceptor.intercept(intent, rInfo, aInfo, resolvedType, inTask, callingPid,
                callingUid, checkedOptions)) {
            // activity被拦截
            // activity start was intercepted, e.g. because the target user is currently in quiet
            // mode (turn off work) or the target application is suspended
            intent &#61; mInterceptor.mIntent;
            rInfo &#61; mInterceptor.mRInfo;
            aInfo &#61; mInterceptor.mAInfo;
            resolvedType &#61; mInterceptor.mResolvedType;
            inTask &#61; mInterceptor.mInTask;
            callingPid &#61; mInterceptor.mCallingPid;
            callingUid &#61; mInterceptor.mCallingUid;
            checkedOptions &#61; mInterceptor.mActivityOptions;
        }

        //终止则返回
        if (abort) {
            if (resultRecord !&#61; null) {
                resultStack.sendActivityResultLocked(-1, resultRecord, resultWho, requestCode,
                        RESULT_CANCELED, null);
            }
            // We pretend to the caller that it was really started, but
            // they will just get a cancel result.
            ActivityOptions.abort(checkedOptions);
            return START_ABORTED;
        }
        //如果需要再检查权限&#xff0c;则启动检查activity
        // If permissions need a review before any of the app components can run, we
        // launch the review activity and pass a pending intent to start the activity
        // we are to launching now after the review is completed.
        if (mService.mPermissionReviewRequired &amp;&amp; aInfo !&#61; null) {
            if (mService.getPackageManagerInternalLocked().isPermissionsReviewRequired(
                    aInfo.packageName, userId)) {
                IIntentSender target &#61; mService.getIntentSenderLocked(
                        ActivityManager.INTENT_SENDER_ACTIVITY, callingPackage,
                        callingUid, userId, null, null, 0, new Intent[]{intent},
                        new String[]{resolvedType}, PendingIntent.FLAG_CANCEL_CURRENT
                                | PendingIntent.FLAG_ONE_SHOT, null);

                final int flags &#61; intent.getFlags();
                Intent newIntent &#61; new Intent(Intent.ACTION_REVIEW_PERMISSIONS);
                newIntent.setFlags(flags
                        | Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS);
                newIntent.putExtra(Intent.EXTRA_PACKAGE_NAME, aInfo.packageName);
                newIntent.putExtra(Intent.EXTRA_INTENT, new IntentSender(target));
                if (resultRecord !&#61; null) {
                    newIntent.putExtra(Intent.EXTRA_RESULT_NEEDED, true);
                }
                intent &#61; newIntent;

                resolvedType &#61; null;
                callingUid &#61; realCallingUid;
                callingPid &#61; realCallingPid;

                rInfo &#61; mSupervisor.resolveIntent(intent, resolvedType, userId, 0,
                        computeResolveFilterUid(
                                callingUid, realCallingUid, mRequest.filterCallingUid));
                aInfo &#61; mSupervisor.resolveActivity(intent, rInfo, startFlags,
                        null /*profilerInfo*/);

                if (DEBUG_PERMISSIONS_REVIEW) {
                    Slog.i(TAG, &#34;START u&#34; &#43; userId &#43; &#34; {&#34; &#43; intent.toShortString(true, true,
                            true, false) &#43; &#34;} from uid &#34; &#43; callingUid &#43; &#34; on display &#34;
                            &#43; (mSupervisor.mFocusedStack &#61;&#61; null
                            ? DEFAULT_DISPLAY : mSupervisor.mFocusedStack.mDisplayId));
                }
            }
        }

        // If we have an ephemeral app, abort the process of launching the resolved intent.
        // Instead, launch the ephemeral installer. Once the installer is finished, it
        // starts either the intent we resolved here [on install error] or the ephemeral
        // app [on install success].
        if (rInfo !&#61; null &amp;&amp; rInfo.auxiliaryInfo !&#61; null) {
            intent &#61; createLaunchIntent(rInfo.auxiliaryInfo, ephemeralIntent,
                    callingPackage, verificationBundle, resolvedType, userId);
            resolvedType &#61; null;
            callingUid &#61; realCallingUid;
            callingPid &#61; realCallingPid;

            aInfo &#61; mSupervisor.resolveActivity(intent, rInfo, startFlags, null /*profilerInfo*/);
        }

        //创建activity记录对象
        ActivityRecord r &#61; new ActivityRecord(mService, callerApp, callingPid, callingUid,
                callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
                resultRecord, resultWho, requestCode, componentSpecified, voiceSession !&#61; null,
                mSupervisor, checkedOptions, sourceRecord);
        if (outActivity !&#61; null) {
            outActivity[0] &#61; r;
        }

        if (r.appTimeTracker &#61;&#61; null &amp;&amp; sourceRecord !&#61; null) {
            // If the caller didn&#39;t specify an explicit time tracker, we want to continue
            // tracking under any it has.
            r.appTimeTracker &#61; sourceRecord.appTimeTracker;
        }

        final ActivityStack stack &#61; mSupervisor.mFocusedStack;

       
        // If we are starting an activity that is not from the same uid as the currently resumed
        // one, check whether app switches are allowed.
        if (voiceSession &#61;&#61; null &amp;&amp; (stack.getResumedActivity() &#61;&#61; null
                || stack.getResumedActivity().info.applicationInfo.uid !&#61; realCallingUid)) {
            //如果前台stack还没有resume状态的activity&#xff0c;则检查app是否允许切换&#xff0c;见2.8.1
            if (!mService.checkAppSwitchAllowedLocked(callingPid, callingUid,
                    realCallingPid, realCallingUid, &#34;Activity start&#34;)) {
                 //如果不允许切换&#xff0c;则把要启动的activity添加到PendingActivity&#xff0c;并且返回
                mController.addPendingActivityLaunch(new PendingActivityLaunch(r,
                        sourceRecord, startFlags, stack, callerApp));
                ActivityOptions.abort(checkedOptions);
                return ActivityManager.START_SWITCHES_CANCELED;
            }
        }
         
        if (mService.mDidAppSwitch) {
            //从第一次app切换到第二次允许app,允许切换时间设置为0&#xff0c;则表示可以任意切换app
            // This is the second allowed switch since we stopped switches,
            // so now just generally allow switches.  Use case: user presses
            // home (switches disabled, switch to home, mDidAppSwitch now true);
            // user taps a home icon (coming from home so allowed, we hit here
            // and now allow anyone to switch again).
            mService.mAppSwitchesAllowedTime &#61; 0;
        } else {
            mService.mDidAppSwitch &#61; true;
        }

        //处理PendingActivity的启动&#xff0c;由于app switch禁用从而被hold的等待的activity&#xff0c;见2.8.2
        mController.doPendingActivityLaunches(false);

        maybeLogActivityStart(callingUid, callingPackage, realCallingUid, intent, callerApp, r,
                originatingPendingIntent);
        //再走startActivity
        return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
                true /* doResume */, checkedOptions, inTask, outActivity);
    }
</code></pre> 
<p>在上面这三个返回值表示启动activity失败</p> 
<p>START_INTENT_NOT_RESOLVED&#xff1a;从intent中无法找到相应的component<br /> START_CLASS_NOT_FOUND &#xff1a;从intent中无法找到相应的ActivityInfo<br /> START_NOT_VOICE_COMPATIBLE :不支持voice task</p> 
<pre><code>private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
                IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
                ActivityRecord[] outActivity) {
        int result &#61; START_CANCELED;
        try {
            mService.mWindowManager.deferSurfaceLayout();
            //见2.9节
            result &#61; startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, outActivity);
        } finally {
            //不能启动则取消task关联
            // If we are not able to proceed, disassociate the activity from the task. Leaving an
            // activity in an incomplete state can lead to issues, such as performing operations
            // without a window container.
            final ActivityStack stack &#61; mStartActivity.getStack();
            if (!ActivityManager.isStartResultSuccessful(result) &amp;&amp; stack !&#61; null) {
                stack.finishActivityLocked(mStartActivity, RESULT_CANCELED,
                        null /* intentResultData */, &#34;startActivity&#34;, true /* oomAdj */);
            }
            mService.mWindowManager.continueSurfaceLayout();
        }
        postStartActivityProcessing(r, result, mTargetStack);
        return result;
    }
</code></pre> 
<h4><a id="281_AMScheckAppSwitchAllowedLocked_1027"></a>2.8.1 AMS.checkAppSwitchAllowedLocked</h4> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code>  boolean checkAppSwitchAllowedLocked(int sourcePid, int sourceUid,
            int callingPid, int callingUid, String name) {
        if (mAppSwitchesAllowedTime &lt; SystemClock.uptimeMillis()) {
            return true;
        }
        if (mRecentTasks.isCallerRecents(sourceUid)) {
            return true;
        }
        int perm &#61; checkComponentPermission(STOP_APP_SWITCHES, sourcePid, sourceUid, -1, true);
        if (perm &#61;&#61; PackageManager.PERMISSION_GRANTED) {
            return true;
        }
        if (checkAllowAppSwitchUid(sourceUid)) {
            return true;
        }
        // If the actual IPC caller is different from the logical source, then
        // also see if they are allowed to control app switches.
        if (callingUid !&#61; -1 &amp;&amp; callingUid !&#61; sourceUid) {
            perm &#61; checkComponentPermission(STOP_APP_SWITCHES, callingPid, callingUid, -1, true);
            if (perm &#61;&#61; PackageManager.PERMISSION_GRANTED) {
                return true;
            }
            if (checkAllowAppSwitchUid(callingUid)) {
                return true;
            }
        }
        Slog.w(TAG, name &#43; &#34; request from &#34; &#43; sourceUid &#43; &#34; stopped&#34;);
        return false;
    }
</code></pre> 
<p>当mAppSwitchesAllowedTime时间小于当前时间或者具有STOP_APP_SWITCHES的权限&#xff0c;则允许app发生切换操作。 其中mAppSwitchesAllowedTime在AMS.stopAppSwitches的过程中会设置&#xff0c; mAppSwitchesAllowedTime &#61; SystemClock.uptimeMillis()&#43;APP_SWITCH_DELAY_TIME(&#61;5s); 禁止app切换的timeout时间为5s。</p> 
<p>当发送5s超时或者执行ASM.resumeAppSwitches过程会将mAppSwitchesAllowedTime 设置为0&#xff0c;都会开启允许app执行切换的操作。禁止app切换的操作&#xff0c;对于同一个app是不受影响的&#xff0c;可查看AMS.checkComponentPermission</p> 
<h4><a id="281_ASCdoPendingActivityLaunches_1067"></a>2.8.1 ASC.doPendingActivityLaunches</h4> 
<p>[-&gt;ActivityStartController.java]</p> 
<pre><code>  void doPendingActivityLaunches(boolean doResume) {
        while (!mPendingActivityLaunches.isEmpty()) {
            final PendingActivityLaunch pal &#61; mPendingActivityLaunches.remove(0);
            final boolean resume &#61; doResume &amp;&amp; mPendingActivityLaunches.isEmpty();
            final ActivityStarter starter &#61; obtainStarter(null /* intent */,
                    &#34;pendingActivityLaunch&#34;);
            try {
                starter.startResolvedActivity(pal.r, pal.sourceRecord, null, null, pal.startFlags,
                        resume, null, null, null /* outRecords */);
            } catch (Exception e) {
                Slog.e(TAG, &#34;Exception during pending activity launch pal&#61;&#34; &#43; pal, e);
                pal.sendErrorResult(e.getMessage());
            }
        }
    }
</code></pre> 
<p>mPendingActivityLaunches记录所有将要启动的Activity&#xff0c;由于在startActivity过程中时app切换功能被禁止&#xff0c;也就是不运行切换的Activity&#xff0c;就会将该Activity加入到mPendingActivityLaunches队列&#xff0c;该队列执行完doPendingActivityLaunches会清空。启动doPendingActivityLaunches的所有Activity&#xff0c;由于doResume&#61;false&#xff0c;那么activity不会进入resume&#xff0c;而是设置delayedResume &#61; true,延迟resume。</p> 
<h3><a id="29__ASstartActivityUnchecked_1091"></a>2.9 AS.startActivityUnchecked</h3> 
<p>[-&gt;ActivityStarter.java]</p> 
<pre><code>  //r是本次要启动的activity&#xff0c;sourceRecord是调用者
  // Note: This method should only be called from {&#64;link startActivity}.
    private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity) {
        //设置初始化状态&#xff0c;见2.9.1
        setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
                voiceInteractor);
        //确定启动taskflag&#xff0c;见2.9.2
        computeLaunchingTaskFlags();
        //确定调用者栈&#xff0c;见2.9.3
        computeSourceStack();

        mIntent.setFlags(mLaunchFlags);

        //得到可用的ActivityRecord
        ActivityRecord reusedActivity &#61; getReusableIntentActivity();

        int preferredWindowingMode &#61; WINDOWING_MODE_UNDEFINED;
        int preferredLaunchDisplayId &#61; DEFAULT_DISPLAY;
        if (mOptions !&#61; null) {
            preferredWindowingMode &#61; mOptions.getLaunchWindowingMode();
            preferredLaunchDisplayId &#61; mOptions.getLaunchDisplayId();
        }

        // windowing mode and preferred launch display values from {&#64;link LaunchParams} take
        // priority over those specified in {&#64;link ActivityOptions}.
        if (!mLaunchParams.isEmpty()) {
            if (mLaunchParams.hasPreferredDisplay()) {
                preferredLaunchDisplayId &#61; mLaunchParams.mPreferredDisplayId;
            }

            if (mLaunchParams.hasWindowingMode()) {
                preferredWindowingMode &#61; mLaunchParams.mWindowingMode;
            }
        }

        if (reusedActivity !&#61; null) {
            //LockTask mode 且设置了NEW_TASK and CLEAR_TASK则返回
            // When the flags NEW_TASK and CLEAR_TASK are set, then the task gets reused but
            // still needs to be a lock task mode violation since the task gets cleared out and
            // the device would otherwise leave the locked task.
            if (mService.getLockTaskController().isLockTaskModeViolation(reusedActivity.getTask(),
                    (mLaunchFlags &amp; (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))
                            &#61;&#61; (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))) {
                Slog.e(TAG, &#34;startActivityUnchecked: Attempt to violate Lock Task Mode&#34;);
                return START_RETURN_LOCK_TASK_MODE_VIOLATION;
            }

            // True if we are clearing top and resetting of a standard (default) launch mode
            // ({&#64;code LAUNCH_MULTIPLE}) activity. The existing activity will be finished.
            final boolean clearTopAndResetStandardLaunchMode &#61;
                    (mLaunchFlags &amp; (FLAG_ACTIVITY_CLEAR_TOP | FLAG_ACTIVITY_RESET_TASK_IF_NEEDED))
                            &#61;&#61; (FLAG_ACTIVITY_CLEAR_TOP | FLAG_ACTIVITY_RESET_TASK_IF_NEEDED)
                    &amp;&amp; mLaunchMode &#61;&#61; LAUNCH_MULTIPLE;

            //如果启动的activity没有管理task&#xff0c;则用存在activity的task
            // If mStartActivity does not have a task associated with it, associate it with the
            // reused activity&#39;s task. Do not do so if we&#39;re clearing top and resetting for a
            // standard launchMode activity.
            if (mStartActivity.getTask() &#61;&#61; null &amp;&amp; !clearTopAndResetStandardLaunchMode) {
                mStartActivity.setTask(reusedActivity.getTask());
            }

            if (reusedActivity.getTask().intent &#61;&#61; null) {
                //设置mStartActivity
                // This task was started because of movement of the activity based on affinity...
                // Now that we are actually launching it, we can assign the base intent.
                reusedActivity.getTask().setIntent(mStartActivity);
            }

            // This code path leads to delivering a new intent, we want to make sure we schedule it
            // as the first operation, in case the activity will be resumed as a result of later
            // operations.
            if ((mLaunchFlags &amp; FLAG_ACTIVITY_CLEAR_TOP) !&#61; 0
                    || isDocumentLaunchesIntoExisting(mLaunchFlags)
                    || isLaunchModeOneOf(LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK)) {
                final TaskRecord task &#61; reusedActivity.getTask();
                //LAUNCH_SINGLE_INSTANCE,LAUNCH_SINGLE_TASK模式下&#xff0c;栈移除所有的activity
                // In this situation we want to remove all activities from the task up to the one
                // being started. In most cases this means we are resetting the task to its initial
                // state.
                final ActivityRecord top &#61; task.performClearTaskForReuseLocked(mStartActivity,
                        mLaunchFlags);

                // The above code can remove {&#64;code reusedActivity} from the task, leading to the
                // the {&#64;code ActivityRecord} removing its reference to the {&#64;code TaskRecord}. The
                // task reference is needed in the call below to
                // {&#64;link setTargetStackAndMoveToFrontIfNeeded}.
                if (reusedActivity.getTask() &#61;&#61; null) {
                    reusedActivity.setTask(task);
                }

                if (top !&#61; null) {
                    //在前台
                    if (top.frontOfTask) {
                        // Activity aliases may mean we use different intents for the top activity,
                        // so make sure the task now has the identity of the new intent.
                        top.getTask().setIntent(mStartActivity);
                    }
                    deliverNewIntent(top);
                }
            }

            mSupervisor.sendPowerHintForLaunchStartIfNeeded(false /* forceSend */, reusedActivity);

            reusedActivity &#61; setTargetStackAndMoveToFrontIfNeeded(reusedActivity);

            final ActivityRecord outResult &#61;
                    outActivity !&#61; null &amp;&amp; outActivity.length &gt; 0 ? outActivity[0] : null;

            // When there is a reused activity and the current result is a trampoline activity,
            // set the reused activity as the result.
            if (outResult !&#61; null &amp;&amp; (outResult.finishing || outResult.noDisplay)) {
                outActivity[0] &#61; reusedActivity;
            }

            if ((mStartFlags &amp; START_FLAG_ONLY_IF_NEEDED) !&#61; 0) {
                // We don&#39;t need to start a new activity, and the client said not to do anything
                // if that is the case, so this is it!  And for paranoia, make sure we have
                // correctly resumed the top activity.
                resumeTargetStackIfNeeded();
                return START_RETURN_INTENT_TO_CALLER;
            }

            if (reusedActivity !&#61; null) {
                setTaskFromIntentActivity(reusedActivity);

                if (!mAddingToTask &amp;&amp; mReuseTask &#61;&#61; null) {
                    // We didn&#39;t do anything...  but it was needed (a.k.a., client don&#39;t use that
                    // intent!)  And for paranoia, make sure we have correctly resumed the top activity.

                    resumeTargetStackIfNeeded();
                    if (outActivity !&#61; null &amp;&amp; outActivity.length &gt; 0) {
                        outActivity[0] &#61; reusedActivity;
                    }

                    return mMovedToFront ? START_TASK_TO_FRONT : START_DELIVERED_TO_TOP;
                }
            }
        }
        
        //启动的activity没有包名&#xff0c;直接返回
        if (mStartActivity.packageName &#61;&#61; null) {
            final ActivityStack sourceStack &#61; mStartActivity.resultTo !&#61; null
                    ? mStartActivity.resultTo.getStack() : null;
            if (sourceStack !&#61; null) {
                sourceStack.sendActivityResultLocked(-1 /* callingUid */, mStartActivity.resultTo,
                        mStartActivity.resultWho, mStartActivity.requestCode, RESULT_CANCELED,
                        null /* data */);
            }
            ActivityOptions.abort(mOptions);
            return START_CLASS_NOT_FOUND;
        }

        // If the activity being launched is the same as the one currently at the top, then
        // we need to check if it should only be launched once.
        final ActivityStack topStack &#61; mSupervisor.mFocusedStack;
        final ActivityRecord topFocused &#61; topStack.getTopActivity();
        final ActivityRecord top &#61; topStack.topRunningNonDelayedActivityLocked(mNotTop);
        final boolean dontStart &#61; top !&#61; null &amp;&amp; mStartActivity.resultTo &#61;&#61; null
                &amp;&amp; top.realActivity.equals(mStartActivity.realActivity)
                &amp;&amp; top.userId &#61;&#61; mStartActivity.userId
                &amp;&amp; top.app !&#61; null &amp;&amp; top.app.thread !&#61; null
                &amp;&amp; ((mLaunchFlags &amp; FLAG_ACTIVITY_SINGLE_TOP) !&#61; 0
                || isLaunchModeOneOf(LAUNCH_SINGLE_TOP, LAUNCH_SINGLE_TASK));        
        if (dontStart) {
            // For paranoia, make sure we have correctly resumed the top activity.
            topStack.mLastPausedActivity &#61; null;
            if (mDoResume) {
                mSupervisor.resumeFocusedStackTopActivityLocked();
            }
            ActivityOptions.abort(mOptions);
            if ((mStartFlags &amp; START_FLAG_ONLY_IF_NEEDED) !&#61; 0) {
                // We don&#39;t need to start a new activity, and the client said not to do
                // anything if that is the case, so this is it!
                return START_RETURN_INTENT_TO_CALLER;
            }
            //触发onNewIntent
            deliverNewIntent(top);

            // Don&#39;t use mStartActivity.task to show the toast. We&#39;re not starting a new activity
            // but reusing &#39;top&#39;. Fields in mStartActivity may not be fully initialized.
            mSupervisor.handleNonResizableTaskIfNeeded(top.getTask(), preferredWindowingMode,
                    preferredLaunchDisplayId, topStack);

            return START_DELIVERED_TO_TOP;
        }

        boolean newTask &#61; false;
        final TaskRecord taskToAffiliate &#61; (mLaunchTaskBehind &amp;&amp; mSourceRecord !&#61; null)
                ? mSourceRecord.getTask() : null;

        // Should this be considered a new task?
        int result &#61; START_SUCCESS;
        if (mStartActivity.resultTo &#61;&#61; null &amp;&amp; mInTask &#61;&#61; null &amp;&amp; !mAddingToTask
                &amp;&amp; (mLaunchFlags &amp; FLAG_ACTIVITY_NEW_TASK) !&#61; 0) {
            newTask &#61; true;
            result &#61; setTaskFromReuseOrCreateNewTask(taskToAffiliate, topStack);
        } else if (mSourceRecord !&#61; null) {
            result &#61; setTaskFromSourceRecord();
        } else if (mInTask !&#61; null) {
            result &#61; setTaskFromInTask();
        } else {
            // This not being started from an existing activity, and not part of a new task...
            // just put it in the top task, though these days this case should never happen.
            setTaskToCurrentTopOrCreateNewTask();
        }
        if (result !&#61; START_SUCCESS) {
            return result;
        }

        mService.grantUriPermissionFromIntentLocked(mCallingUid, mStartActivity.packageName,
                mIntent, mStartActivity.getUriPermissionsLocked(), mStartActivity.userId);
        mService.grantEphemeralAccessLocked(mStartActivity.userId, mIntent,
                mStartActivity.appInfo.uid, UserHandle.getAppId(mCallingUid));
        if (newTask) {
            EventLog.writeEvent(EventLogTags.AM_CREATE_TASK, mStartActivity.userId,
                    mStartActivity.getTask().taskId);
        }
        ActivityStack.logStartActivity(
                EventLogTags.AM_CREATE_ACTIVITY, mStartActivity, mStartActivity.getTask());
        mTargetStack.mLastPausedActivity &#61; null;

        mSupervisor.sendPowerHintForLaunchStartIfNeeded(false /* forceSend */, mStartActivity);
        //见2.10节
        mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransition,
                mOptions);
        if (mDoResume) {
            final ActivityRecord topTaskActivity &#61;
                    mStartActivity.getTask().topRunningActivityLocked();
            if (!mTargetStack.isFocusable()
                    || (topTaskActivity !&#61; null &amp;&amp; topTaskActivity.mTaskOverlay
                    &amp;&amp; mStartActivity !&#61; topTaskActivity)) {
                // 没有获取焦点&#xff0c;不能resume   
                // If the activity is not focusable, we can&#39;t resume it, but still would like to
                // make sure it becomes visible as it starts (this will also trigger entry
                // animation). An example of this are PIP activities.
                // Also, we don&#39;t want to resume activities in a task that currently has an overlay
                // as the starting activity just needs to be in the visible paused state until the
                // over is removed.
                mTargetStack.ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
                // Go ahead and tell window manager to execute app transition for this activity
                // since the app transition will not be triggered through the resume channel.
                mService.mWindowManager.executeAppTransition();
            } else {
                // If the target stack was not previously focusable (previous top running activity
                // on that stack was not visible) then any prior calls to move the stack to the
                // will not update the focused stack.  If starting the new activity now allows the
                // task stack to be focusable, then ensure that we now update the focused stack
                // accordingly.
                if (mTargetStack.isFocusable() &amp;&amp; !mSupervisor.isFocusedStack(mTargetStack)) {
                    mTargetStack.moveToFront(&#34;startActivityUnchecked&#34;);
                }
                //见2.11节
                mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                        mOptions);
            }
        } else if (mStartActivity !&#61; null) {
            mSupervisor.mRecentTasks.add(mStartActivity.getTask());
        }
        mSupervisor.updateUserStackLocked(mStartActivity.userId, mTargetStack);

        mSupervisor.handleNonResizableTaskIfNeeded(mStartActivity.getTask(), preferredWindowingMode,
                preferredLaunchDisplayId, mTargetStack);

        return START_SUCCESS;
    }

</code></pre> 
<p>找到或者创建新的Activity所属的Task对象&#xff0c;之后调用AS.startActivityLocked</p> 
<h4><a id="291_ASsetInitialState_1370"></a>2.9.1 AS.setInitialState</h4> 
<p>[-&gt;ActivityStarter.java]</p> 
<pre><code>private void setInitialState(ActivityRecord r, ActivityOptions options, TaskRecord inTask,
            boolean doResume, int startFlags, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor) {
        reset(false /* clearRequest */);

        mStartActivity &#61; r;
        mIntent &#61; r.intent;
        mOptions &#61; options;
        mCallingUid &#61; r.launchedFromUid;
        mSourceRecord &#61; sourceRecord;
        mVoiceSession &#61; voiceSession;
        mVoiceInteractor &#61; voiceInteractor;

        mPreferredDisplayId &#61; getPreferedDisplayId(mSourceRecord, mStartActivity, options);

        mLaunchParams.reset();

        mSupervisor.getLaunchParamsController().calculate(inTask, null /*layout*/, r, sourceRecord,
                options, mLaunchParams);

        mLaunchMode &#61; r.launchMode;
        //当intent和Activity manifest存在冲突&#xff0c;则manifest优先
        mLaunchFlags &#61; adjustLaunchFlagsToDocumentMode(
                r, LAUNCH_SINGLE_INSTANCE &#61;&#61; mLaunchMode,
                LAUNCH_SINGLE_TASK &#61;&#61; mLaunchMode, mIntent.getFlags());
        mLaunchTaskBehind &#61; r.mLaunchTaskBehind
                &amp;&amp; !isLaunchModeOneOf(LAUNCH_SINGLE_TASK, LAUNCH_SINGLE_INSTANCE)
                &amp;&amp; (mLaunchFlags &amp; FLAG_ACTIVITY_NEW_DOCUMENT) !&#61; 0;

        sendNewTaskResultRequestIfNeeded();
   
        if ((mLaunchFlags &amp; FLAG_ACTIVITY_NEW_DOCUMENT) !&#61; 0 &amp;&amp; r.resultTo &#61;&#61; null) {
            mLaunchFlags |&#61; FLAG_ACTIVITY_NEW_TASK;
        }

        // If we are actually going to launch in to a new task, there are some cases where
        // we further want to do multiple task.
        if ((mLaunchFlags &amp; FLAG_ACTIVITY_NEW_TASK) !&#61; 0) {
            if (mLaunchTaskBehind
                    || r.info.documentLaunchMode &#61;&#61; DOCUMENT_LAUNCH_ALWAYS) {
                mLaunchFlags |&#61; FLAG_ACTIVITY_MULTIPLE_TASK;
            }
        }

        // We&#39;ll invoke onUserLeaving before onPause only if the launching
        // activity did not explicitly state that this is an automated launch.
        mSupervisor.mUserLeaving &#61; (mLaunchFlags &amp; FLAG_ACTIVITY_NO_USER_ACTION) &#61;&#61; 0;
        if (DEBUG_USER_LEAVING) Slog.v(TAG_USER_LEAVING,
                &#34;startActivity() &#61;&gt; mUserLeaving&#61;&#34; &#43; mSupervisor.mUserLeaving);
        //当本次不需要resume时&#xff0c;则设置为延迟resume的状态
        // If the caller has asked not to resume at this point, we make note
        // of this in the record so that we can skip it when trying to find
        // the top running activity.
        mDoResume &#61; doResume;
        if (!doResume || !r.okToShowLocked()) {
            r.delayedResume &#61; true;
            mDoResume &#61; false;
        }

        if (mOptions !&#61; null) {
            if (mOptions.getLaunchTaskId() !&#61; -1 &amp;&amp; mOptions.getTaskOverlay()) {
                r.mTaskOverlay &#61; true;
                if (!mOptions.canTaskOverlayResume()) {
                    final TaskRecord task &#61; mSupervisor.anyTaskForIdLocked(
                            mOptions.getLaunchTaskId());
                    final ActivityRecord top &#61; task !&#61; null ? task.getTopActivity() : null;
                    if (top !&#61; null &amp;&amp; !top.isState(RESUMED)) {

                        // The caller specifies that we&#39;d like to be avoided to be moved to the
                        // front, so be it!
                        mDoResume &#61; false;
                        mAvoidMoveToFront &#61; true;
                    }
                }
            } else if (mOptions.getAvoidMoveToFront()) {
                mDoResume &#61; false;
                mAvoidMoveToFront &#61; true;
            }
        }

        mNotTop &#61; (mLaunchFlags &amp; FLAG_ACTIVITY_PREVIOUS_IS_TOP) !&#61; 0 ? r : null;

        mInTask &#61; inTask;
        // In some flows in to this function, we retrieve the task record and hold on to it
        // without a lock before calling back in to here...  so the task at this point may
        // not actually be in recents.  Check for that, and if it isn&#39;t in recents just
        // consider it invalid.
        if (inTask !&#61; null &amp;&amp; !inTask.inRecents) {
            Slog.w(TAG, &#34;Starting activity in task not in recents: &#34; &#43; inTask);
            mInTask &#61; null;
        }

        mStartFlags &#61; startFlags;
        // If the onlyIfNeeded flag is set, then we can do this if the activity being launched
        // is the same as the one making the call...  or, as a special case, if we do not know
        // the caller then we count the current top activity as the caller.
        if ((startFlags &amp; START_FLAG_ONLY_IF_NEEDED) !&#61; 0) {
            ActivityRecord checkedCaller &#61; sourceRecord;
            if (checkedCaller &#61;&#61; null) {
                checkedCaller &#61; mSupervisor.mFocusedStack.topRunningNonDelayedActivityLocked(
                        mNotTop);
            }
            if (!checkedCaller.realActivity.equals(r.realActivity)) {
                //调用者与将要启动的activity不相同时今日该分支
                // Caller is not the same as launcher, so always needed.
                mStartFlags &amp;&#61; ~START_FLAG_ONLY_IF_NEEDED;
            }
        }

        mNoAnimation &#61; (mLaunchFlags &amp; FLAG_ACTIVITY_NO_ANIMATION) !&#61; 0;
    }
</code></pre> 
<h4><a id="292_AScomputeLaunchingTaskFlags_1488"></a>2.9.2 AS.computeLaunchingTaskFlags</h4> 
<p>[-&gt;ActivityStarter.java]</p> 
<pre><code>private void computeLaunchingTaskFlags() {
        //当调用者不是来自于activity&#xff0c;而是指定明确task的情况
        // If the caller is not coming from another activity, but has given us an explicit task into
        // which they would like us to launch the new activity, then let&#39;s see about doing that.
        if (mSourceRecord &#61;&#61; null &amp;&amp; mInTask !&#61; null &amp;&amp; mInTask.getStack() !&#61; null) {
            final Intent baseIntent &#61; mInTask.getBaseIntent();
            final ActivityRecord root &#61; mInTask.getRootActivity();
            if (baseIntent &#61;&#61; null) {
                ActivityOptions.abort(mOptions);
                throw new IllegalArgumentException(&#34;Launching into task without base intent: &#34;
                        &#43; mInTask);
            }

            // If this task is empty, then we are adding the first activity -- it
            // determines the root, and must be launching as a NEW_TASK.
            if (isLaunchModeOneOf(LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK)) {
                if (!baseIntent.getComponent().equals(mStartActivity.intent.getComponent())) {
                    ActivityOptions.abort(mOptions);
                    throw new IllegalArgumentException(&#34;Trying to launch singleInstance/Task &#34;
                            &#43; mStartActivity &#43; &#34; into different task &#34; &#43; mInTask);
                }
                if (root !&#61; null) {
                    ActivityOptions.abort(mOptions);
                    throw new IllegalArgumentException(&#34;Caller with mInTask &#34; &#43; mInTask
                            &#43; &#34; has root &#34; &#43; root &#43; &#34; but target is singleInstance/Task&#34;);
                }
            }

            // If task is empty, then adopt the interesting intent launch flags in to the
            // activity being started.
            if (root &#61;&#61; null) {
                final int flagsOfInterest &#61; FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_MULTIPLE_TASK
                        | FLAG_ACTIVITY_NEW_DOCUMENT | FLAG_ACTIVITY_RETAIN_IN_RECENTS;
                mLaunchFlags &#61; (mLaunchFlags &amp; ~flagsOfInterest)
                        | (baseIntent.getFlags() &amp; flagsOfInterest);
                mIntent.setFlags(mLaunchFlags);
                mInTask.setIntent(mStartActivity);
                mAddingToTask &#61; true;

                // If the task is not empty and the caller is asking to start it as the root of
                // a new task, then we don&#39;t actually want to start this on the task. We will
                // bring the task to the front, and possibly give it a new intent.
            } else if ((mLaunchFlags &amp; FLAG_ACTIVITY_NEW_TASK) !&#61; 0) {
                mAddingToTask &#61; false;

            } else {
                mAddingToTask &#61; true;
            }

            mReuseTask &#61; mInTask;
        } else {
            mInTask &#61; null;
            // Launch ResolverActivity in the source task, so that it stays in the task bounds
            // when in freeform workspace.
            // Also put noDisplay activities in the source task. These by itself can be placed
            // in any task/stack, however it could launch other activities like ResolverActivity,
            // and we want those to stay in the original task.
            if ((mStartActivity.isResolverActivity() || mStartActivity.noDisplay) &amp;&amp; mSourceRecord !&#61; null
                    &amp;&amp; mSourceRecord.inFreeformWindowingMode())  {
                mAddingToTask &#61; true;
            }
        }

        if (mInTask &#61;&#61; null) {
            if (mSourceRecord &#61;&#61; null) {
                //调用者不是Activity context,则强制创建新task
                // This activity is not being started from another...  in this
                // case we -always- start a new task.
                if ((mLaunchFlags &amp; FLAG_ACTIVITY_NEW_TASK) &#61;&#61; 0 &amp;&amp; mInTask &#61;&#61; null) {
                    Slog.w(TAG, &#34;startActivity called from non-Activity context; forcing &#34; &#43;
                            &#34;Intent.FLAG_ACTIVITY_NEW_TASK for: &#34; &#43; mIntent);
                    mLaunchFlags |&#61; FLAG_ACTIVITY_NEW_TASK;
                }
            } else if (mSourceRecord.launchMode &#61;&#61; LAUNCH_SINGLE_INSTANCE) {
                 //调用者启动模式是single instance&#xff0c;则创建新task
                // The original activity who is starting us is running as a single
                // instance...  this new activity it is starting must go on its
                // own task.
                mLaunchFlags |&#61; FLAG_ACTIVITY_NEW_TASK;
            } else if (isLaunchModeOneOf(LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK)) {
                //目标activity带有single instance或者single task则创建新的task
                // The activity being started is a single instance...  it always
                // gets launched into its own task.
                mLaunchFlags |&#61; FLAG_ACTIVITY_NEW_TASK;
            }
        }
    }
</code></pre> 
<h4><a id="293_AScomputeSourceStack_1582"></a>2.9.3 AS.computeSourceStack</h4> 
<p>[-&gt;ActivityStarter.java]</p> 
<pre><code> private void computeSourceStack() {
        if (mSourceRecord &#61;&#61; null) {
            mSourceStack &#61; null;
            return;
        }
        if (!mSourceRecord.finishing) {
           //当调用者activity不为空&#xff0c;且不在finishing状态&#xff0c;则其所在的栈赋于sourceStack
            mSourceStack &#61; mSourceRecord.getStack();
            return;
        }
        //当调用者处于finishing状态&#xff0c;则创建新的task
        // If the source is finishing, we can&#39;t further count it as our source. This is because the
        // task it is associated with may now be empty and on its way out, so we don&#39;t want to
        // blindly throw it in to that task.  Instead we will take the NEW_TASK flow and try to find
        // a task for it. But save the task information so it can be used when creating the new task.
        if ((mLaunchFlags &amp; FLAG_ACTIVITY_NEW_TASK) &#61;&#61; 0) {
            Slog.w(TAG, &#34;startActivity called from finishing &#34; &#43; mSourceRecord
                    &#43; &#34;; forcing &#34; &#43; &#34;Intent.FLAG_ACTIVITY_NEW_TASK for: &#34; &#43; mIntent);
            mLaunchFlags |&#61; FLAG_ACTIVITY_NEW_TASK;
            mNewTaskInfo &#61; mSourceRecord.info;

            // It is not guaranteed that the source record will have a task associated with it. For,
            // example, if this method is being called for processing a pending activity launch, it
            // is possible that the activity has been removed from the task after the launch was
            // enqueued.
            final TaskRecord sourceTask &#61; mSourceRecord.getTask();
            mNewTaskIntent &#61; sourceTask !&#61; null ? sourceTask.intent : null;
        }
        mSourceRecord &#61; null;
        mSourceStack &#61; null;
    }
</code></pre> 
<h4><a id="294_ASgetReusableIntentActivity_1620"></a>2.9.4 AS.getReusableIntentActivity</h4> 
<p>[-&gt;ActivityStarter.java]</p> 
<pre><code>/**
     * Decide whether the new activity should be inserted into an existing task. Returns null
     * if not or an ActivityRecord with the task into which the new activity should be added.
     */
    private ActivityRecord getReusableIntentActivity() {
        // We may want to try to place the new activity in to an existing task.  We always
        // do this if the target activity is singleTask or singleInstance; we will also do
        // this if NEW_TASK has been requested, and there is not an additional qualifier telling
        // us to still place it in a new task: multi task, always doc mode, or being asked to
        // launch this as a new task behind the current one.
        boolean putIntoExistingTask &#61; ((mLaunchFlags &amp; FLAG_ACTIVITY_NEW_TASK) !&#61; 0 &amp;&amp;
                (mLaunchFlags &amp; FLAG_ACTIVITY_MULTIPLE_TASK) &#61;&#61; 0)
                || isLaunchModeOneOf(LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK);
        // If bring to front is requested, and no result is requested and we have not been given
        // an explicit task to launch in to, and we can find a task that was started with this
        // same component, then instead of launching bring that one to the front.
        putIntoExistingTask &amp;&#61; mInTask &#61;&#61; null &amp;&amp; mStartActivity.resultTo &#61;&#61; null;
        ActivityRecord intentActivity &#61; null;
        if (mOptions !&#61; null &amp;&amp; mOptions.getLaunchTaskId() !&#61; -1) {
            final TaskRecord task &#61; mSupervisor.anyTaskForIdLocked(mOptions.getLaunchTaskId());
            intentActivity &#61; task !&#61; null ? task.getTopActivity() : null;
        } else if (putIntoExistingTask) {
            if (LAUNCH_SINGLE_INSTANCE &#61;&#61; mLaunchMode) {
                // There can be one and only one instance of single instance activity in the
                // history, and it is always in its own unique task, so we do a special search.
               intentActivity &#61; mSupervisor.findActivityLocked(mIntent, mStartActivity.info,
                       mStartActivity.isActivityTypeHome());
            } else if ((mLaunchFlags &amp; FLAG_ACTIVITY_LAUNCH_ADJACENT) !&#61; 0) {
                // For the launch adjacent case we only want to put the activity in an existing
                // task if the activity already exists in the history.
                intentActivity &#61; mSupervisor.findActivityLocked(mIntent, mStartActivity.info,
                        !(LAUNCH_SINGLE_TASK &#61;&#61; mLaunchMode));
            } else {
                // Otherwise find the best task to put the activity in.
                intentActivity &#61; mSupervisor.findTaskLocked(mStartActivity, mPreferredDisplayId);
            }
        }
        return intentActivity;
    }
</code></pre> 
<p>上面是根据不同的启动模式&#xff0c;来获取ActivityRecord信息&#xff0c;来决定将要启动的activity所在的栈。</p> 
<h4><a id="295_Launch_Mode_1668"></a>2.9.5 Launch Mode</h4> 
<p>AcitivityInfo.java定义了四类Launch Mode:</p> 
<pre><code>    /**
     * Constant corresponding to &lt;code&gt;standard&lt;/code&gt; in
     * the {&#64;link android.R.attr#launchMode} attribute.
     */
    //每次启动新Activity&#xff0c;都会创建新的Activity&#xff0c;这是最常见标准情形
    public static final int LAUNCH_MULTIPLE &#61; 0;
    /**
     * Constant corresponding to &lt;code&gt;singleTop&lt;/code&gt; in
     * the {&#64;link android.R.attr#launchMode} attribute.
     */
    //当启动新Activity&#xff0c;如果栈顶存在相同Activity&#xff0c;则不会创建新的Activity  
    public static final int LAUNCH_SINGLE_TOP &#61; 1;
    /**
     * Constant corresponding to &lt;code&gt;singleTask&lt;/code&gt; in
     * the {&#64;link android.R.attr#launchMode} attribute.
     */
    //当启动Activity&#xff0c;在栈中存在相同Activity&#xff0c;则不会创建新的Activity
    //而是移除该Activity之上的所有Activity
    public static final int LAUNCH_SINGLE_TASK &#61; 2;
    /**
     * Constant corresponding to &lt;code&gt;singleInstance&lt;/code&gt; in
     * the {&#64;link android.R.attr#launchMode} attribute.
     */
     //每个Task栈只有一个Activity
    public static final int LAUNCH_SINGLE_INSTANCE &#61; 3;
</code></pre> 
<p>常见的flag含义</p> 
<p>FLAG_ACTIVITY_NEW_TASK</p> 
<p>将新Activity放入新启动的task。</p> 
<p>FLAG_ACTIVITY_CLEAR_TASK</p> 
<p>启动Activity时&#xff0c;将目标Activity关联的task清除&#xff0c;再启动新task,将该Activity放入该Task。这个flag一般配置FLAG_ACTIVITY_NEW_TASK使用。</p> 
<p>FLAG_ACTIVITY_CLEAR_TOP</p> 
<p>启动非栈顶Activity时&#xff0c;先清除该Activity之上的Activity. 例如Task已有A,B,C,D&#xff0c;启动A,则需要先清除B,C,D&#xff0c;类似SingleTop。</p> 
<h3><a id="210_ASstartActivityLocked_1714"></a>2.10 AS.startActivityLocked</h3> 
<p>[-&gt;ActivityStack.java]</p> 
<pre><code> void startActivityLocked(ActivityRecord r, ActivityRecord focusedTopActivity,
            boolean newTask, boolean keepCurTransition, ActivityOptions options) {
        TaskRecord rTask &#61; r.getTask();
        final int taskId &#61; rTask.taskId;
        // mLaunchTaskBehind tasks get placed at the back of the task stack.
        if (!r.mLaunchTaskBehind &amp;&amp; (taskForIdLocked(taskId) &#61;&#61; null || newTask)) {
            // Last activity in task had been removed or ActivityManagerService is reusing task.
            // Insert or replace.
            // Might not even be in.
            //task中上一个activity被移除&#xff0c;或者ams重用task,则将该task移到顶部
            insertTaskAtTop(rTask, r);
        }
        TaskRecord task &#61; null;
        if (!newTask) {
            // If starting in an existing task, find where that is...
            boolean startIt &#61; true;
            for (int taskNdx &#61; mTaskHistory.size() - 1; taskNdx &gt;&#61; 0; --taskNdx) {
                task &#61; mTaskHistory.get(taskNdx);
                if (task.getTopActivity() &#61;&#61; null) {
                    // All activities in task are finishing.
                    //该task所有activity都finishing
                    continue;
                }
                if (task &#61;&#61; rTask) {
                    // Here it is!  Now, if this is not yet visible to the
                    // user, then just add it without starting; it will
                    // get started when the user navigates back to it.
                    if (!startIt) {
                        if (DEBUG_ADD_REMOVE) Slog.i(TAG, &#34;Adding activity &#34; &#43; r &#43; &#34; to task &#34;
                                &#43; task, new RuntimeException(&#34;here&#34;).fillInStackTrace());
                        r.createWindowContainer();
                        ActivityOptions.abort(options);
                        return;
                    }
                    break;
                } else if (task.numFullscreen &gt; 0) {
                    startIt &#61; false;
                }
            }
        }

        // Place a new activity at top of stack, so it is next to interact with the user.

        // If we are not placing the new activity frontmost, we do not want to deliver the
        // onUserLeaving callback to the actual frontmost activity
        final TaskRecord activityTask &#61; r.getTask();
        if (task &#61;&#61; activityTask &amp;&amp; mTaskHistory.indexOf(task) !&#61; (mTaskHistory.size() - 1)) {
            mStackSupervisor.mUserLeaving &#61; false;
            if (DEBUG_USER_LEAVING) Slog.v(TAG_USER_LEAVING,
                    &#34;startActivity() behind front, mUserLeaving&#61;false&#34;);
        }

        task &#61; activityTask;

        // Slot the activity into the history stack and proceed
        if (DEBUG_ADD_REMOVE) Slog.i(TAG, &#34;Adding activity &#34; &#43; r &#43; &#34; to stack to task &#34; &#43; task,
                new RuntimeException(&#34;here&#34;).fillInStackTrace());
        // TODO: Need to investigate if it is okay for the controller to already be created by the
        // time we get to this point. I think it is, but need to double check.
        // Use test in b/34179495 to trace the call path.
        if (r.getWindowContainerController() &#61;&#61; null) {
            r.createWindowContainer();
        }
        task.setFrontOfTask();
       //当切换到新的task或者下一个activity进程目前没有运行
        if (!isHomeOrRecentsStack() || numActivities() &gt; 0) {
            if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION,
                    &#34;Prepare open transition: starting &#34; &#43; r);
            if ((r.intent.getFlags() &amp; Intent.FLAG_ACTIVITY_NO_ANIMATION) !&#61; 0) {
                mWindowManager.prepareAppTransition(TRANSIT_NONE, keepCurTransition);
                mStackSupervisor.mNoAnimActivities.add(r);
            } else {
                int transit &#61; TRANSIT_ACTIVITY_OPEN;
                if (newTask) {
                    if (r.mLaunchTaskBehind) {
                        transit &#61; TRANSIT_TASK_OPEN_BEHIND;
                    } else {
                        // If a new task is being launched, then mark the existing top activity as
                        // supporting picture-in-picture while pausing only if the starting activity
                        // would not be considered an overlay on top of the current activity
                        // (eg. not fullscreen, or the assistant)
                        if (canEnterPipOnTaskSwitch(focusedTopActivity,
                                null /* toFrontTask */, r, options)) {
                            focusedTopActivity.supportsEnterPipOnTaskSwitch &#61; true;
                        }
                        transit &#61; TRANSIT_TASK_OPEN;
                    }
                }
                mWindowManager.prepareAppTransition(transit, keepCurTransition);
                mStackSupervisor.mNoAnimActivities.remove(r);
            }
            boolean doShow &#61; true;
            if (newTask) {
                // Even though this activity is starting fresh, we still need
                // to reset it to make sure we apply affinities to move any
                // existing activities from other tasks in to it.
                // If the caller has requested that the target task be
                // reset, then do so.
                if ((r.intent.getFlags() &amp; Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) !&#61; 0) {
                    resetTaskIfNeededLocked(r, r);
                    doShow &#61; topRunningNonDelayedActivityLocked(null) &#61;&#61; r;
                }
            } else if (options !&#61; null &amp;&amp; options.getAnimationType()
                    &#61;&#61; ActivityOptions.ANIM_SCENE_TRANSITION) {
                doShow &#61; false;
            }
            if (r.mLaunchTaskBehind) {
                // Don&#39;t do a starting window for mLaunchTaskBehind. More importantly make sure we
                // tell WindowManager that r is visible even though it is at the back of the stack.
                r.setVisibility(true);
                ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
            } else if (SHOW_APP_STARTING_PREVIEW &amp;&amp; doShow) {
                // Figure out if we are transitioning from another activity that is
                // &#34;has the same starting icon&#34; as the next one.  This allows the
                // window manager to keep the previous window it had previously
                // created, if it still had one.
                TaskRecord prevTask &#61; r.getTask();
                ActivityRecord prev &#61; prevTask.topRunningActivityWithStartingWindowLocked();
                if (prev !&#61; null) {
                   //当前activity属于不同的task
                    // We don&#39;t want to reuse the previous starting preview if:
                    // (1) The current activity is in a different task.
                    if (prev.getTask() !&#61; prevTask) {
                        prev &#61; null;
                    }
                    //当前activity已经display
                    // (2) The current activity is already displayed.
                    else if (prev.nowVisible) {
                        prev &#61; null;
                    }
                }
                r.showStartingWindow(prev, newTask, isTaskSwitch(r, focusedTopActivity));
            }
        } else {
            // If this is the first activity, don&#39;t do any fancy animations,
            // because there is nothing for it to animate on top of.
            ActivityOptions.abort(options);
        }
    }
</code></pre> 
<h3><a id="211__ASSresumeFocusedStackTopActivityLocked_1860"></a>2.11 ASS.resumeFocusedStackTopActivityLocked</h3> 
<p>[-&gt;ActivityStackSupervisor.java]</p> 
<pre><code> boolean resumeFocusedStackTopActivityLocked(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {

        if (!readyToResume()) {
            return false;
        }

        if (targetStack !&#61; null &amp;&amp; isFocusedStack(targetStack)) {
            return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
        }

        final ActivityRecord r &#61; mFocusedStack.topRunningActivityLocked();
        if (r &#61;&#61; null || !r.isState(RESUMED)) {
            mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
        } else if (r.isState(RESUMED)) {
            // Kick off any lingering app transitions form the MoveTaskToFront operation.
            mFocusedStack.executeAppTransition(targetOptions);
        }

        return false;
    }
</code></pre> 
<h3><a id="212_ASresumeTopActivityUncheckedLocked_1888"></a>2.12 AS.resumeTopActivityUncheckedLocked</h3> 
<p>[-&gt;ActivityStack.java]</p> 
<pre><code>  boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        //防止递归启动
        if (mStackSupervisor.inResumeTopActivity) {
            // Don&#39;t even start recursing.
            return false;
        }

        boolean result &#61; false;
        try {
            // Protect against recursion.
            mStackSupervisor.inResumeTopActivity &#61; true;
            //见2.13节
            result &#61; resumeTopActivityInnerLocked(prev, options);

            // When resuming the top activity, it may be necessary to pause the top activity (for
            // example, returning to the lock screen. We suppress the normal pause logic in
            // {&#64;link #resumeTopActivityUncheckedLocked}, since the top activity is resumed at the
            // end. We call the {&#64;link ActivityStackSupervisor#checkReadyForSleepLocked} again here
            // to ensure any necessary pause logic occurs. In the case where the Activity will be
            // shown regardless of the lock screen, the call to
            // {&#64;link ActivityStackSupervisor#checkReadyForSleepLocked} is skipped.
            final ActivityRecord next &#61; topRunningActivityLocked(true /* focusableOnly */);
            if (next &#61;&#61; null || !next.canTurnScreenOn()) {
                checkReadyForSleep();
            }
        } finally {
            mStackSupervisor.inResumeTopActivity &#61; false;
        }

        return result;
    }
</code></pre> 
<h3><a id="213_ASresumeTopActivityInnerLocked_1926"></a>2.13 AS.resumeTopActivityInnerLocked</h3> 
<p>[-&gt;ActivityStack.java]</p> 
<pre><code> &#64;GuardedBy(&#34;mService&#34;)
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        //系统没有进入booting或者booted状态被&#xff0c;则不允许启动Activity
        if (!mService.mBooting &amp;&amp; !mService.mBooted) {
            // Not ready yet!
            return false;
        }

        //找到top-most activity没有finishing的栈顶activity
        // Find the next top-most activity to resume in this stack that is not finishing and is
        // focusable. If it is not focusable, we will fall into the case below to resume the
        // top activity in the next focusable task.
        final ActivityRecord next &#61; topRunningActivityLocked(true /* focusableOnly */);

        final boolean hasRunningActivity &#61; next !&#61; null;

        // TODO: Maybe this entire condition can get removed?
        if (hasRunningActivity &amp;&amp; !isAttached()) {
            return false;
        }
        //top running之后的任意处于初始化状态且有显示startingWindow,则移除startingWindow
        mStackSupervisor.cancelInitializingActivities();

        // Remember how we&#39;ll process this pause/resume situation, and ensure
        // that the state is reset however we wind up proceeding.
        boolean userLeaving &#61; mStackSupervisor.mUserLeaving;
        mStackSupervisor.mUserLeaving &#61; false;

        if (!hasRunningActivity) {
            //见2.13.1节
            // There are no activities left in the stack, let&#39;s look somewhere else.
            return resumeTopActivityInNextFocusableStack(prev, options, &#34;noMoreActivities&#34;);
        }

        next.delayedResume &#61; false;
        //已经resume的情况
        // If the top activity is the resumed one, nothing to do.
        if (mResumedActivity &#61;&#61; next &amp;&amp; next.isState(RESUMED)
                &amp;&amp; mStackSupervisor.allResumedActivitiesComplete()) {
            // Make sure we have executed any pending transitions, since there
            // should be nothing left to do at this point.
            executeAppTransition(options);
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    &#34;resumeTopActivityLocked: Top activity resumed &#34; &#43; next);
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return false;
        }
        //处于睡眠或者关机状态&#xff0c;top activity已经暂停的情况
        // If we are sleeping, and there is no resumed activity, and the top
        // activity is paused, well that is the state we want.
        if (shouldSleepOrShutDownActivities()
                &amp;&amp; mLastPausedActivity &#61;&#61; next
                &amp;&amp; mStackSupervisor.allPausedActivitiesComplete()) {
            // Make sure we have executed any pending transitions, since there
            // should be nothing left to do at this point.
            executeAppTransition(options);
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    &#34;resumeTopActivityLocked: Going to sleep and all paused&#34;);
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return false;
        }
        //拥有该activity的用户没有启动则直接返回
        // Make sure that the user who owns this activity is started.  If not,
        // we will just leave it as is because someone should be bringing
        // another user&#39;s activities to the top of the stack.
        if (!mService.mUserController.hasStartedUserState(next.userId)) {
            Slog.w(TAG, &#34;Skipping resume of top activity &#34; &#43; next
                    &#43; &#34;: user &#34; &#43; next.userId &#43; &#34; is stopped&#34;);
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return false;
        }
        
        // The activity may be waiting for stop, but that is no longer
        // appropriate for it.
        mStackSupervisor.mStoppingActivities.remove(next);
        mStackSupervisor.mGoingToSleepActivities.remove(next);
        next.sleeping &#61; false;
        mStackSupervisor.mActivitiesWaitingForVisibleActivity.remove(next);

        if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, &#34;Resuming &#34; &#43; next);
        //当处于暂停activity&#xff0c;则直接返回
        // If we are currently pausing an activity, then don&#39;t do anything until that is done.
        if (!mStackSupervisor.allPausedActivitiesComplete()) {
            if (DEBUG_SWITCH || DEBUG_PAUSE || DEBUG_STATES) Slog.v(TAG_PAUSE,
                    &#34;resumeTopActivityLocked: Skip resume: some activity pausing.&#34;);
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return false;
        }

        mStackSupervisor.setLaunchSource(next.info.applicationInfo.uid);

        boolean lastResumedCanPip &#61; false;
        ActivityRecord lastResumed &#61; null;
        final ActivityStack lastFocusedStack &#61; mStackSupervisor.getLastStack();
        if (lastFocusedStack !&#61; null &amp;&amp; lastFocusedStack !&#61; this) {
            // So, why aren&#39;t we using prev here??? See the param comment on the method. prev doesn&#39;t
            // represent the last resumed activity. However, the last focus stack does if it isn&#39;t null.
            lastResumed &#61; lastFocusedStack.mResumedActivity;
            //多窗口模式判断
            if (userLeaving &amp;&amp; inMultiWindowMode() &amp;&amp; lastFocusedStack.shouldBeVisible(next)) {
                // The user isn&#39;t leaving if this stack is the multi-window mode and the last
                // focused stack should still be visible.
                if(DEBUG_USER_LEAVING) Slog.i(TAG_USER_LEAVING, &#34;Overriding userLeaving to false&#34;
                        &#43; &#34; next&#61;&#34; &#43; next &#43; &#34; lastResumed&#61;&#34; &#43; lastResumed);
                userLeaving &#61; false;
            }
            lastResumedCanPip &#61; lastResumed !&#61; null &amp;&amp; lastResumed.checkEnterPictureInPictureState(
                    &#34;resumeTopActivity&#34;, userLeaving /* beforeStopping */);
        }
        //要等待暂停当前activity完成&#xff0c;再resume top activity
        // If the flag RESUME_WHILE_PAUSING is set, then continue to schedule the previous activity
        // to be paused, while at the same time resuming the new resume activity only if the
        // previous activity can&#39;t go into Pip since we want to give Pip activities a chance to
        // enter Pip before resuming the next activity.
        final boolean resumeWhilePausing &#61; (next.info.flags &amp; FLAG_RESUME_WHILE_PAUSING) !&#61; 0
                &amp;&amp; !lastResumedCanPip;
        
        //暂停其他Activity&#xff0c;见13.2节
        boolean pausing &#61; mStackSupervisor.pauseBackStacks(userLeaving, next, false);
        if (mResumedActivity !&#61; null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    &#34;resumeTopActivityLocked: Pausing &#34; &#43; mResumedActivity);
            //当resume状态activity不为空&#xff0c;则需要暂停该Activity
            pausing |&#61; startPausingLocked(userLeaving, false, next, false);
        }
        if (pausing &amp;&amp; !resumeWhilePausing) {
            if (DEBUG_SWITCH || DEBUG_STATES) Slog.v(TAG_STATES,
                    &#34;resumeTopActivityLocked: Skip resume: need to start pausing&#34;);
            // At this point we want to put the upcoming activity&#39;s process
            // at the top of the LRU list, since we know we will be needing it
            // very soon and it would be a waste to let it get killed if it
            // happens to be sitting towards the end.
            if (next.app !&#61; null &amp;&amp; next.app.thread !&#61; null) {
                mService.updateLruProcessLocked(next.app, true, null);
            }
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            if (lastResumed !&#61; null) {
                lastResumed.setWillCloseOrEnterPip(true);
            }
            return true;
        } else if (mResumedActivity &#61;&#61; next &amp;&amp; next.isState(RESUMED)
                &amp;&amp; mStackSupervisor.allResumedActivitiesComplete()) {
            // It is possible for the activity to be resumed when we paused back stacks above if the
            // next activity doesn&#39;t have to wait for pause to complete.
            // So, nothing else to-do except:
            // Make sure we have executed any pending transitions, since there
            // should be nothing left to do at this point.
            executeAppTransition(options);
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    &#34;resumeTopActivityLocked: Top activity resumed (dontWaitForPause) &#34; &#43; next);
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return true;
        }

        // If the most recent activity was noHistory but was only stopped rather
        // than stopped&#43;finished because the device went to sleep, we need to make
        // sure to finish it as we&#39;re making a new activity topmost.
        if (shouldSleepActivities() &amp;&amp; mLastNoHistoryActivity !&#61; null &amp;&amp;
                !mLastNoHistoryActivity.finishing) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    &#34;no-history finish of &#34; &#43; mLastNoHistoryActivity &#43; &#34; on new resume&#34;);
            requestFinishActivityLocked(mLastNoHistoryActivity.appToken, Activity.RESULT_CANCELED,
                    null, &#34;resume-no-history&#34;, false);
            mLastNoHistoryActivity &#61; null;
        }

        if (prev !&#61; null &amp;&amp; prev !&#61; next) {
            if (!mStackSupervisor.mActivitiesWaitingForVisibleActivity.contains(prev)
                    &amp;&amp; next !&#61; null &amp;&amp; !next.nowVisible
                    &amp;&amp; checkKeyguardVisibility(next, true /* shouldBeVisible */,
                            next.isTopRunningActivity())) {
                mStackSupervisor.mActivitiesWaitingForVisibleActivity.add(prev);
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                        &#34;Resuming top, waiting visible to hide: &#34; &#43; prev);
            } else {
                // The next activity is already visible, so hide the previous
                // activity&#39;s windows right now so we can show the new one ASAP.
                // We only do this if the previous is finishing, which should mean
                // it is on top of the one being resumed so hiding it quickly
                // is good.  Otherwise, we want to do the normal route of allowing
                // the resumed activity to be shown so we can decide if the
                // previous should actually be hidden depending on whether the
                // new one is found to be full-screen or not.
                if (prev.finishing) {
                    prev.setVisibility(false);
                    if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                            &#34;Not waiting for visible to hide: &#34; &#43; prev &#43; &#34;, waitingVisible&#61;&#34;
                            &#43; mStackSupervisor.mActivitiesWaitingForVisibleActivity.contains(prev)
                            &#43; &#34;, nowVisible&#61;&#34; &#43; next.nowVisible);
                } else {
                    if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                            &#34;Previous already visible but still waiting to hide: &#34; &#43; prev
                            &#43; &#34;, waitingVisible&#61;&#34;
                            &#43; mStackSupervisor.mActivitiesWaitingForVisibleActivity.contains(prev)
                            &#43; &#34;, nowVisible&#61;&#34; &#43; next.nowVisible);
                }
            }
        }

        // Launching this app&#39;s activity, make sure the app is no longer
        // considered stopped.
        try {
            AppGlobals.getPackageManager().setPackageStoppedState(
                    next.packageName, false, next.userId); /* TODO: Verify if correct userid */
        } catch (RemoteException e1) {
        } catch (IllegalArgumentException e) {
            Slog.w(TAG, &#34;Failed trying to unstop package &#34;
                    &#43; next.packageName &#43; &#34;: &#34; &#43; e);
        }

        // We are starting up the next activity, so tell the window manager
        // that the previous one will be hidden soon.  This way it can know
        // to ignore it when computing the desired screen orientation.
        boolean anim &#61; true;
        if (prev !&#61; null) {
            if (prev.finishing) {
                if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION,
                        &#34;Prepare close transition: prev&#61;&#34; &#43; prev);
                if (mStackSupervisor.mNoAnimActivities.contains(prev)) {
                    anim &#61; false;
                    mWindowManager.prepareAppTransition(TRANSIT_NONE, false);
                } else {
                    mWindowManager.prepareAppTransition(prev.getTask() &#61;&#61; next.getTask()
                            ? TRANSIT_ACTIVITY_CLOSE
                            : TRANSIT_TASK_CLOSE, false);
                }
                prev.setVisibility(false);
            } else {
                if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION,
                        &#34;Prepare open transition: prev&#61;&#34; &#43; prev);
                if (mStackSupervisor.mNoAnimActivities.contains(next)) {
                    anim &#61; false;
                    mWindowManager.prepareAppTransition(TRANSIT_NONE, false);
                } else {
                    mWindowManager.prepareAppTransition(prev.getTask() &#61;&#61; next.getTask()
                            ? TRANSIT_ACTIVITY_OPEN
                            : next.mLaunchTaskBehind
                                    ? TRANSIT_TASK_OPEN_BEHIND
                                    : TRANSIT_TASK_OPEN, false);
                }
            }
        } else {
            if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION, &#34;Prepare open transition: no previous&#34;);
            if (mStackSupervisor.mNoAnimActivities.contains(next)) {
                anim &#61; false;
                mWindowManager.prepareAppTransition(TRANSIT_NONE, false);
            } else {
                mWindowManager.prepareAppTransition(TRANSIT_ACTIVITY_OPEN, false);
            }
        }

        if (anim) {
            next.applyOptionsLocked();
        } else {
            next.clearOptionsLocked();
        }

        mStackSupervisor.mNoAnimActivities.clear();
        ActivityStack lastStack &#61; mStackSupervisor.getLastStack();
        //进程存在的情况
        if (next.app !&#61; null &amp;&amp; next.app.thread !&#61; null) {
            if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, &#34;Resume running: &#34; &#43; next
                    &#43; &#34; stopped&#61;&#34; &#43; next.stopped &#43; &#34; visible&#61;&#34; &#43; next.visible);

            // If the previous activity is translucent, force a visibility update of
            // the next activity, so that it&#39;s added to WM&#39;s opening app list, and
            // transition animation can be set up properly.
            // For example, pressing Home button with a translucent activity in focus.
            // Launcher is already visible in this case. If we don&#39;t add it to opening
            // apps, maybeUpdateTransitToWallpaper() will fail to identify this as a
            // TRANSIT_WALLPAPER_OPEN animation, and run some funny animation.
            final boolean lastActivityTranslucent &#61; lastStack !&#61; null
                    &amp;&amp; (lastStack.inMultiWindowMode()
                    || (lastStack.mLastPausedActivity !&#61; null
                    &amp;&amp; !lastStack.mLastPausedActivity.fullscreen));

            // The contained logic must be synchronized, since we are both changing the visibility
            // and updating the {&#64;link Configuration}. {&#64;link ActivityRecord#setVisibility} will
            // ultimately cause the client code to schedule a layout. Since layouts retrieve the
            // current {&#64;link Configuration}, we must ensure that the below code updates it before
            // the layout can occur.
            synchronized(mWindowManager.getWindowManagerLock()) {
                //设置activity可见
                // This activity is now becoming visible.
                if (!next.visible || next.stopped || lastActivityTranslucent) {
                    next.setVisibility(true);
                }

                // schedule launch ticks to collect information about slow apps.
                next.startLaunchTickingLocked();

                ActivityRecord lastResumedActivity &#61;
                        lastStack &#61;&#61; null ? null :lastStack.mResumedActivity;
                final ActivityState lastState &#61; next.getState();

                mService.updateCpuStats();

                if (DEBUG_STATES) Slog.v(TAG_STATES, &#34;Moving to RESUMED: &#34; &#43; next
                        &#43; &#34; (in existing)&#34;);
               //设置activity resume
                next.setState(RESUMED, &#34;resumeTopActivityInnerLocked&#34;);

                mService.updateLruProcessLocked(next.app, true, null);
                updateLRUListLocked(next);
                mService.updateOomAdjLocked();

                // Have the window manager re-evaluate the orientation of
                // the screen based on the new activity order.
                boolean notUpdated &#61; true;

                if (mStackSupervisor.isFocusedStack(this)) {
                    // We have special rotation behavior when here is some active activity that
                    // requests specific orientation or Keyguard is locked. Make sure all activity
                    // visibilities are set correctly as well as the transition is updated if needed
                    // to get the correct rotation behavior. Otherwise the following call to update
                    // the orientation may cause incorrect configurations delivered to client as a
                    // result of invisible window resize.
                    // TODO: Remove this once visibilities are set correctly immediately when
                    // starting an activity.
                    notUpdated &#61; !mStackSupervisor.ensureVisibilityAndConfig(next, mDisplayId,
                            true /* markFrozenIfConfigChanged */, false /* deferResume */);
                }

                if (notUpdated) {
                    // The configuration update wasn&#39;t able to keep the existing
                    // instance of the activity, and instead started a new one.
                    // We should be all done, but let&#39;s just make sure our activity
                    // is still at the top and schedule another run if something
                    // weird happened.
                    ActivityRecord nextNext &#61; topRunningActivityLocked();
                    if (DEBUG_SWITCH || DEBUG_STATES) Slog.i(TAG_STATES,
                            &#34;Activity config changed during resume: &#34; &#43; next
                                    &#43; &#34;, new next: &#34; &#43; nextNext);
                    if (nextNext !&#61; next) {
                        // Do over!
                        mStackSupervisor.scheduleResumeTopActivities();
                    }
                    if (!next.visible || next.stopped) {
                        next.setVisibility(true);
                    }
                    next.completeResumeLocked();
                    if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                    return true;
                }

                try {
                    //分发所有pending结果
                    final ClientTransaction transaction &#61; ClientTransaction.obtain(next.app.thread,
                            next.appToken);
                    // Deliver all pending results.
                    ArrayList&lt;ResultInfo&gt; a &#61; next.results;
                    if (a !&#61; null) {
                        final int N &#61; a.size();
                        if (!next.finishing &amp;&amp; N &gt; 0) {
                            if (DEBUG_RESULTS) Slog.v(TAG_RESULTS,
                                    &#34;Delivering results to &#34; &#43; next &#43; &#34;: &#34; &#43; a);
                            transaction.addCallback(ActivityResultItem.obtain(a));
                        }
                    }

                    if (next.newIntents !&#61; null) {
                        transaction.addCallback(NewIntentItem.obtain(next.newIntents,
                                false /* andPause */));
                    }

                    // Well the app will no longer be stopped.
                    // Clear app token stopped state in window manager if needed.
                    next.notifyAppResumed(next.stopped);

                    EventLog.writeEvent(EventLogTags.AM_RESUME_ACTIVITY, next.userId,
                            System.identityHashCode(next), next.getTask().taskId,
                            next.shortComponentName);

                    next.sleeping &#61; false;
                    mService.getAppWarningsLocked().onResumeActivity(next);
                    mService.showAskCompatModeDialogLocked(next);
                    next.app.pendingUiClean &#61; true;
                    next.app.forceProcessStateUpTo(mService.mTopProcessState);
                    next.clearOptionsLocked();
                    //处罚onResume
                    transaction.setLifecycleStateRequest(
                            ResumeActivityItem.obtain(next.app.repProcState,
                                    mService.isNextTransitionForward()));
                    mService.getLifecycleManager().scheduleTransaction(transaction);

                    if (DEBUG_STATES) Slog.d(TAG_STATES, &#34;resumeTopActivityLocked: Resumed &#34;
                            &#43; next);
                } catch (Exception e) {
                    // Whoops, need to restart this activity!
                    if (DEBUG_STATES) Slog.v(TAG_STATES, &#34;Resume failed; resetting state to &#34;
                            &#43; lastState &#43; &#34;: &#34; &#43; next);
                    next.setState(lastState, &#34;resumeTopActivityInnerLocked&#34;);

                    // lastResumedActivity being non-null implies there is a lastStack present.
                    if (lastResumedActivity !&#61; null) {
                        lastResumedActivity.setState(RESUMED, &#34;resumeTopActivityInnerLocked&#34;);
                    }

                    Slog.i(TAG, &#34;Restarting because process died: &#34; &#43; next);
                    if (!next.hasBeenLaunched) {
                        next.hasBeenLaunched &#61; true;
                    } else  if (SHOW_APP_STARTING_PREVIEW &amp;&amp; lastStack !&#61; null
                            &amp;&amp; lastStack.isTopStackOnDisplay()) {
                        next.showStartingWindow(null /* prev */, false /* newTask */,
                                false /* taskSwitch */);
                    }
                    mStackSupervisor.startSpecificActivityLocked(next, true, false);
                    if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                    return true;
                }
            }

            // From this point on, if something goes wrong there is no way
            // to recover the activity.
            try {
                next.completeResumeLocked();
            } catch (Exception e) {
                // If any exception gets thrown, toss away this
                // activity and try the next one.
                Slog.w(TAG, &#34;Exception thrown during resume of &#34; &#43; next, e);
                requestFinishActivityLocked(next.appToken, Activity.RESULT_CANCELED, null,
                        &#34;resume-exception&#34;, true);
                if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                return true;
            }
        } else {
            //需要重新启动Activity
            // Whoops, need to restart this activity!
            if (!next.hasBeenLaunched) {
                next.hasBeenLaunched &#61; true;
            } else {
                if (SHOW_APP_STARTING_PREVIEW) {
                    next.showStartingWindow(null /* prev */, false /* newTask */,
                            false /* taskSwich */);
                }
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, &#34;Restarting: &#34; &#43; next);
            }
            if (DEBUG_STATES) Slog.d(TAG_STATES, &#34;resumeTopActivityLocked: Restarting &#34; &#43; next);
            //见2.14节
            mStackSupervisor.startSpecificActivityLocked(next, true, true);
        }

        if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
        return true;
    }
</code></pre> 
<p>主要工作如下&#xff1a;</p> 
<ul><li>当找不到resume的activity时&#xff0c;则直接回到桌面</li><li>当resume状态activity不为空,则执行startPausingLocked,暂停该Activity</li><li>当Activity之前启动过&#xff0c;则直接resume&#xff0c;否则执行startSpecificActivityLocked&#xff0c;2.14节将继续讨论。</li></ul> 
<h4><a id="2131_ASresumeTopActivityInNextFocusableStack_2384"></a>2.13.1 AS.resumeTopActivityInNextFocusableStack</h4> 
<p>[-&gt;ActivityStack.java]</p> 
<pre><code> private boolean resumeTopActivityInNextFocusableStack(ActivityRecord prev,
            ActivityOptions options, String reason) {
        if (adjustFocusToNextFocusableStack(reason)) {
            //如果该栈没有全屏&#xff0c;则尝试下一个可见的stack
            // Try to move focus to the next visible stack with a running activity if this
            // stack is not covering the entire screen or is on a secondary display (with no home
            // stack).
            return mStackSupervisor.resumeFocusedStackTopActivityLocked(
                    mStackSupervisor.getFocusedStack(), prev, null);
        }

        // Let&#39;s just start up the Launcher...
        ActivityOptions.abort(options);
        if (DEBUG_STATES) Slog.d(TAG_STATES,
                &#34;resumeTopActivityInNextFocusableStack: &#34; &#43; reason &#43; &#34;, go home&#34;);
        if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
        // Only resume home if on home display
        //启动桌面activity
        return isOnHomeDisplay() &amp;&amp;
                mStackSupervisor.resumeHomeStackTask(prev, reason);
    }
</code></pre> 
<p>当找不到需要的resume的Activity时&#xff0c;直接回到桌面</p> 
<h4><a id="2132_ASpauseBackStacks_2414"></a>2.13.2 AS.pauseBackStacks</h4> 
<p>[-&gt;ActivityStack.java]</p> 
<pre><code> /**
     * Pause all activities in either all of the stacks or just the back stacks.
     * &#64;param userLeaving Passed to pauseActivity() to indicate whether to call onUserLeaving().
     * &#64;param resuming The resuming activity.
     * &#64;param dontWait The resuming activity isn&#39;t going to wait for all activities to be paused
     *                 before resuming.
     * &#64;return true if any activity was paused as a result of this call.
     */
    boolean pauseBackStacks(boolean userLeaving, ActivityRecord resuming, boolean dontWait) {
        boolean someActivityPaused &#61; false;
        for (int displayNdx &#61; mActivityDisplays.size() - 1; displayNdx &gt;&#61; 0; --displayNdx) {
            final ActivityDisplay display &#61; mActivityDisplays.valueAt(displayNdx);
            for (int stackNdx &#61; display.getChildCount() - 1; stackNdx &gt;&#61; 0; --stackNdx) {
                final ActivityStack stack &#61; display.getChildAt(stackNdx);
                if (!isFocusedStack(stack) &amp;&amp; stack.getResumedActivity() !&#61; null) {
                    if (DEBUG_STATES) Slog.d(TAG_STATES, &#34;pauseBackStacks: stack&#61;&#34; &#43; stack &#43;
                            &#34; mResumedActivity&#61;&#34; &#43; stack.getResumedActivity());
                    //见2.13.2节
                    someActivityPaused |&#61; stack.startPausingLocked(userLeaving, false, resuming,
                            dontWait);
                }
            }
        }
        return someActivityPaused;
    }
</code></pre> 
<p>暂停所有处于后台栈的所有Activity</p> 
<h4><a id="2133_ASstartPausingLocked_2448"></a>2.13.3 AS.startPausingLocked</h4> 
<p>[-&gt;ActivityStack.java]</p> 
<pre><code>final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
            ActivityRecord resuming, boolean pauseImmediately) {
        if (mPausingActivity !&#61; null) {
            Slog.wtf(TAG, &#34;Going to pause when pause is already pending for &#34; &#43; mPausingActivity
                    &#43; &#34; state&#61;&#34; &#43; mPausingActivity.getState());
            if (!shouldSleepActivities()) {
                // Avoid recursion among check for sleep and complete pause during sleeping.
                // Because activity will be paused immediately after resume, just let pause
                // be completed by the order of activity paused from clients.
                completePauseLocked(false, resuming);
            }
        }
        ActivityRecord prev &#61; mResumedActivity;

        if (prev &#61;&#61; null) {
            if (resuming &#61;&#61; null) {
                Slog.wtf(TAG, &#34;Trying to pause when nothing is resumed&#34;);
                mStackSupervisor.resumeFocusedStackTopActivityLocked();
            }
            return false;
        }

        if (prev &#61;&#61; resuming) {
            Slog.wtf(TAG, &#34;Trying to pause activity that is in process of being resumed&#34;);
            return false;
        }

        if (DEBUG_STATES) Slog.v(TAG_STATES, &#34;Moving to PAUSING: &#34; &#43; prev);
        else if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, &#34;Start pausing: &#34; &#43; prev);
        mPausingActivity &#61; prev;
        mLastPausedActivity &#61; prev;
        mLastNoHistoryActivity &#61; (prev.intent.getFlags() &amp; Intent.FLAG_ACTIVITY_NO_HISTORY) !&#61; 0
                || (prev.info.flags &amp; ActivityInfo.FLAG_NO_HISTORY) !&#61; 0 ? prev : null;
        prev.setState(PAUSING, &#34;startPausingLocked&#34;);
        prev.getTask().touchActiveTime();
        clearLaunchTime(prev);

        mStackSupervisor.getActivityMetricsLogger().stopFullyDrawnTraceIfNeeded();

        mService.updateCpuStats();

        if (prev.app !&#61; null &amp;&amp; prev.app.thread !&#61; null) {
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, &#34;Enqueueing pending pause: &#34; &#43; prev);
            try {
                EventLogTags.writeAmPauseActivity(prev.userId, System.identityHashCode(prev),
                        prev.shortComponentName, &#34;userLeaving&#61;&#34; &#43; userLeaving);
                mService.updateUsageStats(prev, false);
                //暂停目标Activity
                mService.getLifecycleManager().scheduleTransaction(prev.app.thread, prev.appToken,
                        PauseActivityItem.obtain(prev.finishing, userLeaving,
                                prev.configChangeFlags, pauseImmediately));
            } catch (Exception e) {
                // Ignore exception, if process died other code will cleanup.
                Slog.w(TAG, &#34;Exception thrown during pause&#34;, e);
                mPausingActivity &#61; null;
                mLastPausedActivity &#61; null;
                mLastNoHistoryActivity &#61; null;
            }
        } else {
            mPausingActivity &#61; null;
            mLastPausedActivity &#61; null;
            mLastNoHistoryActivity &#61; null;
        }

        // If we are not going to sleep, we want to ensure the device is
        // awake until the next activity is started.
        if (!uiSleeping &amp;&amp; !mService.isSleepingOrShuttingDownLocked()) {
            mStackSupervisor.acquireLaunchWakelock();
        }

        if (mPausingActivity !&#61; null) {
            // Have the window manager pause its key dispatching until the new
            // activity has started.  If we&#39;re pausing the activity just because
            // the screen is being turned off and the UI is sleeping, don&#39;t interrupt
            // key dispatch; the same activity will pick it up again on wakeup.
            if (!uiSleeping) {
                prev.pauseKeyDispatchingLocked();
            } else if (DEBUG_PAUSE) {
                 Slog.v(TAG_PAUSE, &#34;Key dispatch not paused for screen off&#34;);
            }

            if (pauseImmediately) {
                // If the caller said they don&#39;t want to wait for the pause, then complete
                // the pause now.
                completePauseLocked(false, resuming);
                return false;

            } else {
                //500ms,执行暂停超时的消息
                schedulePauseTimeout(prev);
                return true;
            }

        } else {
            // This activity failed to schedule the
            // pause, so just treat it as being paused now.
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, &#34;Activity not running, resuming next.&#34;);
            if (resuming &#61;&#61; null) {  //调度失败&#xff0c;则认为暂停结束开始执行resume操作
                mStackSupervisor.resumeFocusedStackTopActivityLocked();
            }
            return false;
        }
    }
</code></pre> 
<p>通过LifecycleManager的方式来暂停Activity操作。对于pauseImmediately&#61; true则执行completePauseLocked操作&#xff0c;否则等待app通知500ms超时再执行该方法。</p> 
<h4><a id="2134_AScompletePauseLocked_2560"></a>2.13.4 AS.completePauseLocked</h4> 
<p>[-&gt;ActivityStack.java]</p> 
<pre><code> private void completePauseLocked(boolean resumeNext, ActivityRecord resuming) {
        ActivityRecord prev &#61; mPausingActivity;
        if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, &#34;Complete pause: &#34; &#43; prev);

        if (prev !&#61; null) {
            prev.setWillCloseOrEnterPip(false);
            final boolean wasStopping &#61; prev.isState(STOPPING);
            prev.setState(PAUSED, &#34;completePausedLocked&#34;);
            if (prev.finishing) {
                if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, &#34;Executing finish of activity: &#34; &#43; prev);
                prev &#61; finishCurrentActivityLocked(prev, FINISH_AFTER_VISIBLE, false,
                        &#34;completedPausedLocked&#34;);
            } else if (prev.app !&#61; null) {
                if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, &#34;Enqueue pending stop if needed: &#34; &#43; prev
                        &#43; &#34; wasStopping&#61;&#34; &#43; wasStopping &#43; &#34; visible&#61;&#34; &#43; prev.visible);
                if (mStackSupervisor.mActivitiesWaitingForVisibleActivity.remove(prev)) {
                    if (DEBUG_SWITCH || DEBUG_PAUSE) Slog.v(TAG_PAUSE,
                            &#34;Complete pause, no longer waiting: &#34; &#43; prev);
                }
                if (prev.deferRelaunchUntilPaused) {
                    // Complete the deferred relaunch that was waiting for pause to complete.
                    if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, &#34;Re-launching after pause: &#34; &#43; prev);
                    prev.relaunchActivityLocked(false /* andResume */,
                            prev.preserveWindowOnDeferredRelaunch);
                } else if (wasStopping) {
                    // We are also stopping, the stop request must have gone soon after the pause.
                    // We can&#39;t clobber it, because the stop confirmation will not be handled.
                    // We don&#39;t need to schedule another stop, we only need to let it happen.
                    prev.setState(STOPPING, &#34;completePausedLocked&#34;);
                } else if (!prev.visible || shouldSleepOrShutDownActivities()) {
                    // Clear out any deferred client hide we might currently have.
                    prev.setDeferHidingClient(false);
                    // If we were visible then resumeTopActivities will release resources before
                    // stopping.
                    addToStopping(prev, true /* scheduleIdle */, false /* idleDelayed */);
                }
            } else {
                if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, &#34;App died during pause, not stopping: &#34; &#43; prev);
                prev &#61; null;
            }
            // It is possible the activity was freezing the screen before it was paused.
            // In that case go ahead and remove the freeze this activity has on the screen
            // since it is no longer visible.
            if (prev !&#61; null) {
                prev.stopFreezingScreenLocked(true /*force*/);
            }
            mPausingActivity &#61; null;
        }

        if (resumeNext) {
            final ActivityStack topStack &#61; mStackSupervisor.getFocusedStack();
            if (!topStack.shouldSleepOrShutDownActivities()) {
                mStackSupervisor.resumeFocusedStackTopActivityLocked(topStack, prev, null);
            } else {
                checkReadyForSleep();
                ActivityRecord top &#61; topStack.topRunningActivityLocked();
                if (top &#61;&#61; null || (prev !&#61; null &amp;&amp; top !&#61; prev)) {
                    // If there are no more activities available to run, do resume anyway to start
                    // something. Also if the top activity on the stack is not the just paused
                    // activity, we need to go ahead and resume it to ensure we complete an
                    // in-flight app switch.
                    mStackSupervisor.resumeFocusedStackTopActivityLocked();
                }
            }
        }

        if (prev !&#61; null) {
            prev.resumeKeyDispatchingLocked();

            if (prev.app !&#61; null &amp;&amp; prev.cpuTimeAtResume &gt; 0
                    &amp;&amp; mService.mBatteryStatsService.isOnBattery()) {
                long diff &#61; mService.mProcessCpuTracker.getCpuTimeForPid(prev.app.pid)
                        - prev.cpuTimeAtResume;
                if (diff &gt; 0) {
                    BatteryStatsImpl bsi &#61; mService.mBatteryStatsService.getActiveStatistics();
                    synchronized (bsi) {
                        BatteryStatsImpl.Uid.Proc ps &#61;
                                bsi.getProcessStatsLocked(prev.info.applicationInfo.uid,
                                        prev.info.packageName);
                        if (ps !&#61; null) {
                            ps.addForegroundTimeLocked(diff);
                        }
                    }
                }
            }
            prev.cpuTimeAtResume &#61; 0; // reset it
        }

        // Notify when the task stack has changed, but only if visibilities changed (not just
        // focus). Also if there is an active pinned stack - we always want to notify it about
        // task stack changes, because its positioning may depend on it.
        if (mStackSupervisor.mAppVisibilitiesChangedSinceLastPause
                || getDisplay().hasPinnedStack()) {
            mService.mTaskChangeNotificationController.notifyTaskStackChanged();
            mStackSupervisor.mAppVisibilitiesChangedSinceLastPause &#61; false;
        }

        mStackSupervisor.ensureActivitiesVisibleLocked(resuming, 0, !PRESERVE_WINDOWS);
    }
</code></pre> 
<p>暂停Activity完成后&#xff0c;修改暂停activity状态</p> 
<h3><a id="214__ASSstartSpecificActivityLocked_2668"></a>2.14 ASS.startSpecificActivityLocked</h3> 
<p>[-&gt;ActivityStackSupervisor.java]</p> 
<pre><code> void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // Is this activity&#39;s application already running?
        ProcessRecord app &#61; mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);

        if (app !&#61; null &amp;&amp; app.thread !&#61; null) {
            try {
                if ((r.info.flags&amp;ActivityInfo.FLAG_MULTIPROCESS) &#61;&#61; 0
                        || !&#34;android&#34;.equals(r.info.packageName)) {
                    // Don&#39;t add this if it is a platform component that is marked
                    // to run in multiple processes, because this is actually
                    // part of the framework so doesn&#39;t make sense to track as a
                    // separate apk in the process.
                    app.addPackage(r.info.packageName, r.info.applicationInfo.longVersionCode,
                            mService.mProcessStats);
                }
                //真正启动Activity&#xff0c;见2.18节
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, &#34;Exception when starting activity &#34;
                        &#43; r.intent.getComponent().flattenToShortString(), e);
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }
        //当进程不存在&#xff0c;则创建进程
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                &#34;activity&#34;, r.intent.getComponent(), false, false, true);
    }
</code></pre> 
<h3><a id="215_AMSstartProcessLocked_2707"></a>2.15 AMS.startProcessLocked</h3> 
<p>这个过程为启动Android进程的过程&#xff0c;在文章Android进程启动过程解析&#xff0c;详细描述了startProcessLocked整个过程&#xff0c;创建完成新进程之后&#xff0c;在新进程中通过binder ipc方式后调用到AMS.attachApplicationLocked。</p> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code>   private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid, int callingUid, long startSeq) {
           ...
            if (app.isolatedEntryPoint !&#61; null) {
                // This is an isolated process which should just call an entry point instead of
                // being bound to an application.
                thread.runIsolatedEntryPoint(app.isolatedEntryPoint, app.isolatedEntryPointArgs);
            } else if (app.instr !&#61; null) {
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
            } else {
                thread.bindApplication(processName, appInfo, providers, null, profilerInfo,
                        null, null, null, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.persistent,
                        new Configuration(getGlobalConfiguration()), app.compat,
                        getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, isAutofillCompatEnabled);
            }
            ...
        // See if the top visible activity is waiting to run in this process...
        if (normalMode) {
            try {
                if (mStackSupervisor.attachApplicationLocked(app)) {
                    didSomething &#61; true;
                }
            } catch (Exception e) {
                Slog.wtf(TAG, &#34;Exception thrown launching activities in &#34; &#43; app, e);
                badApp &#61; true;
            }
        }
   }
</code></pre> 
<p>在bindApplication后&#xff0c;调用了ASS.attachApplicationLocked。</p> 
<h3><a id="216_ASSattachApplicationLocked_2760"></a>2.16 ASS.attachApplicationLocked</h3> 
<p>[-&gt;ActivityStackSupervisor.java]</p> 
<pre><code> boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
        final String processName &#61; app.processName;
        boolean didSomething &#61; false;
        for (int displayNdx &#61; mActivityDisplays.size() - 1; displayNdx &gt;&#61; 0; --displayNdx) {
            final ActivityDisplay display &#61; mActivityDisplays.valueAt(displayNdx);
            for (int stackNdx &#61; display.getChildCount() - 1; stackNdx &gt;&#61; 0; --stackNdx) {
                final ActivityStack stack &#61; display.getChildAt(stackNdx);
                if (!isFocusedStack(stack)) {
                    continue;
                }
                stack.getAllRunningVisibleActivitiesLocked(mTmpActivityList);
                //获取前台栈顶第一个非finishing的Activity
                final ActivityRecord top &#61; stack.topRunningActivityLocked();
                final int size &#61; mTmpActivityList.size();
                for (int i &#61; 0; i &lt; size; i&#43;&#43;) {
                    final ActivityRecord activity &#61; mTmpActivityList.get(i);
                    if (activity.app &#61;&#61; null &amp;&amp; app.uid &#61;&#61; activity.info.applicationInfo.uid
                            &amp;&amp; processName.equals(activity.processName)) {
                        try {
                            //真正启动Activity&#xff0c;见2.15节
                            if (realStartActivityLocked(activity, app,
                                    top &#61;&#61; activity /* andResume */, true /* checkConfig */)) {
                                didSomething &#61; true;
                            }
                        } catch (RemoteException e) {
                            Slog.w(TAG, &#34;Exception in new application when starting activity &#34;
                                    &#43; top.intent.getComponent().flattenToShortString(), e);
                            throw e;
                        }
                    }
                }
            }
        }
        if (!didSomething) {
            //启动Activity不成功&#xff0c;确保有可见的Activity
            ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
        }
        return didSomething;
    }
</code></pre> 
<h3><a id="217_ASSrealStartActivityLocked_2806"></a>2.17 ASS.realStartActivityLocked</h3> 
<p>[-&gt;ActivityStackSupervisor.java]</p> 
<pre><code>  final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
            boolean andResume, boolean checkConfig) throws RemoteException {

        if (!allPausedActivitiesComplete()) {
            //如果Activity没有pausing完成则返回
            // While there are activities pausing we skipping starting any new activities until
            // pauses are complete. NOTE: that we also do this for activities that are starting in
            // the paused state because they will first be resumed then paused on the client side.
            if (DEBUG_SWITCH || DEBUG_PAUSE || DEBUG_STATES) Slog.v(TAG_PAUSE,
                    &#34;realStartActivityLocked: Skipping start of r&#61;&#34; &#43; r
                    &#43; &#34; some activities pausing...&#34;);
            return false;
        }

        final TaskRecord task &#61; r.getTask();
        final ActivityStack stack &#61; task.getStack();

        beginDeferResume();

        try {
            r.startFreezingScreenLocked(app, 0);
            //启动tick&#xff0c;收集应用启动慢的信息
            // schedule launch ticks to collect information about slow apps.
            r.startLaunchTickingLocked();

            r.setProcess(app);

            if (getKeyguardController().isKeyguardLocked()) {
                r.notifyUnknownVisibilityLaunched();
            }
           
            // Have the window manager re-evaluate the orientation of the screen based on the new
            // activity order.  Note that as a result of this, it can call back into the activity
            // manager with a new orientation.  We don&#39;t care about that, because the activity is
            // not currently running so we are just restarting it anyway.
            if (checkConfig) {
                // Deferring resume here because we&#39;re going to launch new activity shortly.
                // We don&#39;t want to perform a redundant launch of the same record while ensuring
                // configurations and trying to resume top activity of focused stack.
                ensureVisibilityAndConfig(r, r.getDisplayId(),
                        false /* markFrozenIfConfigChanged */, true /* deferResume */);
            }

            if (r.getStack().checkKeyguardVisibility(r, true /* shouldBeVisible */,
                    true /* isTop */)) {
                // We only set the visibility to true if the activity is allowed to be visible
                // based on
                // keyguard state. This avoids setting this into motion in window manager that is
                // later cancelled due to later calls to ensure visible activities that set
                // visibility back to false.
                r.setVisibility(true);
            }

            final int applicationInfoUid &#61;
                    (r.info.applicationInfo !&#61; null) ? r.info.applicationInfo.uid : -1;
            if ((r.userId !&#61; app.userId) || (r.appInfo.uid !&#61; applicationInfoUid)) {
                Slog.wtf(TAG,
                        &#34;User ID for activity changing for &#34; &#43; r
                                &#43; &#34; appInfo.uid&#61;&#34; &#43; r.appInfo.uid
                                &#43; &#34; info.ai.uid&#61;&#34; &#43; applicationInfoUid
                                &#43; &#34; old&#61;&#34; &#43; r.app &#43; &#34; new&#61;&#34; &#43; app);
            }

            app.waitingToKill &#61; null;
            r.launchCount&#43;&#43;;
            r.lastLaunchTime &#61; SystemClock.uptimeMillis();

            if (DEBUG_ALL) Slog.v(TAG, &#34;Launching: &#34; &#43; r);

            int idx &#61; app.activities.indexOf(r);
            if (idx &lt; 0) {
                app.activities.add(r);
            }
            
            mService.updateLruProcessLocked(app, true, null);
            mService.updateOomAdjLocked();

            final LockTaskController lockTaskController &#61; mService.getLockTaskController();
            if (task.mLockTaskAuth &#61;&#61; LOCK_TASK_AUTH_LAUNCHABLE
                    || task.mLockTaskAuth &#61;&#61; LOCK_TASK_AUTH_LAUNCHABLE_PRIV
                    || (task.mLockTaskAuth &#61;&#61; LOCK_TASK_AUTH_WHITELISTED
                            &amp;&amp; lockTaskController.getLockTaskModeState()
                                    &#61;&#61; LOCK_TASK_MODE_LOCKED)) {
                lockTaskController.startLockTaskMode(task, false, 0 /* blank UID */);
            }

            try {
                if (app.thread &#61;&#61; null) {
                    throw new RemoteException();
                }
                List&lt;ResultInfo&gt; results &#61; null;
                List&lt;ReferrerIntent&gt; newIntents &#61; null;
                if (andResume) {
                    // We don&#39;t need to deliver new intents and/or set results if activity is going
                    // to pause immediately after launch.
                    results &#61; r.results;
                    newIntents &#61; r.newIntents;
                }
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                        &#34;Launching: &#34; &#43; r &#43; &#34; icicle&#61;&#34; &#43; r.icicle &#43; &#34; with results&#61;&#34; &#43; results
                                &#43; &#34; newIntents&#61;&#34; &#43; newIntents &#43; &#34; andResume&#61;&#34; &#43; andResume);
                EventLog.writeEvent(EventLogTags.AM_RESTART_ACTIVITY, r.userId,
                        System.identityHashCode(r), task.taskId, r.shortComponentName);
                if (r.isActivityTypeHome()) {
                    //home进程是该栈的根进程
                    // Home process is the root process of the task.
                    mService.mHomeProcess &#61; task.mActivities.get(0).app;
                }
                mService.notifyPackageUse(r.intent.getComponent().getPackageName(),
                        PackageManager.NOTIFY_PACKAGE_USE_ACTIVITY);
                r.sleeping &#61; false;
                r.forceNewConfig &#61; false;
                mService.getAppWarningsLocked().onStartActivity(r);
                mService.showAskCompatModeDialogLocked(r);
                r.compat &#61; mService.compatibilityInfoForPackageLocked(r.info.applicationInfo);
                ProfilerInfo profilerInfo &#61; null;
                if (mService.mProfileApp !&#61; null &amp;&amp; mService.mProfileApp.equals(app.processName)) {
                    if (mService.mProfileProc &#61;&#61; null || mService.mProfileProc &#61;&#61; app) {
                        mService.mProfileProc &#61; app;
                        ProfilerInfo profilerInfoSvc &#61; mService.mProfilerInfo;
                        if (profilerInfoSvc !&#61; null &amp;&amp; profilerInfoSvc.profileFile !&#61; null) {
                            if (profilerInfoSvc.profileFd !&#61; null) {
                                try {
                                    profilerInfoSvc.profileFd &#61; profilerInfoSvc.profileFd.dup();
                                } catch (IOException e) {
                                    profilerInfoSvc.closeFd();
                                }
                            }

                            profilerInfo &#61; new ProfilerInfo(profilerInfoSvc);
                        }
                    }
                }

                app.hasShownUi &#61; true;
                app.pendingUiClean &#61; true;
                //将该进程设置为前台进程PROCESS_STATE_TOP
                app.forceProcessStateUpTo(mService.mTopProcessState);
                // Because we could be starting an Activity in the system process this may not go
                // across a Binder interface which would create a new Configuration. Consequently
                // we have to always create a new Configuration here.

                final MergedConfiguration mergedConfiguration &#61; new MergedConfiguration(
                        mService.getGlobalConfiguration(), r.getMergedOverrideConfiguration());
                r.setLastReportedConfiguration(mergedConfiguration);

                logIfTransactionTooLarge(r.intent, r.icicle);

                //创建Activity启动事务
                // Create activity launch transaction.
                final ClientTransaction clientTransaction &#61; ClientTransaction.obtain(app.thread,
                        r.appToken);
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        // TODO: Have this take the merged configuration instead of separate global
                        // and override configs.
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                        r.persistentState, results, newIntents, mService.isNextTransitionForward(),
                        profilerInfo));

                //设置目标事务的状态为onResume
                // Set desired final state.
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                    lifecycleItem &#61; ResumeActivityItem.obtain(mService.isNextTransitionForward());
                } else {
                    lifecycleItem &#61; PauseActivityItem.obtain();
                }
                clientTransaction.setLifecycleStateRequest(lifecycleItem);

                //通过transaciton方式开始activity生命周期&#xff0c;onCreate,onStart,onResume
                // Schedule transaction.
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);


                if ((app.info.privateFlags &amp; ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) !&#61; 0
                        &amp;&amp; mService.mHasHeavyWeightFeature) {
                    //处理heavy-weight进程
                    // This may be a heavy-weight process!  Note that the package
                    // manager will ensure that only activity can run in the main
                    // process of the .apk, which is the only thing that will be
                    // considered heavy-weight.
                    if (app.processName.equals(app.info.packageName)) {
                        if (mService.mHeavyWeightProcess !&#61; null
                                &amp;&amp; mService.mHeavyWeightProcess !&#61; app) {
                            Slog.w(TAG, &#34;Starting new heavy weight process &#34; &#43; app
                                    &#43; &#34; when already running &#34;
                                    &#43; mService.mHeavyWeightProcess);
                        }
                        mService.mHeavyWeightProcess &#61; app;
                        Message msg &#61; mService.mHandler.obtainMessage(
                                ActivityManagerService.POST_HEAVY_NOTIFICATION_MSG);
                        msg.obj &#61; r;
                        mService.mHandler.sendMessage(msg);
                    }
                }

            } catch (RemoteException e) {
                if (r.launchFailed) {
                    //第二次启动失败&#xff0c;则结束该Activity
                    // This is the second time we failed -- finish activity
                    // and give up.
                    Slog.e(TAG, &#34;Second failure launching &#34;
                            &#43; r.intent.getComponent().flattenToShortString()
                            &#43; &#34;, giving up&#34;, e);
                    mService.appDiedLocked(app);
                    stack.requestFinishActivityLocked(r.appToken, Activity.RESULT_CANCELED, null,
                            &#34;2nd-crash&#34;, false);
                    return false;
                }
                //第一次启动失败&#xff0c;则重启进程
                // This is the first time we failed -- restart process and
                // retry.
                r.launchFailed &#61; true;
                app.activities.remove(r);
                throw e;
            }
        } finally {
            endDeferResume();
        }

        r.launchFailed &#61; false;
         //将该进程加入到mLruActivity队列顶部
        if (stack.updateLRUListLocked(r)) {
            Slog.w(TAG, &#34;Activity &#34; &#43; r &#43; &#34; being launched, but already in LRU list&#34;);
        }

        // TODO(lifecycler): Resume or pause requests are done as part of launch transaction,
        // so updating the state should be done accordingly.
        if (andResume &amp;&amp; readyToResume()) {
            // As part of the process of launching, ActivityThread also performs
            // a resume.
            stack.minimalResumeActivityLocked(r);
        } else {
            // This activity is not starting in the resumed state... which should look like we asked
            // it to pause&#43;stop (but remain visible), and it has done so and reported back the
            // current icicle and other state.
            if (DEBUG_STATES) Slog.v(TAG_STATES,
                    &#34;Moving to PAUSED: &#34; &#43; r &#43; &#34; (starting in paused state)&#34;);
            r.setState(PAUSED, &#34;realStartActivityLocked&#34;);
        }

        // Launch the new version setup screen if needed.  We do this -after-
        // launching the initial activity (that is, home), so that it can have
        // a chance to initialize itself while in the background, making the
        // switch back to it faster and look better.
        if (isFocusedStack(stack)) {
            //当系统发生更新时&#xff0c;只会执行一次的用户向导
            mService.getActivityStartController().startSetupActivity();
        }

        // Update any services we are bound to that might care about whether
        // their client may have activities.
        if (r.app !&#61; null) {
            //更新所有与该Activity具有绑定关系的Service连接
            mService.mServices.updateServiceConnectionActivitiesLocked(r.app);
        }

        return true;
    }

</code></pre> 
<p>Android9.0之后&#xff0c;引入了ClientLifecycleManager和ClientTransactionHandler来辅助管理Activity的生命周期。</p> 
<p>一个生命周期都抽象出了一个对象。</p> 
<p>onCreate (LaunchActivityItem),onResume(ResumeActivityItem),</p> 
<p>onPause(PauseActivityItem)&#xff0c;onStop(StopActivityItem),onDestrory(DestroyActivityItem)</p> 
<h3><a id="218__CLMscheduleTransaction_3084"></a>2.18 CLM.scheduleTransaction</h3> 
<p>[-&gt;ClientLifecycleManager.java]</p> 
<p>将2.17节启动Activity生命周期的代码单独分析一下启动的过程</p> 
<pre><code>     //创建Activity启动事务
    final ClientTransaction clientTransaction &#61; ClientTransaction.obtain(app.thread,
            r.appToken);
    //设置事务callback&#xff0c;状态为onCreate ---&gt;1        
    clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
            System.identityHashCode(r), r.info,
            mergedConfiguration.getGlobalConfiguration(),
            mergedConfiguration.getOverrideConfiguration(), r.compat,
            r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
            r.persistentState, results, newIntents, mService.isNextTransitionForward(),
            profilerInfo));

    //设置目标事务的状态为onResume   ----&gt;2
    final ActivityLifecycleItem lifecycleItem;
    lifecycleItem &#61; ResumeActivityItem.obtain(mService.isNextTransitionForward());
    clientTransaction.setLifecycleStateRequest(lifecycleItem);
    
    //通过事务方式开始activity生命周期&#xff0c;onCreate,onStart,onResume   ----&gt;3
    mService.getLifecycleManager().scheduleTransaction(clientTransaction);
</code></pre> 
<h4><a id="2181_AMSgetLifecycleManager_3112"></a>2.18.1 AMS.getLifecycleManager</h4> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code> ClientLifecycleManager getLifecycleManager() {
        return mLifecycleManager;
    }
</code></pre> 
<h4><a id="2182_CLMscheduleTransaction_3122"></a>2.18.2 CLM.scheduleTransaction</h4> 
<p>[-&gt;ClientLifecycleManager.java]</p> 
<pre><code>  void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        final IApplicationThread client &#61; transaction.getClient();
        transaction.schedule();
        if (!(client instanceof Binder)) {
            // If client is not an instance of Binder - it&#39;s a remote call and at this point it is
            // safe to recycle the object. All objects used for local calls will be recycled after
            // the transaction is executed on client in ActivityThread.
            transaction.recycle();
        }
    }
</code></pre> 
<h4><a id="2183_CTschedule_3139"></a>2.18.3 CT.schedule</h4> 
<p>[-&gt;ClientTransaction.java]</p> 
<pre><code> public IApplicationThread getClient() {
        return mClient;
 }
 public void schedule() throws RemoteException {
        mClient.scheduleTransaction(this);
 }
</code></pre> 
<h4><a id="2184_ATscheduleTransaction_3152"></a>2.18.4 AT.scheduleTransaction</h4> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code>  &#64;Override
  public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        ActivityThread.this.scheduleTransaction(transaction);
  }
</code></pre> 
<p>ActivityThread继承ClientTransactionHandler</p> 
<p>[-&gt;ClientTransactionHandler.java]</p> 
<pre><code> /** Prepare and schedule transaction for execution. */
 void scheduleTransaction(ClientTransaction transaction) {
      transaction.preExecute(this);
      sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
 }
</code></pre> 
<h4><a id="2185_AThandleMessage_3175"></a>2.18.5 AT.handleMessage</h4> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code>public void handleMessage(Message msg) {
         ...
         case EXECUTE_TRANSACTION:
                    final ClientTransaction transaction &#61; (ClientTransaction) msg.obj;
                    mTransactionExecutor.execute(transaction);
                    if (isSystem()) {
                        // Client transactions inside system process are recycled on the client side
                        // instead of ClientLifecycleManager to avoid being cleared before this
                        // message is handled.
                        transaction.recycle();
                    }
                    // TODO(lifecycler): Recycle locally scheduled transactions.
                    break;
           ....
     }
</code></pre> 
<h4><a id="2186_TEexecute_3197"></a>2.18.6 TE.execute</h4> 
<p>[-&gt;TransactionExecutor.java]</p> 
<pre><code> /**
     * Resolve transaction.
     * First all callbacks will be executed in the order they appear in the list. If a callback
     * requires a certain pre- or post-execution state, the client will be transitioned accordingly.
     * Then the client will cycle to the final lifecycle state if provided. Otherwise, it will
     * either remain in the initial state, or last state needed by a callback.
     */
    public void execute(ClientTransaction transaction) {
        final IBinder token &#61; transaction.getActivityToken();
        log(&#34;Start resolving transaction for client: &#34; &#43; mTransactionHandler &#43; &#34;, token: &#34; &#43; token);
        //2.18节开始方法的第一步&#xff0c;状态为onCreate
        executeCallbacks(transaction);
        //2.18节开始方法的第二步&#xff0c;最后的状态为onResume
        executeLifecycleState(transaction);
        mPendingActions.clear();
        log(&#34;End resolving transaction&#34;);
    }
</code></pre> 
<p>[-&gt;TransactionExecutor.java]</p> 
<pre><code> public void executeCallbacks(ClientTransaction transaction) {
        final List&lt;ClientTransactionItem&gt; callbacks &#61; transaction.getCallbacks();
        if (callbacks &#61;&#61; null) {
            // No callbacks to execute, return early.
            return;
        }
        log(&#34;Resolving callbacks&#34;);

        final IBinder token &#61; transaction.getActivityToken();
        ActivityClientRecord r &#61; mTransactionHandler.getActivityClient(token);

        // In case when post-execution state of the last callback matches the final state requested
        // for the activity in this transaction, we won&#39;t do the last transition here and do it when
        // moving to final state instead (because it may contain additional parameters from server).
        final ActivityLifecycleItem finalStateRequest &#61; transaction.getLifecycleStateRequest();
        final int finalState &#61; finalStateRequest !&#61; null ? finalStateRequest.getTargetState()
                : UNDEFINED;
        // Index of the last callback that requests some post-execution state.
        final int lastCallbackRequestingState &#61; lastCallbackRequestingState(transaction);

        final int size &#61; callbacks.size();
        for (int i &#61; 0; i &lt; size; &#43;&#43;i) {
            final ClientTransactionItem item &#61; callbacks.get(i);
            log(&#34;Resolving callback: &#34; &#43; item);
            final int postExecutionState &#61; item.getPostExecutionState();
            final int closestPreExecutionState &#61; mHelper.getClosestPreExecutionState(r,
                    item.getPostExecutionState());
            if (closestPreExecutionState !&#61; UNDEFINED) {
                cycleToPath(r, closestPreExecutionState);
            }
            //将会执行execute方法
            item.execute(mTransactionHandler, token, mPendingActions);
            item.postExecute(mTransactionHandler, token, mPendingActions);
            if (r &#61;&#61; null) {
                // Launch activity request will create an activity record.
                r &#61; mTransactionHandler.getActivityClient(token);
            }

            if (postExecutionState !&#61; UNDEFINED &amp;&amp; r !&#61; null) {
                // Skip the very last transition and perform it by explicit state request instead.
                final boolean shouldExcludeLastTransition &#61;
                        i &#61;&#61; lastCallbackRequestingState &amp;&amp; finalState &#61;&#61; postExecutionState;
                cycleToPath(r, postExecutionState, shouldExcludeLastTransition);
            }
        }
    }
</code></pre> 
<p>[-&gt;LaunchActivityItem.java]</p> 
<pre><code> &#64;Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, &#34;activityStart&#34;);
        ActivityClientRecord r &#61; new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mIsForward,
                mProfilerInfo, client);
         //见2.19节&#xff0c;ActivityThread继承ClientTransactionHandler
        client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }
</code></pre> 
<h4><a id="2187_TEcycleToPath_3289"></a>2.18.7 TE.cycleToPath</h4> 
<p>[-&gt;TransactionExecutor.java]</p> 
<pre><code> /**
     * Transition the client between states with an option not to perform the last hop in the
     * sequence. This is used when resolving lifecycle state request, when the last transition must
     * be performed with some specific parameters.
     */
    private void cycleToPath(ActivityClientRecord r, int finish,
            boolean excludeLastState) {
        final int start &#61; r.getLifecycleState();
        log(&#34;Cycle from: &#34; &#43; start &#43; &#34; to: &#34; &#43; finish &#43; &#34; excludeLastState:&#34; &#43; excludeLastState);
        final IntArray path &#61; mHelper.getLifecyclePath(start, finish, excludeLastState);
        performLifecycleSequence(r, path);
    }

    /** Transition the client through previously initialized state sequence. */
    private void performLifecycleSequence(ActivityClientRecord r, IntArray path) {
        final int size &#61; path.size();
        for (int i &#61; 0, state; i &lt; size; i&#43;&#43;) {
            state &#61; path.get(i);
            log(&#34;Transitioning to state: &#34; &#43; state);
            switch (state) {
                case ON_CREATE:
                    mTransactionHandler.handleLaunchActivity(r, mPendingActions,
                            null /* customIntent */);
                    break;
                case ON_START:
                    mTransactionHandler.handleStartActivity(r, mPendingActions);
                    break;
                case ON_RESUME:
                    mTransactionHandler.handleResumeActivity(r.token, false /* finalStateRequest */,
                            r.isForward, &#34;LIFECYCLER_RESUME_ACTIVITY&#34;);
                    break;
                case ON_PAUSE:
                    mTransactionHandler.handlePauseActivity(r.token, false /* finished */,
                            false /* userLeaving */, 0 /* configChanges */, mPendingActions,
                            &#34;LIFECYCLER_PAUSE_ACTIVITY&#34;);
                    break;
                case ON_STOP:
                    mTransactionHandler.handleStopActivity(r.token, false /* show */,
                            0 /* configChanges */, mPendingActions, false /* finalStateRequest */,
                            &#34;LIFECYCLER_STOP_ACTIVITY&#34;);
                    break;
                case ON_DESTROY:
                    mTransactionHandler.handleDestroyActivity(r.token, false /* finishing */,
                            0 /* configChanges */, false /* getNonConfigInstance */,
                            &#34;performLifecycleSequence. cycling to:&#34; &#43; path.get(size - 1));
                    break;
                case ON_RESTART:
                    mTransactionHandler.performRestartActivity(r.token, false /* start */);
                    break;
                default:
                    throw new IllegalArgumentException(&#34;Unexpected lifecycle state: &#34; &#43; state);
            }
        }
    }
</code></pre> 
<p>cycleToPath将会执行从start,finish之间的周期方法。</p> 
<h3><a id="219_AThandleLaunchActivity_3352"></a>2.19 AT.handleLaunchActivity</h3> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code>    /**
     * Extended implementation of activity launch. Used when server requests a launch or relaunch.
     */
    &#64;Override
    public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();
        mSomeActivitiesChanged &#61; true;

        if (r.profilerInfo !&#61; null) {
            mProfiler.setProfiler(r.profilerInfo);
            mProfiler.startProfiling();
        }
        //回调目标Activity的onConfigurationChanged
        // Make sure we are running with the most recent config.
        handleConfigurationChanged(null, null);

        if (localLOGV) Slog.v(
            TAG, &#34;Handling launch of &#34; &#43; r);
     
        // Initialize before creating the activity
        if (!ThreadedRenderer.sRendererDisabled) {
            GraphicsEnvironment.earlyInitEGL();
        }
        WindowManagerGlobal.initialize();
         //回调目标Activity的onCreate&#xff0c;正式开始Activity的生命周期
        final Activity a &#61; performLaunchActivity(r, customIntent);

        if (a !&#61; null) {
            r.createdConfig &#61; new Configuration(mConfiguration);
            reportSizeConfigurations(r);
            if (!r.activity.mFinished &amp;&amp; pendingActions !&#61; null) {
                pendingActions.setOldState(r.state);
                pendingActions.setRestoreInstanceState(true);
                pendingActions.setCallOnPostCreate(true);
            }
        } else {
            //存在error,则停止该Activity
            // If there was an error, for any reason, tell the activity manager to stop us.
            try {
                ActivityManager.getService()
                        .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                                Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        }

        return a;
    }
</code></pre> 
<h3><a id="220_ATperformLaunchActivity_3411"></a>2.20 AT.performLaunchActivity</h3> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code>  private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo &#61; r.activityInfo;
        if (r.packageInfo &#61;&#61; null) {
            r.packageInfo &#61; getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

        ComponentName component &#61; r.intent.getComponent();
        if (component &#61;&#61; null) {
            component &#61; r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity !&#61; null) {
            component &#61; new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }

        ContextImpl appContext &#61; createBaseContextForActivity(r);
        Activity activity &#61; null;
        try {
            java.lang.ClassLoader cl &#61; appContext.getClassLoader();
            activity &#61; mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state !&#61; null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    &#34;Unable to instantiate activity &#34; &#43; component
                    &#43; &#34;: &#34; &#43; e.toString(), e);
            }
        }

        try {
            //创建Application对象
            Application app &#61; r.packageInfo.makeApplication(false, mInstrumentation);

            if (localLOGV) Slog.v(TAG, &#34;Performing launch of &#34; &#43; r);
            if (localLOGV) Slog.v(
                    TAG, r &#43; &#34;: app&#61;&#34; &#43; app
                    &#43; &#34;, appName&#61;&#34; &#43; app.getPackageName()
                    &#43; &#34;, pkg&#61;&#34; &#43; r.packageInfo.getPackageName()
                    &#43; &#34;, comp&#61;&#34; &#43; r.intent.getComponent().toShortString()
                    &#43; &#34;, dir&#61;&#34; &#43; r.packageInfo.getAppDir());

            if (activity !&#61; null) {
                CharSequence title &#61; r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config &#61; new Configuration(mCompatConfiguration);
                if (r.overrideConfig !&#61; null) {
                    config.updateFrom(r.overrideConfig);
                }
                if (DEBUG_CONFIGURATION) Slog.v(TAG, &#34;Launching activity &#34;
                        &#43; r.activityInfo.name &#43; &#34; with config &#34; &#43; config);
                Window window &#61; null;
                if (r.mPendingRemoveWindow !&#61; null &amp;&amp; r.mPreserveWindow) {
                    window &#61; r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow &#61; null;
                    r.mPendingRemoveWindowManager &#61; null;
                }
                appContext.setOuterContext(activity);
                //attach方法
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);

                if (customIntent !&#61; null) {
                    activity.mIntent &#61; customIntent;
                }
                r.lastNonConfigurationInstances &#61; null;
                checkAndBlockForNetworkAccess();
                activity.mStartedActivity &#61; false;
                int theme &#61; r.activityInfo.getThemeResource();
                if (theme !&#61; 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled &#61; false;
                //进入生命周期的onCreate
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        &#34;Activity &#34; &#43; r.intent.getComponent().toShortString() &#43;
                        &#34; did not call through to super.onCreate()&#34;);
                }
                r.activity &#61; activity;
            }
            r.setState(ON_CREATE);

            mActivities.put(r.token, r);

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    &#34;Unable to start activity &#34; &#43; component
                    &#43; &#34;: &#34; &#43; e.toString(), e);
            }
        }

        return activity;
    }
</code></pre> 
<h3><a id="221_callActivityOnCreate_3532"></a>2.21 callActivityOnCreate</h3> 
<p>[-&gt;Instrumentation.java]</p> 
<pre><code>   public void callActivityOnCreate(Activity activity, Bundle icicle) {
        prePerformCreate(activity);
        activity.performCreate(icicle);
        postPerformCreate(activity);
    }
</code></pre> 
<h3><a id="222_performCreate_3544"></a>2.22 performCreate</h3> 
<p>[-&gt;Activity.java]</p> 
<pre><code> &#64;UnsupportedAppUsage
    final void performCreate(Bundle icicle, PersistableBundle persistentState) {
        mCanEnterPictureInPicture &#61; true;
        restoreHasCurrentPermissionRequest(icicle);
        if (persistentState !&#61; null) {
            onCreate(icicle, persistentState);
        } else {
            onCreate(icicle);
        }
        writeEventLog(LOG_AM_ON_CREATE_CALLED, &#34;performCreate&#34;);
        mActivityTransitionState.readState(icicle);

        mVisibleFromClient &#61; !mWindow.getWindowStyle().getBoolean(
                com.android.internal.R.styleable.Window_windowNoDisplay, false);
        mFragments.dispatchActivityCreated();
        mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
    }
</code></pre> 
<p>到此&#xff0c;介绍了完成了Activity从onCreate,onStart,onResume的生命周期详细过程。</p> 
<h2><a id="_3570"></a>三、总结</h2> 
<p>本文从startActivity开始&#xff0c;详细分析了Activity的启动过程。</p> 
<ol><li>流程[2.1~2.3]:运行在调用者的进程当中&#xff0c;比如桌面启动Activity&#xff0c;则调用者所在的进程为launcher&#xff0c;launcher进程通过IActivityManager.aidl生成的代理类&#xff0c;进入到了systemserver进程&#xff08;AMS相应的Server端&#xff09;。</li><li>流程[2.3~2.17]:允许在systemserver系统进程中&#xff0c;这个过程为比较复杂且核心的过程&#xff0c;主要如下&#xff1a; 
  <ul><li>流程[2.7]:调用resolveActivity&#xff0c;通过PackageManager查询系统中所有符合要求的Activity&#xff0c;当存在多个满足多个条件的Activity则会让用户来选择 。</li><li>流程[2.8]:创建ActivityRecord对象&#xff0c;检查intent相关信息和权限&#xff0c;是否允许app切换&#xff0c;然后处理mPendingActivityLaunches中的Activity</li><li>流程[2.9]:为Activity找到或创建新的Task对象&#xff0c;设置flags信息</li><li>流程[2.13]:当没有处于任务栈中没有Activity时&#xff0c;直接回到桌面了;否则当mResumeActivity不为空是&#xff0c;先执行startPausingLocked方法暂停该Activity&#xff0c;然后进入startSpecificActivityLocked</li><li>流程[2.14]:当目标进程已经存在则直接进入2.17&#xff0c;当目标进程不存在时则创建进程&#xff0c;经过调用最后到2.17</li><li>流程[2.17]:systemserver进程通过IApplicationThread binder接口&#xff0c;进入到了目标进程。</li></ul> </li><li>流程[2.18~2.20]:允许在目标进程&#xff0c;通过Handler消息机制&#xff0c;该进程中的binder线程向主线程发送EXECUTE_TRANSACTION消息&#xff0c;进入事务处理阶段调用到handleLaunchActivity&#xff0c;最后通过发射创建的目标Activity&#xff0c;然后进入到onCreate生命周期。</li></ol> 
<p>从进程的角度分析</p> 
<ol><li>点击桌面图标&#xff0c;Launcher进程采用Binder IPC向systemserver进程发送startActivity请求</li><li>systemserver进程接收到请求后&#xff0c;向zygote进程发送创建进程的请求</li><li>zygote进程fork出新的子进程即app进程</li><li>App进程&#xff0c;通过Binder IPC向systemserver进程发起attachApplication请求</li><li>systemserver进程收到请求后&#xff0c;进行一系列准备工作后&#xff0c;在通过binderIPC进程发送scheduleTransaction请求</li><li>APP进程的binder进程&#xff08;ApplicationThread&#xff09;在收到请求后&#xff0c;通过handler向主线程发送EXECUTE_TRANSACTION消息。</li><li>主线程收到Message消息后&#xff0c;通过反射机制创建目标Activity&#xff0c;并回调Activity的onCreate方法。</li></ol> 
<p>到此应用正式启动&#xff0c;开始进入应用的生命周期&#xff0c;执行完onCreate&#xff0c;onStart&#xff0c;onResume方法&#xff0c;makeVisible&#xff0c;UI渲染结束后就可以看到App的主界面了。</p> 
<h2><a id="_3596"></a>附录</h2> 
<p>源码路径</p> 
<pre><code>frameworks/base/core/java/android/content/ContextWrapper.java
frameworks/base/core/java/android/app/ContextImpl.java
frameworks/base/core/java/android/app/Instrumentation.java
frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
frameworks/base/services/core/java/com/android/server/am/ActivityStartController.java
frameworks/base/services/core/java/com/android/server/am/ActivityStarter.java
frameworks/base/services/core/java/com/android/server/am/ActivityStack.java
frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java
frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java  
</code></pre>