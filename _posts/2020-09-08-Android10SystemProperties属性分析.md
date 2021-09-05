---
layout:     post
title:      Android10 SystemProperties属性分析
subtitle:   SystemProperties.set方法可以设置系统属性，通过设置系统属性可以启动一些服务和操作，如关机，重启，uncrypt服务等
date:       2020-09-08
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - 系统架构
    - Android
    - Android10
    - framework
---
<p>SystemProperties.set方法可以设置系统属性&#xff0c;通过设置系统属性可以启动一些服务和操作&#xff0c;如关机&#xff0c;重启、uncrypt服务等。下面将分析为什么设置系统属性&#xff0c;可以做到即时生效某些操作。</p> 
<p>下面将以SystemProperties.set(“ctl.start”, “uncrypt”);为例说明整个流程。</p> 
<h2><a id="_6"></a>一、属性服务客户端</h2> 
<h3><a id="11_SystemPropertiesset_8"></a>1.1 SystemProperties.set</h3> 
<p>[-&gt;SystemProperties.java]</p> 
<pre><code>    /**
     * Set the value for the given {&#64;code key} to {&#64;code val}.
     *
     * &#64;throws IllegalArgumentException if the {&#64;code val} exceeds 91 characters
     * &#64;hide
     */
    &#64;UnsupportedAppUsage
    public static void set(&#64;NonNull String key, &#64;Nullable String val) {
        if (val !&#61; null &amp;&amp; !val.startsWith(&#34;ro.&#34;) &amp;&amp; val.length() &gt; PROP_VALUE_MAX) {
            throw new IllegalArgumentException(&#34;value of system property &#39;&#34; &#43; key
                    &#43; &#34;&#39; is longer than &#34; &#43; PROP_VALUE_MAX &#43; &#34; characters: &#34; &#43; val);
        }
        if (TRACK_KEY_ACCESS) onKeyAccess(key);
        //调用native方法
        native_set(key, val);
    }
</code></pre> 
<h3><a id="12_SystemProperties_set_31"></a>1.2 SystemProperties_set</h3> 
<p>[-&gt;android_os_SystemProperties.cpp]</p> 
<pre><code>void SystemProperties_set(JNIEnv *env, jobject clazz, jstring keyJ,
                          jstring valJ)
{
    auto handler &#61; [&amp;](const std::string&amp; key, bool) {
        std::string val;
        if (valJ !&#61; nullptr) {
            ScopedUtfChars key_utf(env, valJ);
            val &#61; key_utf.c_str();
        }
        return android::base::SetProperty(key, val);
    };
    if (!ConvertKeyAndForward(env, keyJ, true, handler)) {
        // Must have been a failure in SetProperty.
        jniThrowException(env, &#34;java/lang/RuntimeException&#34;,
                          &#34;failed to set system property&#34;);
    }
}
</code></pre> 
<h3><a id="13_SetProperty_55"></a>1.3 SetProperty</h3> 
<p>[-&gt;properties.cpp]</p> 
<pre><code>bool SetProperty(const std::string&amp; key, const std::string&amp; value) {
  return (__system_property_set(key.c_str(), value.c_str()) &#61;&#61; 0);
}
</code></pre> 
<h3><a id="14_system_property_set_65"></a>1.4 system_property_set</h3> 
<p>[-&gt;system_property_set.cpp]</p> 
<pre><code>int __system_property_set(const char* key, const char* value) {
  if (key &#61;&#61; nullptr) return -1;
  if (value &#61;&#61; nullptr) value &#61; &#34;&#34;;

  if (g_propservice_protocol_version &#61;&#61; 0) {
    detect_protocol_version();
  }
  //检查协议
  if (g_propservice_protocol_version &#61;&#61; kProtocolVersion1) {
    // Old protocol does not support long names or values
    if (strlen(key) &gt;&#61; PROP_NAME_MAX) return -1;
    if (strlen(value) &gt;&#61; PROP_VALUE_MAX) return -1;

    prop_msg msg;
    memset(&amp;msg, 0, sizeof msg);
    msg.cmd &#61; PROP_MSG_SETPROP;
    strlcpy(msg.name, key, sizeof msg.name);
    strlcpy(msg.value, value, sizeof msg.value);
    //发送消息&#xff0c;消息中包括key和value
    return send_prop_msg(&amp;msg);
  } else {
    // New protocol only allows long values for ro. properties only.
    if (strlen(value) &gt;&#61; PROP_VALUE_MAX &amp;&amp; strncmp(key, &#34;ro.&#34;, 3) !&#61; 0) return -1;
    // Use proper protocol
    PropertyServiceConnection connection;
    if (!connection.IsValid()) {
      errno &#61; connection.GetLastError();
      async_safe_format_log(
          ANDROID_LOG_WARN, &#34;libc&#34;,
          &#34;Unable to set property /&#34;%s/&#34; to /&#34;%s/&#34;: connection failed; errno&#61;%d (%s)&#34;, key, value,
          errno, strerror(errno));
      return -1;
    }

    SocketWriter writer(&amp;connection);
    if (!writer.WriteUint32(PROP_MSG_SETPROP2).WriteString(key).WriteString(value).Send()) {
      errno &#61; connection.GetLastError();
      async_safe_format_log(ANDROID_LOG_WARN, &#34;libc&#34;,
                            &#34;Unable to set property /&#34;%s/&#34; to /&#34;%s/&#34;: write failed; errno&#61;%d (%s)&#34;,
                            key, value, errno, strerror(errno));
      return -1;
    }

    int result &#61; -1;
    if (!connection.RecvInt32(&amp;result)) {
      errno &#61; connection.GetLastError();
      async_safe_format_log(ANDROID_LOG_WARN, &#34;libc&#34;,
                            &#34;Unable to set property /&#34;%s/&#34; to /&#34;%s/&#34;: recv failed; errno&#61;%d (%s)&#34;,
                            key, value, errno, strerror(errno));
      return -1;
    }

    if (result !&#61; PROP_SUCCESS) {
      async_safe_format_log(ANDROID_LOG_WARN, &#34;libc&#34;,
                            &#34;Unable to set property /&#34;%s/&#34; to /&#34;%s/&#34;: error code: 0x%x&#34;, key, value,
                            result);
      return -1;
    }

    return 0;
  }
}

</code></pre> 
<h3><a id="15_send_prop_msg_135"></a>1.5 send_prop_msg</h3> 
<p>[-&gt;system_property_set.cpp]</p> 
<pre><code>static int send_prop_msg(const prop_msg* msg) {
  PropertyServiceConnection connection;
  if (!connection.IsValid()) {
    return connection.GetLastError();
  }

  int result &#61; -1;
  //连接PROP_SERVICE的socket
  int s &#61; connection.socket();
 
  //通过socket发送消息
  const int num_bytes &#61; TEMP_FAILURE_RETRY(send(s, msg, sizeof(prop_msg), 0));
  if (num_bytes &#61;&#61; sizeof(prop_msg)) {
    // We successfully wrote to the property server but now we
    // wait for the property server to finish its work.  It
    // acknowledges its completion by closing the socket so we
    // poll here (on nothing), waiting for the socket to close.
    // If you &#39;adb shell setprop foo bar&#39; you&#39;ll see the POLLHUP
    // once the socket closes.  Out of paranoia we cap our poll
    // at 250 ms.
    pollfd pollfds[1];
    pollfds[0].fd &#61; s;
    pollfds[0].events &#61; 0;
    const int poll_result &#61; TEMP_FAILURE_RETRY(poll(pollfds, 1, 250 /* ms */));
    if (poll_result &#61;&#61; 1 &amp;&amp; (pollfds[0].revents &amp; POLLHUP) !&#61; 0) {
      result &#61; 0;
    } else {
      // Ignore the timeout and treat it like a success anyway.
      // The init process is single-threaded and its property
      // service is sometimes slow to respond (perhaps it&#39;s off
      // starting a child process or something) and thus this
      // times out and the caller thinks it failed, even though
      // it&#39;s still getting around to it.  So we fake it here,
      // mostly for ctl.* properties, but we do try and wait 250
      // ms so callers who do read-after-write can reliably see
      // what they&#39;ve written.  Most of the time.
      // TODO: fix the system properties design.
      async_safe_format_log(ANDROID_LOG_WARN, &#34;libc&#34;,
                            &#34;Property service has timed out while trying to set /&#34;%s/&#34; to /&#34;%s/&#34;&#34;,
                            msg-&gt;name, msg-&gt;value);
      result &#61; 0;
    }
  }

  return result;
}
</code></pre> 
<p>static const char property_service_socket[] &#61; “/dev/socket/” PROP_SERVICE_NAME;</p> 
<p>这里通过property_service建立socket的连接&#xff0c;将key,value发送到property_service。</p> 
<p>那么property_service是在什么时候启动的&#xff0c;是在开机的时候通过init进程启动的服务&#xff0c;开启property_service后&#xff0c;一直在监听property_service_socket&#xff0c;等待客户端的连接。</p> 
<h2><a id="_194"></a>二、属性服务服务端</h2> 
<h3><a id="21_initmain_196"></a>2.1 init.main</h3> 
<p>[-&gt;init.cpp]</p> 
<pre><code>int main(int argc, char** argv) {
  
    ..... 
    //property初始化&#xff0c;见2.2节
    property_init();

    // If arguments are passed both on the command line and in DT,
    // properties set in DT always have priority over the command-line ones.
    process_kernel_dt();
    process_kernel_cmdline();

    // Propagate the kernel variables to internal variables
    // used by init as well as the current required properties.
    export_kernel_boot_props();

    // Make the time that init started available for bootstat to log.
    property_set(&#34;ro.boottime.init&#34;, getenv(&#34;INIT_STARTED_AT&#34;));
    property_set(&#34;ro.boottime.init.selinux&#34;, getenv(&#34;INIT_SELINUX_TOOK&#34;));

    ....
    property_load_boot_defaults();
    export_oem_lock_status();
    //property服务启动&#xff0c;见2.3节
    start_property_service();
    set_usb_controller();
    ...
    //初始化ActionManager
    ActionManager&amp; am &#61; ActionManager::GetInstance();
    ServiceList&amp; sm &#61; ServiceList::GetInstance();

    LoadBootScripts(am, sm);

    // Turning this on and letting the INFO logging be discarded adds 0.2s to
    // Nexus 9 boot time, so it&#39;s disabled by default.
    if (false) DumpState();

    am.QueueEventTrigger(&#34;early-init&#34;);

    // Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...
    am.QueueBuiltinAction(wait_for_coldboot_done_action, &#34;wait_for_coldboot_done&#34;);
    // ... so that we can start queuing up actions that require stuff from /dev.
    am.QueueBuiltinAction(MixHwrngIntoLinuxRngAction, &#34;MixHwrngIntoLinuxRng&#34;);
    am.QueueBuiltinAction(SetMmapRndBitsAction, &#34;SetMmapRndBits&#34;);
    am.QueueBuiltinAction(SetKptrRestrictAction, &#34;SetKptrRestrict&#34;);
    am.QueueBuiltinAction(keychord_init_action, &#34;keychord_init&#34;);
    am.QueueBuiltinAction(console_init_action, &#34;console_init&#34;);

    // Trigger all the boot actions to get us started.
    am.QueueEventTrigger(&#34;init&#34;);

    // Starting the BoringSSL self test, for NIAP certification compliance.
    am.QueueBuiltinAction(StartBoringSslSelfTest, &#34;StartBoringSslSelfTest&#34;);

    // Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random
    // wasn&#39;t ready immediately after wait_for_coldboot_done
    am.QueueBuiltinAction(MixHwrngIntoLinuxRngAction, &#34;MixHwrngIntoLinuxRng&#34;);

    // Don&#39;t mount filesystems or start core system services in charger mode.
    std::string bootmode &#61; GetProperty(&#34;ro.bootmode&#34;, &#34;&#34;);
    if (bootmode &#61;&#61; &#34;charger&#34;) {
        am.QueueEventTrigger(&#34;charger&#34;);
    } else {
        am.QueueEventTrigger(&#34;late-init&#34;);
    }

    // Run all property triggers based on current state of the properties.
    am.QueueBuiltinAction(queue_property_triggers_action, &#34;queue_property_triggers&#34;);
    //循环查询action
    while (true) {
        // By default, sleep until something happens.
        int epoll_timeout_ms &#61; -1;

        if (do_shutdown &amp;&amp; !shutting_down) {
            do_shutdown &#61; false;
            if (HandlePowerctlMessage(shutdown_command)) {
                shutting_down &#61; true;
            }
        }
        //是否有事件需要处理
        if (!(waiting_for_prop || Service::is_exec_service_running())) {
            //依次执行每个action中携带command对应的执行函数
            am.ExecuteOneCommand();
        }
        if (!(waiting_for_prop || Service::is_exec_service_running())) {
            if (!shutting_down) {
                auto next_process_restart_time &#61; RestartProcesses();

                // If there&#39;s a process that needs restarting, wake up in time for that.
                if (next_process_restart_time) {
                    epoll_timeout_ms &#61; std::chrono::ceil&lt;std::chrono::milliseconds&gt;(
                                           *next_process_restart_time - boot_clock::now())
                                           .count();
                    if (epoll_timeout_ms &lt; 0) epoll_timeout_ms &#61; 0;
                }
            }
            // 有action待处理&#xff0c;不等待
            // If there&#39;s more work to do, wake up again immediately.
            if (am.HasMoreCommands()) epoll_timeout_ms &#61; 0;
        }
        
        //等待子线程终止信号的处理
        epoll_event ev;
        int nr &#61; TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &amp;ev, 1, epoll_timeout_ms));
        if (nr &#61;&#61; -1) {
            PLOG(ERROR) &lt;&lt; &#34;epoll_wait failed&#34;;
        } else if (nr &#61;&#61; 1) {
            ((void (*)()) ev.data.ptr)();
        }
    }
}

</code></pre> 
<h3><a id="22_property_init_314"></a>2.2 property_init</h3> 
<p>[-&gt;property_service.cpp]</p> 
<pre><code>void property_init() {
    mkdir(&#34;/dev/__properties__&#34;, S_IRWXU | S_IXGRP | S_IXOTH);
    CreateSerializedPropertyInfo();
    if (__system_property_area_init()) {
        LOG(FATAL) &lt;&lt; &#34;Failed to initialize property area&#34;;
    }
    if (!property_info_area.LoadDefaultPath()) {
        LOG(FATAL) &lt;&lt; &#34;Failed to load serialized property info file&#34;;
    }
}
</code></pre> 
<h3><a id="23_start_property_service_331"></a>2.3 start_property_service</h3> 
<p>[-&gt;property_service.cpp]</p> 
<pre><code>void start_property_service() {
    selinux_callback cb;
    cb.func_audit &#61; SelinuxAuditCallback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);

    property_set(&#34;ro.property_service.version&#34;, &#34;2&#34;);
    
    //创建端口
    property_set_fd &#61; CreateSocket(PROP_SERVICE_NAME, SOCK_STREAM | SOCK_CLOEXEC | SOCK_NONBLOCK,
                                   false, 0666, 0, 0, nullptr);
    if (property_set_fd &#61;&#61; -1) {
        PLOG(FATAL) &lt;&lt; &#34;start_property_service socket creation failed&#34;;
    }

    listen(property_set_fd, 8);
    
    //注册epoll handler&#xff0c;处理消息
    register_epoll_handler(property_set_fd, handle_property_set_fd);
}
</code></pre> 
<h3><a id="24_handle_property_set_fd_357"></a>2.4 handle_property_set_fd</h3> 
<p>[-&gt;property_service.cpp]</p> 
<pre><code>static void handle_property_set_fd() {
    static constexpr uint32_t kDefaultSocketTimeout &#61; 2000; /* ms */

    int s &#61; accept4(property_set_fd, nullptr, nullptr, SOCK_CLOEXEC);
    if (s &#61;&#61; -1) {
        return;
    }

    ucred cr;
    socklen_t cr_size &#61; sizeof(cr);
    if (getsockopt(s, SOL_SOCKET, SO_PEERCRED, &amp;cr, &amp;cr_size) &lt; 0) {
        close(s);
        PLOG(ERROR) &lt;&lt; &#34;sys_prop: unable to get SO_PEERCRED&#34;;
        return;
    }

    SocketConnection socket(s, cr);
    uint32_t timeout_ms &#61; kDefaultSocketTimeout;

    uint32_t cmd &#61; 0;
    if (!socket.RecvUint32(&amp;cmd, &amp;timeout_ms)) {
        PLOG(ERROR) &lt;&lt; &#34;sys_prop: error while reading command from the socket&#34;;
        socket.SendUint32(PROP_ERROR_READ_CMD);
        return;
    }

    switch (cmd) {
    //处理消息cmd
    case PROP_MSG_SETPROP: {
        char prop_name[PROP_NAME_MAX];
        char prop_value[PROP_VALUE_MAX];

        if (!socket.RecvChars(prop_name, PROP_NAME_MAX, &amp;timeout_ms) ||
            !socket.RecvChars(prop_value, PROP_VALUE_MAX, &amp;timeout_ms)) {
          PLOG(ERROR) &lt;&lt; &#34;sys_prop(PROP_MSG_SETPROP): error while reading name/value from the socket&#34;;
          return;
        }

        prop_name[PROP_NAME_MAX-1] &#61; 0;
        prop_value[PROP_VALUE_MAX-1] &#61; 0;

        const auto&amp; cr &#61; socket.cred();
        std::string error;
        uint32_t result &#61;
            //见2.5节
            HandlePropertySet(prop_name, prop_value, socket.source_context(), cr, &amp;error);
        if (result !&#61; PROP_SUCCESS) {
            LOG(ERROR) &lt;&lt; &#34;Unable to set property &#39;&#34; &lt;&lt; prop_name &lt;&lt; &#34;&#39; to &#39;&#34; &lt;&lt; prop_value
                       &lt;&lt; &#34;&#39; from uid:&#34; &lt;&lt; cr.uid &lt;&lt; &#34; gid:&#34; &lt;&lt; cr.gid &lt;&lt; &#34; pid:&#34; &lt;&lt; cr.pid &lt;&lt; &#34;: &#34;
                       &lt;&lt; error;
        }

        break;
      }

    case PROP_MSG_SETPROP2: {
        std::string name;
        std::string value;
        if (!socket.RecvString(&amp;name, &amp;timeout_ms) ||
            !socket.RecvString(&amp;value, &amp;timeout_ms)) {
          PLOG(ERROR) &lt;&lt; &#34;sys_prop(PROP_MSG_SETPROP2): error while reading name/value from the socket&#34;;
          socket.SendUint32(PROP_ERROR_READ_DATA);
          return;
        }

        const auto&amp; cr &#61; socket.cred();
        std::string error;
        uint32_t result &#61; HandlePropertySet(name, value, socket.source_context(), cr, &amp;error);
        if (result !&#61; PROP_SUCCESS) {
            LOG(ERROR) &lt;&lt; &#34;Unable to set property &#39;&#34; &lt;&lt; name &lt;&lt; &#34;&#39; to &#39;&#34; &lt;&lt; value
                       &lt;&lt; &#34;&#39; from uid:&#34; &lt;&lt; cr.uid &lt;&lt; &#34; gid:&#34; &lt;&lt; cr.gid &lt;&lt; &#34; pid:&#34; &lt;&lt; cr.pid &lt;&lt; &#34;: &#34;
                       &lt;&lt; error;
        }
        socket.SendUint32(result);
        break;
      }

    default:
        LOG(ERROR) &lt;&lt; &#34;sys_prop: invalid command &#34; &lt;&lt; cmd;
        socket.SendUint32(PROP_ERROR_INVALID_CMD);
        break;
    }
}
</code></pre> 
<h3><a id="25__HandlePropertySet_447"></a>2.5 HandlePropertySet</h3> 
<p>[-&gt;property_service.cpp]</p> 
<pre><code>// This returns one of the enum of PROP_SUCCESS or PROP_ERROR*.
uint32_t HandlePropertySet(const std::string&amp; name, const std::string&amp; value,
                           const std::string&amp; source_context, const ucred&amp; cr, std::string* error) {
    if (!IsLegalPropertyName(name)) {
        *error &#61; &#34;Illegal property name&#34;;
        return PROP_ERROR_INVALID_NAME;
    }

    if (StartsWith(name, &#34;ctl.&#34;)) {
         //检查权限
        if (!CheckControlPropertyPerms(name, value, source_context, cr)) {
            *error &#61; StringPrintf(&#34;Invalid permissions to perform &#39;%s&#39; on &#39;%s&#39;&#34;, name.c_str() &#43; 4,
                                  value.c_str());
            return PROP_ERROR_HANDLE_CONTROL_MESSAGE;
        }
        //见2.6节
        HandleControlMessage(name.c_str() &#43; 4, value, cr.pid);
        return PROP_SUCCESS;
    }

    const char* target_context &#61; nullptr;
    const char* type &#61; nullptr;
    property_info_area-&gt;GetPropertyInfo(name.c_str(), &amp;target_context, &amp;type);

    if (!CheckMacPerms(name, target_context, source_context.c_str(), cr)) {
        *error &#61; &#34;SELinux permission check failed&#34;;
        return PROP_ERROR_PERMISSION_DENIED;
    }

    if (type &#61;&#61; nullptr || !CheckType(type, value)) {
        *error &#61; StringPrintf(&#34;Property type check failed, value doesn&#39;t match expected type &#39;%s&#39;&#34;,
                              (type ?: &#34;(null)&#34;));
        return PROP_ERROR_INVALID_VALUE;
    }

    // sys.powerctl is a special property that is used to make the device reboot.  We want to log
    // any process that sets this property to be able to accurately blame the cause of a shutdown.
    // 开关机
   if (name &#61;&#61; &#34;sys.powerctl&#34;) {
        std::string cmdline_path &#61; StringPrintf(&#34;proc/%d/cmdline&#34;, cr.pid);
        std::string process_cmdline;
        std::string process_log_string;
        if (ReadFileToString(cmdline_path, &amp;process_cmdline)) {
            // Since cmdline is null deliminated, .c_str() conveniently gives us just the process
            // path.
            process_log_string &#61; StringPrintf(&#34; (%s)&#34;, process_cmdline.c_str());
        }
        LOG(INFO) &lt;&lt; &#34;Received sys.powerctl&#61;&#39;&#34; &lt;&lt; value &lt;&lt; &#34;&#39; from pid: &#34; &lt;&lt; cr.pid
                  &lt;&lt; process_log_string;
    }

    if (name &#61;&#61; &#34;selinux.restorecon_recursive&#34;) {
        return PropertySetAsync(name, value, RestoreconRecursiveAsync, error);
    }
    //设置属性
    return PropertySet(name, value, error);
}
</code></pre> 
<p>根据不同的key字段的不同进行处理&#xff0c;在进行处理之前需要是否有相应的权限。</p> 
<h3><a id="26_HandleControlMessage_513"></a>2.6 HandleControlMessage</h3> 
<p>[-&gt;init.cpp]</p> 
<pre><code>void HandleControlMessage(const std::string&amp; msg, const std::string&amp; name, pid_t pid) {
    const auto&amp; map &#61; get_control_message_map();
    const auto it &#61; map.find(msg);

    if (it &#61;&#61; map.end()) {
        LOG(ERROR) &lt;&lt; &#34;Unknown control msg &#39;&#34; &lt;&lt; msg &lt;&lt; &#34;&#39;&#34;;
        return;
    }

    std::string cmdline_path &#61; StringPrintf(&#34;proc/%d/cmdline&#34;, pid);
    std::string process_cmdline;
    if (ReadFileToString(cmdline_path, &amp;process_cmdline)) {
        std::replace(process_cmdline.begin(), process_cmdline.end(), &#39;/0&#39;, &#39; &#39;);
        process_cmdline &#61; Trim(process_cmdline);
    } else {
        process_cmdline &#61; &#34;unknown process&#34;;
    }

    LOG(INFO) &lt;&lt; &#34;Received control message &#39;&#34; &lt;&lt; msg &lt;&lt; &#34;&#39; for &#39;&#34; &lt;&lt; name &lt;&lt; &#34;&#39; from pid: &#34; &lt;&lt; pid
              &lt;&lt; &#34; (&#34; &lt;&lt; process_cmdline &lt;&lt; &#34;)&#34;;

    const ControlMessageFunction&amp; function &#61; it-&gt;second;

    //服务
    if (function.target &#61;&#61; ControlTarget::SERVICE) {
        //通过服务名字获取avc
        Service* svc &#61; ServiceList::GetInstance().FindService(name);
        if (svc &#61;&#61; nullptr) {
            LOG(ERROR) &lt;&lt; &#34;No such service &#39;&#34; &lt;&lt; name &lt;&lt; &#34;&#39; for ctl.&#34; &lt;&lt; msg;
            return;
        }
        //触发启动服务
        if (auto result &#61; function.action(svc); !result) {
            LOG(ERROR) &lt;&lt; &#34;Could not ctl.&#34; &lt;&lt; msg &lt;&lt; &#34; for service &#34; &lt;&lt; name &lt;&lt; &#34;: &#34;
                       &lt;&lt; result.error();
        }

        return;
    }

    if (function.target &#61;&#61; ControlTarget::INTERFACE) {
        for (const auto&amp; svc : ServiceList::GetInstance()) {
            if (svc-&gt;interfaces().count(name) &#61;&#61; 0) {
                continue;
            }

            if (auto result &#61; function.action(svc.get()); !result) {
                LOG(ERROR) &lt;&lt; &#34;Could not handle ctl.&#34; &lt;&lt; msg &lt;&lt; &#34; for service &#34; &lt;&lt; svc-&gt;name()
                           &lt;&lt; &#34; with interface &#34; &lt;&lt; name &lt;&lt; &#34;: &#34; &lt;&lt; result.error();
            }

            return;
        }

        LOG(ERROR) &lt;&lt; &#34;Could not find service hosting interface &#34; &lt;&lt; name;
        return;
    }

    LOG(ERROR) &lt;&lt; &#34;Invalid function target from static map key &#39;&#34; &lt;&lt; msg
               &lt;&lt; &#34;&#39;: &#34; &lt;&lt; static_cast&lt;std::underlying_type&lt;ControlTarget&gt;::type&gt;(function.target);
}
</code></pre> 
<p>这里最终通过function.action来启动uncrypt服务。</p> 
<h2><a id="_583"></a>附录</h2> 
<p>源码路径</p> 
<pre><code>frameworks/base/core/java/android/os/SystemProperties.java
frameworks/base/core/jni/android_os_SystemProperties.cpp
system/core/base/properties.cpp
bionic/libc/bionic/system_property_set.cpp
system/core/init/property_service.cpp
system/core/init/init.cpp
</code></pre>