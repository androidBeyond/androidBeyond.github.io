---
layout:     post
title:      Android SWT类问题分析总结
subtitle:   Android SWT(Software Watch Dog ) 主要用来监控SystemServer等重要线程/Service 的运行情况。如果发现系统阻塞会尝试重启系统，以保证恢复到正常状态
date:       2021-12-08
author:     coderman
header-img: img/article-bg.jpg
top: false
catalog: true 
tags:
    - Android
    - SWT
    - Watch dog
---

<h1><a id="_SWT__9"></a>一、 SWT 重启问题简介</h1> 
<p><strong>SWT(Software Watch Dog )</strong> 主要用来监控<code>SystemServer</code>等<code>重要线程/Service</code> 的运行情况。如果发现其阻塞超过 <strong>60s</strong> ,看门狗进程就会把系统重启&#xff0c;进而保证系统可以恢复到正常状态。</p> 
<p><strong>判断阻塞的方法</strong>&#xff1a;</p> 
<ul><li>1.利用 Services 注册monitor 去Check</li></ul> 
<p>主要是&#xff1a; <strong>AMS</strong>、 <strong>Foreground Thread</strong></p> 
<ul><li> 
  <ol><li>发送handler 到重要的Loop 线程来Check 是否阻塞。</li></ol> </li></ul> 
<p>主要是&#xff1a; <strong>Main Thread</strong>、<strong>UI Thread</strong>、<strong>IO Thread</strong>、<strong>Display Thread</strong>、<strong>WMS</strong> 、<strong>Other Services</strong>。</p> 
<p><strong>SWT 判断阻塞的方法 图文描述如下&#xff1a;</strong></p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy81ODUxMjU2LTQxYjg2MWY1ODU0ZWVmNmYucG5n?x-oss-process&#61;image/format,png" alt="img" /></p> 
<p>SWT 判断阻塞的方法</p> 
<h1><a id="_SWT___31"></a>二、 SWT 重启问题处理流程</h1> 
<p><strong>SWT 处理流程&#xff1a;</strong><br /> <strong>1.每半分钟check 一次system_server 进程</strong>&#xff1a;<br /> 检查系统是否卡住&#xff0c;如果卡住&#xff0c;<code>dump</code> 一次<code>system_server</code> 的<code>backtrace</code></p> 
<p><strong>2.一分钟卡住后kill&#xff0c;并重新计数</strong>&#xff1a;<br /> 如果卡住&#xff0c;第二次<code>dump</code>&#xff0c;并<code>kill</code>掉 <code>system_server</code>进程 &#xff0c;否则重新计时。</p> 
<p><strong>3.SWT 处理大致流程如下&#xff1a;</strong></p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy81ODUxMjU2LTUwM2RkZGQ2NGZkYzJkZjAucG5n?x-oss-process&#61;image/format,png" alt="img" /></p> 
<p>SWT 处理流程</p> 
<h1><a id="_SWT___48"></a>三、 SWT 重启问题的原因</h1> 
<p>导致 <code>SWT</code>重启原因的原因有很多种。</p> 
<p>主要导致的原因如下&#xff1a;</p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy81ODUxMjU2LTliMTE5YTMyZmJjNTc4NTcucG5n?x-oss-process&#61;image/format,png" alt="img" /></p> 
<p>检查SWT 原因分类</p> 
<h1><a id="_SWT___60"></a>四、 SWT 重启问题分析流程</h1> 
<p>首先搜索关键 <strong>watchdog</strong>&#xff0c;查看是否有重启发生。</p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy81ODUxMjU2LTU4MTJmOWY3NDdiYjUyNTEucG5n?x-oss-process&#61;image/format,png" alt="img" /></p> 
<p>SWT 流程分析</p> 
<h1><a id="SWT__70"></a>五、SWT 重启问题分析举例</h1> 
<h2><a id="1_trace__72"></a>1.分析 trace &#xff0c;确认线程关系</h2> 
<p>线程被 <strong>Block</strong> 搜索关键字 <strong>held by</strong></p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy81ODUxMjU2LTU1YWNiNTBkNTA5NmRkMDgucG5n?x-oss-process&#61;image/format,png" alt="img" /></p> 
<p>确认线程关系</p> 
<p>线程被 <strong>Waiting</strong> 结合代码分析。</p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy81ODUxMjU2LTVkZGI5MDU2OTg5M2FmOTQucG5n?x-oss-process&#61;image/format,png" alt="img" /></p> 
<p>确认线程关系</p> 
<h2><a id="2_90"></a>2.线程死锁</h2> 
<p>确认<strong>Block</strong>的线程是否有闭环的死锁关系。</p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy81ODUxMjU2LTIxYzEyYmIxNWU1YmIxZWUucG5n?x-oss-process&#61;image/format,png" alt="img" /></p> 
<p>线程死锁</p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy81ODUxMjU2LThiNTFhZWE4MzAzZTVkZDkucG5n?x-oss-process&#61;image/format,png" alt="img" /></p> 
<p>线程死锁</p> 
<h2><a id="3BinderServer__106"></a>3.Binder的Server 端卡住</h2> 
<p>线程状态 <strong>Native</strong>&#xff0c;并且<strong>callstack</strong>中含有一对</p> 
<p><strong>IPCThreadState::waitForResponse</strong><br /> <strong>IPCThreadState::talkWithDriver</strong><br /> 的明显特征。</p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy81ODUxMjU2LTQ4Mzk4MGQyZTIwYzQxOTAucG5n?x-oss-process&#61;image/format,png" alt="img" /></p> 
<p>Bind的Server端卡住</p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy81ODUxMjU2LTQ4NzA3MTgwZTM2ZjdiMGMucG5n?x-oss-process&#61;image/format,png" alt="img" /></p> 
<p>Bind的Server端卡住</p> 
<h2><a id="4SurfaceFlinger__126"></a>4.SurfaceFlinger 卡住导致重启</h2> 
<p>搜索<code>关键字</code> <strong>I watchdog</strong> ,<br /> 查看是否有 <strong>surfaceflinger hang</strong>&#xff0c;默认卡住<code>40s</code>&#xff0c;就会重启。</p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy81ODUxMjU2LTg3OTUxN2UwZmFkMDZjYzgucG5n?x-oss-process&#61;image/format,png" alt="img" /></p> 
<p>SurfaceFlinger 卡住</p> 
<h2><a id="5Native__137"></a>5.Native 方法执行时间过长导致重启</h2> 
<p>线程状态 <strong>Native</strong>&#xff0c;查看是否有<br /> <strong>PowerManagerService.nativeSetAutoSuspend</strong></p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy81ODUxMjU2LTVhMzc4NjM3MDdlNDU4YmIucG5n?x-oss-process&#61;image/format,png" alt="img" /></p> 
<p>Native 方法执行时间过长</p> 
<h2><a id="6Zygote_Fork__148"></a>6.Zygote Fork 进程时卡住</h2> 
<p>线程状态<strong>Native</strong>&#xff0c;查看是否有<br /> <strong>Process.zygoteSendArgsAndGetResult</strong></p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy81ODUxMjU2LWRmZDQ5OWUzNjhkYTA0OTAucG5n?x-oss-process&#61;image/format,png" alt="img" /></p> 
<p>Zygote Fork 进程时卡住</p> 
<h2><a id="7Dump__159"></a>7.Dump 时间过长</h2> 
<p><code>Dump</code> 超过<strong>60s</strong> 可能会引起手机重启。<br /> 搜索<code>关键字</code><br /> <strong>dumpStackTraces</strong><br /> 或<br /> <strong>dumpStackTraces process</strong></p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy81ODUxMjU2LTU1NGZjZWNjN2I5ZDA3YjgucG5n?x-oss-process&#61;image/format,png" alt="img" /></p> 
<p>Dump 时间过长</p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy81ODUxMjU2LTI1MjMzNDc3MzRkMjg1MTcucG5n?x-oss-process&#61;image/format,png" alt="img" /></p> 
<p>前面有ANR 发生</p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy81ODUxMjU2LTkxNjM0M2FlYzlkZDlhZmMucG5n?x-oss-process&#61;image/format,png" alt="img" /></p> 
<p>前面有ANR 发生</p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy81ODUxMjU2LWFjMmYzYmRjNmE1NjdkMmQucG5n?x-oss-process&#61;image/format,png" alt="img" /></p> 
<p>前面有fatal JE NE KE 等Exception发生</p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy81ODUxMjU2LWU1NjFjNWVhN2YxMDQ3OTEucG5n?x-oss-process&#61;image/format,png" alt="img" /></p> 
<p>自动化测试脚本有call dumpsys 去dump 系统信息</p> 
<h1><a id="_Android_O_Log__197"></a>六、 Android O以上导 Log 注意事项</h1> 
<p><code>Android O</code> 以上的 <code>mtklog</code> 和<code>db</code> 不在同一个目录&#xff0c;需要执行以下<code>adb</code>命令 导<code>Log</code>.</p> 
<pre><code class="prism language-bash">//1. 导 MTK log 
adb pull /sdcard/mtklog
//2. 导 AEE log&#xff0c;如果没有&#xff0c;请执行第3步
 adb pull /data/aee_exp
//3.导 data 下MTK缓存 的aee log
 adb pull /data/vendor/mtklog/aee_exp
</code></pre>