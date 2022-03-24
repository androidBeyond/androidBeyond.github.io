---
layout:     post
title:      Android SWT类问题分析总结
subtitle:   SWT(Software Watch Dog ) 主要用来监控SystemServer等重要线程/Service 的运行情况。如果发现其阻塞超过 60s ,看门狗进程就会把系统重启，进而保证系统可以恢复到正常状态。
date:       2021-12-08
author:     coderman
header-img: img/article-bg.jpg
top: false
catalog: true 
tags:
    - Android
	- SWT
---

<article class="baidu_pl">   
 <h1>一、 SWT 手机重启问题简介</h1>
 <p>SWT(Software Watch Dog ) 主要用来监控<code>SystemServer</code>等<code>重要线程/Service</code> 的运行情况。如果发现其阻塞超过 60s ,看门狗进程就会把系统重启&#xff0c;进而保证系统可以恢复到正常状态。</p>
 <p>判断阻塞的方法&#xff1a;</p>
 <ul class="list-paddingleft-2"><li><p>1.利用 Services 注册monitor 去Check</p></li></ul>
 <p>主要是&#xff1a; AMS、 Foreground Thread</p>
 <ul class="list-paddingleft-2"><li><p></p></li></ul>
 <ol start="2" class="list-paddingleft-2"><li><p>发送handler 到重要的Loop 线程来Check 是否阻塞。</p></li></ol>
 <p>主要是&#xff1a; Main Thread、UI Thread、IO Thread、Display Thread、WMS 、Other Services。</p>
 <p>SWT 判断阻塞的方法 图文描述如下&#xff1a;</p>
 <p><img src="https://img-blog.csdnimg.cn/img_convert/ac4f6fa3a87d2c100da7dc6e6ed20e59.png" alt="640?wx_fmt&#61;other" /></p>
 <p>SWT 判断阻塞的方法</p>
 <h1>二、 SWT 手机重启问题处理流程</h1>
 <p>SWT 处理流程&#xff1a;1.每半分钟check 一次system_server 进程&#xff1a;<code>dump</code> 一次<code>system_server</code> 的<code>backtrace</code></p>
 <p>2.一分钟卡住后kill&#xff0c;并重新计数&#xff1a;<code>dump</code>&#xff0c;并<code>kill</code>掉 <code>system_server</code>进程 &#xff0c;否则重新计时。</p>
 <p>3.SWT 处理大致流程如下&#xff1a;</p>
 <p><img src="https://img-blog.csdnimg.cn/img_convert/bff87a15b02bfe3c5b5f7a934b0e17f2.png" alt="640?wx_fmt&#61;other" /></p>
 <p>SWT 处理流程</p>
 <h1>三、 SWT 手机重启问题的原因</h1>
 <p>导致 <code>SWT</code>重启原因的原因有很多种。</p>
 <p>主要导致的原因如下&#xff1a;</p>
 <p><img src="https://img-blog.csdnimg.cn/img_convert/5a45aa37538ec64c9a85b78fbb7c5803.png" alt="640?wx_fmt&#61;other" /></p>
 <p>检查SWT 原因分类</p>
 <h1>四、 SWT 手机重启问题分析流程</h1>
 <p>首先搜索关键 watchdog&#xff0c;查看是否有重启发生。</p>
 <p><img src="https://img-blog.csdnimg.cn/img_convert/445ed1389a99f484ed17aea0e1d73e03.png" alt="640?wx_fmt&#61;other" /></p>
 <p>SWT 流程分析</p>
 <h1>五、SWT 手机重启问题分析举例</h1>
 <h2>1.分析 trace &#xff0c;确认线程关系</h2>
 <p>线程被 Block 搜索关键字 held by</p>
 <p><img src="https://img-blog.csdnimg.cn/img_convert/9ad979d96b642f5a9f0890752d8e1a5b.png" alt="640?wx_fmt&#61;other" /></p>
 <p>确认线程关系</p>
 <p>线程被 Waiting 结合代码分析。</p>
 <p><img src="https://img-blog.csdnimg.cn/img_convert/58c8873f63f604b250b69b4a77c6fdcf.png" alt="640?wx_fmt&#61;other" /></p>
 <p>确认线程关系</p>
 <h2>2.线程死锁</h2>
 <p>确认Block的线程是否有闭环的死锁关系。</p>
 <p><img src="https://img-blog.csdnimg.cn/img_convert/5849e3621dc7811b21e5322ef405327b.png" alt="640?wx_fmt&#61;other" /></p>
 <p>线程死锁</p>
 <p><img src="https://img-blog.csdnimg.cn/img_convert/9b55aba294e660023c1c3835b1fa69c0.png" alt="640?wx_fmt&#61;other" /></p>
 <p>线程死锁</p>
 <h2>3.Binder的Server 端卡住</h2>
 <p>线程状态 Native&#xff0c;并且callstack中含有一对</p>
 <p>IPCThreadState::waitForResponseIPCThreadState::talkWithDriver</p>
 <p><img src="https://img-blog.csdnimg.cn/img_convert/52e3913eca5f0738803a45027d0a687d.png" alt="640?wx_fmt&#61;other" /></p>
 <p>Bind的Server端卡住</p>
 <p><img src="https://img-blog.csdnimg.cn/img_convert/15fb89bee1f210dbad911054388c087a.png" alt="640?wx_fmt&#61;other" /></p>
 <p>Bind的Server端卡住</p>
 <h2>4.SurfaceFlinger 卡住导致重启</h2>
 <p>搜索<code>关键字</code> I watchdog ,surfaceflinger hang&#xff0c;默认卡住<code>40s</code>&#xff0c;就会重启。</p>
 <p><img src="https://img-blog.csdnimg.cn/img_convert/49bc56dc92728a0e12a42b8ccb9faf58.png" alt="640?wx_fmt&#61;other" /></p>
 <p>SurfaceFlinger 卡住</p>
 <h2>5.Native 方法执行时间过长导致重启</h2>
 <p>线程状态 Native&#xff0c;查看是否有PowerManagerService.nativeSetAutoSuspend</p>
 <p><img src="https://img-blog.csdnimg.cn/img_convert/fb43b37ed5a28ef0a4a26c892da90394.png" alt="640?wx_fmt&#61;other" /></p>
 <p>Native 方法执行时间过长</p>
 <h2>6.Zygote Fork 进程时卡住</h2>
 <p>线程状态Native&#xff0c;查看是否有Process.zygoteSendArgsAndGetResult</p>
 <p><img src="https://img-blog.csdnimg.cn/img_convert/10595da5d53e2247bb1a7b1011367dd1.png" alt="640?wx_fmt&#61;other" /></p>
 <p>Zygote Fork 进程时卡住</p>
 <h2>7.Dump 时间过长</h2>
 <p><code>Dump</code> 超过60s 可能会引起手机重启。<code>关键字</code>dumpStackTracesdumpStackTraces process</p>
 <p><img src="https://img-blog.csdnimg.cn/img_convert/c9b41e9ec4802ded25faf3a1dbf8ebbd.png" alt="640?wx_fmt&#61;other" /></p>
 <p>Dump 时间过长</p>
 <p><img src="https://img-blog.csdnimg.cn/img_convert/57d8f1e74d4e6b9ab2307d8c45372954.png" alt="640?wx_fmt&#61;other" /></p>
 <p>前面有ANR 发生</p>
 <p><img src="https://img-blog.csdnimg.cn/img_convert/0d9aa967112f793ae5a6bbdc644f7247.png" alt="640?wx_fmt&#61;other" /></p>
 <p>前面有ANR 发生</p>
 <p><img src="https://img-blog.csdnimg.cn/img_convert/72123e10a440f1f1fd39901708d973a5.png" alt="640?wx_fmt&#61;other" /></p>
 <p>前面有fatal JE NE KE 等Exception发生</p>
 <p><img src="https://img-blog.csdnimg.cn/img_convert/e28e34920472208728dae0f0dfc5fd3e.png" alt="640?wx_fmt&#61;other" /></p>
 <p>自动化测试脚本有call dumpsys 去dump 系统信息</p>
 <h1>六、 Android O以上导 Log 注意事项</h1>
 <p><code>Android O</code> 以上的 <code>mtklog</code> 和<code>db</code> 不在同一个目录&#xff0c;需要执行以下<code>adb</code>命令 导<code>Log</code>.</p>
 <pre class="has"><code class="language-javascript">//1. 导 MTK log 
adb pull /sdcard/mtklog
//2. 导 AEE log&#xff0c;如果没有&#xff0c;请执行第3步
 adb pull /data/aee_exp
//3.导 data 下MTK缓存 的aee log
 adb pull /data/vendor/mtklog/aee_exp</code></pre>
 <p><img src="https://img-blog.csdnimg.cn/img_convert/c9046aed6e0428c13f510b4bf32a6fa2.png" alt="640?wx_fmt&#61;jpeg" /></p>
 <p><img src="https://img-blog.csdnimg.cn/img_convert/7d8a5b7f794217014ff3526a4a860536.png" alt="640?wx_fmt&#61;jpeg" /></p>
 <p>长按识别二维码&#xff0c;领福利</p>
 <p>至此&#xff0c;本篇已结束&#xff0c;如有不对的地方&#xff0c;欢迎您的建议与指正。同时期待您的关注&#xff0c;感谢您的阅读&#xff0c;谢谢&#xff01;</p>
 <p>如有侵权&#xff0c;请联系小编&#xff0c;小编对此深感抱歉&#xff0c;届时小编会删除文章&#xff0c;立即停止侵权行为&#xff0c;请您多多包涵。<img class="rich_pages" src="https://img-blog.csdnimg.cn/img_convert/729bc616ee8d39d368da134cd2572cb4.gif" alt="640?wx_fmt&#61;gif" /></p> 
</div>
                </div>
        </div>
        <div id="treeSkill"></div>
    </article>