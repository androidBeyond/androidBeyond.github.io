---
layout:     post
title:      Android基础探究之三探Thread
subtitle:   对android中的ThreadLocal实现原理进行深入的学习
date:       2019-01-21
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android
    - 组件学习
--- 

<p><strong>一、概念</strong></p> 
<p>ThreadLocal是一个线程内部的数据存储类&#xff0c;通过它可以在指定的线程中存储数据&#xff0c;数据存储以后&#xff0c;只有在指定的线程中才可以访问&#xff0c;其他线程则无法获取。</p> 
<p><strong>二、使用场景</strong></p> 
<p>1、当某些数据是以线程为作用域且不同线程之间具有不同数据副本的时候&#xff0c;就可以考虑使用ThreadLocal&#xff0c;例如Android中的Handler消息机制&#xff0c;Looper的作用域就是线程且不同线程之间具有不同的Looper&#xff0c;这里就使用的是ThreadLocal对Looper与线程进行关联&#xff0c;如果不使用ThreadLocal&#xff0c;那么系统就必须提供一个全局的哈希表供Handler <br /> 查找指定的线程的Looper&#xff0c;这肯定比ThreadLocal复杂多了。</p> 
<p>2、当复杂逻辑下进行对象传递时&#xff0c;也可以考虑使用ThreadLocal&#xff0c;例如一个线程中执行的任务比较复杂&#xff0c;我们需要一个监听器去贯穿整个任务的执行过程&#xff0c;如果不使用ThreadLocal&#xff0c;那么我们就需要将一个监听器从函数调用栈层层传递&#xff0c;这对程序设计来说是不可接受的&#xff0c;而使用ThreadLocal我们就可以在需要的时候get获取&#xff0c;方便至极。</p> 
<p><strong>三、使用</strong></p> 
<p>来看一个简单的例子</p> 
<pre class="prettyprint"><code class=" hljs java"><span class="hljs-keyword">public</span> <span class="hljs-class"><span class="hljs-keyword">class</span> <span class="hljs-title">MainActivity</span> <span class="hljs-keyword">extends</span> <span class="hljs-title">Activity</span> {<!-- --></span>

    <span class="hljs-keyword">private</span> ThreadLocal mThreadLocal &#61; <span class="hljs-keyword">new</span> ThreadLocal();

    <span class="hljs-annotation">&#64;Override</span>
    <span class="hljs-keyword">protected</span> <span class="hljs-keyword">void</span> <span class="hljs-title">onCreate</span>(&#64;Nullable Bundle savedInstanceState) {
        <span class="hljs-keyword">super</span>.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        <span class="hljs-comment">//主线程</span>
        mThreadLocal.set(<span class="hljs-keyword">true</span>);

        <span class="hljs-keyword">new</span> Thread() {
            <span class="hljs-annotation">&#64;Override</span>
            <span class="hljs-keyword">public</span> <span class="hljs-keyword">void</span> <span class="hljs-title">run</span>() {
                <span class="hljs-keyword">super</span>.run();
                <span class="hljs-comment">//子线程</span>
                mThreadLocal.set(<span class="hljs-keyword">false</span>);
                Log.i(<span class="hljs-string">&#34;ThreadLocal子线程&#34;</span>,String.valueOf(mThreadLocal.get()));
            }
        }.start();

        <span class="hljs-comment">//保证子线程先去更新值</span>
        <span class="hljs-keyword">try</span> {
            getMainLooper().getThread().sleep(<span class="hljs-number">3000</span>);
            Log.i(<span class="hljs-string">&#34;ThreadLocal主线程&#34;</span>,String.valueOf(mThreadLocal.get()));
        } <span class="hljs-keyword">catch</span> (InterruptedException e) {
            e.printStackTrace();
        }

    }
}</code></pre> 
<p>执行结果&#xff1a;</p> 
<pre class="prettyprint"><code class=" hljs avrasm"><span class="hljs-number">07</span>-<span class="hljs-number">05</span> <span class="hljs-number">10</span>:<span class="hljs-number">57</span>:<span class="hljs-number">02.995</span> <span class="hljs-number">15966</span>-<span class="hljs-number">15992</span>/android_p<span class="hljs-preprocessor">.lbjfan</span><span class="hljs-preprocessor">.cm</span><span class="hljs-preprocessor">.memoryanalysedemo</span> I/ThreadLocal子线程: false
<span class="hljs-number">07</span>-<span class="hljs-number">05</span> <span class="hljs-number">10</span>:<span class="hljs-number">57</span>:<span class="hljs-number">05.995</span> <span class="hljs-number">15966</span>-<span class="hljs-number">15966</span>/android_p<span class="hljs-preprocessor">.lbjfan</span><span class="hljs-preprocessor">.cm</span><span class="hljs-preprocessor">.memoryanalysedemo</span> I/ThreadLocal主线程: true</code></pre> 
<p>结论&#xff1a;ThreadLocal在当前线程操作数据只对当前线程有效</p> 
<p><strong>三、源码分析</strong></p> 
<p>1、构造方法</p> 
<pre class="prettyprint"><code class=" hljs cs"><span class="hljs-keyword">public</span> <span class="hljs-title">ThreadLocal</span>() {}</code></pre> 
<p>2、set方法</p> 
<pre class="prettyprint"><code class=" hljs cs"><span class="hljs-keyword">public</span> <span class="hljs-keyword">void</span> <span class="hljs-title">set</span>(T <span class="hljs-keyword">value</span>) {
    <span class="hljs-comment">//获取当前线程</span>
    Thread currentThread &#61; Thread.currentThread();
    <span class="hljs-comment">//获取当前线程的Values对象</span>
    Values values &#61; values(currentThread);
    <span class="hljs-comment">//如果是空就新建</span>
    <span class="hljs-keyword">if</span> (values &#61;&#61; <span class="hljs-keyword">null</span>) {
        values &#61; initializeValues(currentThread);
    }
    <span class="hljs-comment">//使用values的put方法存值</span>
    values.put(<span class="hljs-keyword">this</span>, <span class="hljs-keyword">value</span>);
}</code></pre> 
<p>3、get方法</p> 
<pre class="prettyprint"><code class=" hljs axapta"><span class="hljs-keyword">public</span> T get() {
    <span class="hljs-comment">//获取当前线程</span>
    Thread currentThread &#61; Thread.currentThread();
    <span class="hljs-comment">//获取对应的values对象</span>
    Values values &#61; values(currentThread);
    <span class="hljs-keyword">if</span> (values !&#61; <span class="hljs-keyword">null</span>) {
        Object[] table &#61; values.table;
        <span class="hljs-comment">//获取索引</span>
        <span class="hljs-keyword">int</span> <span class="hljs-keyword">index</span> &#61; hash &amp; values.mask;
        <span class="hljs-comment">//获取值</span>
        <span class="hljs-keyword">if</span> (<span class="hljs-keyword">this</span>.reference &#61;&#61; table[<span class="hljs-keyword">index</span>]) {
            <span class="hljs-keyword">return</span> (T) table[<span class="hljs-keyword">index</span> &#43; <span class="hljs-number">1</span>];
        }
    } <span class="hljs-keyword">else</span> {
        <span class="hljs-comment">//初始化Values对象</span>
        values &#61; initializeValues(currentThread);
    }
    <span class="hljs-comment">//通过values的getAfterMiss方法获取值</span>
    <span class="hljs-keyword">return</span> (T) values.getAfterMiss(<span class="hljs-keyword">this</span>);
}</code></pre> 
<p>4、Values基础定义 <br /> Values是ThreadLocal的静态内部类&#xff0c;首先来看一些基础的定义</p> 
<pre class="prettyprint"><code class=" hljs java"><span class="hljs-keyword">static</span> class Values {

    <span class="hljs-comment">//数组初始值大小&#xff0c;必须是2的N次方</span>
    <span class="hljs-keyword">private</span> <span class="hljs-keyword">static</span> <span class="hljs-keyword">final</span> <span class="hljs-keyword">int</span> INITIAL_SIZE &#61; <span class="hljs-number">16</span>;

    <span class="hljs-comment">//被删除的数据</span>
    <span class="hljs-keyword">private</span> <span class="hljs-keyword">static</span> <span class="hljs-keyword">final</span> Object TOMBSTONE &#61; <span class="hljs-keyword">new</span> Object();

    <span class="hljs-comment">//存放数据的数组&#xff0c;使用key/value映射大小总是2的N次方</span>
    <span class="hljs-keyword">private</span> Object[] table;

    <span class="hljs-comment">//和key的hash值进行与运算&#xff0c;获取数组中的索引</span>
    <span class="hljs-keyword">private</span> <span class="hljs-keyword">int</span> mask;

    <span class="hljs-comment">//当前有效的Key的数量</span>
    <span class="hljs-keyword">private</span> <span class="hljs-keyword">int</span> size;

    <span class="hljs-comment">//已经失效的key的数量</span>
    <span class="hljs-keyword">private</span> <span class="hljs-keyword">int</span> tombstones;

    <span class="hljs-comment">//key的总和</span>
    <span class="hljs-keyword">private</span> <span class="hljs-keyword">int</span> maximumLoad;

    <span class="hljs-comment">//下一次查找失效key的起始位置</span>
    <span class="hljs-keyword">private</span> <span class="hljs-keyword">int</span> clean;</code></pre> 
<pre><code>由上面一些基础的定义我们知道&#xff0c;Values作为ThreadLocal的静态内部类&#xff0c;是真正意义上对数据进行存储、更新、及删除的类&#xff0c;内部使用数组对数据进行存储&#xff0c;存储结构为&#xff1a;key&#xff0c;value&#xff0c;key&#xff0c;value......
</code></pre> 
<p>5、Values的构造函数</p> 
<pre class="prettyprint"><code class=" hljs cs"> <span class="hljs-comment">//普通构造函数</span>
    Values() {
        initializeTable(INITIAL_SIZE);
        <span class="hljs-keyword">this</span>.size &#61; <span class="hljs-number">0</span>;
        <span class="hljs-keyword">this</span>.tombstones &#61; <span class="hljs-number">0</span>;
    }
    <span class="hljs-comment">//使用外部Values拷贝的构造函数</span>
    Values(Values fromParent) {
        <span class="hljs-keyword">this</span>.table &#61; fromParent.table.clone();
        <span class="hljs-keyword">this</span>.mask &#61; fromParent.mask;
        <span class="hljs-keyword">this</span>.size &#61; fromParent.size;
        <span class="hljs-keyword">this</span>.tombstones &#61; fromParent.tombstones;
        <span class="hljs-keyword">this</span>.maximumLoad &#61; fromParent.maximumLoad;
        <span class="hljs-keyword">this</span>.clean &#61; fromParent.clean;
        inheritValues(fromParent);
    }

    <span class="hljs-comment">//初始化数组大小</span>
    <span class="hljs-keyword">private</span> <span class="hljs-keyword">void</span> <span class="hljs-title">initializeTable</span>(<span class="hljs-keyword">int</span> capacity) {
        <span class="hljs-comment">//存储数据的table数组大小默认为32</span>
        <span class="hljs-keyword">this</span>.table &#61; <span class="hljs-keyword">new</span> Object[capacity * <span class="hljs-number">2</span>];
        <span class="hljs-comment">//mask的默认大小为table的长度减1&#xff0c;即table数组中最后一个元素的索引</span>
        <span class="hljs-keyword">this</span>.mask &#61; table.length - <span class="hljs-number">1</span>;
        <span class="hljs-keyword">this</span>.clean &#61; <span class="hljs-number">0</span>;
        <span class="hljs-comment">//默认存储的最大值为数组长度的1/3</span>
        <span class="hljs-keyword">this</span>.maximumLoad &#61; capacity * <span class="hljs-number">2</span> / <span class="hljs-number">3</span>; <span class="hljs-comment">// 2/3</span>
    }</code></pre> 
<p>6、Values的put方法</p> 
<pre class="prettyprint"><code class=" hljs axapta"> <span class="hljs-keyword">void</span> put(ThreadLocal&lt;?&gt; key, Object value)
        <span class="hljs-comment">//这个后面在看</span>
        cleanUp();
        <span class="hljs-comment">//标记第一个失效数据的索引</span>
        <span class="hljs-keyword">int</span> firstTombstone &#61; -<span class="hljs-number">1</span>;
        <span class="hljs-comment">//使用key的hash和mask进行&amp;运算&#xff0c;获取当前key在数组中的index</span>
        <span class="hljs-keyword">for</span> (<span class="hljs-keyword">int</span> <span class="hljs-keyword">index</span> &#61; key.hash &amp; mask;; <span class="hljs-keyword">index</span> &#61; next(<span class="hljs-keyword">index</span>)) {
            <span class="hljs-comment">//直接获取key</span>
            Object k &#61; table[<span class="hljs-keyword">index</span>];
            <span class="hljs-comment">//如果key相同&#xff0c;则直接更新value&#xff0c;key对应的索引为index&#xff0c;则value的索引为index&#43;1</span>
            <span class="hljs-keyword">if</span> (k &#61;&#61; key.reference) {
                <span class="hljs-comment">// Replace existing entry.</span>
                table[<span class="hljs-keyword">index</span> &#43; <span class="hljs-number">1</span>] &#61; value;
                <span class="hljs-keyword">return</span>;
            }
            <span class="hljs-comment">//如果key不存在</span>
            <span class="hljs-keyword">if</span> (k &#61;&#61; <span class="hljs-keyword">null</span>) {
                <span class="hljs-comment">//当前不存在失效key</span>
                <span class="hljs-keyword">if</span> (firstTombstone &#61;&#61; -<span class="hljs-number">1</span>) {
                    <span class="hljs-comment">//将key和value保存到数组</span>
                    table[<span class="hljs-keyword">index</span>] &#61; key.reference;
                    table[<span class="hljs-keyword">index</span> &#43; <span class="hljs-number">1</span>] &#61; value;
                    size&#43;&#43;;
                    <span class="hljs-keyword">return</span>;
                }

                <span class="hljs-comment">//如果存在失效的key&#xff0c;则将需要存储的值保存到失效的key所在的位置</span>
                table[firstTombstone] &#61; key.reference;
                table[firstTombstone &#43; <span class="hljs-number">1</span>] &#61; value;
                tombstones--;
                size&#43;&#43;;
                <span class="hljs-keyword">return</span>;
            }

            <span class="hljs-comment">//对失效的key进行标记</span>
            <span class="hljs-keyword">if</span> (firstTombstone &#61;&#61; -<span class="hljs-number">1</span> &amp;&amp; k &#61;&#61; TOMBSTONE) {
                firstTombstone &#61; <span class="hljs-keyword">index</span>;
            }
        }
    }</code></pre> 
<pre><code>可以看到&#xff0c;put方法其实很简单&#xff0c;使用key的hash和table数组的长度减1进行&amp;运算&#xff0c;获取index&#xff0c;然后有则更新&#xff0c;无则添加&#xff0c;添加时则利用key的有效性及失效的key
尽可能的节省空间。
</code></pre> 
<p>7、clearUp方法&#xff1a;将失效的key进行标记&#xff0c;释放它的值</p> 
<pre class="prettyprint"><code class=" hljs axapta"><span class="hljs-keyword">private</span> <span class="hljs-keyword">void</span> cleanUp() {
        <span class="hljs-comment">//如果需要扩容&#xff0c;则直接返回&#xff0c;扩容的过程中对失效的地方进行了标记</span>
        <span class="hljs-keyword">if</span> (rehash()) {
            <span class="hljs-keyword">return</span>;
        }
        <span class="hljs-comment">//没有值的话&#xff0c;什么也不做</span>
        <span class="hljs-keyword">if</span> (size &#61;&#61; <span class="hljs-number">0</span>) {
            <span class="hljs-keyword">return</span>;
        }
        <span class="hljs-comment">//默认是0&#xff0c;标记每次clean的位置</span>
        <span class="hljs-keyword">int</span> <span class="hljs-keyword">index</span> &#61; clean;
        Object[] table &#61; <span class="hljs-keyword">this</span>.table;
        <span class="hljs-keyword">for</span> (<span class="hljs-keyword">int</span> counter &#61; table.length; counter &gt; <span class="hljs-number">0</span>; counter &gt;&gt;&#61; <span class="hljs-number">1</span>,
                <span class="hljs-keyword">index</span> &#61; next(<span class="hljs-keyword">index</span>)) {
            Object k &#61; table[<span class="hljs-keyword">index</span>];
            <span class="hljs-comment">//如果key是失效的或者为null&#xff0c;则不做处理</span>
            <span class="hljs-keyword">if</span> (k &#61;&#61; TOMBSTONE || k &#61;&#61; <span class="hljs-keyword">null</span>) {
                <span class="hljs-keyword">continue</span>; <span class="hljs-comment">// on to next entry</span>
            }
            <span class="hljs-comment">//table只能存储null、tombstone、和references</span>
            &#64;SuppressWarnings(<span class="hljs-string">&#34;unchecked&#34;</span>)
            Reference&lt;ThreadLocal&lt;?&gt;&gt; reference
                    &#61; (Reference&lt;ThreadLocal&lt;?&gt;&gt;) k;
            <span class="hljs-comment">//检查key是否失效&#xff0c;失效的话进行标记并且释放它的值</span>
            <span class="hljs-keyword">if</span> (reference.get() &#61;&#61; <span class="hljs-keyword">null</span>) {
                <span class="hljs-comment">// This thread local was reclaimed by the garbage collector.</span>
                table[<span class="hljs-keyword">index</span>] &#61; TOMBSTONE;
                table[<span class="hljs-keyword">index</span> &#43; <span class="hljs-number">1</span>] &#61; <span class="hljs-keyword">null</span>;
                tombstones&#43;&#43;;
                size--;
            }
        }

        <span class="hljs-comment">//标记下次开始clean的位置</span>
        clean &#61; <span class="hljs-keyword">index</span>;
}</code></pre> 
<p>8、rehash方法&#xff1a;对数组进行扩容</p> 
<pre class="prettyprint"><code class=" hljs axapta">  <span class="hljs-keyword">private</span> <span class="hljs-keyword">boolean</span> rehash() {
        <span class="hljs-comment">//不需要扩容</span>
        <span class="hljs-keyword">if</span> (tombstones &#43; size &lt; maximumLoad) {
            <span class="hljs-keyword">return</span> <span class="hljs-keyword">false</span>;
        }

        <span class="hljs-keyword">int</span> capacity &#61; table.length &gt;&gt; <span class="hljs-number">1</span>;

        <span class="hljs-keyword">int</span> newCapacity &#61; capacity;

        <span class="hljs-keyword">if</span> (size &gt; (capacity &gt;&gt; <span class="hljs-number">1</span>)) {
            <span class="hljs-comment">//双倍扩容</span>
            newCapacity &#61; capacity * <span class="hljs-number">2</span>;
        }

        <span class="hljs-comment">//标记数组</span>
        Object[] oldTable &#61; <span class="hljs-keyword">this</span>.table;

        <span class="hljs-comment">//重新初始化数组大小</span>
        initializeTable(newCapacity);

        <span class="hljs-comment">// 重置失效的key</span>
        <span class="hljs-keyword">this</span>.tombstones &#61; <span class="hljs-number">0</span>;

        <span class="hljs-comment">//没有有效的key</span>
        <span class="hljs-keyword">if</span> (size &#61;&#61; <span class="hljs-number">0</span>) {
            <span class="hljs-keyword">return</span> <span class="hljs-keyword">true</span>;
        }

        <span class="hljs-comment">//数组扩容</span>
        <span class="hljs-keyword">for</span> (<span class="hljs-keyword">int</span> i &#61; oldTable.length - <span class="hljs-number">2</span>; i &gt;&#61; <span class="hljs-number">0</span>; i -&#61; <span class="hljs-number">2</span>) {
            Object k &#61; oldTable[i];
            <span class="hljs-comment">//丢弃失效的key</span>
            <span class="hljs-keyword">if</span> (k &#61;&#61; <span class="hljs-keyword">null</span> || k &#61;&#61; TOMBSTONE) {
                <span class="hljs-keyword">continue</span>;
            }
            &#64;SuppressWarnings(<span class="hljs-string">&#34;unchecked&#34;</span>)
            Reference&lt;ThreadLocal&lt;?&gt;&gt; reference
                    &#61; (Reference&lt;ThreadLocal&lt;?&gt;&gt;) k;
            ThreadLocal&lt;?&gt; key &#61; reference.get();
            <span class="hljs-keyword">if</span> (key !&#61; <span class="hljs-keyword">null</span>) {
                添加有效的key和value
                add(key, oldTable[i &#43; <span class="hljs-number">1</span>]);
            } <span class="hljs-keyword">else</span> {
                <span class="hljs-comment">// The key was reclaimed.</span>
                size--;
            }
        }

        <span class="hljs-keyword">return</span> <span class="hljs-keyword">true</span>;
    }
   add方法&#xff1a;table数组内部是如何进行数据存储的

    <span class="hljs-keyword">void</span> add(ThreadLocal&lt;?&gt; key, Object value) {
        <span class="hljs-comment">//根据key的hash和mask进行&amp;运算&#xff0c;获取index</span>
        <span class="hljs-keyword">for</span> (<span class="hljs-keyword">int</span> <span class="hljs-keyword">index</span> &#61; key.hash &amp; mask;; <span class="hljs-keyword">index</span> &#61; next(<span class="hljs-keyword">index</span>)) {
            Object k &#61; table[<span class="hljs-keyword">index</span>];
            <span class="hljs-keyword">if</span> (k &#61;&#61; <span class="hljs-keyword">null</span>) {
                <span class="hljs-comment">//将key存储到数组的index位置</span>
                table[<span class="hljs-keyword">index</span>] &#61; key.reference;
                <span class="hljs-comment">//将value存储到数组的index&#43;1位置</span>
                table[<span class="hljs-keyword">index</span> &#43; <span class="hljs-number">1</span>] &#61; value;
                <span class="hljs-keyword">return</span>;
            }
        }
    }</code></pre> 
<p>小结&#xff1a;ThreadLocal的set方法内部调用其静态内部类Values的put方法&#xff0c;内部采用table数组对数据进行存储&#xff0c;使用table[index]&#61;key,table[index&#43;1]&#61;value的方式&#xff0c;在存储的过程中会进行数组扩容、失效key的标记&#xff0c;值的释放等操作。</p> 
<p>9、getAfterMiss方法&#xff1a;之前没有set过值&#xff0c;调用get方法时会调用到此方法</p> 
<pre class="prettyprint"><code class=" hljs axapta"> Object getAfterMiss(ThreadLocal&lt;?&gt; key) {
        Object[] table &#61; <span class="hljs-keyword">this</span>.table;
        <span class="hljs-comment">//获取索引</span>
        <span class="hljs-keyword">int</span> <span class="hljs-keyword">index</span> &#61; key.hash &amp; mask;
        <span class="hljs-comment">//如果key不存在&#xff0c;那么直接返回ThreadLocal方法返回值&#xff0c;默认为null</span>
        <span class="hljs-keyword">if</span> (table[<span class="hljs-keyword">index</span>] &#61;&#61; <span class="hljs-keyword">null</span>) {
            Object value &#61; key.initialValue();

            <span class="hljs-comment">//如果是同一个数组且key不存在&#xff0c;&#xff08;疑问为什么回不相等&#xff0c;方法开始赋的值&#xff09;</span>
            <span class="hljs-keyword">if</span> (<span class="hljs-keyword">this</span>.table &#61;&#61; table &amp;&amp; table[<span class="hljs-keyword">index</span>] &#61;&#61; <span class="hljs-keyword">null</span>) {
                table[<span class="hljs-keyword">index</span>] &#61; key.reference;
                table[<span class="hljs-keyword">index</span> &#43; <span class="hljs-number">1</span>] &#61; value;
                size&#43;&#43;;

                cleanUp();
                <span class="hljs-keyword">return</span> value;
            }

            <span class="hljs-comment">// 添加到table数组</span>
            put(key, value);
            <span class="hljs-keyword">return</span> value;
        }

        <span class="hljs-comment">//key存在时的处理</span>
        <span class="hljs-keyword">int</span> firstTombstone &#61; -<span class="hljs-number">1</span>;

        <span class="hljs-comment">// Continue search.</span>
        <span class="hljs-keyword">for</span> (<span class="hljs-keyword">index</span> &#61; next(<span class="hljs-keyword">index</span>);; <span class="hljs-keyword">index</span> &#61; next(<span class="hljs-keyword">index</span>)) {
            Object reference &#61; table[<span class="hljs-keyword">index</span>];
            <span class="hljs-comment">//根据index查到且相等直接返回</span>
            <span class="hljs-keyword">if</span> (reference &#61;&#61; key.reference) {
                <span class="hljs-keyword">return</span> table[<span class="hljs-keyword">index</span> &#43; <span class="hljs-number">1</span>];
            }

            <span class="hljs-comment">//如果没有查到&#xff0c;继续返回默认的value</span>
            <span class="hljs-keyword">if</span> (reference &#61;&#61; <span class="hljs-keyword">null</span>) {
                Object value &#61; key.initialValue();

                <span class="hljs-comment">// If the table is still the same...</span>
                <span class="hljs-keyword">if</span> (<span class="hljs-keyword">this</span>.table &#61;&#61; table) {
                    <span class="hljs-comment">// If we passed a tombstone and that slot still</span>
                    <span class="hljs-comment">// contains a tombstone...</span>
                    <span class="hljs-keyword">if</span> (firstTombstone &gt; -<span class="hljs-number">1</span>
                            &amp;&amp; table[firstTombstone] &#61;&#61; TOMBSTONE) {
                        table[firstTombstone] &#61; key.reference;
                        table[firstTombstone &#43; <span class="hljs-number">1</span>] &#61; value;
                        tombstones--;
                        size&#43;&#43;;

                        <span class="hljs-comment">// No need to clean up here. We aren&#39;t filling</span>
                        <span class="hljs-comment">// in a null slot.</span>
                        <span class="hljs-keyword">return</span> value;
                    }

                    <span class="hljs-comment">// If this slot is still empty...</span>
                    <span class="hljs-keyword">if</span> (table[<span class="hljs-keyword">index</span>] &#61;&#61; <span class="hljs-keyword">null</span>) {
                        table[<span class="hljs-keyword">index</span>] &#61; key.reference;
                        table[<span class="hljs-keyword">index</span> &#43; <span class="hljs-number">1</span>] &#61; value;
                        size&#43;&#43;;

                        cleanUp();
                        <span class="hljs-keyword">return</span> value;
                    }
                }

                <span class="hljs-comment">// The table changed during initialValue().</span>
                put(key, value);
                <span class="hljs-keyword">return</span> value;
            }
            <span class="hljs-comment">//标记无效的key</span>
            <span class="hljs-keyword">if</span> (firstTombstone &#61;&#61; -<span class="hljs-number">1</span> &amp;&amp; reference &#61;&#61; TOMBSTONE) {
                <span class="hljs-comment">// Keep track of this tombstone so we can overwrite it.</span>
                firstTombstone &#61; <span class="hljs-keyword">index</span>;
            }
        }
    }</code></pre> 
<p>经过上面的分析&#xff0c;ThreadLocal之所以能够在不同的线程中存储数据副本&#xff0c;是因为每个Thread都有一个Values对象&#xff0c;该对象中的table数组进行真正的存储。当我们使用ThreadLocal <br /> 的get或者set方法时&#xff0c;会更据当前线程获取到该线程内部的Values对象&#xff0c;然后获取内部的values数组&#xff0c;最后在进行数据的存储或删除。</p> 
<p><strong>四、Android Handler消息机制中的ThreadLocal</strong> <br /> 我们都知道&#xff0c;当我们在一个线程中使用Looper的时候&#xff0c;需要调用Looper的prepare方法和loop方法&#xff0c;不然就会出现异常。那么&#xff0c;我们就看看这两个方法的源码&#xff1a;</p> 
<pre class="prettyprint"><code class=" hljs cs"><span class="hljs-keyword">public</span> <span class="hljs-keyword">static</span> <span class="hljs-keyword">void</span> <span class="hljs-title">prepare</span>() {
    prepare(<span class="hljs-keyword">true</span>);
}

<span class="hljs-keyword">private</span> <span class="hljs-keyword">static</span> <span class="hljs-keyword">void</span> <span class="hljs-title">prepare</span>(boolean quitAllowed) {
    <span class="hljs-keyword">if</span> (sThreadLocal.<span class="hljs-keyword">get</span>() !&#61; <span class="hljs-keyword">null</span>) {
        <span class="hljs-keyword">throw</span> <span class="hljs-keyword">new</span> RuntimeException(<span class="hljs-string">&#34;Only one Looper may be created per thread&#34;</span>);
    }
    sThreadLocal.<span class="hljs-keyword">set</span>(<span class="hljs-keyword">new</span> Looper(quitAllowed));
}
loop方法内部先调用了myLooper方法&#xff1a;
<span class="hljs-keyword">public</span> <span class="hljs-keyword">static</span> &#64;Nullable Looper <span class="hljs-title">myLooper</span>() {
    <span class="hljs-keyword">return</span> sThreadLocal.<span class="hljs-keyword">get</span>();
}</code></pre> 
<p>很明显&#xff0c;Looper 的prepare方法先创建Looper&#xff0c;并使用ThreadLocal存储即与当前的线程进行关联&#xff0c;然后loop方法开启消息机制的时候&#xff0c;使用ThreadLocal方法获取到当前线程的Looper <br /> 方法。</p> 
<p>结语&#xff1a;ThreadLocal是一种针对线程间数据副本不同的巧妙设计&#xff0c;开发者无需理会内部的复杂实现&#xff0c;只需在调用的时候使用get和set方法即可&#xff01;对外屏蔽了细节&#xff0c;是一种设计思想的体现&#xff0c;其内在table数组内存的利用和空间的扩展也值得我们学习。</p>
                </div>
                <link href="https://csdnimg.cn/release/blogv2/dist/mdeditor/css/editerView/markdown_views-d7a94ec6ab.css" rel="stylesheet">
                <link href="https://csdnimg.cn/release/blogv2/dist/mdeditor/css/style-49037e4d27.css" rel="stylesheet">
        </div>
    </article>