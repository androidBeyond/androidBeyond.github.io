---
layout:     post
title:      Android10 Choreographer原理分析
subtitle:   Choreographer翻译成中文是编舞者的意思，在Android系统4.1开始加入这个类，主要来控制同步处理输入（input），动画（animation），绘制（draw）
date:       2021-01-14
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android10
    - 系统组件
    - 系统原理
---


<h3 id="一、概述"><a href="#一、概述" class="headerlink" title="一、概述"></a>一、概述</h3><p>Choreographer翻译成中文是编舞者的意思，在Android系统4.1开始加入这个类，主要来控制同步处理输入（input），动画（animation），绘制（draw），在UI显示的时候每一帧完成的只有这三种。在这个类的前面有一行注释，大概意思就是要协调控制三个UI操作的时序，这也和其编舞者名称相符合。</p>

<pre><code>
/**
 * Coordinates the timing of animations, input and drawing.
 * 
 * The choreographer receives timing pulses (such as vertical synchronization)
 * from the display subsystem then schedules work to occur as part of rendering
 * the next display frame.
 * 
 * Applications typically interact with the choreographer indirectly using
 * higher level abstractions in the animation framework or the view hierarchy.
 * Here are some examples of things you can do using the higher-level APIs.
 * 
 * 
 * To post an animation to be processed on a regular time basis synchronized with
 * display frame rendering, use {@link android.animation.ValueAnimator#start}.
 * To post a {@link Runnable} to be invoked once at the beginning of the next display
 * frame, use {@link View#postOnAnimation}.
 * To post a {@link Runnable} to be invoked once at the beginning of the next display
 * frame after a delay, use {@link View#postOnAnimationDelayed}.
 * To post a call to {@link View#invalidate()} to occur once at the beginning of the
 * next display frame, use {@link View#postInvalidateOnAnimation()} or
 * {@link View#postInvalidateOnAnimation(int, int, int, int)}.
 * To ensure that the contents of a {@link View} scroll smoothly and are drawn in
 * sync with display frame rendering, do nothing.  This already happens automatically.
 * {@link View#onDraw} will be called at the appropriate time.
 * However, there are a few cases where you might want to use the functions of the
 * choreographer directly in your application.  Here are some examples.
 * 
 * 
 * If your application does its rendering in a different thread, possibly using GL,
 * or does not use the animation framework or view hierarchy at all
 * and you want to ensure that it is appropriately synchronized with the display, then use
 * {@link Choreographer#postFrameCallback}.
 * ... and that's about it.
 * 
 * 
 * Each {@link Looper} thread has its own choreographer.  Other threads can
 * post callbacks to run on the choreographer but they will run on the {@link Looper}
 * to which the choreographer belongs.
 *
 */</code></pre>
<p>在类中有四种任务队列，当收到Vsync信号时会执行这四种任务队列里面的任务</p>
<p>  <code>private final CallbackQueue[] mCallbackQueues;</code></p>

<pre><code>
void doFrame(long frameTimeNanos, int frame) {     
     ....
     Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
     AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);

     mFrameInfo.markInputHandlingStart();
     doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

     mFrameInfo.markAnimationsStart();
     doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);

     mFrameInfo.markPerformTraversalsStart();
     doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

     doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
     ....
}</code></pre>
<p>CALLBACK_INPUTL：输入</p>
<p>CALLBACK_ANIMATION：动画</p>
<p>CALLBACK_TRAVERSAL：绘制，执行measure，layout，draw</p>
<p>CALLBACK_COMMIT：绘制完成的提交操作，用来修正动画的启动时间。这里主要是为了解决ValueAnimator问题而引入的，因为遍历时间过长导致动画时间启动过长，时间缩短，导致跳帧，这里修改动画第一个frame开始时间延后。</p>

<pre><code>
// Update the frame time if necessary when committing the frame.
// We only update the frame time if we are more than 2 frames late reaching
// the commit phase.  This ensures that the frame time which is observed by the
// callbacks will always increase from one frame to the next and never repeat.
// We never want the next frame's starting frame time to end up being less than
// or equal to the previous frame's commit frame time.  Keep in mind that the
// next frame has most likely already been scheduled by now so we play it
// safe by ensuring the commit time is always at least one frame behind.
if (callbackType == Choreographer.CALLBACK_COMMIT) {
    final long jitterNanos = now - frameTimeNanos;
    Trace.traceCounter(Trace.TRACE_TAG_VIEW, "jitterNanos", (int) jitterNanos);
    if (jitterNanos >= 2 * mFrameIntervalNanos) {
        final long lastFrameOffset = jitterNanos % mFrameIntervalNanos
                + mFrameIntervalNanos;
        if (DEBUG_JANK) {
            Log.d(TAG, "Commit callback delayed by " + (jitterNanos * 0.000001f)
                    + " ms which is more than twice the frame interval of "
                    + (mFrameIntervalNanos * 0.000001f) + " ms!  "
                    + "Setting frame time to " + (lastFrameOffset * 0.000001f)
                    + " ms in the past.");
            mDebugPrintNextFrameTimeDelta = true;
        }
        frameTimeNanos = now - lastFrameOffset;
        mLastFrameTimeNanos = frameTimeNanos;
    }
}
</code></pre>
<h3 id="二、Choreographer启动流程"><a href="#二、Choreographer启动流程" class="headerlink" title="二、Choreographer启动流程"></a>二、Choreographer启动流程</h3><p>在前面介绍的view绘制原理的流程中，在WMG.addView会对ViewRootImpl进行初始化操作</p>
<h4 id="2-1-创建ViewRootImpl"><a href="#2-1-创建ViewRootImpl" class="headerlink" title="2.1 创建ViewRootImpl"></a>2.1 创建ViewRootImpl</h4><p>[-&gt;ViewRootImpl.java]</p>

<pre><code>
public ViewRootImpl(Context context, Display display) {
       mContext = context;
       mWindowSession = WindowManagerGlobal.getWindowSession();
       mDisplay = display;
       mBasePackageName = context.getBasePackageName();
       mThread = Thread.currentThread();
       mLocation = new WindowLeaked(null);
       mLocation.fillInStackTrace();
       mWidth = -1;
       mHeight = -1;
       mDirty = new Rect();
       mTempRect = new Rect();
       mVisRect = new Rect();
       mWinFrame = new Rect();
       mWindow = new W(this);
       mTargetSdkVersion = context.getApplicationInfo().targetSdkVersion;
       mViewVisibility = View.GONE;
       mTransparentRegion = new Region();
       mPreviousTransparentRegion = new Region();
       mFirst = true; // true for the first time the view is added
       mAdded = false;
       mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this,
               context);
       mAccessibilityManager = AccessibilityManager.getInstance(context);
       mAccessibilityManager.addAccessibilityStateChangeListener(
               mAccessibilityInteractionConnectionManager, mHandler);
       mHighContrastTextManager = new HighContrastTextManager();
       mAccessibilityManager.addHighTextContrastStateChangeListener(
               mHighContrastTextManager, mHandler);
       mViewConfiguration = ViewConfiguration.get(context);
       mDensity = context.getResources().getDisplayMetrics().densityDpi;
       mNoncompatDensity = context.getResources().getDisplayMetrics().noncompatDensityDpi;
       mFallbackEventHandler = new PhoneFallbackEventHandler(context);
       //获取Choreographer
       mChoreographer = Choreographer.getInstance();
       mDisplayManager = (DisplayManager)context.getSystemService(Context.DISPLAY_SERVICE);

       if (!sCompatibilityDone) {
           sAlwaysAssignFocus = mTargetSdkVersion < Build.VERSION_CODES.P;

           sCompatibilityDone = true;
       }

       loadSystemProperties();
   }</code></pre>
<h4 id="2-2-创建Choreographer"><a href="#2-2-创建Choreographer" class="headerlink" title="2.2 创建Choreographer"></a>2.2 创建Choreographer</h4><p>[-&gt;Choreographer.java]</p>

<pre><code>
public static Choreographer getInstance() {
       return sThreadInstance.get();
  }</code></pre>
<p>[-&gt;Choreographer.java]</p>

<pre><code>
// Thread local storage for the choreographer.
  private static final ThreadLocal&lt;Choreographer&gt; sThreadInstance =
          new ThreadLocal&lt;Choreographer&gt;() {
      @Override
      protected Choreographer initialValue() {
          //获取当前线程的Looper
          Looper looper = Looper.myLooper();
          if (looper == null) {
              throw new IllegalStateException("The current thread must have a looper!");
          }
          Choreographer choreographer = new Choreographer(looper, VSYNC_SOURCE_APP);
          //如果是主线程
          if (looper == Looper.getMainLooper()) {
              mMainInstance = choreographer;
          }
          return choreographer;
      }
  };</code></pre>
<p>当前线程所在的为主线程（UI线程），可以看到每个线程都有一个Choreographer</p>
<p>[-&gt;Choreographer.java]</p>

<pre><code>
private Choreographer(Looper looper, int vsyncSource) {
       mLooper = looper;
       //初始化FrameHandler
       mHandler = new FrameHandler(looper);
       //创建用于接收Vsync信号的对象
       mDisplayEventReceiver = USE_VSYNC
               ? new FrameDisplayEventReceiver(looper, vsyncSource)
               : null;
       mLastFrameTimeNanos = Long.MIN_VALUE;
       //getRefreshRate()一般为60Hz
       mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());
       //创建回调任务队列
       mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
       for (int i = 0; i <= CALLBACK_LAST; i++) {
           mCallbackQueues[i] = new CallbackQueue();
       }
       // b/68769804: For low FPS experiments.
       setFPSDivisor(SystemProperties.getInt(ThreadedRenderer.DEBUG_FPS_DIVISOR, 1));
   }</code></pre>
<p>mLastFrameTimeNanos：上一次帧绘制的时间点</p>
<p>mFrameIntervalNanos,帧间时长一般等于16.7ms</p>
<h4 id="2-3-初始化FrameHandler"><a href="#2-3-初始化FrameHandler" class="headerlink" title="2.3 初始化FrameHandler"></a>2.3 初始化FrameHandler</h4><p>[-&gt;Choreographer.java]</p>

<pre><code>
private final class FrameHandler extends Handler {
       public FrameHandler(Looper looper) {
           super(looper);
       }

       @Override
       public void handleMessage(Message msg) {
           switch (msg.what) {
               case MSG_DO_FRAME:
                   doFrame(System.nanoTime(), 0);
                   break;
               case MSG_DO_SCHEDULE_VSYNC:
                   doScheduleVsync();
                   break;
               case MSG_DO_SCHEDULE_CALLBACK:
                   doScheduleCallback(msg.arg1);
                   break;
           }
       }
   }</code></pre>
<h4 id="2-4-创建FrameDisplayEventReceiver"><a href="#2-4-创建FrameDisplayEventReceiver" class="headerlink" title="2.4 创建FrameDisplayEventReceiver"></a>2.4 创建FrameDisplayEventReceiver</h4><p>[-&gt;Choreographer.java]</p>

<pre><code>
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
          implements Runnable {
      ....
      public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
          super(looper, vsyncSource);
      }
      ....
  }</code></pre>
<h5 id="2-4-1-DisplayEventReceiver"><a href="#2-4-1-DisplayEventReceiver" class="headerlink" title="2.4.1 DisplayEventReceiver"></a>2.4.1 DisplayEventReceiver</h5><p>[-&gt;DisplayEventReceiver.java]</p>

<pre><code>
public DisplayEventReceiver(Looper looper) {
       this(looper, VSYNC_SOURCE_APP);
   }

   /**
    * Creates a display event receiver.
    *
    * @param looper The looper to use when invoking callbacks.
    * @param vsyncSource The source of the vsync tick. Must be on of the VSYNC_SOURCE_* values.
    */
   public DisplayEventReceiver(Looper looper, int vsyncSource) {
       if (looper == null) {
           throw new IllegalArgumentException("looper must not be null");
       }
       //获取主线程的消息队列
       mMessageQueue = looper.getQueue();
       mReceiverPtr = nativeInit(new WeakReference&lt;DisplayEventReceiver&gt;(this), mMessageQueue,
               vsyncSource);

       mCloseGuard.open("dispose");
   }</code></pre>
<h5 id="2-4-2-nativeInit"><a href="#2-4-2-nativeInit" class="headerlink" title="2.4.2 nativeInit"></a>2.4.2 nativeInit</h5><p>[-&gt;/core/jni/android_view_DisplayEventReceiver.cpp]</p>

<pre><code>
static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak,
        jobject messageQueueObj, jint vsyncSource) {
    sp&lt;MessageQueue&gt; messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
    if (messageQueue == NULL) {
        jniThrowRuntimeException(env, "MessageQueue is not initialized.");
        return 0;
    }

    sp&lt;NativeDisplayEventReceiver&gt; receiver = new NativeDisplayEventReceiver(env,
            receiverWeak, messageQueue, vsyncSource);
    status_t status = receiver-&gt;initialize();
    if (status) {
        String8 message;
        message.appendFormat("Failed to initialize display event receiver.  status=%d", status);
        jniThrowRuntimeException(env, message.string());
        return 0;
    }
    //获取DisplayEventReceiver对象的引用
    receiver-&gt;incStrong(gDisplayEventReceiverClassInfo.clazz); // retain a reference for the object
    return reinterpret_cast&lt;jlong&gt;(receiver.get());
}</code></pre>
<h5 id="2-4-3-NativeDisplayEventReceiver"><a href="#2-4-3-NativeDisplayEventReceiver" class="headerlink" title="2.4.3 NativeDisplayEventReceiver"></a>2.4.3 NativeDisplayEventReceiver</h5><p>[-&gt;/core/jni/android_view_DisplayEventReceiver.cpp]</p>

<pre><code>
NativeDisplayEventReceiver::NativeDisplayEventReceiver(JNIEnv* env,
        jobject receiverWeak, const sp&lt;MessageQueue&gt;& messageQueue, jint vsyncSource) :
        DisplayEventDispatcher(messageQueue-&gt;getLooper(),
                static_cast&lt;ISurfaceComposer::VsyncSource&gt;(vsyncSource)),
        mReceiverWeakGlobal(env-&gt;NewGlobalRef(receiverWeak)),
        mMessageQueue(messageQueue) {
    ALOGV("receiver %p ~ Initializing display event receiver.", this);
}</code></pre>
<p>DisplayEventDispatcher继承于LooperCallback，mReceiverWeakGlobal记录java层DisplayEventReceiver对象的全局引用。</p>
<h5 id="2-4-4-initialize"><a href="#2-4-4-initialize" class="headerlink" title="2.4.4 initialize"></a>2.4.4 initialize</h5><p>[-&gt;libs/androidfw/DisplayEventDispatcher.cpp]</p>

<pre><code>
status_t DisplayEventDispatcher::initialize() {
    status_t result = mReceiver.initCheck();
    if (result) {
        ALOGW("Failed to initialize display event receiver, status=%d", result);
        return result;
    }

    int rc = mLooper-&gt;addFd(mReceiver.getFd(), 0, Looper::EVENT_INPUT,
            this, NULL);
    if (rc &lt; 0) {
        return UNKNOWN_ERROR;
    }
    return OK;
}</code></pre>
<p>监听mReceiver所获取文件句柄，一旦有数据到来，则回调this,所复写LooperCallback对象的handleEvent方法</p>
<h5 id="2-4-5-addFd"><a href="#2-4-5-addFd" class="headerlink" title="2.4.5 addFd"></a>2.4.5 addFd</h5><p>[-&gt;system/core/libutils/Looper.cpp]</p>

<pre><code>
int Looper::addFd(int fd, int ident, int events, const sp&lt;LooperCallback&gt;& callback, void* data) {
#if DEBUG_CALLBACKS
    ALOGD("%p ~ addFd - fd=%d, ident=%d, events=0x%x, callback=%p, data=%p", this, fd, ident,
            events, callback.get(), data);
#endif

    if (!callback.get()) {
        if (! mAllowNonCallbacks) {
            ALOGE("Invalid attempt to set NULL callback but not allowed for this looper.");
            return -1;
        }

        if (ident &lt; 0) {
            ALOGE("Invalid attempt to set NULL callback with ident &lt; 0.");
            return -1;
        }
    } else {
        ident = POLL_CALLBACK;
    }

    { // acquire lock
        AutoMutex _l(mLock);

        Request request;
        request.fd = fd;
        request.ident = ident;
        request.events = events;
        request.seq = mNextRequestSeq++;
        //将callback赋值
        request.callback = callback;
        request.data = data;
        if (mNextRequestSeq == -1) mNextRequestSeq = 0; // reserve sequence number -1

        struct epoll_event eventItem;
        request.initEventItem(&eventItem);

        ssize_t requestIndex = mRequests.indexOfKey(fd);
        if (requestIndex &lt; 0) {
            int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, fd, & eventItem);
            if (epollResult &lt; 0) {
                ALOGE("Error adding epoll events for fd %d: %s", fd, strerror(errno));
                return -1;
            }
            mRequests.add(fd, request);
        } else {
            int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_MOD, fd, & eventItem);
            if (epollResult &lt; 0) {
                if (errno == ENOENT) {
                    // Tolerate ENOENT because it means that an older file descriptor was
                    // closed before its callback was unregistered and meanwhile a new
                    // file descriptor with the same number has been created and is now
                    // being registered for the first time.  This error may occur naturally
                    // when a callback has the side-effect of closing the file descriptor
                    // before returning and unregistering itself.  Callback sequence number
                    // checks further ensure that the race is benign.
                    //
                    // Unfortunately due to kernel limitations we need to rebuild the epoll
                    // set from scratch because it may contain an old file handle that we are
                    // now unable to remove since its file descriptor is no longer valid.
                    // No such problem would have occurred if we were using the poll system
                    // call instead, but that approach carries others disadvantages.
#if DEBUG_CALLBACKS
                    ALOGD("%p ~ addFd - EPOLL_CTL_MOD failed due to file descriptor "
                            "being recycled, falling back on EPOLL_CTL_ADD: %s",
                            this, strerror(errno));
#endif
                    epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, fd, & eventItem);
                    if (epollResult &lt; 0) {
                        ALOGE("Error modifying or adding epoll events for fd %d: %s",
                                fd, strerror(errno));
                        return -1;
                    }
                    scheduleEpollRebuildLocked();
                } else {
                    ALOGE("Error modifying epoll events for fd %d: %s", fd, strerror(errno));
                    return -1;
                }
            }
            mRequests.replaceValueAt(requestIndex, request);
        }
    } // release lock
    return 1;
}</code></pre>
<h5 id="2-4-6-handleEvent"><a href="#2-4-6-handleEvent" class="headerlink" title="2.4.6  handleEvent"></a>2.4.6  handleEvent</h5><p>[-&gt;system/core/libutils/Looper.cpp]</p>

<pre><code>
int Looper::pollInner(int timeoutMillis) {
    ...
     // Invoke all response callbacks.
    for (size_t i = 0; i &lt; mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
#if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
            ALOGD("%p ~ pollOnce - invoking fd event callback %p: fd=%d, events=0x%x, data=%p",
                    this, response.request.callback.get(), fd, events, data);
#endif
            // Invoke the callback.  Note that the file descriptor may be closed by
            // the callback (and potentially even reused) before the function returns so
            // we need to be a little careful when removing the file descriptor afterwards.
            //调用handleEvent方法
            int callbackResult = response.request.callback-&gt;handleEvent(fd, events, data);
            if (callbackResult == 0) {
                removeFd(fd, response.request.seq);
            }

            // Clear the callback reference in the response structure promptly because we
            // will not clear the response vector itself until the next poll.
            response.request.callback.clear();
            result = POLL_CALLBACK;
        }
    }
    return result;
}</code></pre>
<h3 id="三、Vsync信号回调流程"><a href="#三、Vsync信号回调流程" class="headerlink" title="三、Vsync信号回调流程"></a>三、Vsync信号回调流程</h3><p>当Vsync信号来时，经过层层调用后会执行handleEvent方法，详细流程会在SurfaceFlinger中介绍。</p>
<h4 id="3-1-handleEvent"><a href="#3-1-handleEvent" class="headerlink" title="3.1 handleEvent"></a>3.1 handleEvent</h4><p>[-&gt;libs/androidfw/DisplayEventDispatcher.cpp]</p>

<pre><code>
int DisplayEventDispatcher::handleEvent(int, int events, void*) {
    if (events & (Looper::EVENT_ERROR | Looper::EVENT_HANGUP)) {
        ALOGE("Display event receiver pipe was closed or an error occurred.  "
                "events=0x%x", events);
        return 0; // remove the callback
    }

    if (!(events & Looper::EVENT_INPUT)) {
        ALOGW("Received spurious callback for unhandled poll event.  "
                "events=0x%x", events);
        return 1; // keep the callback
    }

    // Drain all pending events, keep the last vsync.
    nsecs_t vsyncTimestamp;
    int32_t vsyncDisplayId;
    uint32_t vsyncCount;
    //清除所有的pending事件，只保留最后一次Vsync
    if (processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount)) {
        ALOGV("dispatcher %p ~ Vsync pulse: timestamp=%" PRId64 ", id=%d, count=%d",
                this, ns2ms(vsyncTimestamp), vsyncDisplayId, vsyncCount);
        mWaitingForVsync = false;
        //分发Vsync
        dispatchVsync(vsyncTimestamp, vsyncDisplayId, vsyncCount);
    }

    return 1; // keep the callback
}</code></pre>
<h5 id="3-1-1-processPendingEvents"><a href="#3-1-1-processPendingEvents" class="headerlink" title="3.1.1 processPendingEvents"></a>3.1.1 processPendingEvents</h5><p>[-&gt;libs/androidfw/DisplayEventDispatcher.cpp]</p>

<pre><code>
bool DisplayEventDispatcher::processPendingEvents(
        nsecs_t* outTimestamp, int32_t* outId, uint32_t* outCount) {
    bool gotVsync = false;
    DisplayEventReceiver::Event buf[EVENT_BUFFER_SIZE];
    ssize_t n;
    while ((n = mReceiver.getEvents(buf, EVENT_BUFFER_SIZE)) &gt; 0) {
        ALOGV("dispatcher %p ~ Read %d events.", this, int(n));
        for (ssize_t i = 0; i &lt; n; i++) {
            const DisplayEventReceiver::Event& ev = buf[i];
            switch (ev.header.type) {
            case DisplayEventReceiver::DISPLAY_EVENT_VSYNC:
                // Later vsync events will just overwrite the info from earlier
                // ones. That's fine, we only care about the most recent.
                gotVsync = true;
                *outTimestamp = ev.header.timestamp;
                *outId = ev.header.id;
                *outCount = ev.vsync.count;
                break;
            case DisplayEventReceiver::DISPLAY_EVENT_HOTPLUG:
                dispatchHotplug(ev.header.timestamp, ev.header.id, ev.hotplug.connected);
                break;
            default:
                ALOGW("dispatcher %p ~ ignoring unknown event type %#x", this, ev.header.type);
                break;
            }
        }
    }
    if (n &lt; 0) {
        ALOGW("Failed to get events from display event dispatcher, status=%d", status_t(n));
    }
    return gotVsync;
}</code></pre>
<p>遍历所有的事件，当有多个Vsync事件来时，则只关注最近一次的事件</p>
<h5 id="3-1-2-dispatchVsync-C"><a href="#3-1-2-dispatchVsync-C" class="headerlink" title="3.1.2 dispatchVsync(C++)"></a>3.1.2 dispatchVsync(C++)</h5><p>[-&gt;core/jni/android_view_DisplayEventReceiver.cpp]</p>

<pre><code>
void NativeDisplayEventReceiver::dispatchVsync(nsecs_t timestamp, int32_t id, uint32_t count) {
    JNIEnv* env = AndroidRuntime::getJNIEnv();

    ScopedLocalRef&lt;jobject&gt; receiverObj(env, jniGetReferent(env, mReceiverWeakGlobal));
    if (receiverObj.get()) {
        ALOGV("receiver %p ~ Invoking vsync handler.", this);
        //调用java层的dispatchVsync方法
        env-&gt;CallVoidMethod(receiverObj.get(),
                gDisplayEventReceiverClassInfo.dispatchVsync, timestamp, id, count);
        ALOGV("receiver %p ~ Returned from vsync handler.", this);
    }

    mMessageQueue-&gt;raiseAndClearException(env, "dispatchVsync");
}</code></pre>
<h4 id="3-2-dispatchVsync-Java"><a href="#3-2-dispatchVsync-Java" class="headerlink" title="3.2 dispatchVsync(Java)"></a>3.2 dispatchVsync(Java)</h4><p>[-&gt;DisplayEventReceiver.java]</p>

<pre><code>
// Called from native code.
   @SuppressWarnings("unused")
   @UnsupportedAppUsage
   private void dispatchVsync(long timestampNanos, int builtInDisplayId, int frame) {
       onVsync(timestampNanos, builtInDisplayId, frame);
   }</code></pre>
<p>之后的流程参考View的绘制原理。在之前写的文章中有详细分析<a href="{{site.baseurl}}/2020/10/14/Android10从WMS角度分析应用启动过程/"   target="_blank">Android10从WMS角度分析应用启动过程</a></p>
<h3 id="四、总结"><a href="#四、总结" class="headerlink" title="四、总结"></a>四、总结</h3><p>这里主要介绍了Choreographer的启动流程以及其回调的流程。</p>
<p>1.Choreographer主要来控制同步处理输入（input），动画（animation），绘制（draw）；</p>
<p>2.通过监听mReceiver所获取文件句柄，来获取回调；</p>
<p>3.可以通过Choreographer getInstance().postFrameCallback来监听帧率。</p>