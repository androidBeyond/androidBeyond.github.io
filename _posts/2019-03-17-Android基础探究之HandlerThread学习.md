---
layout:     post
title:      Android基础探究之HandlerThread学习
subtitle:   对android中的HandlerThread实现原理进行深入的学习
date:       2019-03-17
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android
    - 组件学习
---  


 <p>HandlerThread作为子线程管理常用类&#xff0c;他自带封装的Looper处理Message&#xff0c;可以说是十分实用。子线程调度任务&#xff0c;方便我们在子线程中做更多的花样。</p> 


<h3><a id="_4"></a>前言</h3> 
<p>HandlerThread内部实现很简单&#xff0c;主要用在需要进行子线程调度任务的时候创建&#xff0c;但是想要完善熟悉原理&#xff0c;你必须熟悉Handler的内部原理实现。

<ol><li> <p>HandlerThread的用法</p> </li><li> <p>HandlerThread内部原理</p> </li></ol> 
<h3><a id="HandlerThread_13"></a>HandlerThread用法</h3> 
<h4><a id="1_14"></a>1.创建使用</h4> 
<ul><li> <p>创建HandlerThread&#xff0c;HandlerThread其实就是继承的Thread类</p> </li><li> <p>将HandlerThread的looper绑定到Handler</p> <pre><code>  //创建HandlerThread
  mHandlerThread &#61; new HandlerThread(&#34;handlerThread-test&#34;);
  mHandlerThread.start();
  //将HandlerThread的looper绑定到Handler
  mHandler &#61; new Handler(mHandlerThread.getLooper(), new Handler.Callback() {
         &#64;Override
         public boolean handleMessage(Message msg) {
                  switch (msg.what) {
                           case DELAY_1000:
                                    String str &#61; (String) msg.obj;
                                    Log.d(&#34;handlerThread-test&#34;, &#34;looper : DELAY_1000 &#34; &#43; str &#43; &#34;   处理线程&#xff1a;&#34; &#43; Thread.currentThread().getName());
                                    break;
                           case DELAY_2000:
                                    String str1 &#61; (String) msg.obj;
                                    Log.d(&#34;handlerThread-test&#34;, &#34;looper : DELAY_2000 &#34; &#43; str1 &#43; &#34;   处理线程&#xff1a;&#34; &#43; Thread.currentThread().getName());
                                    break;
                           case DELAY_3000:
                                    String str2 &#61; (String) msg.obj;
                                    Log.d(&#34;handlerThread-test&#34;, &#34;looper : DELAY_3000 &#34; &#43; str2 &#43; &#34;   处理线程&#xff1a;&#34; &#43; Thread.currentThread().getName());
                                    break;
                           case DELAY_4000:
                                    String str3 &#61; (String) msg.obj;
                                    Log.d(&#34;handlerThread-test&#34;, &#34;looper : DELAY_4000 &#34; &#43; str3 &#43; &#34;   处理线程&#xff1a;&#34; &#43; Thread.currentThread().getName());
                                    break;
                  }
                  return false;
         }
  });
</code></pre> </li></ul> 
<h4><a id="2_47"></a>2.测试</h4> 
<ul><li> <p>在不同线程调度使用</p> <pre><code>  //测试
  new Thread(new Runnable() {
         &#64;Override
         public void run() {
                  try {
                           Thread.sleep(1000);
                  } catch (InterruptedException e) {
                           e.printStackTrace();
                  }
                  asyncMessage(DELAY_1000, 0, &#34;在子线程延时1s发送  调用线程&#xff1a;&#34; &#43; Thread.currentThread().getName());
         }
  }).start();
  asyncMessage(DELAY_2000, 2000, &#34;在主线程延时2s发送  调用线程&#xff1a;&#34; &#43; Thread.currentThread().getName());
  mHandler.post(new Runnable() {
         &#64;Override
         public void run() {
                  try {
                           Thread.sleep(3000);
                  } catch (InterruptedException e) {
                           e.printStackTrace();
                  }
                  asyncMessage(DELAY_3000, 0, &#34;在post Runnable中sleep 3s  调用线程&#xff1a;&#34; &#43; Thread.currentThread().getName());
                  asyncMessage(DELAY_4000, 1000, &#34;在DELAY_3000后延时1s发送  调用线程&#xff1a;&#34; &#43; Thread.currentThread().getName());
         }
  });

   private void asyncMessage(int what, int delay, String s) {
            Message message &#61; mHandler.obtainMessage();
            message.what &#61; what;
            message.obj &#61; s;
            mHandler.sendMessageDelayed(message, delay);
   }
</code></pre> </li><li> <p>结果&#xff1a;结果可以看出通过Handler对消息调度的处理&#xff0c;我们能在不同线程间进行通讯&#xff0c;并最终在HandlerThread异步处理</p> <pre><code>  handlerThread-test: looper : DELAY_1000 在子线程延时1s发送  调用线程&#xff1a;Thread-5   处理线程&#xff1a;handlerThread-test
  handlerThread-test: looper : DELAY_2000 在主线程延时2s发送  调用线程&#xff1a;main   处理线程&#xff1a;handlerThread-test
  handlerThread-test: looper : DELAY_3000 在post Runnable中sleep 3s  调用线程&#xff1a;handlerThread-test   处理线程&#xff1a;handlerThread-test
  handlerThread-test: looper : DELAY_4000 在DELAY_3000后延时1s发送  调用线程&#xff1a;handlerThread-test   处理线程&#xff1a;handlerThread-test
</code></pre> </li></ul> 
<h4><a id="3_90"></a>3.退出</h4> 
<pre><code>private void threadDestory() {	   
      if (mHandlerThread !&#61; null) {
               mHandlerThread.quitSafely();
               mHandlerThread &#61; null;
      }
}
</code></pre> 
<h3><a id="HandlerThread_99"></a>HandlerThread源码分析</h3> 
<p>查看HandlerThread内部源码&#xff0c;会发现&#xff0c;其实就只有一点代码&#xff0c;接下来一起分析吧</p> 
<h4><a id="1__102"></a>1. 构造函数中指定线程等级</h4> 
<pre><code>	 /**
     * Standard priority of application threads.
     * Use with {&#64;link #setThreadPriority(int)} and
     * {&#64;link #setThreadPriority(int, int)}, &lt;b&gt;not&lt;/b&gt; with the normal
     * {&#64;link java.lang.Thread} class.
     */
    public static final int THREAD_PRIORITY_DEFAULT &#61; 0;
	
	...

	public HandlerThread(String name) {
		super(name);
		mPriority &#61; Process.THREAD_PRIORITY_DEFAULT;
	}
</code></pre> 
<h4><a id="2_Runlooper_118"></a>2. Run方法中初始化looper</h4> 
<pre><code>	* 并调用	onLooperPrepared&#xff08;looper执行前执行&#xff0c;用于处理一些准备工作&#xff09;
	*  Looper.myLooper();绑定当前线程Looper轮询器
	*  Looper.loop(); 开始轮询消息

	 protected void onLooperPrepared() {
    }

    &#64;Override
    public void run() {
        mTid &#61; Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper &#61; Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid &#61; -1;
    }
</code></pre> 
<h4><a id="3_getLooper_141"></a>3. getLooper获取当前线程的轮询器</h4> 
<pre><code>	public Looper getLooper() {
	    if (!isAlive()) {
	        return null;
	    }
	    
	    // If the thread has been started, wait until the looper has been created.
	    synchronized (this) {
	        while (isAlive() &amp;&amp; mLooper &#61;&#61; null) {
	            try {
	                wait();
	            } catch (InterruptedException e) {
	            }
	        }
	    }
	    return mLooper;
	}
</code></pre> 
<h4><a id="4_HandlerLooper_quitMessageQueue_161"></a>4. 退出轮询&#xff0c;还记得Handler源码剖析里说的&#xff0c;Looper quit时会清理MessageQueue里面所有的消息</h4> 
<pre><code>	public boolean quit() {
		Looper looper &#61; getLooper();
		if (looper !&#61; null) {
		    looper.quit();
		    return true;
		}
		return false;
	}

	public boolean quitSafely() {
		Looper looper &#61; getLooper();
		if (looper !&#61; null) {
		looper.quitSafely();
		return true;
		}
		return false;
	}
</code></pre>
                </div>
                <link href="https://csdnimg.cn/release/blogv2/dist/mdeditor/css/editerView/markdown_views-d7a94ec6ab.css" rel="stylesheet">
                <link href="https://csdnimg.cn/release/blogv2/dist/mdeditor/css/style-49037e4d27.css" rel="stylesheet">
        </div>
    </article>