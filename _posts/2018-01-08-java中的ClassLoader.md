---
layout:     post
title:      java中的ClassLoader
subtitle:   java中的ClassLoader学习
date:       2018-01-08
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - android
    - framework
    - java
    - classloader
---

<h3><a id="_16"></a><strong>前言</strong></h3> 
<p>热修复和插件化是目前比较热门的技术&#xff0c;要想更好的掌握它们需要了解ClassLoader&#xff0c;因此也就有了本系列的产生&#xff0c;这一篇我们先来学习Java中的ClassLoader。</p> 
 
<h3><a id="1ClassLoader_19"></a><strong>1.ClassLoader的类型</strong></h3> 
<p>在<a href="http://liuwangshu.cn/java/jvm/1-runtime-data-area.html">Java虚拟机&#xff08;一&#xff09;结构原理与运行时数据区域</a>这篇文章中&#xff0c;我提到过类加载子系统&#xff0c;它的主要作用就是通过多种类加载器&#xff08;ClassLoader&#xff09;来查找和加载Class文件到 Java 虚拟机中。<br /> Java中的类加载器主要有两种类型&#xff0c;系统类加载和自定义类加载器。其中系统类加载器包括3种&#xff0c;分别是Bootstrap ClassLoader、 Extensions ClassLoader和 App ClassLoader。</p> 
<h4><a id="11_Bootstrap_ClassLoader_23"></a><strong>1.1 Bootstrap ClassLoader</strong></h4> 
<p>用C/C&#43;&#43;代码实现的加载器&#xff0c;用于加载Java虚拟机运行时所需要的系统类&#xff0c;如<code>java.lang.*、java.uti.*</code>等这些系统类&#xff0c;它们默认在$JAVA_HOME/jre/lib目录中&#xff0c;也可以通过启动Java虚拟机时指定-Xbootclasspath选项&#xff0c;来改变Bootstrap ClassLoader的加载目录。<br /> Java虚拟机的启动就是通过 Bootstrap ClassLoader创建一个初始类来完成的。由于Bootstrap ClassLoader是使用C/C&#43;&#43;语言实现的&#xff0c; 所以该加载器不能被Java代码访问到。需要注意的是Bootstrap ClassLoader并不继承java.lang.ClassLoader。<br /> 我们可以通过如下代码来得出Bootstrap ClassLoader所加载的目录&#xff1a;</p> 
<pre><code class="prism language-java"><span class="token keyword">public</span> <span class="token keyword">class</span> <span class="token class-name">ClassLoaderTest</span> <span class="token punctuation">{<!-- --></span>
    <span class="token keyword">public</span> <span class="token keyword">static</span> <span class="token keyword">void</span> <span class="token function">main</span><span class="token punctuation">(</span>String<span class="token punctuation">[</span><span class="token punctuation">]</span>args<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
        System<span class="token punctuation">.</span>out<span class="token punctuation">.</span><span class="token function">println</span><span class="token punctuation">(</span>System<span class="token punctuation">.</span><span class="token function">getProperty</span><span class="token punctuation">(</span><span class="token string">&#34;sun.boot.class.path&#34;</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre> 
<p>打印结果为&#xff1a;</p> 
<pre><code>C:\Program Files\Java\jdk1.8.0_102\jre\lib\resources.jar;
C:\Program Files\Java\jdk1.8.0_102\jre\lib\rt.jar;
C:\Program Files\Java\jdk1.8.0_102\jre\lib\sunrsasign.jar;
C:\Program Files\Java\jdk1.8.0_102\jre\lib\jsse.jar;
C:\Program Files\Java\jdk1.8.0_102\jre\lib\jce.jar;
C:\Program Files\Java\jdk1.8.0_102\jre\lib\charsets.jar;
C:\Program Files\Java\jdk1.8.0_102\jre\lib\jfr.jar;
C:\Program Files\Java\jdk1.8.0_102\jre\classes
</code></pre> 
<p>可以发现几乎都是$JAVA_HOME/jre/lib目录中的jar包&#xff0c;包括rt.jar、resources.jar和charsets.jar等等。</p> 
<h4><a id="12_Extensions_ClassLoader_46"></a><strong>1.2 Extensions ClassLoader</strong></h4> 
<p>用于加载 Java 的拓展类 &#xff0c;拓展类的jar包一般会放在$JAVA_HOME/jre/lib/ext目录下&#xff0c;用来提供除了系统类之外的额外功能。也可以通过-Djava.ext.dirs选项添加和修改Extensions ClassLoader加载的路径。<br /> 通过以下代码可以得到Extensions ClassLoader加载目录&#xff1a;</p> 
<pre><code class="prism language-java">System<span class="token punctuation">.</span>out<span class="token punctuation">.</span><span class="token function">println</span><span class="token punctuation">(</span>System<span class="token punctuation">.</span><span class="token function">getProperty</span><span class="token punctuation">(</span><span class="token string">&#34;java.ext.dirs&#34;</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
</code></pre> 
<p>打印结果为&#xff1a;</p> 
<pre><code>C:\Program Files\Java\jdk1.8.0_102\jre\lib\ext;
C:\Windows\Sun\Java\lib\ext
</code></pre> 
<h4><a id="13_App_ClassLoader_60"></a><strong>1.3 App ClassLoader</strong></h4> 
<p>负责加载当前应用程序Classpath目录下的所有jar和Class文件。也可以加载通过-Djava.class.path选项所指定的目录下的jar和Class文件。</p> 
<h4><a id="14_Custom_ClassLoader_63"></a><strong>1.4 Custom ClassLoader</strong></h4> 
<p>除了系统提供的类加载器&#xff0c;还可以自定义类加载器&#xff0c;自定义类加载器通过继承java.lang.ClassLoader类的方式来实现自己的类加载器&#xff0c;除了 Bootstrap ClassLoader&#xff0c;Extensions ClassLoader和App ClassLoader也继承了java.lang.ClassLoader类。关于自定义类加载器后面会进行介绍。</p> 
<h3><a id="2ClassLoader_68"></a><strong>2.ClassLoader的继承关系</strong></h3> 
<p>运行一个Java程序需要用到几种类型的类加载器呢&#xff1f;如下所示。</p> 
<pre><code class="prism language-java"><span class="token keyword">public</span> <span class="token keyword">class</span> <span class="token class-name">ClassLoaderTest</span> <span class="token punctuation">{<!-- --></span>
    <span class="token keyword">public</span> <span class="token keyword">static</span> <span class="token keyword">void</span> <span class="token function">main</span><span class="token punctuation">(</span>String<span class="token punctuation">[</span><span class="token punctuation">]</span> args<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
        ClassLoader loader <span class="token operator">&#61;</span> ClassLoaderTest<span class="token punctuation">.</span><span class="token keyword">class</span><span class="token punctuation">.</span><span class="token function">getClassLoader</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token keyword">while</span> <span class="token punctuation">(</span>loader <span class="token operator">!&#61;</span> null<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
            System<span class="token punctuation">.</span>out<span class="token punctuation">.</span><span class="token function">println</span><span class="token punctuation">(</span>loader<span class="token punctuation">)</span><span class="token punctuation">;</span><span class="token comment">//1</span>
            loader <span class="token operator">&#61;</span> loader<span class="token punctuation">.</span><span class="token function">getParent</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span>
    <span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre> 
<p>首先我们得到当前类ClassLoaderTest的类加载器&#xff0c;并在注释1处打印出来&#xff0c;接着打印出当前类的类加载器的父加载器&#xff0c;直到没有父加载器终止循环。打印结果如下所示。</p> 
<pre><code>sun.misc.Launcher$AppClassLoader&#64;75b84c92
sun.misc.Launcher$ExtClassLoader&#64;1b6d3586
</code></pre> 
<p>第1行说明加载ClassLoaderTest的类加载器是AppClassLoader&#xff0c;第2行说明AppClassLoader的父加载器为ExtClassLoader。至于为何没有打印出ExtClassLoader的父加载器Bootstrap ClassLoader&#xff0c;这是因为Bootstrap ClassLoader是由C/C&#43;&#43;编写的&#xff0c;并不是一个Java类&#xff0c;因此我们无法在Java代码中获取它的引用。</p> 
<p>我们知道系统所提供的类加载器有3种类型&#xff0c;但是系统提供的ClassLoader相关类却不只3个。另外&#xff0c;AppClassLoader的父类加载器为ExtClassLoader&#xff0c;并不代表AppClassLoader继承自ExtClassLoader&#xff0c;ClassLoader的继承关系如下所示。</p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly9zMi5heDF4LmNvbS8yMDE5LzA1LzI4L1ZtWmZYRC5wbmc" alt="VmZfXD.png" /></p> 
<p>可以看到上图中共有5个ClassLoader相关类&#xff0c;下面简单对它们进行介绍&#xff1a;</p> 
<ul><li><a href="http://www.grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8u40-b25/java/lang/ClassLoader.java#ClassLoader">ClassLoader</a>是一个抽象类&#xff0c;其中定义了ClassLoader的主要功能。</li><li>SecureClassLoader继承了抽象类ClassLoader&#xff0c;但SecureClassLoader并不是ClassLoader的实现类&#xff0c;而是拓展了ClassLoader类加入了权限方面的功能&#xff0c;加强了ClassLoader的安全性。</li><li>URLClassLoader继承自SecureClassLoader&#xff0c;用来通过URl路径从jar文件和文件夹中加载类和资源。</li><li>ExtClassLoader和AppClassLoader都继承自URLClassLoader&#xff0c;它们都是<a href="http://www.grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8u40-b25/sun/misc/Launcher.java#Launcher.AppClassLoader">Launcher </a>的内部类&#xff0c;Launcher 是Java虚拟机的入口应用&#xff0c;ExtClassLoader和AppClassLoader都是在Launcher中进行初始化的。</li></ul> 
<h3><a id="3__104"></a><strong>3 双亲委托模式</strong></h3> 
<h4><a id="31__105"></a><strong>3.1 双亲委托模式的特点</strong></h4> 
<p>类加载器查找Class所采用的是双亲委托模式&#xff0c;所谓双亲委托模式就是首先判断该Class是否已经加载&#xff0c;如果没有则不是自身去查找而是委托给父加载器进行查找&#xff0c;这样依次的进行递归&#xff0c;直到委托到最顶层的Bootstrap ClassLoader&#xff0c;如果Bootstrap ClassLoader找到了该Class&#xff0c;就会直接返回&#xff0c;如果没找到&#xff0c;则继续依次向下查找&#xff0c;如果还没找到则最后会交由自身去查找。<br /> 这样讲可能会有些抽象&#xff0c;来看下面的图。</p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly9zMi5heDF4LmNvbS8yMDE5LzA1LzI4L1ZtWjRuZS5wbmc" alt="VmZ4ne.png" /></p> 
<p>我们知道类加载子系统用来查找和加载Class文件到 Java 虚拟机中&#xff0c;假设我们要加载一个位于D盘的Class文件&#xff0c;这时系统所提供的类加载器不能满足条件&#xff0c;这时就需要我们自定义类加载器继承自java.lang.ClassLoader&#xff0c;并复写它的findClass方法。加载D盘的Class文件步骤如下&#xff1a;</p> 
<ol><li>自定义类加载器首先从缓存中要查找Class文件是否已经加载&#xff0c;如果已经加载就返回该Class&#xff0c;如果没加载则委托给父加载器也就是App ClassLoader。</li><li>按照上图中红色虚线的方向递归步骤1。</li><li>一直委托到Bootstrap ClassLoader&#xff0c;如果Bootstrap ClassLoader在缓存中还没有查找到Class文件&#xff0c;则在自己的规定路径$JAVA_HOME/jre/libr中或者-Xbootclasspath选项指定路径的jar包中进行查找&#xff0c;如果找到则返回该Class&#xff0c;如果没有则交给子加载器Extensions ClassLoader。</li><li>Extensions ClassLoader查找$JAVA_HOME/jre/lib/ext目录下或者-Djava.ext.dirs选项指定目录下的jar包&#xff0c;如果找到就返回&#xff0c;找不到则交给App ClassLoader。</li><li>App ClassLoade查找Classpath目录下或者-Djava.class.path选项所指定的目录下的jar包和Class文件&#xff0c;如果找到就返回&#xff0c;找不到交给我们自定义的类加载器&#xff0c;如果还找不到则抛出异常。</li></ol> 
<p>总的来说就是Class文件加载到类加载子系统后&#xff0c;先沿着图中红色虚线的方向自下而上进行委托&#xff0c;再沿着黑色虚线的方向自上而下进行查找&#xff0c;整个过程就是先上后下。</p> 
<p>类加载的步骤在JDK8的源码中也得到了体现&#xff0c;来查看抽象类的<a href="http://www.grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8u40-b25/java/lang/ClassLoader.java#ClassLoader">ClassLoader</a>方法&#xff0c;如下所示。</p> 
<pre><code class="prism language-java"> <span class="token keyword">protected</span> Class<span class="token operator">&lt;</span><span class="token operator">?</span><span class="token operator">&gt;</span> More <span class="token punctuation">.</span><span class="token punctuation">.</span><span class="token punctuation">.</span><span class="token function">loadClass</span><span class="token punctuation">(</span>String name<span class="token punctuation">,</span> <span class="token keyword">boolean</span> resolve<span class="token punctuation">)</span>
         <span class="token keyword">throws</span> ClassNotFoundException
     <span class="token punctuation">{<!-- --></span>
         <span class="token keyword">synchronized</span> <span class="token punctuation">(</span><span class="token function">getClassLoadingLock</span><span class="token punctuation">(</span>name<span class="token punctuation">)</span><span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
             Class<span class="token operator">&lt;</span><span class="token operator">?</span><span class="token operator">&gt;</span> c <span class="token operator">&#61;</span> <span class="token function">findLoadedClass</span><span class="token punctuation">(</span>name<span class="token punctuation">)</span><span class="token punctuation">;</span><span class="token comment">//1</span>
             <span class="token keyword">if</span> <span class="token punctuation">(</span>c <span class="token operator">&#61;&#61;</span> null<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                 <span class="token keyword">long</span> t0 <span class="token operator">&#61;</span> System<span class="token punctuation">.</span><span class="token function">nanoTime</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
                 <span class="token keyword">try</span> <span class="token punctuation">{<!-- --></span>
                     <span class="token keyword">if</span> <span class="token punctuation">(</span>parent <span class="token operator">!&#61;</span> null<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                         c <span class="token operator">&#61;</span> parent<span class="token punctuation">.</span><span class="token function">loadClass</span><span class="token punctuation">(</span>name<span class="token punctuation">,</span> <span class="token boolean">false</span><span class="token punctuation">)</span><span class="token punctuation">;</span><span class="token comment">//2</span>
                     <span class="token punctuation">}</span> <span class="token keyword">else</span> <span class="token punctuation">{<!-- --></span>
                         c <span class="token operator">&#61;</span> <span class="token function">findBootstrapClassOrNull</span><span class="token punctuation">(</span>name<span class="token punctuation">)</span><span class="token punctuation">;</span><span class="token comment">//3</span>
                     <span class="token punctuation">}</span>
                 <span class="token punctuation">}</span> <span class="token keyword">catch</span> <span class="token punctuation">(</span><span class="token class-name">ClassNotFoundException</span> e<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>            
                 <span class="token punctuation">}</span>
                 <span class="token keyword">if</span> <span class="token punctuation">(</span>c <span class="token operator">&#61;&#61;</span> null<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                     <span class="token comment">// If still not found, then invoke findClass in order</span>
                     <span class="token comment">// to find the class.</span>
                     <span class="token keyword">long</span> t1 <span class="token operator">&#61;</span> System<span class="token punctuation">.</span><span class="token function">nanoTime</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
                     c <span class="token operator">&#61;</span> <span class="token function">findClass</span><span class="token punctuation">(</span>name<span class="token punctuation">)</span><span class="token punctuation">;</span><span class="token comment">//4</span>
                     <span class="token comment">// this is the defining class loader; record the stats</span>
                     sun<span class="token punctuation">.</span>misc<span class="token punctuation">.</span>PerfCounter<span class="token punctuation">.</span><span class="token function">getParentDelegationTime</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">.</span><span class="token function">addTime</span><span class="token punctuation">(</span>t1 <span class="token operator">-</span> t0<span class="token punctuation">)</span><span class="token punctuation">;</span>
                     sun<span class="token punctuation">.</span>misc<span class="token punctuation">.</span>PerfCounter<span class="token punctuation">.</span><span class="token function">getFindClassTime</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">.</span><span class="token function">addElapsedTimeFrom</span><span class="token punctuation">(</span>t1<span class="token punctuation">)</span><span class="token punctuation">;</span>
                     sun<span class="token punctuation">.</span>misc<span class="token punctuation">.</span>PerfCounter<span class="token punctuation">.</span><span class="token function">getFindClasses</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">.</span><span class="token function">increment</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
                 <span class="token punctuation">}</span>
             <span class="token punctuation">}</span>
            <span class="token keyword">if</span> <span class="token punctuation">(</span>resolve<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                 <span class="token function">resolveClass</span><span class="token punctuation">(</span>c<span class="token punctuation">)</span><span class="token punctuation">;</span>
             <span class="token punctuation">}</span>
            <span class="token keyword">return</span> c<span class="token punctuation">;</span>
         <span class="token punctuation">}</span>
     <span class="token punctuation">}</span>
</code></pre> 
<p>注释1处用来检查类是否已经加载&#xff0c;如果已经加载则后面的代码不会执行&#xff0c;最后会返回该类。没有加载则会接着向下执行。<br /> 注释2处&#xff0c;如果父类加载器不为null&#xff0c;则调用父类加载器的loadClass方法。如果父类加载器为null则调用注释3处的findBootstrapClassOrNull方法&#xff0c;这个方法内部调用了Native方法findBootstrapClass&#xff0c;findBootstrapClass方法中最终会用Bootstrap Classloader来查找类。如果Bootstrap Classloader仍没有找到该类&#xff0c;也就说明向上委托没有找到该类&#xff0c;则调用注释4处的findClass方法继续向下进行查找。</p> 
<h4><a id="32__161"></a><strong>3.2 双亲委托模式的好处</strong></h4> 
<p>采取双亲委托模式主要有两点好处&#xff1a;</p> 
<ol><li>避免重复加载&#xff0c;如果已经加载过一次Class&#xff0c;就不需要再次加载&#xff0c;而是先从缓存中直接读取。</li><li>更加安全&#xff0c;如果不使用双亲委托模式&#xff0c;就可以自定义一个String类来替代系统的String类&#xff0c;这显然会造成安全隐患&#xff0c;采用双亲委托模式会使得系统的String类在Java虚拟机启动时就被加载&#xff0c;也就无法自定义String类来替代系统的String类&#xff0c;除非我们修改<br /> 类加载器搜索类的默认算法。还有一点&#xff0c;只有两个类名一致并且被同一个类加载器加载的类&#xff0c;Java虚拟机才会认为它们是同一个类&#xff0c;想要骗过Java虚拟机显然不会那么容易。</li></ol> 
<h3><a id="4ClassLoader_167"></a><strong>4.自定义ClassLoader</strong></h3> 
<p>系统提供的类加载器只能够加载指定目录下的jar包和Class文件&#xff0c;如果想要加载网络上的或者是D盘某一文件中的jar包和Class文件则需要自定义ClassLoader。<br /> 实现自定义ClassLoader需要两个步骤&#xff1a;</p> 
<ol><li>定义一个自定义ClassLoade并继承抽象类ClassLoader。</li><li>复写findClass方法&#xff0c;并在findClass方法中调用defineClass方法。</li></ol> 
<p>下面我们就自定义一个ClassLoader用来加载位于D:\lib的Class文件。</p> 
<h4><a id="41_Class_174"></a><strong>4.1 编写测试Class文件</strong></h4> 
<p>首先编写测试类并生成Class文件&#xff0c;如下所示。</p> 
<pre><code class="prism language-java"><span class="token keyword">package</span> com<span class="token punctuation">.</span>example<span class="token punctuation">;</span>
<span class="token keyword">public</span> <span class="token keyword">class</span> <span class="token class-name">Jobs</span> <span class="token punctuation">{<!-- --></span>
    <span class="token keyword">public</span> <span class="token keyword">void</span> <span class="token function">say</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
        System<span class="token punctuation">.</span>out<span class="token punctuation">.</span><span class="token function">println</span><span class="token punctuation">(</span><span class="token string">&#34;One more thing&#34;</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre> 
<p>将这个Jobs.java放入到D:\lib中&#xff0c;使用cmd命令进入D:\lib目录中&#xff0c;执行<code>Javac Jobs.java</code>对该java文件进行编译&#xff0c;这时会在D:\lib中生成Jobs.class。</p> 
<h4><a id="42_ClassLoader_187"></a><strong>4.2 编写自定义ClassLoader</strong></h4> 
<p>接下来编写自定义ClassLoader&#xff0c;如下所示。</p> 
<pre><code class="prism language-java"><span class="token keyword">import</span> java<span class="token punctuation">.</span>io<span class="token punctuation">.</span>ByteArrayOutputStream<span class="token punctuation">;</span>
<span class="token keyword">import</span> java<span class="token punctuation">.</span>io<span class="token punctuation">.</span>File<span class="token punctuation">;</span>
<span class="token keyword">import</span> java<span class="token punctuation">.</span>io<span class="token punctuation">.</span>FileInputStream<span class="token punctuation">;</span>
<span class="token keyword">import</span> java<span class="token punctuation">.</span>io<span class="token punctuation">.</span>IOException<span class="token punctuation">;</span>
<span class="token keyword">import</span> java<span class="token punctuation">.</span>io<span class="token punctuation">.</span>InputStream<span class="token punctuation">;</span>
<span class="token keyword">public</span> <span class="token keyword">class</span> <span class="token class-name">DiskClassLoader</span> <span class="token keyword">extends</span> <span class="token class-name">ClassLoader</span> <span class="token punctuation">{<!-- --></span>
    <span class="token keyword">private</span> String path<span class="token punctuation">;</span>
    <span class="token keyword">public</span> <span class="token function">DiskClassLoader</span><span class="token punctuation">(</span>String path<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
        <span class="token keyword">this</span><span class="token punctuation">.</span>path <span class="token operator">&#61;</span> path<span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
    <span class="token annotation punctuation">&#64;Override</span>
    <span class="token keyword">protected</span> Class<span class="token operator">&lt;</span><span class="token operator">?</span><span class="token operator">&gt;</span> <span class="token function">findClass</span><span class="token punctuation">(</span>String name<span class="token punctuation">)</span> <span class="token keyword">throws</span> ClassNotFoundException <span class="token punctuation">{<!-- --></span>
        Class <span class="token class-name">clazz</span> <span class="token operator">&#61;</span> null<span class="token punctuation">;</span>
        <span class="token keyword">byte</span><span class="token punctuation">[</span><span class="token punctuation">]</span> classData <span class="token operator">&#61;</span> <span class="token function">loadClassData</span><span class="token punctuation">(</span>name<span class="token punctuation">)</span><span class="token punctuation">;</span><span class="token comment">//1</span>
        <span class="token keyword">if</span> <span class="token punctuation">(</span>classData <span class="token operator">&#61;&#61;</span> null<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
            <span class="token keyword">throw</span> <span class="token keyword">new</span> <span class="token class-name">ClassNotFoundException</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span> <span class="token keyword">else</span> <span class="token punctuation">{<!-- --></span>
            clazz<span class="token operator">&#61;</span> <span class="token function">defineClass</span><span class="token punctuation">(</span>name<span class="token punctuation">,</span> classData<span class="token punctuation">,</span> <span class="token number">0</span><span class="token punctuation">,</span> classData<span class="token punctuation">.</span>length<span class="token punctuation">)</span><span class="token punctuation">;</span><span class="token comment">//2</span>
        <span class="token punctuation">}</span>
        <span class="token keyword">return</span> clazz<span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
    <span class="token keyword">private</span> <span class="token keyword">byte</span><span class="token punctuation">[</span><span class="token punctuation">]</span> <span class="token function">loadClassData</span><span class="token punctuation">(</span>String name<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
        String fileName <span class="token operator">&#61;</span> <span class="token function">getFileName</span><span class="token punctuation">(</span>name<span class="token punctuation">)</span><span class="token punctuation">;</span>
        File file <span class="token operator">&#61;</span> <span class="token keyword">new</span> <span class="token class-name">File</span><span class="token punctuation">(</span>path<span class="token punctuation">,</span>fileName<span class="token punctuation">)</span><span class="token punctuation">;</span>
        InputStream in<span class="token operator">&#61;</span>null<span class="token punctuation">;</span>
        ByteArrayOutputStream out<span class="token operator">&#61;</span>null<span class="token punctuation">;</span>
        <span class="token keyword">try</span> <span class="token punctuation">{<!-- --></span>
             in <span class="token operator">&#61;</span> <span class="token keyword">new</span> <span class="token class-name">FileInputStream</span><span class="token punctuation">(</span>file<span class="token punctuation">)</span><span class="token punctuation">;</span>
             out <span class="token operator">&#61;</span> <span class="token keyword">new</span> <span class="token class-name">ByteArrayOutputStream</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
            <span class="token keyword">byte</span><span class="token punctuation">[</span><span class="token punctuation">]</span> buffer <span class="token operator">&#61;</span> <span class="token keyword">new</span> <span class="token class-name">byte</span><span class="token punctuation">[</span><span class="token number">1024</span><span class="token punctuation">]</span><span class="token punctuation">;</span>
            <span class="token keyword">int</span> length<span class="token operator">&#61;</span><span class="token number">0</span><span class="token punctuation">;</span>
            <span class="token keyword">while</span> <span class="token punctuation">(</span><span class="token punctuation">(</span>length <span class="token operator">&#61;</span> in<span class="token punctuation">.</span><span class="token function">read</span><span class="token punctuation">(</span>buffer<span class="token punctuation">)</span><span class="token punctuation">)</span> <span class="token operator">!&#61;</span> <span class="token operator">-</span><span class="token number">1</span><span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                out<span class="token punctuation">.</span><span class="token function">write</span><span class="token punctuation">(</span>buffer<span class="token punctuation">,</span> <span class="token number">0</span><span class="token punctuation">,</span> length<span class="token punctuation">)</span><span class="token punctuation">;</span>
            <span class="token punctuation">}</span>
            <span class="token keyword">return</span> out<span class="token punctuation">.</span><span class="token function">toByteArray</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span> <span class="token keyword">catch</span> <span class="token punctuation">(</span><span class="token class-name">IOException</span> e<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
            e<span class="token punctuation">.</span><span class="token function">printStackTrace</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span><span class="token keyword">finally</span> <span class="token punctuation">{<!-- --></span>
            <span class="token keyword">try</span> <span class="token punctuation">{<!-- --></span>
                <span class="token keyword">if</span><span class="token punctuation">(</span>in<span class="token operator">!&#61;</span>null<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                    in<span class="token punctuation">.</span><span class="token function">close</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
                <span class="token punctuation">}</span>
            <span class="token punctuation">}</span> <span class="token keyword">catch</span> <span class="token punctuation">(</span><span class="token class-name">IOException</span> e<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                e<span class="token punctuation">.</span><span class="token function">printStackTrace</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
            <span class="token punctuation">}</span>
            <span class="token keyword">try</span><span class="token punctuation">{<!-- --></span>
                <span class="token keyword">if</span><span class="token punctuation">(</span>out<span class="token operator">!&#61;</span>null<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                    out<span class="token punctuation">.</span><span class="token function">close</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
                <span class="token punctuation">}</span>
            <span class="token punctuation">}</span><span class="token keyword">catch</span> <span class="token punctuation">(</span><span class="token class-name">IOException</span> e<span class="token punctuation">)</span><span class="token punctuation">{<!-- --></span>
                e<span class="token punctuation">.</span><span class="token function">printStackTrace</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
            <span class="token punctuation">}</span>
        <span class="token punctuation">}</span>
        <span class="token keyword">return</span> null<span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
    <span class="token keyword">private</span> String <span class="token function">getFileName</span><span class="token punctuation">(</span>String name<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
        <span class="token keyword">int</span> index <span class="token operator">&#61;</span> name<span class="token punctuation">.</span><span class="token function">lastIndexOf</span><span class="token punctuation">(</span><span class="token string">&#39;.&#39;</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token keyword">if</span><span class="token punctuation">(</span>index <span class="token operator">&#61;&#61;</span> <span class="token operator">-</span><span class="token number">1</span><span class="token punctuation">)</span><span class="token punctuation">{<!-- --></span><span class="token comment">//如果没有找到&#39;.&#39;则直接在末尾添加.class</span>
            <span class="token keyword">return</span> name<span class="token operator">&#43;</span><span class="token string">&#34;.class&#34;</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span><span class="token keyword">else</span><span class="token punctuation">{<!-- --></span>
            <span class="token keyword">return</span> name<span class="token punctuation">.</span><span class="token function">substring</span><span class="token punctuation">(</span>index<span class="token operator">&#43;</span><span class="token number">1</span><span class="token punctuation">)</span><span class="token operator">&#43;</span><span class="token string">&#34;.class&#34;</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span>
    <span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre> 
<p>这段代码有几点需要注意的&#xff0c;注释1处的loadClassData方法会获得class文件的字节码数组&#xff0c;并在注释2处调用defineClass方法将class文件的字节码数组转为Class类的实例。loadClassData方法中需要对流进行操作&#xff0c;关闭流的操作要放在finally语句块中&#xff0c;并且要对in和out分别采用try语句&#xff0c;如果in和out共同在一个try语句中&#xff0c;那么如果<code>in.close()</code>发生异常&#xff0c;则无法执行 <code>out.close()</code>。</p> 
<p>最后我们来验证DiskClassLoader是否可用&#xff0c;代码如下所示。</p> 
<pre><code class="prism language-java"><span class="token keyword">import</span> java<span class="token punctuation">.</span>lang<span class="token punctuation">.</span>reflect<span class="token punctuation">.</span>InvocationTargetException<span class="token punctuation">;</span>
<span class="token keyword">import</span> java<span class="token punctuation">.</span>lang<span class="token punctuation">.</span>reflect<span class="token punctuation">.</span>Method<span class="token punctuation">;</span>

<span class="token keyword">public</span> <span class="token keyword">class</span> <span class="token class-name">ClassLoaderTest</span> <span class="token punctuation">{<!-- --></span>
    <span class="token keyword">public</span> <span class="token keyword">static</span> <span class="token keyword">void</span> <span class="token function">main</span><span class="token punctuation">(</span>String<span class="token punctuation">[</span><span class="token punctuation">]</span> args<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
        DiskClassLoader diskClassLoader <span class="token operator">&#61;</span> <span class="token keyword">new</span> <span class="token class-name">DiskClassLoader</span><span class="token punctuation">(</span><span class="token string">&#34;D:\\lib&#34;</span><span class="token punctuation">)</span><span class="token punctuation">;</span><span class="token comment">//1</span>
        <span class="token keyword">try</span> <span class="token punctuation">{<!-- --></span>
            Class <span class="token class-name">c</span> <span class="token operator">&#61;</span> diskClassLoader<span class="token punctuation">.</span><span class="token function">loadClass</span><span class="token punctuation">(</span><span class="token string">&#34;com.example.Jobs&#34;</span><span class="token punctuation">)</span><span class="token punctuation">;</span><span class="token comment">//2</span>
            <span class="token keyword">if</span> <span class="token punctuation">(</span>c <span class="token operator">!&#61;</span> null<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                <span class="token keyword">try</span> <span class="token punctuation">{<!-- --></span>
                    Object obj <span class="token operator">&#61;</span> c<span class="token punctuation">.</span><span class="token function">newInstance</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
                    System<span class="token punctuation">.</span>out<span class="token punctuation">.</span><span class="token function">println</span><span class="token punctuation">(</span>obj<span class="token punctuation">.</span><span class="token function">getClass</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">.</span><span class="token function">getClassLoader</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
                    Method method <span class="token operator">&#61;</span> c<span class="token punctuation">.</span><span class="token function">getDeclaredMethod</span><span class="token punctuation">(</span><span class="token string">&#34;say&#34;</span><span class="token punctuation">,</span> null<span class="token punctuation">)</span><span class="token punctuation">;</span>
                    method<span class="token punctuation">.</span><span class="token function">invoke</span><span class="token punctuation">(</span>obj<span class="token punctuation">,</span> null<span class="token punctuation">)</span><span class="token punctuation">;</span><span class="token comment">//3</span>
                <span class="token punctuation">}</span> <span class="token keyword">catch</span> <span class="token punctuation">(</span><span class="token class-name">InstantiationException</span> <span class="token operator">|</span> IllegalAccessException
                        <span class="token operator">|</span> NoSuchMethodException
                        <span class="token operator">|</span> SecurityException <span class="token operator">|</span>
                        IllegalArgumentException <span class="token operator">|</span>
                        InvocationTargetException e<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
                    e<span class="token punctuation">.</span><span class="token function">printStackTrace</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
                <span class="token punctuation">}</span>
            <span class="token punctuation">}</span>
        <span class="token punctuation">}</span> <span class="token keyword">catch</span> <span class="token punctuation">(</span><span class="token class-name">ClassNotFoundException</span> e<span class="token punctuation">)</span> <span class="token punctuation">{<!-- --></span>
            e<span class="token punctuation">.</span><span class="token function">printStackTrace</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span>
    <span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre> 
<p>注释1出创建DiskClassLoader并传入要加载类的路径&#xff0c;注释2处加载Class文件&#xff0c;需要注意的是&#xff0c;不要在项目工程中存在名为com.example.Jobs的Java文件&#xff0c;否则就不会使用DiskClassLoader来加载&#xff0c;而是AppClassLoader来负责加载&#xff0c;这样我们定义DiskClassLoader就变得毫无意义。接下来在注释3通过反射来调用Jobs的say方法&#xff0c;打印结果如下&#xff1a;</p> 
<pre><code>com.example.DiskClassLoader&#64;4554617c
One more thing
</code></pre> 
<p>使用了DiskClassLoader来加载Class文件&#xff0c;say方法也正确执行&#xff0c;显然我们的目的达到了。</p> 
<h3><a id="_298"></a><strong>后记</strong></h3> 
<p>这一篇文章我们学习了Java中的ClassLoader&#xff0c;包括ClassLoader的类型、双亲委托模式、ClassLoader继承关系以及自定义ClassLoader&#xff0c;为的是就是更好的理解下一篇所要讲解的Android中的ClassLoader。</p> 
