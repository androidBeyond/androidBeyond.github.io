---
layout:     post
title:      Android基础探究之适配一切的adapter
subtitle:   从源码角度学习一下adapter的适配过程
date:       2019-04-21
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android
    - 组件学习
---


 <p>最近用到了Adapter,以前只是知道怎么用&#xff0c;从没去研究他的原理&#xff0c;这次就想以baseadapter为例研究一下其原理&#xff0c;从设计模式角度看Android在adapter这块用到了典型的适配器模式和观察者模式&#xff0c;那就从这个点开始&#xff0c;看看他是怎样的一个适配器。一般我们会这样设置一个ListView的适配器</p> 
<p></p> 
<pre><code class="language-html"> list.setAdapter(adapter);</code></pre>从这里开始&#xff0c;就开始adapter的神奇探索之旅 
<p></p> 
<p></p> 
<pre><code class="language-java">  public void setAdapter(ListAdapter adapter) {
        if (mAdapter !&#61; null &amp;&amp; mDataSetObserver !&#61; null) {
            mAdapter.unregisterDataSetObserver(mDataSetObserver); //注销现有的观察者
        }

        resetList();
        mRecycler.clear();

        if (mHeaderViewInfos.size() &gt; 0|| mFooterViewInfos.size() &gt; 0) {
            mAdapter &#61; new HeaderViewListAdapter(mHeaderViewInfos, mFooterViewInfos, adapter);
        } else {
            mAdapter &#61; adapter;
        }

        mOldSelectedPosition &#61; INVALID_POSITION;
        mOldSelectedRowId &#61; INVALID_ROW_ID;

        // AbsListView#setAdapter will update choice mode states.
        super.setAdapter(adapter);

        if (mAdapter !&#61; null) {
            mAreAllItemsSelectable &#61; mAdapter.areAllItemsEnabled();
            mOldItemCount &#61; mItemCount;
            mItemCount &#61; mAdapter.getCount();
            checkFocus();

            mDataSetObserver &#61; new AdapterDataSetObserver();  //创建新的观察者
            mAdapter.registerDataSetObserver(mDataSetObserver); //注册观察者

            mRecycler.setViewTypeCount(mAdapter.getViewTypeCount());

            int position;
            if (mStackFromBottom) {
                position &#61; lookForSelectablePosition(mItemCount - 1, false);
            } else {
                position &#61; lookForSelectablePosition(0, true);
            }
            setSelectedPositionInt(position);
            setNextSelectedPositionInt(position);

            if (mItemCount &#61;&#61; 0) {
                // Nothing selected
                checkSelectionChanged();
            }
        } else {
            mAreAllItemsSelectable &#61; true;
            checkFocus();
            // Nothing selected
            checkSelectionChanged();
        }

        requestLayout();  //请求布局
    }
</code></pre> 
<p></p> 
<p><br /> </p> 先看一下
<span style="font-family:宋体; font-size:9pt; background-color:rgb(228,228,255)">requestLayout</span>
<span style="font-family:宋体; font-size:9pt">();</span> 
<p><span style="font-family:宋体"><br /> </span></p> 
<p><span style="font-family:宋体"></span></p> 
<pre><code class="language-java">   &#64;Override
    public void requestLayout() {
        if (!mBlockLayoutRequests &amp;&amp; !mInLayout) {
            super.requestLayout();
        }
    }</code></pre>
<br /> 调用了超类的
<span style="font-family:宋体; background-color:rgb(240,240,240)">requestLayout()</span> 
<p></p> 
<p><span style="font-family:宋体"></span></p> 
<pre><code class="language-java"> public void requestLayout() {
        if (mMeasureCache !&#61; null) mMeasureCache.clear();

        if (mAttachInfo !&#61; null &amp;&amp; mAttachInfo.mViewRequestingLayout &#61;&#61; null) {
            // Only trigger request-during-layout logic if this is the view requesting it,
            // not the views in its parent hierarchy
            ViewRootImpl viewRoot &#61; getViewRootImpl();
            if (viewRoot !&#61; null &amp;&amp; viewRoot.isInLayout()) {
                if (!viewRoot.requestLayoutDuringLayout(this)) {
                    return;
                }
            }
            mAttachInfo.mViewRequestingLayout &#61; this;
        }

        mPrivateFlags |&#61; PFLAG_FORCE_LAYOUT;
        mPrivateFlags |&#61; PFLAG_INVALIDATED;

        if (mParent !&#61; null &amp;&amp; !mParent.isLayoutRequested()) {
            mParent.requestLayout();
        }
        if (mAttachInfo !&#61; null &amp;&amp; mAttachInfo.mViewRequestingLayout &#61;&#61; this) {
            mAttachInfo.mViewRequestingLayout &#61; null;
        }
    }
</code></pre>这里又会调用  mParent.requestLayout();
<span style="font-family:宋体">mParent是什么呢&#xff0c;在View类里有这样一段代码</span> 
<p></p> 
<p><span style="font-family:宋体"></span></p> 
<pre><code class="language-java">    void assignParent(ViewParent parent) {
        if (mParent &#61;&#61; null) {
            mParent &#61; parent;
        } else if (parent &#61;&#61; null) {
            mParent &#61; null;
        } else {
            throw new RuntimeException(&#34;view &#34; &#43; this &#43; &#34; being added, but&#34;
                    &#43; &#34; it already has a parent&#34;);
        }
    }</code></pre>
<br /> 这个函数是ViewRootImpl调用setView(this)将自身做为参数。所以这个parent是一个
<span style="font-family:宋体">ViewRootImpl的实例&#xff0c;至于这个<span style="font-family:宋体">ViewRootImpl是怎么回事不在这次的研究范围&#xff0c;暂且不去考究这个<span style="font-family:宋体">ViewRootImpl是怎么来的</span>&#xff0c;看一下ViewRootImpl的requestLayout()函数。</span></span>
<br /> 
<br /> 
<pre><code class="language-java">    &#64;Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested &#61; true;
            scheduleTraversals();
        }
    }</code></pre>继续往下找  scheduleTraversals() 
<p></p> 
<p></p> 
<pre><code class="language-java">    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled &#61; true;
            mTraversalBarrier &#61; mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }</code></pre>这里面 mChoreographer.postCallback(&#xff09;只看名字就可以理解到这个调用将会发起一个callBack&#xff0c;暂且不看callBack的内容&#xff0c;继续看postCallBack()的实现
<br /> 
<br /> 
<br /> 
<pre><code class="language-java">   public void postCallback(int callbackType, Runnable action, Object token) {
        postCallbackDelayed(callbackType, action, token, 0);
    }</code></pre>
<br /> 
<pre><code class="language-java">    public void postCallbackDelayed(int callbackType,
            Runnable action, Object token, long delayMillis) {
        if (action &#61;&#61; null) {
            throw new IllegalArgumentException(&#34;action must not be null&#34;);
        }
        if (callbackType &lt; 0 || callbackType &gt; CALLBACK_LAST) {
            throw new IllegalArgumentException(&#34;callbackType is invalid&#34;);
        }

        postCallbackDelayedInternal(callbackType, action, token, delayMillis);
    }</code></pre>
<br /> 
<pre><code class="language-java">    private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
        if (DEBUG_FRAMES) {
            Log.d(TAG, &#34;PostCallback: type&#61;&#34; &#43; callbackType
                    &#43; &#34;, action&#61;&#34; &#43; action &#43; &#34;, token&#61;&#34; &#43; token
                    &#43; &#34;, delayMillis&#61;&#34; &#43; delayMillis);
        }

        synchronized (mLock) {
            final long now &#61; SystemClock.uptimeMillis();
            final long dueTime &#61; now &#43; delayMillis;
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

            if (dueTime &lt;&#61; now) {
                scheduleFrameLocked(now);
            } else {
                Message msg &#61; mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
                msg.arg1 &#61; callbackType;
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, dueTime);
            }
        }
    }
</code></pre> 
<p>跟到这里可以发现他吧callback放进message用handler里发送了出去&#xff0c;handler在之前已经学习过了&#xff0c;这样在handler里调用action的run()方法&#xff0c;在这之后再看看action的run()方法在做什么工作</p> 
<p></p> 
<pre><code class="language-java"> final class TraversalRunnable implements Runnable {
        &#64;Override
        public void run() {
            doTraversal();
        }
    }</code></pre> 
<p></p> 
<p><br /> </p> 
<pre><code class="language-java">    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled &#61; false;
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing(&#34;ViewAncestor&#34;);
            }

            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile &#61; false;
            }
        }
    }</code></pre>
<br /> 
<br /> 
<pre><code class="language-java"></code><pre name="code" class="java"><code class="language-java">    private void performTraversals() {
 
 
		略。。。。。。
       
            // Ask host how big it wants to be
            windowSizeMayChange |&#61; measureHierarchy(host, lp, res,
                    desiredWindowWidth, desiredWindowHeight);
        }

       略。。。。。

        if (!cancelDraw &amp;&amp; !newSurface) {
            if (!skipDraw || mReportNextDraw) {
                if (mPendingTransitions !&#61; null &amp;&amp; mPendingTransitions.size() &gt; 0) {
                    for (int i &#61; 0; i &lt; mPendingTransitions.size(); &#43;&#43;i) {
                        mPendingTransitions.get(i).startChangingAnimations();
                    }
                    mPendingTransitions.clear();
                }

                performDraw();
            }
        }
    }
</code></pre><br />
<br />
<pre></pre>
<p>在这里调用了<span style="background-color:rgb(240,240,240)"> </span></p><pre><code class="language-java">measureHierarchy() 和  <span style="font-family: Arial, Helvetica, sans-serif;">performDraw(); 依次看一下</span></code></pre><p></p>
<p></p>
<p></p><pre><code class="language-java"></code><pre name="code" class="java"><code class="language-java">    private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
            final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {

        if (lp.width &#61;&#61; ViewGroup.LayoutParams.WRAP_CONTENT) {

            if (baseSize !&#61; 0 &amp;&amp; desiredWindowWidth &gt; baseSize) {
                childWidthMeasureSpec &#61; getRootMeasureSpec(baseSize, lp.width);
                childHeightMeasureSpec &#61; getRootMeasureSpec(desiredWindowHeight, lp.height);
          <span style="color:#cc0000;"><strong>      performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);</strong></span>
                if ((host.getMeasuredWidthAndState()&amp;View.MEASURED_STATE_TOO_SMALL) &#61;&#61; 0) {
                    goodMeasure &#61; true;
                } else {
                    // Didn&#39;t fit in that size... try expanding a bit.
                    baseSize &#61; (baseSize&#43;desiredWindowWidth)/2;
                    childWidthMeasureSpec &#61; getRootMeasureSpec(baseSize, lp.width);
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                    if ((host.getMeasuredWidthAndState()&amp;View.MEASURED_STATE_TOO_SMALL) &#61;&#61; 0) {
                        goodMeasure &#61; true;
                    }
                }
            }
        }

        if (!goodMeasure) {
            childWidthMeasureSpec &#61; getRootMeasureSpec(desiredWindowWidth, lp.width);
            childHeightMeasureSpec &#61; getRootMeasureSpec(desiredWindowHeight, lp.height);
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            if (mWidth !&#61; host.getMeasuredWidth() || mHeight !&#61; host.getMeasuredHeight()) {
                windowSizeMayChange &#61; true;
            }
        }
        
        return windowSizeMayChange;
    }</code></pre><br />
<br />
<p></p>
<pre></pre>
这里会调用performMeasure&#xff08;&#xff09;方法
<p></p>
<p></p><pre><code class="language-java"> private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, &#34;measure&#34;);
        try {
          <span style="color:#ff0000;">  mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);</span>
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }</code></pre><br />
<br />
<p></p>
<p></p><pre><code class="language-java">public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        boolean optical &#61; isLayoutModeOptical(this);
        if (optical !&#61; isLayoutModeOptical(mParent)) {
            Insets insets &#61; getOpticalInsets();
            int oWidth  &#61; insets.left &#43; insets.right;
            int oHeight &#61; insets.top  &#43; insets.bottom;
            widthMeasureSpec  &#61; MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
            heightMeasureSpec &#61; MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
        }

        // Suppress sign extension for the low bytes
        long key &#61; (long) widthMeasureSpec &lt;&lt; 32 | (long) heightMeasureSpec &amp; 0xffffffffL;
        if (mMeasureCache &#61;&#61; null) mMeasureCache &#61; new LongSparseLongArray(2);

        final boolean forceLayout &#61; (mPrivateFlags &amp; PFLAG_FORCE_LAYOUT) &#61;&#61; PFLAG_FORCE_LAYOUT;
        final boolean isExactly &#61; MeasureSpec.getMode(widthMeasureSpec) &#61;&#61; MeasureSpec.EXACTLY &amp;&amp;
                MeasureSpec.getMode(heightMeasureSpec) &#61;&#61; MeasureSpec.EXACTLY;
        final boolean matchingSize &#61; isExactly &amp;&amp;
                getMeasuredWidth() &#61;&#61; MeasureSpec.getSize(widthMeasureSpec) &amp;&amp;
                getMeasuredHeight() &#61;&#61; MeasureSpec.getSize(heightMeasureSpec);
        if (forceLayout || !matchingSize &amp;&amp;
                (widthMeasureSpec !&#61; mOldWidthMeasureSpec ||
                        heightMeasureSpec !&#61; mOldHeightMeasureSpec)) {

            // first clears the measured dimension flag
            mPrivateFlags &amp;&#61; ~PFLAG_MEASURED_DIMENSION_SET;

            resolveRtlPropertiesIfNeeded();

            int cacheIndex &#61; forceLayout ? -1 : mMeasureCache.indexOfKey(key);
            if (cacheIndex &lt; 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
            <strong><span style="color:#cc0000;">    onMeasure(widthMeasureSpec, heightMeasureSpec);</span></strong>
                mPrivateFlags3 &amp;&#61; ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                long value &#61; mMeasureCache.valueAt(cacheIndex);
                // Casting a long to int drops the high 32 bits, no mask needed
                setMeasuredDimensionRaw((int) (value &gt;&gt; 32), (int) value);
                mPrivateFlags3 |&#61; PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }

            // flag not set, setMeasuredDimension() was not invoked, we raise
            // an exception to warn the developer
            if ((mPrivateFlags &amp; PFLAG_MEASURED_DIMENSION_SET) !&#61; PFLAG_MEASURED_DIMENSION_SET) {
                throw new IllegalStateException(&#34;onMeasure() did not set the&#34;
                        &#43; &#34; measured dimension by calling&#34;
                        &#43; &#34; setMeasuredDimension()&#34;);
            }

            mPrivateFlags |&#61; PFLAG_LAYOUT_REQUIRED;
        }

        mOldWidthMeasureSpec &#61; widthMeasureSpec;
        mOldHeightMeasureSpec &#61; heightMeasureSpec;

        mMeasureCache.put(key, ((long) mMeasuredWidth) &lt;&lt; 32 |
                (long) mMeasuredHeight &amp; 0xffffffffL); // suppress sign extension
    }
</code></pre>这里的public <span style="color:#ff0000"><strong>final</strong> </span>
void measure(int widthMeasureSpec, int heightMeasureSpec)中的final意味着这个函数不会被子类重写&#xff0c;这能在这个类中实现。函数实现中又调用了view的onMeasure().其实这里调用的就是ListView中的onMeasure()<p></p>
<p><br />
<br />
</p>
<p><br />
</p>
<p><br />
</p>
<pre><code class="language-java">    private void performDraw() {
   
        略。。。。
        try {
            draw(fullRedrawNeeded);
        } finally {
            mIsDrawing &#61; false;
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
       略。。。。
      
        }
    }</code></pre><br />
<br />
<p></p>
<pre><code class="language-java">   private void draw(boolean fullRedrawNeeded) {
        略。。。。
        
                if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
                    return;
                }
        略。。。。
    }
</code></pre><br />
<pre><code class="language-java"> private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
            boolean scalingRequired, Rect dirty) {
        final Canvas canvas;
        private void draw(boolean fullRedrawNeeded) {
            略。。。。
        try {
                canvas.translate(-xoff, -yoff);
                if (mTranslator !&#61; null) {
                    mTranslator.translateCanvas(canvas);
                }
                canvas.setScreenDensity(scalingRequired ? mNoncompatDensity : 0);
                attachInfo.mSetIgnoreDirtyState &#61; false;
                mView.draw(canvas);
                drawAccessibilityFocusedDrawableIfNeeded(canvas);
            }
           略。。。。

    return true;
    }</code></pre><br />
<p>这里调用的View的draw()函数&#xff0c;然后就会开始绘制了。</p>
<p></p><pre><code class="language-java"> public void draw(Canvas canvas) {
        final int privateFlags &#61; mPrivateFlags;
        final boolean dirtyOpaque &#61; (privateFlags &amp; PFLAG_DIRTY_MASK) &#61;&#61; PFLAG_DIRTY_OPAQUE &amp;&amp;
                (mAttachInfo &#61;&#61; null || !mAttachInfo.mIgnoreDirtyState);
        mPrivateFlags &#61; (privateFlags &amp; ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;
      
        int saveCount;

        if (!dirtyOpaque) {
            drawBackground(canvas);
        }

        final int viewFlags &#61; mViewFlags;
        boolean horizontalEdges &#61; (viewFlags &amp; FADING_EDGE_HORIZONTAL) !&#61; 0;
        boolean verticalEdges &#61; (viewFlags &amp; FADING_EDGE_VERTICAL) !&#61; 0;
        if (!verticalEdges &amp;&amp; !horizontalEdges) {
            if (!dirtyOpaque) onDraw(canvas);
            dispatchDraw(canvas);
            if (mOverlay !&#61; null &amp;&amp; !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }
            onDrawForeground(canvas);
            return;
        }
</code></pre>
