 ---
layout:     post
title:      Android10 屏幕VIEW刷新机制分析
subtitle:   本文将从startActivity开始讲解Android屏幕刷新机制
date:       2020-10-14
author:     duguma
header-img: img/article-bg.jpg
top: true
catalog: true
tags:
    - Android10
    - Android
	- 组件学习
 ---

<h2><a id="_4"></a>一、概述</h2> 
<p>本文将从startActivity开始讲解Android屏幕刷新机制&#xff0c;前面的文章有分析过<a href="https://skytoby.github.io/2019/startActivity%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/">startActivity的启动过程</a>&#xff0c;这里将重点分析WMS相关的过程&#xff0c;从而了解Android屏幕刷新机制原理。前面介绍的startActivity启动过程的流程图如下&#xff1a;</p> 
<p><img src="https://img-blog.csdnimg.cn/20200729172401558.jpg?x-oss-process&#61;image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Nhbzg2MTU0NDMyNQ&#61;&#61;,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" /></p> 
<h2><a id="View_11"></a>二、View的绘制过程</h2> 
<p>从启动过程中的performLaunchActivity开始分析&#xff0c;View真正的绘制是在Activity中的onResume方法中。</p> 
<h3><a id="21_ATperformLaunchActivity_15"></a>2.1 AT.performLaunchActivity</h3> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code>private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
       ... 
       //创建Application
       Application app &#61; r.packageInfo.makeApplication(false, mInstrumentation);
       ...
       //见2.2节
       activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);
       ...
        if (r.isPersistable()) {
             mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
        } else {
             mInstrumentation.callActivityOnCreate(activity, r.state);
        }
        ...        
       return activity;
 }
</code></pre> 
<h3><a id="22_Activityattach_41"></a>2.2 Activity.attach</h3> 
<p>[-&gt;Activity.java]</p> 
<pre><code> final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback) {
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);
        //创建PhoneWindow
        mWindow &#61; new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode !&#61; WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions !&#61; 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        mUiThread &#61; Thread.currentThread();

        mMainThread &#61; aThread;
        mInstrumentation &#61; instr;
        mToken &#61; token;
        mIdent &#61; ident;
        mApplication &#61; application;
        mIntent &#61; intent;
        mReferrer &#61; referrer;
        mComponent &#61; intent.getComponent();
        mActivityInfo &#61; info;
        mTitle &#61; title;
        mParent &#61; parent;
        mEmbeddedID &#61; id;
        mLastNonConfigurationInstances &#61; lastNonConfigurationInstances;
        if (voiceInteractor !&#61; null) {
            if (lastNonConfigurationInstances !&#61; null) {
                mVoiceInteractor &#61; lastNonConfigurationInstances.voiceInteractor;
            } else {
                mVoiceInteractor &#61; new VoiceInteractor(voiceInteractor, this, this,
                        Looper.myLooper());
            }
        }

        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags &amp; ActivityInfo.FLAG_HARDWARE_ACCELERATED) !&#61; 0);
        if (mParent !&#61; null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager &#61; mWindow.getWindowManager();
        mCurrentConfig &#61; config;

        mWindow.setColorMode(info.colorMode);

        setAutofillCompatibilityEnabled(application.isAutofillCompatibilityEnabled());
    }
</code></pre> 
<p>这里比较重要的操作是创建PhoneWindow&#xff0c;PhoneWindow是window唯一的实现类。每一个Activity都有一个PhoneWindow对象&#xff0c;PhoneWindow对象中有一个DecorView实例&#xff0c;通过这个实例来进行View相关的操作。</p> 
<p><img src="https://img-blog.csdnimg.cn/20200729172418893.png" alt="在这里插入图片描述" /></p> 
<h3><a id="23_ActivitysetContentView_113"></a>2.3 Activity.setContentView</h3> 
<p>[-&gt;Activity.java]</p> 
<p>onCreate方法里面有一个setContentView</p> 
<pre><code>   public void setContentView(&#64;LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
 
  public Window getWindow() {
        return mWindow;
    }

</code></pre> 
<p>mWindow为上面创建的PhoneWindow对象&#xff0c;其类结构如下图所示</p> 
<p><img src="https://img-blog.csdnimg.cn/20200729172440564.jpg?x-oss-process&#61;image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Nhbzg2MTU0NDMyNQ&#61;&#61;,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" /></p> 
<p>[-&gt;PhoneWindow.java]</p> 
<pre><code> public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent &#61;&#61; null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene &#61; Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            //加载布局
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb &#61; getCallback();
        if (cb !&#61; null &amp;&amp; !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet &#61; true;
    }

</code></pre> 
<h4><a id="231_installDecor_167"></a>2.3.1 installDecor</h4> 
<p>[-&gt;PhoneWindow.java]</p> 
<pre><code> private void installDecor() {
        mForceDecorInstall &#61; false;
        //如果mDecor没有初始化&#xff0c;则进行初始化
        if (mDecor &#61;&#61; null) {
            mDecor &#61; generateDecor(-1);
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted &amp;&amp; mInvalidatePanelMenuFeatures !&#61; 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        } else {
            mDecor.setWindow(this);
        }
        //mContentParent为空则初始化
        if (mContentParent &#61;&#61; null) {
            mContentParent &#61; generateLayout(mDecor);

            // Set up decor part of UI to ignore fitsSystemWindows if appropriate.
            mDecor.makeOptionalFitsSystemWindows();

            final DecorContentParent decorContentParent &#61; (DecorContentParent) mDecor.findViewById(
                    R.id.decor_content_parent);

            if (decorContentParent !&#61; null) {
                mDecorContentParent &#61; decorContentParent;
                mDecorContentParent.setWindowCallback(getCallback());
                if (mDecorContentParent.getTitle() &#61;&#61; null) {
                    mDecorContentParent.setWindowTitle(mTitle);
                }

                final int localFeatures &#61; getLocalFeatures();
                for (int i &#61; 0; i &lt; FEATURE_MAX; i&#43;&#43;) {
                    if ((localFeatures &amp; (1 &lt;&lt; i)) !&#61; 0) {
                        mDecorContentParent.initFeature(i);
                    }
                }

                mDecorContentParent.setUiOptions(mUiOptions);

                if ((mResourcesSetFlags &amp; FLAG_RESOURCE_SET_ICON) !&#61; 0 ||
                        (mIconRes !&#61; 0 &amp;&amp; !mDecorContentParent.hasIcon())) {
                    mDecorContentParent.setIcon(mIconRes);
                } else if ((mResourcesSetFlags &amp; FLAG_RESOURCE_SET_ICON) &#61;&#61; 0 &amp;&amp;
                        mIconRes &#61;&#61; 0 &amp;&amp; !mDecorContentParent.hasIcon()) {
                    mDecorContentParent.setIcon(
                            getContext().getPackageManager().getDefaultActivityIcon());
                    mResourcesSetFlags |&#61; FLAG_RESOURCE_SET_ICON_FALLBACK;
                }
                if ((mResourcesSetFlags &amp; FLAG_RESOURCE_SET_LOGO) !&#61; 0 ||
                        (mLogoRes !&#61; 0 &amp;&amp; !mDecorContentParent.hasLogo())) {
                    mDecorContentParent.setLogo(mLogoRes);
                }

                // Invalidate if the panel menu hasn&#39;t been created before this.
                // Panel menu invalidation is deferred avoiding application onCreateOptionsMenu
                // being called in the middle of onCreate or similar.
                // A pending invalidation will typically be resolved before the posted message
                // would run normally in order to satisfy instance state restoration.
                PanelFeatureState st &#61; getPanelState(FEATURE_OPTIONS_PANEL, false);
                if (!isDestroyed() &amp;&amp; (st &#61;&#61; null || st.menu &#61;&#61; null) &amp;&amp; !mIsStartingWindow) {
                    invalidatePanelMenu(FEATURE_ACTION_BAR);
                }
            } else {
                mTitleView &#61; findViewById(R.id.title);
                if (mTitleView !&#61; null) {
                    if ((getLocalFeatures() &amp; (1 &lt;&lt; FEATURE_NO_TITLE)) !&#61; 0) {
                        final View titleContainer &#61; findViewById(R.id.title_container);
                        if (titleContainer !&#61; null) {
                            titleContainer.setVisibility(View.GONE);
                        } else {
                            mTitleView.setVisibility(View.GONE);
                        }
                        mContentParent.setForeground(null);
                    } else {
                        mTitleView.setText(mTitle);
                    }
                }
            }

            if (mDecor.getBackground() &#61;&#61; null &amp;&amp; mBackgroundFallbackResource !&#61; 0) {
                mDecor.setBackgroundFallback(mBackgroundFallbackResource);
            }

            // Only inflate or create a new TransitionManager if the caller hasn&#39;t
            // already set a custom one.
            if (hasFeature(FEATURE_ACTIVITY_TRANSITIONS)) {
                if (mTransitionManager &#61;&#61; null) {
                    final int transitionRes &#61; getWindowStyle().getResourceId(
                            R.styleable.Window_windowContentTransitionManager,
                            0);
                    if (transitionRes !&#61; 0) {
                        final TransitionInflater inflater &#61; TransitionInflater.from(getContext());
                        mTransitionManager &#61; inflater.inflateTransitionManager(transitionRes,
                                mContentParent);
                    } else {
                        mTransitionManager &#61; new TransitionManager();
                    }
                }

                mEnterTransition &#61; getTransition(mEnterTransition, null,
                        R.styleable.Window_windowEnterTransition);
                mReturnTransition &#61; getTransition(mReturnTransition, USE_DEFAULT_TRANSITION,
                        R.styleable.Window_windowReturnTransition);
                mExitTransition &#61; getTransition(mExitTransition, null,
                        R.styleable.Window_windowExitTransition);
                mReenterTransition &#61; getTransition(mReenterTransition, USE_DEFAULT_TRANSITION,
                        R.styleable.Window_windowReenterTransition);
                mSharedElementEnterTransition &#61; getTransition(mSharedElementEnterTransition, null,
                        R.styleable.Window_windowSharedElementEnterTransition);
                mSharedElementReturnTransition &#61; getTransition(mSharedElementReturnTransition,
                        USE_DEFAULT_TRANSITION,
                        R.styleable.Window_windowSharedElementReturnTransition);
                mSharedElementExitTransition &#61; getTransition(mSharedElementExitTransition, null,
                        R.styleable.Window_windowSharedElementExitTransition);
                mSharedElementReenterTransition &#61; getTransition(mSharedElementReenterTransition,
                        USE_DEFAULT_TRANSITION,
                        R.styleable.Window_windowSharedElementReenterTransition);
                if (mAllowEnterTransitionOverlap &#61;&#61; null) {
                    mAllowEnterTransitionOverlap &#61; getWindowStyle().getBoolean(
                            R.styleable.Window_windowAllowEnterTransitionOverlap, true);
                }
                if (mAllowReturnTransitionOverlap &#61;&#61; null) {
                    mAllowReturnTransitionOverlap &#61; getWindowStyle().getBoolean(
                            R.styleable.Window_windowAllowReturnTransitionOverlap, true);
                }
                if (mBackgroundFadeDurationMillis &lt; 0) {
                    mBackgroundFadeDurationMillis &#61; getWindowStyle().getInteger(
                            R.styleable.Window_windowTransitionBackgroundFadeDuration,
                            DEFAULT_BACKGROUND_FADE_DURATION_MS);
                }
                if (mSharedElementsUseOverlay &#61;&#61; null) {
                    mSharedElementsUseOverlay &#61; getWindowStyle().getBoolean(
                            R.styleable.Window_windowSharedElementsUseOverlay, true);
                }
            }
        }
    }
</code></pre> 
<h4><a id="232_DecorView_311"></a>2.3.2 DecorView创建</h4> 
<pre><code>  protected DecorView generateDecor(int featureId) {
        // System process doesn&#39;t have application context and in that case we need to directly use
        // the context we have. Otherwise we want the application context, so we don&#39;t cling to the
        // activity.
        Context context;
        if (mUseDecorContext) {
            Context applicationContext &#61; getContext().getApplicationContext();
            if (applicationContext &#61;&#61; null) {
                context &#61; getContext();
            } else {
                context &#61; new DecorContext(applicationContext, getContext());
                if (mTheme !&#61; -1) {
                    context.setTheme(mTheme);
                }
            }
        } else {
            context &#61; getContext();
        }
        return new DecorView(context, featureId, this, getAttributes());
    }
</code></pre> 
<p>DecorView继承于FrameLayout&#xff0c;通过setContentView将Activity中的布局view添加DecorView中</p> 
<h4><a id="233_generateLayout_338"></a>2.3.3 generateLayout</h4> 
<pre><code>protected ViewGroup generateLayout(DecorView decor) {
        // Apply data from current theme.

        TypedArray a &#61; getWindowStyle();

        if (false) {
            System.out.println(&#34;From style:&#34;);
            String s &#61; &#34;Attrs:&#34;;
            for (int i &#61; 0; i &lt; R.styleable.Window.length; i&#43;&#43;) {
                s &#61; s &#43; &#34; &#34; &#43; Integer.toHexString(R.styleable.Window[i]) &#43; &#34;&#61;&#34;
                        &#43; a.getString(i);
            }
            System.out.println(s);
        }

        mIsFloating &#61; a.getBoolean(R.styleable.Window_windowIsFloating, false);
        int flagsToUpdate &#61; (FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR)
                &amp; (~getForcedWindowFlags());
        if (mIsFloating) {
            setLayout(WRAP_CONTENT, WRAP_CONTENT);
            setFlags(0, flagsToUpdate);
        } else {
            setFlags(FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR, flagsToUpdate);
        }

       ......

        if (params.windowAnimations &#61;&#61; 0) {
            params.windowAnimations &#61; a.getResourceId(
                    R.styleable.Window_windowAnimationStyle, 0);
        }

        // The rest are only done if this window is not embedded; otherwise,
        // the values are inherited from our container.
        if (getContainer() &#61;&#61; null) {
            if (mBackgroundDrawable &#61;&#61; null) {
                if (mBackgroundResource &#61;&#61; 0) {
                    mBackgroundResource &#61; a.getResourceId(
                            R.styleable.Window_windowBackground, 0);
                }
                if (mFrameResource &#61;&#61; 0) {
                    mFrameResource &#61; a.getResourceId(R.styleable.Window_windowFrame, 0);
                }
                mBackgroundFallbackResource &#61; a.getResourceId(
                        R.styleable.Window_windowBackgroundFallback, 0);
                if (false) {
                    System.out.println(&#34;Background: &#34;
                            &#43; Integer.toHexString(mBackgroundResource) &#43; &#34; Frame: &#34;
                            &#43; Integer.toHexString(mFrameResource));
                }
            }
            if (mLoadElevation) {
                mElevation &#61; a.getDimension(R.styleable.Window_windowElevation, 0);
            }
            mClipToOutline &#61; a.getBoolean(R.styleable.Window_windowClipToOutline, false);
            mTextColor &#61; a.getColor(R.styleable.Window_textColor, Color.TRANSPARENT);
        }

        // Inflate the window decor.

        int layoutResource;
        int features &#61; getLocalFeatures();
        // System.out.println(&#34;Features: 0x&#34; &#43; Integer.toHexString(features));
        if ((features &amp; (1 &lt;&lt; FEATURE_SWIPE_TO_DISMISS)) !&#61; 0) {
            layoutResource &#61; R.layout.screen_swipe_dismiss;
            setCloseOnSwipeEnabled(true);
        } else if ((features &amp; ((1 &lt;&lt; FEATURE_LEFT_ICON) | (1 &lt;&lt; FEATURE_RIGHT_ICON))) !&#61; 0) {
            if (mIsFloating) {
                TypedValue res &#61; new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogTitleIconsDecorLayout, res, true);
                layoutResource &#61; res.resourceId;
            } else {
                layoutResource &#61; R.layout.screen_title_icons;
            }
            // XXX Remove this once action bar supports these features.
            removeFeature(FEATURE_ACTION_BAR);
            // System.out.println(&#34;Title Icons!&#34;);
        } else if ((features &amp; ((1 &lt;&lt; FEATURE_PROGRESS) | (1 &lt;&lt; FEATURE_INDETERMINATE_PROGRESS))) !&#61; 0
                &amp;&amp; (features &amp; (1 &lt;&lt; FEATURE_ACTION_BAR)) &#61;&#61; 0) {
            // Special case for a window with only a progress bar (and title).
            // XXX Need to have a no-title version of embedded windows.
            layoutResource &#61; R.layout.screen_progress;
            // System.out.println(&#34;Progress!&#34;);
        } else if ((features &amp; (1 &lt;&lt; FEATURE_CUSTOM_TITLE)) !&#61; 0) {
            // Special case for a window with a custom title.
            // If the window is floating, we need a dialog layout
            if (mIsFloating) {
                TypedValue res &#61; new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogCustomTitleDecorLayout, res, true);
                layoutResource &#61; res.resourceId;
            } else {
                layoutResource &#61; R.layout.screen_custom_title;
            }
            // XXX Remove this once action bar supports these features.
            removeFeature(FEATURE_ACTION_BAR);
        } else if ((features &amp; (1 &lt;&lt; FEATURE_NO_TITLE)) &#61;&#61; 0) {
            // If no other features and not embedded, only need a title.
            // If the window is floating, we need a dialog layout
            if (mIsFloating) {
                TypedValue res &#61; new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogTitleDecorLayout, res, true);
                layoutResource &#61; res.resourceId;
            } else if ((features &amp; (1 &lt;&lt; FEATURE_ACTION_BAR)) !&#61; 0) {
                layoutResource &#61; a.getResourceId(
                        R.styleable.Window_windowActionBarFullscreenDecorLayout,
                        R.layout.screen_action_bar);
            } else {
                layoutResource &#61; R.layout.screen_title;
            }
            // System.out.println(&#34;Title!&#34;);
        } else if ((features &amp; (1 &lt;&lt; FEATURE_ACTION_MODE_OVERLAY)) !&#61; 0) {
            layoutResource &#61; R.layout.screen_simple_overlay_action_mode;
        } else {
            // Embedded, so no decoration is needed.
            layoutResource &#61; R.layout.screen_simple;
            // System.out.println(&#34;Simple!&#34;);
        }

        mDecor.startChanging();
        mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
        //找到contentview ID
        ViewGroup contentParent &#61; (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        if (contentParent &#61;&#61; null) {
            throw new RuntimeException(&#34;Window couldn&#39;t find content container view&#34;);
        }

        if ((features &amp; (1 &lt;&lt; FEATURE_INDETERMINATE_PROGRESS)) !&#61; 0) {
            ProgressBar progress &#61; getCircularProgressBar(false);
            if (progress !&#61; null) {
                progress.setIndeterminate(true);
            }
        }

        if ((features &amp; (1 &lt;&lt; FEATURE_SWIPE_TO_DISMISS)) !&#61; 0) {
            registerSwipeCallbacks(contentParent);
        }

        // Remaining setup -- of background and title -- that only applies
        // to top-level windows.
        //设置背景和title
        if (getContainer() &#61;&#61; null) {
            final Drawable background;
            if (mBackgroundResource !&#61; 0) {
                background &#61; getContext().getDrawable(mBackgroundResource);
            } else {
                background &#61; mBackgroundDrawable;
            }
            mDecor.setWindowBackground(background);

            final Drawable frame;
            if (mFrameResource !&#61; 0) {
                frame &#61; getContext().getDrawable(mFrameResource);
            } else {
                frame &#61; null;
            }
            mDecor.setWindowFrame(frame);

            mDecor.setElevation(mElevation);
            mDecor.setClipToOutline(mClipToOutline);

            if (mTitle !&#61; null) {
                setTitle(mTitle);
            }

            if (mTitleColor &#61;&#61; 0) {
                mTitleColor &#61; mTextColor;
            }
            setTitleColor(mTitleColor);
        }

        mDecor.finishChanging();

        return contentParent;
    }
</code></pre> 
<p>contentParent通过findviewById(com.android.internal.R.id.content)得到&#xff0c;后面通过mLayoutInflater.inflate(layoutResID, mContentParent)加载setContentView中的布局文件。</p> 
<h3><a id="24_AThandleResumeActivity_522"></a>2.4 AT.handleResumeActivity</h3> 
<p>[-&gt;ActivityThread.java]</p> 
<pre><code>  public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();
        mSomeActivitiesChanged &#61; true;

        // TODO Push resumeArgs into the activity for consideration
        //调用Activity的onResume方法
        final ActivityClientRecord r &#61; performResumeActivity(token, finalStateRequest, reason);
        if (r &#61;&#61; null) {
            // We didn&#39;t actually resume the activity, so skipping any follow-up actions.
            return;
        }

        final Activity a &#61; r.activity;

        if (localLOGV) {
            Slog.v(TAG, &#34;Resume &#34; &#43; r &#43; &#34; started activity: &#34; &#43; a.mStartedActivity
                    &#43; &#34;, hideForNow: &#34; &#43; r.hideForNow &#43; &#34;, finished: &#34; &#43; a.mFinished);
        }

        final int forwardBit &#61; isForward
                ? WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;

        // If the window hasn&#39;t yet been added to the window manager,
        // and this guy didn&#39;t finish itself or start another activity,
        // then go ahead and add the window.
        boolean willBeVisible &#61; !a.mStartedActivity;
        if (!willBeVisible) {
            try {
                willBeVisible &#61; ActivityManager.getService().willActivityBeVisible(
                        a.getActivityToken());
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
        if (r.window &#61;&#61; null &amp;&amp; !a.mFinished &amp;&amp; willBeVisible) {
            r.window &#61; r.activity.getWindow();
            View decor &#61; r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm &#61; a.getWindowManager();
            WindowManager.LayoutParams l &#61; r.window.getAttributes();
            a.mDecor &#61; decor;
            l.type &#61; WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |&#61; forwardBit;
            if (r.mPreserveWindow) {
                a.mWindowAdded &#61; true;
                r.mPreserveWindow &#61; false;
                // Normally the ViewRoot sets up callbacks with the Activity
                // in addView-&gt;ViewRootImpl#setView. If we are instead reusing
                // the decor view we have to notify the view root that the
                // callbacks may have changed.
                ViewRootImpl impl &#61; decor.getViewRootImpl();
                if (impl !&#61; null) {
                    impl.notifyChildRebuilt();
                }
            }
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded &#61; true;
                    wm.addView(decor, l);
                } else {
                    // The activity will get a callback for this {&#64;link LayoutParams} change
                    // earlier. However, at that time the decor will not be set (this is set
                    // in this method), so no action will be taken. This call ensures the
                    // callback occurs with the decor set.
                    a.onWindowAttributesChanged(l);
                }
            }

            // If the window has already been added, but during resume
            // we started another activity, then don&#39;t yet make the
            // window visible.
        } else if (!willBeVisible) {
            if (localLOGV) Slog.v(TAG, &#34;Launch &#34; &#43; r &#43; &#34; mStartedActivity set&#34;);
            r.hideForNow &#61; true;
        }

        // Get rid of anything left hanging around.
        cleanUpPendingRemoveWindows(r, false /* force */);

        // The window is now visible if it has been added, we are not
        // simply finishing, and we are not starting another activity.
        // 窗口开始设置可见
        if (!r.activity.mFinished &amp;&amp; willBeVisible &amp;&amp; r.activity.mDecor !&#61; null &amp;&amp; !r.hideForNow) {
            if (r.newConfig !&#61; null) {
                performConfigurationChangedForActivity(r, r.newConfig);
                if (DEBUG_CONFIGURATION) {
                    Slog.v(TAG, &#34;Resuming activity &#34; &#43; r.activityInfo.name &#43; &#34; with newConfig &#34;
                            &#43; r.activity.mCurrentConfig);
                }
                r.newConfig &#61; null;
            }
            if (localLOGV) Slog.v(TAG, &#34;Resuming &#34; &#43; r &#43; &#34; with isForward&#61;&#34; &#43; isForward);
            WindowManager.LayoutParams l &#61; r.window.getAttributes();
            if ((l.softInputMode
                    &amp; WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
                    !&#61; forwardBit) {
                l.softInputMode &#61; (l.softInputMode
                        &amp; (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                        | forwardBit;
                if (r.activity.mVisibleFromClient) {
                    ViewManager wm &#61; a.getWindowManager();
                    View decor &#61; r.window.getDecorView();
                    wm.updateViewLayout(decor, l);
                }
            }

            r.activity.mVisibleFromServer &#61; true;
            mNumVisibleActivities&#43;&#43;;
            if (r.activity.mVisibleFromClient) {
                //见2.5节
                r.activity.makeVisible();
            }
        }

        r.nextIdle &#61; mNewActivities;
        mNewActivities &#61; r;
        if (localLOGV) Slog.v(TAG, &#34;Scheduling idle handler for &#34; &#43; r);
        Looper.myQueue().addIdleHandler(new Idler());
    }
</code></pre> 
<h3><a id="25_ActivitymakeVisible_651"></a>2.5 Activity.makeVisible</h3> 
<p>[-&gt;Activity.java]</p> 
<pre><code>  void makeVisible() {
        if (!mWindowAdded) {
            ViewManager wm &#61; getWindowManager();
            wm.addView(mDecor, getWindow().getAttributes());
            mWindowAdded &#61; true;
        }
        mDecor.setVisibility(View.VISIBLE);
   }
</code></pre> 
<p>在 Activity.attach方法中进行的初始化</p> 
<pre><code>mWindow &#61; new PhoneWindow(this, window, activityConfigCallback);
mWindowManager &#61; mWindow.getWindowManager();
</code></pre> 
<p>mWindowManager 在Window中初始化</p> 
<pre><code>  mWindowManager &#61; ((WindowManagerImpl)wm).createLocalWindowManager(this);
</code></pre> 
<h3><a id="26_WMIaddView_679"></a>2.6 WMI.addView</h3> 
<p>[-&gt;WindowManagerImpl.java]</p> 
<pre><code> private final WindowManagerGlobal mGlobal &#61; WindowManagerGlobal.getInstance();
 &#64;Override
 public void addView(&#64;NonNull View view, &#64;NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        //添加view
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
 }
</code></pre> 
<h3><a id="27_WMGaddView_693"></a>2.7 WMG.addView</h3> 
<p>[-&gt;WindowManagerGlobal.java]</p> 
<pre><code>public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        if (view &#61;&#61; null) {
            throw new IllegalArgumentException(&#34;view must not be null&#34;);
        }
        if (display &#61;&#61; null) {
            throw new IllegalArgumentException(&#34;display must not be null&#34;);
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException(&#34;Params must be WindowManager.LayoutParams&#34;);
        }

        final WindowManager.LayoutParams wparams &#61; (WindowManager.LayoutParams) params;
        if (parentWindow !&#61; null) {
            parentWindow.adjustLayoutParamsForSubWindow(wparams);
        } else {
            // If there&#39;s no parent, then hardware acceleration for this view is
            // set from the application&#39;s hardware acceleration setting.
            final Context context &#61; view.getContext();
            if (context !&#61; null
                    &amp;&amp; (context.getApplicationInfo().flags
                            &amp; ApplicationInfo.FLAG_HARDWARE_ACCELERATED) !&#61; 0) {
                wparams.flags |&#61; WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
            }
        }

        ViewRootImpl root;
        View panelParentView &#61; null;

        synchronized (mLock) {
            // Start watching for system property changes.
            if (mSystemPropertyUpdater &#61;&#61; null) {
                mSystemPropertyUpdater &#61; new Runnable() {
                    &#64;Override public void run() {
                        synchronized (mLock) {
                            for (int i &#61; mRoots.size() - 1; i &gt;&#61; 0; --i) {
                                mRoots.get(i).loadSystemProperties();
                            }
                        }
                    }
                };
                SystemProperties.addChangeCallback(mSystemPropertyUpdater);
            }

            int index &#61; findViewLocked(view, false);
            if (index &gt;&#61; 0) {
                if (mDyingViews.contains(view)) {
                    // Don&#39;t wait for MSG_DIE to make it&#39;s way through root&#39;s queue.
                    mRoots.get(index).doDie();
                } else {
                    throw new IllegalStateException(&#34;View &#34; &#43; view
                            &#43; &#34; has already been added to the window manager.&#34;);
                }
                // The previous removeView() had not completed executing. Now it has.
            }

            // If this is a panel window, then find the window it is being
            // attached to for future reference.
            if (wparams.type &gt;&#61; WindowManager.LayoutParams.FIRST_SUB_WINDOW &amp;&amp;
                    wparams.type &lt;&#61; WindowManager.LayoutParams.LAST_SUB_WINDOW) {
                final int count &#61; mViews.size();
                for (int i &#61; 0; i &lt; count; i&#43;&#43;) {
                    if (mRoots.get(i).mWindow.asBinder() &#61;&#61; wparams.token) {
                        panelParentView &#61; mViews.get(i);
                    }
                }
            }
            
            root &#61; new ViewRootImpl(view.getContext(), display);

            view.setLayoutParams(wparams);

            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);

            // do this last because it fires off messages to start doing things
            try {
                //见2.8节
                root.setView(view, wparams, panelParentView);
            } catch (RuntimeException e) {
                // BadTokenException or InvalidDisplayException, clean up.
                if (index &gt;&#61; 0) {
                    removeViewLocked(index, true);
                }
                throw e;
            }
        }
    }
</code></pre> 
<p>WindowManagerGlobal是WindowManagerImpl中的成员变量&#xff0c;其和ViewRootImpl&#xff0c;WindowSession关系图如下&#xff0c;WindowManager和WindowManagerService是通过Session进行通信的。具体可参考WMS启动分析</p> 
<h3><a id="28_VRIsetView_791"></a>2.8 VRI.setView</h3> 
<p>[-&gt;ViewRootImpl.java]</p> 
<pre><code>public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView &#61;&#61; null) {
                mView &#61; view;

                mAttachInfo.mDisplayState &#61; mDisplay.getState();
                mDisplayManager.registerDisplayListener(mDisplayListener, mHandler);

                mViewLayoutDirectionInitial &#61; mView.getRawLayoutDirection();
                mFallbackEventHandler.setView(view);
                mWindowAttributes.copyFrom(attrs);
                if (mWindowAttributes.packageName &#61;&#61; null) {
                    mWindowAttributes.packageName &#61; mBasePackageName;
                }
                attrs &#61; mWindowAttributes;
                setTag();

                if (DEBUG_KEEP_SCREEN_ON &amp;&amp; (mClientWindowLayoutFlags
                        &amp; WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON) !&#61; 0
                        &amp;&amp; (attrs.flags&amp;WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON) &#61;&#61; 0) {
                    Slog.d(mTag, &#34;setView: FLAG_KEEP_SCREEN_ON changed from true to false!&#34;);
                }
                // Keep track of the actual window flags supplied by the client.
                mClientWindowLayoutFlags &#61; attrs.flags;

                setAccessibilityFocus(null, null);

                if (view instanceof RootViewSurfaceTaker) {
                    mSurfaceHolderCallback &#61;
                            ((RootViewSurfaceTaker)view).willYouTakeTheSurface();
                    if (mSurfaceHolderCallback !&#61; null) {
                        mSurfaceHolder &#61; new TakenSurfaceHolder();
                        mSurfaceHolder.setFormat(PixelFormat.UNKNOWN);
                        mSurfaceHolder.addCallback(mSurfaceHolderCallback);
                    }
                }

                // Compute surface insets required to draw at specified Z value.
                // TODO: Use real shadow insets for a constant max Z.
                if (!attrs.hasManualSurfaceInsets) {
                    attrs.setSurfaceInsets(view, false /*manual*/, true /*preservePrevious*/);
                }

                CompatibilityInfo compatibilityInfo &#61;
                        mDisplay.getDisplayAdjustments().getCompatibilityInfo();
                mTranslator &#61; compatibilityInfo.getTranslator();

                // If the application owns the surface, don&#39;t enable hardware acceleration
                if (mSurfaceHolder &#61;&#61; null) {
                    // While this is supposed to enable only, it can effectively disable
                    // the acceleration too.
                    enableHardwareAcceleration(attrs);
                    final boolean useMTRenderer &#61; MT_RENDERER_AVAILABLE
                            &amp;&amp; mAttachInfo.mThreadedRenderer !&#61; null;
                    if (mUseMTRenderer !&#61; useMTRenderer) {
                        // Shouldn&#39;t be resizing, as it&#39;s done only in window setup,
                        // but end just in case.
                        endDragResizing();
                        mUseMTRenderer &#61; useMTRenderer;
                    }
                }

                boolean restore &#61; false;
                if (mTranslator !&#61; null) {
                    mSurface.setCompatibilityTranslator(mTranslator);
                    restore &#61; true;
                    attrs.backup();
                    mTranslator.translateWindowLayout(attrs);
                }
                if (DEBUG_LAYOUT) Log.d(mTag, &#34;WindowLayout in setView:&#34; &#43; attrs);

                if (!compatibilityInfo.supportsScreen()) {
                    attrs.privateFlags |&#61; WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
                    mLastInCompatMode &#61; true;
                }

                mSoftInputMode &#61; attrs.softInputMode;
                mWindowAttributesChanged &#61; true;
                mWindowAttributesChangesFlag &#61; WindowManager.LayoutParams.EVERYTHING_CHANGED;
                mAttachInfo.mRootView &#61; view;
                mAttachInfo.mScalingRequired &#61; mTranslator !&#61; null;
                mAttachInfo.mApplicationScale &#61;
                        mTranslator &#61;&#61; null ? 1.0f : mTranslator.applicationScale;
                if (panelParentView !&#61; null) {
                    mAttachInfo.mPanelParentWindowToken
                            &#61; panelParentView.getApplicationWindowToken();
                }
                mAdded &#61; true;
                int res; /* &#61; WindowManagerImpl.ADD_OKAY; */

                // Schedule the first layout -before- adding to the window
                // manager, to make sure we do the relayout before receiving
                // any other events from the system.
                //调用该方法进行view的绘制
                requestLayout();
                
                if ((mWindowAttributes.inputFeatures
                        &amp; WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) &#61;&#61; 0) {
                    mInputChannel &#61; new InputChannel();
                }
                mForceDecorViewVisibility &#61; (mWindowAttributes.privateFlags
                        &amp; PRIVATE_FLAG_FORCE_DECOR_VIEW_VISIBILITY) !&#61; 0;
                try {
                    mOrigWindowType &#61; mWindowAttributes.type;
                    mAttachInfo.mRecomputeGlobalAttributes &#61; true;
                    collectViewAttributes();
                    //通过WindowSession和WindowManagerService进行通信&#xff0c;添加到显示
                    res &#61; mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), mWinFrame,
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel);
                } catch (RemoteException e) {
                    mAdded &#61; false;
                    mView &#61; null;
                    mAttachInfo.mRootView &#61; null;
                    mInputChannel &#61; null;
                    mFallbackEventHandler.setView(null);
                    unscheduleTraversals();
                    setAccessibilityFocus(null, null);
                    throw new RuntimeException(&#34;Adding window failed&#34;, e);
                } finally {
                    if (restore) {
                        attrs.restore();
                    }
                }

                if (mTranslator !&#61; null) {
                    mTranslator.translateRectInScreenToAppWindow(mAttachInfo.mContentInsets);
                }
                mPendingOverscanInsets.set(0, 0, 0, 0);
                mPendingContentInsets.set(mAttachInfo.mContentInsets);
                mPendingStableInsets.set(mAttachInfo.mStableInsets);
                mPendingDisplayCutout.set(mAttachInfo.mDisplayCutout);
                mPendingVisibleInsets.set(0, 0, 0, 0);
                mAttachInfo.mAlwaysConsumeNavBar &#61;
                        (res &amp; WindowManagerGlobal.ADD_FLAG_ALWAYS_CONSUME_NAV_BAR) !&#61; 0;
                mPendingAlwaysConsumeNavBar &#61; mAttachInfo.mAlwaysConsumeNavBar;
                if (DEBUG_LAYOUT) Log.v(mTag, &#34;Added window &#34; &#43; mWindow);
                //根据结果进行相应的处理
                if (res &lt; WindowManagerGlobal.ADD_OKAY) {
                    mAttachInfo.mRootView &#61; null;
                    mAdded &#61; false;
                    mFallbackEventHandler.setView(null);
                    unscheduleTraversals();
                    setAccessibilityFocus(null, null);
                    switch (res) {
                        case WindowManagerGlobal.ADD_BAD_APP_TOKEN:
                        case WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN:
                            throw new WindowManager.BadTokenException(
                                    &#34;Unable to add window -- token &#34; &#43; attrs.token
                                    &#43; &#34; is not valid; is your activity running?&#34;);
                        case WindowManagerGlobal.ADD_NOT_APP_TOKEN:
                            throw new WindowManager.BadTokenException(
                                    &#34;Unable to add window -- token &#34; &#43; attrs.token
                                    &#43; &#34; is not for an application&#34;);
                        case WindowManagerGlobal.ADD_APP_EXITING:
                            throw new WindowManager.BadTokenException(
                                    &#34;Unable to add window -- app for token &#34; &#43; attrs.token
                                    &#43; &#34; is exiting&#34;);
                        case WindowManagerGlobal.ADD_DUPLICATE_ADD:
                            throw new WindowManager.BadTokenException(
                                    &#34;Unable to add window -- window &#34; &#43; mWindow
                                    &#43; &#34; has already been added&#34;);
                        case WindowManagerGlobal.ADD_STARTING_NOT_NEEDED:
                            // Silently ignore -- we would have just removed it
                            // right away, anyway.
                            return;
                        case WindowManagerGlobal.ADD_MULTIPLE_SINGLETON:
                            throw new WindowManager.BadTokenException(&#34;Unable to add window &#34;
                                    &#43; mWindow &#43; &#34; -- another window of type &#34;
                                    &#43; mWindowAttributes.type &#43; &#34; already exists&#34;);
                        case WindowManagerGlobal.ADD_PERMISSION_DENIED:
                            throw new WindowManager.BadTokenException(&#34;Unable to add window &#34;
                                    &#43; mWindow &#43; &#34; -- permission denied for window type &#34;
                                    &#43; mWindowAttributes.type);
                        case WindowManagerGlobal.ADD_INVALID_DISPLAY:
                            throw new WindowManager.InvalidDisplayException(&#34;Unable to add window &#34;
                                    &#43; mWindow &#43; &#34; -- the specified display can not be found&#34;);
                        case WindowManagerGlobal.ADD_INVALID_TYPE:
                            throw new WindowManager.InvalidDisplayException(&#34;Unable to add window &#34;
                                    &#43; mWindow &#43; &#34; -- the specified window type &#34;
                                    &#43; mWindowAttributes.type &#43; &#34; is not valid&#34;);
                    }
                    throw new RuntimeException(
                            &#34;Unable to add window -- unknown error code &#34; &#43; res);
                }

                if (view instanceof RootViewSurfaceTaker) {
                    mInputQueueCallback &#61;
                        ((RootViewSurfaceTaker)view).willYouTakeTheInputQueue();
                }
                if (mInputChannel !&#61; null) {
                    if (mInputQueueCallback !&#61; null) {
                        mInputQueue &#61; new InputQueue();
                        mInputQueueCallback.onInputQueueCreated(mInputQueue);
                    }
                    mInputEventReceiver &#61; new WindowInputEventReceiver(mInputChannel,
                            Looper.myLooper());
                }

                view.assignParent(this);
                mAddedTouchMode &#61; (res &amp; WindowManagerGlobal.ADD_FLAG_IN_TOUCH_MODE) !&#61; 0;
                mAppVisible &#61; (res &amp; WindowManagerGlobal.ADD_FLAG_APP_VISIBLE) !&#61; 0;

                if (mAccessibilityManager.isEnabled()) {
                    mAccessibilityInteractionConnectionManager.ensureConnection();
                }

                if (view.getImportantForAccessibility() &#61;&#61; View.IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
                    view.setImportantForAccessibility(View.IMPORTANT_FOR_ACCESSIBILITY_YES);
                }

                // Set up the input pipeline.
                CharSequence counterSuffix &#61; attrs.getTitle();
                mSyntheticInputStage &#61; new SyntheticInputStage();
                InputStage viewPostImeStage &#61; new ViewPostImeInputStage(mSyntheticInputStage);
                InputStage nativePostImeStage &#61; new NativePostImeInputStage(viewPostImeStage,
                        &#34;aq:native-post-ime:&#34; &#43; counterSuffix);
                InputStage earlyPostImeStage &#61; new EarlyPostImeInputStage(nativePostImeStage);
                InputStage imeStage &#61; new ImeInputStage(earlyPostImeStage,
                        &#34;aq:ime:&#34; &#43; counterSuffix);
                InputStage viewPreImeStage &#61; new ViewPreImeInputStage(imeStage);
                InputStage nativePreImeStage &#61; new NativePreImeInputStage(viewPreImeStage,
                        &#34;aq:native-pre-ime:&#34; &#43; counterSuffix);

                mFirstInputStage &#61; nativePreImeStage;
                mFirstPostImeInputStage &#61; earlyPostImeStage;
                mPendingInputEventQueueLengthCounterName &#61; &#34;aq:pending:&#34; &#43; counterSuffix;
            }
        }
    }

</code></pre> 
<h3><a id="29_VRIrequestLayout_1030"></a>2.9 VRI.requestLayout</h3> 
<p>[-&gt;ViewRootImpl.java]</p> 
<pre><code>   &#64;Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested &#61; true;
            scheduleTraversals();
        }
    }
</code></pre> 
<h4><a id="291_VRIscheduleTraversals_1045"></a>2.9.1 VRI.scheduleTraversals</h4> 
<p>[-&gt;ViewRootImpl.java]</p> 
<pre><code> void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled &#61; true;
            mTraversalBarrier &#61; mHandler.getLooper().getQueue().postSyncBarrier();
            //这里向Choreographer发送消息
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
</code></pre> 
<p>mTraversalRunnable实现如下&#xff1a;</p> 
<pre><code> final class TraversalRunnable implements Runnable {
        &#64;Override
        public void run() {
            doTraversal();
        }
    }
</code></pre> 
<p>这里向Choreographer发送一个Runnable消息&#xff0c;Choreographer主要作用是获取Vsync同步信号并控制应用主线程完成图像绘制的类。</p> 
<h4><a id="292_ChoreographerpostCallback_1079"></a>2.9.2 Choreographer.postCallback</h4> 
<p>[-&gt;Choreographer.java]</p> 
<pre><code> public void postCallback(int callbackType, Runnable action, Object token) {
        postCallbackDelayed(callbackType, action, token, 0);
 }
 
  private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
       
        synchronized (mLock) {
            final long now &#61; SystemClock.uptimeMillis();
            final long dueTime &#61; now &#43; delayMillis;
            //将action Runnable添加到mCallbackQueues队列
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
             
            if (dueTime &lt;&#61; now) { //没有延迟
                scheduleFrameLocked(now);
            } else {
                //如果有延迟则发送并行消息
                Message msg &#61; mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
                msg.arg1 &#61; callbackType;
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, dueTime);
            }
        }
    }
</code></pre> 
<p>最后延迟和非延迟都会执行到scheduleFrameLocked方法</p> 
<h5><a id="2921_scheduleFrameLocked_1112"></a>2.9.2.1 scheduleFrameLocked</h5> 
<p>[-&gt;Choreographer.java]</p> 
<pre><code>private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            mFrameScheduled &#61; true;
            if (USE_VSYNC) {
                if (DEBUG_FRAMES) {
                    Log.d(TAG, &#34;Scheduling next frame on vsync.&#34;);
                }

                // If running on the Looper thread, then schedule the vsync immediately,
                // otherwise post a message to schedule the vsync from the UI thread
                // as soon as possible.
                //如果是在同一线程
                if (isRunningOnLooperThreadLocked()) {
                    scheduleVsyncLocked();
                } else {
                    Message msg &#61; mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                    msg.setAsynchronous(true);
                    //插到最前面
                    mHandler.sendMessageAtFrontOfQueue(msg);
                }
            } else {
                final long nextFrameTime &#61; Math.max(
                        mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS &#43; sFrameDelay, now);
                if (DEBUG_FRAMES) {
                    Log.d(TAG, &#34;Scheduling next frame in &#34; &#43; (nextFrameTime - now) &#43; &#34; ms.&#34;);
                }
                Message msg &#61; mHandler.obtainMessage(MSG_DO_FRAME);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, nextFrameTime);
            }
        }
    }
</code></pre> 
<p>USE_VSYNC默认是true,当前线程和Choreographer绑定的线程一致&#xff0c;调用scheduleVsyncLocked&#xff0c;否则通过mHandler发送消息&#xff0c;在Choreographer线程中处理。这两个分支最后都会执行scheduleVsyncLocked方法。</p> 
<h5><a id="2922_scheduleVsyncLocked_1153"></a>2.9.2.2 scheduleVsyncLocked</h5> 
<p>[-&gt;Choreographer.java]</p> 
<pre><code>private void scheduleVsyncLocked() {
    mDisplayEventReceiver.scheduleVsync();
}
</code></pre> 
<h5><a id="2923_scheduleVsync_1163"></a>2.9.2.3 scheduleVsync</h5> 
<p>[-&gt;DisplayEventReceiver.java]</p> 
<pre><code> public void scheduleVsync() {
        if (mReceiverPtr &#61;&#61; 0) {
            Log.w(TAG, &#34;Attempted to schedule a vertical sync pulse but the display event &#34;
                    &#43; &#34;receiver has already been disposed.&#34;);
        } else {
            nativeScheduleVsync(mReceiverPtr);
        }
    }

</code></pre> 
<p>这里调用到了native方法&#xff0c;这里主要是向底层注册Vsync信号&#xff0c;当底层有Vsync信号来时&#xff0c;会调用onVsync方法。后面文章会详细介绍Choreographer这里相关的操作。</p> 
<h5><a id="2924_FrameDisplayEventReceiver_1181"></a>2.9.2.4 FrameDisplayEventReceiver</h5> 
<p>[-&gt;FrameDisplayEventReceiver.java]</p> 
<pre><code> private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
        private boolean mHavePendingVsync;
        private long mTimestampNanos;
        private int mFrame;

        public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
            super(looper, vsyncSource);
        }

        &#64;Override
        public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
            // Ignore vsync from secondary display.
            // This can be problematic because the call to scheduleVsync() is a one-shot.
            // We need to ensure that we will still receive the vsync from the primary
            // display which is the one we really care about.  Ideally we should schedule
            // vsync for a particular display.
            // At this time Surface Flinger won&#39;t send us vsyncs for secondary displays
            // but that could change in the future so let&#39;s log a message to help us remember
            // that we need to fix this.
            if (builtInDisplayId !&#61; SurfaceControl.BUILT_IN_DISPLAY_ID_MAIN) {
                Log.d(TAG, &#34;Received vsync from secondary display, but we don&#39;t support &#34;
                        &#43; &#34;this case yet.  Choreographer needs a way to explicitly request &#34;
                        &#43; &#34;vsync for a specific display to ensure it doesn&#39;t lose track &#34;
                        &#43; &#34;of its scheduled vsync.&#34;);
                scheduleVsync();
                return;
            }

            // Post the vsync event to the Handler.
            // The idea is to prevent incoming vsync events from completely starving
            // the message queue.  If there are no messages in the queue with timestamps
            // earlier than the frame time, then the vsync event will be processed immediately.
            // Otherwise, messages that predate the vsync event will be handled first.
            long now &#61; System.nanoTime();
            if (timestampNanos &gt; now) {
                Log.w(TAG, &#34;Frame time is &#34; &#43; ((timestampNanos - now) * 0.000001f)
                        &#43; &#34; ms in the future!  Check that graphics HAL is generating vsync &#34;
                        &#43; &#34;timestamps using the correct timebase.&#34;);
                timestampNanos &#61; now;
            }

            if (mHavePendingVsync) {
                Log.w(TAG, &#34;Already have a pending vsync event.  There should only be &#34;
                        &#43; &#34;one at a time.&#34;);
            } else {
                mHavePendingVsync &#61; true;
            }

            mTimestampNanos &#61; timestampNanos;
            mFrame &#61; frame;
            //发送该Runnable
            Message msg &#61; Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        }

        &#64;Override
        public void run() {
            mHavePendingVsync &#61; false;
            doFrame(mTimestampNanos, mFrame);
        }
    }

</code></pre> 
<p>发送异步消息&#xff0c;最后会执行doFrame方法</p> 
<h5><a id="2925_doFrame_1254"></a>2.9.2.5 doFrame</h5> 
<p>[-&gt;Choreographer.java]</p> 
<pre><code>void doFrame(long frameTimeNanos, int frame) {
        final long startNanos;
        synchronized (mLock) {
            if (!mFrameScheduled) {
                return; // no work to do
            }

            if (DEBUG_JANK &amp;&amp; mDebugPrintNextFrameTimeDelta) {
                mDebugPrintNextFrameTimeDelta &#61; false;
                Log.d(TAG, &#34;Frame time delta: &#34;
                        &#43; ((frameTimeNanos - mLastFrameTimeNanos) * 0.000001f) &#43; &#34; ms&#34;);
            }

            long intendedFrameTimeNanos &#61; frameTimeNanos;
            startNanos &#61; System.nanoTime();
            final long jitterNanos &#61; startNanos - frameTimeNanos;
            //间隔时间大于1/60ms
            if (jitterNanos &gt;&#61; mFrameIntervalNanos) {
                final long skippedFrames &#61; jitterNanos / mFrameIntervalNanos;
                if (skippedFrames &gt;&#61; SKIPPED_FRAME_WARNING_LIMIT) {
                    Log.i(TAG, &#34;Skipped &#34; &#43; skippedFrames &#43; &#34; frames!  &#34;
                            &#43; &#34;The application may be doing too much work on its main thread.&#34;);
                }
                final long lastFrameOffset &#61; jitterNanos % mFrameIntervalNanos;
                if (DEBUG_JANK) {
                    Log.d(TAG, &#34;Missed vsync by &#34; &#43; (jitterNanos * 0.000001f) &#43; &#34; ms &#34;
                            &#43; &#34;which is more than the frame interval of &#34;
                            &#43; (mFrameIntervalNanos * 0.000001f) &#43; &#34; ms!  &#34;
                            &#43; &#34;Skipping &#34; &#43; skippedFrames &#43; &#34; frames and setting frame &#34;
                            &#43; &#34;time to &#34; &#43; (lastFrameOffset * 0.000001f) &#43; &#34; ms in the past.&#34;);
                }
                frameTimeNanos &#61; startNanos - lastFrameOffset;
            }

            if (frameTimeNanos &lt; mLastFrameTimeNanos) {
                if (DEBUG_JANK) {
                    Log.d(TAG, &#34;Frame time appears to be going backwards.  May be due to a &#34;
                            &#43; &#34;previously skipped frame.  Waiting for next vsync.&#34;);
                }
                scheduleVsyncLocked();
                return;
            }

            if (mFPSDivisor &gt; 1) {
                long timeSinceVsync &#61; frameTimeNanos - mLastFrameTimeNanos;
                if (timeSinceVsync &lt; (mFrameIntervalNanos * mFPSDivisor) &amp;&amp; timeSinceVsync &gt; 0) {
                    scheduleVsyncLocked();
                    return;
                }
            }

            mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
            mFrameScheduled &#61; false;
            mLastFrameTimeNanos &#61; frameTimeNanos;
        }

        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, &#34;Choreographer#doFrame&#34;);
            AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);
            
            //这里开始执行四种类型的Callback
            mFrameInfo.markInputHandlingStart();
            doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

            mFrameInfo.markAnimationsStart();
            doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);

            mFrameInfo.markPerformTraversalsStart();
            doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

            doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
        } finally {
            AnimationUtils.unlockAnimationClock();
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }

        if (DEBUG_FRAMES) {
            final long endNanos &#61; System.nanoTime();
            Log.d(TAG, &#34;Frame &#34; &#43; frame &#43; &#34;: Finished, took &#34;
                    &#43; (endNanos - startNanos) * 0.000001f &#43; &#34; ms, latency &#34;
                    &#43; (startNanos - frameTimeNanos) * 0.000001f &#43; &#34; ms.&#34;);
        }
    }

</code></pre> 
<p>这里主要是执行队列里面的任务&#xff0c;例如之前添加的mTraversalRunnable。</p> 
<h5><a id="2926_doCallbacks_1347"></a>2.9.2.6 doCallbacks</h5> 
<p>[-&gt;Choreographer.java]</p> 
<pre><code> void doCallbacks(int callbackType, long frameTimeNanos) {
        CallbackRecord callbacks;
        synchronized (mLock) {
            // We use &#34;now&#34; to determine when callbacks become due because it&#39;s possible
            // for earlier processing phases in a frame to post callbacks that should run
            // in a following phase, such as an input event that causes an animation to start.
            final long now &#61; System.nanoTime();
            callbacks &#61; mCallbackQueues[callbackType].extractDueCallbacksLocked(
                    now / TimeUtils.NANOS_PER_MS);
            if (callbacks &#61;&#61; null) {
                return;
            }
            mCallbacksRunning &#61; true;

            // Update the frame time if necessary when committing the frame.
            // We only update the frame time if we are more than 2 frames late reaching
            // the commit phase.  This ensures that the frame time which is observed by the
            // callbacks will always increase from one frame to the next and never repeat.
            // We never want the next frame&#39;s starting frame time to end up being less than
            // or equal to the previous frame&#39;s commit frame time.  Keep in mind that the
            // next frame has most likely already been scheduled by now so we play it
            // safe by ensuring the commit time is always at least one frame behind.
            if (callbackType &#61;&#61; Choreographer.CALLBACK_COMMIT) {
                final long jitterNanos &#61; now - frameTimeNanos;
                Trace.traceCounter(Trace.TRACE_TAG_VIEW, &#34;jitterNanos&#34;, (int) jitterNanos);
                if (jitterNanos &gt;&#61; 2 * mFrameIntervalNanos) {
                    final long lastFrameOffset &#61; jitterNanos % mFrameIntervalNanos
                            &#43; mFrameIntervalNanos;
                    if (DEBUG_JANK) {
                        Log.d(TAG, &#34;Commit callback delayed by &#34; &#43; (jitterNanos * 0.000001f)
                                &#43; &#34; ms which is more than twice the frame interval of &#34;
                                &#43; (mFrameIntervalNanos * 0.000001f) &#43; &#34; ms!  &#34;
                                &#43; &#34;Setting frame time to &#34; &#43; (lastFrameOffset * 0.000001f)
                                &#43; &#34; ms in the past.&#34;);
                        mDebugPrintNextFrameTimeDelta &#61; true;
                    }
                    frameTimeNanos &#61; now - lastFrameOffset;
                    mLastFrameTimeNanos &#61; frameTimeNanos;
                }
            }
        }
        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, CALLBACK_TRACE_TITLES[callbackType]);
            for (CallbackRecord c &#61; callbacks; c !&#61; null; c &#61; c.next) {
                if (DEBUG_FRAMES) {
                    Log.d(TAG, &#34;RunCallback: type&#61;&#34; &#43; callbackType
                            &#43; &#34;, action&#61;&#34; &#43; c.action &#43; &#34;, token&#61;&#34; &#43; c.token
                            &#43; &#34;, latencyMillis&#61;&#34; &#43; (SystemClock.uptimeMillis() - c.dueTime));
                }
                //执行任务里面的run方法
                c.run(frameTimeNanos);
            }
        } finally {
            synchronized (mLock) {
                mCallbacksRunning &#61; false;
                do {
                    final CallbackRecord next &#61; callbacks.next;
                    recycleCallbackLocked(callbacks);
                    callbacks &#61; next;
                } while (callbacks !&#61; null);
            }
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
</code></pre> 
<h4><a id="293__VRIdoTraversal_1418"></a>2.9.3 VRI.doTraversal</h4> 
<p>[-&gt;ViewRootImpl.java]</p> 
<pre><code> void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled &#61; false;
            //去除屏障
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing(&#34;ViewAncestor&#34;);
            }
            //见2.9.4节
            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile &#61; false;
            }
        }
    }
</code></pre> 
<h4><a id="294_VRIperformTraversals_1443"></a>2.9.4 VRI.performTraversals</h4> 
<p>[-&gt;ViewRootImpl.java]</p> 
<pre><code> private void performTraversals() {
        // cache mView since it is used so much below...
        final View host &#61; mView;

        if (DBG) {
            System.out.println(&#34;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#34;);
            System.out.println(&#34;performTraversals&#34;);
            host.debug();
        }

        if (host &#61;&#61; null || !mAdded)
            return;

        mIsInTraversal &#61; true;
        mWillDrawSoon &#61; true;
        boolean windowSizeMayChange &#61; false;
        boolean newSurface &#61; false;
        boolean surfaceChanged &#61; false;
        WindowManager.LayoutParams lp &#61; mWindowAttributes;

        int desiredWindowWidth;
        int desiredWindowHeight;

        final int viewVisibility &#61; getHostVisibility();
        final boolean viewVisibilityChanged &#61; !mFirst
                &amp;&amp; (mViewVisibility !&#61; viewVisibility || mNewSurfaceNeeded
                // Also check for possible double visibility update, which will make current
                // viewVisibility value equal to mViewVisibility and we may miss it.
                || mAppVisibilityChanged);
        mAppVisibilityChanged &#61; false;
        final boolean viewUserVisibilityChanged &#61; !mFirst &amp;&amp;
                ((mViewVisibility &#61;&#61; View.VISIBLE) !&#61; (viewVisibility &#61;&#61; View.VISIBLE));

        WindowManager.LayoutParams params &#61; null;
        if (mWindowAttributesChanged) {
            mWindowAttributesChanged &#61; false;
            surfaceChanged &#61; true;
            params &#61; lp;
        }
        CompatibilityInfo compatibilityInfo &#61;
                mDisplay.getDisplayAdjustments().getCompatibilityInfo();
        if (compatibilityInfo.supportsScreen() &#61;&#61; mLastInCompatMode) {
            params &#61; lp;
            mFullRedrawNeeded &#61; true;
            mLayoutRequested &#61; true;
            if (mLastInCompatMode) {
                params.privateFlags &amp;&#61; ~WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
                mLastInCompatMode &#61; false;
            } else {
                params.privateFlags |&#61; WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
                mLastInCompatMode &#61; true;
            }
        }

        mWindowAttributesChangesFlag &#61; 0;

        Rect frame &#61; mWinFrame;
        if (mFirst) {
            mFullRedrawNeeded &#61; true;
            mLayoutRequested &#61; true;

            final Configuration config &#61; mContext.getResources().getConfiguration();
            if (shouldUseDisplaySize(lp)) {
                // NOTE -- system code, won&#39;t try to do compat mode.
                Point size &#61; new Point();
                mDisplay.getRealSize(size);
                desiredWindowWidth &#61; size.x;
                desiredWindowHeight &#61; size.y;
            } else {
                desiredWindowWidth &#61; mWinFrame.width();
                desiredWindowHeight &#61; mWinFrame.height();
            }

            // We used to use the following condition to choose 32 bits drawing caches:
            // PixelFormat.hasAlpha(lp.format) || lp.format &#61;&#61; PixelFormat.RGBX_8888
            // However, windows are now always 32 bits by default, so choose 32 bits
            mAttachInfo.mUse32BitDrawingCache &#61; true;
            mAttachInfo.mHasWindowFocus &#61; false;
            mAttachInfo.mWindowVisibility &#61; viewVisibility;
            mAttachInfo.mRecomputeGlobalAttributes &#61; false;
            mLastConfigurationFromResources.setTo(config);
            mLastSystemUiVisibility &#61; mAttachInfo.mSystemUiVisibility;
            // Set the layout direction if it has not been set before (inherit is the default)
            if (mViewLayoutDirectionInitial &#61;&#61; View.LAYOUT_DIRECTION_INHERIT) {
                host.setLayoutDirection(config.getLayoutDirection());
            }
            host.dispatchAttachedToWindow(mAttachInfo, 0);
            mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(true);
            dispatchApplyInsets(host);
        } else {
            desiredWindowWidth &#61; frame.width();
            desiredWindowHeight &#61; frame.height();
            if (desiredWindowWidth !&#61; mWidth || desiredWindowHeight !&#61; mHeight) {
                if (DEBUG_ORIENTATION) Log.v(mTag, &#34;View &#34; &#43; host &#43; &#34; resized to: &#34; &#43; frame);
                mFullRedrawNeeded &#61; true;
                mLayoutRequested &#61; true;
                windowSizeMayChange &#61; true;
            }
        }

        if (viewVisibilityChanged) {
            mAttachInfo.mWindowVisibility &#61; viewVisibility;
            host.dispatchWindowVisibilityChanged(viewVisibility);
            if (viewUserVisibilityChanged) {
                host.dispatchVisibilityAggregated(viewVisibility &#61;&#61; View.VISIBLE);
            }
            if (viewVisibility !&#61; View.VISIBLE || mNewSurfaceNeeded) {
                endDragResizing();
                destroyHardwareResources();
            }
            if (viewVisibility &#61;&#61; View.GONE) {
                // After making a window gone, we will count it as being
                // shown for the first time the next time it gets focus.
                mHasHadWindowFocus &#61; false;
            }
        }

        // Non-visible windows can&#39;t hold accessibility focus.
        if (mAttachInfo.mWindowVisibility !&#61; View.VISIBLE) {
            host.clearAccessibilityFocus();
        }

        // Execute enqueued actions on every traversal in case a detached view enqueued an action
        getRunQueue().executeActions(mAttachInfo.mHandler);

        boolean insetsChanged &#61; false;

        boolean layoutRequested &#61; mLayoutRequested &amp;&amp; (!mStopped || mReportNextDraw);
        if (layoutRequested) {

            final Resources res &#61; mView.getContext().getResources();

            if (mFirst) {
                // make sure touch mode code executes by setting cached value
                // to opposite of the added touch mode.
                mAttachInfo.mInTouchMode &#61; !mAddedTouchMode;
                ensureTouchModeLocally(mAddedTouchMode);
            } else {
                if (!mPendingOverscanInsets.equals(mAttachInfo.mOverscanInsets)) {
                    insetsChanged &#61; true;
                }
                if (!mPendingContentInsets.equals(mAttachInfo.mContentInsets)) {
                    insetsChanged &#61; true;
                }
                if (!mPendingStableInsets.equals(mAttachInfo.mStableInsets)) {
                    insetsChanged &#61; true;
                }
                if (!mPendingDisplayCutout.equals(mAttachInfo.mDisplayCutout)) {
                    insetsChanged &#61; true;
                }
                if (!mPendingVisibleInsets.equals(mAttachInfo.mVisibleInsets)) {
                    mAttachInfo.mVisibleInsets.set(mPendingVisibleInsets);
                    if (DEBUG_LAYOUT) Log.v(mTag, &#34;Visible insets changing to: &#34;
                            &#43; mAttachInfo.mVisibleInsets);
                }
                if (!mPendingOutsets.equals(mAttachInfo.mOutsets)) {
                    insetsChanged &#61; true;
                }
                if (mPendingAlwaysConsumeNavBar !&#61; mAttachInfo.mAlwaysConsumeNavBar) {
                    insetsChanged &#61; true;
                }
                if (lp.width &#61;&#61; ViewGroup.LayoutParams.WRAP_CONTENT
                        || lp.height &#61;&#61; ViewGroup.LayoutParams.WRAP_CONTENT) {
                    windowSizeMayChange &#61; true;

                    if (shouldUseDisplaySize(lp)) {
                        // NOTE -- system code, won&#39;t try to do compat mode.
                        Point size &#61; new Point();
                        mDisplay.getRealSize(size);
                        desiredWindowWidth &#61; size.x;
                        desiredWindowHeight &#61; size.y;
                    } else {
                        Configuration config &#61; res.getConfiguration();
                        desiredWindowWidth &#61; dipToPx(config.screenWidthDp);
                        desiredWindowHeight &#61; dipToPx(config.screenHeightDp);
                    }
                }
            }

            // Ask host how big it wants to be
            windowSizeMayChange |&#61; measureHierarchy(host, lp, res,
                    desiredWindowWidth, desiredWindowHeight);
        }

        if (collectViewAttributes()) {
            params &#61; lp;
        }
        if (mAttachInfo.mForceReportNewAttributes) {
            mAttachInfo.mForceReportNewAttributes &#61; false;
            params &#61; lp;
        }

        if (mFirst || mAttachInfo.mViewVisibilityChanged) {
            mAttachInfo.mViewVisibilityChanged &#61; false;
            int resizeMode &#61; mSoftInputMode &amp;
                    WindowManager.LayoutParams.SOFT_INPUT_MASK_ADJUST;
            // If we are in auto resize mode, then we need to determine
            // what mode to use now.
            if (resizeMode &#61;&#61; WindowManager.LayoutParams.SOFT_INPUT_ADJUST_UNSPECIFIED) {
                final int N &#61; mAttachInfo.mScrollContainers.size();
                for (int i&#61;0; i&lt;N; i&#43;&#43;) {
                    if (mAttachInfo.mScrollContainers.get(i).isShown()) {
                        resizeMode &#61; WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE;
                    }
                }
                if (resizeMode &#61;&#61; 0) {
                    resizeMode &#61; WindowManager.LayoutParams.SOFT_INPUT_ADJUST_PAN;
                }
                if ((lp.softInputMode &amp;
                        WindowManager.LayoutParams.SOFT_INPUT_MASK_ADJUST) !&#61; resizeMode) {
                    lp.softInputMode &#61; (lp.softInputMode &amp;
                            ~WindowManager.LayoutParams.SOFT_INPUT_MASK_ADJUST) |
                            resizeMode;
                    params &#61; lp;
                }
            }
        }

        if (params !&#61; null) {
            if ((host.mPrivateFlags &amp; View.PFLAG_REQUEST_TRANSPARENT_REGIONS) !&#61; 0) {
                if (!PixelFormat.formatHasAlpha(params.format)) {
                    params.format &#61; PixelFormat.TRANSLUCENT;
                }
            }
            mAttachInfo.mOverscanRequested &#61; (params.flags
                    &amp; WindowManager.LayoutParams.FLAG_LAYOUT_IN_OVERSCAN) !&#61; 0;
        }

        if (mApplyInsetsRequested) {
            mApplyInsetsRequested &#61; false;
            mLastOverscanRequested &#61; mAttachInfo.mOverscanRequested;
            dispatchApplyInsets(host);
            if (mLayoutRequested) {
                // Short-circuit catching a new layout request here, so
                // we don&#39;t need to go through two layout passes when things
                // change due to fitting system windows, which can happen a lot.
                windowSizeMayChange |&#61; measureHierarchy(host, lp,
                        mView.getContext().getResources(),
                        desiredWindowWidth, desiredWindowHeight);
            }
        }

        if (layoutRequested) {
            // Clear this now, so that if anything requests a layout in the
            // rest of this function we will catch it and re-run a full
            // layout pass.
            mLayoutRequested &#61; false;
        }

        boolean windowShouldResize &#61; layoutRequested &amp;&amp; windowSizeMayChange
            &amp;&amp; ((mWidth !&#61; host.getMeasuredWidth() || mHeight !&#61; host.getMeasuredHeight())
                || (lp.width &#61;&#61; ViewGroup.LayoutParams.WRAP_CONTENT &amp;&amp;
                        frame.width() &lt; desiredWindowWidth &amp;&amp; frame.width() !&#61; mWidth)
                || (lp.height &#61;&#61; ViewGroup.LayoutParams.WRAP_CONTENT &amp;&amp;
                        frame.height() &lt; desiredWindowHeight &amp;&amp; frame.height() !&#61; mHeight));
        windowShouldResize |&#61; mDragResizing &amp;&amp; mResizeMode &#61;&#61; RESIZE_MODE_FREEFORM;

        // If the activity was just relaunched, it might have unfrozen the task bounds (while
        // relaunching), so we need to force a call into window manager to pick up the latest
        // bounds.
        windowShouldResize |&#61; mActivityRelaunched;

        // Determine whether to compute insets.
        // If there are no inset listeners remaining then we may still need to compute
        // insets in case the old insets were non-empty and must be reset.
        final boolean computesInternalInsets &#61;
                mAttachInfo.mTreeObserver.hasComputeInternalInsetsListeners()
                || mAttachInfo.mHasNonEmptyGivenInternalInsets;

        boolean insetsPending &#61; false;
        int relayoutResult &#61; 0;
        boolean updatedConfiguration &#61; false;

        final int surfaceGenerationId &#61; mSurface.getGenerationId();

        final boolean isViewVisible &#61; viewVisibility &#61;&#61; View.VISIBLE;
        final boolean windowRelayoutWasForced &#61; mForceNextWindowRelayout;
        if (mFirst || windowShouldResize || insetsChanged ||
                viewVisibilityChanged || params !&#61; null || mForceNextWindowRelayout) {
            mForceNextWindowRelayout &#61; false;

            if (isViewVisible) {
                // If this window is giving internal insets to the window
                // manager, and it is being added or changing its visibility,
                // then we want to first give the window manager &#34;fake&#34;
                // insets to cause it to effectively ignore the content of
                // the window during layout.  This avoids it briefly causing
                // other windows to resize/move based on the raw frame of the
                // window, waiting until we can finish laying out this window
                // and get back to the window manager with the ultimately
                // computed insets.
                insetsPending &#61; computesInternalInsets &amp;&amp; (mFirst || viewVisibilityChanged);
            }

            if (mSurfaceHolder !&#61; null) {
                mSurfaceHolder.mSurfaceLock.lock();
                mDrawingAllowed &#61; true;
            }

            boolean hwInitialized &#61; false;
            boolean contentInsetsChanged &#61; false;
            boolean hadSurface &#61; mSurface.isValid();

            try {
                if (DEBUG_LAYOUT) {
                    Log.i(mTag, &#34;host&#61;w:&#34; &#43; host.getMeasuredWidth() &#43; &#34;, h:&#34; &#43;
                            host.getMeasuredHeight() &#43; &#34;, params&#61;&#34; &#43; params);
                }

                if (mAttachInfo.mThreadedRenderer !&#61; null) {
                    // relayoutWindow may decide to destroy mSurface. As that decision
                    // happens in WindowManager service, we need to be defensive here
                    // and stop using the surface in case it gets destroyed.
                    if (mAttachInfo.mThreadedRenderer.pauseSurface(mSurface)) {
                        // Animations were running so we need to push a frame
                        // to resume them
                        mDirty.set(0, 0, mWidth, mHeight);
                    }
                    mChoreographer.mFrameInfo.addFlags(FrameInfo.FLAG_WINDOW_LAYOUT_CHANGED);
                }
                relayoutResult &#61; relayoutWindow(params, viewVisibility, insetsPending);

                if (DEBUG_LAYOUT) Log.v(mTag, &#34;relayout: frame&#61;&#34; &#43; frame.toShortString()
                        &#43; &#34; overscan&#61;&#34; &#43; mPendingOverscanInsets.toShortString()
                        &#43; &#34; content&#61;&#34; &#43; mPendingContentInsets.toShortString()
                        &#43; &#34; visible&#61;&#34; &#43; mPendingVisibleInsets.toShortString()
                        &#43; &#34; stable&#61;&#34; &#43; mPendingStableInsets.toShortString()
                        &#43; &#34; cutout&#61;&#34; &#43; mPendingDisplayCutout.get().toString()
                        &#43; &#34; outsets&#61;&#34; &#43; mPendingOutsets.toShortString()
                        &#43; &#34; surface&#61;&#34; &#43; mSurface);

                // If the pending {&#64;link MergedConfiguration} handed back from
                // {&#64;link #relayoutWindow} does not match the one last reported,
                // WindowManagerService has reported back a frame from a configuration not yet
                // handled by the client. In this case, we need to accept the configuration so we
                // do not lay out and draw with the wrong configuration.
                if (!mPendingMergedConfiguration.equals(mLastReportedMergedConfiguration)) {
                    if (DEBUG_CONFIGURATION) Log.v(mTag, &#34;Visible with new config: &#34;
                            &#43; mPendingMergedConfiguration.getMergedConfiguration());
                    performConfigurationChange(mPendingMergedConfiguration, !mFirst,
                            INVALID_DISPLAY /* same display */);
                    updatedConfiguration &#61; true;
                }

                final boolean overscanInsetsChanged &#61; !mPendingOverscanInsets.equals(
                        mAttachInfo.mOverscanInsets);
                contentInsetsChanged &#61; !mPendingContentInsets.equals(
                        mAttachInfo.mContentInsets);
                final boolean visibleInsetsChanged &#61; !mPendingVisibleInsets.equals(
                        mAttachInfo.mVisibleInsets);
                final boolean stableInsetsChanged &#61; !mPendingStableInsets.equals(
                        mAttachInfo.mStableInsets);
                final boolean cutoutChanged &#61; !mPendingDisplayCutout.equals(
                        mAttachInfo.mDisplayCutout);
                final boolean outsetsChanged &#61; !mPendingOutsets.equals(mAttachInfo.mOutsets);
                final boolean surfaceSizeChanged &#61; (relayoutResult
                        &amp; WindowManagerGlobal.RELAYOUT_RES_SURFACE_RESIZED) !&#61; 0;
                surfaceChanged |&#61; surfaceSizeChanged;
                final boolean alwaysConsumeNavBarChanged &#61;
                        mPendingAlwaysConsumeNavBar !&#61; mAttachInfo.mAlwaysConsumeNavBar;
                if (contentInsetsChanged) {
                    mAttachInfo.mContentInsets.set(mPendingContentInsets);
                    if (DEBUG_LAYOUT) Log.v(mTag, &#34;Content insets changing to: &#34;
                            &#43; mAttachInfo.mContentInsets);
                }
                if (overscanInsetsChanged) {
                    mAttachInfo.mOverscanInsets.set(mPendingOverscanInsets);
                    if (DEBUG_LAYOUT) Log.v(mTag, &#34;Overscan insets changing to: &#34;
                            &#43; mAttachInfo.mOverscanInsets);
                    // Need to relayout with content insets.
                    contentInsetsChanged &#61; true;
                }
                if (stableInsetsChanged) {
                    mAttachInfo.mStableInsets.set(mPendingStableInsets);
                    if (DEBUG_LAYOUT) Log.v(mTag, &#34;Decor insets changing to: &#34;
                            &#43; mAttachInfo.mStableInsets);
                    // Need to relayout with content insets.
                    contentInsetsChanged &#61; true;
                }
                if (cutoutChanged) {
                    mAttachInfo.mDisplayCutout.set(mPendingDisplayCutout);
                    if (DEBUG_LAYOUT) {
                        Log.v(mTag, &#34;DisplayCutout changing to: &#34; &#43; mAttachInfo.mDisplayCutout);
                    }
                    // Need to relayout with content insets.
                    contentInsetsChanged &#61; true;
                }
                if (alwaysConsumeNavBarChanged) {
                    mAttachInfo.mAlwaysConsumeNavBar &#61; mPendingAlwaysConsumeNavBar;
                    contentInsetsChanged &#61; true;
                }
                if (contentInsetsChanged || mLastSystemUiVisibility !&#61;
                        mAttachInfo.mSystemUiVisibility || mApplyInsetsRequested
                        || mLastOverscanRequested !&#61; mAttachInfo.mOverscanRequested
                        || outsetsChanged) {
                    mLastSystemUiVisibility &#61; mAttachInfo.mSystemUiVisibility;
                    mLastOverscanRequested &#61; mAttachInfo.mOverscanRequested;
                    mAttachInfo.mOutsets.set(mPendingOutsets);
                    mApplyInsetsRequested &#61; false;
                    dispatchApplyInsets(host);
                }
                if (visibleInsetsChanged) {
                    mAttachInfo.mVisibleInsets.set(mPendingVisibleInsets);
                    if (DEBUG_LAYOUT) Log.v(mTag, &#34;Visible insets changing to: &#34;
                            &#43; mAttachInfo.mVisibleInsets);
                }

                if (!hadSurface) {
                    if (mSurface.isValid()) {
                        // If we are creating a new surface, then we need to
                        // completely redraw it.  Also, when we get to the
                        // point of drawing it we will hold off and schedule
                        // a new traversal instead.  This is so we can tell the
                        // window manager about all of the windows being displayed
                        // before actually drawing them, so it can display then
                        // all at once.
                        newSurface &#61; true;
                        mFullRedrawNeeded &#61; true;
                        mPreviousTransparentRegion.setEmpty();

                        // Only initialize up-front if transparent regions are not
                        // requested, otherwise defer to see if the entire window
                        // will be transparent
                        if (mAttachInfo.mThreadedRenderer !&#61; null) {
                            try {
                                hwInitialized &#61; mAttachInfo.mThreadedRenderer.initialize(
                                        mSurface);
                                if (hwInitialized &amp;&amp; (host.mPrivateFlags
                                        &amp; View.PFLAG_REQUEST_TRANSPARENT_REGIONS) &#61;&#61; 0) {
                                    // Don&#39;t pre-allocate if transparent regions
                                    // are requested as they may not be needed
                                    mAttachInfo.mThreadedRenderer.allocateBuffers(mSurface);
                                }
                            } catch (OutOfResourcesException e) {
                                handleOutOfResourcesException(e);
                                return;
                            }
                        }
                    }
                } else if (!mSurface.isValid()) {
                    // If the surface has been removed, then reset the scroll
                    // positions.
                    if (mLastScrolledFocus !&#61; null) {
                        mLastScrolledFocus.clear();
                    }
                    mScrollY &#61; mCurScrollY &#61; 0;
                    if (mView instanceof RootViewSurfaceTaker) {
                        ((RootViewSurfaceTaker) mView).onRootViewScrollYChanged(mCurScrollY);
                    }
                    if (mScroller !&#61; null) {
                        mScroller.abortAnimation();
                    }
                    // Our surface is gone
                    if (mAttachInfo.mThreadedRenderer !&#61; null &amp;&amp;
                            mAttachInfo.mThreadedRenderer.isEnabled()) {
                        mAttachInfo.mThreadedRenderer.destroy();
                    }
                } else if ((surfaceGenerationId !&#61; mSurface.getGenerationId()
                        || surfaceSizeChanged || windowRelayoutWasForced)
                        &amp;&amp; mSurfaceHolder &#61;&#61; null
                        &amp;&amp; mAttachInfo.mThreadedRenderer !&#61; null) {
                    mFullRedrawNeeded &#61; true;
                    try {
                        // Need to do updateSurface (which leads to CanvasContext::setSurface and
                        // re-create the EGLSurface) if either the Surface changed (as indicated by
                        // generation id), or WindowManager changed the surface size. The latter is
                        // because on some chips, changing the consumer side&#39;s BufferQueue size may
                        // not take effect immediately unless we create a new EGLSurface.
                        // Note that frame size change doesn&#39;t always imply surface size change (eg.
                        // drag resizing uses fullscreen surface), need to check surfaceSizeChanged
                        // flag from WindowManager.
                        mAttachInfo.mThreadedRenderer.updateSurface(mSurface);
                    } catch (OutOfResourcesException e) {
                        handleOutOfResourcesException(e);
                        return;
                    }
                }

                final boolean freeformResizing &#61; (relayoutResult
                        &amp; WindowManagerGlobal.RELAYOUT_RES_DRAG_RESIZING_FREEFORM) !&#61; 0;
                final boolean dockedResizing &#61; (relayoutResult
                        &amp; WindowManagerGlobal.RELAYOUT_RES_DRAG_RESIZING_DOCKED) !&#61; 0;
                final boolean dragResizing &#61; freeformResizing || dockedResizing;
                if (mDragResizing !&#61; dragResizing) {
                    if (dragResizing) {
                        mResizeMode &#61; freeformResizing
                                ? RESIZE_MODE_FREEFORM
                                : RESIZE_MODE_DOCKED_DIVIDER;
                        // TODO: Need cutout?
                        startDragResizing(mPendingBackDropFrame,
                                mWinFrame.equals(mPendingBackDropFrame), mPendingVisibleInsets,
                                mPendingStableInsets, mResizeMode);
                    } else {
                        // We shouldn&#39;t come here, but if we come we should end the resize.
                        endDragResizing();
                    }
                }
                if (!mUseMTRenderer) {
                    if (dragResizing) {
                        mCanvasOffsetX &#61; mWinFrame.left;
                        mCanvasOffsetY &#61; mWinFrame.top;
                    } else {
                        mCanvasOffsetX &#61; mCanvasOffsetY &#61; 0;
                    }
                }
            } catch (RemoteException e) {
            }

            if (DEBUG_ORIENTATION) Log.v(
                    TAG, &#34;Relayout returned: frame&#61;&#34; &#43; frame &#43; &#34;, surface&#61;&#34; &#43; mSurface);

            mAttachInfo.mWindowLeft &#61; frame.left;
            mAttachInfo.mWindowTop &#61; frame.top;

            // !!FIXME!! This next section handles the case where we did not get the
            // window size we asked for. We should avoid this by getting a maximum size from
            // the window session beforehand.
            if (mWidth !&#61; frame.width() || mHeight !&#61; frame.height()) {
                mWidth &#61; frame.width();
                mHeight &#61; frame.height();
            }

            if (mSurfaceHolder !&#61; null) {
                // The app owns the surface; tell it about what is going on.
                if (mSurface.isValid()) {
                    // XXX .copyFrom() doesn&#39;t work!
                    //mSurfaceHolder.mSurface.copyFrom(mSurface);
                    mSurfaceHolder.mSurface &#61; mSurface;
                }
                mSurfaceHolder.setSurfaceFrameSize(mWidth, mHeight);
                mSurfaceHolder.mSurfaceLock.unlock();
                if (mSurface.isValid()) {
                    if (!hadSurface) {
                        mSurfaceHolder.ungetCallbacks();

                        mIsCreating &#61; true;
                        SurfaceHolder.Callback callbacks[] &#61; mSurfaceHolder.getCallbacks();
                        if (callbacks !&#61; null) {
                            for (SurfaceHolder.Callback c : callbacks) {
                                c.surfaceCreated(mSurfaceHolder);
                            }
                        }
                        surfaceChanged &#61; true;
                    }
                    if (surfaceChanged || surfaceGenerationId !&#61; mSurface.getGenerationId()) {
                        SurfaceHolder.Callback callbacks[] &#61; mSurfaceHolder.getCallbacks();
                        if (callbacks !&#61; null) {
                            for (SurfaceHolder.Callback c : callbacks) {
                                c.surfaceChanged(mSurfaceHolder, lp.format,
                                        mWidth, mHeight);
                            }
                        }
                    }
                    mIsCreating &#61; false;
                } else if (hadSurface) {
                    notifySurfaceDestroyed();
                    mSurfaceHolder.mSurfaceLock.lock();
                    try {
                        mSurfaceHolder.mSurface &#61; new Surface();
                    } finally {
                        mSurfaceHolder.mSurfaceLock.unlock();
                    }
                }
            }

            final ThreadedRenderer threadedRenderer &#61; mAttachInfo.mThreadedRenderer;
            if (threadedRenderer !&#61; null &amp;&amp; threadedRenderer.isEnabled()) {
                if (hwInitialized
                        || mWidth !&#61; threadedRenderer.getWidth()
                        || mHeight !&#61; threadedRenderer.getHeight()
                        || mNeedsRendererSetup) {
                    threadedRenderer.setup(mWidth, mHeight, mAttachInfo,
                            mWindowAttributes.surfaceInsets);
                    mNeedsRendererSetup &#61; false;
                }
            }

            if (!mStopped || mReportNextDraw) {
                boolean focusChangedDueToTouchMode &#61; ensureTouchModeLocally(
                        (relayoutResult&amp;WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) !&#61; 0);
                if (focusChangedDueToTouchMode || mWidth !&#61; host.getMeasuredWidth()
                        || mHeight !&#61; host.getMeasuredHeight() || contentInsetsChanged ||
                        updatedConfiguration) {
                    int childWidthMeasureSpec &#61; getRootMeasureSpec(mWidth, lp.width);
                    int childHeightMeasureSpec &#61; getRootMeasureSpec(mHeight, lp.height);

                    if (DEBUG_LAYOUT) Log.v(mTag, &#34;Ooops, something changed!  mWidth&#61;&#34;
                            &#43; mWidth &#43; &#34; measuredWidth&#61;&#34; &#43; host.getMeasuredWidth()
                            &#43; &#34; mHeight&#61;&#34; &#43; mHeight
                            &#43; &#34; measuredHeight&#61;&#34; &#43; host.getMeasuredHeight()
                            &#43; &#34; coveredInsetsChanged&#61;&#34; &#43; contentInsetsChanged);

                     // Ask host how big it wants to be
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

                    // Implementation of weights from WindowManager.LayoutParams
                    // We just grow the dimensions as needed and re-measure if
                    // needs be
                    int width &#61; host.getMeasuredWidth();
                    int height &#61; host.getMeasuredHeight();
                    boolean measureAgain &#61; false;

                    if (lp.horizontalWeight &gt; 0.0f) {
                        width &#43;&#61; (int) ((mWidth - width) * lp.horizontalWeight);
                        childWidthMeasureSpec &#61; MeasureSpec.makeMeasureSpec(width,
                                MeasureSpec.EXACTLY);
                        measureAgain &#61; true;
                    }
                    if (lp.verticalWeight &gt; 0.0f) {
                        height &#43;&#61; (int) ((mHeight - height) * lp.verticalWeight);
                        childHeightMeasureSpec &#61; MeasureSpec.makeMeasureSpec(height,
                                MeasureSpec.EXACTLY);
                        measureAgain &#61; true;
                    }

                    if (measureAgain) {
                        if (DEBUG_LAYOUT) Log.v(mTag,
                                &#34;And hey let&#39;s measure once more: width&#61;&#34; &#43; width
                                &#43; &#34; height&#61;&#34; &#43; height);
                        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                    }

                    layoutRequested &#61; true;
                }
            }
        } else {
            // Not the first pass and no window/insets/visibility change but the window
            // may have moved and we need check that and if so to update the left and right
            // in the attach info. We translate only the window frame since on window move
            // the window manager tells us only for the new frame but the insets are the
            // same and we do not want to translate them more than once.
            maybeHandleWindowMove(frame);
        }

        final boolean didLayout &#61; layoutRequested &amp;&amp; (!mStopped || mReportNextDraw);
        boolean triggerGlobalLayoutListener &#61; didLayout
                || mAttachInfo.mRecomputeGlobalAttributes;
        if (didLayout) {
            performLayout(lp, mWidth, mHeight);

            // By this point all views have been sized and positioned
            // We can compute the transparent area

            if ((host.mPrivateFlags &amp; View.PFLAG_REQUEST_TRANSPARENT_REGIONS) !&#61; 0) {
                // start out transparent
                // TODO: AVOID THAT CALL BY CACHING THE RESULT?
                host.getLocationInWindow(mTmpLocation);
                mTransparentRegion.set(mTmpLocation[0], mTmpLocation[1],
                        mTmpLocation[0] &#43; host.mRight - host.mLeft,
                        mTmpLocation[1] &#43; host.mBottom - host.mTop);

                host.gatherTransparentRegion(mTransparentRegion);
                if (mTranslator !&#61; null) {
                    mTranslator.translateRegionInWindowToScreen(mTransparentRegion);
                }

                if (!mTransparentRegion.equals(mPreviousTransparentRegion)) {
                    mPreviousTransparentRegion.set(mTransparentRegion);
                    mFullRedrawNeeded &#61; true;
                    // reconfigure window manager
                    try {
                        mWindowSession.setTransparentRegion(mWindow, mTransparentRegion);
                    } catch (RemoteException e) {
                    }
                }
            }

            if (DBG) {
                System.out.println(&#34;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#34;);
                System.out.println(&#34;performTraversals -- after setFrame&#34;);
                host.debug();
            }
        }

        if (triggerGlobalLayoutListener) {
            mAttachInfo.mRecomputeGlobalAttributes &#61; false;
            mAttachInfo.mTreeObserver.dispatchOnGlobalLayout();
        }

        if (computesInternalInsets) {
            // Clear the original insets.
            final ViewTreeObserver.InternalInsetsInfo insets &#61; mAttachInfo.mGivenInternalInsets;
            insets.reset();

            // Compute new insets in place.
            mAttachInfo.mTreeObserver.dispatchOnComputeInternalInsets(insets);
            mAttachInfo.mHasNonEmptyGivenInternalInsets &#61; !insets.isEmpty();

            // Tell the window manager.
            if (insetsPending || !mLastGivenInsets.equals(insets)) {
                mLastGivenInsets.set(insets);

                // Translate insets to screen coordinates if needed.
                final Rect contentInsets;
                final Rect visibleInsets;
                final Region touchableRegion;
                if (mTranslator !&#61; null) {
                    contentInsets &#61; mTranslator.getTranslatedContentInsets(insets.contentInsets);
                    visibleInsets &#61; mTranslator.getTranslatedVisibleInsets(insets.visibleInsets);
                    touchableRegion &#61; mTranslator.getTranslatedTouchableArea(insets.touchableRegion);
                } else {
                    contentInsets &#61; insets.contentInsets;
                    visibleInsets &#61; insets.visibleInsets;
                    touchableRegion &#61; insets.touchableRegion;
                }

                try {
                    mWindowSession.setInsets(mWindow, insets.mTouchableInsets,
                            contentInsets, visibleInsets, touchableRegion);
                } catch (RemoteException e) {
                }
            }
        }

        if (mFirst) {
            if (sAlwaysAssignFocus || !isInTouchMode()) {
                // handle first focus request
                if (DEBUG_INPUT_RESIZE) {
                    Log.v(mTag, &#34;First: mView.hasFocus()&#61;&#34; &#43; mView.hasFocus());
                }
                if (mView !&#61; null) {
                    if (!mView.hasFocus()) {
                        mView.restoreDefaultFocus();
                        if (DEBUG_INPUT_RESIZE) {
                            Log.v(mTag, &#34;First: requested focused view&#61;&#34; &#43; mView.findFocus());
                        }
                    } else {
                        if (DEBUG_INPUT_RESIZE) {
                            Log.v(mTag, &#34;First: existing focused view&#61;&#34; &#43; mView.findFocus());
                        }
                    }
                }
            } else {
                // Some views (like ScrollView) won&#39;t hand focus to descendants that aren&#39;t within
                // their viewport. Before layout, there&#39;s a good change these views are size 0
                // which means no children can get focus. After layout, this view now has size, but
                // is not guaranteed to hand-off focus to a focusable child (specifically, the edge-
                // case where the child has a size prior to layout and thus won&#39;t trigger
                // focusableViewAvailable).
                View focused &#61; mView.findFocus();
                if (focused instanceof ViewGroup
                        &amp;&amp; ((ViewGroup) focused).getDescendantFocusability()
                                &#61;&#61; ViewGroup.FOCUS_AFTER_DESCENDANTS) {
                    focused.restoreDefaultFocus();
                }
            }
        }

        final boolean changedVisibility &#61; (viewVisibilityChanged || mFirst) &amp;&amp; isViewVisible;
        final boolean hasWindowFocus &#61; mAttachInfo.mHasWindowFocus &amp;&amp; isViewVisible;
        final boolean regainedFocus &#61; hasWindowFocus &amp;&amp; mLostWindowFocus;
        if (regainedFocus) {
            mLostWindowFocus &#61; false;
        } else if (!hasWindowFocus &amp;&amp; mHadWindowFocus) {
            mLostWindowFocus &#61; true;
        }

        if (changedVisibility || regainedFocus) {
            // Toasts are presented as notifications - don&#39;t present them as windows as well
            boolean isToast &#61; (mWindowAttributes &#61;&#61; null) ? false
                    : (mWindowAttributes.type &#61;&#61; WindowManager.LayoutParams.TYPE_TOAST);
            if (!isToast) {
                host.sendAccessibilityEvent(AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED);
            }
        }

        mFirst &#61; false;
        mWillDrawSoon &#61; false;
        mNewSurfaceNeeded &#61; false;
        mActivityRelaunched &#61; false;
        mViewVisibility &#61; viewVisibility;
        mHadWindowFocus &#61; hasWindowFocus;

        if (hasWindowFocus &amp;&amp; !isInLocalFocusMode()) {
            final boolean imTarget &#61; WindowManager.LayoutParams
                    .mayUseInputMethod(mWindowAttributes.flags);
            if (imTarget !&#61; mLastWasImTarget) {
                mLastWasImTarget &#61; imTarget;
                InputMethodManager imm &#61; InputMethodManager.peekInstance();
                if (imm !&#61; null &amp;&amp; imTarget) {
                    imm.onPreWindowFocus(mView, hasWindowFocus);
                    imm.onPostWindowFocus(mView, mView.findFocus(),
                            mWindowAttributes.softInputMode,
                            !mHasHadWindowFocus, mWindowAttributes.flags);
                }
            }
        }

        // Remember if we must report the next draw.
        if ((relayoutResult &amp; WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) !&#61; 0) {
            reportNextDraw();
        }

        boolean cancelDraw &#61; mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;

        if (!cancelDraw &amp;&amp; !newSurface) {
            if (mPendingTransitions !&#61; null &amp;&amp; mPendingTransitions.size() &gt; 0) {
                for (int i &#61; 0; i &lt; mPendingTransitions.size(); &#43;&#43;i) {
                    mPendingTransitions.get(i).startChangingAnimations();
                }
                mPendingTransitions.clear();
            }

            performDraw();
        } else {
            if (isViewVisible) {
                // Try again
                scheduleTraversals();
            } else if (mPendingTransitions !&#61; null &amp;&amp; mPendingTransitions.size() &gt; 0) {
                for (int i &#61; 0; i &lt; mPendingTransitions.size(); &#43;&#43;i) {
                    mPendingTransitions.get(i).endChangingAnimations();
                }
                mPendingTransitions.clear();
            }
        }

        mIsInTraversal &#61; false;
    }
</code></pre> 
<p>这里主要是判断是否需要执行performMeasure() &#xff0c;performLayout()&#xff0c;performDraw()这三个过程都会遍历View树刷新需要更新的view。</p> 
<h3><a id="210_SessionaddToDisplay_2270"></a>2.10 Session.addToDisplay</h3> 
<p>[-&gt;Session.java]</p> 
<pre><code>  &#64;Override
  public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
            int viewVisibility, int displayId, Rect outFrame, Rect outContentInsets,
            Rect outStableInsets, Rect outOutsets,
            DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel) {
        return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId, outFrame,
                outContentInsets, outStableInsets, outOutsets, outDisplayCutout, outInputChannel);
   }
</code></pre> 
<h3><a id="211_WMSaddWindow_2285"></a>2.11 WMS.addWindow</h3> 
<p>[-&gt;WindowManagerService.java]</p> 
<pre><code>  public int addWindow(Session session, IWindow client, int seq,
            LayoutParams attrs, int viewVisibility, int displayId, Rect outFrame,
            Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
            DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel) {
        int[] appOp &#61; new int[1];
        //此处mPolicy为PhoneWindowManager
        int res &#61; mPolicy.checkAddPermission(attrs, appOp);
        if (res !&#61; WindowManagerGlobal.ADD_OKAY) {
            return res;
        }

        boolean reportNewConfig &#61; false;
        WindowState parentWindow &#61; null;
        long origId;
        final int callingUid &#61; Binder.getCallingUid();
        final int type &#61; attrs.type;

        synchronized(mWindowMap) {
            if (!mDisplayReady) {
                throw new IllegalStateException(&#34;Display has not been initialialized&#34;);
            }

            final DisplayContent displayContent &#61; getDisplayContentOrCreate(displayId);

            if (displayContent &#61;&#61; null) {
                Slog.w(TAG_WM, &#34;Attempted to add window to a display that does not exist: &#34;
                        &#43; displayId &#43; &#34;.  Aborting.&#34;);
                return WindowManagerGlobal.ADD_INVALID_DISPLAY;
            }
            if (!displayContent.hasAccess(session.mUid)
                    &amp;&amp; !mDisplayManagerInternal.isUidPresentOnDisplay(session.mUid, displayId)) {
                Slog.w(TAG_WM, &#34;Attempted to add window to a display for which the application &#34;
                        &#43; &#34;does not have access: &#34; &#43; displayId &#43; &#34;.  Aborting.&#34;);
                return WindowManagerGlobal.ADD_INVALID_DISPLAY;
            }

            if (mWindowMap.containsKey(client.asBinder())) {
                Slog.w(TAG_WM, &#34;Window &#34; &#43; client &#43; &#34; is already added&#34;);
                return WindowManagerGlobal.ADD_DUPLICATE_ADD;
            }

            if (type &gt;&#61; FIRST_SUB_WINDOW &amp;&amp; type &lt;&#61; LAST_SUB_WINDOW) {
                parentWindow &#61; windowForClientLocked(null, attrs.token, false);
                if (parentWindow &#61;&#61; null) {
                    Slog.w(TAG_WM, &#34;Attempted to add window with token that is not a window: &#34;
                          &#43; attrs.token &#43; &#34;.  Aborting.&#34;);
                    return WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN;
                }
                if (parentWindow.mAttrs.type &gt;&#61; FIRST_SUB_WINDOW
                        &amp;&amp; parentWindow.mAttrs.type &lt;&#61; LAST_SUB_WINDOW) {
                    Slog.w(TAG_WM, &#34;Attempted to add window with token that is a sub-window: &#34;
                            &#43; attrs.token &#43; &#34;.  Aborting.&#34;);
                    return WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN;
                }
            }

            if (type &#61;&#61; TYPE_PRIVATE_PRESENTATION &amp;&amp; !displayContent.isPrivate()) {
                Slog.w(TAG_WM, &#34;Attempted to add private presentation window to a non-private display.  Aborting.&#34;);
                return WindowManagerGlobal.ADD_PERMISSION_DENIED;
            }

            AppWindowToken atoken &#61; null;
            final boolean hasParent &#61; parentWindow !&#61; null;
            // Use existing parent window token for child windows since they go in the same token
            // as there parent window so we can apply the same policy on them.
            WindowToken token &#61; displayContent.getWindowToken(
                    hasParent ? parentWindow.mAttrs.token : attrs.token);
            // If this is a child window, we want to apply the same type checking rules as the
            // parent window type.
            final int rootType &#61; hasParent ? parentWindow.mAttrs.type : type;

            boolean addToastWindowRequiresToken &#61; false;
            //判读token
            if (token &#61;&#61; null) {
                if (rootType &gt;&#61; FIRST_APPLICATION_WINDOW &amp;&amp; rootType &lt;&#61; LAST_APPLICATION_WINDOW) {
                    Slog.w(TAG_WM, &#34;Attempted to add application window with unknown token &#34;
                          &#43; attrs.token &#43; &#34;.  Aborting.&#34;);
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (rootType &#61;&#61; TYPE_INPUT_METHOD) {
                    Slog.w(TAG_WM, &#34;Attempted to add input method window with unknown token &#34;
                          &#43; attrs.token &#43; &#34;.  Aborting.&#34;);
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (rootType &#61;&#61; TYPE_VOICE_INTERACTION) {
                    Slog.w(TAG_WM, &#34;Attempted to add voice interaction window with unknown token &#34;
                          &#43; attrs.token &#43; &#34;.  Aborting.&#34;);
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (rootType &#61;&#61; TYPE_WALLPAPER) {
                    Slog.w(TAG_WM, &#34;Attempted to add wallpaper window with unknown token &#34;
                          &#43; attrs.token &#43; &#34;.  Aborting.&#34;);
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (rootType &#61;&#61; TYPE_DREAM) {
                    Slog.w(TAG_WM, &#34;Attempted to add Dream window with unknown token &#34;
                          &#43; attrs.token &#43; &#34;.  Aborting.&#34;);
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (rootType &#61;&#61; TYPE_QS_DIALOG) {
                    Slog.w(TAG_WM, &#34;Attempted to add QS dialog window with unknown token &#34;
                          &#43; attrs.token &#43; &#34;.  Aborting.&#34;);
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (rootType &#61;&#61; TYPE_ACCESSIBILITY_OVERLAY) {
                    Slog.w(TAG_WM, &#34;Attempted to add Accessibility overlay window with unknown token &#34;
                            &#43; attrs.token &#43; &#34;.  Aborting.&#34;);
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (type &#61;&#61; TYPE_TOAST) {
                    // Apps targeting SDK above N MR1 cannot arbitrary add toast windows.
                    if (doesAddToastWindowRequireToken(attrs.packageName, callingUid,
                            parentWindow)) {
                        Slog.w(TAG_WM, &#34;Attempted to add a toast window with unknown token &#34;
                                &#43; attrs.token &#43; &#34;.  Aborting.&#34;);
                        return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                    }
                }
                final IBinder binder &#61; attrs.token !&#61; null ? attrs.token : client.asBinder();
                final boolean isRoundedCornerOverlay &#61;
                        (attrs.privateFlags &amp; PRIVATE_FLAG_IS_ROUNDED_CORNERS_OVERLAY) !&#61; 0;
                token &#61; new WindowToken(this, binder, type, false, displayContent,
                        session.mCanAddInternalSystemWindow, isRoundedCornerOverlay);
            } else if (rootType &gt;&#61; FIRST_APPLICATION_WINDOW &amp;&amp; rootType &lt;&#61; LAST_APPLICATION_WINDOW) {
                atoken &#61; token.asAppWindowToken();
                if (atoken &#61;&#61; null) {
                    Slog.w(TAG_WM, &#34;Attempted to add window with non-application token &#34;
                          &#43; token &#43; &#34;.  Aborting.&#34;);
                    return WindowManagerGlobal.ADD_NOT_APP_TOKEN;
                } else if (atoken.removed) {
                    Slog.w(TAG_WM, &#34;Attempted to add window with exiting application token &#34;
                          &#43; token &#43; &#34;.  Aborting.&#34;);
                    return WindowManagerGlobal.ADD_APP_EXITING;
                } else if (type &#61;&#61; TYPE_APPLICATION_STARTING &amp;&amp; atoken.startingWindow !&#61; null) {
                    Slog.w(TAG_WM, &#34;Attempted to add starting window to token with already existing&#34;
                            &#43; &#34; starting window&#34;);
                    return WindowManagerGlobal.ADD_DUPLICATE_ADD;
                }
            } else if (rootType &#61;&#61; TYPE_INPUT_METHOD) {
                if (token.windowType !&#61; TYPE_INPUT_METHOD) {
                    Slog.w(TAG_WM, &#34;Attempted to add input method window with bad token &#34;
                            &#43; attrs.token &#43; &#34;.  Aborting.&#34;);
                      return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (rootType &#61;&#61; TYPE_VOICE_INTERACTION) {
                if (token.windowType !&#61; TYPE_VOICE_INTERACTION) {
                    Slog.w(TAG_WM, &#34;Attempted to add voice interaction window with bad token &#34;
                            &#43; attrs.token &#43; &#34;.  Aborting.&#34;);
                      return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (rootType &#61;&#61; TYPE_WALLPAPER) {
                if (token.windowType !&#61; TYPE_WALLPAPER) {
                    Slog.w(TAG_WM, &#34;Attempted to add wallpaper window with bad token &#34;
                            &#43; attrs.token &#43; &#34;.  Aborting.&#34;);
                      return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (rootType &#61;&#61; TYPE_DREAM) {
                if (token.windowType !&#61; TYPE_DREAM) {
                    Slog.w(TAG_WM, &#34;Attempted to add Dream window with bad token &#34;
                            &#43; attrs.token &#43; &#34;.  Aborting.&#34;);
                      return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (rootType &#61;&#61; TYPE_ACCESSIBILITY_OVERLAY) {
                if (token.windowType !&#61; TYPE_ACCESSIBILITY_OVERLAY) {
                    Slog.w(TAG_WM, &#34;Attempted to add Accessibility overlay window with bad token &#34;
                            &#43; attrs.token &#43; &#34;.  Aborting.&#34;);
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (type &#61;&#61; TYPE_TOAST) {
                // Apps targeting SDK above N MR1 cannot arbitrary add toast windows.
                addToastWindowRequiresToken &#61; doesAddToastWindowRequireToken(attrs.packageName,
                        callingUid, parentWindow);
                if (addToastWindowRequiresToken &amp;&amp; token.windowType !&#61; TYPE_TOAST) {
                    Slog.w(TAG_WM, &#34;Attempted to add a toast window with bad token &#34;
                            &#43; attrs.token &#43; &#34;.  Aborting.&#34;);
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (type &#61;&#61; TYPE_QS_DIALOG) {
                if (token.windowType !&#61; TYPE_QS_DIALOG) {
                    Slog.w(TAG_WM, &#34;Attempted to add QS dialog window with bad token &#34;
                            &#43; attrs.token &#43; &#34;.  Aborting.&#34;);
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (token.asAppWindowToken() !&#61; null) {
                Slog.w(TAG_WM, &#34;Non-null appWindowToken for system window of rootType&#61;&#34; &#43; rootType);
                // It is not valid to use an app token with other system types; we will
                // instead make a new token for it (as if null had been passed in for the token).
                attrs.token &#61; null;
                token &#61; new WindowToken(this, client.asBinder(), type, false, displayContent,
                        session.mCanAddInternalSystemWindow);
            }
            //创建WindowState
            final WindowState win &#61; new WindowState(this, session, client, token, parentWindow,
                    appOp[0], seq, attrs, viewVisibility, session.mUid,
                    session.mCanAddInternalSystemWindow);
            if (win.mDeathRecipient &#61;&#61; null) {
                // Client has apparently died, so there is no reason to
                // continue.
                Slog.w(TAG_WM, &#34;Adding window client &#34; &#43; client.asBinder()
                        &#43; &#34; that is dead, aborting.&#34;);
                return WindowManagerGlobal.ADD_APP_EXITING;
            }

            if (win.getDisplayContent() &#61;&#61; null) {
                Slog.w(TAG_WM, &#34;Adding window to Display that has been removed.&#34;);
                return WindowManagerGlobal.ADD_INVALID_DISPLAY;
            }

            final boolean hasStatusBarServicePermission &#61;
                    mContext.checkCallingOrSelfPermission(permission.STATUS_BAR_SERVICE)
                            &#61;&#61; PackageManager.PERMISSION_GRANTED;
            //调整WindowManager的LayoutParams参数                
            mPolicy.adjustWindowParamsLw(win, win.mAttrs, hasStatusBarServicePermission);
            win.setShowToOwnerOnlyLocked(mPolicy.checkShowToOwnerOnly(attrs));

            res &#61; mPolicy.prepareAddWindowLw(win, attrs);
            if (res !&#61; WindowManagerGlobal.ADD_OKAY) {
                return res;
            }

            final boolean openInputChannels &#61; (outInputChannel !&#61; null
                    &amp;&amp; (attrs.inputFeatures &amp; INPUT_FEATURE_NO_INPUT_CHANNEL) &#61;&#61; 0);
            if  (openInputChannels) {
                //打开输入通道
                win.openInputChannel(outInputChannel);
            }

            // If adding a toast requires a token for this app we always schedule hiding
            // toast windows to make sure they don&#39;t stick around longer then necessary.
            // We hide instead of remove such windows as apps aren&#39;t prepared to handle
            // windows being removed under them.
            //
            // If the app is older it can add toasts without a token and hence overlay
            // other apps. To be maximally compatible with these apps we will hide the
            // window after the toast timeout only if the focused window is from another
            // UID, otherwise we allow unlimited duration. When a UID looses focus we
            // schedule hiding all of its toast windows.
            if (type &#61;&#61; TYPE_TOAST) {
                if (!getDefaultDisplayContentLocked().canAddToastWindowForUid(callingUid)) {
                    Slog.w(TAG_WM, &#34;Adding more than one toast window for UID at a time.&#34;);
                    return WindowManagerGlobal.ADD_DUPLICATE_ADD;
                }
                // Make sure this happens before we moved focus as one can make the
                // toast focusable to force it not being hidden after the timeout.
                // Focusable toasts are always timed out to prevent a focused app to
                // show a focusable toasts while it has focus which will be kept on
                // the screen after the activity goes away.
                if (addToastWindowRequiresToken
                        || (attrs.flags &amp; LayoutParams.FLAG_NOT_FOCUSABLE) &#61;&#61; 0
                        || mCurrentFocus &#61;&#61; null
                        || mCurrentFocus.mOwnerUid !&#61; callingUid) {
                    mH.sendMessageDelayed(
                            mH.obtainMessage(H.WINDOW_HIDE_TIMEOUT, win),
                            win.mAttrs.hideTimeoutMilliseconds);
                }
            }

            // From now on, no exceptions or errors allowed!

            res &#61; WindowManagerGlobal.ADD_OKAY;
            if (mCurrentFocus &#61;&#61; null) {
                mWinAddedSinceNullFocus.add(win);
            }

            if (excludeWindowTypeFromTapOutTask(type)) {
                displayContent.mTapExcludedWindows.add(win);
            }

            origId &#61; Binder.clearCallingIdentity();
            //见2.11.2节
            win.attach();
            mWindowMap.put(client.asBinder(), win);

            win.initAppOpsState();

            final boolean suspended &#61; mPmInternal.isPackageSuspended(win.getOwningPackage(),
                    UserHandle.getUserId(win.getOwningUid()));
            win.setHiddenWhileSuspended(suspended);

            final boolean hideSystemAlertWindows &#61; !mHidingNonSystemOverlayWindows.isEmpty();
            win.setForceHideNonSystemOverlayWindowIfNeeded(hideSystemAlertWindows);

            final AppWindowToken aToken &#61; token.asAppWindowToken();
            if (type &#61;&#61; TYPE_APPLICATION_STARTING &amp;&amp; aToken !&#61; null) {
                aToken.startingWindow &#61; win;
                if (DEBUG_STARTING_WINDOW) Slog.v (TAG_WM, &#34;addWindow: &#34; &#43; aToken
                        &#43; &#34; startingWindow&#61;&#34; &#43; win);
            }

            boolean imMayMove &#61; true;

            win.mToken.addWindow(win);
            if (type &#61;&#61; TYPE_INPUT_METHOD) {
                win.mGivenInsetsPending &#61; true;
                setInputMethodWindowLocked(win);
                imMayMove &#61; false;
            } else if (type &#61;&#61; TYPE_INPUT_METHOD_DIALOG) {
                displayContent.computeImeTarget(true /* updateImeTarget */);
                imMayMove &#61; false;
            } else {
                if (type &#61;&#61; TYPE_WALLPAPER) {
                    displayContent.mWallpaperController.clearLastWallpaperTimeoutTime();
                    displayContent.pendingLayoutChanges |&#61; FINISH_LAYOUT_REDO_WALLPAPER;
                } else if ((attrs.flags&amp;FLAG_SHOW_WALLPAPER) !&#61; 0) {
                    displayContent.pendingLayoutChanges |&#61; FINISH_LAYOUT_REDO_WALLPAPER;
                } else if (displayContent.mWallpaperController.isBelowWallpaperTarget(win)) {
                    // If there is currently a wallpaper being shown, and
                    // the base layer of the new window is below the current
                    // layer of the target window, then adjust the wallpaper.
                    // This is to avoid a new window being placed between the
                    // wallpaper and its target.
                    displayContent.pendingLayoutChanges |&#61; FINISH_LAYOUT_REDO_WALLPAPER;
                }
            }

            // If the window is being added to a stack that&#39;s currently adjusted for IME,
            // make sure to apply the same adjust to this new window.
            win.applyAdjustForImeIfNeeded();

            if (type &#61;&#61; TYPE_DOCK_DIVIDER) {
                mRoot.getDisplayContent(displayId).getDockedDividerController().setWindow(win);
            }

            final WindowStateAnimator winAnimator &#61; win.mWinAnimator;
            winAnimator.mEnterAnimationPending &#61; true;
            winAnimator.mEnteringAnimation &#61; true;
            // Check if we need to prepare a transition for replacing window first.
            if (atoken !&#61; null &amp;&amp; atoken.isVisible()
                    &amp;&amp; !prepareWindowReplacementTransition(atoken)) {
                // If not, check if need to set up a dummy transition during display freeze
                // so that the unfreeze wait for the apps to draw. This might be needed if
                // the app is relaunching.
                prepareNoneTransitionForRelaunching(atoken);
            }

            final DisplayFrames displayFrames &#61; displayContent.mDisplayFrames;
            // TODO: Not sure if onDisplayInfoUpdated() call is needed.
            final DisplayInfo displayInfo &#61; displayContent.getDisplayInfo();
            displayFrames.onDisplayInfoUpdated(displayInfo,
                    displayContent.calculateDisplayCutoutForRotation(displayInfo.rotation));
            final Rect taskBounds;
            if (atoken !&#61; null &amp;&amp; atoken.getTask() !&#61; null) {
                taskBounds &#61; mTmpRect;
                atoken.getTask().getBounds(mTmpRect);
            } else {
                taskBounds &#61; null;
            }
            if (mPolicy.getLayoutHintLw(win.mAttrs, taskBounds, displayFrames, outFrame,
                    outContentInsets, outStableInsets, outOutsets, outDisplayCutout)) {
                res |&#61; WindowManagerGlobal.ADD_FLAG_ALWAYS_CONSUME_NAV_BAR;
            }

            if (mInTouchMode) {
                res |&#61; WindowManagerGlobal.ADD_FLAG_IN_TOUCH_MODE;
            }
            if (win.mAppToken &#61;&#61; null || !win.mAppToken.isClientHidden()) {
                res |&#61; WindowManagerGlobal.ADD_FLAG_APP_VISIBLE;
            }

            mInputMonitor.setUpdateInputWindowsNeededLw();

            boolean focusChanged &#61; false;
            if (win.canReceiveKeys()) {
               //当该窗口可以接收按键事件&#xff0c;则更新窗口
                focusChanged &#61; updateFocusedWindowLocked(UPDATE_FOCUS_WILL_ASSIGN_LAYERS,
                        false /*updateInputWindows*/);
                if (focusChanged) {
                    imMayMove &#61; false;
                }
            }

            if (imMayMove) {
                displayContent.computeImeTarget(true /* updateImeTarget */);
            }

            // Don&#39;t do layout here, the window must call
            // relayout to be displayed, so we&#39;ll do it there.
            win.getParent().assignChildLayers();

            if (focusChanged) {
                mInputMonitor.setInputFocusLw(mCurrentFocus, false /*updateInputWindows*/);
            }
            //更新输入窗口
            mInputMonitor.updateInputWindowsLw(false /*force*/);

            if (localLOGV || DEBUG_ADD_REMOVE) Slog.v(TAG_WM, &#34;addWindow: New client &#34;
                    &#43; client.asBinder() &#43; &#34;: window&#61;&#34; &#43; win &#43; &#34; Callers&#61;&#34; &#43; Debug.getCallers(5));

            if (win.isVisibleOrAdding() &amp;&amp; updateOrientationFromAppTokensLocked(displayId)) {
                reportNewConfig &#61; true;
            }
        }

        if (reportNewConfig) {
            sendNewConfiguration(displayId);
        }

        Binder.restoreCallingIdentity(origId);

        return res;
    }
</code></pre> 
<h4><a id="2111_WindowState_2693"></a>2.11.1 WindowState</h4> 
<p>[-&gt;WindowState.java]</p> 
<pre><code> WindowState(WindowManagerService service, Session s, IWindow c, WindowToken token,
            WindowState parentWindow, int appOp, int seq, WindowManager.LayoutParams a,
            int viewVisibility, int ownerId, boolean ownerCanAddInternalSystemWindow,
            PowerManagerWrapper powerManagerWrapper) {
        super(service);
        //Session的Binder服务端
        mSession &#61; s; 
        //IWindow的代理端
        mClient &#61; c;
        mAppOp &#61; appOp;
        mToken &#61; token;
        mAppToken &#61; mToken.asAppWindowToken();
        //对应app的uid
        mOwnerUid &#61; ownerId;
        mOwnerCanAddInternalSystemWindow &#61; ownerCanAddInternalSystemWindow;
        mWindowId &#61; new WindowId(this);
        mAttrs.copyFrom(a);
        mLastSurfaceInsets.set(mAttrs.surfaceInsets);
        mViewVisibility &#61; viewVisibility;
        mPolicy &#61; mService.mPolicy;
        mContext &#61; mService.mContext;
        DeathRecipient deathRecipient &#61; new DeathRecipient();
        mSeq &#61; seq;
        mEnforceSizeCompat &#61; (mAttrs.privateFlags &amp; PRIVATE_FLAG_COMPATIBLE_WINDOW) !&#61; 0;
        mPowerManagerWrapper &#61; powerManagerWrapper;
        mForceSeamlesslyRotate &#61; token.mRoundedCornerOverlay;
        if (localLOGV) Slog.v(
            TAG, &#34;Window &#34; &#43; this &#43; &#34; client&#61;&#34; &#43; c.asBinder()
            &#43; &#34; token&#61;&#34; &#43; token &#43; &#34; (&#34; &#43; mAttrs.token &#43; &#34;)&#34; &#43; &#34; params&#61;&#34; &#43; a);
        try {
            c.asBinder().linkToDeath(deathRecipient, 0);
        } catch (RemoteException e) {
            mDeathRecipient &#61; null;
            mIsChildWindow &#61; false;
            mLayoutAttached &#61; false;
            mIsImWindow &#61; false;
            mIsWallpaper &#61; false;
            mIsFloatingLayer &#61; false;
            mBaseLayer &#61; 0;
            mSubLayer &#61; 0;
            mInputWindowHandle &#61; null;
            mWinAnimator &#61; null;
            return;
        }
        mDeathRecipient &#61; deathRecipient;

        if (mAttrs.type &gt;&#61; FIRST_SUB_WINDOW &amp;&amp; mAttrs.type &lt;&#61; LAST_SUB_WINDOW) {
            // The multiplier here is to reserve space for multiple
            // windows in the same type layer.
            mBaseLayer &#61; mPolicy.getWindowLayerLw(parentWindow)
                    * TYPE_LAYER_MULTIPLIER &#43; TYPE_LAYER_OFFSET;
            mSubLayer &#61; mPolicy.getSubWindowLayerFromTypeLw(a.type);
            mIsChildWindow &#61; true;

            if (DEBUG_ADD_REMOVE) Slog.v(TAG, &#34;Adding &#34; &#43; this &#43; &#34; to &#34; &#43; parentWindow);
            parentWindow.addChild(this, sWindowSubLayerComparator);

            mLayoutAttached &#61; mAttrs.type !&#61;
                    WindowManager.LayoutParams.TYPE_APPLICATION_ATTACHED_DIALOG;
            mIsImWindow &#61; parentWindow.mAttrs.type &#61;&#61; TYPE_INPUT_METHOD
                    || parentWindow.mAttrs.type &#61;&#61; TYPE_INPUT_METHOD_DIALOG;
            mIsWallpaper &#61; parentWindow.mAttrs.type &#61;&#61; TYPE_WALLPAPER;
        } else {
            // The multiplier here is to reserve space for multiple
            // windows in the same type layer.
            mBaseLayer &#61; mPolicy.getWindowLayerLw(this)
                    * TYPE_LAYER_MULTIPLIER &#43; TYPE_LAYER_OFFSET;
            mSubLayer &#61; 0;
            mIsChildWindow &#61; false;
            mLayoutAttached &#61; false;
            mIsImWindow &#61; mAttrs.type &#61;&#61; TYPE_INPUT_METHOD
                    || mAttrs.type &#61;&#61; TYPE_INPUT_METHOD_DIALOG;
            mIsWallpaper &#61; mAttrs.type &#61;&#61; TYPE_WALLPAPER;
        }
        mIsFloatingLayer &#61; mIsImWindow || mIsWallpaper;

        if (mAppToken !&#61; null &amp;&amp; mAppToken.mShowForAllUsers) {
            // Windows for apps that can show for all users should also show when the device is
            // locked.
            mAttrs.flags |&#61; FLAG_SHOW_WHEN_LOCKED;
        }

        mWinAnimator &#61; new WindowStateAnimator(this);
        mWinAnimator.mAlpha &#61; a.alpha;

        mRequestedWidth &#61; 0;
        mRequestedHeight &#61; 0;
        mLastRequestedWidth &#61; 0;
        mLastRequestedHeight &#61; 0;
        mLayer &#61; 0;
        mInputWindowHandle &#61; new InputWindowHandle(
                mAppToken !&#61; null ? mAppToken.mInputApplicationHandle : null, this, c,
                    getDisplayId());
    }
</code></pre> 
<h4><a id="2112_wsattach_2794"></a>2.11.2 ws.attach</h4> 
<p>[-&gt;WindowState.java]</p> 
<pre><code>  void attach() {
        if (localLOGV) Slog.v(TAG, &#34;Attaching &#34; &#43; this &#43; &#34; token&#61;&#34; &#43; mToken);
        mSession.windowAddedLocked(mAttrs.packageName);
    }
</code></pre> 
<p>[-&gt;Session.java]</p> 
<pre><code> void windowAddedLocked(String packageName) {
        mPackageName &#61; packageName;
        mRelayoutTag &#61; &#34;relayoutWindow: &#34; &#43; mPackageName;
        if (mSurfaceSession &#61;&#61; null) {
            if (WindowManagerService.localLOGV) Slog.v(
                TAG_WM, &#34;First window added to &#34; &#43; this &#43; &#34;, creating SurfaceSession&#34;);
            //创建SurfaceSession对象
            mSurfaceSession &#61; new SurfaceSession();
            if (SHOW_TRANSACTIONS) Slog.i(
                    TAG_WM, &#34;  NEW SURFACE SESSION &#34; &#43; mSurfaceSession);
            //将当前Session添加到mSessions
            mService.mSessions.add(this);
            if (mLastReportedAnimatorScale !&#61; mService.getCurrentAnimatorScale()) {
                mService.dispatchNewAnimatorScaleLocked(this);
            }
        }
        mNumWindow&#43;&#43;;
    }
</code></pre> 
<h4><a id="2113_SurfaceSession_2828"></a>2.11.3 创建SurfaceSession</h4> 
<p>[-&gt;SurfaceSession.java]</p> 
<pre><code>  public SurfaceSession() {
        mNativeClient &#61; nativeCreate();
   }
</code></pre> 
<p>[-&gt;core/jni/android_view_SurfaceSession.cpp]</p> 
<pre><code>static jlong nativeCreate(JNIEnv* env, jclass clazz) {
    SurfaceComposerClient* client &#61; new SurfaceComposerClient();
    client-&gt;incStrong((void*)nativeCreate);
    return reinterpret_cast&lt;jlong&gt;(client);
}
</code></pre> 
<p>创建SurfaceComposerClient&#xff0c;作为SurfaceFlinger通信的对象</p> 
<h3><a id="212_WMSupdateFocusedWindowLocked_2850"></a>2.12 WMS.updateFocusedWindowLocked</h3> 
<p>[-&gt;WindowManagerService.java]</p> 
<pre><code> boolean updateFocusedWindowLocked(int mode, boolean updateInputWindows) {
        WindowState newFocus &#61; mRoot.computeFocusedWindow();
        //当前焦点改变
        if (mCurrentFocus !&#61; newFocus) {
            Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, &#34;wmUpdateFocus&#34;);
            // This check makes sure that we don&#39;t already have the focus
            // change message pending.
            mH.removeMessages(H.REPORT_FOCUS_CHANGE);
            mH.sendEmptyMessage(H.REPORT_FOCUS_CHANGE);
            // TODO(multidisplay): Focused windows on default display only.
            final DisplayContent displayContent &#61; getDefaultDisplayContentLocked();
            boolean imWindowChanged &#61; false;
            if (mInputMethodWindow !&#61; null) {
                final WindowState prevTarget &#61; mInputMethodTarget;
                final WindowState newTarget &#61;
                        displayContent.computeImeTarget(true /* updateImeTarget*/);

                imWindowChanged &#61; prevTarget !&#61; newTarget;

                if (mode !&#61; UPDATE_FOCUS_WILL_ASSIGN_LAYERS
                        &amp;&amp; mode !&#61; UPDATE_FOCUS_WILL_PLACE_SURFACES) {
                    final int prevImeAnimLayer &#61; mInputMethodWindow.mWinAnimator.mAnimLayer;
                    displayContent.assignWindowLayers(false /* setLayoutNeeded */);
                    imWindowChanged |&#61;
                            prevImeAnimLayer !&#61; mInputMethodWindow.mWinAnimator.mAnimLayer;
                }
            }

            if (imWindowChanged) {
                mWindowsChanged &#61; true;
                displayContent.setLayoutNeeded();
                newFocus &#61; mRoot.computeFocusedWindow();
            }

            if (DEBUG_FOCUS_LIGHT || localLOGV) Slog.v(TAG_WM, &#34;Changing focus from &#34; &#43;
                    mCurrentFocus &#43; &#34; to &#34; &#43; newFocus &#43; &#34; Callers&#61;&#34; &#43; Debug.getCallers(4));
            final WindowState oldFocus &#61; mCurrentFocus;
            mCurrentFocus &#61; newFocus;
            mLosingFocus.remove(newFocus);

            if (mCurrentFocus !&#61; null) {
                mWinAddedSinceNullFocus.clear();
                mWinRemovedSinceNullFocus.clear();
            }

            int focusChanged &#61; mPolicy.focusChangedLw(oldFocus, newFocus);

            if (imWindowChanged &amp;&amp; oldFocus !&#61; mInputMethodWindow) {
                // Focus of the input method window changed. Perform layout if needed.
                // 如果焦点改变&#xff0c;则执行layout
                if (mode &#61;&#61; UPDATE_FOCUS_PLACING_SURFACES) {
                    displayContent.performLayout(true /*initial*/,  updateInputWindows);
                    focusChanged &amp;&#61; ~FINISH_LAYOUT_REDO_LAYOUT;
                } else if (mode &#61;&#61; UPDATE_FOCUS_WILL_PLACE_SURFACES) {
                    // Client will do the layout, but we need to assign layers
                    // for handleNewWindowLocked() below.
                    displayContent.assignWindowLayers(false /* setLayoutNeeded */);
                }
            }

            if ((focusChanged &amp; FINISH_LAYOUT_REDO_LAYOUT) !&#61; 0) {
                // The change in focus caused us to need to do a layout.  Okay.
                displayContent.setLayoutNeeded();
                if (mode &#61;&#61; UPDATE_FOCUS_PLACING_SURFACES) {
                    displayContent.performLayout(true /*initial*/, updateInputWindows);
                }
            }

            if (mode !&#61; UPDATE_FOCUS_WILL_ASSIGN_LAYERS) {
                // If we defer assigning layers, then the caller is responsible for
                // doing this part.
                mInputMonitor.setInputFocusLw(mCurrentFocus, updateInputWindows);
            }

            displayContent.adjustForImeIfNeeded();

            // We may need to schedule some toast windows to be removed. The toasts for an app that
            // does not have input focus are removed within a timeout to prevent apps to redress
            // other apps&#39; UI.
            displayContent.scheduleToastWindowsTimeoutIfNeededLocked(oldFocus, newFocus);

            Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
            return true;
        }
        return false;
    }
</code></pre> 
<h2><a id="_2943"></a>三、总结</h2> 
<p>本文详细分析了Android屏幕刷新机制中的View的绘制过程&#xff0c;这里只介绍到framework层。</p> 
<p>1.从Activity的启动流程开始分析&#xff0c;在performLaunchActivity方法中调用Activity.attach方法创建PhoneWindow&#xff1b;</p> 
<p>2.PhoneWindow中有一个DecorView实例&#xff0c;DecorView继承于FrameLayout&#xff0c;通过setContentView将Activity中的布局view添加DecorView中&#xff1b;</p> 
<p>3.调用handleResumeActivity&#xff0c;进入到Activity的onResume方法&#xff0c;后面再调用Activity.makeVisible方法&#xff1b;</p> 
<p>4.makeVisible方法调用了WindowManagerGlobal.addView方法&#xff0c;创建了ViewRootImpl对象&#xff1b;</p> 
<p>5.调用ViewRootImlp中的requestLayout&#xff0c;再调用到里面的scheduleTraversals进行绘制&#xff1b;</p> 
<p>6.scheduleTraversals方法&#xff0c;向主线程发送同步屏障消息&#xff0c;通过mChoreographer.postCallback将mTraversalRunnable放到mCallbackQueues队列里面&#xff0c;在同一个帧内&#xff0c;多次请求requestLayout&#xff0c;只会调用一次scheduleTraversals。</p> 
<p>7.如果当前线程是主线程则调用nativeScheduleVsync&#xff0c;如果不是则通过handler发送到主线程进行处理&#xff1b;</p> 
<p>8.native方法主要是向底层注册监听Vsync屏幕刷新信号&#xff0c;当下一个Vsync信号发出时&#xff0c;底层会回调onVsync方法</p> 
<p>9.onVsync方法&#xff0c;会向主线程发送Runnable异步消息&#xff0c;里面会执行doFrame方法&#xff1b;</p> 
<p>10.doFrame会执行mCallbackQueues队列里面的任务&#xff0c;取出来的任务会执行doTraversal方法</p> 
<p>11.doTraversal方法会先移除主线程中同步屏障&#xff0c;然后调用performTraversals&#xff0c;根据当前的状态判断是否需要执行performMeasure&#xff0c;performLayout&#xff0c;performDraw&#xff0c;这三个过程都会遍历View树刷新需要更新的view。</p> 
<p>.focusChangedLw(oldFocus, newFocus);</p> 
<pre><code>        if (imWindowChanged &amp;&amp; oldFocus !&#61; mInputMethodWindow) {
            // Focus of the input method window changed. Perform layout if needed.
            // 如果焦点改变&#xff0c;则执行layout
            if (mode &#61;&#61; UPDATE_FOCUS_PLACING_SURFACES) {
                displayContent.performLayout(true /*initial*/,  updateInputWindows);
                focusChanged &amp;&#61; ~FINISH_LAYOUT_REDO_LAYOUT;
            } else if (mode &#61;&#61; UPDATE_FOCUS_WILL_PLACE_SURFACES) {
                // Client will do the layout, but we need to assign layers
                // for handleNewWindowLocked() below.
                displayContent.assignWindowLayers(false /* setLayoutNeeded */);
            }
        }

        if ((focusChanged &amp; FINISH_LAYOUT_REDO_LAYOUT) !&#61; 0) {
            // The change in focus caused us to need to do a layout.  Okay.
            displayContent.setLayoutNeeded();
            if (mode &#61;&#61; UPDATE_FOCUS_PLACING_SURFACES) {
                displayContent.performLayout(true /*initial*/, updateInputWindows);
            }
        }

        if (mode !&#61; UPDATE_FOCUS_WILL_ASSIGN_LAYERS) {
            // If we defer assigning layers, then the caller is responsible for
            // doing this part.
            mInputMonitor.setInputFocusLw(mCurrentFocus, updateInputWindows);
        }

        displayContent.adjustForImeIfNeeded();

        // We may need to schedule some toast windows to be removed. The toasts for an app that
        // does not have input focus are removed within a timeout to prevent apps to redress
        // other apps&#39; UI.
        displayContent.scheduleToastWindowsTimeoutIfNeededLocked(oldFocus, newFocus);

        Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
        return true;
    }
    return false;
}
</code></pre> 
<pre><code>
## 三、总结

本文详细分析了Android屏幕刷新机制中的View的绘制过程&#xff0c;这里只介绍到framework层。

1.从Activity的启动流程开始分析&#xff0c;在performLaunchActivity方法中调用Activity.attach方法创建PhoneWindow&#xff1b;

2.PhoneWindow中有一个DecorView实例&#xff0c;DecorView继承于FrameLayout&#xff0c;通过setContentView将Activity中的布局view添加DecorView中&#xff1b;

3.调用handleResumeActivity&#xff0c;进入到Activity的onResume方法&#xff0c;后面再调用Activity.makeVisible方法&#xff1b;

4.makeVisible方法调用了WindowManagerGlobal.addView方法&#xff0c;创建了ViewRootImpl对象&#xff1b;

5.调用ViewRootImlp中的requestLayout&#xff0c;再调用到里面的scheduleTraversals进行绘制&#xff1b;

6.scheduleTraversals方法&#xff0c;向主线程发送同步屏障消息&#xff0c;通过mChoreographer.postCallback将mTraversalRunnable放到mCallbackQueues队列里面&#xff0c;在同一个帧内&#xff0c;多次请求requestLayout&#xff0c;只会调用一次scheduleTraversals。

7.如果当前线程是主线程则调用nativeScheduleVsync&#xff0c;如果不是则通过handler发送到主线程进行处理&#xff1b;

8.native方法主要是向底层注册监听Vsync屏幕刷新信号&#xff0c;当下一个Vsync信号发出时&#xff0c;底层会回调onVsync方法

9.onVsync方法&#xff0c;会向主线程发送Runnable异步消息&#xff0c;里面会执行doFrame方法&#xff1b;

10.doFrame会执行mCallbackQueues队列里面的任务&#xff0c;取出来的任务会执行doTraversal方法

11.doTraversal方法会先移除主线程中同步屏障&#xff0c;然后调用performTraversals&#xff0c;根据当前的状态判断是否需要执行performMeasure&#xff0c;performLayout&#xff0c;performDraw&#xff0c;这三个过程都会遍历View树刷新需要更新的view。

12.通过Session.addToDisplay向WMS添加Window&#xff0c;WMS建立SurfaceComposerClient&#xff0c;然后会在SurfaceFlinger创建Client与之对应&#xff0c;后面通过ISurfaceComposerClient和SurfaceFlinger进行通信。
</code></pre>