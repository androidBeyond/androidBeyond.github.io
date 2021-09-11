---
layout:     post
title:      Android10 dex和oat文件格式分析
subtitle:   Android dex，odex，oat，vdex，art文件结构学习分析
date:       2020-12-13
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android10
    - Android
    - 组件学习
---

<p>在installd守护进程中有提到dexopt的操作，最后执行的操作是 run_dex2oat。本文将对dex和oat文件格式进行介绍分析。</p>
<pre><code>
run_dex2oat(input_fd.get(),
                 out_oat_fd.get(),
                 in_vdex_fd.get(),
                 out_vdex_fd.get(),
                 image_fd.get(),
                 dex_path,
                 out_oat_path,
                 swap_fd.get(),
                 instruction_set,
                 compiler_filter,
                 debuggable,
                 boot_complete,
                 background_job_compile,
                 reference_profile_fd.get(),
                 class_loader_context,
                 target_sdk_version,
                 enable_hidden_api_checks,
                 generate_compact_dex,
                 dex_metadata_fd.get(),
                 compilation_reason); 
</code></pre> 
<h2 id="一、Android-Dex文件"><a href="#一、Android-Dex文件" class="headerlink" title="一、Android Dex文件"></a>一、Android Dex文件</h2><p>Dex是Android平台上(Dalvik虚拟机)的可执行文件, 相当于Windows平台中的exe文件, 每个Apk安装包中都有dex文件, 里面包含了该app的所有源码, 通过反编译工具可以获取到相应的java源码。</p>
<p>在Android平台上，当java程序编译成class后，使用dx工具将所有的class文件整合到一个dex文件，目的是其中各个类能够共享数据，在一定程度上降低了冗余，同时也是文件结构更加经凑，dex文件是传统jar文件大小的50%左右。</p>
<h3 id="1-1-整体结构"><a href="#1-1-整体结构" class="headerlink" title="1.1 整体结构"></a>1.1 整体结构</h3>
<p><img src="https://img-blog.csdnimg.cn/a551c72a734040168e7627dfd4ac7f93.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt=""></p>
<h3 id="1-2-详细描述"><a href="#1-2-详细描述" class="headerlink" title="1.2 详细描述"></a>1.2 详细描述</h3><table>
<thead>
<tr>
<th><strong>名称</strong></th>
<th><strong>格式</strong></th>
<th><strong>说明</strong></th>
</tr>
</thead>
<tbody>
<tr>
<td>header</td>
<td>header_item</td>
<td>标头</td>
</tr>
<tr>
<td>string_ids</td>
<td>string_id_item[]</td>
<td>字符串标识符列表。这些是此文件使用的所有字符串的标识符，用于内部命名（例如类型描述符）或用作代码引用的常量对象。此列表必须使用 UTF-16 代码点值按字符串内容进行排序（不采用语言区域敏感方式），且不得包含任何重复条目。</td>
</tr>
<tr>
<td>type_ids</td>
<td>type_id_item[]</td>
<td>类型标识符列表。这些是此文件引用的所有类型（类、数组或原始类型）的标识符（无论文件中是否已定义）。此列表必须按 <code>string_id</code> 索引进行排序，且不得包含任何重复条目。</td>
</tr>
<tr>
<td>proto_ids</td>
<td>proto_id_item[]</td>
<td>方法原型标识符列表。这些是此文件引用的所有原型的标识符。此列表必须按返回类型（按 <code>type_id</code> 索引排序）主要顺序进行排序，然后按参数列表（按 <code>type_id</code> 索引排序的各个参数，采用字典排序方法）进行排序。该列表不得包含任何重复条目。</td>
</tr>
<tr>
<td>field_ids</td>
<td>field_id_item[]</td>
<td>字段标识符列表。这些是此文件引用的所有字段的标识符（无论文件中是否已定义）。此列表必须进行排序，其中定义类型（按 <code>type_id</code> 索引排序）是主要顺序，字段名称（按 <code>string_id</code> 索引排序）是中间顺序，而类型（按 <code>type_id</code> 索引排序）是次要顺序。该列表不得包含任何重复条目。</td>
</tr>
<tr>
<td>method_ids</td>
<td>method_id_item[]</td>
<td>方法标识符列表。这些是此文件引用的所有方法的标识符（无论文件中是否已定义）。此列表必须进行排序，其中定义类型（按 <code>type_id</code> 索引排序）是主要顺序，方法名称（按 <code>string_id</code> 索引排序）是中间顺序，而方法原型（按 <code>proto_id</code> 索引排序）是次要顺序。该列表不得包含任何重复条目。</td>
</tr>
<tr>
<td>class_defs</td>
<td>class_def_item[]</td>
<td>类定义列表。这些类必须进行排序，以便所指定类的超类和已实现的接口比引用类更早出现在该列表中。此外，对于在该列表中多次出现的同名类，其定义是无效的。</td>
</tr>
<tr>
<td>call_site_ids</td>
<td>call_site_id_item[]</td>
<td>调用站点标识符列表。这些是此文件引用的所有调用站点的标识符（无论文件中是否已定义）。此列表必须按 <code>call_site_off</code>的升序进行排序。</td>
</tr>
<tr>
<td>method_handles</td>
<td>method_handle_item[]</td>
<td>方法句柄列表。此文件引用的所有方法句柄的列表（无论文件中是否已定义）。此列表未进行排序，而且可能包含将在逻辑上对应于不同方法句柄实例的重复项。</td>
</tr>
<tr>
<td>data</td>
<td>ubyte[]</td>
<td>数据区，包含上面所列表格的所有支持数据。不同的项有不同的对齐要求；如有必要，则在每个项之前插入填充字节，以实现所需的对齐效果。</td>
</tr>
<tr>
<td>link_data</td>
<td>ubyte[]</td>
<td>静态链接文件中使用的数据。本文档尚未指定本区段中数据的格式。此区段在未链接文件中为空，而运行时实现可能会在适当的情况下使用这些数据。</td>
</tr>
</tbody>
</table>
<h3 id="1-3-用010editor查看dex文件"><a href="#1-3-用010editor查看dex文件" class="headerlink" title="1.3 用010editor查看dex文件"></a>1.3 用010editor查看dex文件</h3>
<p><img src="https://img-blog.csdnimg.cn/04b8f3b3e0aa4f00b03c3d19f52f0ec4.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt=""></p>
<p>查看每项的定义即情况，具体可以参考<a href="https://source.android.com/devices/tech/dalvik/dex-format" target="_blank" rel="noopener">https://source.android.com/devices/tech/dalvik/dex-format</a></p>
<p>也可以查看dex相关的code进行结构的梳理：</p>
<p><a href="https://android.googlesource.com/platform/dalvik/+/master/dx/src/com/android/dx/dex/file/DexFile.java" target="_blank" rel="noopener">https://android.googlesource.com/platform/dalvik/+/master/dx/src/com/android/dx/dex/file/DexFile.java</a></p>
<p>总之来说，dex是将apk中使用到的class文件信息集合在一起的文件，其中也包含了很多jar包中的类。<br> dex文件是对class文件中的各种函数表、变量表等进行优化过的，整体大小要小于class文件总和。</p>
<h2 id="二-Android-Odex，Oat，Vdex-art文件"><a href="#二-Android-Odex，Oat，Vdex-art文件" class="headerlink" title="二. Android Odex，Oat，Vdex , art文件"></a>二. Android Odex，Oat，Vdex , art文件</h2><h3 id="2-1-odex文件概述（-5-0之前-）"><a href="#2-1-odex文件概述（-5-0之前-）" class="headerlink" title="2.1 odex文件概述（ 5.0之前 ）"></a>2.1 odex文件概述（ 5.0之前 ）</h3><p>全名Optimized DEX，即优化过的DEX。<br> Apk在安装(installer)时，就会进行验证和优化，目的是为了校验代码合法性及优化代码执行速度，验证和优化后，会产生ODEX文件，运行Apk的时候，直接加载ODEX，避免重复验证和优化，加快了Apk的响应时间。</p>
<p>注意：优化过程会根据不同设备上Dalvik虚拟机的版本、Framework库的不同等因素而不同。在一台设备上被优化过的ODEX文件，拷贝到另一台设备上不一定能够运行。</p>
<p>ODEX格式及生成过程：<a href="https://www.jianshu.com/p/242abfb7eb7f" target="_blank" rel="noopener">https://www.jianshu.com/p/242abfb7eb7f</a></p>
<p>整体结构图如下：</p>
<p><img src="https://img-blog.csdnimg.cn/768f10d17e7345f89191e3307a459b8c.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt=""></p>
<h3 id="2-2-oat文件（5-0及5-0之后）"><a href="#2-2-oat文件（5-0及5-0之后）" class="headerlink" title="2.2. oat文件（5.0及5.0之后）"></a>2.2. oat文件（5.0及5.0之后）</h3><p>参考博客：<br> Android运行时ART加载OAT文件的过程分析：[<a href="https://blog.csdn.net/luoshengyang/article/details/39307813]" target="_blank" rel="noopener">https://blog.csdn.net/luoshengyang/article/details/39307813]</a><br> oat格式(1)：[<a href="https://shaomi.github.io/2017/08/18/oat%E6%A0%BC%E5%BC%8F/]" target="_blank" rel="noopener">https://shaomi.github.io/2017/08/18/oat%E6%A0%BC%E5%BC%8F/]</a><br> 从Android运行时出发，打造我们的脱壳神器：<br> [<a href="https://www.feiworks.com/wy/drops_html/%E4%BB%8EAndroid%E8%BF%90%E8%A1%8C%E6%97%B6%E5%87%BA%E5%8F%91%EF%BC%8C%E6%89%93%E9%80%A0%E6%88%91%E4%BB%AC%E7%9A%84%E8%84%B1%E5%A3%B3%E7%A5%9E%E5%99%A8.html]" target="_blank" rel="noopener">https://www.feiworks.com/wy/drops_html]</a></p>
<h4 id="2-2-1-oat文件概述"><a href="#2-2-1-oat文件概述" class="headerlink" title="2.2.1 oat文件概述"></a>2.2.1 oat文件概述</h4><p>oat 文件是 ART 运行的文件，是一种ELF格式的二进制可运行文件，包含 DEX 文件和编译出的本地机器指令文件。因为 oat 文件包含 DEX 文件，因此比 ODEX 文件占用空间更大。</p>
<p>由于其在安装时打包在里面的classes.dex文件会被工具dex2oat翻译成本地机器指令，最终得到一个ELF格式的OAT文件，ART 加载 OAT 文件后不需要经过处理就可以直接运行，它没有了从字节码装换成机器码的过程，因此运行速度更快。</p>
<p>查看三方应用的安装包如下：</p>
<pre><code>
PNC:/data/app/com.android.mywebviewtest-KtextonrnkHCwJ3LnA-5ZQ== # ls -la
total 1564
drwxr-xr-x 4 system system     4096 2018-01-01 10:42 .
drwxrwx--x 5 system system     4096 2018-01-02 09:30 ..
-rw-r--r-- 1 system system  1582454 2018-01-01 10:42 base.apk
drwxr-xr-x 2 system system     4096 2018-01-01 10:42 lib
drwxrwx--x 3 system install    4096 2018-01-01 10:42 oat
PNC:/data/app/com.android.mywebviewtest-KtextonrnkHCwJ3LnA-5ZQ== # cd oat
PNC:/data/app/com.android.mywebviewtest-KtextonrnkHCwJ3LnA-5ZQ==/oat # ls -la
total 12
drwxrwx--x 3 system install 4096 2018-01-01 10:42 .
drwxr-xr-x 4 system system  4096 2018-01-01 10:42 ..
drwxrwx--x 2 system install 4096 2018-01-01 12:06 arm64
PNC:/data/app/com.android.mywebviewtest-KtextonrnkHCwJ3LnA-5ZQ==/oat # cd arm64
PNC:/data/app/com.android.mywebviewtest-KtextonrnkHCwJ3LnA-5ZQ==/oat/arm64 # ls -la
total 2192
drwxrwx--x 2 system install    4096 2018-01-01 12:06 .
drwxrwx--x 3 system install    4096 2018-01-01 10:42 ..
-rw-r----- 1 system all_a71   32768 2018-01-01 12:06 base.art
-rw-r----- 1 system all_a71   45696 2018-01-01 10:42 base.odex
-rw-r----- 1 system all_a71 2152664 2018-01-01 12:06 base.vdex
PNC:/data/app/com.android.mywebviewtest-KtextonrnkHCwJ3LnA-5ZQ==/oat/arm64 # </code></pre> 
<p>怎么还是odex，这个art文件是啥，这个vdex又是啥，</p>
<p>官方回答：<a href="https://source.android.com/devices/tech/dalvik/configure" target="_blank" rel="noopener">https://source.android.com/devices/tech/dalvik/configure</a> .</p>
<p>vdex：其中包含 APK 的未压缩 DEX 代码，另外还有一些旨在加快验证速度的元数据。 </p>
<p>odex：其中包含 APK 中已经过 AOT 编译的方法代码。 </p>
<p>art (optional)：其中包含 APK 中列出的某些字符串和类的 ART 内部表示，用于加快应用启动速度。</p>
<p>使用file命令查看这个base.odex</p>
<pre><code>
PNC:/data/app/com.android.mywebviewtest-KtextonrnkHCwJ3LnA-5ZQ==/oat/arm64 # file base.odex
base.odex: ELF shared object, 64-bit LSB arm64, stripped </code></pre> 
<p><strong>看到这个base.odex文件是ELF格式封装的，所以这里的odex其实就是oat文件，只是还是叫odex后缀。</strong></p>
<p>查看系统自带应用，比如system/priv-app/，system/app/中的apk，最终oat文件存放在/data/dalvik-cache/ 中：</p>
<pre><code>
PNC:/data/dalvik-cache/arm64 #
130|PNC:/data/dalvik-cache/arm64 # ls -la
total 132060
drwx--x--x 2 root   root         28672 2018-01-01 12:07 .
drwxrwx--x 4 root   root          4096 2018-01-01 08:01 ..
-rw-r----- 1 system system        8192 2018-01-01 12:06 system@app@AntHalService@AntHalService.apk@classes.art
-rw-r----- 1 system system       21120 2009-01-01 00:00 system@app@AntHalService@AntHalService.apk@classes.dex
-rw-r----- 1 system system       22850 2018-01-01 12:06 system@app@AntHalService@AntHalService.apk@classes.vdex
-rw-r----- 1 system radio        12288 2018-01-01 12:04 system@app@AutoRegistration@AutoRegistration.apk@classes.art
-rw-r----- 1 system radio        17024 2009-01-01 00:00 system@app@AutoRegistration@AutoRegistration.apk@classes.dex
-rw-r----- 1 system radio        60522 2018-01-01 12:04 system@app@AutoRegistration@AutoRegistration.apk@classes.vdex
-rw-r----- 1 system all_a32       8192 2018-01-01 12:05 system@app@BasicDreams@BasicDreams.apk@classes.art
-rw-r----- 1 system all_a32      17024 2009-01-01 00:00 system@app@BasicDreams@BasicDreams.apk@classes.dex
-rw-r----- 1 system all_a32      15952 2018-01-01 12:05 system@app@BasicDreams@BasicDreams.apk@classes.vdex
-rw-r--r-- 1 system bluetooth 12911232 2009-01-01 00:00 system@app@Bluetooth@Bluetooth.apk@classes.dex
-rw-r--r-- 1 system bluetooth  6188408 2018-01-01 12:07 system@app@Bluetooth@Bluetooth.apk@classes.vdex
-rw-r----- 1 system bluetooth    16384 2018-01-01 12:05 system@app@BluetoothExt@BluetoothExt.apk@classes.art
-rw-r----- 1 system bluetooth    21120 2009-01-01 00:00 system@app@BluetoothExt@BluetoothExt.apk@classes.dex
-rw-r----- 1 system bluetooth   288680 2018-01-01 12:05 system@app@BluetoothExt@BluetoothExt.apk@classes.vdex
-rw-r----- 1 system all_a33       8192 2018-01-01 12:07 system@app@BluetoothMidiService@BluetoothMidiService.apk@classes.art
-rw-r----- 1 system all_a33      17024 2009-01-01 00:00 system@app@BluetoothMidiService@BluetoothMidiService.apk@classes.dex
-rw-r----- 1 system all_a33      19518 2018-01-01 12:07 system@app@BluetoothMidiService@BluetoothMidiService.apk@classes.vde
-rw-r----- 1 system all_a34       8192 2018-01-01 12:06 system@app@BookmarkProvider@BookmarkProvider.apk@classes.art
-rw-r----- 1 system all_a34      17024 2009-01-01 00:00 system@app@BookmarkProvider@BookmarkProvider.apk@classes.dex
-rw-r----- 1 system all_a34       1580 2018-01-01 12:06 system@app@BookmarkProvider@BookmarkProvider.apk@classes.vdex
-rw-r----- 1 system system       32768 2018-01-01 12:04 system@app@CITTest@CITTest.apk@classes.art
-rw-r----- 1 system system       74368 2009-01-01 00:00 system@app@CITTest@CITTest.apk@classes.dex
-rw-r----- 1 system system     4172460 2018-01-01 12:04 system@app@CITTest@CITTest.apk@classes.vdex
-rw-r----- 1 system all_a35      24576 2018-01-01 12:05 system@app@Calendar@Calendar.apk@classes.art
-rw-r----- 1 system all_a35      29312 2009-01-01 00:00 system@app@Calendar@Calendar.apk@classes.dex
-rw-r----- 1 system all_a35    1122530 2018-01-01 12:05 system@app@Calendar@Calendar.apk@classes.vdex
-rw-r----- 1 system system        8192 2018-01-01 12:04 system@app@CallEnhancement@CallEnhancement.apk@classes.art
-rw-r----- 1 system system       17024 2009-01-01 00:00 system@app@CallEnhancement@CallEnhancement.apk@classes.dex
-rw-r----- 1 system system       36950 2018-01-01 12:04 system@app@CallEnhancement@CallEnhancement.apk@classes.vdex
-rw-r----- 1 system radio         8192 2018-01-01 12:05 system@app@CallFeaturesSetting@CallFeaturesSetting.apk@classes.art
-rw-r----- 1 system radio        17024 2009-01-01 00:00 system@app@CallFeaturesSetting@CallFeaturesSetting.apk@classes.dex
-rw-r----- 1 system radio        52222 2018-01-01 12:05 system@app@CallFeaturesSetting@CallFeaturesSetting.apk@classes.vdex
 </code></pre> 
<p>这里的dex文件也是oat文件，只是以dex为后缀命名，/data/dalvik-cache/ 下也有以oat为后缀命名的oat文件，通过如上的file命令就可以看出来。</p>
<pre><code>
PNC:/data/dalvik-cache/arm64 #
130|PNC:/data/dalvik-cache/arm64 # file system@app@BluetoothExt@BluetoothExt.apk@classes.dex
system@app@BluetoothExt@BluetoothExt.apk@classes.dex: ELF shared object, 64-bit LSB arm64, stripped
 </code></pre> 
<h4 id="2-2-2-readelf-查看oat文件结构"><a href="#2-2-2-readelf-查看oat文件结构" class="headerlink" title="2.2.2 readelf 查看oat文件结构"></a>2.2.2 readelf 查看oat文件结构</h4><p>安装包中的oat文件由于是elf格式封装，可以使用readelf命令查看文件信息如下：</p>
<pre><code>
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 03 00 00 00 00 00 00 00 00 
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - GNU
  ABI Version:                       0
  Type:                              DYN (Shared object file)
  Machine:                           ARM
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          52 (bytes into file)
  Start of section headers:          4261112 (bytes into file)
  Flags:                             0x5000000, Version5 EABI
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         8
  Size of section headers:           40 (bytes)
  Number of section headers:         11
  Section header string table index: 10

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .rodata           PROGBITS        00001000 001000 1c9000 00   A  0   0 4096
  [ 2] .text             PROGBITS        001ca000 1ca000 22f98c 00  AX  0   0 4096
  [ 3] .bss              NOBITS          003fa000 000000 0079b8 00   A  0   0 4096
  [ 4] .dex              NOBITS          00402000 000000 40dacf4 00   A  0   0 4096
  [ 5] .dynstr           STRTAB          044dd000 3fa000 00006d 00   A  0   0 4096
  [ 6] .dynsym           DYNSYM          044dd070 3fa070 0000a0 10   A  5   1  4
  [ 7] .hash             HASH            044dd110 3fa110 000034 04   A  6   0  4
  [ 8] .dynamic          DYNAMIC         044de000 3fb000 000038 08   A  5   0 4096
  [ 9] .gnu_debugdata    PROGBITS        00000000 3fc000 0144a4 00      0   0 4096
  [10] .shstrtab         STRTAB          00000000 4104a4 000051 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)

There are no section groups in this file.

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  PHDR           0x000034 0x00000034 0x00000034 0x00100 0x00100 R   0x4
  LOAD           0x000000 0x00000000 0x00000000 0x1ca000 0x1ca000 R   0x1000
  LOAD           0x1ca000 0x001ca000 0x001ca000 0x22f98c 0x22f98c R E 0x1000
  LOAD           0x000000 0x003fa000 0x003fa000 0x00000 0x079b8 RW  0x1000
  LOAD           0x000000 0x00402000 0x00402000 0x00000 0x40dacf4 R   0x1000
  LOAD           0x3fa000 0x044dd000 0x044dd000 0x00144 0x00144 R   0x1000
  LOAD           0x3fb000 0x044de000 0x044de000 0x00038 0x00038 RW  0x1000
  DYNAMIC        0x3fb000 0x044de000 0x044de000 0x00038 0x00038 RW  0x1000

 Section to Segment mapping:
  Segment Sections...
   00     
   01     .rodata 
   02     .text 
   03     .bss 
   04     .dex 
   05     .dynstr .dynsym .hash 
   06     .dynamic 
   07     .dynamic 

Dynamic section at offset 0x3fb000 contains 7 entries:
  Tag        Type                         Name/Value
 0x00000004 (HASH)                       0x44dd110
 0x00000005 (STRTAB)                     0x44dd000
 0x00000006 (SYMTAB)                     0x44dd070
 0x0000000b (SYMENT)                     16 (bytes)
 0x0000000a (STRSZ)                      109 (bytes)
 0x0000000e (SONAME)                     Library soname: [base.odex]
 0x00000000 (NULL)                       0x0

There are no relocations in this file.

There are no unwind sections in this file.

Symbol table '.dynsym' contains 10 entries:
   Num:    Value  Size Type    Bind   Vis      Ndx Name
     0: 00000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 00001000 0x1c9000 OBJECT  GLOBAL DEFAULT    1 oatdata
     2: 001ca000     0 OBJECT  GLOBAL DEFAULT    2 oatexec
     3: 003f9988     4 OBJECT  GLOBAL DEFAULT    2 oatlastword
     4: 003fa000  8456 OBJECT  GLOBAL DEFAULT    3 oatbss
     5: 003fa000  8456 OBJECT  GLOBAL DEFAULT    3 oatbssmethods
     6: 003fc108 22704 OBJECT  GLOBAL DEFAULT    3 oatbssroots
     7: 004019b4     4 OBJECT  GLOBAL DEFAULT    3 oatbsslastword
     8: 00402000     0 OBJECT  GLOBAL DEFAULT    4 oatdex
     9: 044dccf0     4 OBJECT  GLOBAL DEFAULT    4 oatdexlastword

Histogram for bucket list length (total of 1 buckets):
 Length  Number     % of total  Coverage
      0  1          (100.0%)

No version information found in this file. </code></pre> 
<h4 id="2-2-3-oat文件结构大致如图"><a href="#2-2-3-oat文件结构大致如图" class="headerlink" title="2.2.3 oat文件结构大致如图"></a>2.2.3 oat文件结构大致如图</h4><p><img src="https://img-blog.csdnimg.cn/1ec771a3cbb3413e876e7c1fd1c646c7.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt=""></p>
<p>如上图应知：</p>
<p>1.oat文件中有完整的dex文件，oat data section 中对应着真正的oat文件，即 外层是elf 包含着 oat，oat 包含着dex<br>2.符号oatdata和oatlastword分别指定了oat文件在elf文件中的头和尾的位置，符号oatexec指向可执行段的位置；<br>3.oat文件有自己的头和格式，并且其内部包含了一个完整的dex文件。<br>4.oat其实就是一个Elf格式的二进制文件，跟Elf文件不同的是它内部多了oatdata、oatexec、oatlastword几个符号。其中oatdata的起始位置相对文件头固定为0x1000字节，而我们通过oatdump反编译的时候出来的地址是从0x1000开始的，所以这也是为什么我们在backtrace中计算地址的时候最后要减去0x1000，才能去dump里面找对应的地址。</p>
<p><img src="https://img-blog.csdnimg.cn/f663e0776e1640c3895125cdb1fb11c9.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_15,color_FFFFFF,t_70,g_se,x_16" alt=""></p>
<p>oat文件格式如图所示，这里0x1000是oatdata相对于文件头的偏移，接着就是oatdata的大小，也就是oatdump中的executable_offset，这个值保存在Oat文件的OatHeader里面。然后就是oatexec段，也就是机器码。而进程运行过程中异常如果挂在oat文件中，那么其pc一定是在oatexec段内。</p>
<h3 id="2-3-art文件"><a href="#2-3-art文件" class="headerlink" title="2.3. art文件"></a>2.3. art文件</h3><p>看到上面有art文件，.art是一些类/filed/方法，app启动直接map到内存，从odex中拆分出来的,art文件主要为了加快应用的对“热代码”的加载与缓存</p>
<h3 id="2-4-vdex文件"><a href="#2-4-vdex文件" class="headerlink" title="2.4. vdex文件"></a>2.4. vdex文件</h3><p>google在android8.0新增加了vdex文件，其中包含 APK 的未压缩 DEX 代码，另外还有一些旨在加快验证速度的元数据。</p>
<p>VDEX 文件有助于提升软件更新的性能和用户体验。VDEX 文件会存储包含验证程序依赖项且经过预验证的 DEX 文件，以便 ART 在系统更新期间无需再次解压和验证 DEX 文件。无需执行任何操作，即可实现该功能。该功能默认处于启用状态。要停用该功能，请将 ART_ENABLE_VDEX 环境变量设为 false。</p>
<p>定义结构：art/runtime/vdex_file.h</p>
<pre><code>
34// VDEX files contain extracted DEX files. The VdexFile class maps the file to
35// memory and provides tools for accessing its individual sections.
36//
37// File format:
38//   VdexFile::VerifierDepsHeader    fixed-length header
39//      Dex file checksums
40//
41//   Optionally:
42//      VdexFile::DexSectionHeader   fixed-length header
43//
44//      quicken_table_off[0]  offset into QuickeningInfo section for offset table for DEX[0].
45//      DEX[0]                array of the input DEX files, the bytecode may have been quickened.
46//      quicken_table_off[1]
47//      DEX[1]
48//      ...
49//      DEX[D]
50//
51//   VerifierDeps
52//      uint8[D][]                 verification dependencies
53//
54//   Optionally:
55//      QuickeningInfo
56//        uint8[D][]                  quickening data
57//        uint32[D][]                 quickening data offset tables </code></pre> 
<h2 id="三-总结"><a href="#三-总结" class="headerlink" title="三. 总结"></a>三. 总结</h2>
<p><img src="https://img-blog.csdnimg.cn/ea08cebde4d64c2a9d0876dc3be285e3.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_20,color_FFFFFF,t_70,g_se,x_16" alt=""></p>
<h2 id="四-工具使用"><a href="#四-工具使用" class="headerlink" title="四.工具使用"></a>四.工具使用</h2><h3 id="4-1-dx-和-dexdump工具"><a href="#4-1-dx-和-dexdump工具" class="headerlink" title="4.1.dx 和 dexdump工具"></a>4.1.dx 和 dexdump工具</h3><p>在sdk目录：~/Android/Sdk/build-tools/ 下有dex 打包工具:dx </p>
<p>解析dex的工具：dexdump;</p>
<p>源码目录下prebuilts下也有该工具;</p>
<p>手机中也有该工具： system/bin/dexdump</p>
<h3 id="4-2-oatdump工具"><a href="#4-2-oatdump工具" class="headerlink" title="4.2. oatdump工具"></a>4.2. oatdump工具</h3><p>在手机中有该工具： /system/bin/oatdump，可以解析oat文件。</p>
<h3 id="4-3-objdump工具"><a href="#4-3-objdump工具" class="headerlink" title="4.3.objdump工具"></a>4.3.objdump工具</h3><p>对于android开发是源码下对应的arm-linux-androideabi-objdump而非电脑系统的objdump：</p>
<p>./prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9/bin/arm-linux-androideabi-objdump  -S ~/Documents/f3b/symbol/out/target/product/pyxis/symbols/vendor/lib/hw/camera.qcom.so | tee camera.qcom.asm<br> 用来反编译symbol文件为汇编指令用于问题定位。</p>
<h3 id="4-4-Vdex-Extractor工具"><a href="#4-4-Vdex-Extractor工具" class="headerlink" title="4.4. Vdex Extractor工具"></a>4.4. Vdex Extractor工具</h3><p>Vdex Extractor：从Vdex文件反编译和提取Android Dex字节码的工具：[<a href="https://www.freebuf.com/sectool/185881.html]" target="_blank" rel="noopener">https://www.freebuf.com/sectool/185881.html]</a></p>
<h2 id="参考"><a href="#参考" class="headerlink" title="参考"></a>参考</h2><p>Android[art]-Android dex，odex，oat，vdex，art文件结构学习总结：<a href="https://www.jianshu.com/p/0f1b0bdd6e42Android" target="_blank" rel="noopener">https://www.jianshu.com/p/0f1b0bdd6e42Android</a> Dex文件格式(一)：[<a href="https://blog.csdn.net/p312011150/article/details/80501690]" target="_blank" rel="noopener">https://blog.csdn.net/p312011150/article/details/80501690]</a><br>dex文件解析(第三篇) ：[<a href="https://blog.csdn.net/tabactivity/article/details/78950379]" target="_blank" rel="noopener">https://blog.csdn.net/tabactivity/article/details/78950379]</a><br>Android安全–Dex文件格式详解：[<a href="https://www.cnblogs.com/kexing/p/8890162.html]" target="_blank" rel="noopener">https://www.cnblogs.com/kexing/p/8890162.html]</a><br>Dalvik和Art,JIT ,AOT, oat, dex, odex：[<a href="https://www.colabug.com/4516410.html]" target="_blank" rel="noopener">https://www.colabug.com/4516410.html]</a><br>官方文档：[<a href="https://source.android.com/devices/tech/dalvik/dex-format]" target="_blank" rel="noopener">https://source.android.com/devices/tech/dalvik/dex-format]</a></p>
