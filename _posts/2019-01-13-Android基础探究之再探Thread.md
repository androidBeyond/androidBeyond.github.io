---
layout:     post
title:      Android基础探究之再探Thread
subtitle:   对android中的thread实现原理进行深入的学习
date:       2019-01-13
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android
    - 组件学习
--- 

<h3><a id="_3"></a>概述</h3> 
<p>对常用的Thread做一次源码剖析&#xff0c;更好的去理解和使用它&#xff0c;看完之后你会明白的几个问题&#xff1a;</p> 
<ol><li>调用start发生了什么&#xff1f;多次调用start会怎么样&#xff1f;</li><li>start和run方法的区别</li><li>join和sleep的区别</li><li>什么是守护进程</li></ol> 
<h3><a id="_12"></a>一、创建使用</h3> 
<h4><a id="1__13"></a>1. 初始化</h4> 
<blockquote> 
 <p>Thread构造函数</p> 
 <p>内部调用—&gt;init&#xff08;&#xff09;方法</p> 
</blockquote> 
<pre><code>java.lang.Thread#Thread()
java.lang.Thread#Thread(java.lang.Runnable)
java.lang.Thread#Thread(java.lang.ThreadGroup, java.lang.Runnable)
java.lang.Thread#Thread(java.lang.String)
java.lang.Thread#Thread(java.lang.ThreadGroup, java.lang.String)
java.lang.Thread#Thread(java.lang.ThreadGroup, java.lang.String, int, boolean)
java.lang.Thread#Thread(java.lang.Runnable, java.lang.String)
java.lang.Thread#Thread(java.lang.ThreadGroup, java.lang.Runnable, java.lang.String)
java.lang.Thread#Thread(java.lang.ThreadGroup, java.lang.Runnable, java.lang.String, long)
</code></pre> 
<blockquote> 
 <p>init&#xff08;&#xff09;方法指定四个参数&#xff1a;ThreadGroup&#xff0c;任务runable&#xff0c;线程名称&#xff0c;栈大小&#xff0c;其中部分参数初始值都是继承父线程的属性</p> 
</blockquote> 
<pre><code>private void init(ThreadGroup g, Runnable target, String name, long stackSize) {
    Thread parent &#61; currentThread();//获取创建thread的线程
    if (g &#61;&#61; null) {
        g &#61; parent.getThreadGroup();
    }

    g.addUnstarted();//在ThreadGroup中标记增加了一个未启动的线程&#xff0c;里面操作很简单&#xff0c;nUnstartedThreads&#43;&#43;;
    this.group &#61; g;

    this.target &#61; target;
    this.priority &#61; parent.getPriority();//继承父线程的等级
    this.daemon &#61; parent.isDaemon();//继承父线程的属性&#xff1a;是否为守护进程
    setName(name);

    init2(parent);//保存一些常量参数&#xff0c;如上&#xff0c;给子线程调用

    /* Stash the specified stack size in case the VM cares */
    this.stackSize &#61; stackSize;
    tid &#61; nextThreadID();
}

...

//线程 tid递增一个
private static synchronized long nextThreadID() {
    return &#43;&#43;threadSeqNumber;
}
</code></pre> 
<h4><a id="2_start_58"></a>2. start方法</h4> 
<ul><li> <p>在Android中&#xff0c;检测到再次调用start线程会抛出IllegalThreadStateException</p> <pre><code>  public synchronized void start() {
      
      // Android-changed: throw if &#39;started&#39; is true
      if (threadStatus !&#61; 0 || started)
          throw new IllegalThreadStateException();

      //还记得上面init方法中&#xff0c;调用addUnstarted时&#xff0c;标记增加了未启动线程
  	//这里调用add方法&#xff0c;将线程添加到系统线程数组&#xff0c;并且将未启动线程数减一&#xff0c;相当于移出
      group.add(this);

      started &#61; false;
      try {
          nativeCreate(this, stackSize, daemon);
  		//调用native方法启动线程&#xff0c;如果报错&#xff0c;则直接跳到finally执行&#xff0c;started为false,
  		//启动失败&#xff0c;从group中移除&#xff0c;同时group中未启动线程数&#43;&#43;
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
</code></pre> </li></ul> 
<h4><a id="3_run_89"></a>3. run方法</h4> 
<pre><code>//Thread实现的Runnable接口
class Thread implements Runnable {

	...
	
	//调用传入的Runnable的run方法
	&#64;Override
    public void run() {
        if (target !&#61; null) {
            target.run();
        }
    }
</code></pre> 
<h3><a id="Thread_105"></a>二、Thread阻塞</h3> 
<h4><a id="1_join_106"></a>1. join方法</h4> 
<p>join方法用于等待线程执行完成&#xff0c;传入的时间单位为等待的最大时长&#xff0c;里面是一个 while (isAlive())循环函数&#xff0c;当不传入时间参数&#xff0c;则为永久等待直到线程结束&#xff0c;传入时间参数&#xff0c;当时间到达时会结束join方法</p> 
<pre><code>public final void join(long millis) throws InterruptedException {
    synchronized(lock) {
        long base &#61; System.currentTimeMillis();
        long now &#61; 0;

        if (millis &lt; 0) {
            throw new IllegalArgumentException(&#34;timeout value is negative&#34;);
        }

        if (millis &#61;&#61; 0) {
            while (isAlive()) {
                lock.wait(0);
            }
        } else {
			//循环&#xff0c;当达到最大等待时常&#xff0c;则跳出循环
            while (isAlive()) {
                long delay &#61; millis - now;
                if (delay &lt;&#61; 0) {
                    break;
                }
                lock.wait(delay);
                now &#61; System.currentTimeMillis() - base;
            }
        }
    }
}
 public final void join() throws InterruptedException {
    join(0);
}
//等待多少毫秒在加多少纳秒
public final void join(long millis, int nanos)
throws InterruptedException {
    synchronized(lock) {
    if (millis &lt; 0) {
        throw new IllegalArgumentException(&#34;timeout value is negative&#34;);
    }

    if (nanos &lt; 0 || nanos &gt; 999999) {
        throw new IllegalArgumentException(
                            &#34;nanosecond timeout value out of range&#34;);
    }

    if (nanos &gt;&#61; 500000 || (nanos !&#61; 0 &amp;&amp; millis &#61;&#61; 0)) {
        millis&#43;&#43;;
    }

    join(millis);
    }
}
</code></pre> 
<h4><a id="2_sleep_160"></a>2. sleep方法</h4> 
<p>sleep作用是使当前线程睡眠指定时间&#xff0c;其中几个关键点</p> 
<ul><li> <p>获取当前调用线程的lock&#xff1a;currentThread().lock;</p> </li><li> <p>通过while (true)循环sleep当前线程&#xff0c;并检测睡眠时间达到传输参数时间&#xff0c;break当前循环</p> <pre><code>  public static void sleep(long millis, int nanos)throws InterruptedException {
      if (millis &lt; 0) {
          throw new IllegalArgumentException(&#34;millis &lt; 0: &#34; &#43; millis);
      }
      if (nanos &lt; 0) {
          throw new IllegalArgumentException(&#34;nanos &lt; 0: &#34; &#43; nanos);
      }
      if (nanos &gt; 999999) {
          throw new IllegalArgumentException(&#34;nanos &gt; 999999: &#34; &#43; nanos);
      }

     	//当睡眠时间为0&#xff0c;先检测线程是否已经中断&#xff0c;是的话抛出异常&#xff0c;否则直接return
      if (millis &#61;&#61; 0 &amp;&amp; nanos &#61;&#61; 0) {
          // ...but we still have to handle being interrupted.
          if (Thread.interrupted()) {
            throw new InterruptedException();
          }
          return;
      }

      long start &#61; System.nanoTime();
      long duration &#61; (millis * NANOS_PER_MILLI) &#43; nanos;

  	获取当前线程的lock
      Object lock &#61; currentThread().lock;

      // Wait may return early, so loop until sleep duration passes.
      synchronized (lock) {
          while (true) {
              sleep(lock, millis, nanos);

              long now &#61; System.nanoTime();
              long elapsed &#61; now - start;

              if (elapsed &gt;&#61; duration) {
                  break;
              }

              duration -&#61; elapsed;
              start &#61; now;
              millis &#61; duration / NANOS_PER_MILLI;
              nanos &#61; (int) (duration % NANOS_PER_MILLI);
          }
      }
  }
  public static void sleep(long millis) throws InterruptedException {
      Thread.sleep(millis, 0);
  }

  &#64;FastNative
  private static native void sleep(Object lock, long millis, int nanos)
      throws InterruptedException;
</code></pre> </li></ul> 
<h4><a id="3sleepjoin_220"></a>3.sleep与join的区别</h4> 
<ol><li>join里面调用的wait方法&#xff0c;wait方法可以释放锁&#xff0c;而sleep方法是持有锁</li><li>join&#xff08;0&#xff09;是一直等待线程执行完成&#xff0c;只有这个线程执行完后&#xff0c;才能执行其他线程&#xff0c;中间通过循环lock.wait(delay)实现&#xff0c;它是非静态方法&#xff0c;</li><li>sleep是静态方法&#xff0c;通过currentThread获取当前线程的lock&#xff0c;它只能作用当前线程</li></ol> 
<h3><a id="Thread_224"></a>三、Thread终止</h3> 
<h4><a id="1_stop_225"></a>1. stop方法</h4> 
<p>stop方法以及被弃用&#xff0c;强行调用的话会抛出UnsupportedOperationException异常</p> 
<pre><code> &#64;Deprecated
public final void stop() {
    stop(new ThreadDeath());
}

  &#64;Deprecated
public final void stop(Throwable obj) {
    throw new UnsupportedOperationException();
}
</code></pre> 
<h4><a id="2_interrupt_238"></a>2. interrupt方法</h4> 
<p><strong>部分内容引用一篇很详细的文章&#xff0c;戳–&gt;</strong><a href="https://www.jianshu.com/p/1492434f2810" title="https://www.jianshu.com/p/1492434f2810">《Java线程源码解析之interrupt》</a></p> 
<blockquote> 
 <ul><li> <p>interrupt的作用是中断线程&#xff0c;我们经常调用&#xff0c;interrupt的使用有几个注意点</p> </li><li> <p>当线程处于wait,sleep,join等方法阻塞状态时&#xff0c;它会清除当前阻塞状态&#xff0c;并抛出InterruptedException异常</p> </li><li> <p>在I/O通讯状态中调用interrupt&#xff0c;数据通道会被关闭&#xff0c;并将线程状态标记为中断&#xff0c;并抛出ClosedByInterruptException异常</p> </li><li> <p>如果在java.nio.channels.Selector上堵塞&#xff0c;会标记中断状态&#xff0c;并马上返回select方法</p> </li><li> <p>Lock.lock()方法不会响应中断&#xff0c;Lock.lockInterruptibly()方法则会响应中断并抛出异常&#xff0c;区别在于park()等待被唤醒时lock会继续执行park()来等待锁&#xff0c;而 lockInterruptibly会抛出异常</p> </li><li> <p>synchronized被唤醒后会尝试获取锁&#xff0c;失败则会通过循环继续park()等待&#xff0c;因此实际上是不会被interrupt()中断的;</p> </li><li> <p>一般情况下&#xff0c;抛出异常时&#xff0c;会清空Thread的interrupt状态&#xff0c;在编程时需要注意&#xff1b;</p> </li></ul> 
</blockquote> 
<pre><code>//用来中断的IO通讯对象&#xff0c;在调用interrupt方法后会调用blocker的中断方法
private volatile Interruptible blocker;

public void interrupt() {
    if (this !&#61; Thread.currentThread())
        checkAccess();

    synchronized (blockerLock) {
        Interruptible b &#61; blocker;
        if (b !&#61; null) {
            nativeInterrupt();
            b.interrupt(this);
            return;
        }
    }
    nativeInterrupt();
}
</code></pre> 
<h3><a id="_272"></a>四、线程的状态</h3> 
<pre><code>public enum State {
    NEW,
    RUNNABLE,
    BLOCKED,
    WAITING,
    TIMED_WAITING,
    TERMINATED;
}

public State getState() {
    // get current thread state
    return State.values()[nativeGetStatus(started)];
}
</code></pre> 
<ul><li><strong>NEW</strong>&#xff1a;线程创建还未启动时状态</li><li><strong>RUNNABLE</strong>&#xff1a;线程运行状态&#xff0c;包括一些系统资源等待&#xff0c;如&#xff1a;IO等待&#xff0c;CPU时间片切换等</li><li><strong>BLOCKED</strong>&#xff1a;正在等待monitor lock的状态&#xff0c;比如&#xff1a;1. 即将进入synchronized方法或者块前等待获取锁的这个临界时期状态。2.调用wait方法释放锁之后再次进入synchronized方法或者块前的临界状态</li><li><strong>WAITING</strong>&#xff1a;基于上个BLOCKED状态来说&#xff0c;WAITING就是拿到锁了&#xff0c;处于wait过程中的状态&#xff0c;<strong>注意它是特指无限期的等待</strong>&#xff0c;也就是join()或者wait()等,它是join或者直接wait方法当获取到lock执行后&#xff0c;处于等待notify的WAITING状态。</li><li><strong>TIMED_WAITING</strong>&#xff1a;与上面WAITING相对&#xff0c;WAITING是指无限期的等待&#xff0c;TIMED_WAITING就是有限期的等待状态&#xff0c;包括join(long),wait(long),sleep(long)等。</li><li><strong>TERMINATED</strong>&#xff1a;线程执行完成&#xff0c;run结束的状态<br /> <img src="https://i.imgur.com/GCevkZb.png" alt="" /></li></ul> 
<h3><a id="_296"></a>五、总结&#xff1a;回答概述问题</h3> 
<ol><li>调用2次start时&#xff0c;看start源码中&#xff0c;里面判断如果当前线程状态和是否启动标记&#xff0c;<code>if (threadStatus !&#61; 0 || started)</code>&#xff0c;如果已经启动则抛出IllegalThreadStateException异常&#xff0c;可以通过继承Thread类或者实现Runnable去开启线程&#xff0c;这样每次new了新的对象启动线程</li><li>start是启动当前Thread线程&#xff0c;Thread实现了Runnable接口的run方法&#xff0c;当线程启动&#xff0c;run方法会被调用&#xff0c;Thread里面的Run会调用传入Runnable Target的run方法&#xff0c;达到实现我们自定义任务的目的。如果没有传入Runnable参数则do nothing</li><li>join是等待线程执行完成&#xff0c;方法通过内部一个while(alive)的循环函数去实现wait等待&#xff0c;alive是一直检测线程的存活状态&#xff0c;它相当于&#xff0c;在那个线程执行join&#xff0c;即在哪个线程执行wait,调用的线程对象可以理解为lock对象&#xff0c;即调用了lock.wait(), sleep方法是一直持有锁的状态&#xff0c;同时sleep是静态方法&#xff0c;它通过currentThread获取当前线程的lock&#xff0c;并只能作用当前线程</li><li>守护线程意思是后台服务线程&#xff0c;比如垃圾回收线程&#xff0c;要理解它就知道另一个用户线程&#xff0c;用户线程是维持程序运行状态&#xff0c;或者说jvm存活的线程&#xff0c;如果用户线程都跑完了&#xff0c;那么不管守护线程是否运行&#xff0c;程序和jvm都会退出&#xff0c;当然此时&#xff0c;守护线程也会退出&#xff0c;由此可以看出守护线程和用户线程对于程序运行的相关性。由上述线程的init方法可以看出&#xff0c;子线程的创建会继承一些默认参数&#xff0c;包含是否为守护线程&#xff0c;它是低级别的线程&#xff0c;不依赖于终端&#xff0c;但是依赖于系统&#xff0c;与系统“同生共死”。</li></ol>
                </div>
                <link href="https://csdnimg.cn/release/blogv2/dist/mdeditor/css/editerView/markdown_views-d7a94ec6ab.css" rel="stylesheet">
                <link href="https://csdnimg.cn/release/blogv2/dist/mdeditor/css/style-49037e4d27.css" rel="stylesheet">
        </div>
    </article>