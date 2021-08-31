---
layout:     post
title:      Android基础探究之Handler深入学习
subtitle:   之前对handler有过一次学习总结，这次从源码角度再次深入学习一下
date:       2019-04-11
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android
    - 组件学习
---


 <p>Handler作为Android应用层开发&#xff0c;线程通信一大重点&#xff0c;可以说是使用最频繁的一个机制&#xff0c;不管是IntentService,ThreadHandler都绕不开它。本文详解Handler机制的内部源码</p> 


<p><strong>带着以下几个问题我们来开始本篇文章</strong></p> 
<ol><li> <p><em>UI线程的Looper在哪里创建&#xff1f;</em></p> </li><li> <p><em>MessageQueue真的是个队列吗&#xff1f;</em></p> </li><li> <p><em>延迟处理机制的原理&#xff1f;</em></p> </li><li> <p><em>Handler中的Message同步和MessageQueue同步&#xff1f;</em><br /> </p>
  
<h2><a id="Handler_15"></a>一、Handler介绍</h2> 
<p>Handler在Android os包下&#xff0c;当我们创建Handler时&#xff0c;它会绑定一个线程&#xff0c;并且创建一个消息队列&#xff0c;通过发送Message或者Runnable对象到队列并轮询取出&#xff0c;实现关联。<br /> 我们常用的Handler功能是&#xff0c;定时执行Runnable或者处理不同线程通信的问题&#xff0c;比如UI线程和子线程等。<br /> 由此可见Handler内部机制中的几大元素&#xff1a;Handler,Message,MessageQueue,Looper,ThreadLocal等&#xff0c;接下来&#xff0c;分别查看它的内部源码。<br /> <img src="https://i.imgur.com/GjFob24.png" alt="" /></p> 
<h2><a id="Handler_21"></a>二、Handler源码剖析</h2> 
<p>Handler作为封装对外的处理器&#xff0c;我们来看看它对外的接口内部是做了哪些操作。</p> 
<h4><a id="1_Handler_23"></a>1. Handler构造函数&#xff1a;</h4> 
<p>它的构造函数,我归纳为三种方式&#xff0c;分别是&#xff1a;<br /> 1.传入自定义Looper对象&#xff0c;<br /> 2.继承Handler实现Callback接口模式&#xff0c;<br /> 3.默认创建的Looper模式&#xff0c;<br /> 其中2&#xff0c;3是我们常用的&#xff0c;当然1和2也能同时使用&#xff0c;callback接口中实现handleMessage&#xff0c;用于我们自定义Handler是实现回调用的。还有个被hide隐藏的传参&#xff0c;<strong>async</strong>是否同步&#xff0c;默认是不同步&#xff0c;且不支持设置同步模式。</p> 
<p><strong>可以传入自定义的Looper&#xff0c;Callback接口</strong></p> 
<pre><code>public Handler(Looper looper, Callback callback, boolean async) {
	mLooper &#61; looper;
	mQueue &#61; looper.mQueue;
	mCallback &#61; callback;
	mAsynchronous &#61; async;
}
</code></pre> 
<p><strong>常规的构造方法如下:</strong></p> 
<ul><li> <p>其中FIND_POTENTIAL_LEAKS标签是检查“继承类是否为非静态内部类”标签&#xff0c;我们知道&#xff0c;非静态内部类持有对象&#xff0c;容易导致内存泄漏的问题&#xff0c;可以查看我的《Android内存优化分析篇》</p> </li><li> <p>mAsynchronous可以看到一直是false</p> <pre><code>   public Handler(Callback callback, boolean async) {
      if (FIND_POTENTIAL_LEAKS) {
          final Class&lt;? extends Handler&gt; klass &#61; getClass();
          if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &amp;&amp;
                  (klass.getModifiers() &amp; Modifier.STATIC) &#61;&#61; 0) {
              Log.w(TAG, &#34;The following Handler class should be static or leaks might occur: &#34; &#43;
                  klass.getCanonicalName());
          }
      }


      mLooper &#61; Looper.myLooper();
      if (mLooper &#61;&#61; null) {
          throw new RuntimeException(
              &#34;Can&#39;t create handler inside thread that has not called Looper.prepare()&#34;);
      }
      mQueue &#61; mLooper.mQueue;
      mCallback &#61; callback;
      mAsynchronous &#61; async;
  }
</code></pre> </li></ul> 
<h4><a id="2_LoopermQueue_63"></a>2. 创建Looper对象和mQueue消息队列</h4> 
<p>由上构造函数中调用Looper.myLooper()&#xff1b;创建了Looper对象&#xff0c;并取用了新创建Looper对象内部的mQueue队列&#xff0c;详解下Looper分析</p> 
<h4><a id="3_sendMessage_65"></a>3. sendMessage</h4> 
<ul><li> <p>其中sendEmptyMessage通过obtion新获取了一个Message对象</p> <pre><code>  public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
      Message msg &#61; Message.obtain();
      msg.what &#61; what;
      return sendMessageDelayed(msg, delayMillis);
  }
</code></pre> </li><li> <p>发送消息&#xff1a;sendMessageDelayed—&gt;sendMessageAtTime—&gt;enqueueMessage</p> </li><li> <p>注意到&#xff0c;在调用sendMessageAtTime时&#xff0c;传入的时间值&#xff1a; <strong>系统时钟&#43;delayMillis</strong></p> </li><li> <p>其中将 <strong>msg.target标记为当前Handler对象</strong></p> </li><li> <p>最终调用了MessageQueue的enqueueMessage&#xff0c;看后面MessageQueue分析</p> <pre><code>  //----------1
  public final boolean sendMessage(Message msg)
  {
      return sendMessageDelayed(msg, 0);
  }
  //----------2
   public final boolean sendMessageDelayed(Message msg, long delayMillis)
  {
      if (delayMillis &lt; 0) {
          delayMillis &#61; 0;
      }
      return sendMessageAtTime(msg, SystemClock.uptimeMillis() &#43; delayMillis);
  }
  //---------3
  public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
      MessageQueue queue &#61; mQueue;
      if (queue &#61;&#61; null) {
          RuntimeException e &#61; new RuntimeException(
                  this &#43; &#34; sendMessageAtTime() called with no mQueue&#34;);
          Log.w(&#34;Looper&#34;, e.getMessage(), e);
          return false;
      }
      return enqueueMessage(queue, msg, uptimeMillis);
  }
  //----------4
  private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
      msg.target &#61; this;
      if (mAsynchronous) {
          msg.setAsynchronous(true);
      }
      return queue.enqueueMessage(msg, uptimeMillis);
  }
</code></pre> </li></ul> 
<h4><a id="4_removeMessages_112"></a>4. removeMessages</h4> 
<p>从队列删除</p> 
<h4><a id="5_postRunnable_r_115"></a>5. post(Runnable r)</h4> 
<ul><li> <p>在getPostMessage中讲Runnable封装成了Message对象</p> <pre><code>  public final boolean post(Runnable r)
  {
     return  sendMessageDelayed(getPostMessage(r), 0);
  }

   private static Message getPostMessage(Runnable r) {
      Message m &#61; Message.obtain();
      m.callback &#61; r;
      return m;
  }
</code></pre> </li></ul> 
<h4><a id="6_dispatchMessagehandlerMessage_129"></a>6. dispatchMessage和handlerMessage</h4> 
<ul><li> <p>我们看到dispatchMessage调用了callback和handlerMessage分发Message结果</p> </li><li> <p>那么&#xff0c;前面我们看到了经常调用的sendMessage&#xff0c;那么回调是在什么时候调用的呢&#xff1f;</p> </li><li> <p>让我们接下来一起看看Looper类吧。</p> <pre><code>  public void dispatchMessage(Message msg) {
      if (msg.callback !&#61; null) {
          handleCallback(msg);
      } else {
          if (mCallback !&#61; null) {
              if (mCallback.handleMessage(msg)) {
                  return;
              }
          }
          handleMessage(msg);
      }
  }

   public void handleMessage(Message msg) {}
</code></pre> </li></ul> 
<h2><a id="Looper_149"></a>三、Looper源码剖析</h2> 
<p>看looper做了什么&#xff0c;首先看mylooper方法&#xff0c;还记得吗&#xff0c;在Handler初始化时创建Looper对象调用的方法</p> 
<h4><a id="1_myLooper_151"></a>1. myLooper方法</h4> 
<ul><li> <p>调用sThreadLocal取出一个looper对象</p> <pre><code>  public static &#64;Nullable Looper myLooper() {
          return sThreadLocal.get();
    }

  // sThreadLocal.get() will return null unless you&#39;ve called prepare().
  static final ThreadLocal&lt;Looper&gt; sThreadLocal &#61; new ThreadLocal&lt;Looper&gt;();
</code></pre> </li></ul> 
<h4><a id="2_Looperprepare_161"></a>2. Looper.prepare()创建对象</h4> 
<ul><li> <p>上面看到mylooper从sThreadLocal取出&#xff0c;但是什么时候存的呢&#xff0c;looper又是如何创建&#xff1f;</p> </li><li> <p>由下看出Looper通过prepare创建并存入sThreadLocal&#xff0c;在构造同时创建<strong>MessageQueue</strong></p> </li><li> <p>标记成员mThread为当前线程</p> </li><li> <p><strong>quitAllowed</strong>标识能否安全退出</p> <pre><code>  public static void prepare() {
      	prepare(true);
    }

  private static void prepare(boolean quitAllowed) {
      if (sThreadLocal.get() !&#61; null) {
          throw new RuntimeException(&#34;Only one Looper may be created per thread&#34;);
      }
      sThreadLocal.set(new Looper(quitAllowed));
  }

  private Looper(boolean quitAllowed) {
      mQueue &#61; new MessageQueue(quitAllowed);
      mThread &#61; Thread.currentThread();
  }
</code></pre> </li></ul> 
<h4><a id="3_UIHandlerLooper_183"></a>3. UI线程调用Handler&#xff0c;Looper怎么创建</h4> 
<ul><li> <p>prepareMainLooper&#xff1a;在当前线程初始化looper&#xff0c;在ActivityThread调用&#xff0c;也就是我们创建Activity时已经创建了Looper了</p> </li><li> <p>prepare(false)&#xff1a;由于在ActivityThread创建&#xff0c;是不能安全退出的</p> <pre><code>  /**
   * Initialize the current thread as a looper, marking it as an
   * application&#39;s main looper. The main looper for your application
   * is created by the Android environment, so you should never need
   * to call this function yourself.  See also: {&#64;link #prepare()}
   */
  public static void prepareMainLooper() {
      prepare(false);
      synchronized (Looper.class) {
          if (sMainLooper !&#61; null) {
              throw new IllegalStateException(&#34;The main Looper has already been prepared.&#34;);
          }
          sMainLooper &#61; myLooper();
      }
  }

   //--------&gt;ActivityThread: Main:
   public static void main(String[] args) {

      ---

      Looper.prepareMainLooper();

      ActivityThread thread &#61; new ActivityThread();
      thread.attach(false);

      if (sMainThreadHandler &#61;&#61; null) {
          sMainThreadHandler &#61; thread.getHandler();
      }

      if (false) {
          Looper.myLooper().setMessageLogging(new
                  LogPrinter(Log.DEBUG, &#34;ActivityThread&#34;));
      }

      // End of event ActivityThreadMain.
      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
      Looper.loop();
  }
</code></pre> </li></ul> 
<h4><a id="4_Looperloop_228"></a>4. Looper.loop()</h4> 
<ul><li>UI线程创建Looper&#xff0c;上ActivityThread中&#xff0c;在调用prepare后接着调用Looper.loop</li><li>loop通过 for (;;)死循环&#xff0c;从queue中取下一则消息</li><li>其中 msg.target.dispatchMessage(msg);&#xff0c;在上面Handler中将handler对象传给了looper</li><li></li></ul> 
<pre><code>	public static void loop() {
        final Looper me &#61; myLooper();
        if (me &#61;&#61; null) {
            throw new RuntimeException(&#34;No Looper; Looper.prepare() wasn&#39;t called on this thread.&#34;);
        }
        final MessageQueue queue &#61; me.mQueue;

		//--------------确保同一进程
        Binder.clearCallingIdentity();
        final long ident &#61; Binder.clearCallingIdentity();

        for (;;) {
            Message msg &#61; queue.next(); // might block
            if (msg &#61;&#61; null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            //--------------打印日志
            final Printer logging &#61; me.mLogging;
            if (logging !&#61; null) {
                logging.println(&#34;&gt;&gt;&gt;&gt;&gt; Dispatching to &#34; &#43; msg.target &#43; &#34; &#34; &#43;
                        msg.callback &#43; &#34;: &#34; &#43; msg.what);
            }
			
			//--------------从队列中获取分发消息延时
            final long slowDispatchThresholdMs &#61; me.mSlowDispatchThresholdMs;
			
			//--------------Trace标记&#xff0c;用于记录message分发完成
            final long traceTag &#61; me.mTraceTag;
            if (traceTag !&#61; 0 &amp;&amp; Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
            final long start &#61; (slowDispatchThresholdMs &#61;&#61; 0) ? 0 : SystemClock.uptimeMillis();
            final long end;
            try {
                msg.target.dispatchMessage(msg);
                end &#61; (slowDispatchThresholdMs &#61;&#61; 0) ? 0 : SystemClock.uptimeMillis();
            } finally {
                if (traceTag !&#61; 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            if (slowDispatchThresholdMs &gt; 0) {
                final long time &#61; end - start;
                if (time &gt; slowDispatchThresholdMs) {
                    Slog.w(TAG, &#34;Dispatch took &#34; &#43; time &#43; &#34;ms on &#34;
                            &#43; Thread.currentThread().getName() &#43; &#34;, h&#61;&#34; &#43;
                            msg.target &#43; &#34; cb&#61;&#34; &#43; msg.callback &#43; &#34; msg&#61;&#34; &#43; msg.what);
                }
            }

            if (logging !&#61; null) {
                logging.println(&#34;&lt;&lt;&lt;&lt;&lt; Finished to &#34; &#43; msg.target &#43; &#34; &#34; &#43; msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn&#39;t corrupted.
            final long newIdent &#61; Binder.clearCallingIdentity();
            if (ident !&#61; newIdent) {
                Log.wtf(TAG, &#34;Thread identity changed from 0x&#34;
                        &#43; Long.toHexString(ident) &#43; &#34; to 0x&#34;
                        &#43; Long.toHexString(newIdent) &#43; &#34; while dispatching to &#34;
                        &#43; msg.target.getClass().getName() &#43; &#34; &#34;
                        &#43; msg.callback &#43; &#34; what&#61;&#34; &#43; msg.what);
            }
			
			//--------------充值message对象参数
            msg.recycleUnchecked();
        }
    }
</code></pre> 
<h2><a id="MessageQueue_308"></a>四、MessageQueue源码剖析</h2> 
<p>MessageQueue主要分析插入和取出&#xff0c;由下enqueueMessage插入方法看出&#xff0c;它名字带着Queue,但其实并不是&#xff0c;它实际是个单链表结构&#xff0c;通过native操作指针&#xff0c;去进行msg的读取操作。当然&#xff0c;这更加快捷的实施取出&#xff0c;删除和插入操作。</p> 
<h4><a id="1_enqueueMessage_310"></a>1. enqueueMessage</h4> 
<ul><li> <p>msg.markInUse();标记当前msg正在使用</p> </li><li> <p>其中<strong>mMessages</strong>是可以理解为<strong>即将执行的Message对象</strong></p> </li><li> <p>将当前<strong>mMessages</strong>与<strong>新传入的Msg</strong>的<strong>设置触发时间对比</strong>&#xff0c;如果新的Msg设置时间早&#xff0c;则将2者位置对调&#xff0c;将新的排前面&#xff0c;与之对比的mMessages排到其后。反之&#xff0c;则与mMessages后一个对比时间&#xff0c;依次类比&#xff0c;插入到队列中</p> </li><li> <p>其中&#xff0c;如果msg事Asynchronous同步的&#xff0c;那么它只能等到上一个同步msg执行完&#xff0c;才能被唤醒执行。</p> <pre><code>  boolean enqueueMessage(Message msg, long when) {
 	...

     synchronized (this) {
         if (mQuitting) {
             //-------------&gt;抛出一个IllegalStateException
             Log.w(TAG, e.getMessage(), e);
             msg.recycle();
             return false;
         }
 		//-------------&gt;标记当前msg正在使用
         msg.markInUse();
         msg.when &#61; when;
         Message p &#61; mMessages;
         boolean needWake;
         if (p &#61;&#61; null || when &#61;&#61; 0 || when &lt; p.when) {
             // New head, wake up the event queue if blocked.
             msg.next &#61; p;
             mMessages &#61; msg;
             needWake &#61; mBlocked;
         } else {
             // Inserted within the middle of the queue.  Usually we don&#39;t have to wake
             // up the event queue unless there is a barrier at the head of the queue
             // and the message is the earliest asynchronous message in the queue.
             needWake &#61; mBlocked &amp;&amp; p.target &#61;&#61; null &amp;&amp; msg.isAsynchronous();
             Message prev;
             for (;;) {
                 prev &#61; p;
                 p &#61; p.next;
                 if (p &#61;&#61; null || when &lt; p.when) {
                     break;
                 }
                 if (needWake &amp;&amp; p.isAsynchronous()) {
                     needWake &#61; false;
                 }
             }
             msg.next &#61; p; // invariant: p &#61;&#61; prev.next
             prev.next &#61; msg;
         }

         // We can assume mPtr !&#61; 0 because mQuitting is false.
         if (needWake) {
             nativeWake(mPtr);
         }
     }
     return true;
 }
</code></pre> </li></ul> 
<h4><a id="2_next_364"></a>2. next取出</h4> 
<ul><li> <p>可以看出enqueueMessage和next都是同步的</p> </li><li> <p>通过循环&#xff0c;把mMessages当前msg</p> </li><li> <p>比较当前时间和Msg标记时间&#xff0c;如果早的话就设置一段指针查找超时时间</p> </li><li> <p>将msg标记为使用&#xff0c;并取出消息返回</p> <pre><code>  Message next() {

      //-----&gt;当消息轮询退出时&#xff0c;mPtr指针找不到地址&#xff0c;返回空取不到对象
      final long ptr &#61; mPtr;
      if (ptr &#61;&#61; 0) {
          return null;
      }
  	//-----&gt;同步指针查找的时间&#xff0c;根据超时时间计算&#xff0c;比如当前未到msg的时间&#xff0c;指针会在一段计算好的超时时间后去查询
      int pendingIdleHandlerCount &#61; -1; // -1 only during first iteration
      int nextPollTimeoutMillis &#61; 0;
      for (;;) {
          if (nextPollTimeoutMillis !&#61; 0) {
              Binder.flushPendingCommands();
          }

          nativePollOnce(ptr, nextPollTimeoutMillis);

          synchronized (this) {
              // Try to retrieve the next message.  Return if found.
              final long now &#61; SystemClock.uptimeMillis();
              Message prevMsg &#61; null;
              Message msg &#61; mMessages;
  				
  			//-----&gt;如果target为null,寻找下一个带“同步”标签的msg

              if (msg !&#61; null &amp;&amp; msg.target &#61;&#61; null) {
                  // Stalled by a barrier.  Find the next asynchronous message in the queue.
                  do {
                      prevMsg &#61; msg;
                      msg &#61; msg.next;
                  } while (msg !&#61; null &amp;&amp; !msg.isAsynchronous());
              }
              if (msg !&#61; null) {

  				//-----&gt;比较当前时间和Msg标记时间&#xff0c;如果早的话就设置一段指针查找超时时间

                  if (now &lt; msg.when) {
                      // Next message is not ready.  Set a timeout to wake up when it is ready.
                      nextPollTimeoutMillis &#61; (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                  } else {
                      // Got a message.
                      mBlocked &#61; false;
                      if (prevMsg !&#61; null) {
                          prevMsg.next &#61; msg.next;
                      } else {
                          mMessages &#61; msg.next;
                      }
                      msg.next &#61; null;
                      if (DEBUG) Log.v(TAG, &#34;Returning message: &#34; &#43; msg);
                      msg.markInUse();
                      return msg;
                  }
              } else {
                  // No more messages.
                  nextPollTimeoutMillis &#61; -1;
              }

              // Process the quit message now that all pending messages have been handled.
              if (mQuitting) {
                  dispose();
                  return null;
              }

              // If first time idle, then get the number of idlers to run.
              // Idle handles only run if the queue is empty or if the first message
              // in the queue (possibly a barrier) is due to be handled in the future.
              if (pendingIdleHandlerCount &lt; 0
                      &amp;&amp; (mMessages &#61;&#61; null || now &lt; mMessages.when)) {
                  pendingIdleHandlerCount &#61; mIdleHandlers.size();
              }
              if (pendingIdleHandlerCount &lt;&#61; 0) {
                  // No idle handlers to run.  Loop and wait some more.
                  mBlocked &#61; true;
                  continue;
              }

              if (mPendingIdleHandlers &#61;&#61; null) {
                  mPendingIdleHandlers &#61; new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
              }
              mPendingIdleHandlers &#61; mIdleHandlers.toArray(mPendingIdleHandlers);
          }

          // Run the idle handlers.
          // We only ever reach this code block during the first iteration.
          for (int i &#61; 0; i &lt; pendingIdleHandlerCount; i&#43;&#43;) {
              final IdleHandler idler &#61; mPendingIdleHandlers[i];
              mPendingIdleHandlers[i] &#61; null; // release the reference to the handler

              boolean keep &#61; false;
              try {
                  keep &#61; idler.queueIdle();
              } catch (Throwable t) {
                  Log.wtf(TAG, &#34;IdleHandler threw exception&#34;, t);
              }

              if (!keep) {
                  synchronized (this) {
                      mIdleHandlers.remove(idler);
                  }
              }
          }

          // Reset the idle handler count to 0 so we do not run them again.
          pendingIdleHandlerCount &#61; 0;

          // While calling an idle handler, a new message could have been delivered
          // so go back and look again for a pending message without waiting.
          nextPollTimeoutMillis &#61; 0;
      }
  }
</code></pre> </li></ul> 
<h4><a id="3_quit_480"></a>3. quit操作</h4> 
<ul><li> <p>前面标记是否能安全退出&#xff0c;否则报错</p> </li><li> <p>退出后唤醒指针&#xff0c;接触msg的锁</p> <pre><code>   void quit(boolean safe) {
          if (!mQuitAllowed) {
              throw new IllegalStateException(&#34;Main thread not allowed to quit.&#34;);
          }
  
          synchronized (this) {
              if (mQuitting) {
                  return;
              }
              mQuitting &#61; true;
  
              if (safe) {
                  removeAllFutureMessagesLocked();
              } else {
                  removeAllMessagesLocked();
              }
  
              // We can assume mPtr !&#61; 0 because mQuitting was previously false.
              nativeWake(mPtr);
          }
      }
</code></pre> </li></ul> 
<h2><a id="Message_507"></a>五、Message源码剖析</h2> 
<p>Message主要是一个Parcelable序列号对象&#xff0c;封装了不分信息和操作&#xff0c;它构造了一个对象池&#xff0c;这也是为什么我们一直发送msg&#xff0c;不会内存爆炸的原因&#xff0c;来看看实现</p> 
<h4><a id="1_obtain_509"></a>1. obtain()</h4> 
<ul><li> <p>维持一个大小为50的同步线程池</p> </li><li> <p>这里可以看出Message是个链表结构&#xff0c;obtain将sPool取出return Message&#xff0c;并对象池下一个msg标记为sPool</p> <pre><code>  private static final int MAX_POOL_SIZE &#61; 50;
  
  ...

  public static Message obtain() {
      synchronized (sPoolSync) {
          if (sPool !&#61; null) {
              Message m &#61; sPool;
              sPool &#61; m.next;
              m.next &#61; null;
              m.flags &#61; 0; // clear in-use flag
              sPoolSize--;
              return m;
          }
      }
      return new Message();
  }
</code></pre> </li></ul> 
<h4><a id="2recycleUnchecked__531"></a>2.recycleUnchecked 回收消息</h4> 
<ul><li> <p>回收初始化当前msg</p> </li><li> <p>如果当前对象池大小小于MAX_POOL_SIZE&#xff0c;则将初始化后的msg放到表头sPool&#xff0c;sPoolSize&#43;&#43;。</p> </li><li> <p>由此可以看出&#xff0c;如果每次new新的Message传入Handler&#xff0c;必然增加内存消耗&#xff0c;通过obtain服用才是正确的做法</p> <pre><code>  /**
   * Recycles a Message that may be in-use.
   * Used internally by the MessageQueue and Looper when disposing of queued Messages.
   */
  void recycleUnchecked() {
      // Mark the message as in use while it remains in the recycled object pool.
      // Clear out all other details.
      flags &#61; FLAG_IN_USE;
      what &#61; 0;
      arg1 &#61; 0;
      arg2 &#61; 0;
      obj &#61; null;
      replyTo &#61; null;
      sendingUid &#61; -1;
      when &#61; 0;
      target &#61; null;
      callback &#61; null;
      data &#61; null;

      synchronized (sPoolSync) {
          if (sPoolSize &lt; MAX_POOL_SIZE) {
              next &#61; sPool;
              sPool &#61; this;
              sPoolSize&#43;&#43;;
          }
      }
  }
</code></pre> </li></ul> 
<h4><a id="3_Message_564"></a>3. Message标签&#xff1a;是否使用&#xff0c;同步标签</h4> 
<pre><code>public void setAsynchronous(boolean async) {
    if (async) {
        flags |&#61; FLAG_ASYNCHRONOUS;
    } else {
        flags &amp;&#61; ~FLAG_ASYNCHRONOUS;
    }
}

/*package*/ boolean isInUse() {
    return ((flags &amp; FLAG_IN_USE) &#61;&#61; FLAG_IN_USE);
}

/*package*/ void markInUse() {
    flags |&#61; FLAG_IN_USE;
} 
</code></pre> 
<h2><a id="Message_507"></a>六、总结</h2> 
<p>相信阅读完本篇文章后，文章开头提出的几个问题，已经都有了答案。在此也可以发现Android源码的严谨性。平常只是简单一用，只有深入了解才能更好地去使用它，理解它。</p>