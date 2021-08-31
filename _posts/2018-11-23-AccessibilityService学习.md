---
layout:     post
title:      AccessibilityService学习
subtitle:   AccessibilityService学习记录
date:       2018-11-23
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android
    - 组件学习
---   


 <h3><a id="_0"></a>前言</h3> 
<p>今天我们将使用AccessibilityService实现:</p> 
<ol><li>监听第三方程序的界面变化&#xff08;监听第三方程序的启动的实现原理&#xff09;。</li><li>模拟点击第三方应用的按钮&#xff08;自动抢红包程序的实现原理&#xff09;。</li><li>监听第三方程序的点击事件。</li></ol> 

<h3><a id="_10"></a>模拟程序</h3> 
<p>我们先写一个模拟程序&#xff0c;该模拟程序只有一个按钮用于模拟点击事件。</p> 
<pre><code class="prism language-java"><span class="token keyword">public</span> <span class="token keyword">class</span> <span class="token class-name">MainActivity</span> <span class="token keyword">extends</span> <span class="token class-name">AppCompatActivity</span> <span class="token punctuation">{<!-- --></span>
    <span class="token keyword">private</span> <span class="token keyword">static</span> <span class="token keyword">final</span> String TAG <span class="token operator">&#61;</span> <span class="token string">&#34;MainActivity&#34;</span><span class="token punctuation">;</span>

    <span class="token annotation punctuation">&#64;Override</span>
    <span class="token keyword">protected</span> <span class="token keyword">void</span> <span class="token function">onCreate</span><span class="token punctuation">(</span>Bundle savedInstanceState<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
        <span class="token keyword">super</span><span class="token punctuation">.</span><span class="token function">onCreate</span><span class="token punctuation">(</span>savedInstanceState<span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token function">setContentView</span><span class="token punctuation">(</span>R<span class="token punctuation">.</span>layout<span class="token punctuation">.</span>activity_main<span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token function">findViewById</span><span class="token punctuation">(</span>R<span class="token punctuation">.</span>id<span class="token punctuation">.</span>btn_click<span class="token punctuation">)</span><span class="token punctuation">.</span><span class="token function">setOnClickListener</span><span class="token punctuation">(</span><span class="token keyword">new</span> <span class="token class-name">View<span class="token punctuation">.</span>OnClickListener</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
            <span class="token annotation punctuation">&#64;Override</span>
            <span class="token keyword">public</span> <span class="token keyword">void</span> <span class="token function">onClick</span><span class="token punctuation">(</span>View v<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                <span class="token comment">//Toast.makeText(MainActivity.this, &#34;我被点击了&#xff01;&#xff01;&#xff01;&#34;, Toast.LENGTH_SHORT).show();</span>
                Log<span class="token punctuation">.</span><span class="token function">i</span><span class="token punctuation">(</span>TAG<span class="token punctuation">,</span> <span class="token string">&#34;onClick: 我被点击了&#xff01;&#xff01;&#xff01;&#34;</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
            <span class="token punctuation">}</span>
        <span class="token punctuation">}</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre> 
<pre><code class="prism language-xml"><span class="token prolog">&lt;?xml version&#61;&#34;1.0&#34; encoding&#61;&#34;utf-8&#34;?&gt;</span>
<span class="token tag"><span class="token tag"><span class="token punctuation">&lt;</span>LinearLayout</span> <span class="token attr-name"><span class="token namespace">xmlns:</span>android</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>http://schemas.android.com/apk/res/android<span class="token punctuation">&#34;</span></span>
    <span class="token attr-name"><span class="token namespace">android:</span>layout_width</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>match_parent<span class="token punctuation">&#34;</span></span>
    <span class="token attr-name"><span class="token namespace">android:</span>layout_height</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>match_parent<span class="token punctuation">&#34;</span></span>
    <span class="token attr-name"><span class="token namespace">android:</span>gravity</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>center<span class="token punctuation">&#34;</span></span>
    <span class="token attr-name"><span class="token namespace">android:</span>orientation</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>vertical<span class="token punctuation">&#34;</span></span><span class="token punctuation">&gt;</span></span>

    <span class="token tag"><span class="token tag"><span class="token punctuation">&lt;</span>Button</span>
        <span class="token attr-name"><span class="token namespace">android:</span>id</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>&#64;&#43;id/btn_click<span class="token punctuation">&#34;</span></span>
        <span class="token attr-name"><span class="token namespace">android:</span>layout_width</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>wrap_content<span class="token punctuation">&#34;</span></span>
        <span class="token attr-name"><span class="token namespace">android:</span>layout_height</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>wrap_content<span class="token punctuation">&#34;</span></span>
        <span class="token attr-name"><span class="token namespace">android:</span>text</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>模拟点击<span class="token punctuation">&#34;</span></span> <span class="token punctuation">/&gt;</span></span>
<span class="token tag"><span class="token tag"><span class="token punctuation">&lt;/</span>LinearLayout</span><span class="token punctuation">&gt;</span></span>
</code></pre> 
<h3><a id="_48"></a>监听程序</h3> 
<h4><a id="AccessibilityService_50"></a>AccessibilityService</h4> 
<p>代码具体的详情请看注释。</p> 
<pre><code class="prism language-java"><span class="token keyword">public</span> <span class="token keyword">class</span> <span class="token class-name">ListeningService</span> <span class="token keyword">extends</span> <span class="token class-name">AccessibilityService</span> <span class="token punctuation">{<!-- --></span>
    <span class="token keyword">private</span> <span class="token keyword">static</span> <span class="token keyword">final</span> String TAG <span class="token operator">&#61;</span> <span class="token string">&#34;WindowChange&#34;</span><span class="token punctuation">;</span>

    <span class="token annotation punctuation">&#64;Override</span>
    <span class="token keyword">protected</span> <span class="token keyword">void</span> <span class="token function">onServiceConnected</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
        <span class="token keyword">super</span><span class="token punctuation">.</span><span class="token function">onServiceConnected</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        AccessibilityServiceInfo config <span class="token operator">&#61;</span> <span class="token keyword">new</span> <span class="token class-name">AccessibilityServiceInfo</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token comment">//配置监听的事件类型为界面变化|点击事件</span>
        config<span class="token punctuation">.</span>eventTypes <span class="token operator">&#61;</span> AccessibilityEvent<span class="token punctuation">.</span>TYPE_WINDOW_STATE_CHANGED <span class="token operator">|</span> AccessibilityEvent<span class="token punctuation">.</span>TYPE_VIEW_CLICKED<span class="token punctuation">;</span>
        config<span class="token punctuation">.</span>feedbackType <span class="token operator">&#61;</span> AccessibilityServiceInfo<span class="token punctuation">.</span>FEEDBACK_GENERIC<span class="token punctuation">;</span>
        <span class="token keyword">if</span> <span class="token punctuation">(</span>Build<span class="token punctuation">.</span>VERSION<span class="token punctuation">.</span>SDK_INT <span class="token operator">&gt;&#61;</span> <span class="token number">16</span><span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
            config<span class="token punctuation">.</span>flags <span class="token operator">&#61;</span> AccessibilityServiceInfo<span class="token punctuation">.</span>FLAG_INCLUDE_NOT_IMPORTANT_VIEWS<span class="token punctuation">;</span>
        <span class="token punctuation">}</span>
        <span class="token function">setServiceInfo</span><span class="token punctuation">(</span>config<span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span>

    <span class="token annotation punctuation">&#64;Override</span>
    <span class="token keyword">public</span> <span class="token keyword">void</span> <span class="token function">onAccessibilityEvent</span><span class="token punctuation">(</span>AccessibilityEvent event<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
        AccessibilityNodeInfo nodeInfo <span class="token operator">&#61;</span> event<span class="token punctuation">.</span><span class="token function">getSource</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span><span class="token comment">//当前界面的可访问节点信息</span>
        <span class="token keyword">if</span> <span class="token punctuation">(</span>event<span class="token punctuation">.</span><span class="token function">getEventType</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token operator">&#61;&#61;</span> AccessibilityEvent<span class="token punctuation">.</span>TYPE_WINDOW_STATE_CHANGED<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span><span class="token comment">//界面变化事件</span>
            ComponentName componentName <span class="token operator">&#61;</span> <span class="token keyword">new</span> <span class="token class-name">ComponentName</span><span class="token punctuation">(</span>event<span class="token punctuation">.</span><span class="token function">getPackageName</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">.</span><span class="token function">toString</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span> event<span class="token punctuation">.</span><span class="token function">getClassName</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">.</span><span class="token function">toString</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
            ActivityInfo activityInfo <span class="token operator">&#61;</span> <span class="token function">tryGetActivity</span><span class="token punctuation">(</span>componentName<span class="token punctuation">)</span><span class="token punctuation">;</span>
            <span class="token keyword">boolean</span> isActivity <span class="token operator">&#61;</span> activityInfo <span class="token operator">!&#61;</span> null<span class="token punctuation">;</span>
            <span class="token keyword">if</span> <span class="token punctuation">(</span>isActivity<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                Log<span class="token punctuation">.</span><span class="token function">i</span><span class="token punctuation">(</span>TAG<span class="token punctuation">,</span> componentName<span class="token punctuation">.</span><span class="token function">flattenToShortString</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
                <span class="token comment">//格式为&#xff1a;(包名/.&#43;当前Activity所在包的类名)</span>
                <span class="token comment">//如果是模拟程序的操作界面</span>
                <span class="token keyword">if</span> <span class="token punctuation">(</span>componentName<span class="token punctuation">.</span><span class="token function">flattenToShortString</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">.</span><span class="token function">equals</span><span class="token punctuation">(</span><span class="token string">&#34;com.demon.simulationclick/.MainActivity&#34;</span><span class="token punctuation">)</span><span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                    <span class="token comment">//当前是模拟程序的主页面&#xff0c;则模拟点击按钮</span>
                    <span class="token keyword">if</span> <span class="token punctuation">(</span>android<span class="token punctuation">.</span>os<span class="token punctuation">.</span>Build<span class="token punctuation">.</span>VERSION<span class="token punctuation">.</span>SDK_INT <span class="token operator">&gt;&#61;</span> android<span class="token punctuation">.</span>os<span class="token punctuation">.</span>Build<span class="token punctuation">.</span>VERSION_CODES<span class="token punctuation">.</span>JELLY_BEAN_MR2<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                        <span class="token comment">//通过id寻找控件&#xff0c;id格式为&#xff1a;(包名:id/&#43;制定控件的id)</span>
                        <span class="token comment">//一般除非第三方应该是自己的&#xff0c;否则我们很难通过这种方式找到控件</span>
                        <span class="token comment">//List&lt;AccessibilityNodeInfo&gt; list &#61; nodeInfo.findAccessibilityNodeInfosByViewId(&#34;com.demon.simulationclick:id/btn_click&#34;);</span>
                        <span class="token comment">//通过控件的text寻找控件</span>
                        List<span class="token generics function"><span class="token punctuation">&lt;</span>AccessibilityNodeInfo<span class="token punctuation">&gt;</span></span> list <span class="token operator">&#61;</span> nodeInfo<span class="token punctuation">.</span><span class="token function">findAccessibilityNodeInfosByText</span><span class="token punctuation">(</span><span class="token string">&#34;模拟点击&#34;</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
                        <span class="token keyword">if</span> <span class="token punctuation">(</span>list <span class="token operator">!&#61;</span> null <span class="token operator">&amp;&amp;</span> list<span class="token punctuation">.</span><span class="token function">size</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token operator">&gt;</span> <span class="token number">0</span><span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                            list<span class="token punctuation">.</span><span class="token function">get</span><span class="token punctuation">(</span><span class="token number">0</span><span class="token punctuation">)</span><span class="token punctuation">.</span><span class="token function">performAction</span><span class="token punctuation">(</span>AccessibilityNodeInfo<span class="token punctuation">.</span>ACTION_CLICK<span class="token punctuation">)</span><span class="token punctuation">;</span>
                        <span class="token punctuation">}</span>
                    <span class="token punctuation">}</span>
                <span class="token punctuation">}</span>
            <span class="token punctuation">}</span>
        <span class="token punctuation">}</span>
        <span class="token keyword">if</span> <span class="token punctuation">(</span>event<span class="token punctuation">.</span><span class="token function">getEventType</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token operator">&#61;&#61;</span> AccessibilityEvent<span class="token punctuation">.</span>TYPE_VIEW_CLICKED<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span><span class="token comment">//View点击事件</span>
            <span class="token comment">//Log.i(TAG, &#34;onAccessibilityEvent: &#34; &#43; nodeInfo.getText());</span>
            <span class="token keyword">if</span> <span class="token punctuation">(</span><span class="token punctuation">(</span>nodeInfo<span class="token punctuation">.</span><span class="token function">getText</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token operator">&#43;</span> <span class="token string">&#34;&#34;</span><span class="token punctuation">)</span><span class="token punctuation">.</span><span class="token function">equals</span><span class="token punctuation">(</span><span class="token string">&#34;模拟点击&#34;</span><span class="token punctuation">)</span><span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                <span class="token comment">//Toast.makeText(this, &#34;这是来自监听Service的响应&#xff01;&#34;, Toast.LENGTH_SHORT).show();</span>
                Log<span class="token punctuation">.</span><span class="token function">i</span><span class="token punctuation">(</span>TAG<span class="token punctuation">,</span> <span class="token string">&#34;onAccessibilityEvent: 这是来自监听Service的响应&#xff01;&#34;</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
            <span class="token punctuation">}</span>
        <span class="token punctuation">}</span>
    <span class="token punctuation">}</span>

    <span class="token keyword">private</span> ActivityInfo <span class="token function">tryGetActivity</span><span class="token punctuation">(</span>ComponentName componentName<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
        <span class="token keyword">try</span> <span class="token punctuation">{<!-- --></span>
            <span class="token keyword">return</span> <span class="token function">getPackageManager</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">.</span><span class="token function">getActivityInfo</span><span class="token punctuation">(</span>componentName<span class="token punctuation">,</span> <span class="token number">0</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span> <span class="token keyword">catch</span> <span class="token punctuation">(</span><span class="token class-name">PackageManager<span class="token punctuation">.</span>NameNotFoundException</span> e<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
            <span class="token keyword">return</span> null<span class="token punctuation">;</span>
        <span class="token punctuation">}</span>
    <span class="token punctuation">}</span>

    <span class="token annotation punctuation">&#64;Override</span>
    <span class="token keyword">public</span> <span class="token keyword">void</span> <span class="token function">onInterrupt</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
    <span class="token punctuation">}</span>

<span class="token punctuation">}</span>

</code></pre> 
<h4><a id="Service_121"></a>配置Service</h4> 
<h5><a id="AndroidManifestxml_123"></a>AndroidManifest.xml</h5> 
<pre><code class="prism language-xml"><span class="token tag"><span class="token tag"><span class="token punctuation">&lt;</span>uses-permission</span> <span class="token attr-name"><span class="token namespace">android:</span>name</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>android.permission.READ_PHONE_STATE<span class="token punctuation">&#34;</span></span> <span class="token punctuation">/&gt;</span></span>
···
<span class="token tag"><span class="token tag"><span class="token punctuation">&lt;</span>service</span>
            <span class="token attr-name"><span class="token namespace">android:</span>name</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>.ListeningService<span class="token punctuation">&#34;</span></span>
            <span class="token attr-name"><span class="token namespace">android:</span>label</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>&#64;string/app_name<span class="token punctuation">&#34;</span></span>
            <span class="token attr-name"><span class="token namespace">android:</span>permission</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>android.permission.BIND_ACCESSIBILITY_SERVICE<span class="token punctuation">&#34;</span></span><span class="token punctuation">&gt;</span></span>
            <span class="token tag"><span class="token tag"><span class="token punctuation">&lt;</span>intent-filter</span><span class="token punctuation">&gt;</span></span>
                <span class="token tag"><span class="token tag"><span class="token punctuation">&lt;</span>action</span> <span class="token attr-name"><span class="token namespace">android:</span>name</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>android.accessibilityservice.AccessibilityService<span class="token punctuation">&#34;</span></span> <span class="token punctuation">/&gt;</span></span>
            <span class="token tag"><span class="token tag"><span class="token punctuation">&lt;/</span>intent-filter</span><span class="token punctuation">&gt;</span></span>
            <span class="token tag"><span class="token tag"><span class="token punctuation">&lt;</span>meta-data</span>
                <span class="token attr-name"><span class="token namespace">android:</span>name</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>android.accessibilityservice<span class="token punctuation">&#34;</span></span>
                <span class="token attr-name"><span class="token namespace">android:</span>resource</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>&#64;xml/accessibilityservice<span class="token punctuation">&#34;</span></span> <span class="token punctuation">/&gt;</span></span>
        <span class="token tag"><span class="token tag"><span class="token punctuation">&lt;/</span>service</span><span class="token punctuation">&gt;</span></span>
 ···
</code></pre> 
<h5><a id="accessibilityservicexml_141"></a>accessibilityservice.xml</h5> 
<table><thead><tr><th>方法</th><th>说明</th></tr></thead><tbody><tr><td>android:accessibilityEventTypes</td><td>设置响应事件的类型typeAllMask&#xff0c;typeViewClicked&#xff0c;typeViewFocused&#xff0c;typeNotificationStateChanged&#xff0c;typeWindowStateChanged等</td></tr><tr><td>android:accessibilityFeedbackType</td><td>设置反馈给用户的方式</td></tr><tr><td>android:canRetrieveWindowContent</td><td>是否检索当前窗口的内容&#xff0c;即是否可以获取当前窗口的View</td></tr><tr><td>android:description</td><td>服务申请的权限描述说明</td></tr></tbody></table>
<pre><code class="prism language-xml"><span class="token prolog">&lt;?xml version&#61;&#34;1.0&#34; encoding&#61;&#34;utf-8&#34;?&gt;</span>
<span class="token tag"><span class="token tag"><span class="token punctuation">&lt;</span>accessibility-service</span> <span class="token attr-name"><span class="token namespace">xmlns:</span>android</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>http://schemas.android.com/apk/res/android<span class="token punctuation">&#34;</span></span>
    <span class="token attr-name"><span class="token namespace">xmlns:</span>tools</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>http://schemas.android.com/tools<span class="token punctuation">&#34;</span></span>
    <span class="token attr-name"><span class="token namespace">android:</span>accessibilityEventTypes</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>typeAllMask<span class="token punctuation">&#34;</span></span>
    <span class="token attr-name"><span class="token namespace">android:</span>accessibilityFeedbackType</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>feedbackGeneric<span class="token punctuation">&#34;</span></span>
    <span class="token attr-name"><span class="token namespace">android:</span>accessibilityFlags</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>flagIncludeNotImportantViews<span class="token punctuation">&#34;</span></span>
    <span class="token attr-name"><span class="token namespace">android:</span>canRetrieveWindowContent</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>true<span class="token punctuation">&#34;</span></span>
    <span class="token attr-name"><span class="token namespace">android:</span>description</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>&#64;string/accessibility_service_description<span class="token punctuation">&#34;</span></span>
    <span class="token attr-name"><span class="token namespace">tools:</span>ignore</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>UnusedAttribute<span class="token punctuation">&#34;</span></span> <span class="token punctuation">/&gt;</span></span>
</code></pre> 
<h4><a id="MainActivity_162"></a>MainActivity</h4> 
<p>使用AccessibilityService服务需要申请辅助功能&#xff08;小米手机叫&#xff1a;无障碍&#xff09;的服务支持&#xff0c;无法主动给与&#xff0c;需要到指定界面用户手动开启。</p> 
<pre><code class="prism language-java"><span class="token keyword">public</span> <span class="token keyword">class</span> <span class="token class-name">MainActivity</span> <span class="token keyword">extends</span> <span class="token class-name">AppCompatActivity</span> <span class="token punctuation">{<!-- --></span>
    <span class="token keyword">private</span> <span class="token keyword">static</span> <span class="token keyword">final</span> String TAG <span class="token operator">&#61;</span> <span class="token string">&#34;MainActivity&#34;</span><span class="token punctuation">;</span>
    <span class="token keyword">private</span> Intent intent<span class="token punctuation">;</span>

    <span class="token annotation punctuation">&#64;Override</span>
    <span class="token keyword">protected</span> <span class="token keyword">void</span> <span class="token function">onCreate</span><span class="token punctuation">(</span>Bundle savedInstanceState<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
        <span class="token keyword">super</span><span class="token punctuation">.</span><span class="token function">onCreate</span><span class="token punctuation">(</span>savedInstanceState<span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token function">setContentView</span><span class="token punctuation">(</span>R<span class="token punctuation">.</span>layout<span class="token punctuation">.</span>activity_main<span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token function">findViewById</span><span class="token punctuation">(</span>R<span class="token punctuation">.</span>id<span class="token punctuation">.</span>start<span class="token punctuation">)</span><span class="token punctuation">.</span><span class="token function">setOnClickListener</span><span class="token punctuation">(</span><span class="token keyword">new</span> <span class="token class-name">View<span class="token punctuation">.</span>OnClickListener</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
            <span class="token annotation punctuation">&#64;Override</span>
            <span class="token keyword">public</span> <span class="token keyword">void</span> <span class="token function">onClick</span><span class="token punctuation">(</span>View view<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                <span class="token keyword">if</span> <span class="token punctuation">(</span><span class="token operator">!</span><span class="token function">isAccessibilitySettingsOn</span><span class="token punctuation">(</span>MainActivity<span class="token punctuation">.</span><span class="token keyword">this</span><span class="token punctuation">,</span> ListeningService<span class="token punctuation">.</span><span class="token keyword">class</span><span class="token punctuation">.</span><span class="token function">getCanonicalName</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                    Intent intent <span class="token operator">&#61;</span> <span class="token keyword">new</span> <span class="token class-name">Intent</span><span class="token punctuation">(</span>Settings<span class="token punctuation">.</span>ACTION_ACCESSIBILITY_SETTINGS<span class="token punctuation">)</span><span class="token punctuation">;</span>
                    <span class="token function">startActivity</span><span class="token punctuation">(</span>intent<span class="token punctuation">)</span><span class="token punctuation">;</span>
                <span class="token punctuation">}</span> <span class="token keyword">else</span> <span class="token punctuation">{<!-- --></span>
                    intent <span class="token operator">&#61;</span> <span class="token keyword">new</span> <span class="token class-name">Intent</span><span class="token punctuation">(</span>MainActivity<span class="token punctuation">.</span><span class="token keyword">this</span><span class="token punctuation">,</span> ListeningService<span class="token punctuation">.</span><span class="token keyword">class</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
                    <span class="token function">startService</span><span class="token punctuation">(</span>intent<span class="token punctuation">)</span><span class="token punctuation">;</span>
                <span class="token punctuation">}</span>
            <span class="token punctuation">}</span>
        <span class="token punctuation">}</span><span class="token punctuation">)</span><span class="token punctuation">;</span>

        <span class="token function">findViewById</span><span class="token punctuation">(</span>R<span class="token punctuation">.</span>id<span class="token punctuation">.</span>stop<span class="token punctuation">)</span><span class="token punctuation">.</span><span class="token function">setOnClickListener</span><span class="token punctuation">(</span><span class="token keyword">new</span> <span class="token class-name">View<span class="token punctuation">.</span>OnClickListener</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
            <span class="token annotation punctuation">&#64;Override</span>
            <span class="token keyword">public</span> <span class="token keyword">void</span> <span class="token function">onClick</span><span class="token punctuation">(</span>View v<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                <span class="token keyword">if</span> <span class="token punctuation">(</span>intent <span class="token operator">!&#61;</span> null<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                    <span class="token function">stopService</span><span class="token punctuation">(</span>intent<span class="token punctuation">)</span><span class="token punctuation">;</span>
                <span class="token punctuation">}</span>
            <span class="token punctuation">}</span>
        <span class="token punctuation">}</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span>


    <span class="token comment">/**
     * 检测辅助功能是否开启
     *
     * &#64;param mContext
     * &#64;return boolean
     */</span>
    <span class="token keyword">private</span> <span class="token keyword">boolean</span> <span class="token function">isAccessibilitySettingsOn</span><span class="token punctuation">(</span>Context mContext<span class="token punctuation">,</span> String serviceName<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
        <span class="token keyword">int</span> accessibilityEnabled <span class="token operator">&#61;</span> <span class="token number">0</span><span class="token punctuation">;</span>
        <span class="token comment">// 对应的服务</span>
        <span class="token keyword">final</span> String service <span class="token operator">&#61;</span> <span class="token function">getPackageName</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token operator">&#43;</span> <span class="token string">&#34;/&#34;</span> <span class="token operator">&#43;</span> serviceName<span class="token punctuation">;</span>
        <span class="token comment">//Log.i(TAG, &#34;service:&#34; &#43; service);</span>
        <span class="token keyword">try</span> <span class="token punctuation">{<!-- --></span>
            accessibilityEnabled <span class="token operator">&#61;</span> Settings<span class="token punctuation">.</span>Secure<span class="token punctuation">.</span><span class="token function">getInt</span><span class="token punctuation">(</span>mContext<span class="token punctuation">.</span><span class="token function">getApplicationContext</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">.</span><span class="token function">getContentResolver</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span>
                    android<span class="token punctuation">.</span>provider<span class="token punctuation">.</span>Settings<span class="token punctuation">.</span>Secure<span class="token punctuation">.</span>ACCESSIBILITY_ENABLED<span class="token punctuation">)</span><span class="token punctuation">;</span>
            Log<span class="token punctuation">.</span><span class="token function">v</span><span class="token punctuation">(</span>TAG<span class="token punctuation">,</span> <span class="token string">&#34;accessibilityEnabled &#61; &#34;</span> <span class="token operator">&#43;</span> accessibilityEnabled<span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span> <span class="token keyword">catch</span> <span class="token punctuation">(</span><span class="token class-name">Settings<span class="token punctuation">.</span>SettingNotFoundException</span> e<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
            Log<span class="token punctuation">.</span><span class="token function">e</span><span class="token punctuation">(</span>TAG<span class="token punctuation">,</span> <span class="token string">&#34;Error finding setting, default accessibility to not found: &#34;</span> <span class="token operator">&#43;</span> e<span class="token punctuation">.</span><span class="token function">getMessage</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span>
        TextUtils<span class="token punctuation">.</span>SimpleStringSplitter mStringColonSplitter <span class="token operator">&#61;</span> <span class="token keyword">new</span> <span class="token class-name">TextUtils<span class="token punctuation">.</span>SimpleStringSplitter</span><span class="token punctuation">(</span><span class="token string">&#39;:&#39;</span><span class="token punctuation">)</span><span class="token punctuation">;</span>

        <span class="token keyword">if</span> <span class="token punctuation">(</span>accessibilityEnabled <span class="token operator">&#61;&#61;</span> <span class="token number">1</span><span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
            Log<span class="token punctuation">.</span><span class="token function">v</span><span class="token punctuation">(</span>TAG<span class="token punctuation">,</span> <span class="token string">&#34;***ACCESSIBILITY IS ENABLED*** -----------------&#34;</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
            String settingValue <span class="token operator">&#61;</span> Settings<span class="token punctuation">.</span>Secure<span class="token punctuation">.</span><span class="token function">getString</span><span class="token punctuation">(</span>mContext<span class="token punctuation">.</span><span class="token function">getApplicationContext</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">.</span><span class="token function">getContentResolver</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span>
                    Settings<span class="token punctuation">.</span>Secure<span class="token punctuation">.</span>ENABLED_ACCESSIBILITY_SERVICES<span class="token punctuation">)</span><span class="token punctuation">;</span>
            <span class="token keyword">if</span> <span class="token punctuation">(</span>settingValue <span class="token operator">!&#61;</span> null<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                mStringColonSplitter<span class="token punctuation">.</span><span class="token function">setString</span><span class="token punctuation">(</span>settingValue<span class="token punctuation">)</span><span class="token punctuation">;</span>
                <span class="token keyword">while</span> <span class="token punctuation">(</span>mStringColonSplitter<span class="token punctuation">.</span><span class="token function">hasNext</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                    String accessibilityService <span class="token operator">&#61;</span> mStringColonSplitter<span class="token punctuation">.</span><span class="token function">next</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>

                    Log<span class="token punctuation">.</span><span class="token function">v</span><span class="token punctuation">(</span>TAG<span class="token punctuation">,</span> <span class="token string">&#34;-------------- &gt; accessibilityService :: &#34;</span> <span class="token operator">&#43;</span> accessibilityService <span class="token operator">&#43;</span> <span class="token string">&#34; &#34;</span> <span class="token operator">&#43;</span> service<span class="token punctuation">)</span><span class="token punctuation">;</span>
                    <span class="token keyword">if</span> <span class="token punctuation">(</span>accessibilityService<span class="token punctuation">.</span><span class="token function">equalsIgnoreCase</span><span class="token punctuation">(</span>service<span class="token punctuation">)</span><span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                        Log<span class="token punctuation">.</span><span class="token function">v</span><span class="token punctuation">(</span>TAG<span class="token punctuation">,</span> <span class="token string">&#34;We&#39;ve found the correct setting - accessibility is switched on!&#34;</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
                        <span class="token keyword">return</span> <span class="token boolean">true</span><span class="token punctuation">;</span>
                    <span class="token punctuation">}</span>
                <span class="token punctuation">}</span>
            <span class="token punctuation">}</span>
        <span class="token punctuation">}</span> <span class="token keyword">else</span> <span class="token punctuation">{<!-- --></span>
            Log<span class="token punctuation">.</span><span class="token function">v</span><span class="token punctuation">(</span>TAG<span class="token punctuation">,</span> <span class="token string">&#34;***ACCESSIBILITY IS DISABLED***&#34;</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span>
        <span class="token keyword">return</span> <span class="token boolean">false</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre> 
<pre><code class="prism language-xml"><span class="token prolog">&lt;?xml version&#61;&#34;1.0&#34; encoding&#61;&#34;utf-8&#34;?&gt;</span>
<span class="token tag"><span class="token tag"><span class="token punctuation">&lt;</span>LinearLayout</span> <span class="token attr-name"><span class="token namespace">xmlns:</span>android</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>http://schemas.android.com/apk/res/android<span class="token punctuation">&#34;</span></span>
    <span class="token attr-name"><span class="token namespace">android:</span>layout_width</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>match_parent<span class="token punctuation">&#34;</span></span>
    <span class="token attr-name"><span class="token namespace">android:</span>layout_height</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>match_parent<span class="token punctuation">&#34;</span></span>
    <span class="token attr-name"><span class="token namespace">android:</span>gravity</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>center<span class="token punctuation">&#34;</span></span>
    <span class="token attr-name"><span class="token namespace">android:</span>orientation</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>vertical<span class="token punctuation">&#34;</span></span><span class="token punctuation">&gt;</span></span>

    <span class="token tag"><span class="token tag"><span class="token punctuation">&lt;</span>Button</span>
        <span class="token attr-name"><span class="token namespace">android:</span>id</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>&#64;&#43;id/start<span class="token punctuation">&#34;</span></span>
        <span class="token attr-name"><span class="token namespace">android:</span>layout_width</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>wrap_content<span class="token punctuation">&#34;</span></span>
        <span class="token attr-name"><span class="token namespace">android:</span>layout_height</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>wrap_content<span class="token punctuation">&#34;</span></span>
        <span class="token attr-name"><span class="token namespace">android:</span>text</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>开启监听服务<span class="token punctuation">&#34;</span></span> <span class="token punctuation">/&gt;</span></span>

    <span class="token tag"><span class="token tag"><span class="token punctuation">&lt;</span>Button</span>
        <span class="token attr-name"><span class="token namespace">android:</span>id</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>&#64;&#43;id/stop<span class="token punctuation">&#34;</span></span>
        <span class="token attr-name"><span class="token namespace">android:</span>layout_width</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>wrap_content<span class="token punctuation">&#34;</span></span>
        <span class="token attr-name"><span class="token namespace">android:</span>layout_height</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>wrap_content<span class="token punctuation">&#34;</span></span>
        <span class="token attr-name"><span class="token namespace">android:</span>text</span><span class="token attr-value"><span class="token punctuation">&#61;</span><span class="token punctuation">&#34;</span>关闭监听服务<span class="token punctuation">&#34;</span></span> <span class="token punctuation">/&gt;</span></span>
<span class="token tag"><span class="token tag"><span class="token punctuation">&lt;/</span>LinearLayout</span><span class="token punctuation">&gt;</span></span>
</code></pre> 
<h4><a id="_264"></a>效果</h4> 
<p>运行监听程序&#xff0c;给与服务支持后&#xff0c;打开监听程序。<br /> 然后运行模拟程序&#xff0c;效果如下。</p> 

 <p>监听到com.demon.simulationclick/.MainActivity-------模拟点击按钮-----监听到按钮被点击------从监听Service发出响应<br /> 
<imgsrc="https://img-blog.csdn.net/20180813180236344?aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RlTW9ubGl1aHVp/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70" /></p> 
