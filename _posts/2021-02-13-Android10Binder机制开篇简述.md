---
layout:     post
title:      Android10 Binder机制开篇简述
subtitle:   Binder作为Android系统提供的一种IPC机制，无论从事系统开发还是应用开发，都应该有所了解
date:       2021-02-13
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android10
    - Android
    - framework
---

<h2 id="一概述">一、概述</h2>
<p>Android系统中，每个应用程序是由Android的<code class="language-plaintext highlighter-rouge">Activity</code>，<code class="language-plaintext highlighter-rouge">Service</code>，<code class="language-plaintext highlighter-rouge">Broadcast</code>，<code class="language-plaintext highlighter-rouge">ContentProvider</code>这四剑客的中一个或多个组合而成，这四剑客所涉及的多进程间的通信底层都是依赖于Binder IPC机制。例如当进程A中的Activity要向进程B中的Service通信，这便需要依赖于Binder IPC。不仅于此，整个Android系统架构中，大量采用了Binder机制作为IPC（进程间通信）方案，当然也存在部分其他的IPC方式，比如Zygote通信便是采用socket。</p>

<p>Binder作为Android系统提供的一种IPC机制，无论从事系统开发还是应用开发，都应该有所了解，这是Android系统中最重要的组成，也是最难理解的一块知识点，错综复杂。要深入了解Binder机制，最好的方法便是阅读源码，借用Linux鼻祖Linus Torvalds曾说过的一句话：<code class="language-plaintext highlighter-rouge">Read The Fucking Source Code</code>。</p>

<h2 id="二-binder">二、 Binder</h2>

<h3 id="21-ipc原理">2.1 IPC原理</h3>

<p>从进程角度来看IPC机制</p>

<p><img src="https://img-blog.csdnimg.cn/08dadd726d0d43139dc748a53756e7a0.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_16,color_FFFFFF,t_70,g_se,x_16" alt="binder_interprocess_communication" /></p>

<p>每个Android的进程，只能运行在自己进程所拥有的虚拟地址空间。对应一个4GB的虚拟地址空间，其中3GB是用户空间，1GB是内核空间，当然内核空间的大小是可以通过参数配置调整的。对于用户空间，不同进程之间彼此是不能共享的，而内核空间却是可共享的。Client进程向Server进程通信，恰恰是利用进程间可共享的内核内存空间来完成底层通信工作的，Client端与Server端进程往往采用ioctl等方法跟内核空间的驱动进行交互。</p>

<h3 id="22-binder原理">2.2 Binder原理</h3>

<p>Binder通信采用C/S架构，从组件视角来说，包含Client、Server、ServiceManager以及binder驱动，其中ServiceManager用于管理系统中的各种服务。架构图如下所示：</p>

<p><img src="https://img-blog.csdnimg.cn/51aa29ea36cf4d85aa0cdf6c894989d2.jpg?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt="ServiceManager" /></p>

<p>可以看出无论是注册服务和获取服务的过程都需要ServiceManager，需要注意的是此处的Service Manager是指Native层的ServiceManager（C++），并非指framework层的ServiceManager(Java)。ServiceManager是整个Binder通信机制的大管家，是Android进程间通信机制Binder的守护进程，要掌握Binder机制，首先需要了解系统是如何首次<a href="http://gityuan.com/2015/11/07/binder-start-sm/">启动Service Manager</a>。当Service Manager启动之后，Client端和Server端通信时都需要先<a href="http://gityuan.com/2015/11/08/binder-get-sm/">获取Service Manager</a>接口，才能开始通信服务。</p>

<p>图中Client/Server/ServiceManage之间的相互通信都是基于Binder机制。既然基于Binder机制通信，那么同样也是C/S架构，则图中的3大步骤都有相应的Client端与Server端。</p>

<ol>
  <li><strong><a href="http://gityuan.com/2015/11/14/binder-add-service/">注册服务(addService)</a></strong>：Server进程要先注册Service到ServiceManager。该过程：Server是客户端，ServiceManager是服务端。</li>
  <li><strong><a href="http://gityuan.com/2015/11/15/binder-get-service/">获取服务(getService)</a></strong>：Client进程使用某个Service前，须先向ServiceManager中获取相应的Service。该过程：Client是客户端，ServiceManager是服务端。</li>
  <li><strong>使用服务</strong>：Client根据得到的Service信息建立与Service所在的Server进程通信的通路，然后就可以直接与Service交互。该过程：client是客户端，server是服务端。</li>
</ol>

<p>图中的Client,Server,Service Manager之间交互都是虚线表示，是由于它们彼此之间不是直接交互的，而是都通过与<a href="http://gityuan.com/2015/11/01/binder-driver/">Binder驱动</a>进行交互的，从而实现IPC通信方式。其中Binder驱动位于内核空间，Client,Server,Service Manager位于用户空间。Binder驱动和Service Manager可以看做是Android平台的基础架构，而Client和Server是Android的应用层，开发人员只需自定义实现client、Server端，借助Android的基本平台架构便可以直接进行IPC通信。</p>

<h3 id="23-cs模式">2.3 C/S模式</h3>

<p>BpBinder(客户端)和BBinder(服务端)都是Android中Binder通信相关的代表，它们都从IBinder类中派生而来，关系图如下：</p>

<p><img src="https://img-blog.csdnimg.cn/a0a9cc94251e4fb0bdf68b38d9caf544.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_9,color_FFFFFF,t_70,g_se,x_16" alt="Binder关系图" /></p>

<ul>
  <li>client端：BpBinder.transact()来发送事务请求；</li>
  <li>server端：BBinder.onTransact()会接收到相应事务。</li>
</ul>

<h2 id="三-提纲">三、 提纲</h2>

<p>在后续的Binder源码分析过程中所涉及的源码，会有部分的精简，主要是去掉所有log输出语句，已减少代码篇幅过于长</p>

<h2 id="四-源码目录">四. 源码目录</h2>
<p>从上之下, 整个Binder架构所涉及的总共有以下5个目录:</p>

<pre><code>
/framework/base/core/java/               (Java)
/framework/base/core/jni/                (JNI)
/framework/native/libs/binder            (Native)
/framework/native/cmds/servicemanager/   (Native)
/kernel/drivers/staging/android          (Driver)
</code></pre>

<h4 id="41-java-framework">4.1 Java framework</h4>

<pre><code>/framework/base/core/java/android/os/  
    - IInterface.java
    - IBinder.java
    - Parcel.java
    - IServiceManager.java
    - ServiceManager.java
    - ServiceManagerNative.java
    - Binder.java  


/framework/base/core/jni/    
    - android_os_Parcel.cpp
    - AndroidRuntime.cpp
    - android_util_Binder.cpp (核心类)
</code></pre>

<h4 id="42-native-framework">4.2 Native framework</h4>

<pre><code>
/framework/native/libs/binder         
    - IServiceManager.cpp
    - BpBinder.cpp
    - Binder.cpp
    - IPCThreadState.cpp (核心类)
    - ProcessState.cpp  (核心类)

/framework/native/include/binder/
    - IServiceManager.h
    - IInterface.h

/framework/native/cmds/servicemanager/
    - service_manager.c
    - binder.c
</code></pre>

<h4 id="43-kernel">4.3 Kernel</h4>

<pre><code>/kernel/drivers/android/
    - binder.c
    - uapi/binder.h
</code></pre>