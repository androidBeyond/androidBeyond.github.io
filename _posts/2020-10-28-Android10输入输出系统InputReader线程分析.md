---
layout:     post
title:      Android10 输入输出系统InputReader线程分析
subtitle:   本篇博文我们来学习下InputReader线程的业务逻辑过程
date:       2020-10-28
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android10
    - Android
    - Input系统
---


<h2 id="一-inputreader起点">一. InputReader起点</h2>

<p>上一篇文章我们介绍过IMS服务的启动过程会创建两个native线程，分别是InputReader,InputDispatcher.
接下来从InputReader线程的执行过程从threadLoop为起点开始分析。</p>

<h4 id="11-threadloop">1.1 threadLoop</h4>
<p>[-&gt; InputReader.cpp]</p>

<div ><div ><pre ><code>bool InputReaderThread::threadLoop() {
    mReader-&gt;loopOnce(); //【见小节1.2】
    return true;
}
</code></pre></div></div>

<p>threadLoop返回值true代表的是会不断地循环调用loopOnce()。另外，如果当返回值为false则会
退出循环。整个过程是不断循环的地调用InputReader的loopOnce()方法，先来回顾一下InputReader对象构造方法。</p>

<h4 id="12-looponce">1.2 loopOnce</h4>
<p>[-&gt; InputReader.cpp]</p>

<div ><div ><pre ><code>void InputReader::loopOnce() {
    ...
    {
        AutoMutex _l(mLock);
        uint32_t changes = mConfigurationChangesToRefresh;
        if (changes) {
            timeoutMillis = 0;
            ...
        } else if (mNextTimeout != LLONG_MAX) {
            nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
            timeoutMillis = toMillisecondTimeoutDelay(now, mNextTimeout);
        }
    }

    //从EventHub读取事件，其中EVENT_BUFFER_SIZE = 256【见小节2.1】
    size_t count = mEventHub-&gt;getEvents(timeoutMillis, mEventBuffer, EVENT_BUFFER_SIZE);

    { // acquire lock
        AutoMutex _l(mLock);
         mReaderIsAliveCondition.broadcast();
        if (count) { //处理事件【见小节3.1】
            processEventsLocked(mEventBuffer, count);
        }
        if (oldGeneration != mGeneration) {
            inputDevicesChanged = true;
            getInputDevicesLocked(inputDevices);
        }
        ...
    } // release lock


    if (inputDevicesChanged) { //输入设备发生改变
        mPolicy-&gt;notifyInputDevicesChanged(inputDevices);
    }
    //发送事件到nputDispatcher【见小节4.1】
    mQueuedListener-&gt;flush();
}
</code></pre></div></div>

<h2 id="二-eventhub">二. EventHub</h2>

<h3 id="21-getevents">2.1 getEvents</h3>
<p>[-&gt; EventHub.cpp]</p>

<div ><div ><pre ><code>size_t EventHub::getEvents(int timeoutMillis, RawEvent* buffer, size_t bufferSize) {
    AutoMutex _l(mLock); //加锁

    struct input_event readBuffer[bufferSize];
    RawEvent* event = buffer; //原始事件
    size_t capacity = bufferSize; //容量大小为256
    bool awoken = false;
    for (;;) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        ...

        if (mNeedToScanDevices) {
            mNeedToScanDevices = false;
            scanDevicesLocked(); //扫描设备【见小节2.2】
            mNeedToSendFinishedDeviceScan = true;
        }

        while (mOpeningDevices != NULL) {
            Device* device = mOpeningDevices;
            mOpeningDevices = device-&gt;next;
            event-&gt;when = now;
            event-&gt;deviceId = device-&gt;id == mBuiltInKeyboardId ? 0 : device-&gt;id;
            event-&gt;type = DEVICE_ADDED; //添加设备的事件
            event += 1;
            mNeedToSendFinishedDeviceScan = true;
            if (--capacity == 0) {
                break;
            }
        }
        ...

        bool deviceChanged = false;
        while (mPendingEventIndex &lt; mPendingEventCount) {
            //从mPendingEventItems读取事件项
            const struct epoll_event&amp; eventItem = mPendingEventItems[mPendingEventIndex++];
            ...
            //获取设备ID所对应的device
            ssize_t deviceIndex = mDevices.indexOfKey(eventItem.data.u32);
            Device* device = mDevices.valueAt(deviceIndex);
            if (eventItem.events &amp; EPOLLIN) {
                //从设备不断读取事件，放入到readBuffer
                int32_t readSize = read(device-&gt;fd, readBuffer,
                        sizeof(struct input_event) * capacity);

                if (readSize == 0 || (readSize &lt; 0 &amp;&amp; errno == ENODEV)) {
                    deviceChanged = true;
                    closeDeviceLocked(device);//设备已被移除则执行关闭操作
                } else if (readSize &lt; 0) {
                    ...
                } else if ((readSize % sizeof(struct input_event)) != 0) {
                    ...
                } else {
                    int32_t deviceId = device-&gt;id == mBuiltInKeyboardId ? 0 : device-&gt;id;
                    size_t count = size_t(readSize) / sizeof(struct input_event);

                    for (size_t i = 0; i &lt; count; i++) {
                        //获取readBuffer的数据
                        struct input_event&amp; iev = readBuffer[i];
                        //将input_event信息, 封装成RawEvent
                        event-&gt;when = nsecs_t(iev.time.tv_sec) * 1000000000LL
                                + nsecs_t(iev.time.tv_usec) * 1000LL;
                        event-&gt;deviceId = deviceId;
                        event-&gt;type = iev.type;
                        event-&gt;code = iev.code;
                        event-&gt;value = iev.value;
                        event += 1;
                        capacity -= 1;
                    }
                    if (capacity == 0) {
                        mPendingEventIndex -= 1;
                        break;
                    }
                }
            }
            ...
        }
        ...
        mLock.unlock(); //poll之前先释放锁
        //等待input事件的到来
        int pollResult = epoll_wait(mEpollFd, mPendingEventItems, EPOLL_MAX_EVENTS, timeoutMillis);
        ...
        mLock.lock(); //poll之后再次请求锁

        if (pollResult &lt; 0) { //出现错误
            mPendingEventCount = 0;
            if (errno != EINTR) {
                usleep(100000); //系统发生错误则休眠1s
            }
        } else {
            mPendingEventCount = size_t(pollResult);
        }
    }

    return event - buffer; //返回所读取的事件个数
}
</code></pre></div></div>

<p>EventHub采用INotify + epoll机制实现监听目录<code >/dev/input</code>下的设备节点，经过EventHub将input_event结构体 + deviceId 转换成RawEvent结构体，如下：</p>

<h4 id="211-rawevent">2.1.1 RawEvent</h4>
<p>[-&gt; InputEventReader.h]</p>

<div ><div ><pre ><code>struct input_event {
 struct timeval time; //事件发生的时间点
 __u16 type;
 __u16 code;
 __s32 value;
};

struct RawEvent {
    nsecs_t when; //事件发生的时间店
    int32_t deviceId; //产生事件的设备Id
    int32_t type; // 事件类型
    int32_t code;
    int32_t value;
};
</code></pre></div></div>

<p>此处事件类型:</p>

<ul>
  <li>DEVICE_ADDED(添加)</li>
  <li>DEVICE_REMOVED(删除)</li>
  <li>FINISHED_DEVICE_SCAN(扫描完成)</li>
  <li>type&lt;FIRST_SYNTHETIC_EVENT(其他事件)</li>
</ul>

<p>getEvents()已完成转换事件转换工作, 接下来,顺便看看设备扫描过程.</p>

<h3 id="22-设备扫描">2.2 设备扫描</h3>

<h4 id="221-scandeviceslocked">2.2.1 scanDevicesLocked</h4>

<div ><div ><pre ><code>void EventHub::scanDevicesLocked() {
    //此处DEVICE_PATH="/dev/input"【见小节2.3】
    status_t res = scanDirLocked(DEVICE_PATH);
    ...
}
</code></pre></div></div>

<h4 id="222-scandirlocked">2.2.2 scanDirLocked</h4>

<div ><div ><pre ><code>status_t EventHub::scanDirLocked(const char *dirname)
{
    char devname[PATH_MAX];
    char *filename;
    DIR *dir;
    struct dirent *de;
    dir = opendir(dirname);

    strcpy(devname, dirname);
    filename = devname + strlen(devname);
    *filename++ = '/';
    //读取/dev/input/目录下所有的设备节点
    while((de = readdir(dir))) {
        if(de-&gt;d_name[0] == '.' &amp;&amp;
           (de-&gt;d_name[1] == '\0' ||
            (de-&gt;d_name[1] == '.' &amp;&amp; de-&gt;d_name[2] == '\0')))
            continue;
        strcpy(filename, de-&gt;d_name);
        //打开相应的设备节点【2.2.3】
        openDeviceLocked(devname);
    }
    closedir(dir);
    return 0;
}
</code></pre></div></div>

<h4 id="223-opendevicelocked">2.2.3 openDeviceLocked</h4>

<div ><div ><pre ><code>status_t EventHub::openDeviceLocked(const char *devicePath) {
    char buffer[80];
    //打开设备文件
    int fd = open(devicePath, O_RDWR | O_CLOEXEC);
    InputDeviceIdentifier identifier;
    //获取设备名
    if(ioctl(fd, EVIOCGNAME(sizeof(buffer) - 1), &amp;buffer) &lt; 1){
    } else {
        buffer[sizeof(buffer) - 1] = '\0';
        identifier.name.setTo(buffer);
    }

    identifier.bus = inputId.bustype;
    identifier.product = inputId.product;
    identifier.vendor = inputId.vendor;
    identifier.version = inputId.version;

    //获取设备物理地址
    if(ioctl(fd, EVIOCGPHYS(sizeof(buffer) - 1), &amp;buffer) &lt; 1) {
    } else {
        buffer[sizeof(buffer) - 1] = '\0';
        identifier.location.setTo(buffer);
    }

    //获取设备唯一ID
    if(ioctl(fd, EVIOCGUNIQ(sizeof(buffer) - 1), &amp;buffer) &lt; 1) {
    } else {
        buffer[sizeof(buffer) - 1] = '\0';
        identifier.uniqueId.setTo(buffer);
    }
    //将identifier信息填充到fd
    assignDescriptorLocked(identifier);
    //设置fd为非阻塞方式
    fcntl(fd, F_SETFL, O_NONBLOCK);

    //获取设备ID，分配设备对象内存
    int32_t deviceId = mNextDeviceId++;
    Device* device = new Device(fd, deviceId, String8(devicePath), identifier);
    ...

    //注册epoll
    struct epoll_event eventItem;
    memset(&amp;eventItem, 0, sizeof(eventItem));
    eventItem.events = EPOLLIN;
    if (mUsingEpollWakeup) {
        eventItem.events |= EPOLLWAKEUP;
    }
    eventItem.data.u32 = deviceId;
    if (epoll_ctl(mEpollFd, EPOLL_CTL_ADD, fd, &amp;eventItem)) {
        delete device; //添加失败则删除该设备
        return -1;
    }
    ...
    //【见小节2.2.4】
    addDeviceLocked(device);
}
</code></pre></div></div>

<h4 id="224-adddevicelocked">2.2.4 addDeviceLocked</h4>

<div ><div ><pre ><code>void EventHub::addDeviceLocked(Device* device) {
    mDevices.add(device-&gt;id, device); //添加到mDevices队列
    device-&gt;next = mOpeningDevices;
    mOpeningDevices = device;
}
</code></pre></div></div>

<p>介绍了EventHub从设备节点获取事件的流程，当收到事件后接下里便开始处理事件。</p>

<h2 id="三-inputreader">三. InputReader</h2>

<h3 id="31-processeventslocked">3.1 processEventsLocked</h3>
<p>[-&gt; InputReader.cpp]</p>

<div ><div ><pre ><code>void InputReader::processEventsLocked(const RawEvent* rawEvents, size_t count) {
    for (const RawEvent* rawEvent = rawEvents; count;) {
        int32_t type = rawEvent-&gt;type;
        size_t batchSize = 1;
        if (type &lt; EventHubInterface::FIRST_SYNTHETIC_EVENT) {
            int32_t deviceId = rawEvent-&gt;deviceId;
            while (batchSize &lt; count) {
                if (rawEvent[batchSize].type &gt;= EventHubInterface::FIRST_SYNTHETIC_EVENT
                        || rawEvent[batchSize].deviceId != deviceId) {
                    break;
                }
                batchSize += 1; //同一设备的事件打包处理
            }
            //数据事件的处理【见小节3.3】
            processEventsForDeviceLocked(deviceId, rawEvent, batchSize);
        } else {
            switch (rawEvent-&gt;type) {
            case EventHubInterface::DEVICE_ADDED:
                //设备添加【见小节3.2】
                addDeviceLocked(rawEvent-&gt;when, rawEvent-&gt;deviceId);
                break;
            case EventHubInterface::DEVICE_REMOVED:
                //设备移除
                removeDeviceLocked(rawEvent-&gt;when, rawEvent-&gt;deviceId);
                break;
            case EventHubInterface::FINISHED_DEVICE_SCAN:
                //设备扫描完成
                handleConfigurationChangedLocked(rawEvent-&gt;when);
                break;
            default:
                ALOG_ASSERT(false);//不会发生
                break;
            }
        }
        count -= batchSize;
        rawEvent += batchSize;
    }
}
</code></pre></div></div>

<p>事件处理总共有下几类类型：</p>

<ul>
  <li>DEVICE_ADDED(设备增加), [见小节3.2]</li>
  <li>DEVICE_REMOVED(设备移除)</li>
  <li>FINISHED_DEVICE_SCAN(设备扫描完成)</li>
  <li>数据事件[见小节3.4]</li>
</ul>

<p>先来说说DEVICE_ADDED设备增加的过程。</p>

<h3 id="32-设备增加">3.2 设备增加</h3>

<h4 id="321-adddevicelocked">3.2.1 addDeviceLocked</h4>

<div ><div ><pre ><code>void InputReader::addDeviceLocked(nsecs_t when, int32_t deviceId) {
    ssize_t deviceIndex = mDevices.indexOfKey(deviceId);
    if (deviceIndex &gt;= 0) {
        return; //已添加的相同设备则不再添加
    }

    InputDeviceIdentifier identifier = mEventHub-&gt;getDeviceIdentifier(deviceId);
    uint32_t classes = mEventHub-&gt;getDeviceClasses(deviceId);
    int32_t controllerNumber = mEventHub-&gt;getDeviceControllerNumber(deviceId);
    //【见小节3.2.2】
    InputDevice* device = createDeviceLocked(deviceId, controllerNumber, identifier, classes);
    device-&gt;configure(when, &amp;mConfig, 0);
    device-&gt;reset(when);
    mDevices.add(deviceId, device); //添加设备到mDevices
    ...
}
</code></pre></div></div>

<h4 id="322-createdevicelocked">3.2.2 createDeviceLocked</h4>

<div ><div ><pre ><code>InputDevice* InputReader::createDeviceLocked(int32_t deviceId, int32_t controllerNumber,
        const InputDeviceIdentifier&amp; identifier, uint32_t classes) {
    //创建InputDevice对象
    InputDevice* device = new InputDevice(&amp;mContext, deviceId, bumpGenerationLocked(),
            controllerNumber, identifier, classes);
    ...

    //获取键盘源类型
    uint32_t keyboardSource = 0;
    int32_t keyboardType = AINPUT_KEYBOARD_TYPE_NON_ALPHABETIC;
    if (classes &amp; INPUT_DEVICE_CLASS_KEYBOARD) {
        keyboardSource |= AINPUT_SOURCE_KEYBOARD;
    }
    if (classes &amp; INPUT_DEVICE_CLASS_ALPHAKEY) {
        keyboardType = AINPUT_KEYBOARD_TYPE_ALPHABETIC;
    }
    if (classes &amp; INPUT_DEVICE_CLASS_DPAD) {
        keyboardSource |= AINPUT_SOURCE_DPAD;
    }
    if (classes &amp; INPUT_DEVICE_CLASS_GAMEPAD) {
        keyboardSource |= AINPUT_SOURCE_GAMEPAD;
    }

    //添加键盘类设备InputMapper
    if (keyboardSource != 0) {
        device-&gt;addMapper(new KeyboardInputMapper(device, keyboardSource, keyboardType));
    }

    //添加鼠标类设备InputMapper
    if (classes &amp; INPUT_DEVICE_CLASS_CURSOR) {
        device-&gt;addMapper(new CursorInputMapper(device));
    }

    //添加触摸屏设备InputMapper
    if (classes &amp; INPUT_DEVICE_CLASS_TOUCH_MT) {
        device-&gt;addMapper(new MultiTouchInputMapper(device));
    } else if (classes &amp; INPUT_DEVICE_CLASS_TOUCH) {
        device-&gt;addMapper(new SingleTouchInputMapper(device));
    }
    ...
    return device;
}
</code></pre></div></div>

<p>该方法主要功能：</p>

<ul>
  <li>创建InputDevice对象，将InputReader的mContext赋给InputDevice对象所对应的变量</li>
  <li>根据设备类型来创建并添加相对应的InputMapper，同时设置mContext.</li>
</ul>

<p>input设备类型有很多种，以上代码只列举部分常见的设备以及相应的InputMapper：</p>

<ul>
  <li>键盘类设备：KeyboardInputMapper</li>
  <li>触摸屏设备：MultiTouchInputMapper或SingleTouchInputMapper</li>
  <li>鼠标类设备：CursorInputMapper</li>
</ul>

<p>介绍完设备增加过程，继续回到[小节3.1]除了设备的增删，更常见事件便是数据事件，那么接下来介绍数据事件的
处理过程。</p>

<h3 id="33-事件处理">3.3 事件处理</h3>

<h4 id="331-processeventsfordevicelocked">3.3.1 processEventsForDeviceLocked</h4>

<div ><div ><pre ><code>void InputReader::processEventsForDeviceLocked(int32_t deviceId,
        const RawEvent* rawEvents, size_t count) {
    ssize_t deviceIndex = mDevices.indexOfKey(deviceId);
    ...

    InputDevice* device = mDevices.valueAt(deviceIndex);
    if (device-&gt;isIgnored()) {
        return; //可忽略则直接返回
    }
    //【见小节3.3.2】
    device-&gt;process(rawEvents, count);
}
</code></pre></div></div>

<h4 id="332-inputdeviceprocess">3.3.2 InputDevice.process</h4>

<div ><div ><pre ><code>void InputDevice::process(const RawEvent* rawEvents, size_t count) {
    size_t numMappers = mMappers.size();
    for (const RawEvent* rawEvent = rawEvents; count--; rawEvent++) {
        if (mDropUntilNextSync) {
            if (rawEvent-&gt;type == EV_SYN &amp;&amp; rawEvent-&gt;code == SYN_REPORT) {
                mDropUntilNextSync = false;
            }
        } else if (rawEvent-&gt;type == EV_SYN &amp;&amp; rawEvent-&gt;code == SYN_DROPPED) {
            mDropUntilNextSync = true;
            reset(rawEvent-&gt;when);
        } else {
            for (size_t i = 0; i &lt; numMappers; i++) {
                InputMapper* mapper = mMappers[i];
                //调用具体mapper来处理【见小节3.4】
                mapper-&gt;process(rawEvent);
            }
        }
    }
}
</code></pre></div></div>

<p>小节[3.2]createDeviceLocked创建设备并添加InputMapper，提到会有多种InputMapper。
这里以KeyboardInputMapper(按键事件)为例来展开说明</p>

<h3 id="34-按键事件处理">3.4 按键事件处理</h3>

<h4 id="341-keyboardinputmapperprocess">3.4.1 KeyboardInputMapper.process</h4>
<p>[-&gt; InputReader.cpp ::KeyboardInputMapper]</p>

<div ><div ><pre ><code>void KeyboardInputMapper::process(const RawEvent* rawEvent) {
    switch (rawEvent-&gt;type) {
    case EV_KEY: {
        int32_t scanCode = rawEvent-&gt;code;
        int32_t usageCode = mCurrentHidUsage;
        mCurrentHidUsage = 0;

        if (isKeyboardOrGamepadKey(scanCode)) {
            int32_t keyCode;
            //获取所对应的KeyCode【见小节3.4.2】
            if (getEventHub()-&gt;mapKey(getDeviceId(), scanCode, usageCode, &amp;keyCode, &amp;flags)) {
                keyCode = AKEYCODE_UNKNOWN;
                flags = 0;
            }
            //【见小节3.4.4】
            processKey(rawEvent-&gt;when, rawEvent-&gt;value != 0, keyCode, scanCode, flags);
        }
        break;
    }
    case EV_MSC: ...
    case EV_SYN: ...
    }
}
</code></pre></div></div>

<h4 id="342-eventhubmapkey">3.4.2 EventHub::mapKey</h4>
<p>[-&gt; EventHub.cpp]</p>

<div ><div ><pre ><code>status_t EventHub::mapKey(int32_t deviceId,
        int32_t scanCode, int32_t usageCode, int32_t metaState,
        int32_t* outKeycode, int32_t* outMetaState, uint32_t* outFlags) const {
    AutoMutex _l(mLock);
    Device* device = getDeviceLocked(deviceId); //获取设备对象
    status_t status = NAME_NOT_FOUND;

    if (device) {
        sp&lt;KeyCharacterMap&gt; kcm = device-&gt;getKeyCharacterMap();
        if (kcm != NULL) {
            //根据scanCode找到keyCode【见小节3.4.3】
            if (!kcm-&gt;mapKey(scanCode, usageCode, outKeycode)) {
                *outFlags = 0;
                status = NO_ERROR;
            }
        }
    }
    ...
    return status;
}
</code></pre></div></div>

<p>将事件的扫描码(scanCode)转换成键盘码(Keycode)</p>

<h4 id="343-keycharactermapmapkey">3.4.3 KeyCharacterMap::mapKey</h4>
<p>[-&gt; KeyCharacterMap.cpp]</p>

<div ><div ><pre ><code>status_t KeyCharacterMap::mapKey(int32_t scanCode, int32_t usageCode, int32_t* outKeyCode) const {
    ...
    if (scanCode) {
        ssize_t index = mKeysByScanCode.indexOfKey(scanCode);
        if (index &gt;= 0) {
            //根据scanCode找到keyCode
            *outKeyCode = mKeysByScanCode.valueAt(index);
            return OK;
        }
    }
    *outKeyCode = AKEYCODE_UNKNOWN;
    return NAME_NOT_FOUND;
}
</code></pre></div></div>

<p>再回到[3.4.1],接下来进入如下过程:</p>

<h4 id="344-inputmapperprocesskey">3.4.4 InputMapper.processKey</h4>
<p>[-&gt; InputReader.cpp]</p>

<div ><div ><pre ><code>void KeyboardInputMapper::processKey(nsecs_t when, bool down, int32_t keyCode,
        int32_t scanCode, uint32_t policyFlags) {

    if (down) {
        if (mParameters.orientationAware &amp;&amp; mParameters.hasAssociatedDisplay) {
            keyCode = rotateKeyCode(keyCode, mOrientation);
        }

        ssize_t keyDownIndex = findKeyDown(scanCode);
        if (keyDownIndex &gt;= 0) {
            //mKeyDowns记录着所有按下的键
            keyCode = mKeyDowns.itemAt(keyDownIndex).keyCode;
        } else {
            ...
            mKeyDowns.push(); //压入栈顶
            KeyDown&amp; keyDown = mKeyDowns.editTop();
            keyDown.keyCode = keyCode;
            keyDown.scanCode = scanCode;
        }
        mDownTime = when; //记录按下时间点

    } else {
        ssize_t keyDownIndex = findKeyDown(scanCode);
        if (keyDownIndex &gt;= 0) {
            //键抬起操作，则移除按下事件
            keyCode = mKeyDowns.itemAt(keyDownIndex).keyCode;
            mKeyDowns.removeAt(size_t(keyDownIndex));
        } else {
            return;  //键盘没有按下操作，则直接忽略抬起操作
        }
    }
    nsecs_t downTime = mDownTime;
    ...

    //创建NotifyKeyArgs对象, when记录eventTime, downTime记录按下时间；
    NotifyKeyArgs args(when, getDeviceId(), mSource, policyFlags,
            down ? AKEY_EVENT_ACTION_DOWN : AKEY_EVENT_ACTION_UP,
            AKEY_EVENT_FLAG_FROM_SYSTEM, keyCode, scanCode, newMetaState, downTime);
    //通知key事件【见小节3.4.5】
    getListener()-&gt;notifyKey(&amp;args);
}
</code></pre></div></div>

<p>参数说明：</p>

<ul>
  <li>mKeyDowns记录着所有按下的键;</li>
  <li>mDownTime记录按下时间点;</li>
  <li>此处KeyboardInputMapper的mContext指向InputReader，getListener()获取的便是mQueuedListener。
接下来调用该对象的notifyKey.</li>
</ul>

<h4 id="345-queuedinputlistenernotifykey">3.4.5 QueuedInputListener.notifyKey</h4>
<p>[-&gt; InputListener.cpp]</p>

<div ><div ><pre ><code>void QueuedInputListener::notifyKey(const NotifyKeyArgs* args) {
    mArgsQueue.push(new NotifyKeyArgs(*args));
}
</code></pre></div></div>

<p>mArgsQueue的数据类型为Vector&lt;NotifyArgs*&gt;，将该key事件压人该栈顶。 到此,整个事件加工完成,
再然后就是将事件发送给InputDispatcher线程.</p>

<p>接下来,再回调小节[1.2] InputReader的loopOnce过程, 可知当执行完processEventsLocked()过程,
然后便开始执行mQueuedListener-&gt;flush()过程, 如下文.</p>

<h2 id="四-queuedlistener">四. QueuedListener</h2>

<h3 id="41-queuedinputlistenerflush">4.1 QueuedInputListener.flush</h3>
<p>[-&gt; InputListener.cpp]</p>

<div ><div ><pre ><code>void QueuedInputListener::flush() {
    size_t count = mArgsQueue.size();
    for (size_t i = 0; i &lt; count; i++) {
        NotifyArgs* args = mArgsQueue[i];
        //【见小节4.2】
        args-&gt;notify(mInnerListener);
        delete args;
    }
    mArgsQueue.clear();
}
</code></pre></div></div>

<p>遍历整个mArgsQueue数组, 在input架构中NotifyArgs的实现子类主要有以下几类:</p>

<ul>
  <li>NotifyConfigurationChangedArgs</li>
  <li>NotifyKeyArgs</li>
  <li>NotifyMotionArgs</li>
  <li>NotifySwitchArgs</li>
  <li>NotifyDeviceResetArgs</li>
</ul>

<p>紧接着上述的小节[3.4.5], 可知此处是NotifyKeyArgs对象.
从InputManager对象初始化的过程可知，<code >mInnerListener</code>便是InputDispatcher对象。</p>

<h3 id="42-notifykeyargsnotify">4.2 NotifyKeyArgs.notify</h3>
<p>[-&gt; InputListener.cpp]</p>

<div ><div ><pre ><code>void NotifyKeyArgs::notify(const sp&lt;InputListenerInterface&gt;&amp; listener) const {
    listener-&gt;notifyKey(this); // this是指NotifyKeyArgs【见小节4.3】
}
</code></pre></div></div>

<h3 id="43-inputdispatchernotifykey">4.3 InputDispatcher.notifyKey</h3>
<p>[-&gt; InputDispatcher.cpp]</p>

<div ><div ><pre ><code>void InputDispatcher::notifyKey(const NotifyKeyArgs* args) {
    if (!validateKeyEvent(args-&gt;action)) {
        return;
    }
    ...
    int32_t keyCode = args-&gt;keyCode;

    if (keyCode == AKEYCODE_HOME) {
        if (args-&gt;action == AKEY_EVENT_ACTION_DOWN) {
            property_set("sys.domekey.down", "1");
        } else if (args-&gt;action == AKEY_EVENT_ACTION_UP) {
            property_set("sys.domekey.down", "0");
        }
    }

    if (metaState &amp; AMETA_META_ON &amp;&amp; args-&gt;action == AKEY_EVENT_ACTION_DOWN) {
        ...
    } else if (args-&gt;action == AKEY_EVENT_ACTION_UP) {
        ...
    }

    KeyEvent event; //初始化KeyEvent对象
    event.initialize(args-&gt;deviceId, args-&gt;source, args-&gt;action,
            flags, keyCode, args-&gt;scanCode, metaState, 0,
            args-&gt;downTime, args-&gt;eventTime);
    //mPolicy是指NativeInputManager对象。【小节4.3.1】
    mPolicy-&gt;interceptKeyBeforeQueueing(&amp;event, /*byref*/ policyFlags);

    bool needWake;
    {
        mLock.lock();
        if (shouldSendKeyToInputFilterLocked(args)) {
            mLock.unlock();
            policyFlags |= POLICY_FLAG_FILTERED;
            //当inputEventObj不为空, 则事件被filter所拦截【见小节4.3.2】
            if (!mPolicy-&gt;filterInputEvent(&amp;event, policyFlags)) {
                return;
            }
            mLock.lock();
        }

        int32_t repeatCount = 0;
        //创建KeyEntry对象
        KeyEntry* newEntry = new KeyEntry(args-&gt;eventTime,
                args-&gt;deviceId, args-&gt;source, policyFlags,
                args-&gt;action, flags, keyCode, args-&gt;scanCode,
                metaState, repeatCount, args-&gt;downTime);
        //将KeyEntry放入队列【见小节4.3.3】
        needWake = enqueueInboundEventLocked(newEntry);
        mLock.unlock();
    }

    if (needWake) {
        //唤醒InputDispatcher线程【见小节4.3.5】
        mLooper-&gt;wake();
    }
}
</code></pre></div></div>

<p>该方法的主要功能：</p>

<ol>
  <li>调用NativeInputManager.interceptKeyBeforeQueueing，加入队列前执行拦截动作，但并不改变流程，调用链：
    <ul>
      <li>IMS.interceptKeyBeforeQueueing</li>
      <li>InputMonitor.interceptKeyBeforeQueueing (继承IMS.WindowManagerCallbacks)</li>
      <li>PhoneWindowManager.interceptKeyBeforeQueueing (继承WindowManagerPolicy)</li>
    </ul>
  </li>
  <li>当mInputFilterEnabled=true(该值默认为false,可通过setInputFilterEnabled设置),则调用NativeInputManager.filterInputEvent过滤输入事件；
    <ul>
      <li>当返回值为false则过滤该事件，不再往下分发；</li>
    </ul>
  </li>
  <li>生成KeyEvent，并调用enqueueInboundEventLocked，将该事件加入到InputDispatcherd的成员变量mInboundQueue。</li>
</ol>

<h4 id="431-interceptkeybeforequeueing">4.3.1 interceptKeyBeforeQueueing</h4>

<div ><div ><pre ><code>void NativeInputManager::interceptKeyBeforeQueueing(const KeyEvent* keyEvent,
        uint32_t&amp; policyFlags) {
    ...
    if ((policyFlags &amp; POLICY_FLAG_TRUSTED)) {
        nsecs_t when = keyEvent-&gt;getEventTime(); //时间
        JNIEnv* env = jniEnv();
        jobject keyEventObj = android_view_KeyEvent_fromNative(env, keyEvent);
        if (keyEventObj) {
            // 调用Java层的IMS.interceptKeyBeforeQueueing
            wmActions = env-&gt;CallIntMethod(mServiceObj,
                    gServiceClassInfo.interceptKeyBeforeQueueing,
                    keyEventObj, policyFlags);
            ...
        } else {
            ...
        }
        handleInterceptActions(wmActions, when, /*byref*/ policyFlags);
    } else {
        ...
    }
}
</code></pre></div></div>

<p>该方法会调用Java层的InputManagerService的interceptKeyBeforeQueueing()方法。</p>

<h4 id="432-filterinputevent">4.3.2 filterInputEvent</h4>

<div ><div ><pre ><code>bool NativeInputManager::filterInputEvent(const InputEvent* inputEvent, uint32_t policyFlags) {
    jobject inputEventObj;

    JNIEnv* env = jniEnv();
    switch (inputEvent-&gt;getType()) {
    case AINPUT_EVENT_TYPE_KEY:
        inputEventObj = android_view_KeyEvent_fromNative(env,
                static_cast&lt;const KeyEvent*&gt;(inputEvent));
        break;
    case AINPUT_EVENT_TYPE_MOTION:
        inputEventObj = android_view_MotionEvent_obtainAsCopy(env,
                static_cast&lt;const MotionEvent*&gt;(inputEvent));
        break;
    default:
        return true; // 走事件正常的分发流程
    }

    if (!inputEventObj) {
        return true; // 当inputEventObj为空, 则走事件正常的分发流程
    }

    //当inputEventObj不为空,则调用Java层的IMS.filterInputEvent()
    jboolean pass = env-&gt;CallBooleanMethod(mServiceObj, gServiceClassInfo.filterInputEvent,
            inputEventObj, policyFlags);
    if (checkAndClearExceptionFromCallback(env, "filterInputEvent")) {
        pass = true; //出现Exception，则走事件正常的分发流程
    }
    env-&gt;DeleteLocalRef(inputEventObj);
    return pass;
}
</code></pre></div></div>

<p>当inputEventObj不为空,则调用Java层的IMS.filterInputEvent(). 经过层层调用后,
最终会再调用InputDispatcher.injectInputEvent(),该基本等效于该方法的后半段:</p>

<ul>
  <li>enqueueInboundEventLocked</li>
  <li>wakeup</li>
</ul>

<h4 id="433-enqueueinboundeventlocked">4.3.3 enqueueInboundEventLocked</h4>

<div ><div ><pre ><code>bool InputDispatcher::enqueueInboundEventLocked(EventEntry* entry) {
    bool needWake = mInboundQueue.isEmpty();
    mInboundQueue.enqueueAtTail(entry); //将该事件放入mInboundQueue队列尾部

    switch (entry-&gt;type) {
    case EventEntry::TYPE_KEY: {
        KeyEntry* keyEntry = static_cast&lt;KeyEntry*&gt;(entry);
        if (isAppSwitchKeyEventLocked(keyEntry)) {
            if (keyEntry-&gt;action == AKEY_EVENT_ACTION_DOWN) {
                mAppSwitchSawKeyDown = true; //按下事件
            } else if (keyEntry-&gt;action == AKEY_EVENT_ACTION_UP) {
                if (mAppSwitchSawKeyDown) {
                    //其中APP_SWITCH_TIMEOUT=500ms
                    mAppSwitchDueTime = keyEntry-&gt;eventTime + APP_SWITCH_TIMEOUT;
                    mAppSwitchSawKeyDown = false;
                    needWake = true;
                }
            }
        }
        break;
    }

    case EventEntry::TYPE_MOTION: {
        //当前App无响应且用户希望切换到其他应用窗口，则drop该窗口事件，并处理其他窗口事件
        MotionEntry* motionEntry = static_cast&lt;MotionEntry*&gt;(entry);
        if (motionEntry-&gt;action == AMOTION_EVENT_ACTION_DOWN
                &amp;&amp; (motionEntry-&gt;source &amp; AINPUT_SOURCE_CLASS_POINTER)
                &amp;&amp; mInputTargetWaitCause == INPUT_TARGET_WAIT_CAUSE_APPLICATION_NOT_READY
                &amp;&amp; mInputTargetWaitApplicationHandle != NULL) {
            int32_t displayId = motionEntry-&gt;displayId;
            int32_t x = int32_t(motionEntry-&gt;pointerCoords[0].
                    getAxisValue(AMOTION_EVENT_AXIS_X));
            int32_t y = int32_t(motionEntry-&gt;pointerCoords[0].
                    getAxisValue(AMOTION_EVENT_AXIS_Y));
            //查询可触摸的窗口【见小节4.3.4】
            sp&lt;InputWindowHandle&gt; touchedWindowHandle = findTouchedWindowAtLocked(displayId, x, y);
            if (touchedWindowHandle != NULL
                    &amp;&amp; touchedWindowHandle-&gt;inputApplicationHandle
                            != mInputTargetWaitApplicationHandle) {
                mNextUnblockedEvent = motionEntry;
                needWake = true;
            }
        }
        break;
    }
    }

    return needWake;
}
</code></pre></div></div>

<p>AppSwitchKeyEvent是指keyCode等于以下值：</p>

<ul>
  <li>AKEYCODE_HOME</li>
  <li>AKEYCODE_ENDCALL</li>
  <li>AKEYCODE_APP_SWITCH</li>
</ul>

<h4 id="434-findtouchedwindowatlocked">4.3.4 findTouchedWindowAtLocked</h4>
<p>[-&gt; InputDispatcher.cpp]</p>

<div ><div ><pre ><code>sp&lt;InputWindowHandle&gt; InputDispatcher::findTouchedWindowAtLocked(int32_t displayId,
        int32_t x, int32_t y) {
    //从前台到后台来遍历查询可触摸的窗口
    size_t numWindows = mWindowHandles.size();
    for (size_t i = 0; i &lt; numWindows; i++) {
        sp&lt;InputWindowHandle&gt; windowHandle = mWindowHandles.itemAt(i);
        const InputWindowInfo* windowInfo = windowHandle-&gt;getInfo();
        if (windowInfo-&gt;displayId == displayId) {
            int32_t flags = windowInfo-&gt;layoutParamsFlags;

            if (windowInfo-&gt;visible) {
                if (!(flags &amp; InputWindowInfo::FLAG_NOT_TOUCHABLE)) {
                    bool isTouchModal = (flags &amp; (InputWindowInfo::FLAG_NOT_FOCUSABLE
                            | InputWindowInfo::FLAG_NOT_TOUCH_MODAL)) == 0;
                    if (isTouchModal || windowInfo-&gt;touchableRegionContainsPoint(x, y)) {
                        return windowHandle; //找到目标窗口
                    }
                }
            }
        }
    }
    return NULL;
}
</code></pre></div></div>

<p>此处mWindowHandles的赋值过程是由Java层的InputMonitor.setInputWindows(),经过JNI调用后进入InputDispatcher::setInputWindows()方法完成.
进一步说, 就是WMS执行addWindow()过程或许UI改变等场景,都会触发该方法的修改.</p>

<h4 id="435-looperwake">4.3.5 Looper.wake</h4>
<p>[-&gt; system/core/libutils/Looper.cpp]</p>

<div ><div ><pre ><code>void Looper::wake() {
    uint64_t inc = 1;

    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &amp;inc, sizeof(uint64_t)));
    if (nWrite != sizeof(uint64_t)) {
        if (errno != EAGAIN) {
            ALOGW("Could not write wake signal, errno=%d", errno);
        }
    }
}
</code></pre></div></div>

<p>[小节4.3]的过程会调用enqueueInboundEventLocked()方法来决定是否需要将数字1写入句柄mWakeEventFd来唤醒InputDispatcher线程.
满足唤醒的条件:</p>

<ol>
  <li>执行enqueueInboundEventLocked方法前,mInboundQueue队列为空,执行完必然不再为空,则需要唤醒分发线程;</li>
  <li>当事件类型为key事件,且发生一对按下和抬起操作,则需要唤醒;</li>
  <li>当事件类型为motion事件,且当前可触摸的窗口属于另一个应用,则需要唤醒.</li>
</ol>

<h2 id="五-总结">五. 总结</h2>

<h3 id="51-核心工作">5.1 核心工作</h3>

<p>InputReader整个过程涉及多次事件封装转换，其主要工作核心是以下三大步骤:</p>

<ul>
  <li>getEvents：通过EventHub(监听目录/dev/input)读取事件放入mEventBuffer,而mEventBuffer是一个大小为256的数组, 再将事件input_event转换为RawEvent; [见小节2.1]</li>
  <li>processEventsLocked: 对事件进行加工, 转换RawEvent -&gt; NotifyKeyArgs(NotifyArgs) [见小节3.1]</li>
  <li>QueuedListener-&gt;flush：将事件发送到InputDispatcher线程, 转换NotifyKeyArgs -&gt; KeyEntry(EventEntry) [见小节4.1]</li>
</ul>

<p>InputReader线程不断循环地执行InputReader.loopOnce(), 每次处理完生成的是EventEntry(比如KeyEntry, MotionEntry), 接下来的工作就交给InputDispatcher线程。</p>

<h3 id="52-流程图">5.2 流程图</h3>


<p><img src="https://img-blog.csdnimg.cn/5013c6aff19c472596212664b732ba80.jpg?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt="input_reader_seq" /></p>

<p>InputReader的核心工作就是从EventHub获取数据后生成EventEntry事件，加入到InputDispatcher的mInboundQueue队列，再唤醒InputDispatcher线程。</p>

<p><img src="https://img-blog.csdnimg.cn/a4acb4626d0543f29197b4a57a64deb6.jpg?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt="input_reader" /></p>

<p>说明:</p>

<ul>
  <li>IMS.filterInputEvent可以过滤无需上报的事件，当该方法返回值为false则代表是需要被过滤掉的事件，无机会交给InputDispatcher来分发。</li>
  <li>节点/dev/input的event事件所对应的输入设备信息位于<code >/proc/bus/input/devices</code>，也可以通过<code >getevent</code>来获取事件. 不同的input事件所对应的物理input节点，比如常见的情形：
    <ul>
      <li>屏幕触摸和(MENU,HOME,BACK)3按键：对应同一个input设备节点；</li>
      <li>POWER和音量(下)键：对应同一个input设备节点；</li>
      <li>音量(上)键：对应同一个input设备节点；</li>
    </ul>
  </li>
</ul>