---
layout:     post
title:      Android Monkey hang机问题分析总结
subtitle:   最近公司项目在monkey过程中出现大量的hang机重启问题，block项目进度，影响CF
date:       2022-03-15
author:     coderman
header-img: img/article-bg.jpg
top: false
no-catalog: false
tags:
    - android
    - 死机重启
    - Monkey
--- 
<p>
<img src="" alt="" />
</p>
<p> </p>
<h4> 前言 </h4>
最近公司项目在monkey过程中出现大量的hang机重启问题，block项目进度，影响CF,领导要求重点攻关,在此记录下攻关分析过程
<h4>初步分析 </h4>
<p>从hang 机的DB发现是swap耗光导致，有camera的相关信息打印，此hang 机问题问题是kernel 一套踢狗机制，基本都是kernel 的相关log,一般是bsp同事入手分析，会较快的解决问题，如下是看到的DB log：</p>

<p>
<img src="https://img-blog.csdnimg.cn/bec5f288db394e6fbc52e9016d65182b.png" alt="" />
</p>
<p>根据上图我们看到swap为0，有大量的Camera信息打印，所以怀疑可能是camera有内存泄漏导致，经过camera排查未见异常，通过dump一段时间的内存信息并未发现异常，只能进一步做一些反向排查。正向排查不能很快定位到异常根因的，反向排查也是看问题的非常重要的手段，有时反向排查或者侧面的手段解问题反而更有效更快。 </p>
<p>从现有日志的信息只能确定是低内存 io高导致，每次有camera相关进程信息打印。
测试部同事协助做了大量的工作：<br>
1,排查16号版本-验证是否16号前后版本导致->16号版本仍然复现。<br>
2,排查2号版本测试- 验证是否之前的版本有问题->能复现<br>
3,填充内存测试->复现问题<br>
4,填充内存对相机进行单测->复现问题<br>
5,monkey整机安装大应用测试复现->复现问题<br>
做了如上测试后会有一些新的怀疑点，从而可以从增加一些正向分析排查的点。</p>
<p>
通过相关排查，和之前怀疑的近期修改无关，问题一直存在，只是某些修改如性能调优等让概率变大，我们同步进行如下排查和处理：<br>
1，回退lmkd 水位相关修改进行整机monkey测试->复现问题。<br>
2，打开kmemleak定期抓取内存信息排查内存泄漏->未获取到内存泄漏相关异常。<br>
3，camera 进行排查内存是否有大内存申请->未获取异常信息。<br>
4，驱动进行排查是否有底层异常->怀疑CMDQ-但mtk确认无关。<br>
5，合入mtk 上层内存泄漏bug的patch->复现问题。<br>
通过如上工作基本确定是低内存导致，导致低内存的原因一般是内存泄漏或者有大应用一直被启动或者启动太多进程导致，也就是是查杀机制如kswapd,lmkd出现异常，所以加lmkd异常log的同时打开一些开关继续定期抓取slabinfo内存信息排查泄漏。
</p>
<h4>抓取kmemleak和slabinfo</h4>
<strong>Kmemleak</strong><br>
<p>Kmemleak 为kernel 2.6.31引入的工具，用于检查内存泄漏。
可以追踪kmalloc(), vmalloc(), kmem_cache_alloc()等函数引起的内存泄漏，一般用于slab内存泄漏，<br>
打开kmemleak:<br>
    CONFIG_DEBUG_KMEMLEAK=y  <br>
    CONFIG_DEBUG_KMEMLEAK_DEFAULT_OFF=n<br>
    CONFIG_DEBUG_KMEMLEAK_EARLY_LOG_SIZE =40000//logsize最大<br>
adb shell 进去 看是否存在sys/kernel/debug/kmemleak这个节点，如果存在表明enable。<br>
第一次scan：echo scan > sys/kernel/debug/kmemleak//开始扫描 <br>
然后 cat sys/kernel/debug/kmemleak   会得到很多backtrace，但是这其中有些是误抓的
（kmemleak存在误报情况）<br>
然后echo clear > sys/kernel/debug/kmemleak   清除log <br>
第二次scan：echo scan > sys/kernel/debug/kmemleak//开始扫描<br>
过段时间等待leak的积累，然后 cat sys/kernel/debug/kmemleak  <br>
很多第一次误报的backtrace没有了，会得到很多重复的backtrace，假设这样的backtrace
称为A <br>
kmemleak的特征是A backtrace会越来越多，不断增长,而且这里就是泄漏的点
<strong>slabinfo</strong>
<p>
通过抓取slabinfo持续关注slabtrace 是否有明显增长的backtrace, 即可能为泄露点
打开slabinfo：
     CONFIG_SLUB=y
     CONFIG_SLUB_DEBUG=y
     CONFIG_SLUB_DEBUG_ON=y
如上其实都是从kernel方面入手分析的一些检查内存泄漏的手段，上层其实只是做些辅助，为提高效率便写了一个定时抓取各种内存信息的脚本，处理没有固定复现路径的问题很有用.
在继续看mainlog的过程中，我们发现monkey dump的procrank 数据中swap是基本被所有的进程用光，各个进程的内存并没有比较离谱的，都是正常的申请，</p>
<p>
<img src="https://img-blog.csdnimg.cn/0e0000f34fb0435a93fc22e21d8ddaef.png" alt="" />
</p>
所以怀疑是查杀出了问题并做了如下处理：<br>
   1，修改swapnass和psi测试->未复现。<br>
   2，负责性能的同事添加lmkd的相关log打印排查lmkd为何后期未查杀进程。<br>
   通过复测我们误认为是psi查杀和swapnass的修改修复了问题，其实凑巧psi的修改关闭了导致问题的feature,使得问题被意外的处理掉了，最终是lmkd中复现抓到的添加的日志，看到的问题的根因。
<h4>问题根因</h4>
 LMK没有去杀进程：
 ActivityManager和LMK的Socket连接断开了一次(相机会在monkey过程中被频繁拉起，不断的更新lmkd参数，
 概率会导致socket达到了最大数而重连)，而重新连接的时候，会把水位值重新设置下去。设置水位我们有客制化代码，
 在开机时设置的时候，用了我们修改后的水位值，用了一个新的变量mFixOomMinFree来保存，这个时候走了我司的Feature,
 初始化了客制化的水位，原生的水位不会被克制化，但是在ActivityManager和LMK的Socket连接断开重新连接时，
 设置水位时用的mOomMinFree变量，这个原生变量由于没有初始化始终为0 所以导致往LMK设置水位时，设置了0kB，
 即下面LOG的 limit值，即剩余内存低于0kB时才会杀进程，这就导致了LMK不会再去杀进程。   
 <p>
<img src="https://img-blog.csdnimg.cn/5918bf0ce04d406cbbd05676855badb5.png" alt="" />
</p>
相关代码如下
 <p>
<img src="https://img-blog.csdnimg.cn/6964471c8dd44c4c8be31be1be01b832.png" alt="" />
</p>
相关客制化feature如下
CustomFeature.DIS_OOM_PARAM_AUTO_CALC
<h4>解决方案</h4>
在onLmkdConnect 重新初始化的时候加上Feature判断,代码在此就不列出来了,知道思路就行
<h4>总结</h4>
此类问题出现的非常隐蔽，复现了也很难找到根因,且在特定的条件下才能复现，除非是对lmkd做过修改或者业务非常熟悉，否则就是造问题简单发现非常难。
最终也是曾经添加过这个feature的同事怀疑这里有问题才加的日志，后期项目应该对公司的相关Feature进行梳理排雷，
很多Feature可能在前一个Android的版本上是好的，但是随着升级可能有很大的漏洞或者难发现的漏洞，造成不可预测的奇怪问题。导致项目后期非常多的问题发生且都不好排查。
