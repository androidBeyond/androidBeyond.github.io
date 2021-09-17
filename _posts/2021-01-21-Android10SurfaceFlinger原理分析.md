---
layout:     post
title:      Android10 SurfaceFlinger原理分析
subtitle:   SurfaceFlinger作为负责绘制应用UI的核心，Android平台所创建的Window都是由surface所支持
date:       2021-01-21
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android10
    - Android
    - 系统组件
    - 系统原理
---

 <h3 id="一、概述"><a href="#一、概述" class="headerlink" title="一、概述"></a>一、概述</h3><p>SurfaceFlinger作为负责绘制应用UI的核心，Android平台所创建的Window都是由surface所支持，所有可见的surface渲染到显示设备都是通过SurfaceFlinger来完成的。</p>
<p>SurfaceFlinger进程是由init进程创建，运行在独立的SurfaceFlinger进程。Android应用进程必须和SurfaceFlinger进程交互，才能完成应用UI绘制到frameBuffer(帧缓冲区)，这里是通过IPC的方式进行通信的。</p>
<p>SurfaceComposerClient对象是一个比较重要的类，WMS通过该类中的mClient、mParent和SurfaceFlinger进行交互。</p>

<pre><code>
sp&lt;ISurfaceComposerClient&gt;  mClient;
wp&lt;IGraphicBufferProducer&gt;  mParent;
</code></pre>

<p>每一个应用在SurfaceFlinger中都有一个client与之相对应，当应用执行onResume方法时流程如下：</p>
<p>1.WMS会请求SurfaceFlinger来绘制Surface；</p>
<p>2.SurfaceFlinger创建Layer；</p>
<p>3.一个生产者的Binder对象通过WMS传递给应用，因此应用可以直接向SurfaceFlinger发送帧消息。</p>
<h3 id="二、启动过程"><a href="#二、启动过程" class="headerlink" title="二、启动过程"></a>二、启动过程</h3><p>SurfaceFlinger启动是通过surfaceflinger.rc启动</p>
<p>[-&gt;native/services/surfaceflinger/surfaceflinger.rc]</p>

<pre><code>
service surfaceflinger /system/bin/surfaceflinger
    class core animation
    user system
    group graphics drmrpc readproc
    onrestart restart zygote
    writepid /dev/stune/foreground/tasks
    socket pdx/system/vr/display/client     stream 0666 system graphics u:object_r:pdx_display_client_endpoint_socket:s0
    socket pdx/system/vr/display/manager    stream 0666 system graphics u:object_r:pdx_display_manager_endpoint_socket:s0
    socket pdx/system/vr/display/vsync      stream 0666 system graphics u:object_r:pdx_display_vsync_endpoint_socket:s0
</code></pre>

<p>SurfaceFlinger属于核心类，当SurfaceFlinger重启时会触发zygote重启，SurfaceFlinger服务启动是在其main函数中</p>
<h4 id="2-1-main"><a href="#2-1-main" class="headerlink" title="2.1 main"></a>2.1 main</h4><p>[-&gt;native/services/surfaceflinger/main_surfaceflinger.cpp]</p>

<pre><code>
int main(int, char**) {
    signal(SIGPIPE, SIG_IGN);
   
    hardware::configureRpcThreadpool(1 /* maxThreads */,
            false /* callerWillJoin */);

    //启动图形处理服务
    startGraphicsAllocatorService();

    // When SF is launched in its own process, limit the number of
    // binder threads to 4.
    //最大binder线程池数的个数为4
    ProcessState::self()-&gt;setThreadPoolMaxThreadCount(4);

    // start the thread pool
    // 开启线程池
    sp&lt;ProcessState&gt; ps(ProcessState::self());
    ps-&gt;startThreadPool();

    // instantiate surfaceflinger
    // 创建SurfaceFlinger
    sp&lt;SurfaceFlinger&gt; flinger = DisplayUtils::getInstance()-&gt;getSFInstance();
     
    setpriority(PRIO_PROCESS, 0, PRIORITY_URGENT_DISPLAY);

    set_sched_policy(0, SP_FOREGROUND);

    // Put most SurfaceFlinger threads in the system-background cpuset
    // Keeps us from unnecessarily using big cores
    // Do this after the binder thread pool init
    if (cpusets_enabled()) set_cpuset_policy(0, SP_SYSTEM);

    // initialize before clients can connect
    //初始化
    flinger-&gt;init();

    // publish surface flinger
    // 发布surface flinger
    sp&lt;IServiceManager&gt; sm(defaultServiceManager());
    sm-&gt;addService(String16(SurfaceFlinger::getServiceName()), flinger, false,
                   IServiceManager::DUMP_FLAG_PRIORITY_CRITICAL | IServiceManager::DUMP_FLAG_PROTO);

    // publish GpuService
    // 发布Gpu服务
    sp&lt;GpuService&gt; gpuservice = new GpuService();
    sm-&gt;addService(String16(GpuService::SERVICE_NAME), gpuservice, false);
    
    //开始显示服务
    startDisplayService(); // dependency on SF getting registered above

    struct sched_param param = {0};
    param.sched_priority = 2;
    if (sched_setscheduler(0, SCHED_FIFO, &param) != 0) {
        ALOGE("Couldn't set SCHED_FIFO");
    }

    // run surface flinger in this thread
    // 运行在当前线程
    flinger-&gt;run();

    return 0;
}</code></pre>

<p>主要的工作如下：</p>
<p>1.启动图形处理服务</p>
<p>2.开启线程池，最大binder线程池数的个数为4</p>
<p>3.设置surfacefinger进程为高优先级以及后台调度策略</p>
<p>4.创建SurfaceFlinger，并初始化</p>
<p>5.注册SurfaceFlinger服务和GpuService</p>
<p>6.开启显示服务，最后执行surfacefinger的run方法</p>
<h4 id="2-2-创建SurfaceFlinger"><a href="#2-2-创建SurfaceFlinger" class="headerlink" title="2.2 创建SurfaceFlinger"></a>2.2 创建SurfaceFlinger</h4>
<pre><code>
SurfaceFlinger* DisplayUtils::getSFInstance() {
    if (sUseExtendedImpls) {
        return new ExSurfaceFlinger();
    } else {
        return new SurfaceFlinger();
    }
}
</code></pre>

<p>[-&gt;G:/AOSP/native/services/surfaceflinger/SurfaceFlinger.cpp]</p>

<pre><code>
SurfaceFlinger::SurfaceFlinger(SurfaceFlinger::SkipInitializationTag)
      : BnSurfaceComposer(),
        mTransactionFlags(0),
        mTransactionPending(false),
        mAnimTransactionPending(false),
        mLayersRemoved(false),
        mLayersAdded(false),
        mRepaintEverything(0),
        mBootTime(systemTime()),
        mBuiltinDisplays(),
        mVisibleRegionsDirty(false),
        mGeometryInvalid(false),
        mAnimCompositionPending(false),
        mBootStage(BootStage::BOOTLOADER),
        mActiveDisplays(0),
        mBuiltInBitmask(0),
        mPluggableBitmask(0),
        mDebugRegion(0),
        mDebugDDMS(0),
        mDebugDisableHWC(0),
        mDebugDisableTransformHint(0),
        mDebugInSwapBuffers(0),
        mLastSwapBufferTime(0),
        mDebugInTransaction(0),
        mLastTransactionTime(0),
        mForceFullDamage(false),
        mPrimaryDispSync("PrimaryDispSync"),
        mPrimaryHWVsyncEnabled(false),
        mHWVsyncAvailable(false),
        mHasPoweredOff(false),
        mNumLayers(0),
        mVrFlingerRequestsDisplay(false),
        mMainThreadId(std::this_thread::get_id()),
        mCreateBufferQueue(&BufferQueue::createBufferQueue),
        mCreateNativeWindowSurface(&impl::NativeWindowSurface::create) {}
</code></pre>

<p>SurfaceFlinger继承于BnSurfaceComposer，flinger的数据类型为sp强指针类型，当首次被强指针引用时会执行onFirstRef方法。</p>
<h5 id="2-2-1-onFirstRef"><a href="#2-2-1-onFirstRef" class="headerlink" title="2.2.1 onFirstRef"></a>2.2.1 onFirstRef</h5><p>[-&gt;G:/AOSP/native/services/surfaceflinger/SurfaceFlinger.cpp]</p>

<pre><code>
void SurfaceFlinger::onFirstRef()
{
    mEventQueue-&gt;init(this);
}</code></pre>

<h5 id="2-2-2-MQ-init"><a href="#2-2-2-MQ-init" class="headerlink" title="2.2.2  MQ.init"></a>2.2.2  MQ.init</h5><p>[-&gt;G:/AOSP/native/services/surfaceflinger/MessageQueue.cpp]</p>

<pre><code>
void MessageQueue::init(const sp&lt;SurfaceFlinger&gt;& flinger) {
    mFlinger = flinger;
    mLooper = new Looper(true);
    mHandler = new Handler(*this);
}
</code></pre>

<p>handler是MessageQueue的内部类，native层的handler机制和java层的一样</p>
<h4 id="2-3-SF-init"><a href="#2-3-SF-init" class="headerlink" title="2.3 SF.init"></a>2.3 SF.init</h4><p>[-&gt;native/services/surfaceflinger/SurfaceFlinger.cpp]</p>

<pre><code>
// Do not call property_set on main thread which will be blocked by init
// Use StartPropertySetThread instead.
void SurfaceFlinger::init() {
    ALOGI(  "SurfaceFlinger's main thread ready to run. "
            "Initializing graphics H/W...");

    ALOGI("Phase offest NS: %" PRId64 "", vsyncPhaseOffsetNs);

    Mutex::Autolock _l(mStateLock);

    // start the EventThread
    // 启动app和sf的两个EventThread线程
    mEventThreadSource =
            std::make_unique&lt;DispSyncSource&gt;(&mPrimaryDispSync, SurfaceFlinger::vsyncPhaseOffsetNs,
                                             true, "app");
    mEventThread = std::make_unique&lt;impl::EventThread&gt;(mEventThreadSource.get(),
                                                       [this]() { resyncWithRateLimit(); },
                                                       impl::EventThread::InterceptVSyncsCallback(),
                                                       "appEventThread");
    mSfEventThreadSource =
            std::make_unique&lt;DispSyncSource&gt;(&mPrimaryDispSync,
                                             SurfaceFlinger::sfVsyncPhaseOffsetNs, true, "sf");

    mSFEventThread =
            std::make_unique&lt;impl::EventThread&gt;(mSfEventThreadSource.get(),
                                                [this]() { resyncWithRateLimit(); },
                                                [this](nsecs_t timestamp) {
                                                    mInterceptor-&gt;saveVSyncEvent(timestamp);
                                                },
                                                "sfEventThread");
    mEventQueue-&gt;setEventThread(mSFEventThread.get());
    mVsyncModulator.setEventThreads(mSFEventThread.get(), mEventThread.get());

    // Get a RenderEngine for the given display / config (can't fail)
    // 获取渲染引擎
    getBE().mRenderEngine =
            RE::impl::RenderEngine::create(HAL_PIXEL_FORMAT_RGBA_8888,
                                           hasWideColorDisplay
                                                   ? RE::RenderEngine::WIDE_COLOR_SUPPORT
                                                   : 0);
    LOG_ALWAYS_FATAL_IF(getBE().mRenderEngine == nullptr, "couldn't create RenderEngine");

    LOG_ALWAYS_FATAL_IF(mVrFlingerRequestsDisplay,
            "Starting with vr flinger active is not currently supported.");
    //创建HWComposer
    getBE().mHwc.reset(
            new HWComposer(std::make_unique&lt;Hwc2::impl::Composer&gt;(getBE().mHwcServiceName)));
    //注册回调
    getBE().mHwc-&gt;registerCallback(this, getBE().mComposerSequenceId);
    // Process any initial hotplug and resulting display changes.
    //处理热插拔的显示设备
    processDisplayHotplugEventsLocked();
    LOG_ALWAYS_FATAL_IF(!getBE().mHwc-&gt;isConnected(HWC_DISPLAY_PRIMARY),
            "Registered composer callback but didn't create the default primary display");

    // make the default display GLContext current so that we can create textures
    // when creating Layers (which may happens before we render something)
    getDefaultDisplayDeviceLocked()-&gt;makeCurrent();

    if (useVrFlinger) {
        auto vrFlingerRequestDisplayCallback = [this] (bool requestDisplay) {
            // This callback is called from the vr flinger dispatch thread. We
            // need to call signalTransaction(), which requires holding
            // mStateLock when we're not on the main thread. Acquiring
            // mStateLock from the vr flinger dispatch thread might trigger a
            // deadlock in surface flinger (see b/66916578), so post a message
            // to be handled on the main thread instead.
            sp&lt;LambdaMessage&gt; message = new LambdaMessage([=]() {
                ALOGI("VR request display mode: requestDisplay=%d", requestDisplay);
                mVrFlingerRequestsDisplay = requestDisplay;
                signalTransaction();
            });
            postMessageAsync(message);
        };
        mVrFlinger = dvr::VrFlinger::Create(getBE().mHwc-&gt;getComposer(),
                getBE().mHwc-&gt;getHwcDisplayId(HWC_DISPLAY_PRIMARY).value_or(0),
                vrFlingerRequestDisplayCallback);
        if (!mVrFlinger) {
            ALOGE("Failed to start vrflinger");
        }
    }
    //EventControl线程
    mEventControlThread = std::make_unique&lt;impl::EventControlThread&gt;(
            [this](bool enabled) { setVsyncEnabled(HWC_DISPLAY_PRIMARY, enabled); });

    // initialize our drawing state
    mDrawingState = mCurrentState;

    // set initial conditions (e.g. unblank default device)
    //初始化显示设备
    initializeDisplays();

    getBE().mRenderEngine-&gt;primeCache();

    // Inform native graphics APIs whether the present timestamp is supported:
    if (getHwComposer().hasCapability(
            HWC2::Capability::PresentFenceIsNotReliable)) {
        mStartPropertySetThread = new StartPropertySetThread(false);
    } else {
        mStartPropertySetThread = new StartPropertySetThread(true);
    }

    if (mStartPropertySetThread-&gt;Start() != NO_ERROR) {
        ALOGE("Run StartPropertySetThread failed!");
    }

    // This is a hack. Per definition of getDataspaceSaturationMatrix, the returned matrix
    // is used to saturate legacy sRGB content. However, to make sure the same color under
    // Display P3 will be saturated to the same color, we intentionally break the API spec
    // and apply this saturation matrix on Display P3 content. Unless the risk of applying
    // such saturation matrix on Display P3 is understood fully, the API should always return
    // identify matrix.
    mEnhancedSaturationMatrix = getBE().mHwc-&gt;getDataspaceSaturationMatrix(HWC_DISPLAY_PRIMARY,
            Dataspace::SRGB_LINEAR);

    // we will apply this on Display P3.
    if (mEnhancedSaturationMatrix != mat4()) {
        ColorSpace srgb(ColorSpace::sRGB());
        ColorSpace displayP3(ColorSpace::DisplayP3());
        mat4 srgbToP3 = mat4(ColorSpaceConnector(srgb, displayP3).getTransform());
        mat4 p3ToSrgb = mat4(ColorSpaceConnector(displayP3, srgb).getTransform());
        mEnhancedSaturationMatrix = srgbToP3 * mEnhancedSaturationMatrix * p3ToSrgb;
    }

    mBuiltInBitmask.set(HWC_DISPLAY_PRIMARY);
    for (int disp = HWC_DISPLAY_BUILTIN_2; disp &lt;= HWC_DISPLAY_BUILTIN_4; disp++) {
      mBuiltInBitmask.set(disp);
    }

    mPluggableBitmask.set(HWC_DISPLAY_EXTERNAL);
    for (int disp = HWC_DISPLAY_EXTERNAL_2; disp &lt;= HWC_DISPLAY_EXTERNAL_4; disp++) {
      mPluggableBitmask.set(disp);
    }

    ALOGV("Done initializing");
}</code></pre>

<h5 id="2-3-1-创建HWComposer"><a href="#2-3-1-创建HWComposer" class="headerlink" title="2.3.1 创建HWComposer"></a>2.3.1 创建HWComposer</h5><p>[-&gt;native/services/surfaceflinger/DisplayHardware/HWComposer.cpp]</p>

<pre><code>
HWComposer::HWComposer(std::unique_ptr&lt;android::Hwc2::Composer&gt; composer)
      : mHwcDevice(std::make_unique&lt;HWC2::Device&gt;(std::move(composer))) {}
</code></pre>

<h5 id="2-3-2-processDisplayHotplugEventsLocked"><a href="#2-3-2-processDisplayHotplugEventsLocked" class="headerlink" title="2.3.2 processDisplayHotplugEventsLocked"></a>2.3.2 processDisplayHotplugEventsLocked</h5>
<pre><code>
void SurfaceFlinger::processDisplayHotplugEventsLocked() {
    //遍历连接的设置
    for (const auto& event : mPendingHotplugEvents) {
        auto displayType = determineDisplayType(event.display, event.connection);
        if (displayType == DisplayDevice::DISPLAY_ID_INVALID) {
            ALOGW("Unable to determine the display type for display %" PRIu64, event.display);
            continue;
        }

        if (getBE().mHwc-&gt;isUsingVrComposer() && displayType == DisplayDevice::DISPLAY_EXTERNAL) {
            ALOGE("External displays are not supported by the vr hardware composer.");
            continue;
        }

        if (!getBE().mHwc-&gt;onHotplug(event.display, displayType, event.connection)) {
            continue;
        }

        if (event.connection == HWC2::Connection::Connected) {
            if (!mBuiltinDisplays[displayType].get()) {
                ALOGV("Creating built in display %d", displayType);
                mBuiltinDisplays[displayType] = new BBinder();
                // All non-virtual displays are currently considered secure.
                DisplayDeviceState info(displayType, true);
                info.displayName = displayType == DisplayDevice::DISPLAY_PRIMARY ?
                        "Built-in Screen" : "External Screen";
                mCurrentState.displays.add(mBuiltinDisplays[displayType], info);
                mInterceptor-&gt;saveDisplayCreation(info);
            }
        } else {
            ALOGV("Removing built in display %d", displayType);

            ssize_t idx = mCurrentState.displays.indexOfKey(mBuiltinDisplays[displayType]);
            if (idx &gt;= 0) {
                const DisplayDeviceState& info(mCurrentState.displays.valueAt(idx));
                mInterceptor-&gt;saveDisplayDeletion(info.displayId);
                mCurrentState.displays.removeItemsAt(idx);
            }
            mBuiltinDisplays[displayType].clear();
            if ((event.display &gt;= 0) &&
                (event.display &lt; DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES)) {
                // Display no longer exists.
                mActiveDisplays.reset(event.display);
            }
        }

        processDisplayChangesLocked();
    }

    mPendingHotplugEvents.clear();
}</code></pre>

<p>这里遍历连接的显示设备，这里的显示设置主要分成3类：主设备，拓展设备，虚拟设备，具体的处理操作在processDisplayChangesLocked函数中，见2.4.3节</p>

<pre><code>
enum DisplayType {
       DISPLAY_ID_INVALID = -1,
       DISPLAY_PRIMARY     = HWC_DISPLAY_PRIMARY,
       DISPLAY_EXTERNAL    = HWC_DISPLAY_EXTERNAL,
       DISPLAY_VIRTUAL     = HWC_DISPLAY_VIRTUAL,
       NUM_BUILTIN_DISPLAY_TYPES = HWC_NUM_PHYSICAL_DISPLAY_TYPES,
   };</code></pre>

<h5 id="2-3-3-processDisplayChangesLocked"><a href="#2-3-3-processDisplayChangesLocked" class="headerlink" title="2.3.3 processDisplayChangesLocked"></a>2.3.3 processDisplayChangesLocked</h5>
<pre><code>
void SurfaceFlinger::processDisplayChangesLocked() {
    // here we take advantage of Vector's copy-on-write semantics to
    // improve performance by skipping the transaction entirely when
    // know that the lists are identical
    const KeyedVector&lt;wp&lt;IBinder&gt;, DisplayDeviceState&gt;& curr(mCurrentState.displays);
    const KeyedVector&lt;wp&lt;IBinder&gt;, DisplayDeviceState&gt;& draw(mDrawingState.displays);
    if (!curr.isIdenticalTo(draw)) {
        mVisibleRegionsDirty = true;
        const size_t cc = curr.size();
        size_t dc = draw.size();

        // find the displays that were removed
        // (ie: in drawing state but not in current state)
        // also handle displays that changed
        // (ie: displays that are in both lists)
        for (size_t i = 0; i &lt; dc;) {
            const ssize_t j = curr.indexOfKey(draw.keyAt(i));
            if (j &lt; 0) {
                // in drawing state but not in current state
                // Call makeCurrent() on the primary display so we can
                // be sure that nothing associated with this display
                // is current.
                const sp&lt;const DisplayDevice&gt; defaultDisplay(getDefaultDisplayDeviceLocked());
                if (defaultDisplay != nullptr) defaultDisplay-&gt;makeCurrent();
                sp&lt;DisplayDevice&gt; hw(getDisplayDeviceLocked(draw.keyAt(i)));
                if (hw != nullptr) hw-&gt;disconnect(getHwComposer());
                if (draw[i].type &lt; DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES)
                    mEventThread-&gt;onHotplugReceived(draw[i].type, false);
                mDisplays.removeItem(draw.keyAt(i));
            } else {
                // this display is in both lists. see if something changed.
                const DisplayDeviceState& state(curr[j]);
                const wp&lt;IBinder&gt;& display(curr.keyAt(j));
                const sp&lt;IBinder&gt; state_binder = IInterface::asBinder(state.surface);
                const sp&lt;IBinder&gt; draw_binder = IInterface::asBinder(draw[i].surface);
                if (state_binder != draw_binder) {
                    // changing the surface is like destroying and
                    // recreating the DisplayDevice, so we just remove it
                    // from the drawing state, so that it get re-added
                    // below.
                    sp&lt;DisplayDevice&gt; hw(getDisplayDeviceLocked(display));
                    if (hw != nullptr) hw-&gt;disconnect(getHwComposer());
                    mDisplays.removeItem(display);
                    mDrawingState.displays.removeItemsAt(i);
                    dc--;
                    // at this point we must loop to the next item
                    continue;
                }

                const sp&lt;DisplayDevice&gt; disp(getDisplayDeviceLocked(display));
                if (disp != nullptr) {
                    if (state.layerStack != draw[i].layerStack) {
                        disp-&gt;setLayerStack(state.layerStack);
                    }
                    if ((state.orientation != draw[i].orientation) ||
                        (state.viewport != draw[i].viewport) || (state.frame != draw[i].frame)) {
                        disp-&gt;setProjection(state.orientation, state.viewport, state.frame);
                    }
                    if (state.width != draw[i].width || state.height != draw[i].height) {
                        disp-&gt;setDisplaySize(state.width, state.height);
                    }
                }
            }
            ++i;
        }

        // find displays that were added
        // (ie: in current state but not in drawing state)
        for (size_t i = 0; i &lt; cc; i++) {
            if (draw.indexOfKey(curr.keyAt(i)) &lt; 0) {
                const DisplayDeviceState& state(curr[i]);

                sp&lt;DisplaySurface&gt; dispSurface;
                sp&lt;IGraphicBufferProducer&gt; producer;
                sp&lt;IGraphicBufferProducer&gt; bqProducer;
                sp&lt;IGraphicBufferConsumer&gt; bqConsumer;
                //创建BufferQueue的生产者和消费者
                mCreateBufferQueue(&bqProducer, &bqConsumer, false);

                int32_t hwcId = -1;
                if (state.isVirtualDisplay()) {
                    // Virtual displays without a surface are dormant:
                    // they have external state (layer stack, projection,
                    // etc.) but no internal state (i.e. a DisplayDevice).
                    if (state.surface != nullptr) {
                        // Allow VR composer to use virtual displays.
                        if (mUseHwcVirtualDisplays || getBE().mHwc-&gt;isUsingVrComposer()) {
                            DisplayUtils *displayUtils = DisplayUtils::getInstance();
                            int width = 0;
                            int status = state.surface-&gt;query(NATIVE_WINDOW_WIDTH, &width);
                            ALOGE_IF(status != NO_ERROR, "Unable to query width (%d)", status);
                            int height = 0;
                            status = state.surface-&gt;query(NATIVE_WINDOW_HEIGHT, &height);
                            ALOGE_IF(status != NO_ERROR, "Unable to query height (%d)", status);
                            int intFormat = 0;
                            status = state.surface-&gt;query(NATIVE_WINDOW_FORMAT, &intFormat);
                            ALOGE_IF(status != NO_ERROR, "Unable to query format (%d)", status);
                            auto format = static_cast&lt;ui::PixelFormat&gt;(intFormat);

                            if (maxVirtualDisplaySize == 0 ||
                                 ( (uint64_t)width &lt;= maxVirtualDisplaySize &&
                                 (uint64_t)height &lt;= maxVirtualDisplaySize)) {
                                uint64_t usage = 0;
                                // Replace with native_window_get_consumer_usage ?
                                status = state.surface-&gt;getConsumerUsage(&usage);
                                ALOGW_IF(status != NO_ERROR, "Unable to query usage (%d)", status);
                                if ( (status == NO_ERROR) &&
                                     displayUtils-&gt;canAllocateHwcDisplayIdForVDS(usage)) {
                                    getBE().mHwc-&gt;allocateVirtualDisplay(
                                            width, height, &format, &hwcId);
                                 }
                            }
                        }

                        // TODO: Plumb requested format back up to consumer
                        DisplayUtils::getInstance()-&gt;initVDSInstance(*getBE().mHwc,
                                                        hwcId, state.surface,
                                                        dispSurface, producer,
                                                        bqProducer, bqConsumer,
                                                        state.displayName, state.isSecure);
                    }
                } else {
                    ALOGE_IF(state.surface != nullptr,
                             "adding a supported display, but rendering "
                             "surface is provided (%p), ignoring it",
                             state.surface.get());

                    hwcId = state.type;
                    dispSurface = new FramebufferSurface(*getBE().mHwc, hwcId, bqConsumer);
                    producer = bqProducer;
                }

                const wp&lt;IBinder&gt;& display(curr.keyAt(i));
                if (dispSurface != nullptr) {
                    mDisplays.add(display,
                                  setupNewDisplayDeviceInternal(display, hwcId, state, dispSurface,
                                                                producer));
                    if (!state.isVirtualDisplay()) {
                        mEventThread-&gt;onHotplugReceived(state.type, true);
                    }
                }
            }
        }
    }

    mDrawingState.displays = mCurrentState.displays;
}
</code></pre>

<h5 id="2-3-4-initializeDisplays"><a href="#2-3-4-initializeDisplays" class="headerlink" title="2.3.4 initializeDisplays"></a>2.3.4 initializeDisplays</h5><p>[-&gt;native/services/surfaceflinger/SurfaceFlinger.cpp]</p>

<pre><code>
void SurfaceFlinger::initializeDisplays() {
    class MessageScreenInitialized : public MessageBase {
        SurfaceFlinger* flinger;
    public:
        explicit MessageScreenInitialized(SurfaceFlinger* flinger) : flinger(flinger) { }
        virtual bool handler() {
            flinger-&gt;onInitializeDisplays();
            return true;
        }
    };
    sp&lt;MessageBase&gt; msg = new MessageScreenInitialized(this);
    postMessageAsync(msg);  // we may be called from main thread, use async message
} 

void SurfaceFlinger::onInitializeDisplays() {
    // reset screen orientation and use primary layer stack
    Vector&lt;ComposerState&gt; state;
    Vector&lt;DisplayState&gt; displays;
    DisplayState d;
    d.what = DisplayState::eDisplayProjectionChanged |
             DisplayState::eLayerStackChanged;
    d.token = mBuiltinDisplays[DisplayDevice::DISPLAY_PRIMARY];
    d.layerStack = 0;
    d.orientation = DisplayState::eOrientationDefault;
    d.frame.makeInvalid();
    d.viewport.makeInvalid();
    d.width = 0;
    d.height = 0;
    displays.add(d);
    setTransactionState(state, displays, 0);
    setPowerModeInternal(getDisplayDevice(d.token), HWC_POWER_MODE_NORMAL,
                         /*stateLockHeld*/ false);

    const auto& activeConfig = getBE().mHwc-&gt;getActiveConfig(HWC_DISPLAY_PRIMARY);
    const nsecs_t period = activeConfig-&gt;getVsyncPeriod();
    mAnimFrameTracker.setDisplayRefreshPeriod(period);

    // Use phase of 0 since phase is not known.
    // Use latency of 0, which will snap to the ideal latency.
    setCompositorTimingSnapped(0, period, 0);
}</code></pre>

<p>这里通过handler发送消息，进行显示设备的初始化操作。</p>
<h4 id="2-4-EventThread线程"><a href="#2-4-EventThread线程" class="headerlink" title="2.4 EventThread线程"></a>2.4 EventThread线程</h4><p>[-&gt;native/services/surfaceflinger/EventThread.cpp]</p>

<pre><code>
EventThread::EventThread(VSyncSource* src, ResyncWithRateLimitCallback resyncWithRateLimitCallback,
                         InterceptVSyncsCallback interceptVSyncsCallback, const char* threadName)
      : mVSyncSource(src),
        mResyncWithRateLimitCallback(resyncWithRateLimitCallback),
        mInterceptVSyncsCallback(interceptVSyncsCallback) {
    for (auto& event : mVSyncEvent) {
        event.header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;
        event.header.id = 0;
        event.header.timestamp = 0;
        event.vsync.count = 0;
    }

    mThread = std::thread(&EventThread::threadMain, this);

    pthread_setname_np(mThread.native_handle(), threadName);

    pid_t tid = pthread_gettid_np(mThread.native_handle());

    // Use SCHED_FIFO to minimize jitter
    constexpr int EVENT_THREAD_PRIORITY = 2;
    struct sched_param param = {0};
    param.sched_priority = EVENT_THREAD_PRIORITY;
    if (pthread_setschedparam(mThread.native_handle(), SCHED_FIFO, &param) != 0) {
        ALOGE("Couldn't set SCHED_FIFO for EventThread");
    }

    set_sched_policy(tid, SP_FOREGROUND);
}</code></pre>

<p>EventThread继承于VSyncSource::Callback</p>
<h5 id="2-4-1-onFirstRef"><a href="#2-4-1-onFirstRef" class="headerlink" title="2.4.1 onFirstRef"></a>2.4.1 onFirstRef</h5>
<pre><code>
void EventThread::Connection::onFirstRef() {
    // NOTE: mEventThread doesn't hold a strong reference on us
    mEventThread->registerDisplayEventConnection(this);
}</code></pre>

<p>注册显示设备事件</p>
<h5 id="2-4-2-threadMain"><a href="#2-4-2-threadMain" class="headerlink" title="2.4.2 threadMain"></a>2.4.2 threadMain</h5>
<pre><code>
void EventThread::threadMain() NO_THREAD_SAFETY_ANALYSIS {
    std::unique_lock&lt;std::mutex&gt; lock(mMutex);
    while (mKeepRunning) {
        DisplayEventReceiver::Event event;
        Vector&lt;sp&lt;EventThread::Connection&gt; &gt; signalConnections;
        //见2.4.3节
        signalConnections = waitForEventLocked(&lock, &event);

        // dispatch events to listeners...
        const size_t count = signalConnections.size();
        for (size_t i = 0; i &lt; count; i++) {
            const sp&lt;Connection&gt;& conn(signalConnections[i]);
            // now see if we still need to report this event
            //分发事件
            status_t err = conn-&gt;
            postEvent(event);
            if (err == -EAGAIN || err == -EWOULDBLOCK) {
                // The destination doesn't accept events anymore, it's probably
                // full. For now, we just drop the events on the floor.
                // FIXME: Note that some events cannot be dropped and would have
                // to be re-sent later.
                // Right-now we don't have the ability to do this.
                ALOGW("EventThread: dropping event (%08x) for connection %p", event.header.type,
                      conn.get());
            } else if (err &lt; 0) {
                // handle any other error on the pipe as fatal. the only
                // reasonable thing to do is to clean-up this connection.
                // The most common error we'll get here is -EPIPE.
                //清除连接
                removeDisplayEventConnectionLocked(signalConnections[i]);
            }
        }
    }
}
</code></pre>

<h5 id="2-4-3-waitForEventLocked"><a href="#2-4-3-waitForEventLocked" class="headerlink" title="2.4.3 waitForEventLocked"></a>2.4.3 waitForEventLocked</h5>
<pre><code>
// This will return when (1) a vsync event has been received, and (2) there was
// at least one connection interested in receiving it when we started waiting.
Vector&lt;sp&lt;EventThread::Connection&gt; &gt; EventThread::waitForEventLocked(
        std::unique_lock&lt;std::mutex&gt;* lock, DisplayEventReceiver::Event* event) {
    Vector&lt;sp&lt;EventThread::Connection&gt; &gt; signalConnections;

    while (signalConnections.isEmpty() && mKeepRunning) {
        bool eventPending = false;
        bool waitForVSync = false;

        size_t vsyncCount = 0;
        nsecs_t timestamp = 0;
        for (int32_t i = 0; i &lt; DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES; i++) {
            timestamp = mVSyncEvent[i].header.timestamp;
            if (timestamp) {
                // we have a vsync event to dispatch
                if (mInterceptVSyncsCallback) {
                    mInterceptVSyncsCallback(timestamp);
                }
                *event = mVSyncEvent[i];
                mVSyncEvent[i].header.timestamp = 0;
                vsyncCount = mVSyncEvent[i].vsync.count;
                break;
            }
        }

        // find out connections waiting for events
        size_t count = mDisplayEventConnections.size();
        if (!timestamp && count) {
            // no vsync event, see if there are some other event
            // 没有vsync事件查看其他事件
            eventPending = !mPendingEvents.isEmpty();
            if (eventPending) {
                // we have some other event to dispatch
                *event = mPendingEvents[0];
                mPendingEvents.removeAt(0);
            }
        }

        for (size_t i = 0; i &lt; count;) {
            sp&lt;Connection&gt; connection(mDisplayEventConnections[i].promote());
            if (connection != nullptr) {
                bool added = false;
                if (connection-&gt;count &gt;= 0) {
                    // we need vsync events because at least
                    // one connection is waiting for it
                    //需要vysnc事件，因为至少需要一个连接正在等待vsync
                    waitForVSync = true;
                    if (timestamp) {
                        // we consume the event only if it's time
                        // (ie: we received a vsync event)
                        if (connection-&gt;count == 0) {
                            // fired this time around
                            connection-&gt;count = -1;
                            signalConnections.add(connection);
                            added = true;
                        } else if (connection-&gt;count == 1 ||
                                   (vsyncCount % connection-&gt;count) == 0) {
                            // continuous event, and time to report it
                            signalConnections.add(connection);
                            added = true;
                        }
                    }
                }

                if (eventPending && !timestamp && !added) {
                    // we don't have a vsync event to process
                    // (timestamp==0), but we have some pending
                    // messages.
                    signalConnections.add(connection);
                }
                ++i;
            } else {
                // we couldn't promote this reference, the connection has
                // died, so clean-up!
                mDisplayEventConnections.removeAt(i);
                --count;
            }
        }

        // Here we figure out if we need to enable or disable vsyncs
        if (timestamp && !waitForVSync) {
            // we received a VSYNC but we have no clients
            // don't report it, and disable VSYNC events
            //接收Vsync，但没有client需要它，则直接关闭VYSNC
            disableVSyncLocked();
        } else if (!timestamp && waitForVSync) {
            // we have at least one client, so we want vsync enabled
            // (TODO: this function is called right after we finish
            // notifying clients of a vsync, so this call will be made
            // at the vsync rate, e.g. 60fps.  If we can accurately
            // track the current state we could avoid making this call
            // so often.)
            //至少存在一个Client，则需要使能VSYNC
            enableVSyncLocked();
        }

        // note: !timestamp implies signalConnections.isEmpty(), because we
        // don't populate signalConnections if there's no vsync pending
        if (!timestamp && !eventPending) {
            // wait for something to happen
            if (waitForVSync) {
                // This is where we spend most of our time, waiting
                // for vsync events and new client registrations.
                //
                // If the screen is off, we can't use h/w vsync, so we
                // use a 16ms timeout instead.  It doesn't need to be
                // precise, we just need to keep feeding our clients.
                //
                // We don't want to stall if there's a driver bug, so we
                // use a (long) timeout when waiting for h/w vsync, and
                // generate fake events when necessary.
                bool softwareSync = mUseSoftwareVSync;
                auto timeout = softwareSync ? 16ms : 1000ms;
                if (mCondition.wait_for(*lock, timeout) == std::cv_status::timeout) {
                    if (!softwareSync) {
                        ALOGW("Timed out waiting for hw vsync; faking it");
                    }
                    // FIXME: how do we decide which display id the fake
                    // vsync came from ?
                    mVSyncEvent[0].header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;
                    mVSyncEvent[0].header.id = DisplayDevice::DISPLAY_PRIMARY;
                    mVSyncEvent[0].header.timestamp = systemTime(SYSTEM_TIME_MONOTONIC);
                    mVSyncEvent[0].vsync.count++;
                }
            } else {
                // Nobody is interested in vsync, so we just want to sleep.
                // h/w vsync should be disabled, so this will wait until we
                // get a new connection, or an existing connection becomes
                // interested in receiving vsync again.
                //不存在对vsync感兴趣的连接，则进入休眠
                mCondition.wait(*lock);
            }
        }
    }

    // here we're guaranteed to have a timestamp and some connections to signal
    // (The connections might have dropped out of mDisplayEventConnections
    // while we were asleep, but we'll still have strong references to them.)
    return signalConnections;
}</code></pre>

<p>EventThread线程进入mCondition的wait方法，等待唤醒</p>
<h4 id="2-5-setEventThread"><a href="#2-5-setEventThread" class="headerlink" title="2.5 setEventThread"></a>2.5 setEventThread</h4><p>[-&gt;native/services/surfaceflinger/MessageQueue.cpp]</p>

<pre><code>
void MessageQueue::setEventThread(android::EventThread* eventThread) {
    if (mEventThread == eventThread) {
        return;
    }

    if (mEventTube.getFd() &gt;= 0) {
        mLooper-&gt;removeFd(mEventTube.getFd());
    }

    mEventThread = eventThread;
    mEvents = eventThread-&gt;createEventConnection();
    mEvents-&gt;stealReceiveChannel(&mEventTube);
    mLooper-&gt;addFd(mEventTube.getFd(), 0, Looper::EVENT_INPUT, MessageQueue::cb_eventReceiver,
                   this);
}</code></pre>

<p>这里主要是监听EventTube（类型为BitTube），当有数据来的时候，调用cb_eventReceiver方法</p>
<h4 id="2-6-SF-run"><a href="#2-6-SF-run" class="headerlink" title="2.6 SF.run"></a>2.6 SF.run</h4><p>[-&gt;native/services/surfaceflinger/SurfaceFlinger.cpp]</p>

<pre><code>
void SurfaceFlinger::run() {
    do {
        waitForEvent();
    } while (true);
}

void SurfaceFlinger::waitForEvent() {
    mEventQueue-&gt;waitMessage();
}

void MessageQueue::waitMessage() {
    do {
        IPCThreadState::self()-&gt;flushCommands();
        int32_t ret = mLooper-&gt;pollOnce(-1);
        switch (ret) {
            case Looper::POLL_WAKE:
            case Looper::POLL_CALLBACK:
                continue;
            case Looper::POLL_ERROR:
                ALOGE("Looper::POLL_ERROR");
                continue;
            case Looper::POLL_TIMEOUT:
                // timeout (should not happen)
                continue;
            default:
                // should not happen
                ALOGE("Looper::pollOnce() returned unknown status %d", ret);
                continue;
        }
    } while (true);
}</code></pre>

<p>这里是一个while循环，一直在等待消息，如果有消息就进行处理。</p>
<h3 id="三、Vsync信号"><a href="#三、Vsync信号" class="headerlink" title="三、Vsync信号"></a>三、Vsync信号</h3><p>前面2.4.1创建HWComposer过程中，会注册一些回调方法。</p>
<h4 id="3-1-registerCallback"><a href="#3-1-registerCallback" class="headerlink" title="3.1 registerCallback"></a>3.1 registerCallback</h4>
<pre><code>
//创建HWComposer
getBE().mHwc.reset(
        new HWComposer(std::make_unique&lt;Hwc2::impl::Composer&gt;(getBE().mHwcServiceName)));
//注册回调
getBE().mHwc-&gt;registerCallback(this, getBE().mComposerSequenceId); //注册回调
getBE().mHwc-&gt;registerCallback(this, getBE().mComposerSequenceId);</code></pre>

<p>当硬件产生Vsync信号时，则会回调onVsyncReceived方法，SurfaceFlinger继承ComposerCallback。</p>

<pre><code>
class ComposerCallback {
 public:
    virtual void onHotplugReceived(int32_t sequenceId, hwc2_display_t display,
                                   Connection connection) = 0;
    virtual void onRefreshReceived(int32_t sequenceId,
                                   hwc2_display_t display) = 0;
    virtual void onVsyncReceived(int32_t sequenceId, hwc2_display_t display,
                                 int64_t timestamp) = 0;
    virtual ~ComposerCallback() = default;
};
</code></pre>

<h4 id="3-2-onVsyncReceived"><a href="#3-2-onVsyncReceived" class="headerlink" title="3.2 onVsyncReceived"></a>3.2 onVsyncReceived</h4>
<pre><code>
void SurfaceFlinger::onVsyncReceived(int32_t sequenceId,
        hwc2_display_t displayId, int64_t timestamp) {
    Mutex::Autolock lock(mStateLock);
    // Ignore any vsyncs from a previous hardware composer.
    if (sequenceId != getBE().mComposerSequenceId) {
        return;
    }

    int32_t type;
    if (!getBE().mHwc-&gt;onVsync(displayId, timestamp, &type)) {
        return;
    }

    bool needsHwVsync = false;

    { // Scope for the lock
        Mutex::Autolock _l(mHWVsyncLock);
        if (type == DisplayDevice::DISPLAY_PRIMARY && mPrimaryHWVsyncEnabled) {
            needsHwVsync = mPrimaryDispSync.addResyncSample(timestamp);
        }
    }

    if (needsHwVsync) {
        enableHardwareVsync();
    } else {
        disableHardwareVsync(false);
    }
}</code></pre>

<h4 id="3-3-addResyncSample"><a href="#3-3-addResyncSample" class="headerlink" title="3.3 addResyncSample"></a>3.3 addResyncSample</h4><p>[-&gt;native/services/surfaceflinger/DispSync.cpp]</p>

<pre><code>
bool DispSync::addResyncSample(nsecs_t timestamp) {
    Mutex::Autolock lock(mMutex);

    ALOGV("[%s] addResyncSample(%" PRId64 ")", mName, ns2us(timestamp));

    size_t idx = (mFirstResyncSample + mNumResyncSamples) % MAX_RESYNC_SAMPLES;
    mResyncSamples[idx] = timestamp;
    if (mNumResyncSamples == 0) {
        mPhase = 0;
        mReferenceTime = timestamp;
        ALOGV("[%s] First resync sample: mPeriod = %" PRId64 ", mPhase = 0, "
              "mReferenceTime = %" PRId64,
              mName, ns2us(mPeriod), ns2us(mReferenceTime));
        mThread-&gt;updateModel(mPeriod, mPhase, mReferenceTime);
    }

    if (mNumResyncSamples &lt; MAX_RESYNC_SAMPLES) {
        mNumResyncSamples++;
    } else {
        mFirstResyncSample = (mFirstResyncSample + 1) % MAX_RESYNC_SAMPLES;
    }
    //见3.4节
    updateModelLocked();

    if (mNumResyncSamplesSincePresent++ &gt; MAX_RESYNC_SAMPLES_WITHOUT_PRESENT) {
        resetErrorLocked();
    }

    if (mIgnorePresentFences) {
        // If we don't have the sync framework we will never have
        // addPresentFence called.  This means we have no way to know whether
        // or not we're synchronized with the HW vsyncs, so we just request
        // that the HW vsync events be turned on whenever we need to generate
        // SW vsync events.
        return mThread-&gt;hasAnyEventListeners();
    }

    // Check against kErrorThreshold / 2 to add some hysteresis before having to
    // resync again
    bool modelLocked = mModelUpdated && mError &lt; (kErrorThreshold / 2);
    ALOGV("[%s] addResyncSample returning %s", mName, modelLocked ? "locked" : "unlocked");
    return !modelLocked;
}</code></pre>

<h5 id="3-3-1-DispSync初始化"><a href="#3-3-1-DispSync初始化" class="headerlink" title="3.3.1 DispSync初始化"></a>3.3.1 DispSync初始化</h5>
<pre><code>
void DispSync::init(bool hasSyncFramework, int64_t dispSyncPresentTimeOffset) {
    mIgnorePresentFences = !hasSyncFramework;
    mPresentTimeOffset = dispSyncPresentTimeOffset;
    mThread-&gt;run("DispSync", PRIORITY_URGENT_DISPLAY + PRIORITY_MORE_FAVORABLE);

    // set DispSync to SCHED_FIFO to minimize jitter
    struct sched_param param = {0};
    param.sched_priority = 2;
    if (sched_setscheduler(mThread-&gt;getTid(), SCHED_FIFO, &param) != 0) {
        ALOGE("Couldn't set SCHED_FIFO for DispSyncThread");
    }

    reset();
    beginResync();

    if (kTraceDetailedInfo) {
        // If we're not getting present fences then the ZeroPhaseTracer
        // would prevent HW vsync event from ever being turned off.
        // Even if we're just ignoring the fences, the zero-phase tracing is
        // not needed because any time there is an event registered we will
        // turn on the HW vsync events.
        if (!mIgnorePresentFences && kEnableZeroPhaseTracer) {
            mZeroPhaseTracer = std::make_unique&lt;ZeroPhaseTracer&gt;();
            addEventListener("ZeroPhaseTracer", 0, mZeroPhaseTracer.get());
        }
    }
}</code></pre>

<h5 id="3-3-2-DispSyncThread-run"><a href="#3-3-2-DispSyncThread-run" class="headerlink" title="3.3.2 DispSyncThread.run"></a>3.3.2 DispSyncThread.run</h5>
<pre><code>
virtual bool threadLoop() {
       status_t err;
       nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);

       while (true) {
           Vector&lt;CallbackInvocation&gt; callbackInvocations;

           nsecs_t targetTime = 0;

           { // Scope for lock
               Mutex::Autolock lock(mMutex);

               if (kTraceDetailedInfo) {
                   ATRACE_INT64("DispSync:Frame", mFrameNumber);
               }
               ALOGV("[%s] Frame %" PRId64, mName, mFrameNumber);
               ++mFrameNumber;

               if (mStop) {
                   return false;
               }

               if (mPeriod == 0) {
                   err = mCond.wait(mMutex);
                   if (err != NO_ERROR) {
                       ALOGE("error waiting for new events: %s (%d)", strerror(-err), err);
                       return false;
                   }
                   continue;
               }

               targetTime = computeNextEventTimeLocked(now);

               bool isWakeup = false;

               if (now &lt; targetTime) {
                   if (kTraceDetailedInfo) ATRACE_NAME("DispSync waiting");

                   if (targetTime == INT64_MAX) {
                       ALOGV("[%s] Waiting forever", mName);
                       err = mCond.wait(mMutex);
                   } else {
                       ALOGV("[%s] Waiting until %" PRId64, mName, ns2us(targetTime));
                       err = mCond.waitRelative(mMutex, targetTime - now);
                   }

                   if (err == TIMED_OUT) {
                       isWakeup = true;
                   } else if (err != NO_ERROR) {
                       ALOGE("error waiting for next event: %s (%d)", strerror(-err), err);
                       return false;
                   }
               }

               now = systemTime(SYSTEM_TIME_MONOTONIC);

               // Don't correct by more than 1.5 ms
               static const nsecs_t kMaxWakeupLatency = us2ns(1500);

               if (isWakeup) {
                   mWakeupLatency = ((mWakeupLatency * 63) + (now - targetTime)) / 64;
                   mWakeupLatency = min(mWakeupLatency, kMaxWakeupLatency);
                   if (kTraceDetailedInfo) {
                       ATRACE_INT64("DispSync:WakeupLat", now - targetTime);
                       ATRACE_INT64("DispSync:AvgWakeupLat", mWakeupLatency);
                   }
               }

               callbackInvocations = gatherCallbackInvocationsLocked(now);
           }

           if (callbackInvocations.size() &gt; 0) {
               fireCallbackInvocations(callbackInvocations);
           }
       }

       return false;
   }</code></pre>

<h4 id="3-4-DS-updateModelLocked"><a href="#3-4-DS-updateModelLocked" class="headerlink" title="3.4 DS.updateModelLocked"></a>3.4 DS.updateModelLocked</h4>
<pre><code>
void DispSync::updateModelLocked() {
    ALOGV("[%s] updateModelLocked %zu", mName, mNumResyncSamples);
    if (mNumResyncSamples &gt;= MIN_RESYNC_SAMPLES_FOR_UPDATE) {
        ALOGV("[%s] Computing...", mName);
        nsecs_t durationSum = 0;
        nsecs_t minDuration = INT64_MAX;
        nsecs_t maxDuration = 0;
        for (size_t i = 1; i &lt; mNumResyncSamples; i++) {
            size_t idx = (mFirstResyncSample + i) % MAX_RESYNC_SAMPLES;
            size_t prev = (idx + MAX_RESYNC_SAMPLES - 1) % MAX_RESYNC_SAMPLES;
            nsecs_t duration = mResyncSamples[idx] - mResyncSamples[prev];
            durationSum += duration;
            minDuration = min(minDuration, duration);
            maxDuration = max(maxDuration, duration);
        }

        // Exclude the min and max from the average
        durationSum -= minDuration + maxDuration;
        mPeriod = durationSum / (mNumResyncSamples - 3);

        ALOGV("[%s] mPeriod = %" PRId64, mName, ns2us(mPeriod));

        double sampleAvgX = 0;
        double sampleAvgY = 0;
        double scale = 2.0 * M_PI / double(mPeriod);
        // Intentionally skip the first sample
        for (size_t i = 1; i &lt; mNumResyncSamples; i++) {
            size_t idx = (mFirstResyncSample + i) % MAX_RESYNC_SAMPLES;
            nsecs_t sample = mResyncSamples[idx] - mReferenceTime;
            double samplePhase = double(sample % mPeriod) * scale;
            sampleAvgX += cos(samplePhase);
            sampleAvgY += sin(samplePhase);
        }

        sampleAvgX /= double(mNumResyncSamples - 1);
        sampleAvgY /= double(mNumResyncSamples - 1);

        mPhase = nsecs_t(atan2(sampleAvgY, sampleAvgX) / scale);

        ALOGV("[%s] mPhase = %" PRId64, mName, ns2us(mPhase));

        if (mPhase &lt; -(mPeriod / 2)) {
            mPhase += mPeriod;
            ALOGV("[%s] Adjusting mPhase -&gt; %" PRId64, mName, ns2us(mPhase));
        }

        if (kTraceDetailedInfo) {
            ATRACE_INT64("DispSync:Period", mPeriod);
            ATRACE_INT64("DispSync:Phase", mPhase + mPeriod / 2);
        }

        // Artificially inflate the period if requested.
        mPeriod += mPeriod * mRefreshSkipCount;
        //见3.5节
        mThread-&gt;updateModel(mPeriod, mPhase, mReferenceTime);
        mModelUpdated = true;
    }
}</code></pre>

<h4 id="3-5-DST-updateModel"><a href="#3-5-DST-updateModel" class="headerlink" title="3.5 DST.updateModel"></a>3.5 DST.updateModel</h4><p>[-&gt;native/services/surfaceflinger/DispSync.cpp]</p>

<pre><code>
void updateModel(nsecs_t period, nsecs_t phase, nsecs_t referenceTime) {
        if (kTraceDetailedInfo) ATRACE_CALL();
        Mutex::Autolock lock(mMutex);
        mPeriod = period;
        mPhase = phase;
        mReferenceTime = referenceTime;
        ALOGV("[%s] updateModel: mPeriod = %" PRId64 ", mPhase = %" PRId64
              " mReferenceTime = %" PRId64,
              mName, ns2us(mPeriod), ns2us(mPhase), ns2us(mReferenceTime));
        //唤醒目标线程
        mCond.signal();
    }
</code></pre>

<p>后面进入到DispSyncThread线程，线程里面具体的执行方法在3.3.2中有详细介绍，这里主要看下 fireCallbackInvocations方法。</p>

<pre><code>
void fireCallbackInvocations(const Vector&lt;CallbackInvocation&gt;& callbacks) {
       if (kTraceDetailedInfo) ATRACE_CALL();
       for (size_t i = 0; i &lt; callbacks.size(); i++) {
           callbacks[i].mCallback-&gt;onDispSyncEvent(callbacks[i].mEventTime);
       }
   }
</code></pre>

<p>在SurfaceFlinger里面有调用init方法，其中创建了DispSyncSource对象，这里是调用了DispSyncSource的onDispSyncEvent方法。</p>
<h4 id="3-6-DSS-onDispSyncEvent"><a href="#3-6-DSS-onDispSyncEvent" class="headerlink" title="3.6 DSS.onDispSyncEvent"></a>3.6 DSS.onDispSyncEvent</h4><p>[-&gt;native/services/surfaceflinger/SurfaceFlinger.cpp::DispSyncSource]</p>

<pre><code>
virtual void onDispSyncEvent(nsecs_t when) {
    VSyncSource::Callback* callback;
    {
        Mutex::Autolock lock(mCallbackMutex);
        callback = mCallback;

        if (mTraceVsync) {
            mValue = (mValue + 1) % 2;
            ATRACE_INT(mVsyncEventLabel.string(), mValue);
        }
    }

    if (callback != nullptr) {
        callback-&gt;onVSyncEvent(when);
    }
}
</code></pre>

<h4 id="3-7-onVSyncEvent"><a href="#3-7-onVSyncEvent" class="headerlink" title="3.7 onVSyncEvent"></a>3.7 onVSyncEvent</h4>
<pre><code>
void EventThread::onVSyncEvent(nsecs_t timestamp) {
    std::lock_guard&lt;std::mutex&gt; lock(mMutex);
    mVSyncEvent[0].header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;
    mVSyncEvent[0].header.id = 0;
    mVSyncEvent[0].header.timestamp = timestamp;
    mVSyncEvent[0].vsync.count++;
    mCondition.notify_all();
}</code></pre>

<p> mCondition.notify_all能够唤醒处理waitForEventLocked的EventThread（2.4.2）,并执行postEvent</p>
<h4 id="3-8-ET-postEvent"><a href="#3-8-ET-postEvent" class="headerlink" title="3.8 ET.postEvent"></a>3.8 ET.postEvent</h4>
<pre><code>
status_t EventThread::Connection::postEvent(const DisplayEventReceiver::Event& event) {
    ssize_t size = DisplayEventReceiver::sendEvents(&mChannel, &event, 1);
    return size &lt; 0 ? status_t(size) : status_t(NO_ERROR);
}</code></pre>

<h4 id="3-9-DER-sendEvents"><a href="#3-9-DER-sendEvents" class="headerlink" title="3.9 DER.sendEvents"></a>3.9 DER.sendEvents</h4><p>[-&gt;native\libs\gui\DisplayEventReceiver.cpp]</p>

<pre><code>
ssize_t DisplayEventReceiver::sendEvents(gui::BitTube* dataChannel,
        Event const* events, size_t count)
{
    return gui::BitTube::sendObjects(dataChannel, events, count);
}
</code></pre>

<p>在2.5节中有监听BitTube，此处调用sendObjects，当收到数据时，则调用回调方法。</p>
<h5 id="3-9-1-MQ-cb-eventReceiver"><a href="#3-9-1-MQ-cb-eventReceiver" class="headerlink" title="3.9.1 MQ.cb_eventReceiver"></a>3.9.1 MQ.cb_eventReceiver</h5>
<pre><code>
int MessageQueue::cb_eventReceiver(int fd, int events, void* data) {
    MessageQueue* queue = reinterpret_cast&lt;MessageQueue*&gt;(data);
    return queue-&gt;eventReceiver(fd, events);
}</code></pre>

<h5 id="3-9-2-MQ-eventReceiver"><a href="#3-9-2-MQ-eventReceiver" class="headerlink" title="3.9.2  MQ.eventReceiver"></a>3.9.2  MQ.eventReceiver</h5>
<pre><code>
int MessageQueue::eventReceiver(int /*fd*/, int /*events*/) {
    ssize_t n;
    DisplayEventReceiver::Event buffer[8];
    while ((n = DisplayEventReceiver::getEvents(&mEventTube, buffer, 8)) &gt; 0) {
        for (int i = 0; i &lt; n; i++) {
            if (buffer[i].header.type == DisplayEventReceiver::DISPLAY_EVENT_VSYNC) {
                mHandler-&gt;dispatchInvalidate();
                break;
            }
        }
    }
    return 1;
}
</code></pre>

<h4 id="3-10-MQ-dispatchInvalidate"><a href="#3-10-MQ-dispatchInvalidate" class="headerlink" title="3.10 MQ.dispatchInvalidate"></a>3.10 MQ.dispatchInvalidate</h4>
<pre><code>
void MessageQueue::Handler::dispatchInvalidate() {
    if ((android_atomic_or(eventMaskInvalidate, &mEventMask) & eventMaskInvalidate) == 0) {
        mQueue.mLooper-&gt;sendMessage(this, Message(MessageQueue::INVALIDATE));
    }
}</code></pre>

<h4 id="3-11-MQ-handleMessage"><a href="#3-11-MQ-handleMessage" class="headerlink" title="3.11 MQ.handleMessage"></a>3.11 MQ.handleMessage</h4>
<pre><code>
void MessageQueue::Handler::handleMessage(const Message& message) {
    switch (message.what) {
        case INVALIDATE:
            android_atomic_and(~eventMaskInvalidate, &mEventMask);
            mQueue.mFlinger-&gt;onMessageReceived(message.what);
            break;
        case REFRESH:
            android_atomic_and(~eventMaskRefresh, &mEventMask);
            mQueue.mFlinger-&gt;onMessageReceived(message.what);
            break;
    }
}</code></pre>

<h4 id="3-12-SF-onMessageReceived"><a href="#3-12-SF-onMessageReceived" class="headerlink" title="3.12 SF.onMessageReceived"></a>3.12 SF.onMessageReceived</h4>
<pre><code>
void SurfaceFlinger::onMessageReceived(int32_t what) {
    ATRACE_CALL();
    switch (what) {
        case MessageQueue::INVALIDATE: {
            bool frameMissed = !mHadClientComposition &&
                    mPreviousPresentFence != Fence::NO_FENCE &&
                    (mPreviousPresentFence-&gt;getSignalTime() ==
                            Fence::SIGNAL_TIME_PENDING);
            ATRACE_INT("FrameMissed", static_cast&lt;int&gt;(frameMissed));
            if (frameMissed) {
                mTimeStats.incrementMissedFrames();
                if (mPropagateBackpressure) {
                    signalLayerUpdate();
                    break;
                }
            }

            if (mDolphinFuncsEnabled) {
                int maxQueuedFrames = 0;
                mDrawingState.traverseInZOrder([&](Layer* layer) {
                    if (layer-&gt;hasQueuedFrame() &&
                            layer-&gt;shouldPresentNow(mPrimaryDispSync)) {
                        int layerQueuedFrames = layer-&gt;getQueuedFrameCount();
                        if (maxQueuedFrames &lt; layerQueuedFrames &&
                                !layer-&gt;visibleNonTransparentRegion.isEmpty()) {
                            maxQueuedFrames = layerQueuedFrames;
                        }
                    }
                });
                if(mDolphinMonitor(maxQueuedFrames)) {
                    signalLayerUpdate();
                    break;
                }
            }

            // Now that we're going to make it to the handleMessageTransaction()
            // call below it's safe to call updateVrFlinger(), which will
            // potentially trigger a display handoff.
            updateVrFlinger();

            bool refreshNeeded = handleMessageTransaction();
            refreshNeeded |= handleMessageInvalidate();
            refreshNeeded |= mRepaintEverything;
            //如果需要刷新
            if (refreshNeeded && CC_LIKELY(mBootStage != BootStage::BOOTLOADER)) {
                // Signal a refresh if a transaction modified the window state,
                // a new buffer was latched, or if HWC has requested a full
                // repaint
                if (mDolphinFuncsEnabled) {
                    mDolphinRefresh();
                }
                signalRefresh();
            }
            break;
        }
        case MessageQueue::REFRESH: {
            handleMessageRefresh();
            break;
        }
    }
}</code></pre>

<h3 id="四、图像输出"><a href="#四、图像输出" class="headerlink" title="四、图像输出"></a>四、图像输出</h3><p>上面经过Vsync信号后，经过层层调用到onMessageReceived方法，如果屏幕刷新，则会调用到handleMessageRefresh方法流程，具体如下：</p>
<h4 id="4-1-SF-handleMessageRefresh"><a href="#4-1-SF-handleMessageRefresh" class="headerlink" title="4.1 SF.handleMessageRefresh"></a>4.1 SF.handleMessageRefresh</h4>
<pre><code>
void SurfaceFlinger::handleMessageRefresh() {
    ATRACE_CALL();

    mRefreshPending = false;

    nsecs_t refreshStartTime = systemTime(SYSTEM_TIME_MONOTONIC);
    //主要是这个5步操作
    preComposition(refreshStartTime);
    rebuildLayerStacks();
    setUpHWComposer();
    doDebugFlashRegions();
    doTracing("handleRefresh");
    logLayerStats();
    doComposition();
    postComposition(refreshStartTime);

    int id = getVsyncSource();
    mPreviousPresentFence = (id != -1) ? getBE().mHwc-&gt;getPresentFence(id) : Fence::NO_FENCE;
    ALOGV("Checking for backpressure against %d retire fence", id);

    mHadClientComposition = false;
    for (size_t displayId = 0; displayId &lt; mDisplays.size(); ++displayId) {
        const sp&lt;DisplayDevice&gt;& displayDevice = mDisplays[displayId];
        mHadClientComposition = mHadClientComposition ||
                getBE().mHwc-&gt;hasClientComposition(displayDevice-&gt;getHwcDisplayId());
    }
    mVsyncModulator.onRefreshed(mHadClientComposition);

    mLayersWithQueuedFrames.clear();
}</code></pre>

<h4 id="4-2-SF-preComposition"><a href="#4-2-SF-preComposition" class="headerlink" title="4.2 SF.preComposition"></a>4.2 SF.preComposition</h4>
<pre><code>
void SurfaceFlinger::preComposition(nsecs_t refreshStartTime)
{
    ATRACE_CALL();
    ALOGV("preComposition");

    bool needExtraInvalidate = false;
    mDrawingState.traverseInZOrder([&](Layer* layer) {
        //回调每一个图层的onPreComposition方法
        if (layer->onPreComposition(refreshStartTime)) {
            needExtraInvalidate = true;
        }
    });
    //当图层信息
    if (needExtraInvalidate) {
        signalLayerUpdate();
    }
}
</code></pre>

<h4 id="4-3-SF-rebuildLayerStacks"><a href="#4-3-SF-rebuildLayerStacks" class="headerlink" title="4.3 SF.rebuildLayerStacks"></a>4.3 SF.rebuildLayerStacks</h4>
<pre><code>
void SurfaceFlinger::rebuildLayerStacks() {
    ATRACE_CALL();
    ALOGV("rebuildLayerStacks");
    Mutex::Autolock lock(mDolphinStateLock);

    // rebuild the visible layer list per screen
    // 重建每个显示屏中可见的图层列表
    if (CC_UNLIKELY(mVisibleRegionsDirty)) {
        ATRACE_NAME("rebuildLayerStacks VR Dirty");
        mVisibleRegionsDirty = false;
        invalidateHwcGeometry();

        for (size_t dpy=0 ; dpy&lt;mDisplays.size() ; dpy++) {
            Region opaqueRegion;
            Region dirtyRegion;
            Vector&lt;sp&lt;Layer&gt;&gt; layersSortedByZ;
            Vector&lt;sp&lt;Layer&gt;&gt; layersNeedingFences;
            const sp&lt;DisplayDevice&gt;& displayDevice(mDisplays[dpy]);
            const Transform& tr(displayDevice-&gt;getTransform());
            const Rect bounds(displayDevice-&gt;getBounds());
            if (displayDevice-&gt;isDisplayOn()) {
                //计算每个layer的可见区域
                computeVisibleRegions(displayDevice, dirtyRegion, opaqueRegion);

                mDrawingState.traverseInZOrder([&](Layer* layer) {
                    bool hwcLayerDestroyed = false;
                    //LayerStack一样
                    if (layer-&gt;belongsToDisplay(displayDevice-&gt;getLayerStack(),
                                displayDevice-&gt;isPrimary())) {
                        Region drawRegion(tr.transform(
                                layer-&gt;visibleNonTransparentRegion));
                        drawRegion.andSelf(bounds);
                        if (!drawRegion.isEmpty()) {
                            layersSortedByZ.add(layer);
                        } else {
                            // Clear out the HWC layer if this layer was
                            // previously visible, but no longer is
                            hwcLayerDestroyed = layer-&gt;destroyHwcLayer(
                                    displayDevice-&gt;getHwcDisplayId());
                        }
                    } else {
                        // WM changes displayDevice-&gt;layerStack upon sleep/awake.
                        // Here we make sure we delete the HWC layers even if
                        // WM changed their layer stack.
                        hwcLayerDestroyed = layer-&gt;destroyHwcLayer(
                                displayDevice-&gt;getHwcDisplayId());
                    }

                    // If a layer is not going to get a release fence because
                    // it is invisible, but it is also going to release its
                    // old buffer, add it to the list of layers needing
                    // fences.
                    if (hwcLayerDestroyed) {
                        auto found = std::find(mLayersWithQueuedFrames.cbegin(),
                                mLayersWithQueuedFrames.cend(), layer);
                        if (found != mLayersWithQueuedFrames.cend()) {
                            layersNeedingFences.add(layer);
                        }
                    }
                });
            }
            displayDevice-&gt;setVisibleLayersSortedByZ(layersSortedByZ);
            displayDevice-&gt;setLayersNeedingFences(layersNeedingFences);
            displayDevice-&gt;undefinedRegion.set(bounds);
            displayDevice-&gt;undefinedRegion.subtractSelf(
                    tr.transform(opaqueRegion));
            displayDevice-&gt;dirtyRegion.orSelf(dirtyRegion);
        }
    }
}</code></pre>

<h4 id="4-4-SF-setUpHWComposer"><a href="#4-4-SF-setUpHWComposer" class="headerlink" title="4.4 SF.setUpHWComposer"></a>4.4 SF.setUpHWComposer</h4>
<pre><code>
void SurfaceFlinger::setUpHWComposer() {
    ATRACE_CALL();
    ALOGV("setUpHWComposer");

    for (size_t dpy=0 ; dpy&lt;mDisplays.size() ; dpy++) {
        bool dirty = !mDisplays[dpy]-&gt;getDirtyRegion(mRepaintEverything).isEmpty();
        bool empty = mDisplays[dpy]-&gt;getVisibleLayersSortedByZ().size() == 0;
        bool wasEmpty = !mDisplays[dpy]-&gt;lastCompositionHadVisibleLayers;

        // If nothing has changed (!dirty), don't recompose.
        // If something changed, but we don't currently have any visible layers,
        //   and didn't when we last did a composition, then skip it this time.
        // The second rule does two things:
        // - When all layers are removed from a display, we'll emit one black
        //   frame, then nothing more until we get new layers.
        // - When a display is created with a private layer stack, we won't
        //   emit any black frames until a layer is added to the layer stack.
        bool mustRecompose = dirty && !(empty && wasEmpty);

        ALOGV_IF(mDisplays[dpy]-&gt;getDisplayType() == DisplayDevice::DISPLAY_VIRTUAL,
                "dpy[%zu]: %s composition (%sdirty %sempty %swasEmpty)", dpy,
                mustRecompose ? "doing" : "skipping",
                dirty ? "+" : "-",
                empty ? "+" : "-",
                wasEmpty ? "+" : "-");

        mDisplays[dpy]-&gt;beginFrame(mustRecompose);

        if (mustRecompose) {
            mDisplays[dpy]-&gt;lastCompositionHadVisibleLayers = !empty;
        }
    }

    // build the h/w work list
    if (CC_UNLIKELY(mGeometryInvalid)) {
        mGeometryInvalid = false;
        for (size_t dpy=0 ; dpy&lt;mDisplays.size() ; dpy++) {
            sp&lt;const DisplayDevice&gt; displayDevice(mDisplays[dpy]);
            const auto hwcId = displayDevice-&gt;getHwcDisplayId();
            if (hwcId &gt;= 0) {
                const Vector&lt;sp&lt;Layer&gt;&gt;& currentLayers(
                        displayDevice-&gt;getVisibleLayersSortedByZ());
                setDisplayAnimating(displayDevice);
                for (size_t i = 0; i &lt; currentLayers.size(); i++) {
                    const auto& layer = currentLayers[i];
                    if (!layer-&gt;hasHwcLayer(hwcId)) {
                        if (!layer-&gt;createHwcLayer(getBE().mHwc.get(), hwcId)) {
                            layer-&gt;forceClientComposition(hwcId);
                            continue;
                        }
                        if (layer-&gt;isPrimaryDisplayOnly()) {
                            setLayerAsMask(hwcId, layer-&gt;getLayerId());
                        }
                    }

                    layer-&gt;setGeometry(displayDevice, i);
                    if (mDebugDisableHWC || mDebugRegion) {
                        layer-&gt;forceClientComposition(hwcId);
                    }
                }
            }
        }
    }

    // Set the per-frame data
    // 设置每一帧的数据
    for (size_t displayId = 0; displayId &lt; mDisplays.size(); ++displayId) {
        auto& displayDevice = mDisplays[displayId];
        const auto hwcId = displayDevice-&gt;getHwcDisplayId();

        if (hwcId &lt; 0) {
            continue;
        }
        if (mDrawingState.colorMatrixChanged) {
            displayDevice-&gt;setColorTransform(mDrawingState.colorMatrix);
            status_t result = getBE().mHwc-&gt;setColorTransform(hwcId, mDrawingState.colorMatrix);
            ALOGE_IF(result != NO_ERROR, "Failed to set color transform on "
                    "display %zd: %d", displayId, result);
        }
        for (auto& layer : displayDevice-&gt;getVisibleLayersSortedByZ()) {
            if (layer-&gt;isHdrY410()) {
                layer-&gt;forceClientComposition(hwcId);
            } else if ((layer-&gt;getDataSpace() == Dataspace::BT2020_PQ ||
                        layer-&gt;getDataSpace() == Dataspace::BT2020_ITU_PQ) &&
                    !displayDevice-&gt;hasHDR10Support()) {
                layer-&gt;forceClientComposition(hwcId);
            } else if ((layer-&gt;getDataSpace() == Dataspace::BT2020_HLG ||
                        layer-&gt;getDataSpace() == Dataspace::BT2020_ITU_HLG) &&
                    !displayDevice-&gt;hasHLGSupport()) {
                layer-&gt;forceClientComposition(hwcId);
            }

            if (layer-&gt;getForceClientComposition(hwcId)) {
                ALOGV("[%s] Requesting Client composition", layer-&gt;getName().string());
                layer-&gt;setCompositionType(hwcId, HWC2::Composition::Client);
                continue;
            }

            layer-&gt;setPerFrameData(displayDevice);
        }

        if (hasWideColorDisplay) {
            ColorMode colorMode;
            Dataspace dataSpace;
            RenderIntent renderIntent;
            pickColorMode(displayDevice, &colorMode, &dataSpace, &renderIntent);
            setActiveColorModeInternal(displayDevice, colorMode, dataSpace, renderIntent);
        }
    }

    mDrawingState.colorMatrixChanged = false;

    dumpDrawCycle(true);

    for (size_t displayId = 0; displayId &lt; mDisplays.size(); ++displayId) {
        auto& displayDevice = mDisplays[displayId];
        if (!displayDevice-&gt;isDisplayOn()) {
            continue;
        }

        status_t result = displayDevice-&gt;prepareFrame(*getBE().mHwc);
        ALOGE_IF(result != NO_ERROR, "prepareFrame for display %zd failed:"
                " %d (%s)", displayId, result, strerror(-result));
    }
}</code></pre>

<h4 id="4-5-SF-doComposition"><a href="#4-5-SF-doComposition" class="headerlink" title="4.5 SF.doComposition"></a>4.5 SF.doComposition</h4>
<pre><code>
void SurfaceFlinger::doComposition() {
    ATRACE_CALL();
    ALOGV("doComposition");

    const bool repaintEverything = android_atomic_and(0, &mRepaintEverything);
    for (size_t dpy=0 ; dpy&lt;mDisplays.size() ; dpy++) {
        const sp&lt;DisplayDevice&gt;& hw(mDisplays[dpy]);
        if (hw-&gt;isDisplayOn()) {
            // transform the dirty region into this screen's coordinate space
            // 将脏区域转换为此屏幕的坐标空间
            const Region dirtyRegion(hw-&gt;getDirtyRegion(repaintEverything));

            // repaint the framebuffer (if needed)
            // 如果需要，重绘framebuffer
            doDisplayComposition(hw, dirtyRegion);

            hw-&gt;dirtyRegion.clear();
            hw-&gt;flip();
        }
    }
    postFramebuffer();
}
</code></pre>

<h5 id="4-5-1-doDisplayComposition"><a href="#4-5-1-doDisplayComposition" class="headerlink" title="4.5.1 doDisplayComposition"></a>4.5.1 doDisplayComposition</h5>
<pre><code>
void SurfaceFlinger::doDisplayComposition(
        const sp&lt;const DisplayDevice&gt;& displayDevice,
        const Region& inDirtyRegion)
{
    // We only need to actually compose the display if:
    // 1) It is being handled by hardware composer, which may need this to
    //    keep its virtual display state machine in sync, or
    // 2) There is work to be done (the dirty region isn't empty)
    bool isHwcDisplay = displayDevice-&gt;getHwcDisplayId() &gt;= 0;
    if (!isHwcDisplay && inDirtyRegion.isEmpty()) {
        ALOGV("Skipping display composition");
        return;
    }

    ALOGV("doDisplayComposition");
    if (!doComposeSurfaces(displayDevice)) return;

    // swap buffers (presentation)
    // 交换buffer,输出图像
    displayDevice-&gt;swapBuffers(getHwComposer());
}</code></pre>

<h5 id="4-5-2-doComposeSurfaces"><a href="#4-5-2-doComposeSurfaces" class="headerlink" title="4.5.2 doComposeSurfaces"></a>4.5.2 doComposeSurfaces</h5>
<pre><code>
bool SurfaceFlinger::doComposeSurfaces(const sp&lt;const DisplayDevice&gt;& displayDevice)
{
    ALOGV("doComposeSurfaces");

    const Region bounds(displayDevice-&gt;bounds());
    const DisplayRenderArea renderArea(displayDevice);
    const auto hwcId = displayDevice-&gt;getHwcDisplayId();
    const bool hasClientComposition = getBE().mHwc-&gt;hasClientComposition(hwcId);
    ATRACE_INT("hasClientComposition", hasClientComposition);

    bool applyColorMatrix = false;
    bool needsEnhancedColorMatrix = false;

    if (hasClientComposition) {
        ALOGV("hasClientComposition");

        Dataspace outputDataspace = Dataspace::UNKNOWN;
        if (displayDevice-&gt;hasWideColorGamut()) {
            outputDataspace = displayDevice-&gt;getCompositionDataSpace();
        }
        getBE().mRenderEngine-&gt;setOutputDataSpace(outputDataspace);
        getBE().mRenderEngine-&gt;setDisplayMaxLuminance(
                displayDevice-&gt;getHdrCapabilities().getDesiredMaxLuminance());

        const bool hasDeviceComposition = getBE().mHwc-&gt;hasDeviceComposition(hwcId);
        const bool skipClientColorTransform = getBE().mHwc-&gt;hasCapability(
            HWC2::Capability::SkipClientColorTransform);

        mat4 colorMatrix;
        applyColorMatrix = !hasDeviceComposition && !skipClientColorTransform;
        if (applyColorMatrix) {
            colorMatrix = mDrawingState.colorMatrix;
        }

        // The current enhanced saturation matrix is designed to enhance Display P3,
        // thus we only apply this matrix when the render intent is not colorimetric
        // and the output color space is Display P3.
        needsEnhancedColorMatrix =
            (displayDevice-&gt;getActiveRenderIntent() &gt;= RenderIntent::ENHANCE &&
             outputDataspace == Dataspace::DISPLAY_P3);
        if (needsEnhancedColorMatrix) {
            colorMatrix *= mEnhancedSaturationMatrix;
        }

        getRenderEngine().setupColorTransform(colorMatrix);

        if (!displayDevice-&gt;makeCurrent()) {
            ALOGW("DisplayDevice::makeCurrent failed. Aborting surface composition for display %s",
                  displayDevice-&gt;getDisplayName().string());
            getRenderEngine().resetCurrentSurface();

            // |mStateLock| not needed as we are on the main thread
            if(!getDefaultDisplayDeviceLocked()-&gt;makeCurrent()) {
              ALOGE("DisplayDevice::makeCurrent on default display failed. Aborting.");
            }
            return false;
        }

        // Never touch the framebuffer if we don't have any framebuffer layers
        // 如果没有framebuffer的layer层，则不需要改变framebuffer
        if (hasDeviceComposition) {
            // when using overlays, we assume a fully transparent framebuffer
            // NOTE: we could reduce how much we need to clear, for instance
            // remove where there are opaque FB layers. however, on some
            // GPUs doing a "clean slate" clear might be more efficient.
            // We'll revisit later if needed.
            getBE().mRenderEngine-&gt;clearWithColor(0, 0, 0, 0);
        } else {
            // we start with the whole screen area and remove the scissor part
            // we're left with the letterbox region
            // (common case is that letterbox ends-up being empty)
            const Region letterbox(bounds.subtract(displayDevice-&gt;getScissor()));

            // compute the area to clear
            Region region(displayDevice-&gt;undefinedRegion.merge(letterbox));

            // screen is already cleared here
            if (!region.isEmpty()) {
                // can happen with SurfaceView
                drawWormhole(displayDevice, region);
            }
        }

        const Rect& bounds(displayDevice-&gt;getBounds());
        const Rect& scissor(displayDevice-&gt;getScissor());
        if (scissor != bounds) {
            // scissor doesn't match the screen's dimensions, so we
            // need to clear everything outside of it and enable
            // the GL scissor so we don't draw anything where we shouldn't

            // enable scissor for this frame
            const uint32_t height = displayDevice-&gt;getHeight();
            getBE().mRenderEngine-&gt;setScissor(scissor.left, height - scissor.bottom,
                                              scissor.getWidth(), scissor.getHeight());
        }
    }

    /*
     * and then, render the layers targeted at the framebuffer
     */

    ALOGV("Rendering client layers");
    const Transform& displayTransform = displayDevice-&gt;getTransform();
    bool firstLayer = true;
    for (auto& layer : displayDevice-&gt;getVisibleLayersSortedByZ()) {
        const Region clip(bounds.intersect(
                displayTransform.transform(layer-&gt;visibleRegion)));
        ALOGV("Layer: %s", layer-&gt;getName().string());
        ALOGV("  Composition type: %s",
                to_string(layer-&gt;getCompositionType(hwcId)).c_str());
        if (!clip.isEmpty()) {
            switch (layer-&gt;getCompositionType(hwcId)) {
                case HWC2::Composition::Cursor:
                case HWC2::Composition::Device:
                case HWC2::Composition::Sideband:
                case HWC2::Composition::SolidColor: {
                    const Layer::State& state(layer-&gt;getDrawingState());
                    if (layer-&gt;getClearClientTarget(hwcId) && !firstLayer &&
                            layer-&gt;isOpaque(state) && (layer-&gt;getAlpha() == 1.0f)
                            && hasClientComposition) {
                        // never clear the very first layer since we're
                        // guaranteed the FB is already cleared
                        layer-&gt;clearWithOpenGL(renderArea);
                    }
                    break;
                }
                case HWC2::Composition::Client: {
                    if ((hwcId &lt; 0) &&
                        (DisplayUtils::getInstance()-&gt;skipColorLayer(layer-&gt;getTypeId()))) {
                        // We are not using h/w composer.
                        // Skip color (dim) layer for WFD direct streaming.
                        continue;
                    }
                    layer-&gt;draw(renderArea, clip);
                    break;
                }
                default:
                    break;
            }
        } else {
            ALOGV("  Skipping for empty clip");
        }
        firstLayer = false;
    }

    if (applyColorMatrix || needsEnhancedColorMatrix) {
        getRenderEngine().setupColorTransform(mat4());
    }

    // disable scissor at the end of the frame
    getBE().mRenderEngine-&gt;disableScissor();
    return true;
}</code></pre>

<h4 id="4-6-SF-postComposition"><a href="#4-6-SF-postComposition" class="headerlink" title="4.6 SF.postComposition"></a>4.6 SF.postComposition</h4>
<pre><code>
void SurfaceFlinger::postComposition(nsecs_t refreshStartTime)
{
    ATRACE_CALL();
    ALOGV("postComposition");

    // Release any buffers which were replaced this frame
    nsecs_t dequeueReadyTime = systemTime();
    for (auto& layer : mLayersWithQueuedFrames) {
        layer-&gt;releasePendingBuffer(dequeueReadyTime);
    }

    // |mStateLock| not needed as we are on the main thread
    const sp&lt;const DisplayDevice&gt; hw(getDefaultDisplayDeviceLocked());

    getBE().mGlCompositionDoneTimeline.updateSignalTimes();
    std::shared_ptr&lt;FenceTime&gt; glCompositionDoneFenceTime;
    if (hw && getBE().mHwc-&gt;hasClientComposition(HWC_DISPLAY_PRIMARY)) {
        glCompositionDoneFenceTime =
                std::make_shared&lt;FenceTime&gt;(hw-&gt;getClientTargetAcquireFence());
        getBE().mGlCompositionDoneTimeline.push(glCompositionDoneFenceTime);
    } else {
        glCompositionDoneFenceTime = FenceTime::NO_FENCE;
    }

    getBE().mDisplayTimeline.updateSignalTimes();

    int disp = getVsyncSource();
    sp&lt;Fence&gt; presentFence = (disp != -1) ? getBE().mHwc-&gt;getPresentFence(disp) : Fence::NO_FENCE;
    auto presentFenceTime = std::make_shared&lt;FenceTime&gt;(presentFence);
    getBE().mDisplayTimeline.push(presentFenceTime);

    nsecs_t vsyncPhase = mPrimaryDispSync.computeNextRefresh(0);
    nsecs_t vsyncInterval = mPrimaryDispSync.getPeriod();

    // We use the refreshStartTime which might be sampled a little later than
    // when we started doing work for this frame, but that should be okay
    // since updateCompositorTiming has snapping logic.
    updateCompositorTiming(
        vsyncPhase, vsyncInterval, refreshStartTime, presentFenceTime);
    CompositorTiming compositorTiming;
    {
        std::lock_guard&lt;std::mutex&gt; lock(getBE().mCompositorTimingLock);
        compositorTiming = getBE().mCompositorTiming;
    }

    mDrawingState.traverseInZOrder([&](Layer* layer) {
        bool frameLatched = layer-&gt;onPostComposition(glCompositionDoneFenceTime,
                presentFenceTime, compositorTiming);
        if (frameLatched) {
            recordBufferingStats(layer-&gt;getName().string(),
                    layer-&gt;getOccupancyHistory(false));
        }
    });

    if (presentFenceTime-&gt;isValid()) {
        if (mPrimaryDispSync.addPresentFence(presentFenceTime)) {
            enableHardwareVsync();
        } else {
            disableHardwareVsync(false);
        }
    }

    forceResyncModel();
    if (!hasSyncFramework) {
        if (getBE().mHwc-&gt;isConnected(HWC_DISPLAY_PRIMARY) && hw-&gt;isDisplayOn()) {
            enableHardwareVsync();
        }
    }

    if (mAnimCompositionPending) {
        mAnimCompositionPending = false;

        if (presentFenceTime-&gt;isValid()) {
            mAnimFrameTracker.setActualPresentFence(
                    std::move(presentFenceTime));
        } else if (getBE().mHwc-&gt;isConnected(HWC_DISPLAY_PRIMARY)) {
            // The HWC doesn't support present fences, so use the refresh
            // timestamp instead.
            nsecs_t presentTime =
                    getBE().mHwc-&gt;getRefreshTimestamp(HWC_DISPLAY_PRIMARY);
            mAnimFrameTracker.setActualPresentTime(presentTime);
        }
        mAnimFrameTracker.advanceFrame();
    }

    dumpDrawCycle(false);

    mTimeStats.incrementTotalFrames();
    if (mHadClientComposition) {
        mTimeStats.incrementClientCompositionFrames();
    }

    if (getBE().mHwc-&gt;isConnected(HWC_DISPLAY_PRIMARY) &&
            hw-&gt;getPowerMode() == HWC_POWER_MODE_OFF) {
        return;
    }

    nsecs_t currentTime = systemTime();
    if (mHasPoweredOff) {
        mHasPoweredOff = false;
    } else {
        nsecs_t elapsedTime = currentTime - getBE().mLastSwapTime;
        size_t numPeriods = static_cast&lt;size_t&gt;(elapsedTime / vsyncInterval);
        if (numPeriods &lt; SurfaceFlingerBE::NUM_BUCKETS - 1) {
            getBE().mFrameBuckets[numPeriods] += elapsedTime;
        } else {
            getBE().mFrameBuckets[SurfaceFlingerBE::NUM_BUCKETS - 1] += elapsedTime;
        }
        getBE().mTotalTime += elapsedTime;
    }
    getBE().mLastSwapTime = currentTime;

    {
        std::lock_guard lock(mTexturePoolMutex);
        const size_t refillCount = mTexturePoolSize - mTexturePool.size();
        if (refillCount &gt; 0) {
            const size_t offset = mTexturePool.size();
            mTexturePool.resize(mTexturePoolSize);
            getRenderEngine().genTextures(refillCount, mTexturePool.data() + offset);
            ATRACE_INT("TexturePoolSize", mTexturePool.size());
        }
    }
}</code></pre>

<h3 id="五、总结"><a href="#五、总结" class="headerlink" title="五、总结"></a>五、总结</h3><p>本文主要介绍了SurfaceFlinger和绘制相关的流程，首先介绍了SurfcaeFlinger的启动过程，后面分析Vsync信号如何进行通知屏幕的绘制，最后分析了屏幕的绘制图像输出流程。</p>
<p>这三个过程主要的线程如下：</p>
<ul>
<li>主线程“/system/bin/surfaceflinger”: 主线程</li>
<li>线程“EventThread”：EventThread</li>
<li>线程“EventControl”： EventControlThread</li>
<li>线程“DispSync”：DispSyncThread</li>
</ul>
<p><strong>SurfcaeFlinger的启动过程</strong></p>
<p>1.启动图形处理服务</p>
<p>2.开启线程池，最大binder线程池数的个数为4</p>
<p>3.设置SurfaceFlinger进程为高优先级以及后台调度策略</p>
<p>4.创建SurfaceFlinger，并初始化，启动app和sf的两个EventThread线程</p>
<p>5.注册SurfaceFlinger服务和GpuService</p>
<p>6.启动显示服务，最后执行surfacefinger的run方法</p>
<p><strong>Vsync信号处理</strong></p>
<p>1.如果要接收Vsync信号，则必须先注册，当Vsync信号来时会调用onVsyncReceived方法</p>
<p>2.通过调用DispSyncThread.updateModel中的mCond.signal() 来唤醒DispSyncThread线程；</p>
<p>3.执行EventThread::onVSyncEvent()方法中调用 mCondition.notify_all() 唤醒EventThread线程；</p>
<p>4.在DisplayEventReceiver.sendEvents调用BitTube::sendObjects，当收到数据时会执行MQ.cb_eventReceiver，然后通过handler消息机制进入到SurfaceFlinger主线程，调用SF.onMessageReceived方法</p>
<p><strong>图形输出</strong></p>
<p>1.根据上次绘制的图层是否有更新，来判断是否执行invalidate过程</p>
<p>2.重新每个显示屏所有可见点Layer列表</p>
<p>3.更新HWComposer图层</p>
<p>4.合成所有Layer的图像</p>
<p>5.回调每个Layer的onPostComposition</p>
