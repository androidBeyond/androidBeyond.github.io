---
layout:     post
title:      Android10 Binder机制10-架构总结
subtitle:   本文对Binder机制进行一个最终总结，从不懂角度阐述一下Binder机制
date:       2021-04-20
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android10
    - Android
    - Binder
    - 进程间通信
---  


<h3 id="1-Binder架构"><a href="#1-Binder架构" class="headerlink" title="1.Binder架构"></a>1.Binder架构</h3><p>下面将从不同的角度对binder进行描述：</p>
<ul>
<li>从IPC角度：Binder是Android中一种跨进程通信方式，这种通信方式是Android独有的;</li>
<li>从Android APP层：Binder是客户端和服务端进行通信的媒介，当bindService的时候，服务端就会返回一个包含服务端业务调用的Binder对象，通过这个Binder对象，客户端就可以获取服务端提供的服务或者数据，这里服务包括普通的服务和基于AIDL的服务;</li>
<li>从Android Framework层：Binder是各种Manager(ActivityManager、WindowManager等)和相应XXXManagerService的桥梁;</li>
<li>从Android Native层：Binder是创建ServiceManager以及BpBinder/BBinder模型，搭建与binder驱动的桥梁;</li>
<li>从Android Driver层：Binder可以理解为一种虚拟的物理设备，它的设备驱动是/dev/binder。</li>
</ul>
<p><img src="https://img-blog.csdnimg.cn/f7fcb03b897e4125a9e181d67d8f92bb.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt="java_binder_frame" style="zoom: 67%;"></p>
<p>Binder在整个Android系统中具有重要的作用，在Native层有一套完整的binder通信的C/S架构，BpBinder作为客户端，BBinder作为服务端。基于native层的Binder架构，Java也有一套镜像功能的binder C/S架构，通过JNI技术和Native层的binder相对应，Java层的binder功能最终都是交给native的binder来完sss成的。从kernel到native，jni，framework层的架构所涉及的相关类和方法如下：</p>
<p><img src="https://img-blog.csdnimg.cn/a48a51e75de143e0836718b5b9d55d66.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt="java_binder_framework_class" style="zoom: 67%;"></p>
<h3 id="2-Binder内存机制"><a href="#2-Binder内存机制" class="headerlink" title="2.Binder内存机制"></a>2.Binder内存机制</h3><p>binder_mmap是Binder进程间通信的高效的核心机制所在，其模型如下：</p>
<p><img src="https://img-blog.csdnimg.cn/69bbd7b28a654f2ca72c93d2b14cba04.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_16,color_FFFFFF,t_70,g_se,x_16" alt="内存模型" style="zoom:80%;"></p>
<p>虚拟进程地址空间（vm_area_struct）和虚拟内核地址空间（vm_struct）都映射到同一块物理内存空间。当client端与server端发送数据时，client作为数据发送端，先从自己的进程空间把IPC通信数据copy_from_user拷贝到内核空间，而server端作为数据接收端，与内核共享数据，不再需要拷贝数据，而是通过内存地址空间的偏移量获取内存地址，整个过程只发生一次内存拷贝。一般的做法，需要Client端进程空间拷贝到内核空间，再由内核空间拷贝到server进程空间，会发生两次拷贝。</p>
<p><img src="https://img-blog.csdnimg.cn/6f06e6836354483b81acd240f20cb60a.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_17,color_FFFFFF,t_70,g_se,x_16" alt="binder内存转换" style="zoom:80%;"></p>
<p>对于进程和内核虚拟地址映射到同一个物理内存的操作（通过地址偏移量来实现）是发生在数据接收端，而数据发送端还是需要将用户态的数据复制到内核态。为什么不直接让发送端和接收端直接映射到同一块物理空间，那样连一次复制的操作都不需要，0次复制那就和Linux标准内核的共享内存IPC没有区别了，对于共享内存虽然效率高，但是对于多进程同步的问题比较复杂，而管道/消息队列等IPC需要复制两次，效率较低。总之Android选择Binder是基于速度和安全性的考虑。</p>
<h3 id="3-Binder通信流程"><a href="#3-Binder通信流程" class="headerlink" title="3.Binder通信流程"></a>3.Binder通信流程</h3><p>以bindService为例总结Binder通信流程</p>
<p>1.发起端线程向Binder驱动发起binder_ioctl请求后，waitForResponse进入while循环，不断进行talkWithDriver,此时该线程处理阻塞状态，直到收到BR_XX命令才会结束该过程。</p>
<ul>
<li>BR_TRANSACTION_COMPLETE: oneway模式下,收到该命令则退出；</li>
<li>BR_DEAD_REPLY: 目标进程/线程/binder实体为空, 以及释放正在等待reply的binder thread或者binder buffer;</li>
<li>BR_FAILED_REPLY: 情况较多,比如非法handle, 错误事务栈, security, 内存不足, buffer不足, 数据拷贝失败, 节点创建失败, 各种不匹配等问题；</li>
<li>BR_ACQUIRE_RESULT: 目前未使用的协议;</li>
<li>BR_REPLY: 非oneway模式下,收到该命令才退出;</li>
</ul>
<p>2.waitForResponse收到BR_TRANSACTION_COMPLETE，则直接退出循环，不会执行executeCommand方法，除上述五种BR_XXX命令，当收到其他BR命令，则会执行executeCommand方法。</p>
<p>3.目标Binder线程创建之后，便进入joinThreadPool方法，不断循环执行getAndExecuteCommand方法，当bwr的读写buffer没有数据时，则阻塞在binder_thread_read的wait_event过程。正常情况下binder线程一旦创建就不会退出。</p>
<p><img src="https://img-blog.csdnimg.cn/89ae038ad1e543e988d7b135734b3fc6.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_16,color_FFFFFF,t_70,g_se,x_16" alt="bindservice_cm_frame" style="zoom:80%;"></p>
<h3 id="4-Binder进程与线程"><a href="#4-Binder进程与线程" class="headerlink" title="4.Binder进程与线程"></a>4.Binder进程与线程</h3><p>对于底层Binder驱动，通过binder_procs链表记录所有创建的binder_proc结构体，binder驱动层的每一个binder_proc结构体都与用户空间的一个用于binder通信的进程一一对应，且每个进程有且只有一个ProcessState对象，这是通过单例模式来保证的。每个进程中可以有很多线程，每个线程对应一个IPCThreadState对象，IPCThreadState对象也是单例模式即一个线程对应一个IPCThreadState对象。在Binder驱动层也有与之相对应的结构，那就是binder_thread结构体，在binder_proc结构体中通过成员变量rb_root threads，来记录当前进程内所有的binder_thread。</p>
<p><img src="https://img-blog.csdnimg.cn/44cd34ecea484e58a3afaf46976f3ae5.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt="binder_proc_relation" style="zoom: 50%;"></p>
<p>Binder线程池：每个Server进程在启动时会创建一个binder线程池，并向其中注册一个Binder线程，之后Server进程也可以向binder线程池注册新的线程，或者binder驱动在检测到没有空闲binder线程时主动向Server进程注册新的binder线程。对于一个Server进程有一个最大的线程数限制，默认是16个binder线程，例如：Android的system_server进程就有16个线程。对于所有Client端进程的binder请求都是交给Server端进程的binder线程进行处理的。</p>
<h3 id="5-Binder传输过程"><a href="#5-Binder传输过程" class="headerlink" title="5.Binder传输过程"></a>5.Binder传输过程</h3><p>Binder IPC机制是指在进程间传输数据（binder_transaction_data）,一次数据的传输称为事务（binder_transaction）。对于多个不同进程向同一个进程发送事务时，这个同一个进程或线程的事务需要串行执行，在Binder驱动中为binder_proc和binder_thread的todo队列。</p>
<p>也就是说对于进程间的通信就是发送端把binder_transaction节点，插入到目标进程或其子线程的todo队列，等目标进程或线程不断循环的从todo队列中取出数据并进行相应的操作。</p>
<p><img src="https://img-blog.csdnimg.cn/b7df9a0561df42c6b241799385393664.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt="binder_transaction" style="zoom:67%;"></p>
<p>在Binder驱动层，每个接收端进程都有一个todo队列，用于保存发送端进程发送过来的binder进程，这类请求可以由接收端的任意一个空闲的binder线程处理；接收端进程存在一个或多个binder线程，在每个binder线程都有一个todo队列，也是用于保存发送端进程发送过来的binder请求，这类请求只能由当前线程进行处理。binder线程在空闲时进入可中断的休眠状态，当自己的todo队列或者所属进程的todo队列有新的请求到来时便会唤醒，如果是所需进程唤醒的，那么进程会让一个线程处理相应的请求，其它线程再次进入休眠状态。</p>
<h3 id="6-Binder路由"><a href="#6-Binder路由" class="headerlink" title="6.Binder路由"></a>6.Binder路由</h3><p>Native Binder IPC的两个重量级对象：BpBinder（客户端）和BBinder（服务端）都是Android中Binder通信相关的代表，它们都是从IBinder类中派生而来，关系图如下：</p>
<p><img src="https://img-blog.csdnimg.cn/d7834146fa4a4258b01fa4666b32c136.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_9,color_FFFFFF,t_70,g_se,x_16" alt="Ibinder_classes"></p>
<p>IBinder有一个重要方法queryLocalInterface，默认返回值为NULL;</p>
<ul>
<li>BBinder/BpBinder都没有实现,默认返回NULL;BnInterface重写该方法；</li>
<li>BinderProxy(Java)默认返回NULL;Binder(Java)重写该方法；</li>
</ul>
<p>IInterface有一个重要方法asBinder；</p>
<p>Interface子类（服务类）有一个方法asInterface;</p>
<p>Native层通过宏IMPLEMENT_META_INTERFACE来完成asInterface实现和descriptor的赋值过程；</p>
<p>对于Java层跟Native一样，也有完全对应的一套对象和方法。例如AIDL自动生成asInterface和descriptor赋值过程。</p>
<p>同一个进程，请求binder服务，不需要创建binder_ref，BpBinder等这些对象，但是是否需要经过binder call取决于descriptor是否设置，这里涉及到Java服务Native使用，或Native服务在Java层使用。</p>
<p><strong>Binder路由原理</strong>：BpBinder发送端，根据handler，在当前binder_proc中，找到相应的binder_ref，由binder_ref再找到目标binder_node实体，由目标binder_node再找到目标进程binder_proc。简单的方式是直接把binder_transaction节点插入到binder_proc的todo队列中，完成传输过程。</p>
<p>对于binder驱动尽可能的把binder_transaction节点插入到目标进程的某个线程的todo队列，效率更高。当binder驱动找到合适的线程，就会把binder_transaction节点插入到相应线程的todo队列中，如果找不到合适的线程，就把节点插入到binder_proc的todo队列。</p>
<h3 id="后记"><a href="#后记" class="headerlink" title="后记"></a>后记</h3><p>整个Binder系列文章参考了gityuan的分析，作为学习Binder机制的总结，对Binder机制有了更深的理解，不再只是看到简单api的调用，其里面详细的调用逻辑才是学习的核心，对于以后的设计架构和代码有很大的帮助。</p>

      