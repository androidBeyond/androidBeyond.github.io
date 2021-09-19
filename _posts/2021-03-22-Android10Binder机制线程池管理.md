---
layout:     post
title:      Android10 Binder机制7-线程池管理
subtitle:   Binder线程创建与其所在进程的创建中产生，Java层进程的创建都是通过Process.start()方法
date:       2021-03-22
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


<h2 id="一-概述">一. 概述</h2>

<p>Android系统启动完成后，ActivityManager, PackageManager等各大服务都运行在system_server进程，app应用需要使用系统服务都是通过binder来完成进程之间的通信，那么对于binder线程是如何管理的呢，又是如何创建的呢？其实无论是system_server进程，还是app进程，都是在进程fork完成后，便会在新进程中执行onZygoteInit()的过程中，启动binder线程池。接下来，就以此为起点展开从线程的视角来看看binder的世界。</p>

<h2 id="二-binder线程创建">二. Binder线程创建</h2>

<p>Binder线程创建与其所在进程的创建中产生，Java层进程的创建都是通过Process.start()方法，向Zygote进程发出创建进程的socket消息，Zygote收到消息后会调用Zygote.forkAndSpecialize()来fork出新进程，在新进程中会调用到<code class="language-plaintext highlighter-rouge">RuntimeInit.nativeZygoteInit</code>方法，该方法经过jni映射，最终会调用到app_main.cpp中的onZygoteInit，那么接下来从这个方法说起。</p>

<h3 id="21-onzygoteinit">2.1 onZygoteInit</h3>
<p>[-&gt; app_main.cpp]</p>

<pre ><code>virtual void onZygoteInit()
{
    //获取ProcessState对象
    sp&lt;ProcessState&gt; proc = ProcessState::self();
    //启动新binder线程 【见小节2.2】
    proc-&gt;startThreadPool();
}
</code></pre>

<p>ProcessState::self()是单例模式，主要工作是调用open()打开/dev/binder驱动设备，再利用mmap()映射内核的地址空间，将Binder驱动的fd赋值ProcessState对象中的变量mDriverFD，用于交互操作。startThreadPool()是创建一个新的binder线程，不断进行talkWithDriver()。</p>

<h3 id="22-psstartthreadpool">2.2 PS.startThreadPool</h3>
<p>[-&gt; ProcessState.cpp]</p>

<pre ><code>void ProcessState::startThreadPool()
{
    AutoMutex _l(mLock);    //多线程同步
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true);  【见小节2.3】
    }
}
</code></pre>

<p>启动Binder线程池后, 则设置mThreadPoolStarted=true. 通过变量mThreadPoolStarted来保证每个应用进程只允许启动一个binder线程池, 且本次创建的是binder主线程(isMain=true).
其余binder线程池中的线程都是由Binder驱动来控制创建的。</p>

<h3 id="23-psspawnpooledthread">2.3 PS.spawnPooledThread</h3>
<p>[-&gt; ProcessState.cpp]</p>

<pre ><code>void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        //获取Binder线程名【见小节2.3.1】
        String8 name = makeBinderThreadName();
        //此处isMain=true【见小节2.3.2】
        sp&lt;Thread&gt; t = new PoolThread(isMain);
        t-&gt;run(name.string());
    }
}
</code></pre>

<h4 id="231-makebinderthreadname">2.3.1 makeBinderThreadName</h4>
<p>[-&gt; ProcessState.cpp]</p>

<pre ><code>String8 ProcessState::makeBinderThreadName() {
    int32_t s = android_atomic_add(1, &amp;mThreadPoolSeq);
    String8 name;
    name.appendFormat("Binder_%X", s);
    return name;
}
</code></pre>

<p>获取Binder线程名，格式为<code class="language-plaintext highlighter-rouge">Binder_x</code>, 其中x为整数。每个进程中的binder编码是从1开始，依次递增; 只有通过spawnPooledThread方法来创建的线程才符合这个格式，对于直接将当前线程通过joinThreadPool加入线程池的线程名则不符合这个命名规则。
另外,目前Android N中Binder命令已改为<code class="language-plaintext highlighter-rouge">Binder:&lt;pid&gt;_x</code>格式, 则对于分析问题很有帮忙,通过binder名称的pid字段可以快速定位该binder线程所属的进程p.</p>

<h4 id="232-poolthreadrun">2.3.2 PoolThread.run</h4>
<p>[-&gt; ProcessState.cpp]</p>

<pre ><code>class PoolThread : public Thread
{
public:
    PoolThread(bool isMain)
        : mIsMain(isMain)
    {
    }

protected:
    virtual bool threadLoop()
    {
        IPCThreadState::self()-&gt;joinThreadPool(mIsMain); //【见小节2.4】
        return false;
    }
    const bool mIsMain;
};
</code></pre>

<p>从函数名看起来是创建线程池，其实就只是创建一个线程，该PoolThread继承Thread类。t-&gt;run()方法最终调用 PoolThread的threadLoop()方法。</p>

<h3 id="24-ipcjointhreadpool">2.4 IPC.joinThreadPool</h3>
<p>[-&gt; IPCThreadState.cpp]</p>

<pre ><code>void IPCThreadState::joinThreadPool(bool isMain)
{
    //创建Binder线程
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);
    set_sched_policy(mMyThreadId, SP_FOREGROUND); //设置前台调度策略

    status_t result;
    do {
        processPendingDerefs(); //清除队列的引用[见小节2.5]
        result = getAndExecuteCommand(); //处理下一条指令[见小节2.6]

        if (result &lt; NO_ERROR &amp;&amp; result != TIMED_OUT
                &amp;&amp; result != -ECONNREFUSED &amp;&amp; result != -EBADF) {
            abort();
        }

        if(result == TIMED_OUT &amp;&amp; !isMain) {
            break; ////非主线程出现timeout则线程退出
        }
    } while (result != -ECONNREFUSED &amp;&amp; result != -EBADF);

    mOut.writeInt32(BC_EXIT_LOOPER);  // 线程退出循环
    talkWithDriver(false); //false代表bwr数据的read_buffer为空
}
</code></pre>

<ul>
  <li>对于<code class="language-plaintext highlighter-rouge">isMain</code>=true的情况下， command为BC_ENTER_LOOPER，代表的是Binder主线程，不会退出的线程；</li>
  <li>对于<code class="language-plaintext highlighter-rouge">isMain</code>=false的情况下，command为BC_REGISTER_LOOPER，表示是由binder驱动创建的线程。</li>
</ul>

<h3 id="25-processpendingderefs">2.5 processPendingDerefs</h3>
<p>[-&gt; IPCThreadState.cpp]</p>

<pre ><code>void IPCThreadState::processPendingDerefs()
{
    if (mIn.dataPosition() &gt;= mIn.dataSize()) {
        size_t numPending = mPendingWeakDerefs.size();
        if (numPending &gt; 0) {
            for (size_t i = 0; i &lt; numPending; i++) {
                RefBase::weakref_type* refs = mPendingWeakDerefs[i];
                refs-&gt;decWeak(mProcess.get()); //弱引用减一
            }
            mPendingWeakDerefs.clear();
        }

        numPending = mPendingStrongDerefs.size();
        if (numPending &gt; 0) {
            for (size_t i = 0; i &lt; numPending; i++) {
                BBinder* obj = mPendingStrongDerefs[i];
                obj-&gt;decStrong(mProcess.get()); //强引用减一
            }
            mPendingStrongDerefs.clear();
        }
    }
}
</code></pre>

<h3 id="26-getandexecutecommand">2.6 getAndExecuteCommand</h3>
<p>[-&gt; IPCThreadState.cpp]</p>

<pre ><code>status_t IPCThreadState::getAndExecuteCommand()
{
    status_t result;
    int32_t cmd;

    result = talkWithDriver(); //与binder进行交互[见小节2.7]
    if (result &gt;= NO_ERROR) {
        size_t IN = mIn.dataAvail();
        if (IN &lt; sizeof(int32_t)) return result;
        cmd = mIn.readInt32();

        pthread_mutex_lock(&amp;mProcess-&gt;mThreadCountLock);
        mProcess-&gt;mExecutingThreadsCount++;
        pthread_mutex_unlock(&amp;mProcess-&gt;mThreadCountLock);

        result = executeCommand(cmd); //执行Binder响应码 [见小节2.8]

        pthread_mutex_lock(&amp;mProcess-&gt;mThreadCountLock);
        mProcess-&gt;mExecutingThreadsCount--;
        pthread_cond_broadcast(&amp;mProcess-&gt;mThreadCountDecrement);
        pthread_mutex_unlock(&amp;mProcess-&gt;mThreadCountLock);

        set_sched_policy(mMyThreadId, SP_FOREGROUND);
    }
    return result;
}
</code></pre>

<h3 id="27-talkwithdriver">2.7 talkWithDriver</h3>

<pre ><code>//mOut有数据，mIn还没有数据。doReceive默认值为true
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    binder_write_read bwr;
    ...
    // 当同时没有输入和输出数据则直接返回
    if ((bwr.write_size == 0) &amp;&amp; (bwr.read_size == 0)) return NO_ERROR;
    ...

    do {
        //ioctl执行binder读写操作，经过syscall，进入Binder驱动。调用Binder_ioctl
        if (ioctl(mProcess-&gt;mDriverFD, BINDER_WRITE_READ, &amp;bwr) &gt;= 0)
            err = NO_ERROR;
        ...
    } while (err == -EINTR);
    ...
    return err;
}
</code></pre>

<p>在这里调用的isMain=true，也就是向mOut例如写入的便是<code class="language-plaintext highlighter-rouge">BC_ENTER_LOOPER</code>. 经过talkWithDriver(), 接下来从<code class="language-plaintext highlighter-rouge">binder_thread_write()</code>往下说<code class="language-plaintext highlighter-rouge">BC_ENTER_LOOPER</code>的处理过程。</p>

<h4 id="271-binder_thread_write">2.7.1 binder_thread_write</h4>
<p>[-&gt; binder.c]</p>

<pre ><code>static int binder_thread_write(struct binder_proc *proc,
            struct binder_thread *thread,
            binder_uintptr_t binder_buffer, size_t size,
            binder_size_t *consumed)
{
    uint32_t cmd;
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;
    while (ptr &lt; end &amp;&amp; thread-&gt;return_error == BR_OK) {
        //拷贝用户空间的cmd命令，此时为BC_ENTER_LOOPER
        if (get_user(cmd, (uint32_t __user *)ptr)) -EFAULT;
        ptr += sizeof(uint32_t);
        switch (cmd) {
          case BC_REGISTER_LOOPER:
              if (thread-&gt;looper &amp; BINDER_LOOPER_STATE_ENTERED) {
                //出错原因：线程调用完BC_ENTER_LOOPER，不能执行该分支
                thread-&gt;looper |= BINDER_LOOPER_STATE_INVALID;

              } else if (proc-&gt;requested_threads == 0) {
                //出错原因：没有请求就创建线程
                thread-&gt;looper |= BINDER_LOOPER_STATE_INVALID;

              } else {
                proc-&gt;requested_threads--;
                proc-&gt;requested_threads_started++;
              }
              thread-&gt;looper |= BINDER_LOOPER_STATE_REGISTERED;
              break;

          case BC_ENTER_LOOPER:
              if (thread-&gt;looper &amp; BINDER_LOOPER_STATE_REGISTERED) {
                //出错原因：线程调用完BC_REGISTER_LOOPER，不能立刻执行该分支
                thread-&gt;looper |= BINDER_LOOPER_STATE_INVALID;
              }
              //创建Binder主线程
              thread-&gt;looper |= BINDER_LOOPER_STATE_ENTERED;
              break;

          case BC_EXIT_LOOPER:
              thread-&gt;looper |= BINDER_LOOPER_STATE_EXITED;
              break;
        }
        ...
    }
    *consumed = ptr - buffer;
  }
  return 0;
}
</code></pre>

<p>处理完BC_ENTER_LOOPER命令后，一般情况下成功设置thread-&gt;looper |= <code class="language-plaintext highlighter-rouge">BINDER_LOOPER_STATE_ENTERED</code>。那么binder线程的创建是在什么时候呢？
那就当该线程有事务需要处理的时候，进入binder_thread_read()过程。</p>

<h4 id="272-binder_thread_read">2.7.2 binder_thread_read</h4>

<pre ><code>binder_thread_read（）{
  ...
retry:
    //当前线程todo队列为空且transaction栈为空，则代表该线程是空闲的
    wait_for_proc_work = thread-&gt;transaction_stack == NULL &amp;&amp;
        list_empty(&amp;thread-&gt;todo);

    if (thread-&gt;return_error != BR_OK &amp;&amp; ptr &lt; end) {
        ...
        put_user(thread-&gt;return_error, (uint32_t __user *)ptr);
        ptr += sizeof(uint32_t);
        goto done; //发生error，则直接进入done
    }

    thread-&gt;looper |= BINDER_LOOPER_STATE_WAITING;
    if (wait_for_proc_work)
        proc-&gt;ready_threads++; //可用线程个数+1
    binder_unlock(__func__);

    if (wait_for_proc_work) {
        if (non_block) {
            ...
        } else
            //当进程todo队列没有数据,则进入休眠等待状态
            ret = wait_event_freezable_exclusive(proc-&gt;wait, binder_has_proc_work(proc, thread));
    } else {
        if (non_block) {
            ...
        } else
            //当线程todo队列没有数据，则进入休眠等待状态
            ret = wait_event_freezable(thread-&gt;wait, binder_has_thread_work(thread));
    }

    binder_lock(__func__);
    if (wait_for_proc_work)
        proc-&gt;ready_threads--; //可用线程个数-1
    thread-&gt;looper &amp;= ~BINDER_LOOPER_STATE_WAITING;

    if (ret)
        return ret; //对于非阻塞的调用，直接返回

    while (1) {
        uint32_t cmd;
        struct binder_transaction_data tr;
        struct binder_work *w;
        struct binder_transaction *t = NULL;

        //先考虑从线程todo队列获取事务数据
        if (!list_empty(&amp;thread-&gt;todo)) {
            w = list_first_entry(&amp;thread-&gt;todo, struct binder_work, entry);
        //线程todo队列没有数据, 则从进程todo对获取事务数据
        } else if (!list_empty(&amp;proc-&gt;todo) &amp;&amp; wait_for_proc_work) {
            w = list_first_entry(&amp;proc-&gt;todo, struct binder_work, entry);
        } else {
            ... //没有数据,则返回retry
        }

        switch (w-&gt;type) {
            case BINDER_WORK_TRANSACTION: ...  break;
            case BINDER_WORK_TRANSACTION_COMPLETE:...  break;
            case BINDER_WORK_NODE: ...    break;
            case BINDER_WORK_DEAD_BINDER:
            case BINDER_WORK_DEAD_BINDER_AND_CLEAR:
            case BINDER_WORK_CLEAR_DEATH_NOTIFICATION:
                struct binder_ref_death *death;
                uint32_t cmd;

                death = container_of(w, struct binder_ref_death, work);
                if (w-&gt;type == BINDER_WORK_CLEAR_DEATH_NOTIFICATION)
                  cmd = BR_CLEAR_DEATH_NOTIFICATION_DONE;
                else
                  cmd = BR_DEAD_BINDER;
                put_user(cmd, (uint32_t __user *)ptr;
                ptr += sizeof(uint32_t);
                put_user(death-&gt;cookie, (void * __user *)ptr);
                ptr += sizeof(void *);
                ...
                if (cmd == BR_DEAD_BINDER)
                  goto done; //Binder驱动向client端发送死亡通知，则进入done
                break;
        }

        if (!t)
            continue; //只有BINDER_WORK_TRANSACTION命令才能继续往下执行
        ...
        break;
    }

done:
    *consumed = ptr - buffer;
    //创建线程的条件
    if (proc-&gt;requested_threads + proc-&gt;ready_threads == 0 &amp;&amp;
        proc-&gt;requested_threads_started &lt; proc-&gt;max_threads &amp;&amp;
        (thread-&gt;looper &amp; (BINDER_LOOPER_STATE_REGISTERED |
         BINDER_LOOPER_STATE_ENTERED))) {
        proc-&gt;requested_threads++;
        // 生成BR_SPAWN_LOOPER命令，用于创建新的线程
        put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer)；
    }
    return 0;
}
</code></pre>

<p>当发生以下3种情况之一，便会进入<code class="language-plaintext highlighter-rouge">done</code>：</p>

<ul>
  <li>当前线程的return_error发生error的情况；</li>
  <li>当Binder驱动向client端发送死亡通知的情况；</li>
  <li>当类型为BINDER_WORK_TRANSACTION(即收到命令是BC_TRANSACTION或BC_REPLY)的情况；</li>
</ul>

<p>任何一个Binder线程当同时满足以下条件，则会生成用于创建新线程的BR_SPAWN_LOOPER命令：</p>

<ol>
  <li>当前进程中没有请求创建binder线程，即requested_threads = 0；</li>
  <li>当前进程没有空闲可用的binder线程，即ready_threads = 0；（线程进入休眠状态的个数就是空闲线程数）</li>
  <li>当前进程已启动线程个数小于最大上限(默认15)；</li>
  <li>当前线程已接收到BC_ENTER_LOOPER或者BC_REGISTER_LOOPER命令，即当前处于BINDER_LOOPER_STATE_REGISTERED或者BINDER_LOOPER_STATE_ENTERED状态。【小节2.6】已设置状态为BINDER_LOOPER_STATE_ENTERED，显然这条件是满足的。</li>
</ol>

<p>从system_server的binder线程一直的执行流: IPC.joinThreadPool –&gt; IPC.getAndExecuteCommand() -&gt; IPC.talkWithDriver() ,但talkWithDriver收到事务之后, 便进入IPC.executeCommand(), 接下来,从executeCommand说起.</p>

<h3 id="28-ipcexecutecommand">2.8 IPC.executeCommand</h3>

<pre ><code>status_t IPCThreadState::executeCommand(int32_t cmd)
{
    status_t result = NO_ERROR;
    switch ((uint32_t)cmd) {
      ...
      case BR_SPAWN_LOOPER:
          //创建新的binder线程 【见小节2.3】
          mProcess-&gt;spawnPooledThread(false);
          break;
      ...
    }
    return result;
}
</code></pre>

<p>Binder主线程的创建是在其所在进程创建的过程一起创建的，后面再创建的普通binder线程是由spawnPooledThread(false)方法所创建的。</p>

<h3 id="29-思考">2.9 思考</h3>

<p>默认地，每个进程的binder线程池的线程个数上限为15，该上限不统计通过BC_ENTER_LOOPER命令创建的binder主线程， 只计算BC_REGISTER_LOOPER命令创建的线程。 对此，或者很多人不理解，例个栗子：某个进程的主线程执行如下方法，那么该进程可创建的binder线程个数上限是多少呢？</p>

<pre ><code>ProcessState::self()-&gt;setThreadPoolMaxThreadCount(6);  // 6个线程
ProcessState::self()-&gt;startThreadPool();   // 1个线程
IPCThread::self()-&gt;joinThreadPool();   // 1个线程
</code></pre>

<p>首先线程池的binder线程个数上限为6个，通过startThreadPool()创建的主线程不算在最大线程上限，最后一句是将当前线程成为binder线程，所以说可创建的binder线程个数上限为8，如果还不理解，建议再多看看这几个方案的源码，多思考整个binder架构。</p>

<h2 id="三-总结">三. 总结</h2>

<p>Binder设计架构中，只有第一个Binder主线程(也就是Binder_1线程)是由应用程序主动创建，Binder线程池的普通线程都是由Binder驱动根据IPC通信需求创建，Binder线程的创建流程图：</p>

<p><img src="https://img-blog.csdnimg.cn/fe64d1229eb34935a3e9d777784ed787.jpg?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt="binder_thread_create" /></p>

<p>每次由Zygote fork出新进程的过程中，伴随着创建binder线程池，调用spawnPooledThread来创建binder主线程。当线程执行binder_thread_read的过程中，发现当前没有空闲线程，没有请求创建线程，且没有达到上限，则创建新的binder线程。</p>

<p><strong>Binder的transaction有3种类型：</strong></p>

<ol>
  <li>call: 发起进程的线程不一定是在Binder线程， 大多數情況下，接收者只指向进程，并不确定会有哪个线程来处理，所以不指定线程；</li>
  <li>reply: 发起者一定是binder线程，并且接收者线程便是上次call时的发起线程(该线程不一定是binder线程，可以是任意线程)。</li>
  <li>async: 与call类型差不多，唯一不同的是async是oneway方式不需要回复，发起进程的线程不一定是在Binder线程， 接收者只指向进程，并不确定会有哪个线程来处理，所以不指定线程。</li>
</ol>

<p><strong>Binder系统中可分为3类binder线程：</strong></p>

<ul>
  <li>Binder主线程：进程创建过程会调用startThreadPool()过程中再进入spawnPooledThread(true)，来创建Binder主线程。编号从1开始，也就是意味着binder主线程名为<code class="language-plaintext highlighter-rouge">binder_1</code>，并且主线程是不会退出的。</li>
  <li>Binder普通线程：是由Binder Driver来根据是否有空闲的binder线程来决定是否创建binder线程，回调spawnPooledThread(false)
，isMain=false，该线程名格式为<code class="language-plaintext highlighter-rouge">binder_x</code>。</li>
  <li>Binder其他线程：其他线程是指并没有调用spawnPooledThread方法，而是直接调用IPC.joinThreadPool()，将当前线程直接加入binder线程队列。例如：
mediaserver和servicemanager的主线程都是binder线程，但system_server的主线程并非binder线程。</li>
</ul>