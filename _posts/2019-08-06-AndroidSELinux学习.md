---
layout:     post
title:      Android SELinux学习
subtitle:   SELinux粗浅学习
date:       2019-08-06
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android
    - framework
---

<article class="baidu_pl">
        <div id="article_content" class="article_content clearfix">
        <link rel="stylesheet" href="https://csdnimg.cn/release/blogv2/dist/mdeditor/css/editerView/ck_htmledit_views-1a85854398.css">
                <div id="content_views" class="markdown_views prism-atom-one-dark">
                    <svg xmlns="http://www.w3.org/2000/svg" style="display: none;">
                        <path stroke-linecap="round" d="M5,0 0,2.5 5,5z" id="raphael-marker-block" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path>
                    </svg>
                    <h2 id="1-selinux-背景知识">1. SELinux 背景知识</h2> 
<p>详细了解 SELinux 可以参阅 <a href="https://source.android.com/security/selinux/images/SELinux_Treble.pdf">Google 官方文档</a></p> 
<h3 id="11-dac-与-mac">1.1 DAC 与 MAC</h3> 
<p>在 SELinux 出现之前&#xff0c;Linux 上的安全模型叫 DAC&#xff0c;全称是 Discretionary Access Control&#xff0c;翻译为自主访问控制。</p> 
<p>DAC 的核心思想很简单&#xff0c;就是&#xff1a;进程理论上所拥有的权限与执行它的用户的权限相同。比如&#xff0c;以 root 用户启动 Browser&#xff0c;那么 Browser 就有 root 用户的权限&#xff0c;在 Linux 系统上能干任何事情。</p> 
<p>显然&#xff0c;DAD 管理太过宽松&#xff0c;只要想办法在 Android 系统上获取到 root 权限就可以了。那么 SELinux 是怎么解决这个问题呢&#xff1f;在 DAC 之外&#xff0c;它设计了一种新的安全模型&#xff0c;叫 MAC&#xff08;Mandatory Access Control&#xff09;&#xff0c;翻译为强制访问控制。</p> 
<p>MAC 的理论也很简单&#xff0c;任何进程想在 SELinux 系统上干任何事情&#xff0c;都必须在《安全策略文件》中赋予权限&#xff0c;凡是没有出现在安全策略文件中的权限&#xff0c;就不行。</p> 
<p>关于 DAC 和 MAC&#xff0c;可以总结几个知识点&#xff1a; <br /> 1. Linux 系统先做 DAC 检查。如果没有通过 DAC 权限检查&#xff0c;则操作直接失败。通过 DAC 检查之后&#xff0c;再做 MAC 权限检查 <br /> 2. SELinux 有自己的一套规则来编写安全策略文件&#xff0c;这套规则被称之为 SELinux Policy 语言。</p> 
<h3 id="12-sepolicy-语言">1.2 SEPolicy 语言</h3> 
<p>Linux中有两种东西&#xff0c;一种死的&#xff08;Inactive&#xff09;&#xff0c;一种活的&#xff08;Active&#xff09;。死的东西就是文件&#xff08;Linux哲学&#xff0c;万物皆文件。注意&#xff0c;万不可狭义解释为File&#xff09;&#xff0c;而活的东西就是进程。此处的 死 和 活 是一种比喻&#xff0c;映射到软件层面的意思是&#xff1a;进程能发起动作&#xff0c;例如它能打开文件并操作它。而文件只能被进程操作。</p> 
<blockquote> 
 <p>根据 SELinux 规范&#xff0c;完整的 Secure Context 字符串为&#xff1a;user:role:type[:range]</p> 
</blockquote> 
<h4 id="121-进程的-secure-context">1.2.1 进程的 Secure Context</h4> 
<p>在 SELinux 中&#xff0c;每种东西都会被赋予一个安全属性&#xff0c;官方说法叫做 Security Context&#xff0c;Security Context 是一个字符串&#xff0c;主要由三个部分组成&#xff0c;例如 SEAndroid 中&#xff0c;进程的 Security Context 可通过 <strong>ps -Z</strong> 命令查看&#xff1a;</p> 
<pre class="prettyprint"><code class=" hljs avrasm"><span class="hljs-label">rk3288:</span>/ $ ps -AZ
<span class="hljs-label">u:</span>r:hal_wifi_supplicant_default:s0 wifi      <span class="hljs-number">1816</span>     <span class="hljs-number">1</span>   <span class="hljs-number">11388</span>   <span class="hljs-number">6972</span> <span class="hljs-number">0</span>                   <span class="hljs-number">0</span> S wpa_supplicant
<span class="hljs-label">u:</span>r:platform_app:s0:c512,c768  u0_a14        <span class="hljs-number">1388</span>   <span class="hljs-number">228</span> <span class="hljs-number">1612844</span>  <span class="hljs-number">57396</span> <span class="hljs-number">0</span>                   <span class="hljs-number">0</span> S android<span class="hljs-preprocessor">.ext</span><span class="hljs-preprocessor">.services</span>
<span class="hljs-label">u:</span>r:system_app:s0              system        <span class="hljs-number">1531</span>   <span class="hljs-number">228</span> <span class="hljs-number">1669680</span> <span class="hljs-number">119364</span> <span class="hljs-number">0</span>                   <span class="hljs-number">0</span> S <span class="hljs-keyword">com</span><span class="hljs-preprocessor">.android</span><span class="hljs-preprocessor">.gallery</span>3d
<span class="hljs-label">u:</span>r:kernel:s0                  root           <span class="hljs-number">582</span>     <span class="hljs-number">2</span>       <span class="hljs-number">0</span>      <span class="hljs-number">0</span> <span class="hljs-number">0</span>                   <span class="hljs-number">0</span> S [kworker/<span class="hljs-number">1</span>:<span class="hljs-number">2</span>]
<span class="hljs-label">u:</span>r:radio:s0                   radio          <span class="hljs-number">594</span>   <span class="hljs-number">228</span> <span class="hljs-number">1634876</span>  <span class="hljs-number">89296</span> <span class="hljs-number">0</span>                   <span class="hljs-number">0</span> S <span class="hljs-keyword">com</span><span class="hljs-preprocessor">.android</span><span class="hljs-preprocessor">.phone</span>
<span class="hljs-label">u:</span>r:system_app:s0              system         <span class="hljs-number">672</span>   <span class="hljs-number">228</span> <span class="hljs-number">1686204</span> <span class="hljs-number">141716</span> <span class="hljs-number">0</span>                   <span class="hljs-number">0</span> S <span class="hljs-keyword">com</span><span class="hljs-preprocessor">.android</span><span class="hljs-preprocessor">.settings</span>
<span class="hljs-label">u:</span>r:platform_app:s0:c512,c768  u0_a18         <span class="hljs-number">522</span>   <span class="hljs-number">223</span> <span class="hljs-number">1721656</span> <span class="hljs-number">152116</span> <span class="hljs-number">0</span>                   <span class="hljs-number">0</span> S <span class="hljs-keyword">com</span><span class="hljs-preprocessor">.android</span><span class="hljs-preprocessor">.systemui</span></code></pre> 
<p>上面的最左边的一列就是进程的 Security Context&#xff0c;以第一个进程 wpa_supplicant 为例</p> 
<pre class="prettyprint"><code class=" hljs css"><span class="hljs-tag">u</span><span class="hljs-pseudo">:r</span><span class="hljs-pseudo">:hal_wifi_supplicant_default</span><span class="hljs-pseudo">:s0</span></code></pre> 
<p>其中 <br /> - u 为 user 的意思&#xff0c;SEAndroid 中定义了一个 SELinux 用户&#xff0c;值为 u <br /> - r 为 role 的意思&#xff0c;role 是角色之意&#xff0c;它是 SELinux 中一个比较高层次&#xff0c;更方便的权限管理思路。简单点说&#xff0c;一个 u 可以属于多个 role&#xff0c;不同的 role 具有不同的权限。 <br /> - hal_wifi_supplicant_default 代表该进程所属的 Domain 为 hal_wifi_supplicant_default。MAC&#xff08;Mandatory Access Control&#xff09;强制访问控制 的基础管理思路其实是 Type Enforcement Access Control&#xff08;简称TEAC&#xff0c;一般用TE表示&#xff09;&#xff0c;对进程来说&#xff0c;Type 就是 Domain&#xff0c;比如 hal_wifi_supplicant_default 需要什么权限&#xff0c;都需要通过 allow 语句在 te 文件中进行说明。 <br /> - s0 是 SELinux 为了满足军用和教育行业而设计的 Multi-Level Security&#xff08;MLS&#xff09;机制有关。简单点说&#xff0c;MLS 将系统的进程和文件进行了分级&#xff0c;不同级别的资源需要对应级别的进程才能访问</p> 
<h4 id="122-文件的-secure-context">1.2.2 文件的 Secure Context</h4> 
<p>文件的 Secure Context 可以通过 <strong>ls -Z</strong> 来查看&#xff0c;如下</p> 
<pre class="prettyprint"><code class=" hljs avrasm"><span class="hljs-label">rk3288:</span>/vendor/lib $ ls libOMX_Core<span class="hljs-preprocessor">.so</span> -<span class="hljs-built_in">Z</span>
<span class="hljs-label">u:</span>object_r:vendor_file:s0 libOMX_Core<span class="hljs-preprocessor">.so</span></code></pre> 
<ul><li>u&#xff1a;同样是 user 之意&#xff0c;它代表创建这个文件的 SELinux user</li><li>object_r&#xff1a;文件是死的东西&#xff0c;它没法扮演角色&#xff0c;所以在 SELinux 中&#xff0c;死的东西都用 object_r 来表示它的 role</li><li>vendor_file&#xff1a;type&#xff0c;和进程的 Domain 是一个意思&#xff0c;它表示 libOMX_Core.so 文件所属的 Type 是 vendor_file</li><li>s0&#xff1a;MLS 的等级</li></ul> 
<h3 id="13-te-介绍">1.3 TE 介绍</h3> 
<p>MAC 基本管理单位是 TEAC&#xff08;Type Enforcement Accesc Control&#xff09;&#xff0c;然后是高一级别的 Role Based Accesc Control。RBAC 是基于 TE 的&#xff0c;而 TE 也是 SELinux 中最主要的部分。上面说的 allow 语句就是 TE 的范畴。</p> 
<p>根据 SELinux 规范&#xff0c;完整的 SELinux 策略规则语句格式为&#xff1a;</p> 
<pre class="prettyprint"><code class=" hljs asciidoc">allow domains types:classes permissions;

<span class="hljs-bullet">- </span>Domain - 一个进程或一组进程的标签。也称为域类型&#xff0c;因为它只是指进程的类型。
<span class="hljs-bullet">- </span>Type - 一个对象&#xff08;例如&#xff0c;文件、套接字&#xff09;或一组对象的标签。
<span class="hljs-bullet">- </span>Class - 要访问的对象&#xff08;例如&#xff0c;文件、套接字&#xff09;的类型。
<span class="hljs-bullet">- </span>Permission - 要执行的操作&#xff08;例如&#xff0c;读取、写入&#xff09;。

<span class="hljs-header">&#61; allow &#xff1a; 允许主体对客体进行操作</span>
<span class="hljs-header">&#61; neverallow &#xff1a;拒绝主体对客体进行操作</span>
<span class="hljs-header">&#61; dontaudit &#xff1a; 表示不记录某条违反规则的决策信息</span>
<span class="hljs-header">&#61; auditallow &#xff1a;记录某项决策信息&#xff0c;通常 SElinux 只记录失败的信息&#xff0c;应用这条规则后会记录成功的决策信息。</span>
</code></pre> 
<p>使用政策规则时将遵循的结构示例&#xff1a;</p> 
<pre class="prettyprint"><code class=" hljs css">语句&#xff1a;
<span class="hljs-tag">allow</span> <span class="hljs-tag">appdomain</span> <span class="hljs-tag">app_data_file</span><span class="hljs-pseudo">:file</span> <span class="hljs-tag">rw_file_perms</span>;

这表示所有应用域都可以读取和写入带有 <span class="hljs-tag">app_data_file</span> 标签的文件</code></pre> 
<h5 id="相关实例">相关实例</h5> 
<pre class="prettyprint"><code class=" hljs bash"><span class="hljs-number">1</span>. SEAndroid 中的安全策略文件 policy.conf
<span class="hljs-comment"># 允许 zygote 域中的进程向 init 域中的进程&#xff08;Object Class 为 process&#xff09;发送 sigchld 信号</span>

allow zygote init:process sigchld;

<span class="hljs-number">2</span>. <span class="hljs-comment"># 允许 zygote 域中的进程 search 或 getattr 类型为 appdomain 的目录。</span>
   <span class="hljs-comment"># 注意&#xff0c;多个 perm_set 可用 {} 括起来</span>
   allow zygote appdomain:dir { getattr search };

<span class="hljs-number">3</span>. <span class="hljs-comment"># perm_set 语法比较奇特&#xff0c;前面有一个 ~ 号。</span>
   <span class="hljs-comment"># 它表示除了{entrypoint relabelto}之外&#xff0c;{chr_file #file}这两个object_class所拥有的其他操作 </span>
  allow unconfineddomain {fs_<span class="hljs-built_in">type</span> dev_<span class="hljs-built_in">type</span> file_<span class="hljs-built_in">type</span>}:{ chr_file file }   \
  ~{entrypoint relabelto};</code></pre> 
<h5 id="object-class-类型">Object Class 类型</h5> 
<pre class="prettyprint"><code class=" hljs cs">文件路径&#xff1a; system/sepolicy/<span class="hljs-keyword">private</span>/security_classes

# file-related classes
<span class="hljs-keyword">class</span> filesystem
<span class="hljs-keyword">class</span> file  #代表普通文件
<span class="hljs-keyword">class</span> dir   #代表目录
<span class="hljs-keyword">class</span> fd    #代表文件描述符
<span class="hljs-keyword">class</span> lnk_file  #代表链接文件
<span class="hljs-keyword">class</span> chr_file  #代表字符设备文件

# network-related classes
<span class="hljs-keyword">class</span> socket   #socket
<span class="hljs-keyword">class</span> tcp_socket
<span class="hljs-keyword">class</span> udp_socket

......
<span class="hljs-keyword">class</span> binder   #Android 平台特有的 binder
<span class="hljs-keyword">class</span> zygote   #Android 平台特有的 zygote</code></pre> 
<h5 id="perm-set-类型">Perm Set 类型</h5> 
<p>Perm Set 指得是某种 Object class 所拥有的权限。以 file 这种 Object class 而言&#xff0c;其拥有的 Perm Set 就包括 read、write、open、create、execute等。</p> 
<p>和 Object Class 一样&#xff0c;SELinux 或 SEAndroid 所支持的 Perm set 也需要声明&#xff1a;</p> 
<pre class="prettyprint"><code class=" hljs livecodeserver">文件路径&#xff1a; <span class="hljs-keyword">system</span>/sepolicy/<span class="hljs-keyword">private</span>/access_vectors</code></pre> 
<h2 id="2-selinux-相关设置">2. SELinux 相关设置</h2> 
<h3 id="21-强制执行等级">2.1 强制执行等级</h3> 
<p>熟悉以下术语&#xff0c;了解如何按不同的强制执行级别实现 SELinux <br /> - 宽容模式&#xff08;permissive&#xff09; - 仅记录但不强制执行 SELinux 安全政策。 <br /> - 强制模式&#xff08;enforcing&#xff09; - 强制执行并记录安全政策。如果失败&#xff0c;则显示为 EPERM 错误。</p> 
<h3 id="22-关闭-selinux">2.2 关闭 SELinux</h3> 
<h5 id="临时关闭">临时关闭</h5> 
<p>&#xff08;1&#xff09; setenforce </p> 
<pre class="prettyprint"><code class=" hljs ruby"><span class="hljs-variable">$ </span>setenforce <span class="hljs-number">0</span></code></pre> 
<p>setenforce 命令修改的是 /sys/fs/selinux/enforce 节点的值&#xff0c;是 kernel 意义上的修改 selinux 的策略。断电之后&#xff0c;节点值会复位</p> 
<h5 id="永久关闭">永久关闭</h5> 
<p>&#xff08;2&#xff09; kernel 关闭 selinux</p> 
<pre class="prettyprint"><code class=" hljs bash">SECURITY_SELINUX 设置为 <span class="hljs-literal">false</span>&#xff0c;重新编译 kernel</code></pre> 
<p>&#xff08;3&#xff09; 设置 ro.boot.selinux&#61;permissive 属性&#xff0c;并且修改在 system/core/init/Android.mk 中设置用于 user 版本下 selinux 模式为 permissive</p> 
<pre class="prettyprint"><code class=" hljs haml">ifneq (,$(filter userdebug eng,$(TARGET_BUILD_VARIANT)))
init_options &#43;&#61; \
    -<span class="ruby"><span class="hljs-constant">DALLOW_LOCAL_PROP_OVERRIDE</span>&#61;<span class="hljs-number">1</span> \
</span>    -<span class="ruby"><span class="hljs-constant">DALLOW_PERMISSIVE_SELINUX</span>&#61;<span class="hljs-number">1</span> \                                                                                              
</span>    -<span class="ruby"><span class="hljs-constant">DREBOOT_BOOTLOADER_ON_PANIC</span>&#61;<span class="hljs-number">1</span> \
</span>    -<span class="ruby"><span class="hljs-constant">DDUMP_ON_UMOUNT_FAILURE</span>&#61;<span class="hljs-number">1</span>
</span>else
init_options &#43;&#61; \
    -<span class="ruby"><span class="hljs-constant">DALLOW_LOCAL_PROP_OVERRIDE</span>&#61;<span class="hljs-number">0</span> \
</span>    -<span class="ruby"><span class="hljs-constant">DALLOW_PERMISSIVE_SELINUX</span>&#61;<span class="hljs-number">1</span> \ /<span class="hljs-regexp">/ 修改为1&#xff0c;表示允许 selinux 为 permissive
</span></span>    -<span class="ruby"><span class="hljs-constant">DREBOOT_BOOTLOADER_ON_PANIC</span>&#61;<span class="hljs-number">0</span> \
</span>    -<span class="ruby"><span class="hljs-constant">DDUMP_ON_UMOUNT_FAILURE</span>&#61;<span class="hljs-number">0</span>
</span>endif
</code></pre> 
<h3 id="23-selinux-权限不足-avc-denied-问题解决">2.3 SELinux 权限不足 avc-denied 问题解决</h3> 
<p>目前所有的 SELinux 权限检测失败&#xff0c;在 Kernel Log 或者 Android Log 中都有对应的 avc-denied Log 与之对应。反过来&#xff0c;有 avc-denied Log&#xff0c;并非就会直接失败&#xff0c;还需要确认当时 SELinux 的模式, 是 Enforcing 还是 Permissve。</p> 
<p>如果是 Enforcing 模式&#xff0c;就要检测对应的进程访问权限资源是否正常&#xff1f;是否有必要添加&#xff1f; 如果有必要添加&#xff0c;可以找下面的方式生成需要的 sepolicy 规则并添加到对应 te 文件。</p> 
<p><strong>使用 audit2allow 简化方法</strong> <br /> 1. 从 logcat 或串口中提取相应的 avc-denied log&#xff0c;下面的语句为提取所有的 avc- denied log</p> 
<pre class="prettyprint"><code class=" hljs ruby"><span class="hljs-variable">$ </span>adb shell <span class="hljs-string">&#34;cat /proc/kmsg | grep avc&#34;</span> &gt; avc_log.txt </code></pre> 
<ol><li>使用 audit2allow 工具生成对应的 policy 规则</li></ol> 
<pre class="prettyprint"><code class=" hljs ruby">/<span class="hljs-regexp">/ audio2allow 使用必须先 source build/envsetup</span>.sh&#xff0c;导入环境变量
<span class="hljs-variable">$ </span>audit2allow -i avc_log.txt -p <span class="hljs-variable">$OUT</span>/vendor/etc/selinux/precompiled_sepolicy
<span class="hljs-number">3</span>. 将对应的policy 添加到 te 文件中
    &#61; 一般添加在 /device/&lt;company&gt;<span class="hljs-regexp">/common/sepolicy</span> 或者 /device/&lt;company&gt;<span class="hljs-regexp">/$DEVICE/sepolicy</span> 目录下</code></pre> 
<blockquote> 
 <p>BOARD_SEPOLICY_DIRS &#43;&#61; device/$SoC/common/sepolicy 通过这个命令添加厂家自定义的 sepolicy 规则</p> 
</blockquote> 
<h2 id="3-seandroid-安全机制框架">3. SEAndroid 安全机制框架</h2> 
<blockquote> 
 <p>SELinux 系统比起通常的 Linux 系统来&#xff0c;安全性能要高的多&#xff0c;它通过对于用户&#xff0c;进程权限的最小化&#xff0c;即使受到攻击&#xff0c;进程或者用户权限被夺去&#xff0c;也不会对整个系统造成重大影响。</p> 
</blockquote> 
<p>我们知道&#xff0c;Android 系统是基于 Linux 内核实现&#xff0c;针对 Linux 系统&#xff0c;NSA 开发了一套安全机制 SELinux&#xff0c;用来加强安全性。然而&#xff0c;由于 Android 系统有着独特的用户空间运行时&#xff0c;因此 SELinux 不能完全适用于 Android 系统。为此&#xff0c;NSA 针对 Android 系统&#xff0c;在 SELinux 基础上开发了 SEAndroid。</p> 
<p>SEAndroid 安全机制所要保护的对象是系统中的资源&#xff0c;这些资源分布在各个子系统中&#xff0c;例如我们经常接触的文件就是分布文件子系统中的。实际上&#xff0c;系统中需要保护的资源非常多&#xff0c;除了前面说的文件之外&#xff0c;还有进程、socket 和 IPC 等等。对于 Android 系统来说&#xff0c;由于使用了与传统 Linux 系统不一样的用户空间运行时&#xff0c;即应用程序运行时框架&#xff0c;因此它在用户空间有一些特有的资源是需要特别保护的&#xff0c;例如系统属性的设置等。</p> 
<h3 id="31-seandroid-框架流程">3.1 SEAndroid 框架流程</h3> 
<p>SEAndroid 安全机制的整体框架&#xff0c;可以使用下图来概括&#xff1a;</p> 
<p><img src="https://gitee.com/kevin1993175/image_resource/raw/master/SEAndroid_logic.png" alt="SEAndroid_frame" title="" /></p> 
<p>以 SELinux 文件系统接口问边界&#xff0c;SEAndroid 安全机制包含内核空间和用户空间两部分支持。</p> 
<p>这些内核空间模块与用户空间模块空间的作用及交互有&#xff1a;</p> 
<pre class="prettyprint"><code class=" hljs markdown"><span class="hljs-bullet">1. </span>内核空间的 SELinux LSM 模块负责内核资源的安全访问控制
<span class="hljs-bullet">2. </span>用户空间的 SEAndroid Policy 描述的是资源安全访问策略。
   系统在启动的时候&#xff0c;用户空间的 Security Server 需要将这些安全访问策略加载内核空间的 SELinux LSM 模块中去。
   这是通过SELinux文件系统接口实现的
<span class="hljs-bullet">3. </span>用户空间的 Security Context 描述的是资源安全上下文。
   SEAndroid 的安全访问策略就是在资源的安全上下文基础上实现的
<span class="hljs-bullet">4. </span>用户空间的 Security Server 一方面需要到用户空间的 Security Context 去检索对象的安全上下文&#xff0c;
   另一方面也需要到内核空间去操作对象的安全上下文
<span class="hljs-bullet">5. </span>用户空间的 libselinux 库封装了对 SELinux 文件系统接口的读写操作。
   用户空间的 Security Server 访问内核空间的 SELinux LSM 模块时&#xff0c;都是间接地通过 libselinux进行的。
   这样可以将对 SELinux 文件系统接口的读写操作封装成更有意义的函数调用。
<span class="hljs-bullet">6. </span>用户空间的 Security Server 到用户空间的 Security Context 去检索对象的安全上下文时&#xff0c;同样也是通过 selinux 库来进行的</code></pre> 
<h4 id="311-内核空间">3.1.1 内核空间</h4> 
<ol><li>在内核空间&#xff0c;存在一个 SELinux LSM&#xff08;Linux Secrity Moudle&#xff09;模块&#xff0c;&#xff08;用 MAC 强制访问控制&#xff09;负责资源的安全访问控制。</li><li>LSM 模块中包含一个访问向量缓冲&#xff08;Access Vector Cache&#xff09;和一个安全服务&#xff08;Security Server&#xff09;。Security Server 负责安全访问控制逻辑&#xff0c;即由它来决定一个主体访问一个客体是否是合法的&#xff0c;这个主体一般是指进程&#xff0c;而客体主要指资源&#xff0c;例如文件。</li><li>SELinux、LSM 和内核中的子系统是如何交互的呢&#xff1f;首先&#xff0c;SELinux 会在 LSM 中注册相应的回调函数。其次&#xff0c;LSM 会在相应的内核对象子系统中会加入一些 Hook 代码。例如&#xff0c;我们调用系统接口 read 函数来读取一个文件的时候&#xff0c;就会进入到内核的文件子系统中。在文件子系统中负责读取文件函数 vfs_read 就会调用 LSM 加入的 Hook 代码。这些 Hook 代码就会调用之前 SELinux 注册进来的回调函数&#xff0c;以便后者可以进行安全检查。</li><li>SELinux 在进行安全检查的时候&#xff0c;首先是看一下自己的 Access Vector Cache 是否已经有结果。如果有的话&#xff0c;就直接将结果返回给相应的内核子系统就可以了。如果没有的话&#xff0c;就需要到 Security Server 中去进行检查。检查出来的结果在返回给相应的内核子系统的同时&#xff0c;也会保存在自己的 Access Vector Cache 中&#xff0c;以便下次可以快速地得到检查结果</li></ol> 
<p>上面概述的安全访问控制流程&#xff0c;可以使用下图来总结&#xff1a;</p> 
<p><img src="https://gitee.com/kevin1993175/image_resource/raw/master/SELinux_check_logic.png" alt="SELinux_check_logic" title="" /></p> 
<pre class="prettyprint"><code class=" hljs markdown"><span class="hljs-bullet">1. </span>一般性错误检查&#xff0c;例如访问的对象是否存在、访问参数是否正确等
<span class="hljs-bullet">2. </span>DAC 检查&#xff0c;即基于 Linux UID/GID 的安全检查
<span class="hljs-bullet">3. </span>SELinux 检查&#xff0c;即基于安全上下文和安全策略的安全检查</code></pre> 
<h4 id="312-用户空间">3.1.2 用户空间</h4> 
<p>在用户空间&#xff0c;SEAndorid 主要包含三个模块&#xff0c;分别是安全上下文&#xff08;Security Context&#xff09;、安全策略&#xff08;SEAndroid Policy&#xff09;和安全服务&#xff08;Security Server&#xff09;。</p> 
<h5 id="1安全上下文">&#xff08;1&#xff09;安全上下文</h5> 
<p>前面已经描述过了&#xff0c;SEAndroid 是一种基于安全策略的 MAC 安全机制。这种安全策略又是建立在对象的安全上下文的基础上的&#xff0c;SEAndroid 中的对象分为主体&#xff08;Subject&#xff09;和客体&#xff08;Object&#xff09;&#xff0c;主体通常是指进程&#xff0c;而客体是指进程所要访问的资源&#xff0c;如文件、系统属性等</p> 
<p>安全上下文实际上就是一个附加在对象上的标签&#xff08;Tag&#xff09;。这个标签实际上就是一个字符串&#xff0c;它由四部分内容组成&#xff0c;分别是 SELinux 用户、SELinux 角色、类型、安全级别&#xff0c;每一个部分都通过一个冒号来分隔&#xff0c;格式为 “user:role:type:sensitivity”</p> 
<blockquote> 
 <p>用 ps -Z 来查看主体的安全上下文&#xff0c;而用 ls -Z 来查看客体的安全上下文。</p> 
</blockquote> 
<h5 id="安全上下文类型说明">安全上下文&lt;类型&gt;说明</h5> 
<p>通常将用来标注文件的安全上下文中的类型称为 file_type&#xff0c;用来标注进程的安全上下文的类型称为 domain。</p> 
<p>将两个类型相关联可以通过 type 语句实现&#xff0c;例如用来描述 init 进程安全策略文件 /system/sepolicy/public/init.te 文件中&#xff0c;使用 type 语句将 init 与 domain 相关联&#xff08;等价&#xff09;</p> 
<pre class="prettyprint"><code class=" hljs fsharp"><span class="hljs-class"><span class="hljs-keyword">type</span> <span class="hljs-title">init</span> <span class="hljs-title">domain</span>;</span>
<span class="hljs-comment">// 这样就可以表明 init 描述的类型是用来描述进程的安全上下文的</span></code></pre> 
<h5 id="android-系统中对象的安全上下文定义">Android 系统中对象的安全上下文定义</h5> 
<p>系统中各种类型对象&#xff08;包含主体和客体&#xff09;的安全上下文是在 system/sepolicy 工程中定义的&#xff0c;我们讨论四种类型对象的安全上下文&#xff0c;分别是 app 进程、app 数据文件、系统文件和系统属性。这四种类型对象的安全上下文通过四个文件来描述&#xff1a;mac_permissions.xml、seapp_contexts、file_contexts 和 property_contexts。</p> 
<p>mac_permissions 文件给不同签名的 app 分配不同的 seinfo 字符串&#xff0c;例如&#xff0c;在 AOSP 源码环境下编译并且使用平台签名的 App 获得的 seinfo 为 “platform”&#xff0c;使用第三方签名安装的 App 获得的 seinfo 签名为”default”。</p> 
<pre class="prettyprint"><code class=" hljs xml"><span class="hljs-comment">&lt;!-- Platform dev key in AOSP --&gt;</span>
<span class="hljs-tag">&lt;<span class="hljs-title">signer</span> <span class="hljs-attribute">signature</span>&#61;<span class="hljs-value">&#34;&#64;PLATFORM&#34;</span> &gt;</span>
  <span class="hljs-tag">&lt;<span class="hljs-title">seinfo</span> <span class="hljs-attribute">value</span>&#61;<span class="hljs-value">&#34;platform&#34;</span> /&gt;</span>
<span class="hljs-tag">&lt;/<span class="hljs-title">signer</span>&gt;</span>

<span class="hljs-comment">&lt;!-- Media key in AOSP --&gt;</span>
<span class="hljs-tag">&lt;<span class="hljs-title">signer</span> <span class="hljs-attribute">signature</span>&#61;<span class="hljs-value">&#34;&#64;MEDIA&#34;</span> &gt;</span>
  <span class="hljs-tag">&lt;<span class="hljs-title">seinfo</span> <span class="hljs-attribute">value</span>&#61;<span class="hljs-value">&#34;media&#34;</span> /&gt;</span>
<span class="hljs-tag">&lt;/<span class="hljs-title">signer</span>&gt;</span></code></pre> 
<p>这里的 seinfo 描述的并不是安全上下文的对象类型&#xff0c;它用来在 seapp_contexts 中查找对应的对象类型。在 seapp_contexts 对 seinfo 也就是平台签名为”platform”的 app 如下定义&#xff1a;</p> 
<pre class="prettyprint"><code class=" hljs rust">user&#61;<span class="hljs-number">_</span>app seinfo&#61;platform domain&#61;platform_app <span class="hljs-keyword">type</span>&#61;app_data_file levelFrom&#61;user

也就是说明使用<span class="hljs-string">&#34;platform&#34;</span>平台签名的 app 所运行的进程 domain 为<span class="hljs-string">&#34;platform_app&#34;</span>&#xff0c;
并且他的数据文件类型为<span class="hljs-string">&#34;app_data_file&#34;</span>。</code></pre> 
<p>接下来看一下系统文件的安全上下文是如何定义的&#xff1a;</p> 
<pre class="prettyprint"><code class=" hljs coffeescript"><span class="hljs-comment">######</span><span class="hljs-comment">######</span><span class="hljs-comment">######</span><span class="hljs-comment">######</span><span class="hljs-comment">######</span><span class="hljs-comment">######</span><span class="hljs-comment">######</span><span class="hljs-comment">#</span>
<span class="hljs-comment"># Root</span>
/                   <span class="hljs-attribute">u</span>:<span class="hljs-attribute">object_r</span>:<span class="hljs-attribute">rootfs</span>:s0

<span class="hljs-comment"># Data files</span>
/adb_keys           <span class="hljs-attribute">u</span>:<span class="hljs-attribute">object_r</span>:<span class="hljs-attribute">adb_keys_file</span>:s0
/build\.prop        <span class="hljs-attribute">u</span>:<span class="hljs-attribute">object_r</span>:<span class="hljs-attribute">rootfs</span>:s0
/<span class="hljs-reserved">default</span>\.prop      <span class="hljs-attribute">u</span>:<span class="hljs-attribute">object_r</span>:<span class="hljs-attribute">rootfs</span>:s0
/fstab\..*          <span class="hljs-attribute">u</span>:<span class="hljs-attribute">object_r</span>:<span class="hljs-attribute">rootfs</span>:s0
/init\..*           <span class="hljs-attribute">u</span>:<span class="hljs-attribute">object_r</span>:<span class="hljs-attribute">rootfs</span>:s0
<span class="hljs-regexp">/res(/</span>.*)?          <span class="hljs-attribute">u</span>:<span class="hljs-attribute">object_r</span>:<span class="hljs-attribute">rootfs</span>:s0
/selinux_version    <span class="hljs-attribute">u</span>:<span class="hljs-attribute">object_r</span>:<span class="hljs-attribute">rootfs</span>:s0
/ueventd\..*        <span class="hljs-attribute">u</span>:<span class="hljs-attribute">object_r</span>:<span class="hljs-attribute">rootfs</span>:s0
/verity_key         <span class="hljs-attribute">u</span>:<span class="hljs-attribute">object_r</span>:<span class="hljs-attribute">rootfs</span>:s0</code></pre> 
<p>可以看到使用正则表达式描述了系统文件的安全上下文。</p> 
<p>在 Android 系统中有一种比较特殊的权限——属性&#xff0c;app 能够读写他们获取相应的系统信息以及控制系统的行为。因此不同与 SELinux&#xff0c;SEAndroid 中对系统属性也进行了保护&#xff0c;这意味着 Android 系统的属性也需要关联安全上下文。这是在 property_contexts 文件中描述的</p> 
<pre class="prettyprint"><code class=" hljs css">##########################
# <span class="hljs-tag">property</span> <span class="hljs-tag">service</span> <span class="hljs-tag">keys</span>
#
#
<span class="hljs-tag">net</span><span class="hljs-class">.rmnet</span>               <span class="hljs-tag">u</span><span class="hljs-pseudo">:object_r</span><span class="hljs-pseudo">:net_radio_prop</span><span class="hljs-pseudo">:s0</span>
<span class="hljs-tag">net</span><span class="hljs-class">.gprs</span>                <span class="hljs-tag">u</span><span class="hljs-pseudo">:object_r</span><span class="hljs-pseudo">:net_radio_prop</span><span class="hljs-pseudo">:s0</span>
<span class="hljs-tag">net</span><span class="hljs-class">.ppp</span>                 <span class="hljs-tag">u</span><span class="hljs-pseudo">:object_r</span><span class="hljs-pseudo">:net_radio_prop</span><span class="hljs-pseudo">:s0</span>
<span class="hljs-tag">net</span><span class="hljs-class">.qmi</span>                 <span class="hljs-tag">u</span><span class="hljs-pseudo">:object_r</span><span class="hljs-pseudo">:net_radio_prop</span><span class="hljs-pseudo">:s0</span>
<span class="hljs-tag">net</span><span class="hljs-class">.lte</span>                 <span class="hljs-tag">u</span><span class="hljs-pseudo">:object_r</span><span class="hljs-pseudo">:net_radio_prop</span><span class="hljs-pseudo">:s0</span>
<span class="hljs-tag">net</span><span class="hljs-class">.cdma</span>                <span class="hljs-tag">u</span><span class="hljs-pseudo">:object_r</span><span class="hljs-pseudo">:net_radio_prop</span><span class="hljs-pseudo">:s0</span>
<span class="hljs-tag">net</span><span class="hljs-class">.dns</span>                 <span class="hljs-tag">u</span><span class="hljs-pseudo">:object_r</span><span class="hljs-pseudo">:net_dns_prop</span><span class="hljs-pseudo">:s0</span>
<span class="hljs-tag">sys</span><span class="hljs-class">.usb</span><span class="hljs-class">.config</span>          <span class="hljs-tag">u</span><span class="hljs-pseudo">:object_r</span><span class="hljs-pseudo">:system_radio_prop</span><span class="hljs-pseudo">:s0</span>
<span class="hljs-tag">ril</span>.                    <span class="hljs-tag">u</span><span class="hljs-pseudo">:object_r</span><span class="hljs-pseudo">:radio_prop</span><span class="hljs-pseudo">:s0</span>
<span class="hljs-tag">ro</span><span class="hljs-class">.ril</span>.                 <span class="hljs-tag">u</span><span class="hljs-pseudo">:object_r</span><span class="hljs-pseudo">:radio_prop</span><span class="hljs-pseudo">:s0</span>
<span class="hljs-tag">gsm</span>.                    <span class="hljs-tag">u</span><span class="hljs-pseudo">:object_r</span><span class="hljs-pseudo">:radio_prop</span><span class="hljs-pseudo">:s0</span>
<span class="hljs-tag">persist</span><span class="hljs-class">.radio</span>           <span class="hljs-tag">u</span><span class="hljs-pseudo">:object_r</span><span class="hljs-pseudo">:radio_prop</span><span class="hljs-pseudo">:s0</span></code></pre> 
<p>这边的 net.rmnet 行类型为 net_radio_prop&#xff0c;意味着只有有权限访问 net_radio_prop 的进程才可以访问这个属性。</p> 
<h5 id="2安全策略">&#xff08;2&#xff09;安全策略</h5> 
<p>SEAndroid 安全机制中的安全策略是在安全上下文的基础上进行描述的&#xff0c;也就是说&#xff0c;它通过主体和客体的安全上下文&#xff0c;定义主体是否有权限访问客体。</p> 
<h5 id="type-enforcement">Type Enforcement</h5> 
<p>SEAndroid 安全机制主要是使用对象安全上下文中的类型来定义安全策略&#xff0c;这种安全策略就称 Type Enforcement&#xff0c;简称TE。在 system/sepolicy 目录和其他所有客制化 te 目录&#xff08;通常在 device//common&#xff0c;用 BOARD_SEPOLICY_DIRS 添加&#xff09;&#xff0c;所有以 .te 为后缀的文件经过编译之后&#xff0c;就会生成一个 sepolicy 文件。这个 sepolicy 文件会打包在ROM中&#xff0c;并且保存在设备上的根目录下。</p> 
<p>一个 Type 所具有的权限是通过allow语句来描述的</p> 
<pre class="prettyprint"><code class=" hljs sql">allow unconfineddomain domain:binder { <span class="hljs-operator"><span class="hljs-keyword">call</span> transfer set_context_mgr };</span></code></pre> 
<p>表明 domain 为 unconfineddomain 的进程可以与其它进程进行 binder ipc 通信&#xff08;call&#xff09;&#xff0c;并且能够向这些进程传递 Binder 对象&#xff08;transfer&#xff09;&#xff0c;以及将自己设置为 Binder 上下文管理器&#xff08;set_context_mgr&#xff09;。</p> 
<blockquote> 
 <p>注意&#xff0c;SEAndroid 使用的是最小权限原则&#xff0c;也就是说&#xff0c;只有通过 allow 语句声明的权限才是允许的&#xff0c;而其它没有通过 allow 语句声明的权限都是禁止&#xff0c;这样就可以最大限度地保护系统中的资源</p> 
</blockquote> 
<p>前面我们提到&#xff0c;SEAndroid 安全机制的安全策略经过编译之后会得到一个 sepolicy 文件&#xff0c;并且最终保存在设备上的根目录上。这个 sepolicy 文件中的安全策略是不会自动加载的到内核空间的 SELinux LSM 模块中去的&#xff0c;它需要我们在系统启动的时候进行加载。</p> 
<p>具体的源码分析在 4.2 节中。</p> 
<h5 id="3security-server">&#xff08;3&#xff09;Security Server</h5> 
<p>Security Server 是一个比较笼统的概念&#xff0c;主要是用来保护用户空间资源的&#xff0c;以及用来操作内核空间对象的安全上下文的&#xff0c;它由应用程序安装服务 PackageManagerService、应用程序安装守护进程 installd、应用程序进程孵化器 Zygote 进程以及 init 进程组成。其中&#xff0c;PackageManagerService 和 installd 负责创建 App 数据目录的安全上下文&#xff0c;Zygote 进程负责创建 App 进程的安全上下文&#xff0c;而 init 进程负责控制系统属性的安全访问。</p> 
<h5 id="packagemanagerservice-installed-app-数据目录的安全上下文">PackageManagerService &amp; installed —— app 数据目录的安全上下文</h5> 
<p>PackageManagerService 在启动的时候&#xff0c;会找到我们前面分析的 mac_permissions.xml 文件&#xff0c;然后对它进行解析&#xff0c;得到 App 签名或者包名与 seinfo 的对应关系。当 PackageManagerService 安装 App 的时候&#xff0c;它就会根据其签名或者包名查找到对应的 seinfo&#xff0c;并且将这个 seinfo 传递给另外一个守护进程 installed。</p> 
<p>守护进程 installd 负责创建 App 数据目录。在创建 App 数据目录的时候&#xff0c;需要给它设置安全上下文&#xff0c;使得 SEAndroid 安全机制可以对它进行安全访问控制。Installd 根据 PackageManagerService 传递过来的 seinfo&#xff0c;并且调用 libselinux 库提供的 selabel_lookup 函数到前面我们分析的 seapp_contexts 文件中查找到对应的 Type。有了这个 Type 之后&#xff0c;installd 就可以给正在安装的 App 的数据目录设置安全上下文了&#xff0c;这是通过调用 libselinux 库提供的 lsetfilecon 函数来实现的。</p> 
<h5 id="zygote-设置进程的安全上下文">Zygote 设置进程的安全上下文</h5> 
<p>在 Android 系统中&#xff0c;Zygote 进程负责创建应用程序进程。应用程序进程是 SEAndroid 安全机制中的主体&#xff0c;因此它们也需要设置安全上下文&#xff0c;这是由 Zygote 进程来设置的。</p> 
<p>ActivityManagerService 在请求 Zygote 进程创建应用程序进程之前&#xff0c;会到 PackageManagerService 中去查询对应的 seinfo&#xff0c;并且将这个 seinfo 传递到 Zygote 进程。于是&#xff0c;Zygote 进程在 fork 一个应用程序进程之后&#xff0c;就会使用 ActivityManagerService 传递过来的 seinfo&#xff0c;并且调用 libselinux 库提供的 selabel_lookup 函数到前面我们分析的 seapp_contexts 文件中查找到对应的 Domain。有了这个 Domain 之后&#xff0c;Zygote 进程就可以给刚才创建的应用程序进程设置安全上下文了&#xff0c;这是通过调用 libselinux 库提供的 lsetcon 函数来实现的。</p> 
<h5 id="init-系统属性的安全上下文">init 系统属性的安全上下文</h5> 
<p>init 进程在启动的时候会创建一块内存区域来维护系统中的属性&#xff0c;接着还会创建一个 Property 服务系统&#xff0c;这个服务系统通过 socket 提供接口给其他进程访问 android 系统中的属性。</p> 
<p>其他进程通过 socket 和属性系统通信请求访问某项系统属性的值&#xff0c;属性服务系统可以通过 libselinux 库提供的 selabel_lookup 函数到前面我们分析的 property_contexts 中查找要访问的属性的安全上下文了。有了该进程的安全上下文和要访问属性的安全上下文之后&#xff0c;属性系统就能决定是否允许一个进程访问它所指定的服务了。</p> 
<h2 id="4-seandroid-源码分析">4. SEAndroid 源码分析</h2> 
<h3 id="41-seandroid-源码架构">4.1 SEAndroid 源码架构</h3> 
<pre class="prettyprint"><code class=" hljs haml">-<span class="ruby"> externel/selinux&#xff1a;包含编译 sepolicy 策略文件的一些实用构建工具
</span>    -<span class="ruby"> external/selinux/libselinux&#xff1a;提供了帮助用户进程使用 <span class="hljs-constant">SELinux</span> 的一些函数
</span>    -<span class="ruby"> external/selinux/libsepol&#xff1a;提供了供安全策略文件编译时使用的一个工具 checkcon
</span>-<span class="ruby"> system/sepolicy&#xff1a;包含 <span class="hljs-constant">Android</span> <span class="hljs-constant">SELinux</span> 核心安全策略&#xff08;te 文件&#xff09;&#xff0c;编译生成 sepolicy 文件
</span>    -<span class="ruby"> <span class="hljs-symbol">file_contexts:</span> 系统中所有文件的安全上下文
</span>    -<span class="ruby"> <span class="hljs-symbol">property_contexts:</span> 系统中所有属性的安全上下文
</span>    -<span class="ruby"> seapp_contexts&#xff1a;定义用户、seinfo和域之间的关系&#xff0c;用于确定用户进程的安全上下文
</span>    -<span class="ruby"> sepolicy&#xff1a;二进制文件&#xff0c;保存系统安全策略&#xff0c;系统初始化时会把它设置到内核中</span></code></pre> 
<blockquote> 
 <p>SELinux 虚拟文件系统在 sys/fs/selinux 下&#xff0c;该目录下的文件是 SELinux 内核和用户进程进行通信的接口&#xff0c;libselinux 就是利用这边的接口进行操作</p> 
</blockquote> 
<h3 id="42-init-进程-seandroid-启动源码分析">4.2 init 进程 SEAndroid 启动源码分析</h3> 
<pre class="prettyprint"><code class=" hljs r">文件路径&#xff1a;/system/core/init/init.cpp

int main(int argc, char** argv) {
    <span class="hljs-keyword">...</span>
    <span class="hljs-keyword">if</span> (is_first_stage) {
        <span class="hljs-keyword">...</span> 

        // Set up SELinux, loading the SELinux policy.
        selinux_initialize(true); 

        // We<span class="hljs-string">&#39;re in the kernel domain, so re-exec init to transition to the init domain now
        // that the SELinux policy has been loaded.
        if (selinux_android_restorecon(&#34;/init&#34;, 0) &#61;&#61; -1) {
            PLOG(ERROR) &lt;&lt; &#34;restorecon failed&#34;;
            security_failure();
        }
        ...
    }
}</span></code></pre> 
<p>可以看到 SEAndroid 的启动设置在 init 进程内核态执行&#xff08;first_stage&#xff09;过程中&#xff0c;selinux_initialize 函数就是主要的初始化过程&#xff0c;包含加载 sepolicy 策略文件到内核的 LSM 模块中。</p> 
<pre class="prettyprint"><code class=" hljs cpp">文件路径&#xff1a;/system/core/init/init.cpp

<span class="hljs-keyword">static</span> <span class="hljs-keyword">void</span> selinux_initialize(<span class="hljs-keyword">bool</span> in_kernel_domain) {
    Timer t;

    selinux_callback cb;
    cb.func_log &#61; selinux_klog_callback;
    selinux_set_callback(SELINUX_CB_LOG, cb);
    cb.func_audit &#61; audit_callback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);

    <span class="hljs-comment">// 标识 init 进程的内核态执行和用户态执行</span>
    <span class="hljs-keyword">if</span> (in_kernel_domain) {
        LOG(INFO) &lt;&lt; <span class="hljs-string">&#34;Loading SELinux policy&#34;</span>;
        <span class="hljs-keyword">if</span> (!selinux_load_policy()) {
            panic();
        }

        <span class="hljs-keyword">bool</span> kernel_enforcing &#61; (security_getenforce() &#61;&#61; <span class="hljs-number">1</span>);
        <span class="hljs-keyword">bool</span> is_enforcing &#61; selinux_is_enforcing();
        <span class="hljs-keyword">if</span> (kernel_enforcing !&#61; is_enforcing) {
            <span class="hljs-keyword">if</span> (security_setenforce(is_enforcing)) {
                PLOG(ERROR) &lt;&lt; <span class="hljs-string">&#34;security_setenforce(%s) failed&#34;</span> &lt;&lt; (is_enforcing ? <span class="hljs-string">&#34;true&#34;</span> : <span class="hljs-string">&#34;false&#34;</span>);
                security_failure();
            }
        }

        <span class="hljs-built_in">std</span>::<span class="hljs-built_in">string</span> err;
        <span class="hljs-keyword">if</span> (!WriteFile(<span class="hljs-string">&#34;/sys/fs/selinux/checkreqprot&#34;</span>, <span class="hljs-string">&#34;0&#34;</span>, &amp;err)) {
            LOG(ERROR) &lt;&lt; err;
            security_failure();
        }

        <span class="hljs-comment">// init&#39;s first stage can&#39;t set properties, so pass the time to the second stage.</span>
        setenv(<span class="hljs-string">&#34;INIT_SELINUX_TOOK&#34;</span>, <span class="hljs-built_in">std</span>::to_string(t.duration().count()).c_str(), <span class="hljs-number">1</span>);
    } <span class="hljs-keyword">else</span> {
        selinux_init_all_handles();
    }
}</code></pre> 
<h5 id="selinuxsetcallback">selinux_set_callback</h5> 
<p>selinux_set_callback 用来向 libselinux 设置 SEAndroid 日志和审计回调函数</p> 
<h5 id="selinuxloadpolicy">selinux_load_policy</h5> 
<p>这个函数用来加载 sepolicy 策略文件&#xff0c;并通过 mmap 映射的方式将 sepolicy 的安全策略加载到 SELinux LSM 模块中去。</p> 
<p>下面具体分析一下流程&#xff1a;</p> 
<pre class="prettyprint"><code class=" hljs cs">文件路径&#xff1a;system/core/init/init.cpp

<span class="hljs-keyword">static</span> <span class="hljs-keyword">bool</span> selinux_load_policy() {
    <span class="hljs-keyword">return</span> selinux_is_split_policy_device() ? selinux_load_split_policy()
                                            : selinux_load_monolithic_policy();
}</code></pre> 
<p>这里区分了从哪里加载安全策略文件&#xff0c;第一个是从 /vendor/etc/selinux/precompiled_sepolicy 读取&#xff0c;第二个是直接从根目录 /sepolicy <br /> 中读取&#xff0c;最终调用的方法都是 selinux_android_load_policy_from_fd</p> 
<pre class="prettyprint"><code class=" hljs perl"><span class="hljs-keyword">int</span> selinux_android_load_policy_from_fd(<span class="hljs-keyword">int</span> fd, const char <span class="hljs-variable">*description</span>)
{
    <span class="hljs-keyword">int</span> rc;
    struct <span class="hljs-keyword">stat</span> sb;
    void <span class="hljs-variable">*map</span> &#61; NULL;
    static <span class="hljs-keyword">int</span> load_successful &#61; <span class="hljs-number">0</span>;

    <span class="hljs-regexp">/*
     * Since updating policy at runtime has been abolished
     * we just check whether a policy has been loaded before
     * and return if this is the case.
     * There is no point in reloading policy.
     */</span>
    <span class="hljs-keyword">if</span> (load_successful){
      selinux_log(SELINUX_WARNING, <span class="hljs-string">&#34;SELinux: Attempted reload of SELinux policy!/n&#34;</span>);
      <span class="hljs-keyword">return</span> <span class="hljs-number">0</span>;
    }

    set_selinuxmnt(SELINUXMNT);
    <span class="hljs-keyword">if</span> (fstat(fd, &amp;sb) &lt; <span class="hljs-number">0</span>) {
        selinux_log(SELINUX_ERROR, <span class="hljs-string">&#34;SELinux:  Could not stat <span class="hljs-variable">%s</span>:  <span class="hljs-variable">%s</span>\n&#34;</span>,
                description, strerror(errno));
        <span class="hljs-keyword">return</span> -<span class="hljs-number">1</span>;
    }
    <span class="hljs-keyword">map</span> &#61; mmap(NULL, sb.st_size, PROT_READ, MAP_PRIVATE, fd, <span class="hljs-number">0</span>);
    <span class="hljs-keyword">if</span> (<span class="hljs-keyword">map</span> &#61;&#61; MAP_FAILED) {
        selinux_log(SELINUX_ERROR, <span class="hljs-string">&#34;SELinux:  Could not map <span class="hljs-variable">%s</span>:  <span class="hljs-variable">%s</span>\n&#34;</span>,
                description, strerror(errno));
        <span class="hljs-keyword">return</span> -<span class="hljs-number">1</span>;
    }

    rc &#61; security_load_policy(<span class="hljs-keyword">map</span>, sb.st_size);
    <span class="hljs-keyword">if</span> (rc &lt; <span class="hljs-number">0</span>) {
        selinux_log(SELINUX_ERROR, <span class="hljs-string">&#34;SELinux:  Could not load policy:  <span class="hljs-variable">%s</span>\n&#34;</span>,
                strerror(errno));
        munmap(<span class="hljs-keyword">map</span>, sb.st_size);
        <span class="hljs-keyword">return</span> -<span class="hljs-number">1</span>;
    }

    munmap(<span class="hljs-keyword">map</span>, sb.st_size);
    selinux_log(SELINUX_INFO, <span class="hljs-string">&#34;SELinux: Loaded policy from <span class="hljs-variable">%s</span>\n&#34;</span>, description);
    load_successful &#61; <span class="hljs-number">1</span>;
    <span class="hljs-keyword">return</span> <span class="hljs-number">0</span>;
}</code></pre> 
<p>&#xff08;1&#xff09;set_selinuxmnt 函数设置内核 SELinux 文件系统路径&#xff0c;这里的值为 /sys/fs/selinux&#xff0c;SELinux 文件系统用来与内核空间 SELinux LSM 模块空间。</p> 
<p>&#xff08;2&#xff09;通过 fstat 获取 sepolicy 文件&#xff08;fd 值&#xff09;的状态信息&#xff0c;通过 mmap 函数将文件内容映射到内存中&#xff0c;起始地址为 map</p> 
<p>&#xff08;3&#xff09;security_load_policy 调用另一个 security_load_policy 函数将已经映射到内存中的 SEAndroid 的安全策略加载到内核空间的 SELinux LSM 模块中去。</p> 
<pre class="prettyprint"><code class=" hljs perl"><span class="hljs-keyword">int</span> security_load_policy(void <span class="hljs-variable">*data</span>, size_t len)
{
    char path[PATH_MAX];
    <span class="hljs-keyword">int</span> fd, ret;

    <span class="hljs-keyword">if</span> (!selinux_mnt) {
        errno &#61; ENOENT;
        <span class="hljs-keyword">return</span> -<span class="hljs-number">1</span>;
    }

    snprintf(path, sizeof path, <span class="hljs-string">&#34;<span class="hljs-variable">%s</span>/load&#34;</span>, selinux_mnt);
    fd &#61; <span class="hljs-keyword">open</span>(path, O_RDWR | O_CLOEXEC);
    <span class="hljs-keyword">if</span> (fd &lt; <span class="hljs-number">0</span>)
        <span class="hljs-keyword">return</span> -<span class="hljs-number">1</span>;

    ret &#61; <span class="hljs-keyword">write</span>(fd, data, len);
    <span class="hljs-keyword">close</span>(fd);
    <span class="hljs-keyword">if</span> (ret &lt; <span class="hljs-number">0</span>)
        <span class="hljs-keyword">return</span> -<span class="hljs-number">1</span>;
    <span class="hljs-keyword">return</span> <span class="hljs-number">0</span>;
}</code></pre> 
<p>函数 security_load_policy 的实现很简单&#xff0c;它首先打开 /sys/fs/selinux/load 文件&#xff0c;然后将参数 data 所描述的安全策略写入到这个文件中去。由于 /sys/fs/selinux 是由内核空间的 SELinux LSM 模块导出来的文件系统接口&#xff0c;因此当我们将安全策略写入到位于该文件系统中的 load 文件时&#xff0c;就相当于是将安全策略从用户空间加载到 SELinux LSM 模块中去了。</p> 
<p>以后 SELinux LSM 模块中的 Security Server 就可以通过它来进行安全检查</p> 
<p>&#xff08;4&#xff09;加载完成&#xff0c;释放 sepolicy 文件占用的内存&#xff0c;并且关闭 sepolicy 文件</p> 
<h5 id="securitysetenforce">security_setenforce</h5> 
<p>前面已经提过&#xff0c;selinux 有两种工作模式&#xff1a; <br /> - 宽容模式&#xff08;permissive&#xff09; - 仅记录但不强制执行 SELinux 安全政策。 <br /> - 强制模式&#xff08;enforcing&#xff09; - 强制执行并记录安全政策。如果失败&#xff0c;则显示为 EPERM 错误。</p> 
<p>这个函数用来设置 kernel SELinux 的模式&#xff0c;实际上都是去操作 /sys/fs/selinux/enforce 文件, 0 表示permissive&#xff0c;1 表示 enforcing</p> 
<h5 id="selinuxinitallhandles">selinux_init_all_handles</h5> 
<p>在 init 进程的用户态启动过程中会调用这个函数初始化 file_context、 property_context 相关内容 handler&#xff0c;根据前面的描述&#xff0c;init 进程给一些文件或者系统属性进行安全上下文检查时会使用 libselinux 的 API&#xff0c;查询文件根目录下 file_context、file_context 的安全上下文内容。所以需要先得到这些文件的 handler&#xff0c;以便可以用来查询系统文件和系统属性的安全上下文。 </p> 
<pre class="prettyprint"><code class=" hljs cs"><span class="hljs-keyword">static</span> <span class="hljs-keyword">void</span> selinux_init_all_handles(<span class="hljs-keyword">void</span>)
{
    sehandle &#61; selinux_android_file_context_handle();
    selinux_android_set_sehandle(sehandle);
    sehandle_prop &#61; selinux_android_prop_context_handle();
}</code></pre>
                </div>
                <link href="https://csdnimg.cn/release/blogv2/dist/mdeditor/css/editerView/markdown_views-d7a94ec6ab.css" rel="stylesheet">
                <link href="https://csdnimg.cn/release/blogv2/dist/mdeditor/css/style-49037e4d27.css" rel="stylesheet">
        </div>
    </article>