 ---
layout:     post
title:      Android10 ContentProvider原理分析
subtitle:   ContentProvider用于提供数据的统一访问格式，封装具体的实现。对于数据的使用无需知道是数据库、文件、网络
date:       2020-08-12
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
<p>ContentProvider用于提供数据的统一访问格式&#xff0c;封装具体的实现。对于数据的使用无需知道是数据库、文件、网络&#xff0c;只需要使用ContentProvider的数据操作接口&#xff0c;即增&#xff08;insert&#xff09;删&#xff08;delete&#xff09;改&#xff08;update&#xff09;查&#xff08;query&#xff09;。</p> 
<h3><a id="11_ContentProvider_7"></a>1.1 ContentProvider</h3> 
<p>ContentProvider作为四大组件之一&#xff0c;没有Activity复杂的生命周期&#xff0c;只有简单的onCreate过程。ContentProvider是一个抽象类&#xff0c;当实现自己的ContentProvider类&#xff0c;需要继承ContentProvider&#xff0c;并且实现下面6个抽象即可&#xff1a;</p> 
<pre><code>insert(Uri, ContentValues)&#xff1a;插入新数据&#xff1b;
delete(Uri, String, String[])&#xff1a;删除已有数据&#xff1b;
update(Uri, ContentValues, String, String[])&#xff1a;更新数据&#xff1b;
query(Uri, String[], String, String[], String)&#xff1a;查询数据&#xff1b;
onCreate()&#xff1a;执行初始化工作&#xff1b;
getType(Uri)&#xff1a;获取数据MIME类型。
</code></pre> 
<p>Uri数据格式如下&#xff1a;<code>content://com.skytoby.articles/android/1</code></p> 
<table><thead><tr><th>字段</th><th>含义</th><th>对应项</th></tr></thead><tbody><tr><td>前缀</td><td>默认的固定开头格式</td><td>content://</td></tr><tr><td>授权</td><td>唯一表示provider</td><td>com.skytoby.articles</td></tr><tr><td>路径</td><td>数据类别以及数据项</td><td>/android/1</td></tr></tbody></table>
<h3><a id="12_ContentResolver_28"></a>1.2 ContentResolver</h3> 
<p>其他进程想要操作ContentProvider&#xff0c;需要获取其对应的ContentResolver&#xff0c;利用ContentResolver类完成对数据的增删改查操作&#xff0c;下面列举一个查询的操作。</p> 
<pre><code>//获取ContentResolver
ContentResolver cr &#61; getContentResolver();  
Uri uri &#61; Uri.parse(&#34;content://com.skytoby.articles/android/1&#34;);
Cursor cursor &#61; cr.query(uri, null, null, null, null);  //执行查询操作
...
cursor.close(); //关闭
</code></pre> 
<h3><a id="13___41"></a>1.3 类图</h3> 
<p><img src="https://img-blog.csdnimg.cn/20200105192951555.jpg?x-oss-process&#61;image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Nhbzg2MTU0NDMyNQ&#61;&#61;,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" /></p> 
<p>CPP和CPN是Binder通信的C/S两端</p> 
<p>ACR&#xff08;ApplicationContentResolver&#xff09;继承于ContentResolver&#xff0c;是ContextImpl的内部类&#xff0c;ACR的实现通过调用其成员变量mMainThread来实现。</p> 
<h3><a id="14__48"></a>1.4 重要成员变量</h3> 
<table><thead><tr><th align="left">类名</th><th align="left">成员变量</th><th align="left">含义</th></tr></thead><tbody><tr><td align="left">AMS</td><td align="left">CONTENT_PROVIDER_PUBLISH_TIMEOUT</td><td align="left">默认值为10s</td></tr><tr><td align="left">AMS</td><td align="left">mProviderMap</td><td align="left">记录所有contentProvider</td></tr><tr><td align="left">AMS</td><td align="left">mLaunchingProviders</td><td align="left">记录存在客户端等待publish的ContentProviderRecord</td></tr><tr><td align="left">PR</td><td align="left">pubProviders</td><td align="left">该进程创建的ContentProviderRecord</td></tr><tr><td align="left">PR</td><td align="left">conProviders</td><td align="left">该进程使用的ContentProviderConnection</td></tr><tr><td align="left">AT</td><td align="left">mLocalProviders</td><td align="left">记录所有本地的ContentProvider&#xff0c;以IBinder以key</td></tr><tr><td align="left">AT</td><td align="left">mLocalProvidersByName</td><td align="left">记录所有本地的ContentProvider&#xff0c;以组件名为key</td></tr><tr><td align="left">AT</td><td align="left">mProviderMap</td><td align="left">记录该进程的contentProvider</td></tr><tr><td align="left">AT</td><td align="left">mProviderRefCountMap</td><td align="left">记录所有对其他进程中的ContentProvider的引用计数</td></tr></tbody></table>
<ul><li><code>CONTENT_PROVIDER_PUBLISH_TIMEOUT</code>(10s): provider所在进程发布其ContentProvider的超时时长为10s&#xff0c;超过10s则会系统所杀;</li><li><code>mProviderMap</code>&#xff1a; AMS和AT都有一个同名的成员变量, AMS的数据类型为ProviderMap,而AT则是以ProviderKey为key的ArrayMap类型;</li><li><code>mLaunchingProviders</code>&#xff1a;记录的每一项是一个ContentProviderRecord对象, 所有的存在client等待其发布完成的contentProvider列表&#xff0c;一旦发布完成则相应的contentProvider便会从该列表移除&#xff1b;</li><li>PR:ProcessRecord, AT: ActivityThread&#xff1b;</li><li><code>mLocalProviders</code>和<code>mLocalProvidersByName</code>&#xff1a;都是用于记录所有本地的ContentProvider,不同的只是key。</li></ul> 
<h3><a id="15_query_68"></a>1.5 query流程图</h3> 
<p><img src="https://img-blog.csdnimg.cn/20200105193016177.jpg?x-oss-process&#61;image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Nhbzg2MTU0NDMyNQ&#61;&#61;,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" /></p> 
<h2><a id="ContentProvider_72"></a>二、发布ContentProvider</h2> 
<p>通过ContentProvider共享数据时&#xff0c;需要编写一个类继承ContentProvier&#xff0c;创建数据库&#xff0c;并在AndroidManifest声明该Provider&#xff0c;这样其他的进行就可以通过ContentResolver去查询共享的信息。先来看一下应用时如何发布ContentProvider&#xff0c;提供给其他应用使用。ContentProvider一般是在应用进程启动的时候启动&#xff0c;是四大组件中最早启动的。进程的启动在四大组件与进程启动那里有详细的分析。</p> 
<p>发布ContentProvider分两种情况&#xff1a;Provider进程未启动&#xff0c;Provider进程已经启动但未发布。</p> 
<ul><li> <p>Provider进程未启动</p> <p>systemserver进程调用startProcessLocked创建provider进程&#xff0c;并attach到systemserver后&#xff0c;通过binder机制到Provider进程执行bindApplication方法&#xff0c;见2.1节。</p> </li><li> <p>Provider进程已经启动但未发布</p> <p>如果发现provider进程已经存在且attach到systemserver&#xff0c;但对应的provider还没有发布&#xff0c;</p> <p>则通过binder机制到provider进程执行scheduleInstallProvider方法&#xff0c;见2.7节。</p> <p>这两种情况最后都会走到installProvider这个方法。</p> </li></ul> 
<h3><a id="21_Provider_91"></a>2.1 Provider进程未启动</h3> 
<h4><a id="211__ATbindApplication_93"></a>2.1.1 AT.bindApplication</h4> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code>  public final void bindApplication(String processName, ApplicationInfo appInfo,
                List&lt;ProviderInfo&gt; providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableBinderTracking, boolean trackAllocation,
                boolean isRestrictedBackupMode, boolean persistent, Configuration config,
                CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
                String buildSerial, boolean autofillCompatibilityEnabled) {
                
            AppBindData data &#61; new AppBindData();
            data.processName &#61; processName;
            data.appInfo &#61; appInfo;
            //provider
            data.providers &#61; providers;
            ...
            sendMessage(H.BIND_APPLICATION, data);
        }
</code></pre> 
<p>发送BIND_APPLICATION消息到主线程。</p> 
<h4><a id="212__AThandleMessage_120"></a>2.1.2 AT.handleMessage</h4> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code> public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, &#34;&gt;&gt;&gt; handling: &#34; &#43; codeToString(msg.what));
            switch (msg.what) {
                case BIND_APPLICATION:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, &#34;bindApplication&#34;);
                    AppBindData data &#61; (AppBindData)msg.obj;
                    handleBindApplication(data);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                }
                ...
   }             
</code></pre> 
<p>主线程收到BIND_APPLICATION后&#xff0c;执行handleBindApplication方法。</p> 
<h4><a id="213__AThandleMessage_141"></a>2.1.3 AT.handleMessage</h4> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code> private void handleBindApplication(AppBindData data) {
         
        // send up app name; do this *before* waiting for debugger
        //设置进程名
        Process.setArgV0(data.processName);
        android.ddm.DdmHandleAppName.setAppName(data.processName,
                                                UserHandle.myUserId());
        VMRuntime.setProcessPackageName(data.appInfo.packageName);
        ...
        // Allow disk access during application and provider setup. This could
        // block processing ordered broadcasts, but later processing would
        // probably end up doing the same disk access.
        Application app;
        final StrictMode.ThreadPolicy savedPolicy &#61; StrictMode.allowThreadDiskWrites();
        final StrictMode.ThreadPolicy writesAllowedPolicy &#61; StrictMode.getThreadPolicy();
        try {
        
            // If the app is being launched for full backup or restore, bring it up in
            // a restricted environment with the base application class.
            //实例化Application,会调用attachBaseContext方法
            app &#61; data.info.makeApplication(data.restrictedBackupMode, null);
            
            // Propagate autofill compat state
            app.setAutofillCompatibilityEnabled(data.autofillCompatibilityEnabled);
            mInitialApplication &#61; app;
            //不是限制备份模式
            // don&#39;t bring up providers in restricted mode; they may depend on the
            // app&#39;s custom Application class
            if (!data.restrictedBackupMode) {
                if (!ArrayUtils.isEmpty(data.providers)) {
                    //安装ContentProvider&#xff0c;见2.4节
                    installContentProviders(app, data.providers);
                }
            }
            try {
                //调用Application的onCreate方法
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                if (!mInstrumentation.onException(app, e)) {
                    throw new RuntimeException(
                      &#34;Unable to create application &#34; &#43; app.getClass().getName()
                      &#43; &#34;: &#34; &#43; e.toString(), e);
                }
            }
        } 
        // 加载字体资源
        // Preload fonts resources
        FontsContract.setApplicationContextForResources(appContext);
        if (!Process.isIsolated()) {
            try {
                final ApplicationInfo info &#61;
                        getPackageManager().getApplicationInfo(
                                data.appInfo.packageName,
                                PackageManager.GET_META_DATA /*flags*/,
                                UserHandle.myUserId());
                if (info.metaData !&#61; null) {
                    final int preloadedFontsResource &#61; info.metaData.getInt(
                            ApplicationInfo.METADATA_PRELOADED_FONTS, 0);
                    if (preloadedFontsResource !&#61; 0) {
                        data.info.getResources().preloadFonts(preloadedFontsResource);
                    }
                }
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
    }

</code></pre> 
<h4><a id="214__ATinstallContentProviders_216"></a>2.1.4 AT.installContentProviders</h4> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code>&#64;UnsupportedAppUsage
    private void installContentProviders(
            Context context, List&lt;ProviderInfo&gt; providers) {
        final ArrayList&lt;ContentProviderHolder&gt; results &#61; new ArrayList&lt;&gt;();

        for (ProviderInfo cpi : providers) {
            if (DEBUG_PROVIDER) {
                StringBuilder buf &#61; new StringBuilder(128);
                buf.append(&#34;Pub &#34;);
                buf.append(cpi.authority);
                buf.append(&#34;: &#34;);
                buf.append(cpi.name);
                Log.i(TAG, buf.toString());
            }
            //安装provider&#xff0c;见2.1.5节
            ContentProviderHolder cph &#61; installProvider(context, null, cpi,
                    false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
            if (cph !&#61; null) {
                cph.noReleaseNeeded &#61; true;
                results.add(cph);
            }
        }

        try {
            //发布provider见2.2.1节
            ActivityManager.getService().publishContentProviders(
                getApplicationThread(), results);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
</code></pre> 
<h4><a id="215__ATinstallProvider_254"></a>2.1.5 AT.installProvider</h4> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code>  private ContentProviderHolder installProvider(Context context,
            ContentProviderHolder holder, ProviderInfo info,
            boolean noisy, boolean noReleaseNeeded, boolean stable) {
        ContentProvider localProvider &#61; null;
        IContentProvider provider;
        if (holder &#61;&#61; null || holder.provider &#61;&#61; null) {
            if (DEBUG_PROVIDER || noisy) {
                Slog.d(TAG, &#34;Loading provider &#34; &#43; info.authority &#43; &#34;: &#34;
                        &#43; info.name);
            }
            Context c &#61; null;
            ApplicationInfo ai &#61; info.applicationInfo;
            //赋值context
            if (context.getPackageName().equals(ai.packageName)) {
                c &#61; context;
            } else if (mInitialApplication !&#61; null &amp;&amp;
                    mInitialApplication.getPackageName().equals(ai.packageName)) {
                c &#61; mInitialApplication;
            } else {
                try {
                    c &#61; context.createPackageContext(ai.packageName,
                            Context.CONTEXT_INCLUDE_CODE);
                } catch (PackageManager.NameNotFoundException e) {
                    // Ignore
                }
            }
            //无法获取context则直接返回
            if (c &#61;&#61; null) {
                Slog.w(TAG, &#34;Unable to get context for package &#34; &#43;
                      ai.packageName &#43;
                      &#34; while loading content provider &#34; &#43;
                      info.name);
                return null;
            }

            if (info.splitName !&#61; null) {
                try {
                    c &#61; c.createContextForSplit(info.splitName);
                } catch (NameNotFoundException e) {
                    throw new RuntimeException(e);
                }
            }

            try {
                final java.lang.ClassLoader cl &#61; c.getClassLoader();
                LoadedApk packageInfo &#61; peekPackageInfo(ai.packageName, true);
                if (packageInfo &#61;&#61; null) {
                    // System startup case.
                    packageInfo &#61; getSystemContext().mPackageInfo;
                }
                //通过反射&#xff0c;创建目标ContentProvider对象
                localProvider &#61; packageInfo.getAppFactory()
                        .instantiateProvider(cl, info.name);
                //获取contentprovider        
                provider &#61; localProvider.getIContentProvider();
                if (provider &#61;&#61; null) {
                    Slog.e(TAG, &#34;Failed to instantiate class &#34; &#43;
                          info.name &#43; &#34; from sourceDir &#34; &#43;
                          info.applicationInfo.sourceDir);
                    return null;
                }
                if (DEBUG_PROVIDER) Slog.v(
                    TAG, &#34;Instantiating local provider &#34; &#43; info.name);
                // XXX Need to create the correct context for this provider.
                //回调目标 ContentProvider.this.onCreate()方法
                localProvider.attachInfo(c, info);
            } catch (java.lang.Exception e) {
                if (!mInstrumentation.onException(null, e)) {
                    throw new RuntimeException(
                            &#34;Unable to get provider &#34; &#43; info.name
                            &#43; &#34;: &#34; &#43; e.toString(), e);
                }
                return null;
            }
        } else {
            provider &#61; holder.provider;
            if (DEBUG_PROVIDER) Slog.v(TAG, &#34;Installing external provider &#34; &#43; info.authority &#43; &#34;: &#34;
                    &#43; info.name);
        }
        
        //retHolder赋值&#xff0c;用于将contentProvider放入缓存
        ContentProviderHolder retHolder;

        synchronized (mProviderMap) {
            if (DEBUG_PROVIDER) Slog.v(TAG, &#34;Checking to add &#34; &#43; provider
                    &#43; &#34; / &#34; &#43; info.name);
            IBinder jBinder &#61; provider.asBinder();
            if (localProvider !&#61; null) {
                ComponentName cname &#61; new ComponentName(info.packageName, info.name);
                ProviderClientRecord pr &#61; mLocalProvidersByName.get(cname);
                if (pr !&#61; null) {
                    if (DEBUG_PROVIDER) {
                        Slog.v(TAG, &#34;installProvider: lost the race, &#34;
                                &#43; &#34;using existing local provider&#34;);
                    }
                    provider &#61; pr.mProvider;
                } else {
                    holder &#61; new ContentProviderHolder(info);
                    holder.provider &#61; provider;
                    holder.noReleaseNeeded &#61; true;
                    pr &#61; installProviderAuthoritiesLocked(provider, localProvider, holder);
                    mLocalProviders.put(jBinder, pr);
                    mLocalProvidersByName.put(cname, pr);
                }
                retHolder &#61; pr.mHolder;
            } else {
                ProviderRefCount prc &#61; mProviderRefCountMap.get(jBinder);
                if (prc !&#61; null) {
                    if (DEBUG_PROVIDER) {
                        Slog.v(TAG, &#34;installProvider: lost the race, updating ref count&#34;);
                    }
                    // We need to transfer our new reference to the existing
                    // ref count, releasing the old one...  but only if
                    // release is needed (that is, it is not running in the
                    // system process).
                    //不再需要则remove
                    if (!noReleaseNeeded) {
                        incProviderRefLocked(prc, stable);
                        try {
                            ActivityManager.getService().removeContentProvider(
                                    holder.connection, stable);
                        } catch (RemoteException e) {
                            //do nothing content provider object is dead any way
                        }
                    }
                } else {
                    ProviderClientRecord client &#61; installProviderAuthoritiesLocked(
                            provider, localProvider, holder);
                    if (noReleaseNeeded) {
                        prc &#61; new ProviderRefCount(holder, client, 1000, 1000);
                    } else {
                        prc &#61; stable
                                ? new ProviderRefCount(holder, client, 1, 0)
                                : new ProviderRefCount(holder, client, 0, 1);
                    }
                    mProviderRefCountMap.put(jBinder, prc);
                }
                retHolder &#61; prc.holder;
            }
        }
        return retHolder;
    }
</code></pre> 
<p>这里主要是通过反射获取获取contentprovider&#xff0c;并回调其onCreate方法&#xff0c;最后对ContentProviderHolder进行赋值并返回。</p> 
<h4><a id="216__AMSpublishContentProviders_405"></a>2.1.6 AMS.publishContentProviders</h4> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code>public final void publishContentProviders(IApplicationThread caller,
            List&lt;ContentProviderHolder&gt; providers) {
        if (providers &#61;&#61; null) {
            return;
        }

        enforceNotIsolatedCaller(&#34;publishContentProviders&#34;);
        synchronized (this) {
            final ProcessRecord r &#61; getRecordForAppLocked(caller);
            if (DEBUG_MU) Slog.v(TAG_MU, &#34;ProcessRecord uid &#61; &#34; &#43; r.uid);
            if (r &#61;&#61; null) {
                throw new SecurityException(
                        &#34;Unable to find app for caller &#34; &#43; caller
                      &#43; &#34; (pid&#61;&#34; &#43; Binder.getCallingPid()
                      &#43; &#34;) when publishing content providers&#34;);
            }

            final long origId &#61; Binder.clearCallingIdentity();

            final int N &#61; providers.size();
            for (int i &#61; 0; i &lt; N; i&#43;&#43;) {
                ContentProviderHolder src &#61; providers.get(i);
                if (src &#61;&#61; null || src.info &#61;&#61; null || src.provider &#61;&#61; null) {
                    continue;
                }
                ContentProviderRecord dst &#61; r.pubProviders.get(src.info.name);
                if (DEBUG_MU) Slog.v(TAG_MU, &#34;ContentProviderRecord uid &#61; &#34; &#43; dst.uid);
                if (dst !&#61; null) {
                    ComponentName comp &#61; new ComponentName(dst.info.packageName, dst.info.name);
                    //将该provider添加到mProviderMap
                    mProviderMap.putProviderByClass(comp, dst);
                    String names[] &#61; dst.info.authority.split(&#34;;&#34;);
                    for (int j &#61; 0; j &lt; names.length; j&#43;&#43;) {
                        mProviderMap.putProviderByName(names[j], dst);
                    }

                    int launchingCount &#61; mLaunchingProviders.size();
                    int j;
                    boolean wasInLaunchingProviders &#61; false;
                    for (j &#61; 0; j &lt; launchingCount; j&#43;&#43;) {
                        if (mLaunchingProviders.get(j) &#61;&#61; dst) {
                            mLaunchingProviders.remove(j);
                            wasInLaunchingProviders &#61; true;
                            j--;
                            launchingCount--;
                        }
                    }
                    if (wasInLaunchingProviders) {
                        //移除超时消息
                        mHandler.removeMessages(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG, r);
                    }
                    synchronized (dst) {
                        dst.provider &#61; src.provider;
                        dst.proc &#61; r;
                        //唤醒客户端的wait等待方法
                        dst.notifyAll();
                    }
                    //更新adj
                    updateOomAdjLocked(r, true);
                    maybeUpdateProviderUsageStatsLocked(r, src.info.packageName,
                            src.info.authority);
                }
            }

            Binder.restoreCallingIdentity(origId);
        }
    }
</code></pre> 
<p>ContentProviders一旦publish成功&#xff0c;则会移除超时发布的消息&#xff0c;并调用notifyAll来唤醒所有等待client端进程。</p> 
<h3><a id="22_Provider_481"></a>2.2 Provider进程启动但未发布</h3> 
<p>下面在看下Provider进程未发布的情况&#xff0c;。。。。。。。。。。。。。。。。。。。。。。</p> 
<h4><a id="221__ATscheduleInstallProvider_485"></a>2.2.1 AT.scheduleInstallProvider</h4> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code>        &#64;Override
        public void scheduleInstallProvider(ProviderInfo provider) {
            sendMessage(H.INSTALL_PROVIDER, provider);
        }
        public void handleMessage(Message msg) {
              ...
              case INSTALL_PROVIDER:
                    handleInstallProvider((ProviderInfo) msg.obj);
                    break;
              ...
         }
</code></pre> 
<h4><a id="222__AThandleInstallProvider_503"></a>2.2.2 AT.handleInstallProvider</h4> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code> public void handleInstallProvider(ProviderInfo info) {
        final StrictMode.ThreadPolicy oldPolicy &#61; StrictMode.allowThreadDiskWrites();
        try {
            installContentProviders(mInitialApplication, Arrays.asList(info));
        } finally {
            StrictMode.setThreadPolicy(oldPolicy);
        }
    }

</code></pre> 
<p>这个方法之后就是2.1.4节的流程installContentProviders</p> 
<h2><a id="ContentResolver_521"></a>三、查询ContentResolver</h2> 
<p>一般用ContentProvider的时候&#xff0c;会先得到ContentResolver&#xff0c;之后通过uri可以获取相应的信息&#xff0c;下面就从查询方法开始&#xff0c;分析这个过程的源码流程。</p> 
<pre><code>ContentResolver contentResolver &#61; getContentResolver();
Uri uri &#61; Uri.parse(&#34;content://com.skytoby.articles/android/1&#34;);
Cursor cursor &#61; contentResolver.query(uri, null, null, null, null);
</code></pre> 
<h3><a id="31_CIgetContentResolver_531"></a>3.1 CI.getContentResolver</h3> 
<p>[-&gt;ContextImpl.java]</p> 
<pre><code>    &#64;Override
    public ContentResolver getContentResolver() {
        return mContentResolver;
    }
    
    private ContextImpl(&#64;Nullable ContextImpl container, &#64;NonNull ActivityThread mainThread,
            &#64;NonNull LoadedApk packageInfo, &#64;Nullable String splitName,
            &#64;Nullable IBinder activityToken, &#64;Nullable UserHandle user, int flags,
            &#64;Nullable ClassLoader classLoader) {
        ...
        mContentResolver &#61; new ApplicationContentResolver(this, mainThread);
    }

</code></pre> 
<p>Context中调用getContentResolver&#xff0c;经过层层调用到ContextImpl&#xff0c;返回是在ContextImpl对象创建过程中完成赋值的。下面看下查询的操作。</p> 
<h3><a id="32_CRquery_553"></a>3.2 CR.query</h3> 
<p>[-&gt;ContentResolver.java]</p> 
<pre><code>    public final &#64;Nullable Cursor query(&#64;RequiresPermission.Read &#64;NonNull Uri uri,
            &#64;Nullable String[] projection, &#64;Nullable String selection,
            &#64;Nullable String[] selectionArgs, &#64;Nullable String sortOrder) {
        return query(uri, projection, selection, selectionArgs, sortOrder, null);
    }
    
    public final &#64;Nullable Cursor query(final &#64;RequiresPermission.Read &#64;NonNull Uri uri,
            &#64;Nullable String[] projection, &#64;Nullable Bundle queryArgs,
            &#64;Nullable CancellationSignal cancellationSignal) {
        Preconditions.checkNotNull(uri, &#34;uri&#34;);
        //获取unstableProvider见3.3节
        IContentProvider unstableProvider &#61; acquireUnstableProvider(uri);
        if (unstableProvider &#61;&#61; null) {
            return null;
        }
        IContentProvider stableProvider &#61; null;
        Cursor qCursor &#61; null;
        try {
            long startTime &#61; SystemClock.uptimeMillis();

            ICancellationSignal remoteCancellationSignal &#61; null;
            if (cancellationSignal !&#61; null) {
                cancellationSignal.throwIfCanceled();
                remoteCancellationSignal &#61; unstableProvider.createCancellationSignal();
                cancellationSignal.setRemote(remoteCancellationSignal);
            }
            try {
                //执行查询操作
                qCursor &#61; unstableProvider.query(mPackageName, uri, projection,
                        queryArgs, remoteCancellationSignal);
            } catch (DeadObjectException e) {
                // The remote process has died...  but we only hold an unstable
                // reference though, so we might recover!!!  Let&#39;s try!!!!
                // This is exciting!!1!!1!!!!1
                //远程进程死亡&#xff0c;处理unstableProvider死亡过程
                unstableProviderDied(unstableProvider);
                //unstable类型死亡后&#xff0c;在创建stable类型的provider&#xff0c;见3.5节
                stableProvider &#61; acquireProvider(uri);
                if (stableProvider &#61;&#61; null) {
                    return null;
                }
                //再次执行查询操作
                qCursor &#61; stableProvider.query(
                        mPackageName, uri, projection, queryArgs, remoteCancellationSignal);
            }
            if (qCursor &#61;&#61; null) {
                return null;
            }
            //强制执行操作&#xff0c;可能会失败并抛出RuntimeException
            // Force query execution.  Might fail and throw a runtime exception here.
            qCursor.getCount();
            long durationMillis &#61; SystemClock.uptimeMillis() - startTime;
            maybeLogQueryToEventLog(durationMillis, uri, projection, queryArgs);
          
            //创建CursorWrapperInner对象
            // Wrap the cursor object into CursorWrapperInner object.
            final IContentProvider provider &#61; (stableProvider !&#61; null) ? stableProvider
                    : acquireProvider(uri);
            final CursorWrapperInner wrapper &#61; new CursorWrapperInner(qCursor, provider);
            stableProvider &#61; null;
            qCursor &#61; null;
            return wrapper;
        } catch (RemoteException e) {
            // Arbitrary and not worth documenting, as Activity
            // Manager will kill this process shortly anyway.
            return null;
        } finally {
            if (qCursor !&#61; null) {
                qCursor.close();
            }
            if (cancellationSignal !&#61; null) {
                cancellationSignal.setRemote(null);
            }
            if (unstableProvider !&#61; null) {
                releaseUnstableProvider(unstableProvider);
            }
            if (stableProvider !&#61; null) {
                releaseProvider(stableProvider);
            }
        }
    }

</code></pre> 
<p>一般获取unstable的Provider&#xff1a;调用acquireUnstableProvider&#xff0c;尝试获取unstable的ContentProvider&#xff1b;然后执行query操作</p> 
<p>当执行query操作过程中抛出DeadObjectException&#xff0c;即ContentProvider所在的进程死亡&#xff0c;则尝试获取stable的ContentProvider:</p> 
<p>1.先调用unstableProviderDied&#xff0c;清理刚创建的unstable的ContentProvider</p> 
<p>2.调用acquireProvider&#xff0c;尝试获取stable的ContentProvider&#xff0c;此时当ContentProvider进程死亡&#xff0c;则会杀掉该ContentProvider的客户端进程&#xff1b;</p> 
<p>3.执行query操作。</p> 
<p>stable和unstable的区别&#xff1a;</p> 
<p>采用unstable类型的ContentProvider的应用不会因为远程ContentProvider进程的死亡而被杀。</p> 
<p>采用stable类型的ContentProvider的应用会因为远程ContentProvider进程的死亡而被杀。</p> 
<p>对于应用无法决定创建的ContentProvider是stable还是unstable的&#xff0c;也无法知道自己的进程是否依赖于远程进程的生死。</p> 
<h3><a id="33_CRacquireUnstableProvider_660"></a>3.3 CR.acquireUnstableProvider</h3> 
<p>[-&gt;ContentResolver.java]</p> 
<pre><code> /**
     * Returns the content provider for the given content URI.
     *
     * &#64;param uri The URI to a content provider
     * &#64;return The ContentProvider for the given URI, or null if no content provider is found.
     * &#64;hide
     */
    public final IContentProvider acquireUnstableProvider(Uri uri) {
        if (!SCHEME_CONTENT.equals(uri.getScheme())) {
            return null;
        }
        String auth &#61; uri.getAuthority();
        if (auth !&#61; null) {
            //获取UnstableProvider,见3.4节
            return acquireUnstableProvider(mContext, uri.getAuthority());
        }
        return null;
    }

</code></pre> 
<h3><a id="34_ACRacquireUnstableProvider_686"></a>3.4 ACR.acquireUnstableProvider</h3> 
<p>[-&gt;ContextImpl.java::ApplicationContentResolver]</p> 
<pre><code>private static final class ApplicationContentResolver extends ContentResolver {
     ...
    private static final class ApplicationContentResolver extends ContentResolver {
        &#64;UnsupportedAppUsage
        private final ActivityThread mMainThread;
        ...
        //stable provider
        &#64;Override
        &#64;UnsupportedAppUsage
        protected IContentProvider acquireProvider(Context context, String auth) {
            return mMainThread.acquireProvider(context,
                    ContentProvider.getAuthorityWithoutUserId(auth),
                    resolveUserIdFromAuthority(auth), true);
        }
        //unstable provider
        &#64;Override
        protected IContentProvider acquireUnstableProvider(Context c, String auth) {
            return mMainThread.acquireProvider(c,
                    ContentProvider.getAuthorityWithoutUserId(auth),
                    resolveUserIdFromAuthority(auth), false);
        }
    }
}
</code></pre> 
<p>从上面可以看出&#xff0c;无论是acquireProvider还是acquireUnstableProvider方法&#xff0c;最后调用的都市ActivityThread的同一个方法的acquireProvider。getAuthorityWithoutUserId是字符截断过程&#xff0c;即去掉auth中的userid信息&#xff0c;比如com.skytoby.article&#64;666,经过该方法则变成了com.skytoby.article。</p> 
<h3><a id="35_ATacquireProvider_718"></a>3.5 AT.acquireProvider</h3> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code>  public final IContentProvider acquireProvider(
            Context c, String auth, int userId, boolean stable) {
        //获取已经存在的provider    
        final IContentProvider provider &#61; acquireExistingProvider(c, auth, userId, stable);
        if (provider !&#61; null) {
            //成功则返回
            return provider;
        }

        // There is a possible race here.  Another thread may try to acquire
        // the same provider at the same time.  When this happens, we want to ensure
        // that the first one wins.
        // Note that we cannot hold the lock while acquiring and installing the
        // provider since it might take a long time to run and it could also potentially
        // be re-entrant in the case where the provider is in the same process.
        ContentProviderHolder holder &#61; null;
        try {
            synchronized (getGetProviderLock(auth, userId)) {
                //见3.6节
                holder &#61; ActivityManager.getService().getContentProvider(
                        getApplicationThread(), auth, userId, stable);
            }
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
        if (holder &#61;&#61; null) {
            Slog.e(TAG, &#34;Failed to find provider info for &#34; &#43; auth);
            return null;
        }
        
        // 安装provider将会增加引用计数&#xff0c;见3.8节
        // Install provider will increment the reference count for us, and break
        // any ties in the race.
        holder &#61; installProvider(c, holder, holder.info,
                true /*noisy*/, holder.noReleaseNeeded, stable);
        return holder.provider;
    }
</code></pre> 
<p>主要过程&#xff1a;1.获取已经存在的provider,如果存在则返回&#xff0c;否则继续执行&#xff1b;</p> 
<p>​ 2.通过AMS获取provider&#xff0c;无法获取则返回&#xff0c;否则继续执行&#xff1b;</p> 
<p>​ 3.通过installProvider方法安装provider&#xff0c;并增加该provider的引用计数。</p> 
<h4><a id="351_ATacquireExistingProvider_768"></a>3.5.1 AT.acquireExistingProvider</h4> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code>    &#64;UnsupportedAppUsage
    public final IContentProvider acquireExistingProvider(
            Context c, String auth, int userId, boolean stable) {
        synchronized (mProviderMap) {
            final ProviderKey key &#61; new ProviderKey(auth, userId);
            //从mProviderMap中查询是否存在&#xff0c;在2.1.6节发布时将provider添加到了mProviderMap
            final ProviderClientRecord pr &#61; mProviderMap.get(key);
            if (pr &#61;&#61; null) {
                return null;
            }

            IContentProvider provider &#61; pr.mProvider;
            IBinder jBinder &#61; provider.asBinder();
            if (!jBinder.isBinderAlive()) {
                // The hosting process of the provider has died; we can&#39;t
                // use this one.
                Log.i(TAG, &#34;Acquiring provider &#34; &#43; auth &#43; &#34; for user &#34; &#43; userId
                        &#43; &#34;: existing object&#39;s process dead&#34;);
                //当provider所在的进程已经死亡则返回        
                handleUnstableProviderDiedLocked(jBinder, true);
                return null;
            }

            // Only increment the ref count if we have one.  If we don&#39;t then the
            // provider is not reference counted and never needs to be released.
            ProviderRefCount prc &#61; mProviderRefCountMap.get(jBinder);
            if (prc !&#61; null) {
                //增加引用计数
                incProviderRefLocked(prc, stable);
            }
            return provider;
        }
    }
</code></pre> 
<p>查询mProviderMap是否存在provider&#xff0c;如果不存在则直接返回。如果存在判断provider的进程是否死亡&#xff0c;死亡则返回null。如果provider所在的进程还在&#xff0c;则增加引用计数。</p> 
<h3><a id="36__AMSgetContentProvider_810"></a>3.6 AMS.getContentProvider</h3> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code> &#64;Override
    public final ContentProviderHolder getContentProvider(
            IApplicationThread caller, String name, int userId, boolean stable) {
        enforceNotIsolatedCaller(&#34;getContentProvider&#34;);
        if (caller &#61;&#61; null) {
            String msg &#61; &#34;null IApplicationThread when getting content provider &#34;
                    &#43; name;
            Slog.w(TAG, msg);
            throw new SecurityException(msg);
        }
        // The incoming user check is now handled in checkContentProviderPermissionLocked() to deal
        // with cross-user grant.
        return getContentProviderImpl(caller, name, null, stable, userId);
    }
</code></pre> 
<h3><a id="37__AMSgetContentProviderImpl_831"></a>3.7 AMS.getContentProviderImpl</h3> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code>  private ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
            String name, IBinder token, boolean stable, int userId) {
        ContentProviderRecord cpr;
        ContentProviderConnection conn &#61; null;
        ProviderInfo cpi &#61; null;
        boolean providerRunning &#61; false;

        synchronized(this) {
            long startTime &#61; SystemClock.uptimeMillis();

            ProcessRecord r &#61; null;
            if (caller !&#61; null) {
                //获取调用者的进程记录
                r &#61; getRecordForAppLocked(caller);
                if (r &#61;&#61; null) {
                    throw new SecurityException(
                            &#34;Unable to find app for caller &#34; &#43; caller
                          &#43; &#34; (pid&#61;&#34; &#43; Binder.getCallingPid()
                          &#43; &#34;) when getting content provider &#34; &#43; name);
                }
            }

            boolean checkCrossUser &#61; true;

            checkTime(startTime, &#34;getContentProviderImpl: getProviderByName&#34;);
            
            //从mProviderMap中获取ContentProviderRecord
            // First check if this content provider has been published...
            cpr &#61; mProviderMap.getProviderByName(name, userId);
            // If that didn&#39;t work, check if it exists for user 0 and then
            // verify that it&#39;s a singleton provider before using it.
            if (cpr &#61;&#61; null &amp;&amp; userId !&#61; UserHandle.USER_SYSTEM) {
                //从USER_SYSTEM获取&#xff0c;这里的name是componentName
                cpr &#61; mProviderMap.getProviderByName(name, UserHandle.USER_SYSTEM);
                if (cpr !&#61; null) {
                    cpi &#61; cpr.info;
                    if (isSingleton(cpi.processName, cpi.applicationInfo,
                            cpi.name, cpi.flags)
                            &amp;&amp; isValidSingletonCall(r.uid, cpi.applicationInfo.uid)) {
                        userId &#61; UserHandle.USER_SYSTEM;
                        checkCrossUser &#61; false;
                    } else {
                        cpr &#61; null;
                        cpi &#61; null;
                    }
                }
            }

            if (cpr !&#61; null &amp;&amp; cpr.proc !&#61; null) {
                providerRunning &#61; !cpr.proc.killed;

                // Note if killedByAm is also set, this means the provider process has just been
                // killed by AM (in ProcessRecord.kill()), but appDiedLocked() hasn&#39;t been called
                // yet. So we need to call appDiedLocked() here and let it clean up.
                // (See the commit message on I2c4ba1e87c2d47f2013befff10c49b3dc337a9a7 to see
                // how to test this case.)
                //目标provider进程死亡
                if (cpr.proc.killed &amp;&amp; cpr.proc.killedByAm) {
                    checkTime(startTime, &#34;getContentProviderImpl: before appDied (killedByAm)&#34;);
                    final long iden &#61; Binder.clearCallingIdentity();
                    try {
                        appDiedLocked(cpr.proc);
                    } finally {
                        Binder.restoreCallingIdentity(iden);
                    }
                    checkTime(startTime, &#34;getContentProviderImpl: after appDied (killedByAm)&#34;);
                }
            }
           //目标provider存在的情况&#xff0c;见3.7.1
           if (providerRunning) {
              ....
           }
           //目标provider不存在的情况&#xff0c;见3.7.2
           if (!providerRunning) {
              ....
           }
          //循环等待provider发布完成&#xff0c;见3.7.3
           synchronized (cpr) {
               while (cpr.provider &#61;&#61; null) {
                   ....
                }
           }
     }
       return super.onTransact(code, data, reply, flags);
  }
</code></pre> 
<p>这个方法比较长&#xff0c;主要分为三个部分&#xff1a;</p> 
<p>目标provider存在的情况&#xff1b;目标provider不存在的情况&#xff1b;循环等待目标provider发布完成。</p> 
<h4><a id="371_provider_927"></a>3.7.1 目标provider存在</h4> 
<pre><code>     if (providerRunning) {
                cpi &#61; cpr.info;
                String msg;
                checkTime(startTime, &#34;getContentProviderImpl: before checkContentProviderPermission&#34;);
                //检查权限
                if ((msg &#61; checkContentProviderPermissionLocked(cpi, r, userId, checkCrossUser))
                        !&#61; null) {
                    throw new SecurityException(msg);
                }
                checkTime(startTime, &#34;getContentProviderImpl: after checkContentProviderPermission&#34;);

                if (r !&#61; null &amp;&amp; cpr.canRunHere(r)) {
                    //当允许运行在调用者进程&#xff0c;且已经发布&#xff0c;则直接返回
                    // This provider has been published or is in the process
                    // of being published...  but it is also allowed to run
                    // in the caller&#39;s process, so don&#39;t make a connection
                    // and just let the caller instantiate its own instance.
                    ContentProviderHolder holder &#61; cpr.newHolder(null);
                    // don&#39;t give caller the provider object, it needs
                    // to make its own.
                    holder.provider &#61; null;
                    return holder;
                }
                // Don&#39;t expose providers between normal apps and instant apps
                try {
                    if (AppGlobals.getPackageManager()
                            .resolveContentProvider(name, 0 /*flags*/, userId) &#61;&#61; null) {
                        return null;
                    }
                } catch (RemoteException e) {
                }

                final long origId &#61; Binder.clearCallingIdentity();

                checkTime(startTime, &#34;getContentProviderImpl: incProviderCountLocked&#34;);
                // 增加引用计数&#xff0c;见3.8.1节
                // In this case the provider instance already exists, so we can
                // return it right away.
                conn &#61; incProviderCountLocked(r, cpr, token, stable);
                if (conn !&#61; null &amp;&amp; (conn.stableCount&#43;conn.unstableCount) &#61;&#61; 1) {
                    if (cpr.proc !&#61; null &amp;&amp; r.setAdj &lt;&#61; ProcessList.PERCEPTIBLE_APP_ADJ) {
                        // If this is a perceptible app accessing the provider,
                        // make sure to count it as being accessed and thus
                        // back up on the LRU list.  This is good because
                        // content providers are often expensive to start.
                        //更新进程LRU队列
                        checkTime(startTime, &#34;getContentProviderImpl: before updateLruProcess&#34;);
                        updateLruProcessLocked(cpr.proc, false, null);
                        checkTime(startTime, &#34;getContentProviderImpl: after updateLruProcess&#34;);
                    }
                }

                checkTime(startTime, &#34;getContentProviderImpl: before updateOomAdj&#34;);
                final int verifiedAdj &#61; cpr.proc.verifiedAdj;
                //更新进程adj
                boolean success &#61; updateOomAdjLocked(cpr.proc, true);
                // XXX things have changed so updateOomAdjLocked doesn&#39;t actually tell us
                // if the process has been successfully adjusted.  So to reduce races with
                // it, we will check whether the process still exists.  Note that this doesn&#39;t
                // completely get rid of races with LMK killing the process, but should make
                // them much smaller.
                if (success &amp;&amp; verifiedAdj !&#61; cpr.proc.setAdj &amp;&amp; !isProcessAliveLocked(cpr.proc)) {
                    success &#61; false;
                }
                maybeUpdateProviderUsageStatsLocked(r, cpr.info.packageName, name);
                checkTime(startTime, &#34;getContentProviderImpl: after updateOomAdj&#34;);
                if (DEBUG_PROVIDER) Slog.i(TAG_PROVIDER, &#34;Adjust success: &#34; &#43; success);
                // NOTE: there is still a race here where a signal could be
                // pending on the process even though we managed to update its
                // adj level.  Not sure what to do about this, but at least
                // the race is now smaller.
                if (!success) {
                    //provider进程被杀死&#xff0c;则减少应用计数&#xff0c;见3.8.3节
                    // Uh oh...  it looks like the provider&#39;s process
                    // has been killed on us.  We need to wait for a new
                    // process to be started, and make sure its death
                    // doesn&#39;t kill our process.
                    Slog.i(TAG, &#34;Existing provider &#34; &#43; cpr.name.flattenToShortString()
                            &#43; &#34; is crashing; detaching &#34; &#43; r);
                    boolean lastRef &#61; decProviderCountLocked(conn, cpr, token, stable);
                    checkTime(startTime, &#34;getContentProviderImpl: before appDied&#34;);
                    appDiedLocked(cpr.proc);
                    checkTime(startTime, &#34;getContentProviderImpl: after appDied&#34;);
                    if (!lastRef) {
                        // This wasn&#39;t the last ref our process had on
                        // the provider...  we have now been killed, bail.
                        return null;
                    }
                    providerRunning &#61; false;
                    conn &#61; null;
                } else {
                    cpr.proc.verifiedAdj &#61; cpr.proc.setAdj;
                }

                Binder.restoreCallingIdentity(origId);
            }

</code></pre> 
<p>当ContentProvider所在的进程已经存在&#xff1a;</p> 
<p>检查权限&#xff1b;当允许运行在调用者进程且已经发布&#xff0c;则直接返回</p> 
<p>增加引用计数&#xff1b;更新进程LRU队列&#xff1b;更新adj</p> 
<p>当provider进程被杀时&#xff0c;则奸商引用计数并调用appDiedLocked&#xff0c;设置provider为未发布状态。</p> 
<h4><a id="372__provider_1037"></a>3.7.2 目标provider不存在</h4> 
<pre><code>          if (!providerRunning) {
                try {
                    checkTime(startTime, &#34;getContentProviderImpl: before resolveContentProvider&#34;);
                    //获取ProviderInfo对象
                    cpi &#61; AppGlobals.getPackageManager().
                        resolveContentProvider(name,
                            STOCK_PM_FLAGS | PackageManager.GET_URI_PERMISSION_PATTERNS, userId);
                    checkTime(startTime, &#34;getContentProviderImpl: after resolveContentProvider&#34;);
                } catch (RemoteException ex) {
                }
                if (cpi &#61;&#61; null) {
                    return null;
                }
                //provider是否是单例
                // If the provider is a singleton AND
                // (it&#39;s a call within the same user || the provider is a
                // privileged app)
                // Then allow connecting to the singleton provider
                boolean singleton &#61; isSingleton(cpi.processName, cpi.applicationInfo,
                        cpi.name, cpi.flags)
                        &amp;&amp; isValidSingletonCall(r.uid, cpi.applicationInfo.uid);
                if (singleton) {
                    userId &#61; UserHandle.USER_SYSTEM;
                }
                cpi.applicationInfo &#61; getAppInfoForUser(cpi.applicationInfo, userId);
                checkTime(startTime, &#34;getContentProviderImpl: got app info for user&#34;);

                String msg;
                checkTime(startTime, &#34;getContentProviderImpl: before checkContentProviderPermission&#34;);
                //权限检查
                if ((msg &#61; checkContentProviderPermissionLocked(cpi, r, userId, !singleton))
                        !&#61; null) {
                    throw new SecurityException(msg);
                }
                checkTime(startTime, &#34;getContentProviderImpl: after checkContentProviderPermission&#34;);

                if (!mProcessesReady
                        &amp;&amp; !cpi.processName.equals(&#34;system&#34;)) {
                    // If this content provider does not run in the system
                    // process, and the system is not yet ready to run other
                    // processes, then fail fast instead of hanging.
                    throw new IllegalArgumentException(
                            &#34;Attempt to launch content provider before system ready&#34;);
                }

                // If system providers are not installed yet we aggressively crash to avoid
                // creating multiple instance of these providers and then bad things happen!
                if (!mSystemProvidersInstalled &amp;&amp; cpi.applicationInfo.isSystemApp()
                        &amp;&amp; &#34;system&#34;.equals(cpi.processName)) {
                    throw new IllegalStateException(&#34;Cannot access system provider: &#39;&#34;
                            &#43; cpi.authority &#43; &#34;&#39; before system providers are installed!&#34;);
                }
                
                //拥有该provider的用户没有运行&#xff0c;则直接返回
                // Make sure that the user who owns this provider is running.  If not,
                // we don&#39;t want to allow it to run.
                if (!mUserController.isUserRunning(userId, 0)) {
                    Slog.w(TAG, &#34;Unable to launch app &#34;
                            &#43; cpi.applicationInfo.packageName &#43; &#34;/&#34;
                            &#43; cpi.applicationInfo.uid &#43; &#34; for provider &#34;
                            &#43; name &#43; &#34;: user &#34; &#43; userId &#43; &#34; is stopped&#34;);
                    return null;
                }

                ComponentName comp &#61; new ComponentName(cpi.packageName, cpi.name);
                checkTime(startTime, &#34;getContentProviderImpl: before getProviderByClass&#34;);
                cpr &#61; mProviderMap.getProviderByClass(comp, userId);
                checkTime(startTime, &#34;getContentProviderImpl: after getProviderByClass&#34;);
                final boolean firstClass &#61; cpr &#61;&#61; null;
                if (firstClass) {
                    final long ident &#61; Binder.clearCallingIdentity();

                    // If permissions need a review before any of the app components can run,
                    // we return no provider and launch a review activity if the calling app
                    // is in the foreground.
                    if (mPermissionReviewRequired) {
                        if (!requestTargetProviderPermissionsReviewIfNeededLocked(cpi, r, userId)) {
                            return null;
                        }
                    }

                    try {
                        checkTime(startTime, &#34;getContentProviderImpl: before getApplicationInfo&#34;);
                        ApplicationInfo ai &#61;
                            AppGlobals.getPackageManager().
                                getApplicationInfo(
                                        cpi.applicationInfo.packageName,
                                        STOCK_PM_FLAGS, userId);
                        checkTime(startTime, &#34;getContentProviderImpl: after getApplicationInfo&#34;);
                        if (ai &#61;&#61; null) {
                            Slog.w(TAG, &#34;No package info for content provider &#34;
                                    &#43; cpi.name);
                            return null;
                        }
                        ai &#61; getAppInfoForUser(ai, userId);
                        cpr &#61; new ContentProviderRecord(this, cpi, ai, comp, singleton);
                    } catch (RemoteException ex) {
                        // pm is in same process, this will never happen.
                    } finally {
                        Binder.restoreCallingIdentity(ident);
                    }
                }

                checkTime(startTime, &#34;getContentProviderImpl: now have ContentProviderRecord&#34;);
                //见3.7.4节
                if (r !&#61; null &amp;&amp; cpr.canRunHere(r)) {
                    // If this is a multiprocess provider, then just return its
                    // info and allow the caller to instantiate it.  Only do
                    // this if the provider is the same user as the caller&#39;s
                    // process, or can run as root (so can be in any process).
                    return cpr.newHolder(null);
                }

                if (DEBUG_PROVIDER) Slog.w(TAG_PROVIDER, &#34;LAUNCHING REMOTE PROVIDER (myuid &#34;
                            &#43; (r !&#61; null ? r.uid : null) &#43; &#34; pruid &#34; &#43; cpr.appInfo.uid &#43; &#34;): &#34;
                            &#43; cpr.info.name &#43; &#34; callers&#61;&#34; &#43; Debug.getCallers(6));
                
                //从mLaunchingProviders中查询
                // This is single process, and our app is now connecting to it.
                // See if we are already in the process of launching this
                // provider.
                final int N &#61; mLaunchingProviders.size();
                int i;
                for (i &#61; 0; i &lt; N; i&#43;&#43;) {
                    if (mLaunchingProviders.get(i) &#61;&#61; cpr) {
                        break;
                    }
                }

               //如果mLaunchingProviders中没有该provider&#xff0c;则启动它
                // If the provider is not already being launched, then get it
                // started.
                if (i &gt;&#61; N) {
                    final long origId &#61; Binder.clearCallingIdentity();

                    try {
                        // Content provider is now in use, its package can&#39;t be stopped.
                        try {
                            checkTime(startTime, &#34;getContentProviderImpl: before set stopped state&#34;);
                            AppGlobals.getPackageManager().setPackageStoppedState(
                                    cpr.appInfo.packageName, false, userId);
                            checkTime(startTime, &#34;getContentProviderImpl: after set stopped state&#34;);
                        } catch (RemoteException e) {
                        } catch (IllegalArgumentException e) {
                            Slog.w(TAG, &#34;Failed trying to unstop package &#34;
                                    &#43; cpr.appInfo.packageName &#43; &#34;: &#34; &#43; e);
                        }

                        // Use existing process if already started
                        checkTime(startTime, &#34;getContentProviderImpl: looking for process record&#34;);
                        //查询进程记录
                        ProcessRecord proc &#61; getProcessRecordLocked(
                                cpi.processName, cpr.appInfo.uid, false);
                        if (proc !&#61; null &amp;&amp; proc.thread !&#61; null &amp;&amp; !proc.killed) {
                            if (DEBUG_PROVIDER) Slog.d(TAG_PROVIDER,
                                    &#34;Installing in existing process &#34; &#43; proc);
                            if (!proc.pubProviders.containsKey(cpi.name)) {
                                checkTime(startTime, &#34;getContentProviderImpl: scheduling install&#34;);
                                proc.pubProviders.put(cpi.name, cpr);
                                try {
                                     //发布provider&#xff0c;见2.2.1节
                                    proc.thread.scheduleInstallProvider(cpi);
                                } catch (RemoteException e) {
                                }
                            }
                        } else {
                            checkTime(startTime, &#34;getContentProviderImpl: before start process&#34;);
                            //启动进程
                            proc &#61; startProcessLocked(cpi.processName,
                                    cpr.appInfo, false, 0, &#34;content provider&#34;,
                                    new ComponentName(cpi.applicationInfo.packageName,
                                            cpi.name), false, false, false);
                            checkTime(startTime, &#34;getContentProviderImpl: after start process&#34;);
                            if (proc &#61;&#61; null) {
                                Slog.w(TAG, &#34;Unable to launch app &#34;
                                        &#43; cpi.applicationInfo.packageName &#43; &#34;/&#34;
                                        &#43; cpi.applicationInfo.uid &#43; &#34; for provider &#34;
                                        &#43; name &#43; &#34;: process is bad&#34;);
                                return null;
                            }
                        }
                        cpr.launchingApp &#61; proc;
                        //将cpr添加到mLaunchingProviders
                        mLaunchingProviders.add(cpr);
                    } finally {
                        Binder.restoreCallingIdentity(origId);
                    }
                }

                checkTime(startTime, &#34;getContentProviderImpl: updating data structures&#34;);

                // Make sure the provider is published (the same provider class
                // may be published under multiple names).
                if (firstClass) {
                    mProviderMap.putProviderByClass(comp, cpr);
                }

                mProviderMap.putProviderByName(name, cpr);
                conn &#61; incProviderCountLocked(r, cpr, token, stable);
                if (conn !&#61; null) {
                    conn.waiting &#61; true;
                }
            }
            checkTime(startTime, &#34;getContentProviderImpl: done!&#34;);

            grantEphemeralAccessLocked(userId, null /*intent*/,
                    cpi.applicationInfo.uid, UserHandle.getAppId(Binder.getCallingUid()));
        }
</code></pre> 
<p>当ContentProvider所在的进程不存在&#xff1a;</p> 
<p>根据authority获取ProviderInfo对象&#xff1b;权限检查&#xff1b;</p> 
<p>当系统不是运行在system进程&#xff0c;且系统未准备好&#xff0c;则抛出异常&#xff1b;</p> 
<p>当拥有该provider的用户没有运行&#xff0c;则直接返回&#xff1b;</p> 
<p>当允许运行在调用者进程且ProcessRecord不为空&#xff0c;则直接返回&#xff1b;</p> 
<p>当provider并没有在mLaunchingProviders队列&#xff0c;则启动它&#xff1a;</p> 
<ul><li> <p>当ProcessRecord不为空&#xff0c;则直接加入到pubProviders队列&#xff0c;并安装provider</p> </li><li> <p>当ProcessRecord为空&#xff0c;则启动进程</p> </li></ul> 
<p>增加引用计数</p> 
<h4><a id="373__provider_1268"></a>3.7.3 等待目标provider发布</h4> 
<p>【ContentProviderRecord</p> 
<pre><code>        static final int CONTENT_PROVIDER_WAIT_TIMEOUT &#61; 20 * 1000;
        // Wait for the provider to be published...
        final long timeout &#61; SystemClock.uptimeMillis() &#43; CONTENT_PROVIDER_WAIT_TIMEOUT;
        synchronized (cpr) {
            //
            while (cpr.provider &#61;&#61; null) {
                if (cpr.launchingApp &#61;&#61; null) {
                    Slog.w(TAG, &#34;Unable to launch app &#34;
                            &#43; cpi.applicationInfo.packageName &#43; &#34;/&#34;
                            &#43; cpi.applicationInfo.uid &#43; &#34; for provider &#34;
                            &#43; name &#43; &#34;: launching app became null&#34;);
                    EventLog.writeEvent(EventLogTags.AM_PROVIDER_LOST_PROCESS,
                            UserHandle.getUserId(cpi.applicationInfo.uid),
                            cpi.applicationInfo.packageName,
                            cpi.applicationInfo.uid, name);
                    return null;
                }
                try {
                    final long wait &#61; Math.max(0L, timeout - SystemClock.uptimeMillis());
                    if (DEBUG_MU) Slog.v(TAG_MU,
                            &#34;Waiting to start provider &#34; &#43; cpr
                            &#43; &#34; launchingApp&#61;&#34; &#43; cpr.launchingApp &#43; &#34; for &#34; &#43; wait &#43; &#34; ms&#34;);
                    if (conn !&#61; null) {
                        conn.waiting &#61; true;
                    }
                    cpr.wait(wait);
                    if (cpr.provider &#61;&#61; null) {
                        Slog.wtf(TAG, &#34;Timeout waiting for provider &#34;
                                &#43; cpi.applicationInfo.packageName &#43; &#34;/&#34;
                                &#43; cpi.applicationInfo.uid &#43; &#34; for provider &#34;
                                &#43; name
                                &#43; &#34; providerRunning&#61;&#34; &#43; providerRunning);
                        return null;
                    }
                } catch (InterruptedException ex) {
                } finally {
                    if (conn !&#61; null) {
                        conn.waiting &#61; false;
                    }
                }
            }
        }
        return cpr !&#61; null ? cpr.newHolder(conn) : null;
</code></pre> 
<p>循环等待20s&#xff0c;直到发布完成才退出。</p> 
<h4><a id="374__CPRcanRunHere_1320"></a>3.7.4 CPR.canRunHere</h4> 
<p>[-&gt;ContentProviderRecord.java]</p> 
<pre><code>  public boolean canRunHere(ProcessRecord app) {
        return (info.multiprocess || info.processName.equals(app.processName))
                &amp;&amp; uid &#61;&#61; app.info.uid;
    }
</code></pre> 
<p>ContentProvider是否能运行在调用者所在的进程需要满足以下条件</p> 
<p>1.ContentProvider在AndroidManifest.xml文件配置中multiprocess&#61;true&#xff1b;或者调用者进程和ContentProvider在同一进程</p> 
<p>2.ContentProvider进程跟调用者所在的进程是同一个uid。</p> 
<h3><a id="38_ATinstallProvider_1337"></a>3.8 AT.installProvider</h3> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code>private ContentProviderHolder installProvider(Context context,
            ContentProviderHolder holder, ProviderInfo info,
            boolean noisy, boolean noReleaseNeeded, boolean stable) {
        ContentProvider localProvider &#61; null;
        IContentProvider provider;
        if (holder &#61;&#61; null || holder.provider &#61;&#61; null) {
            //从packageInfo中获取provider
            if (DEBUG_PROVIDER || noisy) {
                Slog.d(TAG, &#34;Loading provider &#34; &#43; info.authority &#43; &#34;: &#34;
                        &#43; info.name);
            }
            Context c &#61; null;
            ApplicationInfo ai &#61; info.applicationInfo;
            if (context.getPackageName().equals(ai.packageName)) {
                c &#61; context;
            } else if (mInitialApplication !&#61; null &amp;&amp;
                    mInitialApplication.getPackageName().equals(ai.packageName)) {
                c &#61; mInitialApplication;
            } else {
                try {
                    c &#61; context.createPackageContext(ai.packageName,
                            Context.CONTEXT_INCLUDE_CODE);
                } catch (PackageManager.NameNotFoundException e) {
                    // Ignore
                }
            }
            if (c &#61;&#61; null) {
                Slog.w(TAG, &#34;Unable to get context for package &#34; &#43;
                      ai.packageName &#43;
                      &#34; while loading content provider &#34; &#43;
                      info.name);
                return null;
            }

            if (info.splitName !&#61; null) {
                try {
                    c &#61; c.createContextForSplit(info.splitName);
                } catch (NameNotFoundException e) {
                    throw new RuntimeException(e);
                }
            }

            try {
                final java.lang.ClassLoader cl &#61; c.getClassLoader();
                LoadedApk packageInfo &#61; peekPackageInfo(ai.packageName, true);
                if (packageInfo &#61;&#61; null) {
                    // System startup case.
                    packageInfo &#61; getSystemContext().mPackageInfo;
                }
                localProvider &#61; packageInfo.getAppFactory()
                        .instantiateProvider(cl, info.name);
                provider &#61; localProvider.getIContentProvider();
                if (provider &#61;&#61; null) {
                    Slog.e(TAG, &#34;Failed to instantiate class &#34; &#43;
                          info.name &#43; &#34; from sourceDir &#34; &#43;
                          info.applicationInfo.sourceDir);
                    return null;
                }
                if (DEBUG_PROVIDER) Slog.v(
                    TAG, &#34;Instantiating local provider &#34; &#43; info.name);
                // XXX Need to create the correct context for this provider.
                localProvider.attachInfo(c, info);
            } catch (java.lang.Exception e) {
                if (!mInstrumentation.onException(null, e)) {
                    throw new RuntimeException(
                            &#34;Unable to get provider &#34; &#43; info.name
                            &#43; &#34;: &#34; &#43; e.toString(), e);
                }
                return null;
            }
        } else {
            provider &#61; holder.provider;
            if (DEBUG_PROVIDER) Slog.v(TAG, &#34;Installing external provider &#34; &#43; info.authority &#43; &#34;: &#34;
                    &#43; info.name);
        }

        ContentProviderHolder retHolder;

        synchronized (mProviderMap) {
            if (DEBUG_PROVIDER) Slog.v(TAG, &#34;Checking to add &#34; &#43; provider
                    &#43; &#34; / &#34; &#43; info.name);
            IBinder jBinder &#61; provider.asBinder();
            if (localProvider !&#61; null) {
                ComponentName cname &#61; new ComponentName(info.packageName, info.name);
                ProviderClientRecord pr &#61; mLocalProvidersByName.get(cname);
                if (pr !&#61; null) {
                    if (DEBUG_PROVIDER) {
                        Slog.v(TAG, &#34;installProvider: lost the race, &#34;
                                &#43; &#34;using existing local provider&#34;);
                    }
                    provider &#61; pr.mProvider;
                } else {
                    holder &#61; new ContentProviderHolder(info);
                    holder.provider &#61; provider;
                    holder.noReleaseNeeded &#61; true;
                    pr &#61; installProviderAuthoritiesLocked(provider, localProvider, holder);
                    mLocalProviders.put(jBinder, pr);
                    mLocalProvidersByName.put(cname, pr);
                }
                retHolder &#61; pr.mHolder;
            } else {
                //查询ProviderRefCount
                ProviderRefCount prc &#61; mProviderRefCountMap.get(jBinder);
                if (prc !&#61; null) {
                    if (DEBUG_PROVIDER) {
                        Slog.v(TAG, &#34;installProvider: lost the race, updating ref count&#34;);
                    }
                    // We need to transfer our new reference to the existing
                    // ref count, releasing the old one...  but only if
                    // release is needed (that is, it is not running in the
                    // system process).
                    if (!noReleaseNeeded) {
                        //增加引用计数,通过AMS&#xff0c;见3.8.4
                        incProviderRefLocked(prc, stable);
                        try {
                            
                            ActivityManager.getService().removeContentProvider(
                                    holder.connection, stable);
                        } catch (RemoteException e) {
                            //do nothing content provider object is dead any way
                        }
                    }
                } else {
                    //见3.8.5节
                    ProviderClientRecord client &#61; installProviderAuthoritiesLocked(
                            provider, localProvider, holder);
                    if (noReleaseNeeded) {
                       //见3.8.6节
                        prc &#61; new ProviderRefCount(holder, client, 1000, 1000);
                    } else {
                        prc &#61; stable
                                ? new ProviderRefCount(holder, client, 1, 0)
                                : new ProviderRefCount(holder, client, 0, 1);
                    }
                    mProviderRefCountMap.put(jBinder, prc);
                }
                retHolder &#61; prc.holder;
            }
        }
        return retHolder;
    }
</code></pre> 
<h4><a id="381__AMSincProviderCountLocked_1485"></a>3.8.1 AMS.incProviderCountLocked</h4> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code> ContentProviderConnection incProviderCountLocked(ProcessRecord r,
            final ContentProviderRecord cpr, IBinder externalProcessToken, boolean stable) {
        if (r !&#61; null) {
            for (int i&#61;0; i&lt;r.conProviders.size(); i&#43;&#43;) {
                //从当前进程所使用的provider中查询与目标provider一致的信息
                ContentProviderConnection conn &#61; r.conProviders.get(i);
                if (conn.provider &#61;&#61; cpr) {
                    if (DEBUG_PROVIDER) Slog.v(TAG_PROVIDER,
                            &#34;Adding provider requested by &#34;
                            &#43; r.processName &#43; &#34; from process &#34;
                            &#43; cpr.info.processName &#43; &#34;: &#34; &#43; cpr.name.flattenToShortString()
                            &#43; &#34; scnt&#61;&#34; &#43; conn.stableCount &#43; &#34; uscnt&#61;&#34; &#43; conn.unstableCount);
                    if (stable) {
                        conn.stableCount&#43;&#43;;
                        conn.numStableIncs&#43;&#43;;
                    } else {
                        conn.unstableCount&#43;&#43;;
                        conn.numUnstableIncs&#43;&#43;;
                    }
                    return conn;
                }
            }
            //查不到则新建provider连接对象
            ContentProviderConnection conn &#61; new ContentProviderConnection(cpr, r);
            if (stable) {
                conn.stableCount &#61; 1;
                conn.numStableIncs &#61; 1;
            } else {
                conn.unstableCount &#61; 1;
                conn.numUnstableIncs &#61; 1;
            }
            cpr.connections.add(conn);
            r.conProviders.add(conn);
            startAssociationLocked(r.uid, r.processName, r.curProcState,
                    cpr.uid, cpr.name, cpr.info.processName);
            return conn;
        }
        cpr.addExternalProcessHandleLocked(externalProcessToken);
        return null;
    }
</code></pre> 
<h4><a id="382__AMSremoveContentProvider_1532"></a>3.8.2 AMS.removeContentProvider</h4> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code> /**
     * Drop a content provider from a ProcessRecord&#39;s bookkeeping
     */
    public void removeContentProvider(IBinder connection, boolean stable) {
        enforceNotIsolatedCaller(&#34;removeContentProvider&#34;);
        long ident &#61; Binder.clearCallingIdentity();
        try {
            synchronized (this) {
                ContentProviderConnection conn;
                try {
                    conn &#61; (ContentProviderConnection)connection;
                } catch (ClassCastException e) {
                    String msg &#61;&#34;removeContentProvider: &#34; &#43; connection
                            &#43; &#34; not a ContentProviderConnection&#34;;
                    Slog.w(TAG, msg);
                    throw new IllegalArgumentException(msg);
                }
                if (conn &#61;&#61; null) {
                    throw new NullPointerException(&#34;connection is null&#34;);
                }
                //见3.8.3节
                if (decProviderCountLocked(conn, null, null, stable)) {
                    updateOomAdjLocked();
                }
            }
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }
</code></pre> 
<h4><a id="383__AMSdecProviderCountLocked_1568"></a>3.8.3 AMS.decProviderCountLocked</h4> 
<p>[-&gt;ActivityManagerService.java]</p> 
<pre><code>boolean decProviderCountLocked(ContentProviderConnection conn,
            ContentProviderRecord cpr, IBinder externalProcessToken, boolean stable) {
        if (conn !&#61; null) {
            cpr &#61; conn.provider;
            if (DEBUG_PROVIDER) Slog.v(TAG_PROVIDER,
                    &#34;Removing provider requested by &#34;
                    &#43; conn.client.processName &#43; &#34; from process &#34;
                    &#43; cpr.info.processName &#43; &#34;: &#34; &#43; cpr.name.flattenToShortString()
                    &#43; &#34; scnt&#61;&#34; &#43; conn.stableCount &#43; &#34; uscnt&#61;&#34; &#43; conn.unstableCount);
            if (stable) {
                conn.stableCount--;
            } else {
                conn.unstableCount--;
            }
            //当provider连接的stable和unstable的引用次数为01时&#xff0c;则移除连接对象信息
            if (conn.stableCount &#61;&#61; 0 &amp;&amp; conn.unstableCount &#61;&#61; 0) {
                cpr.connections.remove(conn);
                conn.client.conProviders.remove(conn);
                if (conn.client.setProcState &lt; ActivityManager.PROCESS_STATE_LAST_ACTIVITY) {
                    // The client is more important than last activity -- note the time this
                    // is happening, so we keep the old provider process around a bit as last
                    // activity to avoid thrashing it.
                    if (cpr.proc !&#61; null) {
                        cpr.proc.lastProviderTime &#61; SystemClock.uptimeMillis();
                    }
                }
                stopAssociationLocked(conn.client.uid, conn.client.processName, cpr.uid, cpr.name);
                return true;
            }
            return false;
        }
        cpr.removeExternalProcessHandleLocked(externalProcessToken);
        return false;
    }

</code></pre> 
<h4><a id="384__ATincProviderRefLocked_1610"></a>3.8.4 AT.incProviderRefLocked</h4> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code> private final void incProviderRefLocked(ProviderRefCount prc, boolean stable) {
        //stable provider
        if (stable) {
            prc.stableCount &#43;&#61; 1;
            if (prc.stableCount &#61;&#61; 1) {
                //数量为1&#xff0c;状态为removePending&#xff0c;发送REMOVE_PROVIDER信息
                // We are acquiring a new stable reference on the provider.
                int unstableDelta;
                if (prc.removePending) {
                    // We have a pending remove operation, which is holding the
                    // last unstable reference.  At this point we are converting
                    // that unstable reference to our new stable reference.
                    unstableDelta &#61; -1;
                    // Cancel the removal of the provider.
                    if (DEBUG_PROVIDER) {
                        Slog.v(TAG, &#34;incProviderRef: stable &#34;
                                &#43; &#34;snatched provider from the jaws of death&#34;);
                    }
                    prc.removePending &#61; false;
                    // There is a race! It fails to remove the message, which
                    // will be handled in completeRemoveProvider().
                    mH.removeMessages(H.REMOVE_PROVIDER, prc);
                } else {
                    unstableDelta &#61; 0;
                }
                try {
                    if (DEBUG_PROVIDER) {
                        Slog.v(TAG, &#34;incProviderRef Now stable - &#34;
                                &#43; prc.holder.info.name &#43; &#34;: unstableDelta&#61;&#34;
                                &#43; unstableDelta);
                    }
                    //调用AMS方法
                    ActivityManager.getService().refContentProvider(
                            prc.holder.connection, 1, unstableDelta);
                } catch (RemoteException e) {
                    //do nothing content provider object is dead any way
                }
            }
        } else {
            prc.unstableCount &#43;&#61; 1;
            if (prc.unstableCount &#61;&#61; 1) {
                // We are acquiring a new unstable reference on the provider.
                if (prc.removePending) {
                    // Oh look, we actually have a remove pending for the
                    // provider, which is still holding the last unstable
                    // reference.  We just need to cancel that to take new
                    // ownership of the reference.
                    if (DEBUG_PROVIDER) {
                        Slog.v(TAG, &#34;incProviderRef: unstable &#34;
                                &#43; &#34;snatched provider from the jaws of death&#34;);
                    }
                    prc.removePending &#61; false;
                    mH.removeMessages(H.REMOVE_PROVIDER, prc);
                } else {
                    // First unstable ref, increment our count in the
                    // activity manager.
                    try {
                        if (DEBUG_PROVIDER) {
                            Slog.v(TAG, &#34;incProviderRef: Now unstable - &#34;
                                    &#43; prc.holder.info.name);
                        }
                        ActivityManager.getService().refContentProvider(
                                prc.holder.connection, 0, 1);
                    } catch (RemoteException e) {
                        //do nothing content provider object is dead any way
                    }
                }
            }
        }
    }
</code></pre> 
<h4><a id="385__ATinstallProviderAuthoritiesLocked_1687"></a>3.8.5 AT.installProviderAuthoritiesLocked</h4> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code> private ProviderClientRecord installProviderAuthoritiesLocked(IContentProvider provider,
            ContentProvider localProvider, ContentProviderHolder holder) {
        final String auths[] &#61; holder.info.authority.split(&#34;;&#34;);
        final int userId &#61; UserHandle.getUserId(holder.info.applicationInfo.uid);

        if (provider !&#61; null) {
            // If this provider is hosted by the core OS and cannot be upgraded,
            // then I guess we&#39;re okay doing blocking calls to it.
            for (String auth : auths) {
                switch (auth) {
                    case ContactsContract.AUTHORITY:
                    case CallLog.AUTHORITY:
                    case CallLog.SHADOW_AUTHORITY:
                    case BlockedNumberContract.AUTHORITY:
                    case CalendarContract.AUTHORITY:
                    case Downloads.Impl.AUTHORITY:
                    case &#34;telephony&#34;:
                        Binder.allowBlocking(provider.asBinder());
                }
            }
        }

        final ProviderClientRecord pcr &#61; new ProviderClientRecord(
                auths, provider, localProvider, holder);
        for (String auth : auths) {
            final ProviderKey key &#61; new ProviderKey(auth, userId);
            final ProviderClientRecord existing &#61; mProviderMap.get(key);
            if (existing !&#61; null) {
                Slog.w(TAG, &#34;Content provider &#34; &#43; pcr.mHolder.info.name
                        &#43; &#34; already published as &#34; &#43; auth);
            } else {
                //发布
                mProviderMap.put(key, pcr);
            }
        }
        return pcr;
    }

</code></pre> 
<h4><a id="386_ProviderRefCount_1732"></a>3.8.6 ProviderRefCount</h4> 
<p>[-&gt;ActivityThread.java::ProviderRefCount]</p> 
<pre><code>  private static final class ProviderRefCount {
        public final ContentProviderHolder holder;
        public final ProviderClientRecord client;
        public int stableCount;
        public int unstableCount;
        
        //当该标记设置时&#xff0c;stableCount和unstableCount引用都会设置为0
        // When this is set, the stable and unstable ref counts are 0 and
        // we have a pending operation scheduled to remove the ref count
        // from the activity manager.  On the activity manager we are still
        // holding an unstable ref, though it is not reflected in the counts
        // here.
        public boolean removePending;

        ProviderRefCount(ContentProviderHolder inHolder,
                ProviderClientRecord inClient, int sCount, int uCount) {
            holder &#61; inHolder;
            client &#61; inClient;
            stableCount &#61; sCount;
            unstableCount &#61; uCount;
        }
    }
</code></pre> 
<p>获取到Provider后&#xff0c;就可以进行查询操作了。</p> 
<h3><a id="39_CPPquery_1763"></a>3.9 CPP.query</h3> 
<p>[-&gt;ContentProviderNative.java::ContentProviderProxy]</p> 
<pre><code>  &#64;Override
    public Cursor query(String callingPkg, Uri url, &#64;Nullable String[] projection,
            &#64;Nullable Bundle queryArgs, &#64;Nullable ICancellationSignal cancellationSignal)
            throws RemoteException {
        BulkCursorToCursorAdaptor adaptor &#61; new BulkCursorToCursorAdaptor();
        Parcel data &#61; Parcel.obtain();
        Parcel reply &#61; Parcel.obtain();
        try {
            data.writeInterfaceToken(IContentProvider.descriptor);

            data.writeString(callingPkg);
            url.writeToParcel(data, 0);
            int length &#61; 0;
            if (projection !&#61; null) {
                length &#61; projection.length;
            }
            data.writeInt(length);
            for (int i &#61; 0; i &lt; length; i&#43;&#43;) {
                data.writeString(projection[i]);
            }
            data.writeBundle(queryArgs);
            data.writeStrongBinder(adaptor.getObserver().asBinder());
            data.writeStrongBinder(
                    cancellationSignal !&#61; null ? cancellationSignal.asBinder() : null);
            //发送给binder服务
            mRemote.transact(IContentProvider.QUERY_TRANSACTION, data, reply, 0);

            DatabaseUtils.readExceptionFromParcel(reply);

            if (reply.readInt() !&#61; 0) {
                BulkCursorDescriptor d &#61; BulkCursorDescriptor.CREATOR.createFromParcel(reply);
                Binder.copyAllowBlocking(mRemote, (d.cursor !&#61; null) ? d.cursor.asBinder() : null);
                adaptor.initialize(d);
            } else {
                adaptor.close();
                adaptor &#61; null;
            }
            return adaptor;
        } catch (RemoteException ex) {
            adaptor.close();
            throw ex;
        } catch (RuntimeException ex) {
            adaptor.close();
            throw ex;
        } finally {
            data.recycle();
            reply.recycle();
        }
    }
</code></pre> 
<h3><a id="310_CPNonTransact_1819"></a>3.10 CPN.onTransact</h3> 
<p>[-&gt;ContentProviderNative.java]</p> 
<pre><code>   &#64;Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        try {
            switch (code) {
                case QUERY_TRANSACTION:
                {
                    data.enforceInterface(IContentProvider.descriptor);

                    String callingPkg &#61; data.readString();
                    Uri url &#61; Uri.CREATOR.createFromParcel(data);

                    // String[] projection
                    int num &#61; data.readInt();
                    String[] projection &#61; null;
                    if (num &gt; 0) {
                        projection &#61; new String[num];
                        for (int i &#61; 0; i &lt; num; i&#43;&#43;) {
                            projection[i] &#61; data.readString();
                        }
                    }

                    Bundle queryArgs &#61; data.readBundle();
                    IContentObserver observer &#61; IContentObserver.Stub.asInterface(
                            data.readStrongBinder());
                    ICancellationSignal cancellationSignal &#61; ICancellationSignal.Stub.asInterface(
                            data.readStrongBinder());
                    //见3.11节      
                    Cursor cursor &#61; query(callingPkg, url, projection, queryArgs, cancellationSignal);
                    if (cursor !&#61; null) {
                        CursorToBulkCursorAdaptor adaptor &#61; null;

                        try {
                            adaptor &#61; new CursorToBulkCursorAdaptor(cursor, observer,
                                    getProviderName());
                            cursor &#61; null;

                            BulkCursorDescriptor d &#61; adaptor.getBulkCursorDescriptor();
                            adaptor &#61; null;

                            reply.writeNoException();
                            reply.writeInt(1);
                            d.writeToParcel(reply, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
                        } finally {
                            // Close cursor if an exception was thrown while constructing the adaptor.
                            if (adaptor !&#61; null) {
                                adaptor.close();
                            }
                            if (cursor !&#61; null) {
                                cursor.close();
                            }
                        }
                    } else {
                        reply.writeNoException();
                        reply.writeInt(0);
                    }

                    return true;
                }

                case GET_TYPE_TRANSACTION:
                case INSERT_TRANSACTION:
                case BULK_INSERT_TRANSACTION&#xff1a; 
                case APPLY_BATCH_TRANSACTION:
                case DELETE_TRANSACTION:
                case UPDATE_TRANSACTION:
                ...
            }
        } catch (Exception e) {
            DatabaseUtils.writeExceptionToParcel(reply, e);
            return true;
        }
</code></pre> 
<h3><a id="311_Transportquery_1898"></a>3.11 Transport.query</h3> 
<p>[-&gt;ContentProvider.java::Transport]</p> 
<pre><code>  public Cursor query(String callingPkg, Uri uri, &#64;Nullable String[] projection,
                &#64;Nullable Bundle queryArgs, &#64;Nullable ICancellationSignal cancellationSignal) {
            uri &#61; validateIncomingUri(uri);
            uri &#61; maybeGetUriWithoutUserId(uri);
            if (enforceReadPermission(callingPkg, uri, null) !&#61; AppOpsManager.MODE_ALLOWED) {
                // The caller has no access to the data, so return an empty cursor with
                // the columns in the requested order. The caller may ask for an invalid
                // column and we would not catch that but this is not a problem in practice.
                // We do not call ContentProvider#query with a modified where clause since
                // the implementation is not guaranteed to be backed by a SQL database, hence
                // it may not handle properly the tautology where clause we would have created.
                if (projection !&#61; null) {
                    return new MatrixCursor(projection, 0);
                }

                // Null projection means all columns but we have no idea which they are.
                // However, the caller may be expecting to access them my index. Hence,
                // we have to execute the query as if allowed to get a cursor with the
                // columns. We then use the column names to return an empty cursor.
                Cursor cursor &#61; ContentProvider.this.query(
                        uri, projection, queryArgs,
                        CancellationSignal.fromTransport(cancellationSignal));
                if (cursor &#61;&#61; null) {
                    return null;
                }

                // Return an empty cursor for all columns.
                return new MatrixCursor(cursor.getColumnNames(), 0);
            }
            final String original &#61; setCallingPackage(callingPkg);
            try {
                //调用目标provider的query方法
                return ContentProvider.this.query(
                        uri, projection, queryArgs,
                        CancellationSignal.fromTransport(cancellationSignal));
            } finally {
                setCallingPackage(original);
            }
        }
</code></pre> 
<h2><a id="_1944"></a>四、总结</h2> 
<p>本文通过分析了发布ContentProvider的过程&#xff0c;并分析query过程&#xff0c;先获取provider在安装provider信息&#xff0c;最后才查询。而查询操作主要分为两种情况&#xff1a;</p> 
<h3><a id="41__1948"></a>4.1 进程未启动</h3> 
<p>provider进程不存在&#xff0c;需要创建进程并发布相关的provider</p> 
<p><img src="https://img-blog.csdnimg.cn/20200105193155938.jpg?x-oss-process&#61;image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Nhbzg2MTU0NDMyNQ&#61;&#61;,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" /></p> 
<p>1.client进程&#xff1a;通过Binder向systemserver进程请求相应的provider</p> 
<p>2.systemserver进程&#xff1a;如果目标进程未启动&#xff0c;则调用startProcessLocked启动进程&#xff0c;当启动完成&#xff0c;当cpr.provider &#61;&#61;null&#xff0c;则systemserver进入wait阶段&#xff0c;等待目标provider发布&#xff1b;</p> 
<p>3.provider进程&#xff1a;进程启动后执行attach到systemserver&#xff0c;而后bindApplication&#xff0c;在这个过程会installProvider和PublishContentProviders,再binder到systemserver进程&#xff1b;</p> 
<p>4.systemserver进程&#xff1a;回到systemserver发布provider信息&#xff0c;并且通过notify机制唤醒当前处于wait状态的binder线程&#xff0c;并将结果返回给client进程&#xff1b;</p> 
<p>5.client进程&#xff1a;回到client进程&#xff0c;执行installProvider操作&#xff0c;安装provider</p> 
<p>关于<code>CONTENT_PROVIDER_PUBLISH_TIMEOUT</code>超时机制所统计的时机区间是指在startProcessLocked之后会调用AMS.attachApplicationLocked为起点&#xff0c;一直到AMS.publishContentProviders的过程。</p> 
<h3><a id="42__1966"></a>4.2 进程已启动</h3> 
<p>provider进程已启动但未发布&#xff0c;需要发布相关的provider</p> 
<p><img src="https://img-blog.csdnimg.cn/20200105193449997.jpg?x-oss-process&#61;image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Nhbzg2MTU0NDMyNQ&#61;&#61;,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" /><br /> Client进程&#xff1a;获取provider发现cpr为空&#xff0c;则调用scheduleInstallProvider来向provider所在的进程发出一个oneway的binder请求&#xff0c;进入wait状态&#xff1b;</p> 
<p>provider进程&#xff1a;安装完成provider信息后&#xff0c;通过notify机制唤醒当前处于wait状态的binder线程。</p> 
<p>如果provider在publish之后&#xff0c;这是在请求provider则没用最右边的过程&#xff0c;直接AMS.getContentProvierImpl之后便进入AT.installProvider的过程&#xff0c;而不会再进入wait过程。</p> 
<h3><a id="43__1977"></a>4.3 引用计数</h3> 
<p>provider分为stable provider和unstable provider,主要在于引用计数的不同。</p> 
<p>provider引用计数的增加与减少关系&#xff0c;removePending是指即将被移除的引用&#xff0c;lastRef表示当前引用为0。</p> 
<table><thead><tr><th align="left">方法</th><th align="left">stableCount</th><th align="left">unstableCount</th><th align="left">条件</th></tr></thead><tbody><tr><td align="left">acquireProvider</td><td align="left">&#43;1</td><td align="left">0</td><td align="left">removePending&#61;false</td></tr><tr><td align="left">acquireProvider</td><td align="left">&#43;1</td><td align="left">-1</td><td align="left">removePending&#61;true</td></tr><tr><td align="left">acquireUnstableProvider</td><td align="left">0</td><td align="left">&#43;1</td><td align="left">removePending&#61;false</td></tr><tr><td align="left">acquireUnstableProvider</td><td align="left">0</td><td align="left">0</td><td align="left">removePending&#61;true</td></tr><tr><td align="left">releaseProvider</td><td align="left">-1</td><td align="left">0</td><td align="left">lastRef&#61;false</td></tr><tr><td align="left">releaseProvider</td><td align="left">-1</td><td align="left">1</td><td align="left">lastRef&#61;true</td></tr><tr><td align="left">releaseUnstableProvider</td><td align="left">0</td><td align="left">-1</td><td align="left">lastRef&#61;false</td></tr><tr><td align="left">releaseUnstableProvider</td><td align="left">0</td><td align="left">0</td><td align="left">lastRef&#61;true</td></tr></tbody></table>
<p>当Client进程存在对某个provider的引用时&#xff0c;Provider进程死亡则会根据provider类型进行不同的处理&#xff1a;</p> 
<ul><li> <p>对于stable provider,会杀掉所有和该provider建立stable连接的非persistent进程&#xff1b;</p> </li><li> <p>对于unstable provider&#xff0c;不会导致client进程被级联所杀&#xff0c;会回调unstableProviderDied来清理相关信息。</p> </li></ul> 
<p>当stable和unstable引用计数都为0则移除connection信息</p> 
<ul><li>AMS.removeContentProvider过程移除connection相关所有信息。</li></ul> 
<h2><a id="_2004"></a>附录</h2> 
<p>源码路径</p> 
<pre><code>frameworks/base/core/java/android/app/ActivityThread.java
frameworks/base/core/java/android/app/ContextImpl.java
frameworks/base/core/java/android/app/ActivityThread.java
frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
frameworks/base/core/java/android/content/ContentResolver.java
frameworks/base/core/java/android/content/ContentProviderNative.java
frameworks/base/services/core/java/com/android/server/am/ContentProviderRecord.java

</code></pre>