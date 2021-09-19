---
layout:     post
title:      Android10 Binder机制8-驱动层分析
subtitle:   Binder驱动是Android专用的，但底层的驱动架构与Linux驱动一样。binder驱动在以misc设备进行注册
date:       2021-03-28
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
 
 <h2 id="一、概述"><a href="#一、概述" class="headerlink" title="一、概述"></a>一、概述</h2><p>Binder驱动是Android专用的，但底层的驱动架构与Linux驱动一样。binder驱动在以misc设备进行注册，作为虚拟字符设备，没有直接操作硬件，只是对设备内存的处理，主要是驱动设备的初始化（binder_init）,打开（binder_open）,映射（binder_mmap）,数据操作（binder_ioctl）。</p>
<ol>
<li>通过init()，创建/dev/binder设备节点</li>
<li>通过open()，获取Binder Driver的文件描述符</li>
<li>通过mmap()，在内核分配一块内存，用于存放数据</li>
<li>通过ioctl()，将IPC数据作为参数传递给Binder Driver</li>
</ol>
<p>用户态的程序调用Kernel层驱动是需要陷入内核态，进行系统调用（syscall），比如打开Binder驱动方法的调用链是：open()-&gt;__open()-&gt;binder_open()。open()为用户空间的方法，  _open()是系统调用中相应的处理方法，通过查找，对应调用到内核binder驱动的binder_open方法。</p>
<p><img src="https://img-blog.csdnimg.cn/80d4e791f3454a3abe870c858289d07c.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_9,color_FFFFFF,t_70,g_se,x_16" alt="systemcall" style="zoom:80%;"></p>
<p>Client进程通过RPC(Remote Procedure Call Protocol)与Server通信，可以简单的分为三层，驱动层、IPC层、业务层。demo()是client和server共同协商好的统一方法，RPC数据、code、handle、协议这四项组成了IPC的层的数据，通过IPC层进行数据传输，而真正在Client和Server两端建立通信的基础设施是Binder Driver。</p>
<p><img src="https://img-blog.csdnimg.cn/c93441db5ede4144ad5a5ff79e423e57.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_18,color_FFFFFF,t_70,g_se,x_16" alt="binderdriver_frame" style="zoom:80%;"></p>
<p>例如：当AMS的client向ServiceManger注册服务的过程中，IPC层的数据组成为：handle=0，RPC数据为AMS,code为ADD_SERVICE_TRANSACTION，binder协议为BC_TRANSACTION。</p>
<h2 id="二、Binder核心方法"><a href="#二、Binder核心方法" class="headerlink" title="二、Binder核心方法"></a>二、Binder核心方法</h2><h3 id="2-1-binder-init"><a href="#2-1-binder-init" class="headerlink" title="2.1 binder_init"></a>2.1 binder_init</h3><p>[-&gt;android/binder.c]</p>

<pre><code>
static int __init binder_init(void)
{
	int ret;
	char *device_name, *device_names;
	struct binder_device *device;
	struct hlist_node *tmp;
    //初始化shrinker
	binder_alloc_shrinker_init();

	atomic_set(&binder_transaction_log.cur, ~0U);
	atomic_set(&binder_transaction_log_failed.cur, ~0U);
     
    //创建一序列文件
	binder_debugfs_dir_entry_root = debugfs_create_dir("binder", NULL);
	if (binder_debugfs_dir_entry_root)
		binder_debugfs_dir_entry_proc = debugfs_create_dir("proc",
						 binder_debugfs_dir_entry_root);

	if (binder_debugfs_dir_entry_root) {
		debugfs_create_file("state",
				    S_IRUGO,
				    binder_debugfs_dir_entry_root,
				    NULL,
				    &binder_state_fops);
		debugfs_create_file("stats",
				    S_IRUGO,
				    binder_debugfs_dir_entry_root,
				    NULL,
				    &binder_stats_fops);
		debugfs_create_file("transactions",
				    S_IRUGO,
				    binder_debugfs_dir_entry_root,
				    NULL,
				    &binder_transactions_fops);
		debugfs_create_file("transaction_log",
				    S_IRUGO,
				    binder_debugfs_dir_entry_root,
				    &binder_transaction_log,
				    &binder_transaction_log_fops);
		debugfs_create_file("failed_transaction_log",
				    S_IRUGO,
				    binder_debugfs_dir_entry_root,
				    &binder_transaction_log_failed,
				    &binder_transaction_log_fops);
	}

	/*
	 * Copy the module_parameter string, because we don't want to
	 * tokenize it in-place.
	 */
	//分配内存
	device_names = kzalloc(strlen(binder_devices_param) + 1, GFP_KERNEL);
	if (!device_names) {
		ret = -ENOMEM;
		goto err_alloc_device_names_failed;
	}
	strcpy(device_names, binder_devices_param);

	while ((device_name = strsep(&device_names, ","))) {
	    //注册misc设置，见2.2.1
		ret = init_binder_device(device_name);
		if (ret)
			goto err_init_binder_device_failed;
	}

	return ret;

err_init_binder_device_failed:
	hlist_for_each_entry_safe(device, tmp, &binder_devices, hlist) {
		misc_deregister(&device->miscdev);
		hlist_del(&device->hlist);
		kfree(device);
	}
err_alloc_device_names_failed:
	debugfs_remove_recursive(binder_debugfs_dir_entry_root);

	return ret;
}</code></pre>

<p>主要工作是注册misc设备。</p>
<p>debugfs_create_file是指在debugfs文件系统中创建一个目录，返回值是指向dentry的指针。当kernel禁用debugfs的话，返回值是%ENODEV，默认是禁用的。如果需要打开，在目录kernel/arch/arm64/configs下找到defconfig文件下添加一行CONFIG_DEBUG_FS=y，即可开启debugfs。</p>
<h4 id="2-1-1-init-binder-device"><a href="#2-1-1-init-binder-device" class="headerlink" title="2.1.1 init_binder_device"></a>2.1.1 init_binder_device</h4><p>[-&gt;android/binder.c]</p>

<pre><code>
static int __init init_binder_device(const char *name)
{
	int ret;
	struct binder_device *binder_device;

	binder_device = kzalloc(sizeof(*binder_device), GFP_KERNEL);
	if (!binder_device)
		return -ENOMEM;
    //设备的文件操作结构，这是file_operation结构
	binder_device-&gt;miscdev.fops = &binder_fops;
	//次设备号，动态分配
	binder_device-&gt;miscdev.minor = MISC_DYNAMIC_MINOR;
	//设备名
	binder_device-&gt;miscdev.name = name;

	binder_device-&gt;context.binder_context_mgr_uid = INVALID_UID;
	binder_device-&gt;context.name = name;
	mutex_init(&binder_device-&gt;context.context_mgr_node_lock);
    //注册misc设备
	ret = misc_register(&binder_device-&gt;miscdev);
	if (ret &lt; 0) {
		kfree(binder_device);
		return ret;
	}
    //添加到链表头部
	hlist_add_head(&binder_device-&gt;hlist, &binder_devices);

	return ret;
}</code></pre>

<h3 id="2-2-binder-open"><a href="#2-2-binder-open" class="headerlink" title="2.2 binder_open"></a>2.2 binder_open</h3><p>[-&gt;android/binder.c]</p>

<pre><code>
static int binder_open(struct inode *nodp, struct file *filp)
{
    //binder进程
	struct binder_proc *proc;
	//为binder_proc结构体分配kernel内存空间
	struct binder_device *binder_dev;

	binder_debug(BINDER_DEBUG_OPEN_CLOSE, "binder_open: %d:%d/n",
		     current-&gt;group_leader-&gt;pid, current-&gt;pid);

	proc = kzalloc(sizeof(*proc), GFP_KERNEL);
	if (proc == NULL)
		return -ENOMEM;
	spin_lock_init(&proc-&gt;inner_lock);
	spin_lock_init(&proc-&gt;outer_lock);
	get_task_struct(current-&gt;group_leader);
	//将当前线程的task保存到binder进程的tsk
	proc-&gt;tsk = current-&gt;group_leader;
	mutex_init(&proc-&gt;files_lock);
	//初始化todo队列
	INIT_LIST_HEAD(&proc-&gt;todo);
	//初始化进程优先级
	if (binder_supported_policy(current-&gt;policy)) {
		proc-&gt;default_priority.sched_policy = current-&gt;policy;
		proc-&gt;default_priority.prio = current-&gt;normal_prio;
	} else {
		proc-&gt;default_priority.sched_policy = SCHED_NORMAL;
		proc-&gt;default_priority.prio = NICE_TO_PRIO(0);
	}
    //binder_dev初始化
	binder_dev = container_of(filp-&gt;private_data, struct binder_device,
				  miscdev);
    //初始化context
	proc-&gt;context = &binder_dev-&gt;context;
	
    //binder_alloc初始化
	binder_alloc_init(&proc-&gt;alloc);
	//BINDER_PROC对象创建数加1
	binder_stats_created(BINDER_STAT_PROC);
	proc-&gt;pid = current-&gt;group_leader-&gt;pid;
	INIT_LIST_HEAD(&proc-&gt;delivered_death);
	INIT_LIST_HEAD(&proc-&gt;waiting_threads);
	//file文件指针的private_data变量指向binder_proc数据
	filp-&gt;private_data = proc;
   
	mutex_lock(&binder_procs_lock);
	//proc节点添加到头部
	hlist_add_head(&proc-&gt;proc_node, &binder_procs);
	mutex_unlock(&binder_procs_lock);

	if (binder_debugfs_dir_entry_proc) {
		char strbuf[11];

		snprintf(strbuf, sizeof(strbuf), "%u", proc-&gt;pid);
		/*
		 * proc debug entries are shared between contexts, so
		 * this will fail if the process tries to open the driver
		 * again with a different context. The priting code will
		 * anyway print all contexts that a given PID has, so this
		 * is not a problem.
		 */
		proc-&gt;debugfs_entry = debugfs_create_file(strbuf, S_IRUGO,
			binder_debugfs_dir_entry_proc,
			(void *)(unsigned long)proc-&gt;pid,
			&binder_proc_fops);
	}

	return 0;
}</code></pre>

<p>创建binder_proc对象，并把当前进程等信息保存到binder_proc对象，该对象管理IPC所需要的各种信息并拥有其他结构体的根结构体；再把binder_proc对象保存到文件指针filp中,并且把binder_proc加入到全局链表binder_procs。</p>
<p>Binder驱动中 HLIST_HEAD创建全局的哈希链表binder_procs，用于保存所有的binder_proc队列,每次新创建的binder_proc都会加入到binder_procs链表中。</p>
<h3 id="2-3-binder-mmap"><a href="#2-3-binder-mmap" class="headerlink" title="2.3 binder_mmap"></a>2.3 binder_mmap</h3><p>[-&gt;android/binder.c]</p>

<pre><code>
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
	int ret;
	struct binder_proc *proc = filp-&gt;private_data;
	const char *failure_string;

	if (proc-&gt;tsk != current-&gt;group_leader)
		return -EINVAL;

	if ((vma-&gt;vm_end - vma-&gt;vm_start) &gt; SZ_4M)
		vma-&gt;vm_end = vma-&gt;vm_start + SZ_4M;  //保证映射内存大小不超过4M

	binder_debug(BINDER_DEBUG_OPEN_CLOSE,
		     "%s: %d %lx-%lx (%ld K) vma %lx pagep %lx/n",
		     __func__, proc-&gt;pid, vma-&gt;vm_start, vma-&gt;vm_end,
		     (vma-&gt;vm_end - vma-&gt;vm_start) / SZ_1K, vma-&gt;vm_flags,
		     (unsigned long)pgprot_val(vma-&gt;vm_page_prot));

	if (vma-&gt;vm_flags & FORBIDDEN_MMAP_FLAGS) {
		ret = -EPERM;
		failure_string = "bad vm_flags";
		goto err_bad_arg;
	}
	vma-&gt;vm_flags = (vma-&gt;vm_flags | VM_DONTCOPY) & ~VM_MAYWRITE;
	vma-&gt;vm_ops = &binder_vm_ops;
	vma-&gt;vm_private_data = proc;

	ret = binder_alloc_mmap_handler(&proc-&gt;alloc, vma);
	if (ret)
		return ret;
	mutex_lock(&proc-&gt;files_lock);
	proc-&gt;files = get_files_struct(current);
	mutex_unlock(&proc-&gt;files_lock);
	return 0;

err_bad_arg:
	pr_err("binder_mmap: %d %lx-%lx %s failed %d/n",
	       proc-&gt;pid, vma-&gt;vm_start, vma-&gt;vm_end, failure_string, ret);
	return ret;
}</code></pre>

<p>mmap的主要功能：首先在内核虚拟地址空间，申请一块与用户虚拟空间相同大小的内存，然后再申请一个page大小的物理内存，再将同一块物理内存分别映射到内核虚拟地址空间和用户虚拟空间，从而实现了用户空间的buffer和内核空间的buffer同步操作的功能。</p>
<h4 id="2-3-1-binder-alloc-mmap-handler"><a href="#2-3-1-binder-alloc-mmap-handler" class="headerlink" title="2.3.1  binder_alloc_mmap_handler"></a>2.3.1  binder_alloc_mmap_handler</h4><p>[-&gt;android/binder.c]</p>

<pre><code>
int binder_alloc_mmap_handler(struct binder_alloc *alloc,
			      struct vm_area_struct *vma)
{
	int ret;
	struct vm_struct *area;
	const char *failure_string;
	struct binder_buffer *buffer;
    //加锁
	mutex_lock(&binder_alloc_mmap_lock);
	if (alloc-&gt;buffer) {
		ret = -EBUSY;
		failure_string = "already mapped";
		goto err_already_mapped;
	}
    //采用IOREMAP方式，分配一个连续的内核虚拟空间，与进程虚拟空间大小一致
	area = get_vm_area(vma-&gt;vm_end - vma-&gt;vm_start, VM_IOREMAP);
	if (area == NULL) {
		ret = -ENOMEM;
		failure_string = "get_vm_area";
		goto err_get_vm_area_failed;
	}
	//指向内核虚拟空间地址
	alloc-&gt;buffer = area-&gt;addr;
	//地址偏移量 = 用户虚拟地址空间-内核虚拟地址空间
	alloc-&gt;user_buffer_offset =
		vma-&gt;vm_start - (uintptr_t)alloc-&gt;buffer;
    //释放锁
	mutex_unlock(&binder_alloc_mmap_lock);

#ifdef CONFIG_CPU_CACHE_VIPT
	if (cache_is_vipt_aliasing()) {
		while (CACHE_COLOUR(
				(vma-&gt;vm_start ^ (uint32_t)alloc-&gt;buffer))) {
			pr_info("%s: %d %lx-%lx maps %pK bad alignment\n",
				__func__, alloc-&gt;pid, vma-&gt;vm_start,
				vma-&gt;vm_end, alloc-&gt;buffer);
			vma-&gt;vm_start += PAGE_SIZE;
		}
	}
#endif
    //分配物理页的指针数组，数组大小为vma的等效page个数
	alloc-&gt;pages = kzalloc(sizeof(alloc-&gt;pages[0]) *
				   ((vma-&gt;vm_end - vma-&gt;vm_start) / PAGE_SIZE),
			       GFP_KERNEL);
	if (alloc-&gt;pages == NULL) {
		ret = -ENOMEM;
		failure_string = "alloc page array";
		goto err_alloc_pages_failed;
	}
	alloc-&gt;buffer_size = vma-&gt;vm_end - vma-&gt;vm_start;
    
    //为buffer分配物理内存
	buffer = kzalloc(sizeof(*buffer), GFP_KERNEL);
	if (!buffer) {
		ret = -ENOMEM;
		failure_string = "alloc buffer struct";
		goto err_alloc_buf_struct_failed;
	}
    //物理内存binder-&gt;data指向虚拟内存alloc-&gt;buffer
	buffer-&gt;data = alloc-&gt;buffer;
	//将binder_buffer地址，加入到所属进程的buffers队列
	list_add(&buffer-&gt;entry, &alloc-&gt;buffers);
	buffer-&gt;free = 1;
	//将空间buffer放入proc-&gt;free_buffers中
	binder_insert_free_buffer(alloc, buffer);
	//异步可用空间大小为buffer总大小的一半
	alloc-&gt;free_async_space = alloc-&gt;buffer_size / 2;
	barrier();
	alloc-&gt;vma = vma;
	alloc-&gt;vma_vm_mm = vma-&gt;vm_mm;
	/* Same as mmgrab() in later kernel versions */
	atomic_inc(&alloc-&gt;vma_vm_mm-&gt;mm_count);

	return 0;

err_alloc_buf_struct_failed:
	kfree(alloc-&gt;pages);
	alloc-&gt;pages = NULL;
err_alloc_pages_failed:
	mutex_lock(&binder_alloc_mmap_lock);
	vfree(alloc-&gt;buffer);
	alloc-&gt;buffer = NULL;
err_get_vm_area_failed:
err_already_mapped:
	mutex_unlock(&binder_alloc_mmap_lock);
	pr_err("%s: %d %lx-%lx %s failed %d\n", __func__,
	       alloc-&gt;pid, vma-&gt;vm_start, vma-&gt;vm_end, failure_string, ret);
	return ret;
}</code></pre>

<p>binder_mmap通过加锁，保证一次只有一个进程分配内存，保证多进程间的并发访问。user_buffer_offset是虚拟进程地址与虚拟内核地址的差值，该值为负值。</p>
<h3 id="2-4-binder-ioctl"><a href="#2-4-binder-ioctl" class="headerlink" title="2.4 binder_ioctl"></a>2.4 binder_ioctl</h3><p>[-&gt;android/binder.c]</p>

<pre><code>
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	int ret;
	struct binder_proc *proc = filp-&gt;private_data;
	//binder线程
	struct binder_thread *thread;  
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;

	/*pr_info("binder_ioctl: %d:%d %x %lx/n",
			proc-&gt;pid, current-&gt;pid, cmd, arg);*/

	binder_selftest_alloc(&proc-&gt;alloc);

	trace_binder_ioctl(cmd, arg);
    //进入休眠状态，直到中断唤醒
	ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error &lt; 2);
	if (ret)
		goto err_unlocked;
    //获取binder_thead,见2.4.1小节
	thread = binder_get_thread(proc);
	if (thread == NULL) {
		ret = -ENOMEM;
		goto err;
	}

	switch (cmd) {
	//对binder进行读写操作
	case BINDER_WRITE_READ: 
	     //见2.4.2小节
		ret = binder_ioctl_write_read(filp, cmd, arg, thread);
		if (ret)
			goto err;
		break;
    //设置binder最大支持的线程数
	case BINDER_SET_MAX_THREADS: {
		int max_threads;

		if (copy_from_user(&max_threads, ubuf,
				   sizeof(max_threads))) {
			ret = -EINVAL;
			goto err;
		}
		binder_inner_proc_lock(proc);
		proc-&gt;max_threads = max_threads;
		binder_inner_proc_unlock(proc);
		break;
	}
	//成为binder的上下文管理者，也就是ServiceManager成为守护进程
	case BINDER_SET_CONTEXT_MGR:
		ret = binder_ioctl_set_ctx_mgr(filp);
		if (ret)
			goto err;
		break;
    //当binder线程退出，释放binder线程
	case BINDER_THREAD_EXIT:
		binder_debug(BINDER_DEBUG_THREADS, "%d:%d exit/n",
			     proc-&gt;pid, thread-&gt;pid);
		binder_thread_release(proc, thread);
		thread = NULL;
		break;
	//获取binder的版本号
	case BINDER_VERSION: {
		struct binder_version __user *ver = ubuf;

		if (size != sizeof(struct binder_version)) {
			ret = -EINVAL;
			goto err;
		}
		if (put_user(BINDER_CURRENT_PROTOCOL_VERSION,
			     &ver-&gt;protocol_version)) {
			ret = -EINVAL;
			goto err;
		}
		break;
	}
	//获取节点的debug信息
	case BINDER_GET_NODE_DEBUG_INFO: {
		struct binder_node_debug_info info;

		if (copy_from_user(&info, ubuf, sizeof(info))) {
			ret = -EFAULT;
			goto err;
		}

		ret = binder_ioctl_get_node_debug_info(proc, &info);
		if (ret &lt; 0)
			goto err;

		if (copy_to_user(ubuf, &info, sizeof(info))) {
			ret = -EFAULT;
			goto err;
		}
		break;
	}
	default:
		ret = -EINVAL;
		goto err;
	}
	ret = 0;
err:
	if (thread)
		thread-&gt;looper_need_return = false;
	wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error &lt; 2);
	if (ret && ret != -ERESTARTSYS)
		pr_info("%d:%d ioctl %x %lx returned %d/n", proc-&gt;pid, current-&gt;pid, cmd, arg, ret);
err_unlocked:
	trace_binder_ioctl_done(ret);
	return ret;
}</code></pre>

<h4 id="2-4-1-binder-get-thread"><a href="#2-4-1-binder-get-thread" class="headerlink" title="2.4.1 binder_get_thread"></a>2.4.1 binder_get_thread</h4><p>[-&gt;android/binder.c]</p>

<pre><code>
static struct binder_thread *binder_get_thread(struct binder_proc *proc)
{
	struct binder_thread *thread;
	struct binder_thread *new_thread;
   
	binder_inner_proc_lock(proc);
	//根据当前进行的pid，从binder_proc中查找对应的binder_thread
	thread = binder_get_thread_ilocked(proc, NULL);
	binder_inner_proc_unlock(proc);
	//如果不存在
	if (!thread) {
	    //新建binder_thread结构体
		new_thread = kzalloc(sizeof(*thread), GFP_KERNEL);
		if (new_thread == NULL)
			return NULL;
		binder_inner_proc_lock(proc);
		thread = binder_get_thread_ilocked(proc, new_thread);
		binder_inner_proc_unlock(proc);
		if (thread != new_thread)
			kfree(new_thread);
	}
	return thread;
}</code></pre>

<h4 id="2-4-2-binder-ioctl-write-read"><a href="#2-4-2-binder-ioctl-write-read" class="headerlink" title="2.4.2 binder_ioctl_write_read"></a>2.4.2 binder_ioctl_write_read</h4><p>[-&gt;android/binder.c]</p>

<pre><code>
static int binder_ioctl_write_read(struct file *filp,
				unsigned int cmd, unsigned long arg,
				struct binder_thread *thread)
{
	int ret = 0;
	struct binder_proc *proc = filp-&gt;private_data;
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;
	struct binder_write_read bwr;

	if (size != sizeof(struct binder_write_read)) {
		ret = -EINVAL;
		goto out;
	}
	//把用户空间的数据ubuf写到bwr
	if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
		ret = -EFAULT;
		goto out;
	}
	binder_debug(BINDER_DEBUG_READ_WRITE,
		     "%d:%d write %lld at %016llx, read %lld at %016llx\n",
		     proc-&gt;pid, thread-&gt;pid,
		     (u64)bwr.write_size, (u64)bwr.write_buffer,
		     (u64)bwr.read_size, (u64)bwr.read_buffer);

	if (bwr.write_size &gt; 0) {
	    //当写缓存中有数据，则执行binder写操作
		ret = binder_thread_write(proc, thread,
					  bwr.write_buffer,
					  bwr.write_size,
					  &bwr.write_consumed);
		trace_binder_write_done(ret);
		//当写失败，在将bwr数据写回到用户空间并返回
		if (ret &lt; 0) {
			bwr.read_consumed = 0;
			if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
				ret = -EFAULT;
			goto out;
		}
	}
	if (bwr.read_size &gt; 0) {
	    //当读缓存中有数据，则执行binder读操作
		ret = binder_thread_read(proc, thread, bwr.read_buffer,
					 bwr.read_size,
					 &bwr.read_consumed,
					 filp-&gt;f_flags & O_NONBLOCK);
		trace_binder_read_done(ret);
		binder_inner_proc_lock(proc);
		if (!binder_worklist_empty_ilocked(&proc-&gt;todo))
		    //唤醒等待状态的线程
			binder_wakeup_proc_ilocked(proc);
		binder_inner_proc_unlock(proc);
		//当读失败，再将bwr数据写回到用户空间并返回
		if (ret &lt; 0) {
			if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
				ret = -EFAULT;
			goto out;
		}
	}
	binder_debug(BINDER_DEBUG_READ_WRITE,
		     "%d:%d wrote %lld of %lld, read return %lld of %lld\n",
		     proc-&gt;pid, thread-&gt;pid,
		     (u64)bwr.write_consumed, (u64)bwr.write_size,
		     (u64)bwr.read_consumed, (u64)bwr.read_size);
	//将内核数据bwr拷贝到用户空间ubuf	     
	if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
		ret = -EFAULT;
		goto out;
	}
out:
	return ret;
}</code></pre>

<p>binder_ioctl_write_read主要流程如下：</p>
<p>1.把用户空间ubuf拷贝到内核空间bwr；</p>
<p>2.当bwr写缓存有数据，则执行binder_thread_write；当写失败时则将bwr数据写回到用户空间并退出；</p>
<p>3.当bwr读缓存有数据，则执行binder_thread_read；当读失败时则将bwr数据写回到用户空间并退出；</p>
<p>4.把内核数据bwr拷贝到用户空间ubuf。</p>
<p>binder_thread_write、binder_thread_read作为两个核心方法，在第三节中详细介绍。</p>
<h3 id="2-5-小结"><a href="#2-5-小结" class="headerlink" title="2.5 小结"></a>2.5 小结</h3><p>binder_init：初始化字符设备，注册misc设备；</p>
<p>binder_open：打开驱动设备，初始化binder_proc；</p>
<p>binder_mmap：申请内核空间，将用户空间和内核空间映射到同一块物理内存；</p>
<p>binder_ioctl：执行相应的ioctl操作，主要进行读写操作。</p>
<p>下面看下binder通信的具体协议。</p>
<h2 id="三、Binder通信协议"><a href="#三、Binder通信协议" class="headerlink" title="三、Binder通信协议"></a>三、Binder通信协议</h2><h3 id="3-1-通信模型"><a href="#3-1-通信模型" class="headerlink" title="3.1 通信模型"></a>3.1 通信模型</h3><p>一次完整的Binder通信过程如下(非oneway)：</p>
<p><img src="https://img-blog.csdnimg.cn/b9d681f869ca4b0e84b1fe14aa9e8a0c.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_18,color_FFFFFF,t_70,g_se,x_16" alt="bindermodel" style="zoom:80%;"></p>
<p>Binder协议包含在IPC数据中，分为两类：</p>
<p>1.BINDER_COMMAND_PROTOCOL:binder请求码，以BC_开头，简称BC码，用于从IPC层传递到Binder Driver层;</p>
<p>2.BINDER_RETURN_PROTOCOL:binder响应码，以BR_开头，简称BR码，用于从Binder Driver层传递到IPC层。</p>
<p>Binder IPC通信至少是两个进程的交互</p>
<ul>
<li><p>client进程执行binder_thread_write，根据BC_xx命令，生成相应的binder_work;</p>
</li>
<li><p>server进程执行binder_thread_read，根据binder_work_type类型，生成BR_xx，发送到用户空间处理。</p>
</li>
</ul>
<h3 id="3-2-binder-thread-write"><a href="#3-2-binder-thread-write" class="headerlink" title="3.2 binder_thread_write"></a>3.2 binder_thread_write</h3><p>请求过程是通过binder_thread_write方法，该方法用于处理Binder协议中的请求码，当binder_buffer存在数据，binder线程的写操作循环执行。</p>

<pre><code>
static int binder_thread_write(struct binder_proc *proc,
			struct binder_thread *thread,
			binder_uintptr_t binder_buffer, size_t size,
			binder_size_t *consumed)
{
	uint32_t cmd;
	struct binder_context *context = proc-&gt;context;
	void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
	void __user *ptr = buffer + *consumed;
	void __user *end = buffer + size;

	while (ptr &lt; end && thread-&gt;return_error.cmd == BR_OK) {
		int ret;
         //获取IPC数据中的Binder协议
		if (get_user(cmd, (uint32_t __user *)ptr))
			return -EFAULT;
		ptr += sizeof(uint32_t);
		trace_binder_command(cmd);
		if (_IOC_NR(cmd) &lt; ARRAY_SIZE(binder_stats.bc)) {
			atomic_inc(&binder_stats.bc[_IOC_NR(cmd)]);
			atomic_inc(&proc-&gt;stats.bc[_IOC_NR(cmd)]);
			atomic_inc(&thread-&gt;stats.bc[_IOC_NR(cmd)]);
		}
		switch (cmd) {
		case BC_INCREFS:
		case BC_ACQUIRE:
		case BC_RELEASE:
		case BC_DECREFS: ...
		case BC_INCREFS_DONE:
		case BC_ACQUIRE_DONE:...
		case BC_ATTEMPT_ACQUIRE:
		case BC_ACQUIRE_RESULT:...
		case BC_FREE_BUFFER:...
		case BC_TRANSACTION_SG:
		case BC_REPLY_SG: ...	
		case BC_TRANSACTION:
		case BC_REPLY: {
			struct binder_transaction_data tr;
			//拷贝用户空间tr到内核
			if (copy_from_user(&tr, ptr, sizeof(tr)))
				return -EFAULT;
			ptr += sizeof(tr);
			//见3.2.1小节
			binder_transaction(proc, thread, &tr,
					   cmd == BC_REPLY, 0);
			break;
		}
		case BC_REGISTER_LOOPER:...
		case BC_ENTER_LOOPER:...
		case BC_EXIT_LOOPER:...	
		case BC_REQUEST_DEATH_NOTIFICATION:
		case BC_CLEAR_DEATH_NOTIFICATION: ...
		case BC_DEAD_BINDER_DONE:...
		default:...		
	}
	return 0;
}</code></pre>

<p>对于BC_TRANSACTION、BC_REPLY的请求码，回执行binder_transaction方法，这是最为频繁的操作，对于其他命令则不同。</p>
<h4 id="3-2-1-binder-transaction"><a href="#3-2-1-binder-transaction" class="headerlink" title="3.2.1 binder_transaction"></a>3.2.1 binder_transaction</h4>
<pre><code>
static void binder_transaction(struct binder_proc *proc,
			       struct binder_thread *thread,
			       struct binder_transaction_data *tr, int reply,
			       binder_size_t extra_buffers_size)
{
	int ret;
	//传进来的binder_transaction
	struct binder_transaction *t;
	struct binder_work *tcomplete;
	binder_size_t *offp, *off_end, *off_start;
	binder_size_t off_min;
	u8 *sg_bufp, *sg_buf_end;
	//根据各种判断获取一下信息：
	//目标进程
	struct binder_proc *target_proc = NULL;
	//目标线程
	struct binder_thread *target_thread = NULL;
	//目标binder结点
	struct binder_node *target_node = NULL;
	//回复的binder_transaction
	struct binder_transaction *in_reply_to = NULL;
	...
	/* TODO: reuse incoming transaction for reply */
	t = kzalloc(sizeof(*t), GFP_KERNEL);
	...
	//为target_proc分配一块buffer
	t-&gt;buffer = binder_alloc_new_buf(&target_proc-&gt;alloc, tr-&gt;data_size,
		tr-&gt;offsets_size, extra_buffers_size,
		!reply && (t-&gt;flags & TF_ONE_WAY));
	for (; offp &lt; off_end; offp++) {
		...
		hdr = (struct binder_object_header *)(t-&gt;buffer-&gt;data + *offp);
		off_min = *offp + object_size;
		switch (hdr-&gt;type) {
		case BINDER_TYPE_BINDER:
		case BINDER_TYPE_WEAK_BINDER: ...
		case BINDER_TYPE_HANDLE:
		case BINDER_TYPE_WEAK_HANDLE: ...
		case BINDER_TYPE_FD: ...
		case BINDER_TYPE_FDA: ...
		case BINDER_TYPE_PTR: ...
		default:
		}
	}
	//当前线程的type
	tcomplete-&gt;type = BINDER_WORK_TRANSACTION_COMPLETE;
	//向目标进程的work_type
	t-&gt;work.type = BINDER_WORK_TRANSACTION;
	//后面会将这些事务添加到相应的队列
	...	
}</code></pre>

<h4 id="3-2-2-BC-PROTOCOL"><a href="#3-2-2-BC-PROTOCOL" class="headerlink" title="3.2.2 BC_PROTOCOL"></a>3.2.2 BC_PROTOCOL</h4><p>Binder的请求码是在binder_driver_command_protocol中定义的，用于应用程序向binder驱动设备发送请求消息，应用程序包含Client和Server端，以BC_开头，总共19条。</p>
<table>
<thead>
<tr>
<th>请求码</th>
<th>参数类型</th>
<th>作用</th>
</tr>
</thead>
<tbody>
<tr>
<td>BC_TRANSACTION</td>
<td>binder_transaction_data</td>
<td>Client向Binder驱动发送的请求数据</td>
</tr>
<tr>
<td>BC_REPLY</td>
<td>binder_transaction_data</td>
<td>Server向Binder驱动发送的回复数据</td>
</tr>
<tr>
<td>BC_ACQUIRE_RESULT</td>
<td>__s32</td>
<td>暂时不支持</td>
</tr>
<tr>
<td>BC_FREE_BUFFER</td>
<td>binder_uintptr_t</td>
<td>释放内存</td>
</tr>
<tr>
<td>BC_INCREFS</td>
<td>__u32</td>
<td>binder_ref弱引用加1操作</td>
</tr>
<tr>
<td>BC_ACQUIRE</td>
<td>__u32</td>
<td>binder_ref弱引用减1操作</td>
</tr>
<tr>
<td>BC_RELEASE</td>
<td>__u32</td>
<td>binder_ref强引用加1操作</td>
</tr>
<tr>
<td>BC_DECREFS</td>
<td>__u32</td>
<td>binder_ref强引用减1操作</td>
</tr>
<tr>
<td>BC_INCREFS_DONE</td>
<td>binder_ptr_cookie</td>
<td>binder_node强引用减1操作</td>
</tr>
<tr>
<td>BC_ACQUIRE_DONE</td>
<td>binder_ptr_cookie</td>
<td>binder_node弱引用减1操作</td>
</tr>
<tr>
<td>BC_ATTEMPT_ACQUIRE</td>
<td>binder_pri_desc</td>
<td>暂时不支持</td>
</tr>
<tr>
<td>BC_REGISTER_LOOPER</td>
<td>无参数</td>
<td>创建新的Looper线程</td>
</tr>
<tr>
<td>BC_ENTER_LOOPER</td>
<td>无参数</td>
<td>应用线程进入Looper</td>
</tr>
<tr>
<td>BC_EXIT_LOOPER</td>
<td>无参数</td>
<td>应用线程退出Looper</td>
</tr>
<tr>
<td>BC_REQUEST_DEATH_NOTIFICATION</td>
<td>binder_handle_cookie</td>
<td>注册死亡通知</td>
</tr>
<tr>
<td>BC_CLEAR_DEATH_NOTIFICATION</td>
<td>binder_handle_cookie</td>
<td>取消注册的死亡通知</td>
</tr>
<tr>
<td>BC_DEAD_BINDER_DONE</td>
<td>binder_uintptr_t</td>
<td>已经完成的死亡通知</td>
</tr>
<tr>
<td>BC_TRANSACTION_SG</td>
<td>binder_transaction_data_sg</td>
<td>Client向Binder驱动发送的Command</td>
</tr>
<tr>
<td>BC_REPLY_SG</td>
<td>binder_transaction_data_sg</td>
<td>Server向Binder驱动发送的Command</td>
</tr>
</tbody>
</table>
<p><strong>BC_FREE_BUFFER</strong></p>
<ul>
<li><p>通过mmap映射内存，其中ServiceManager映射的空间大小为128K,其他Binder应用进程映射的内存大小为1M-8k；</p>
</li>
<li><p>Binder驱动基于这种映射的内存采用最佳匹配来动态分配和释放，通过bind_buffer结构体中的free字段来表示相应的buffer是空闲还是已分配状态。对于已分配的buffers加入到binder_proc中的allocated_buffers红黑树，对于空闲的buffer加入到free_buffers红黑树；</p>
</li>
<li><p>当应用程序需要内存时，根据所需内存大小从free_buffers中找到最合适的内存，并放入allocated_buffers树中，当应用程序处理完成后必须尽快使用BC_FREE_BUFFER命令来释放该buffer，从而添加回到free_buffers树。</p>
</li>
</ul>
<p><strong>BC_INCREFS、BC_ACQUIRE、BC_RELEASE、BC_DECREFS</strong>等请求码的作用是对binder的强/弱引用的计数操作，用于实现强/弱指针的功能；</p>
<p><strong>BC_REGISTER_LOOPER</strong>：Binder用于驱动层决策而创建新的binder线程，joniThreadPool过程，创建非binder主线程；</p>
<p><strong>BC_ENTER_LOOPER</strong>：binder主线程（由应用层发起）的创建会向驱动发送该消息；joniThreadPool过程,创建binder主线程；</p>
<p><strong>BC_EXIT_LOOPER</strong>：退出binder线程，对于binder主线程是不能退出；jointThreadPool的过程出现timeout且是非binder主线程，则会退出该binder线程。</p>
<h3 id="3-3-binder-thread-read"><a href="#3-3-binder-thread-read" class="headerlink" title="3.3 binder_thread_read"></a>3.3 binder_thread_read</h3><p>响应处理的过程是通过binder_thread_read方法，该方法根据不同的binder_work-&gt;type已经不同的状态，生成相应的响应码。</p>

<pre><code>
static int binder_thread_read(struct binder_proc *proc,
			      struct binder_thread *thread,
			      binder_uintptr_t binder_buffer, size_t size,
			      binder_size_t *consumed, int non_block)
{
	
	wait_for_proc_work = binder_available_for_proc_work_ilocked(thread);
	//根据wait_for_proc_work来决定wait在当前线程还是进程的等待队列
	if (wait_for_proc_work) {
	  ...
	}
    ...
	while (1) {
	     //当thread-&gt;todo和proc-&gt;todo为空时，goto到retry标志处，否则往下执行
	     if (!binder_worklist_empty_ilocked(&thread-&gt;todo))
			list = &thread-&gt;todo;
		 else if (!binder_worklist_empty_ilocked(&proc-&gt;todo) &&
			   wait_for_proc_work)
			list = &proc-&gt;todo;
		else {
			binder_inner_proc_unlock(proc);
			/* no data added */
			if (ptr - buffer == 4 && !thread-&gt;looper_need_return)
				goto retry;
			break;
		}
		...
		switch (w-&gt;type) {
		case BINDER_WORK_TRANSACTION: 
		case BINDER_WORK_RETURN_ERROR: ...
		case BINDER_WORK_TRANSACTION_COMPLETE: ...
		case BINDER_WORK_NODE: ...
		case BINDER_WORK_DEAD_BINDER:
		case BINDER_WORK_DEAD_BINDER_AND_CLEAR:
		case BINDER_WORK_CLEAR_DEATH_NOTIFICATION: ...
	}

done:
	*consumed = ptr - buffer;
	binder_inner_proc_lock(proc);
	//当满足请求线程已准备线程数等于0，已启动线程数小于最大线程数15
	//且looper状态为已经注册或者已进入时创建新的线程
	if (proc-&gt;requested_threads == 0 &&
	    list_empty(&thread-&gt;proc-&gt;waiting_threads) &&
	    proc-&gt;requested_threads_started &lt; proc-&gt;max_threads &&
	    (thread-&gt;looper & (BINDER_LOOPER_STATE_REGISTERED |
	     BINDER_LOOPER_STATE_ENTERED)) /* the user-space code fails to */
	     /*spawn a new thread if we leave this out */) {
		proc-&gt;requested_threads++;
		binder_inner_proc_unlock(proc);
		binder_debug(BINDER_DEBUG_THREADS,
			     "%d:%d BR_SPAWN_LOOPER\n",
			     proc-&gt;pid, thread-&gt;pid);
	    //生成BR_SPAWN_LOOPER命令，用于创建新的线程
		if (put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer))
			return -EFAULT;
		binder_stat_br(proc, thread, BR_SPAWN_LOOPER);
	} else
		binder_inner_proc_unlock(proc);
	return 0;
}</code></pre>

<p>当transaction堆栈为空且线程todo链表为空，且non_block=false时，则意味着没有任何事物需要处理，会进入等待客户端请求的状态。当有事务需要处理时便会进入循环处理过程，并生成相应的响应码。在Binder驱动层，只有在进入binder_thread_read方法时，同时满足以下条件才会生成BR_SPAWN_LOOPER命令，当用户态进程收到该命令则会创建新的线程。</p>
<p>1.binder_proc的requested_threads线程数等于0；</p>
<p>2.binder_proc的waiting_threads的列表为空；</p>
<p>3.binder_proc的requested_threads_started个数小于15即最大线程个数；</p>
<p>4.binder_thead的looper状态为BINDER_LOOPER_STATE_REGISTERED或BINDER_LOOPER_STATE_ENTERED。</p>
<p>响应码的处理（文章注册Servicemanager中），在用户空间的 IPCThreadState类中的 IPCThreadState::waitForResponse和 IPCThreadState::executeCommand两个方法共同处理Binder协议中的响应码。</p>
<h4 id="3-3-1-BR-PROTOCOL"><a href="#3-3-1-BR-PROTOCOL" class="headerlink" title="3.3.1 BR_PROTOCOL"></a>3.3.1 BR_PROTOCOL</h4><p>Binder响应码，在binder_driver_return_protocol中定义，是binder设备向应用程序回复的消息，应用程序包括client和server端，以BR_开头，总共18条。</p>
<table>
<thead>
<tr>
<th>响应码</th>
<th>参数类型</th>
<th>作用</th>
</tr>
</thead>
<tbody>
<tr>
<td>BR_ERROR</td>
<td>__s32</td>
<td>操作发送错误</td>
</tr>
<tr>
<td>BR_OK</td>
<td>无参数</td>
<td>操作完成</td>
</tr>
<tr>
<td>BR_TRANSACTION</td>
<td>binder_transaction_data</td>
<td>Binder驱动向Server发送的请求数据</td>
</tr>
<tr>
<td>BR_REPLY</td>
<td>binder_transaction_data</td>
<td>Binder驱动向Client发送的回复数据</td>
</tr>
<tr>
<td>BR_ACQUIRE_RESULT</td>
<td>__s32</td>
<td>暂时不支持</td>
</tr>
<tr>
<td>BR_DEAD_REPLY</td>
<td>无参数</td>
<td>回复失败，线程或节点为空</td>
</tr>
<tr>
<td>BR_TRANSACTION_COMPLETE</td>
<td>无参数</td>
<td>对请求发送的成功反馈</td>
</tr>
<tr>
<td>BR_INCREFS</td>
<td>binder_ptr_cookie</td>
<td>binder_ref弱引用加1操作</td>
</tr>
<tr>
<td>BR_ACQUIRE</td>
<td>binder_ptr_cookie</td>
<td>binder_ref弱引用减1操作</td>
</tr>
<tr>
<td>BR_RELEASE</td>
<td>binder_ptr_cookie</td>
<td>binder_ref强引用加1操作</td>
</tr>
<tr>
<td>BR_DECREFS</td>
<td>binder_ptr_cookie</td>
<td>binder_ref强引用减1操作</td>
</tr>
<tr>
<td>BR_ATTEMPT_ACQUIRE</td>
<td>binder_pri_ptr_cookie</td>
<td>暂时不支持</td>
</tr>
<tr>
<td>BR_NOOP</td>
<td>无参数</td>
<td>不做任何事情</td>
</tr>
<tr>
<td>BR_SPAWN_LOOPER</td>
<td>无参数</td>
<td>创建新的Looper线程</td>
</tr>
<tr>
<td>BR_FINISHED</td>
<td>无参数</td>
<td>暂时不支持</td>
</tr>
<tr>
<td>BR_DEAD_BINDER</td>
<td>binder_uintptr_t</td>
<td>Binder驱动向client发送死亡通知</td>
</tr>
<tr>
<td>BR_CLEAR_DEATH_NOTIFICATION_DONE</td>
<td>binder_uintptr_t</td>
<td>清除死亡通知</td>
</tr>
<tr>
<td>BR_FAILED_REPLY</td>
<td>无参数</td>
<td>回复失败，transaction出错导致</td>
</tr>
</tbody>
</table>
<p><strong>BR_SPAWN_LOOPER</strong>：binder驱动已经检测到进程中没有线程等待即将到来的事务，那么当一个进程接受到这条命令时，该进程必须创建一个新的服务线程并注册该线程，在接下来的响应过程会看到何时生成该响应码。</p>
<p><strong>BR_TRANSACTION_COMPLETE</strong>：当Client端向Binder驱动发送BC_TRANSACTION命令后，Client会收到BR_TRANSACTION_COMPLETE命令，告知Client端请求命令发送成功；对于Server向Binder驱动发送BC_REPLY命令后，server端会收到BR_TRANSACTION_COMPLETE命令，告知Server端请求回应命令发送成功。</p>
<p><strong>BR_DEAD_REPLY</strong>：当应用层向Binder驱动发送Binder调用时，若Binder应用层的另一个端已经死亡，则驱动回应BR_DEAD_REPLY命令。</p>
<p><strong>BR_FAILED_REPLY</strong>：当应用层向Binder驱动发送Binder调用时，若transaction出错，比如调用的函数号不存在，则驱动回应BR_FAILED_REPLY。</p>
<h3 id="3-4-协议使用场景"><a href="#3-4-协议使用场景" class="headerlink" title="3.4 协议使用场景"></a>3.4 协议使用场景</h3><h4 id="3-4-1-BC协议"><a href="#3-4-1-BC协议" class="headerlink" title="3.4.1 BC协议"></a>3.4.1 BC协议</h4><table>
<thead>
<tr>
<th style="text-align:left">BC协议</th>
<th style="text-align:left">调用方法</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left">BC_TRANSACTION</td>
<td style="text-align:left">IPC.transact()</td>
</tr>
<tr>
<td style="text-align:left">BC_REPLY</td>
<td style="text-align:left">IPC.sendReply()</td>
</tr>
<tr>
<td style="text-align:left">BC_FREE_BUFFER</td>
<td style="text-align:left">IPC.freeBuffer()</td>
</tr>
<tr>
<td style="text-align:left">BC_REQUEST_DEATH_NOTIFICATION</td>
<td style="text-align:left">IPC.requestDeathNotification()</td>
</tr>
<tr>
<td style="text-align:left">BC_CLEAR_DEATH_NOTIFICATION</td>
<td style="text-align:left">IPC.clearDeathNotification()</td>
</tr>
<tr>
<td style="text-align:left">BC_DEAD_BINDER_DONE</td>
<td style="text-align:left">IPC.execute()</td>
</tr>
</tbody>
</table>
<p>binder_thread_write()根据不同的BC协议而执行不同的流程。 其中BC_TRANSACTION和BC_REPLY协议，会进入binder_transaction()过程。</p>
<h4 id="3-4-2-BR协议"><a href="#3-4-2-BR协议" class="headerlink" title="3.4.2 BR协议"></a>3.4.2 BR协议</h4><table>
<thead>
<tr>
<th style="text-align:left">BR协议</th>
<th style="text-align:left">触发时机</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left">BR_TRANSACTION</td>
<td style="text-align:left">收到BINDER_WORK_TRANSACTION</td>
</tr>
<tr>
<td style="text-align:left">BR_REPLY</td>
<td style="text-align:left">收到BINDER_WORK_TRANSACTION</td>
</tr>
<tr>
<td style="text-align:left">BR_TRANSACTION_COMPLETE</td>
<td style="text-align:left">收到BINDER_WORK_TRANSACTION_COMPLETE</td>
</tr>
<tr>
<td style="text-align:left">BR_DEAD_BINDER</td>
<td style="text-align:left">收到BINDER_WORK_DEAD_BINDER或BINDER_WORK_DEAD_BINDER_AND_CLEAR</td>
</tr>
<tr>
<td style="text-align:left">BR_CLEAR_DEATH_NOTIFICATION_DONE</td>
<td style="text-align:left">收到BINDER_WORK_CLEAR_DEATH_NOTIFICATION</td>
</tr>
</tbody>
</table>
<p>BR_DEAD_REPLY，BR_FAILED_REPLY，BR_ERROR这些都是失败或错误相关的应答协议。</p>
<h4 id="3-4-3-协议转换图"><a href="#3-4-3-协议转换图" class="headerlink" title="3.4.3 协议转换图"></a>3.4.3 协议转换图</h4><p>以BC_TRANSACTION为例，说明协议转换过程。</p>
<ul>
<li>发起端进程：binder_transaction()过程将BC_TRANSACTION转换为BW_TRANSACTION；</li>
<li>接收端进程：binder_thread_read()过程，将BW_TRANSACTION转换为BR_TRANSACTION;</li>
<li>接收端进程：IPC.execute()过程，处理BR_TRANSACTION命令。</li>
</ul>
<p>注：BINDER_WORK_xxx –&gt; BW_xxx</p>
<p><img src="https://img-blog.csdnimg.cn/19751b0cbbc54c82b43f19df1f91085e.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_17,color_FFFFFF,t_70,g_se,x_16" alt="binder_procol1" style="zoom:80%;"></p>
<p><img src="https://img-blog.csdnimg.cn/b778aabc8b4542f196752fa2631baab9.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_18,color_FFFFFF,t_70,g_se,x_16" alt="binder_procol2" style="zoom:80%;"></p>
<h2 id="四、Binder内存机制"><a href="#四、Binder内存机制" class="headerlink" title="四、Binder内存机制"></a>四、Binder内存机制</h2><p>binder_mmap是Binder进程间通信的高效的核心机制所在，其模型如下：</p>
<p><img src="https://img-blog.csdnimg.cn/5ac956519c494c2a9468103f92f37567.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_16,color_FFFFFF,t_70,g_se,x_16" alt="内存模型" style="zoom:80%;"></p>
<p>虚拟进程地址空间（vm_area_struct）和虚拟内核地址空间（vm_struct）都映射到同一块物理内存空间。当client端与server端发送数据时，client作为数据发送端，先从自己的进程空间把IPC通信数据copy_from_user拷贝到内核空间，而server端作为数据接收端，与内核共享数据，不再需要拷贝数据，而是通过内存地址空间的偏移量获取内存地址，整个过程只发生一次内存拷贝。一般的做法，需要Client端进程空间拷贝到内核空间，再由内核空间拷贝到server进程空间，会发生两次拷贝。</p>
<p><img src="https://img-blog.csdnimg.cn/e880c1b8f618434aad1bec02ea09004a.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_17,color_FFFFFF,t_70,g_se,x_16" alt="binder内存转换" style="zoom:80%;"></p>
<p>对于进程和内核虚拟地址映射到同一个物理内存的操作（通过地址偏移量来实现）是发生在数据接收端，而数据发送端还是需要将用户态的数据复制到内核态。为什么不直接让发送端和接收端直接映射到同一块物理空间，那样连一次复制的操作都不需要，0次复制那就和Linux标准内核的共享内存IPC没有区别了，对于共享内存虽然效率高，但是对于多进程同步的问题比较复杂，而管道/消息队列等IPC需要复制两次，效率较低。总之Android选择Binder是基于速度和安全性的考虑。</p>
<h2 id="附录"><a href="#附录" class="headerlink" title="附录"></a>附录</h2><p>下面列举Binder驱动相关的一些重要结构体</p>
<h3 id="结构体列表"><a href="#结构体列表" class="headerlink" title="结构体列表"></a>结构体列表</h3><table>
<thead>
<tr>
<th style="text-align:left">序号</th>
<th style="text-align:left">结构体</th>
<th style="text-align:left">名称</th>
<th style="text-align:left">解释</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left">1</td>
<td style="text-align:left">binder_proc</td>
<td style="text-align:left">binder进程</td>
<td style="text-align:left">每个进程调用open()打开binder驱动都会创建该结构体，用于管理IPC所需的各种信息</td>
</tr>
<tr>
<td style="text-align:left">2</td>
<td style="text-align:left">binder_thread</td>
<td style="text-align:left">binder线程</td>
<td style="text-align:left">对应于上层的binder线程</td>
</tr>
<tr>
<td style="text-align:left">3</td>
<td style="text-align:left">binder_node</td>
<td style="text-align:left">binder实体</td>
<td style="text-align:left">对应于BBinder对象，记录BBinder的进程、指针、引用计数等</td>
</tr>
<tr>
<td style="text-align:left">4</td>
<td style="text-align:left">binder_ref</td>
<td style="text-align:left">binder引用</td>
<td style="text-align:left">对应于BpBinder对象，记录BpBinder的引用计数、死亡通知、BBinder指针等</td>
</tr>
<tr>
<td style="text-align:left">5</td>
<td style="text-align:left">binder_ref_death</td>
<td style="text-align:left">binder死亡引用</td>
<td style="text-align:left">记录binder死亡的引用信息</td>
</tr>
<tr>
<td style="text-align:left">6</td>
<td style="text-align:left">binder_write_read</td>
<td style="text-align:left">binder读写</td>
<td style="text-align:left">记录buffer中读和写的数据信息</td>
</tr>
<tr>
<td style="text-align:left">7</td>
<td style="text-align:left">binder_transaction_data</td>
<td style="text-align:left">binder事务数据</td>
<td style="text-align:left">记录传输数据内容，比如发送方pid/uid，RPC数据</td>
</tr>
<tr>
<td style="text-align:left">8</td>
<td style="text-align:left">flat_binder_object</td>
<td style="text-align:left">binder扁平对象</td>
<td style="text-align:left">Binder对象在两个进程间传递的扁平结构</td>
</tr>
<tr>
<td style="text-align:left">9</td>
<td style="text-align:left">binder_buffer</td>
<td style="text-align:left">binder内存</td>
<td style="text-align:left">调用mmap()创建用于Binder传输数据的缓存区</td>
</tr>
<tr>
<td style="text-align:left">10</td>
<td style="text-align:left">binder_transaction</td>
<td style="text-align:left">binder事务</td>
<td style="text-align:left">记录传输事务的发送方和接收方线程、进程等</td>
</tr>
<tr>
<td style="text-align:left">11</td>
<td style="text-align:left">binder_work</td>
<td style="text-align:left">binder工作</td>
<td style="text-align:left">记录binder工作类型</td>
</tr>
<tr>
<td style="text-align:left">12</td>
<td style="text-align:left">binder_state</td>
<td style="text-align:left">binder状态</td>
</tr>
</tbody>
</table>
<p>6~9 用于数据传输相关，其中binder_write_read，binder_transaction_data进程空间和内核空间是通用的。</p>
<p><strong>BWR核心数据表</strong></p>
<p><img src="https://img-blog.csdnimg.cn/29a6137a6ad24b628884102954194612.png?x-oss-process=,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYW5kcm9pZEJleW9uZA==,size_17,color_FFFFFF,t_70,g_se,x_16" alt="bwr"></p>
<p>binder_write_read是整个Binder IPC过程，最为核心的数据结构之一。</p>
<h4 id="1-binder-proc"><a href="#1-binder-proc" class="headerlink" title="1.binder_proc"></a>1.binder_proc</h4><p>binder_proc结构体：用于管理IPC所需的各种信息，拥有其他结构体的结构体。</p>
<table>
<thead>
<tr>
<th style="text-align:left">类型</th>
<th style="text-align:left">成员变量</th>
<th style="text-align:left">解释</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left">struct hlist_node</td>
<td style="text-align:left">proc_node</td>
<td style="text-align:left">进程节点</td>
</tr>
<tr>
<td style="text-align:left">struct rb_root</td>
<td style="text-align:left">threads</td>
<td style="text-align:left">binder_thread红黑树的根节点</td>
</tr>
<tr>
<td style="text-align:left">struct rb_root</td>
<td style="text-align:left">nodes</td>
<td style="text-align:left">binder_node红黑树的根节点</td>
</tr>
<tr>
<td style="text-align:left">struct rb_root</td>
<td style="text-align:left">refs_by_desc</td>
<td style="text-align:left">binder_ref红黑树的根节点(以handle为key)</td>
</tr>
<tr>
<td style="text-align:left">struct rb_root</td>
<td style="text-align:left">refs_by_node</td>
<td style="text-align:left">binder_ref红黑树的根节点（以ptr为key）</td>
</tr>
<tr>
<td style="text-align:left">int</td>
<td style="text-align:left">pid</td>
<td style="text-align:left">相应进程id</td>
</tr>
<tr>
<td style="text-align:left">struct vm_area_struct *</td>
<td style="text-align:left">vma</td>
<td style="text-align:left">指向进程虚拟地址空间的指针</td>
</tr>
<tr>
<td style="text-align:left">struct mm_struct *</td>
<td style="text-align:left">vma_vm_mm</td>
<td style="text-align:left">相应进程的内存结构体</td>
</tr>
<tr>
<td style="text-align:left">struct task_struct *</td>
<td style="text-align:left">tsk</td>
<td style="text-align:left">相应进程的task结构体</td>
</tr>
<tr>
<td style="text-align:left">struct files_struct *</td>
<td style="text-align:left">files</td>
<td style="text-align:left">相应进程的文件结构体</td>
</tr>
<tr>
<td style="text-align:left">struct hlist_node</td>
<td style="text-align:left">deferred_work_node</td>
<td style="text-align:left"></td>
</tr>
<tr>
<td style="text-align:left">int</td>
<td style="text-align:left">deferred_work</td>
<td style="text-align:left"></td>
</tr>
<tr>
<td style="text-align:left">void *</td>
<td style="text-align:left">buffer</td>
<td style="text-align:left">内核空间的起始地址</td>
</tr>
<tr>
<td style="text-align:left">ptrdiff_t</td>
<td style="text-align:left">user_buffer_offset</td>
<td style="text-align:left">内核空间与用户空间的地址偏移量</td>
</tr>
<tr>
<td style="text-align:left">struct list_head</td>
<td style="text-align:left">buffers</td>
<td style="text-align:left">所有的buffer</td>
</tr>
<tr>
<td style="text-align:left">struct rb_root</td>
<td style="text-align:left">free_buffers</td>
<td style="text-align:left">空闲的buffer</td>
</tr>
<tr>
<td style="text-align:left">struct rb_root</td>
<td style="text-align:left">allocated_buffers</td>
<td style="text-align:left">已分配的buffer</td>
</tr>
<tr>
<td style="text-align:left">size_t</td>
<td style="text-align:left">free_async_space</td>
<td style="text-align:left">异步的可用空闲空间大小</td>
</tr>
<tr>
<td style="text-align:left">struct page **</td>
<td style="text-align:left">pages</td>
<td style="text-align:left">指向物理内存页指针的指针</td>
</tr>
<tr>
<td style="text-align:left">size_t</td>
<td style="text-align:left">buffer_size</td>
<td style="text-align:left">映射的内核空间大小</td>
</tr>
<tr>
<td style="text-align:left">uint32_t</td>
<td style="text-align:left">buffer_free</td>
<td style="text-align:left">可用内存总大小</td>
</tr>
<tr>
<td style="text-align:left">struct list_head</td>
<td style="text-align:left">todo</td>
<td style="text-align:left">进程将要做的事</td>
</tr>
<tr>
<td style="text-align:left">wait_queue_head_t</td>
<td style="text-align:left">wait</td>
<td style="text-align:left">等待队列</td>
</tr>
<tr>
<td style="text-align:left">struct binder_stats</td>
<td style="text-align:left">stats</td>
<td style="text-align:left">binder统计信息</td>
</tr>
<tr>
<td style="text-align:left">struct list_head</td>
<td style="text-align:left">delivered_death</td>
<td style="text-align:left">已分发的死亡通知</td>
</tr>
<tr>
<td style="text-align:left">int</td>
<td style="text-align:left">max_threads</td>
<td style="text-align:left">最大线程数</td>
</tr>
<tr>
<td style="text-align:left">int</td>
<td style="text-align:left">requested_threads</td>
<td style="text-align:left">请求的线程数</td>
</tr>
<tr>
<td style="text-align:left">int</td>
<td style="text-align:left">requested_threads_started</td>
<td style="text-align:left">已启动的请求线程数</td>
</tr>
<tr>
<td style="text-align:left">int</td>
<td style="text-align:left">ready_threads</td>
<td style="text-align:left">准备就绪的线程个数</td>
</tr>
<tr>
<td style="text-align:left">long</td>
<td style="text-align:left">default_priority</td>
<td style="text-align:left">默认优先级</td>
</tr>
<tr>
<td style="text-align:left">struct dentry *</td>
<td style="text-align:left">debugfs_entry</td>
</tr>
</tbody>
</table>
<ul>
<li>free_buffers：记录所有空闲的buffer，记录以buffer_size为key的binder_buffer的红黑树结构</li>
<li>allocated_buffers:记录所有已分配的buffer，记录以buffer_size为key的binder_buffer的红黑树结构</li>
<li>buffers: 所有buffer（包含空闲的和已分配的buffer）的按地址由从低到高都连入到buffers链表中</li>
<li>ready_threads: 准备就绪的线程个数，往往是指进入binder_thread_read()，处于休眠等待状态的线程个数；ready_threads线程个数越多，代表系统越空闲。</li>
<li>requested_threads_started：是指系统已经启动的线程个数，在方法binder_thread_write()中，执行一次<code>BC_REGISTER_LOOPER</code>，则requested_threads_started++，requested_threads–；上限为<code>max_threads</code>.<code>BC_REGISTER_LOOPER</code>次数与<code>requested_threads_started</code>个数应该相等；</li>
<li>requested_threads:请求的线程个数，在方法binder_thread_read()中，当同时满足requested_threads_started小于最大线程数，没有ready_threads线程，且requested_threads=0，则执行requested_threads++。可见requested_threads取值要么为0，要么为1.</li>
</ul>
<h4 id="2-binder-thread"><a href="#2-binder-thread" class="headerlink" title="2.binder_thread"></a>2.binder_thread</h4><p>binder_thread结构体代表当前binder操作所在的线程</p>
<table>
<thead>
<tr>
<th style="text-align:left">类型</th>
<th style="text-align:left">成员变量</th>
<th style="text-align:left">解释</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left">struct binder_proc *</td>
<td style="text-align:left">proc</td>
<td style="text-align:left">线程所属的进程</td>
</tr>
<tr>
<td style="text-align:left">struct rb_node</td>
<td style="text-align:left">rb_node</td>
<td style="text-align:left"></td>
</tr>
<tr>
<td style="text-align:left">int</td>
<td style="text-align:left">pid</td>
<td style="text-align:left">线程pid</td>
</tr>
<tr>
<td style="text-align:left">int</td>
<td style="text-align:left">looper</td>
<td style="text-align:left">looper的状态</td>
</tr>
<tr>
<td style="text-align:left">struct binder_transaction *</td>
<td style="text-align:left">transaction_stack</td>
<td style="text-align:left">线程正在处理的事务</td>
</tr>
<tr>
<td style="text-align:left">struct list_head</td>
<td style="text-align:left">todo</td>
<td style="text-align:left">将要处理的链表</td>
</tr>
<tr>
<td style="text-align:left">uint32_t</td>
<td style="text-align:left">return_error</td>
<td style="text-align:left">write失败后，返回的错误码</td>
</tr>
<tr>
<td style="text-align:left">uint32_t</td>
<td style="text-align:left">return_error2</td>
<td style="text-align:left">write失败后，返回的错误码2</td>
</tr>
<tr>
<td style="text-align:left">wait_queue_head_t</td>
<td style="text-align:left">wait</td>
<td style="text-align:left">等待队列的队头</td>
</tr>
<tr>
<td style="text-align:left">struct binder_stats</td>
<td style="text-align:left">stats</td>
<td style="text-align:left">binder线程的统计信息</td>
</tr>
</tbody>
</table>
<p>looper的状态如下：</p>

<pre><code>
enum {
    BINDER_LOOPER_STATE_REGISTERED  = 0x01, // 创建注册线程BC_REGISTER_LOOPER
    BINDER_LOOPER_STATE_ENTERED     = 0x02, // 创建主线程BC_ENTER_LOOPER
    BINDER_LOOPER_STATE_EXITED      = 0x04, // 已退出
    BINDER_LOOPER_STATE_INVALID     = 0x08, // 非法
    BINDER_LOOPER_STATE_WAITING     = 0x10, // 等待中
    BINDER_LOOPER_STATE_NEED_RETURN = 0x20, // 需要返回
};</code></pre>

<p>binder_thread_write()过程:</p>
<ul>
<li>收到 BC_REGISTER_LOOPER,则线程状态为BINDER_LOOPER_STATE_REGISTERED;</li>
<li>收到 BC_ENTER_LOOPER,则线程状态为 BINDER_LOOPER_STATE_ENTERED;</li>
<li>收到 BC_EXIT_LOOPER, 则线程状态为BINDER_LOOPER_STATE_EXITED;</li>
</ul>
<p>其他3个状态的时机：</p>
<ul>
<li>BINDER_LOOPER_STATE_WAITING:<ul>
<li>当停留在binder_thread_read()的wait_event_xxx过程, 则设置该状态;</li>
</ul>
</li>
<li>BINDER_LOOPER_STATE_NEED_RETURN:<ul>
<li>binder_get_thread()过程, 根据binder_proc查询不到当前线程所对应的binder_thread,会新建binder_thread对象；</li>
<li>binder_deferred_flush()过程；</li>
</ul>
</li>
<li>BINDER_LOOPER_STATE_INVALID:<ul>
<li>当binder_thread创建过程状态不正确时会设置.</li>
</ul>
</li>
</ul>
<h4 id="3-binder-node"><a href="#3-binder-node" class="headerlink" title="3.binder_node"></a>3.binder_node</h4><p>binder_node代表一个binder实体</p>
<table>
<thead>
<tr>
<th style="text-align:left">类型</th>
<th style="text-align:left">成员变量</th>
<th style="text-align:left">解释</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left">int</td>
<td style="text-align:left">debug_id</td>
<td style="text-align:left">节点创建时分配，具有全局唯一性，用于调试使用</td>
</tr>
<tr>
<td style="text-align:left">struct binder_work</td>
<td style="text-align:left">work</td>
<td style="text-align:left"></td>
</tr>
<tr>
<td style="text-align:left">struct rb_node</td>
<td style="text-align:left">rb_node</td>
<td style="text-align:left">binder节点正常使用，union</td>
</tr>
<tr>
<td style="text-align:left">struct hlist_node</td>
<td style="text-align:left">dead_node</td>
<td style="text-align:left">binder节点已销毁，union</td>
</tr>
<tr>
<td style="text-align:left">struct binder_proc *</td>
<td style="text-align:left">proc</td>
<td style="text-align:left">binder所在的进程</td>
</tr>
<tr>
<td style="text-align:left">struct hlist_head</td>
<td style="text-align:left">refs</td>
<td style="text-align:left">所有指向该节点的binder引用队列</td>
</tr>
<tr>
<td style="text-align:left">int</td>
<td style="text-align:left">internal_strong_refs</td>
<td style="text-align:left"></td>
</tr>
<tr>
<td style="text-align:left">int</td>
<td style="text-align:left">local_weak_refs</td>
<td style="text-align:left"></td>
</tr>
<tr>
<td style="text-align:left">int</td>
<td style="text-align:left">local_strong_refs</td>
<td style="text-align:left"></td>
</tr>
<tr>
<td style="text-align:left">binder_uintptr_t</td>
<td style="text-align:left">ptr</td>
<td style="text-align:left">指向用户空间binder_node的指针</td>
</tr>
<tr>
<td style="text-align:left">binder_uintptr_t</td>
<td style="text-align:left">cookie</td>
<td style="text-align:left">附件数据</td>
</tr>
<tr>
<td style="text-align:left">unsigned</td>
<td style="text-align:left">has_strong_ref</td>
<td style="text-align:left">占位1bit</td>
</tr>
<tr>
<td style="text-align:left">unsigned</td>
<td style="text-align:left">pending_strong_ref</td>
<td style="text-align:left">占位1bit</td>
</tr>
<tr>
<td style="text-align:left">unsigned</td>
<td style="text-align:left">has_weak_ref</td>
<td style="text-align:left">占位1bit</td>
</tr>
<tr>
<td style="text-align:left">unsigned</td>
<td style="text-align:left">pending_weak_ref</td>
<td style="text-align:left">占位1bit</td>
</tr>
<tr>
<td style="text-align:left">unsigned</td>
<td style="text-align:left">has_async_transaction</td>
<td style="text-align:left">占位1bit</td>
</tr>
<tr>
<td style="text-align:left">unsigned</td>
<td style="text-align:left">accept_fds</td>
<td style="text-align:left">占位1bit</td>
</tr>
<tr>
<td style="text-align:left">unsigned</td>
<td style="text-align:left">min_priority</td>
<td style="text-align:left">占位8bit，最小优先级</td>
</tr>
<tr>
<td style="text-align:left">struct list_head</td>
<td style="text-align:left">async_todo</td>
<td style="text-align:left">异步todo队列</td>
</tr>
</tbody>
</table>
<p>binder_node有一个联合类型：</p>

<pre><code>
union {
        struct rb_node rb_node;
        struct hlist_node dead_node;
    };</code></pre>

<p>当Binder对象已销毁，但还存在该Binder节点引用，则采用dead_node，并加入到全局列表<code>binder_dead_nodes</code>；否则使用rb_node节点。</p>
<p>另外：</p>
<ul>
<li>binder_node.ptr对应于flat_binder_object.binder；</li>
<li>binder_node.cookie对应于flat_binder_object.cookie。</li>
</ul>
<h4 id="4-binder-ref"><a href="#4-binder-ref" class="headerlink" title="4.binder_ref"></a>4.binder_ref</h4><table>
<thead>
<tr>
<th style="text-align:left">类型</th>
<th style="text-align:left">成员变量</th>
<th style="text-align:left">解释</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left">int</td>
<td style="text-align:left">debug_id</td>
<td style="text-align:left">用于调试使用</td>
</tr>
<tr>
<td style="text-align:left">struct rb_node</td>
<td style="text-align:left">rb_node_desc</td>
<td style="text-align:left">以desc为索引的红黑树</td>
</tr>
<tr>
<td style="text-align:left">struct rb_node</td>
<td style="text-align:left">rb_node_node</td>
<td style="text-align:left">以node为索引的红黑树</td>
</tr>
<tr>
<td style="text-align:left">struct hlist_node</td>
<td style="text-align:left">node_entry</td>
<td style="text-align:left"></td>
</tr>
<tr>
<td style="text-align:left">struct binder_proc *</td>
<td style="text-align:left">proc</td>
<td style="text-align:left">binder进程</td>
</tr>
<tr>
<td style="text-align:left">struct binder_node *</td>
<td style="text-align:left">node</td>
<td style="text-align:left">binder节点</td>
</tr>
<tr>
<td style="text-align:left">uint32_t</td>
<td style="text-align:left">desc</td>
<td style="text-align:left">handle</td>
</tr>
<tr>
<td style="text-align:left">int</td>
<td style="text-align:left">strong</td>
<td style="text-align:left">强引用次数</td>
</tr>
<tr>
<td style="text-align:left">int</td>
<td style="text-align:left">weak</td>
<td style="text-align:left">弱引用次数</td>
</tr>
<tr>
<td style="text-align:left">struct binder_ref_death *</td>
<td style="text-align:left">death</td>
<td style="text-align:left">当应用注册死亡通知时，此域不为空</td>
</tr>
</tbody>
</table>
<p>binder引用的查询方式如下：</p>
<ul>
<li>node + proc =&gt; ref (transaction)</li>
<li>desc + proc =&gt; ref (transaction, inc/dec ref)</li>
<li>node =&gt; refs + procs (proc exit)</li>
</ul>
<h4 id="5-binder-ref-death"><a href="#5-binder-ref-death" class="headerlink" title="5. binder_ref_death"></a>5. binder_ref_death</h4>
<pre><code>
struct binder_ref_death {
    struct binder_work work;
    binder_uintptr_t cookie;
};</code></pre>

<p>cookie只是死亡通知的BpBinder代理对象的指针</p>
<h4 id="6-binder-write-read"><a href="#6-binder-write-read" class="headerlink" title="6.binder_write_read"></a>6.binder_write_read</h4><p>用户空间程序和Binder驱动程序交互基本都是通过BINDER_WRITE_READ命令，来进行数据的读写操作。</p>
<table>
<thead>
<tr>
<th style="text-align:left">类型</th>
<th style="text-align:left">成员变量</th>
<th style="text-align:left">解释</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left">binder_size_t</td>
<td style="text-align:left">write_size</td>
<td style="text-align:left">write_buffer的总字节数</td>
</tr>
<tr>
<td style="text-align:left">binder_size_t</td>
<td style="text-align:left">write_consumed</td>
<td style="text-align:left">write_buffer已消费的字节数</td>
</tr>
<tr>
<td style="text-align:left">binder_uintptr_t</td>
<td style="text-align:left">write_buffer</td>
<td style="text-align:left">写缓冲数据的指针</td>
</tr>
<tr>
<td style="text-align:left">binder_size_t</td>
<td style="text-align:left">read_size</td>
<td style="text-align:left">read_buffer的总字节数</td>
</tr>
<tr>
<td style="text-align:left">binder_size_t</td>
<td style="text-align:left">read_consumed</td>
<td style="text-align:left">read_buffer已消费的字节数</td>
</tr>
<tr>
<td style="text-align:left">binder_uintptr_t</td>
<td style="text-align:left">read_buffer</td>
<td style="text-align:left">读缓存数据的指针</td>
</tr>
</tbody>
</table>
<ul>
<li>write_buffer变量：用于发送IPC(或IPC reply)数据，即传递经由Binder Driver的数据时使用。</li>
<li>read_buffer 变量：用于接收来自Binder Driver的数据，即Binder Driver在接收IPC(或IPC reply)数据后，保存到read_buffer，再传递到用户空间；</li>
</ul>
<p>write_buffer和read_buffer都是包含Binder协议命令和binder_transaction_data结构体。</p>
<ul>
<li>copy_from_user()将用户空间IPC数据拷贝到内核态binder_write_read结构体；</li>
<li>copy_to_user()将用内核态binder_write_read结构体数据拷贝到用户空间；</li>
</ul>
<h4 id="7-binder-transaction-data"><a href="#7-binder-transaction-data" class="headerlink" title="7.binder_transaction_data"></a>7.binder_transaction_data</h4><p>当BINDER_WRITE_READ命令的目标是本地Binder node时，target使用ptr，否则使用handle。只有当这是Binder node时，cookie才有意义，表示附加数据，由进程自己解释。</p>

<pre><code>
struct binder_transaction_data {
    union {
        __u32    handle;       //binder_ref（即handle）
        binder_uintptr_t ptr;     //Binder_node的内存地址
    } target;  //RPC目标
    binder_uintptr_t    cookie;    //BBinder指针
    __u32        code;        //RPC代码，代表Client与Server双方约定的命令码

    __u32            flags; //标志位，比如TF_ONE_WAY代表异步，即不等待Server端回复
    pid_t        sender_pid;  //发送端进程的pid
    uid_t        sender_euid; //发送端进程的uid
    binder_size_t    data_size;    //data数据的总大小
    binder_size_t    offsets_size; //IPC对象的大小

    union {
        struct {
            binder_uintptr_t    buffer; //数据区起始地址
            binder_uintptr_t    offsets; //数据区IPC对象偏移量
        } ptr;
        __u8    buf[8];
    } data;   //RPC数据
};</code></pre>

<ul>
<li><code>target</code>: 对于BpBinder则使用handle，对于BBinder则使用ptr，故使用union数据类型来表示；</li>
<li><code>code</code>: 比如注册服务过程code为ADD_SERVICE_TRANSACTION，又比如获取服务code为CHECK_SERVICE_TRANSACTION</li>
<li><code>data</code>：代表整个数据区，其中data.ptr指向的是传递给Binder驱动的数据区的起始地址，data.offsets指的是数据区中IPC数据地址的偏移量。</li>
<li><code>cookie</code>: 记录着BBinder指针。</li>
<li>data_size：代表本次传输的parcel数据的大小；</li>
<li>offsets_size： 代表传递的IPC对象的大小；根据这个可以推测出传递了多少个binder对象。<ul>
<li>对于64位IPC，一个IPC对象大小等于8；</li>
<li>对于32位IPC，一个IPC对象大小等于4；</li>
</ul>
</li>
</ul>
<h4 id="8-flat-binder-object"><a href="#8-flat-binder-object" class="headerlink" title="8.flat_binder_object"></a>8.flat_binder_object</h4><p>flat_binder_object结构体代表Binder对象在两个进程间传递的扁平结构。</p>
<table>
<thead>
<tr>
<th style="text-align:left">类型</th>
<th style="text-align:left">成员变量</th>
<th style="text-align:left">解释</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left">__u32</td>
<td style="text-align:left">type</td>
<td style="text-align:left">类型</td>
</tr>
<tr>
<td style="text-align:left">__u32</td>
<td style="text-align:left">flags</td>
<td style="text-align:left">记录优先级、文件描述符许可</td>
</tr>
<tr>
<td style="text-align:left">binder_uintptr_t</td>
<td style="text-align:left">binder</td>
<td style="text-align:left">（union）当传递的是binder_node时使用，指向binder_node在应用程序的地址</td>
</tr>
<tr>
<td style="text-align:left">__u32</td>
<td style="text-align:left">handle</td>
<td style="text-align:left">（union）当传递的是binder_ref时使用，存放Binder在进程中的引用号</td>
</tr>
<tr>
<td style="text-align:left">binder_uintptr_t</td>
<td style="text-align:left">cookie</td>
<td style="text-align:left">只对binder_node有效，存放binder_node的额外数据</td>
</tr>
</tbody>
</table>
<p>此处的类型type的可能取值来自于<code>enum</code>，成员如下：</p>
<table>
<thead>
<tr>
<th style="text-align:left">成员变量</th>
<th style="text-align:left">解释</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left">BINDER_TYPE_BINDER</td>
<td style="text-align:left">binder_node的强引用</td>
</tr>
<tr>
<td style="text-align:left">BINDER_TYPE_WEAK_BINDER</td>
<td style="text-align:left">binder_node的弱引用</td>
</tr>
<tr>
<td style="text-align:left">BINDER_TYPE_HANDLE</td>
<td style="text-align:left">binder_ref强引用</td>
</tr>
<tr>
<td style="text-align:left">BINDER_TYPE_WEAK_HANDLE</td>
<td style="text-align:left">binder_ref弱引用</td>
</tr>
<tr>
<td style="text-align:left">BINDER_TYPE_FD</td>
<td style="text-align:left">binder文件描述符</td>
</tr>
</tbody>
</table>
<p>说明：</p>
<ul>
<li>当type等于BINDER_TYPE_BINDER或BINDER_TYPE_WEAK_BINDER类型时， 代表Server进程向ServiceManager进程注册服务，则创建binder_node对象；</li>
<li>当type等于BINDER_TYPE_HANDLE或BINDER_TYPE_WEAK_HEANDLE类型时， 代表Client进程向Server进程请求代理，则创建binder_ref对象；</li>
<li>当type等于BINDER_TYPE_FD类型时， 代表进程向另一个进程发送文件描述符，只打开文件，则无需创建任何对象。</li>
</ul>
<h4 id="9-binder-buffer"><a href="#9-binder-buffer" class="headerlink" title="9.binder_buffer"></a>9.binder_buffer</h4><p>每一次Binder传输数据时，都会先从Binder内存缓存区中分配一个binder_buffer来存储传输数据。</p>
<table>
<thead>
<tr>
<th style="text-align:left">类型</th>
<th style="text-align:left">成员变量</th>
<th style="text-align:left">解释</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left">struct list_head</td>
<td style="text-align:left">entry</td>
<td style="text-align:left">buffer实体的地址</td>
</tr>
<tr>
<td style="text-align:left">struct rb_node</td>
<td style="text-align:left">rb_node</td>
<td style="text-align:left">buffer实体的地址</td>
</tr>
<tr>
<td style="text-align:left">unsigned</td>
<td style="text-align:left">free</td>
<td style="text-align:left">标记是否是空闲buffer，占位1bit</td>
</tr>
<tr>
<td style="text-align:left">unsigned</td>
<td style="text-align:left">allow_user_free</td>
<td style="text-align:left">是否允许用户释放，占位1bit</td>
</tr>
<tr>
<td style="text-align:left">unsigned</td>
<td style="text-align:left">async_transaction</td>
<td style="text-align:left">占位1bit</td>
</tr>
<tr>
<td style="text-align:left">unsigned</td>
<td style="text-align:left">debug_id</td>
<td style="text-align:left">占位29bit</td>
</tr>
<tr>
<td style="text-align:left">struct binder_transaction *</td>
<td style="text-align:left">transaction</td>
<td style="text-align:left">该缓存区的需要处理的事务</td>
</tr>
<tr>
<td style="text-align:left">struct binder_node *</td>
<td style="text-align:left">target_node</td>
<td style="text-align:left">该缓存区所需处理的Binder实体</td>
</tr>
<tr>
<td style="text-align:left">size_t</td>
<td style="text-align:left">data_size</td>
<td style="text-align:left">数据大小</td>
</tr>
<tr>
<td style="text-align:left">size_t</td>
<td style="text-align:left">offsets_size</td>
<td style="text-align:left">数据偏移量</td>
</tr>
<tr>
<td style="text-align:left">uint8_t</td>
<td style="text-align:left">data[0]</td>
<td style="text-align:left">数据地址</td>
</tr>
</tbody>
</table>
<p>每一个binder_buffer分为空闲和已分配的，通过free标记来区分。空闲和已分配的binder_buffer通过各自的成员变量rb_node分别连入binder_proc的free_buffers(红黑树)和allocated_buffers(红黑树)。</p>
<h4 id="10-binder-transaction"><a href="#10-binder-transaction" class="headerlink" title="10.binder_transaction"></a>10.binder_transaction</h4><table>
<thead>
<tr>
<th style="text-align:left">类型</th>
<th style="text-align:left">成员变量</th>
<th style="text-align:left">解释</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left">int</td>
<td style="text-align:left">debug_id</td>
<td style="text-align:left">用于调试</td>
</tr>
<tr>
<td style="text-align:left">struct binder_work</td>
<td style="text-align:left">work</td>
<td style="text-align:left">binder工作类型</td>
</tr>
<tr>
<td style="text-align:left">struct binder_thread *</td>
<td style="text-align:left">from</td>
<td style="text-align:left">发送端线程</td>
</tr>
<tr>
<td style="text-align:left">struct binder_transaction *</td>
<td style="text-align:left">from_parent</td>
<td style="text-align:left">上一个事务</td>
</tr>
<tr>
<td style="text-align:left">struct binder_proc *</td>
<td style="text-align:left">to_proc</td>
<td style="text-align:left">接收端进程</td>
</tr>
<tr>
<td style="text-align:left">struct binder_thread *</td>
<td style="text-align:left">to_thread</td>
<td style="text-align:left">接收端线程</td>
</tr>
<tr>
<td style="text-align:left">struct binder_transaction *</td>
<td style="text-align:left">to_parent</td>
<td style="text-align:left">下一个事务</td>
</tr>
<tr>
<td style="text-align:left">unsigned</td>
<td style="text-align:left">need_reply</td>
<td style="text-align:left">是否需要回复</td>
</tr>
<tr>
<td style="text-align:left">struct binder_buffer *</td>
<td style="text-align:left">buffer</td>
<td style="text-align:left">数据buffer</td>
</tr>
<tr>
<td style="text-align:left">unsigned int</td>
<td style="text-align:left">code</td>
<td style="text-align:left">通信方法，比如startService</td>
</tr>
<tr>
<td style="text-align:left">unsigned int</td>
<td style="text-align:left">flags</td>
<td style="text-align:left">标志，比如是否oneway</td>
</tr>
<tr>
<td style="text-align:left">long</td>
<td style="text-align:left">priority</td>
<td style="text-align:left">优先级</td>
</tr>
<tr>
<td style="text-align:left">long</td>
<td style="text-align:left">saved_priority</td>
<td style="text-align:left">保存的优先级</td>
</tr>
<tr>
<td style="text-align:left">kuid_t</td>
<td style="text-align:left">sender_euid</td>
<td style="text-align:left">发送端uid</td>
</tr>
</tbody>
</table>
<p>执行binder_transaction()过程创建的结构体</p>
<ul>
<li>debug_id：是一个全局静态变量，每当创建一个<code>binder_transaction</code>或<code>binder_node</code>或<code>binder_ref</code>对象，则++debug_id</li>
<li>from与to_thread是一对，分别是发送端线程和接收端线程；</li>
<li>from_parent与to_parent是一对，分别是上一个和下一个binder_transaction，组成一个链表。<ul>
<li>执行binder_transaction()方法过程，当非oneway的BC_TRANSACTION时，则设置当前事务t-&gt;from_parent等于当前线程的transaction_stack；</li>
<li>执行binder_thread_read()方法过程，当非oneway的BR_TRANSACTION时，则设置当前事务t-&gt;to_parent等于当前线程的transaction_stack；</li>
</ul>
</li>
</ul>
<h4 id="11-binder-work"><a href="#11-binder-work" class="headerlink" title="11.binder_work"></a>11.binder_work</h4>
<pre><code>
struct binder_work {
    struct list_head entry;
    enum {
        BINDER_WORK_TRANSACTION = 1, 
        BINDER_WORK_TRANSACTION_COMPLETE,
        BINDER_WORK_NODE, 
        BINDER_WORK_DEAD_BINDER, 
        BINDER_WORK_DEAD_BINDER_AND_CLEAR, 
        BINDER_WORK_CLEAR_DEATH_NOTIFICATION,
    } type;
};</code></pre>

<p>binder_work.type设置时机：</p>
<ul>
<li>binder_transaction()</li>
<li>binder_thread_write()</li>
<li>binder_new_node()</li>
</ul>
<h4 id="12-binder-state"><a href="#12-binder-state" class="headerlink" title="12.binder_state"></a>12.binder_state</h4><table>
<thead>
<tr>
<th style="text-align:left">类型</th>
<th style="text-align:left">成员变量</th>
<th style="text-align:left">解释</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left">int</td>
<td style="text-align:left">fd</td>
<td style="text-align:left">文件描述符</td>
</tr>
<tr>
<td style="text-align:left">void *</td>
<td style="text-align:left">mapped</td>
<td style="text-align:left">映射到进程空间的起始地址</td>
</tr>
<tr>
<td style="text-align:left">size_t</td>
<td style="text-align:left">mapsize</td>
<td style="text-align:left">内存空间的映射大小</td>
</tr>
</tbody>
</table>
