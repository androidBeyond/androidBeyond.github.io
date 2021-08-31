---
layout:     post
title:      Android基础探究之Thread学习
subtitle:   Android基础探究之Thread学习
date:       2019-01-02
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android
    - 基础学习
    - 系统线程
---    

   <p>最近面试被问了Thread与runable的原理有什么不同&#xff0c;本人当时回答的是没什么不同&#xff0c;都是开一个新线程而已&#xff0c;面试官也没有给我个正面反馈告诉我到底有什么不同&#xff0c;索性趁着这个热乎劲我就去深入剖析一下这个Thread。首先写一个例子看看Thread和runable分别是怎么用的。<a href="https://github.com/wk415190639/blog/commit/6bf974ab7ad418c69879f8aa794522a063cfaf53">&#xff08;查看源码&#xff09;</a></p> 
<p>先添加一个Thread的子类&#xff0c;并重新run方法即可</p> 
<pre class="has"><code class="language-java">package com.example.threaddemo;

import android.util.Log;

public class ThreadSub extends Thread implements TAG{

    &#64;Override
    public void run() {
        Log.e(TAG, &#34;run: ThreadSub&#34; );


    }
}
</code></pre> 
<p>再添加一个runnable 的实现类&#xff0c;实现他的run方法</p> 
<pre class="has"><code class="language-java">package com.example.threaddemo;

import android.util.Log;

public class RunnableImpl implements Runnable,TAG {

    String text &#61;&#34;&#34;;

    private RunnableImpl() {
    }

    public RunnableImpl(String text) {
        this.text &#61; text;
    }

    &#64;Override
    public void run() {
        Log.e(TAG, &#34;run: RunableImpl &#34;&#43;text );
        try {
            Thread.sleep(5000);

        }catch (Exception e){

        }

    }
}
</code></pre> 
<p>再加一个使用刚刚写好的类的代码</p> 
<pre class="has"><code class="language-java">    void createThreadWay1() {
        new ThreadSub().start();
    }

    void createThreadWay2() {

        new Thread(new RunnableImpl(&#34;createThreadWay2&#34;)).start();
    }
</code></pre> 
<p>然后点击运行就可以了&#xff0c;实现非常简单&#xff0c;下面就先来看一下Thread是如何在调用start()之后让run()方法运行在新创建的线程里的&#xff0c;这个过程需要jdk源码的协助&#xff0c;这次的学习是基于openjdk8的版本,我下载了一份放到了github上&#xff08;<a href="https://github.com/wk415190639/openJdk8">点这里去下载</a>&#xff09;&#xff0c;接下来就从start()入手</p> 
<pre class="has"><code class="language-java">  public synchronized void start() {
  
        
        if (threadStatus !&#61; 0)
            throw new IllegalThreadStateException();

        group.add(this);

        boolean started &#61; false;
        try {
            start0();
            started &#61; true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
                /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
            }
        }
    }
</code></pre> 
<p>这里首先判断了一下Thread的状态&#xff0c;只有当Threadstatus等于0的时候才说明Thread当前处于尚未启动的状态。之后将thread加入所属的线程组&#xff0c;接着就调用了最关键的一步start0(),start0()是本地方法</p> 
<pre class="has"><code>   private native void start0();</code></pre> 
<p>找一找该方法native层对应的实现</p> 
<pre class="has"><code>
jdk/src/share/native/java/lang/Thread.c:   
 {&#34;start0&#34;,           &#34;()V&#34;,        (void *)&amp;JVM_StartThread},
</code></pre> 
<p>最终在jvm.cpp内找到了该实现</p> 
<pre class="has"><code class="language-cpp">JVM_ENTRY(void, JVM_StartThread(JNIEnv* env, jobject jthread))
  JVMWrapper(&#34;JVM_StartThread&#34;);
  JavaThread *native_thread &#61; NULL;

  bool throw_illegal_thread_state &#61; false;

  // We must release the Threads_lock before we can post a jvmti event
  // in Thread::start.
  {

    MutexLocker mu(Threads_lock);

    if (java_lang_Thread::thread(JNIHandles::resolve_non_null(jthread)) !&#61; NULL) {
      throw_illegal_thread_state &#61; true;
    } else {
      // We could also check the stillborn flag to see if this thread was already stopped, but
      // for historical reasons we let the thread detect that itself when it starts running

      jlong size &#61;
             java_lang_Thread::stackSize(JNIHandles::resolve_non_null(jthread));
      size_t sz &#61; size &gt; 0 ? (size_t) size : 0;
      native_thread &#61; new JavaThread(&amp;thread_entry, sz);

      if (native_thread-&gt;osthread() !&#61; NULL) {
        // Note: the current thread is not being used within &#34;prepare&#34;.
        native_thread-&gt;prepare(jthread);
      }
    }
  }

  if (throw_illegal_thread_state) {
    THROW(vmSymbols::java_lang_IllegalThreadStateException());
  }

  assert(native_thread !&#61; NULL, &#34;Starting null thread?&#34;);

  if (native_thread-&gt;osthread() &#61;&#61; NULL) {
    // No one should hold a reference to the &#39;native_thread&#39;.
    delete native_thread;
    if (JvmtiExport::should_post_resource_exhausted()) {
      JvmtiExport::post_resource_exhausted(
        JVMTI_RESOURCE_EXHAUSTED_OOM_ERROR | JVMTI_RESOURCE_EXHAUSTED_THREADS,
        &#34;unable to create new native thread&#34;);
    }
    THROW_MSG(vmSymbols::java_lang_OutOfMemoryError(),
              &#34;unable to create new native thread&#34;);
  }

  Thread::start(native_thread);

JVM_END</code></pre> 
<p>代码里用了很多的宏&#xff0c;开起来不是很方便&#xff0c;可以利用预编译将宏展开</p> 
<pre class="has"><code>gcc -E jvm.cpp -I../ -I/home/wk/android/openJdk8/build/linux-x86_64-normal-server-release/hotspot/linux_amd64_compiler2/generated/ -I./ &gt;temp.cpp
</code></pre> 
<p>宏展开之后</p> 
<pre class="has"><code class="language-cpp">
extern &#34;C&#34; {
void JNICALL JVM_StartThread(JNIEnv *env, jobject jthread) {
    JavaThread *thread &#61; JavaThread::thread_from_jni_environment(env);
    ThreadInVMfromNative __tiv(thread);
    HandleMarkCleaner __hm(thread);
    Thread *__the_thread__ &#61; thread;
    os::verify_stack_alignment();;
    JavaThread *native_thread &#61; __null;

    bool throw_illegal_thread_state &#61; false;
    {
        MutexLocker mu(Threads_lock);

        if (java_lang_Thread::thread(JNIHandles::resolve_non_null(jthread)) !&#61; __null) {
            throw_illegal_thread_state &#61; true;
        } else {

            jlong size &#61;
                    java_lang_Thread::stackSize(JNIHandles::resolve_non_null(jthread));

            size_t sz &#61; size &gt; 0 ? (size_t) size : 0;
            native_thread &#61; new JavaThread(&amp;thread_entry, sz);
            if (native_thread-&gt;osthread() !&#61; __null) {

                native_thread-&gt;prepare(jthread);
            }
        }
    }

    if (throw_illegal_thread_state) {
        {
            Exceptions::_throw_msg(__the_thread__, &#34;jvm.cpp&#34;, 2867,
                                   vmSymbols::java_lang_IllegalThreadStateException(), __null);
            return;
        };
    };

    if (native_thread-&gt;osthread() &#61;&#61; __null) {

        delete native_thread;
        if (JvmtiExport::should_post_resource_exhausted()) {
            JvmtiExport::post_resource_exhausted(
                    JVMTI_RESOURCE_EXHAUSTED_OOM_ERROR | JVMTI_RESOURCE_EXHAUSTED_THREADS,
                    &#34;unable to create new native thread&#34;);
        }
        {
            Exceptions::_throw_msg(__the_thread__, &#34;jvm.cpp&#34;, 2881, vmSymbols::java_lang_OutOfMemoryError(),
                                   &#34;unable to create new native thread&#34;);
            return;
        };
    }

    Thread::start(native_thread);
}
}</code></pre> 
<p>这里面new了一个JavaThread,有名字可以联想到这个JavaThread应该是直接关联Java层的Thread的&#xff0c;下面重点关注这个类&#xff0c;先观察一下他的构造函数</p> 
<p> </p> 
<pre class="has"><code class="language-cpp">
JavaThread::JavaThread(ThreadFunction entry_point, size_t stack_sz) :
  Thread()
#if INCLUDE_ALL_GCS
  , _satb_mark_queue(&amp;_satb_mark_queue_set),
  _dirty_card_queue(&amp;_dirty_card_queue_set)
#endif // INCLUDE_ALL_GCS
{
  if (TraceThreadEvents) {
    tty-&gt;print_cr(&#34;creating thread %p&#34;, this);
  }
  initialize();
  _jni_attach_state &#61; _not_attaching_via_jni;
  set_entry_point(entry_point);
  // Create the native thread itself.
  // %note runtime_23
  os::ThreadType thr_type &#61; os::java_thread;
  thr_type &#61; entry_point &#61;&#61; &amp;compiler_thread_entry ? os::compiler_thread :
                                                     os::java_thread;
  os::create_thread(this, thr_type, stack_sz);
  _safepoint_visible &#61; false;

}
</code></pre> 
<p>在函数内调用了 os::create_thread</p> 
<pre class="has"><code class="language-cpp">
bool os::create_thread(Thread* thread, ThreadType thr_type, size_t stack_size) {
  assert(thread-&gt;osthread() &#61;&#61; NULL, &#34;caller responsible&#34;);

  // Allocate the OSThread object
  OSThread* osthread &#61; new OSThread(NULL, NULL);
  if (osthread &#61;&#61; NULL) {
    return false;
  }

  // set the correct thread state
  osthread-&gt;set_thread_type(thr_type);

  // Initial state is ALLOCATED but not INITIALIZED
  osthread-&gt;set_state(ALLOCATED);

  thread-&gt;set_osthread(osthread);

  // init thread attributes
  pthread_attr_t attr;
  pthread_attr_init(&amp;attr);
  pthread_attr_setdetachstate(&amp;attr, PTHREAD_CREATE_DETACHED);

  // stack size
  if (os::Linux::supports_variable_stack_size()) {
    // calculate stack size if it&#39;s not specified by caller
    if (stack_size &#61;&#61; 0) {
      stack_size &#61; os::Linux::default_stack_size(thr_type);

      switch (thr_type) {
      case os::java_thread:
        // Java threads use ThreadStackSize which default value can be
        // changed with the flag -Xss
        assert (JavaThread::stack_size_at_create() &gt; 0, &#34;this should be set&#34;);
        stack_size &#61; JavaThread::stack_size_at_create();
        break;
      case os::compiler_thread:
        if (CompilerThreadStackSize &gt; 0) {
          stack_size &#61; (size_t)(CompilerThreadStackSize * K);
          break;
        } // else fall through:
          // use VMThreadStackSize if CompilerThreadStackSize is not defined
      case os::vm_thread:
      case os::pgc_thread:
      case os::cgc_thread:
      case os::watcher_thread:
        if (VMThreadStackSize &gt; 0) stack_size &#61; (size_t)(VMThreadStackSize * K);
        break;
      }
    }

    stack_size &#61; MAX2(stack_size, os::Linux::min_stack_allowed);
    pthread_attr_setstacksize(&amp;attr, stack_size);
  } else {
    // let pthread_create() pick the default value.
  }

  // glibc guard page
  pthread_attr_setguardsize(&amp;attr, os::Linux::default_guard_size(thr_type));

  ThreadState state;

  {
    // Serialize thread creation if we are running with fixed stack LinuxThreads
    bool lock &#61; os::Linux::is_LinuxThreads() &amp;&amp; !os::Linux::is_floating_stack();
    if (lock) {
      os::Linux::createThread_lock()-&gt;lock_without_safepoint_check();
    }

    pthread_t tid;
    int ret &#61; pthread_create(&amp;tid, &amp;attr, (void* (*)(void*)) java_start, thread);

    pthread_attr_destroy(&amp;attr);

    if (ret !&#61; 0) {
      if (PrintMiscellaneous &amp;&amp; (Verbose || WizardMode)) {
        perror(&#34;pthread_create()&#34;);
      }
      // Need to clean up stuff we&#39;ve allocated so far
      thread-&gt;set_osthread(NULL);
      delete osthread;
      if (lock) os::Linux::createThread_lock()-&gt;unlock();
      return false;
    }

    // Store pthread info into the OSThread
    osthread-&gt;set_pthread_id(tid);

    // Wait until child thread is either initialized or aborted
    {
      Monitor* sync_with_child &#61; osthread-&gt;startThread_lock();
      MutexLockerEx ml(sync_with_child, Mutex::_no_safepoint_check_flag);
      while ((state &#61; osthread-&gt;get_state()) &#61;&#61; ALLOCATED) {
        sync_with_child-&gt;wait(Mutex::_no_safepoint_check_flag);
      }
    }

    if (lock) {
      os::Linux::createThread_lock()-&gt;unlock();
    }
  }

  // Aborted due to thread limit being reached
  if (state &#61;&#61; ZOMBIE) {
      thread-&gt;set_osthread(NULL);
      delete osthread;
      return false;
  }

  // The thread is returned suspended (in state INITIALIZED),
  // and is started higher up in the call chain
  assert(state &#61;&#61; INITIALIZED, &#34;race condition&#34;);
  return true;
}</code></pre> 
<p>走到这看到了曙光&#xff0c;在上面函数体内看到了熟悉的 pthread_create&#xff0c;这是在C&#43;&#43;中创建线程的接口&#xff0c;原理java的Thread也是调用了 pthread_create的&#xff0c;</p> 
<pre class="has"><code class="language-cpp">  int ret &#61; pthread_create(&amp;tid, &amp;attr, (void* (*)(void*)) java_start, thread);</code></pre> 
<p>这里又产生了个问题&#xff0c;这个传入的函数指针java_start 难道对应的就是Thread层的run函数&#xff1f;带着这个疑问开始掉头返航找找这个函数指针是什么&#xff0c;</p> 
<pre class="has"><code>static void *java_start(Thread *thread) {
  static int counter &#61; 0;
  int pid &#61; os::current_process_id();
  alloca(((pid ^ counter&#43;&#43;) &amp; 7) * 128);

  ThreadLocalStorage::set_thread(thread);

  OSThread* osthread &#61; thread-&gt;osthread();
  Monitor* sync &#61; osthread-&gt;startThread_lock();
  if (!_thread_safety_check(thread)) {
    // notify parent thread
    MutexLockerEx ml(sync, Mutex::_no_safepoint_check_flag);
    osthread-&gt;set_state(ZOMBIE);
    sync-&gt;notify_all();
    return NULL;
  }

  osthread-&gt;set_thread_id(os::Linux::gettid());

  if (UseNUMA) {
    int lgrp_id &#61; os::numa_get_group_id();
    if (lgrp_id !&#61; -1) {
      thread-&gt;set_lgrp_id(lgrp_id);
    }
  }
  os::Linux::hotspot_sigmask(thread);
  os::Linux::init_thread_fpu_state();
  {
    MutexLockerEx ml(sync, Mutex::_no_safepoint_check_flag);
    osthread-&gt;set_state(INITIALIZED);
    sync-&gt;notify_all();
    while (osthread-&gt;get_state() &#61;&#61; INITIALIZED) {
      sync-&gt;wait(Mutex::_no_safepoint_check_flag);
    }
  }
  thread-&gt;run();
  return 0;
}
</code></pre> 
<p>这个函数也不短啊&#xff0c;有趣的是这个函数的参数居然是Thread*&#xff0c;而且函数最后也调用了这个Thread的run函数&#xff0c;由于本人是个大菜鸟&#xff0c;还是头一次看见创建线程时候还可以传Thread*&#xff0c;虽然他也是个指针&#xff0c;但是没有强制转成void*就可以直接编译成功&#xff0c;真是长见识了。话说故事发展到这&#xff0c;这个Thread-&gt;run应该是个重头戏&#xff0c;继续看</p> 
<pre class="has"><code class="language-cpp">
// The first routine called by a new Java thread
void JavaThread::run() {
  // initialize thread-local alloc buffer related fields
  this-&gt;initialize_tlab();

  // used to test validitity of stack trace backs
  this-&gt;record_base_of_stack_pointer();

  // Record real stack base and size.
  this-&gt;record_stack_base_and_size();

  // Initialize thread local storage; set before calling MutexLocker
  this-&gt;initialize_thread_local_storage();

  this-&gt;create_stack_guard_pages();

  this-&gt;cache_global_variables();

  // Thread is now sufficient initialized to be handled by the safepoint code as being
  // in the VM. Change thread state from _thread_new to _thread_in_vm
  ThreadStateTransition::transition_and_fence(this, _thread_new, _thread_in_vm);

  assert(JavaThread::current() &#61;&#61; this, &#34;sanity check&#34;);
  assert(!Thread::current()-&gt;owns_locks(), &#34;sanity check&#34;);

  DTRACE_THREAD_PROBE(start, this);

  // This operation might block. We call that after all safepoint checks for a new thread has
  // been completed.
  this-&gt;set_active_handles(JNIHandleBlock::allocate_block());

  if (JvmtiExport::should_post_thread_life()) {
    JvmtiExport::post_thread_start(this);
  }

  EventThreadStart event;
  if (event.should_commit()) {
     event.set_javalangthread(java_lang_Thread::thread_id(this-&gt;threadObj()));
     event.commit();
  }

  // We call another function to do the rest so we are sure that the stack addresses used
  // from there will be lower than the stack base just computed
  thread_main_inner();

  // Note, thread is no longer valid at this point!
}
</code></pre> 
<p> </p> 
<pre class="has"><code>

void JavaThread::thread_main_inner() {
  assert(JavaThread::current() &#61;&#61; this, &#34;sanity check&#34;);
  assert(this-&gt;threadObj() !&#61; NULL, &#34;just checking&#34;);

  // Execute thread entry point unless this thread has a pending exception
  // or has been stopped before starting.
  // Note: Due to JVM_StopThread we can have pending exceptions already!
  if (!this-&gt;has_pending_exception() &amp;&amp;
      !java_lang_Thread::is_stillborn(this-&gt;threadObj())) {
    {
      ResourceMark rm(this);
      this-&gt;set_native_thread_name(this-&gt;get_thread_name());
    }
    HandleMark hm(this);
    this-&gt;entry_point()(this, this);
  }

  DTRACE_THREAD_PROBE(stop, this);

  this-&gt;exit(false);
  delete this;
}

</code></pre> 
<pre class="has"><code>    this-&gt;entry_point()(this, this);</code></pre> 
<p>这个函数指着应该就是直接调用java层Thread 的run了&#xff0c;他是在JavaThread的构造中被赋值的&#xff0c;金身就是这个函数</p> 
<pre class="has"><code class="language-cpp">static void thread_entry(JavaThread* thread, TRAPS) {
  HandleMark hm(THREAD);
  Handle obj(THREAD, thread-&gt;threadObj());
  JavaValue result(T_VOID);
  JavaCalls::call_virtual(&amp;result,
                          obj,
                          KlassHandle(THREAD, SystemDictionary::Thread_klass()),
                          vmSymbols::run_method_name(),
                          vmSymbols::void_method_signature(),
                          THREAD);
}
</code></pre> 
<p>到这里就实现了在新线程内运行了run函数</p>
 