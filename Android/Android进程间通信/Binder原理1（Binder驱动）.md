# 1. 概述
Android的Binder架构主要分为四部分：
1. Binder客户端
2. Binder服务端
3. ServiceManager
4. Binder驱动运行在内核，注册在/dev/binder

Linux进程间通信，除了共享内存，都要从进程1复制到内核缓存区，再从内核缓存区复制到进程2
copy_from_user：将用户空间的数据拷贝到内核空间。
copy_to_user：将内核空间的数据拷贝到用户空间。

驱动概述：每块可以在运行时添加到内核的代码, 被称为一个模块。Linux 内核提供了对许多模块类型的支持, 包括但不限于设备驱动。
Linux内核中设备分为字符设备、块设备、网络接口。字符设备顺序读写，数据传输量较小，无缓冲区，如终端、USB串口、键盘；块设备随机读写，有缓冲区，如硬盘、闪存。设备驱动用以控制设备，Binder驱动属于字符设备中的misc（杂项）设备的驱动，Binder驱动实际上没有操作硬件设备，只是实现方式和设备驱动程序一样。

Binder架构采用分层架构设计，每一层都有其不同的功能：

* Java应用层：对于上层应用通过调用AMP.startService，完全可以不用关心底层,经过层层调用，最终必然会调用到AMS.startService；
* Java Binder层：Binder通信是采用C/S架构，Android系统的基础架构便已设计好Binder在Java framework层的Binder客户类BinderProxy和服务类Binder；
* Native Binder层：对于Native层,如果需要直接使用Binder(比如media相关)，则可以直接使用BpBinder和BBinder(当然这里还有JavaBBinder)即可，对于上一层Java IPC的通信也是基于这个层面；
* Kernel Binder层：这里是Binder Driver, 前面3层都跑在用户空间,对于用户空间的内存资源是不共享的，每个Android的进程只能运行在自己进程所拥有的虚拟地址空间，而内核空间却是可共享的。真正通信的核心环节还是在Binder Driver。

Binder通信采用C/S架构，从组件视角来说，包含Client、Server、ServiceManager以及binder驱动，其中ServiceManager用于管理系统中的各种服务。Binder 在 framework 层进行了封装，通过 JNI 技术调用 Native（C/C++）层的 Binder 架构，Binder 在 Native 层以 ioctl 的方式与 Binder 驱动通讯。

# 2. Binder驱动
Linux内核初始化时会调用驱动的初始化程序，从module_init开始，最终调用到各个驱动程序的初始化函数。Binder的设备驱动入口函数不是module_init，而是device_initcall(binder_init)，Binder的初始化函数为 `binder_init`，代码位于/kernel/drivers/staging/android/binder.c。

```
//绑定binder驱动操作函数
static const struct file_operations binder_fops = {
	.owner = THIS_MODULE,
	.poll = binder_poll,
	.unlocked_ioctl = binder_ioctl,
	.compat_ioctl = binder_ioctl,
	.mmap = binder_mmap,
	.open = binder_open,
	.flush = binder_flush,
	.release = binder_release,
};

//创建misc类型的驱动，在binder_init中调用misc_register函数时作为参数
static struct miscdevice binder_miscdev = {
	.minor = MISC_DYNAMIC_MINOR,
	.name = "binder",
	.fops = &binder_fops  //绑定binder驱动操作函数
};

//binder驱动初始化
static int __init binder_init(void)
{
	int ret;
    //创建名为binder的工作队列
	binder_deferred_workqueue = create_singlethread_workqueue("binder");
	if (!binder_deferred_workqueue)
		return -ENOMEM;
    //创建目录/binder
	binder_debugfs_dir_entry_root = debugfs_create_dir("binder", NULL);
	if (binder_debugfs_dir_entry_root)
        //创建目录/binder/proc
		binder_debugfs_dir_entry_proc = debugfs_create_dir("proc",
						 binder_debugfs_dir_entry_root);
    // 注册misc设备（binder驱动）
	ret = misc_register(&binder_miscdev);
	if (binder_debugfs_dir_entry_root) {
        //创建文件/binder/proc/state
		debugfs_create_file("state",
				    S_IRUGO,
				    binder_debugfs_dir_entry_root,
				    NULL,
				    &binder_state_fops);
        //创建文件/binder/proc/stats
		debugfs_create_file("stats",
				    S_IRUGO,
				    binder_debugfs_dir_entry_root,
				    NULL,
				    &binder_stats_fops);
        //创建文件/binder/proc/transactions
		debugfs_create_file("transactions",
				    S_IRUGO,
				    binder_debugfs_dir_entry_root,
				    NULL,
				    &binder_transactions_fops);
        //创建文件/binder/proc/transaction_log
		debugfs_create_file("transaction_log",
				    S_IRUGO,
				    binder_debugfs_dir_entry_root,
				    &binder_transaction_log,
				    &binder_transaction_log_fops);
        //创建文件/binder/proc/failed_transaction_log
		debugfs_create_file("failed_transaction_log",
				    S_IRUGO,
				    binder_debugfs_dir_entry_root,
				    &binder_transaction_log_failed,
				    &binder_transaction_log_fops);
	}
	return ret;
}
```

binder驱动初始化过程主要是：创建binder设备，绑定操作函数：binder_open、binder_mmap、binder_ioctl等，创建相关文件，注册驱动。

## 2.1 打开Binder驱动----binder_open
上层进程通过open系统调用，传入/dev/binder，就会调用到binder_open
```
static int binder_open(struct inode *nodp, struct file *filp)
{
	struct binder_proc *proc;//进程对应的binder状态数据的记录体

	binder_debug(BINDER_DEBUG_OPEN_CLOSE, "binder_open: %d:%d\n",
		     current->group_leader->pid, current->pid);

//为每个打开binder设备的进程创建一个 binder_proc 结构，分配内核内存空间，此结构包含了该进程中与binder通信有关的所有信息
	proc = kzalloc(sizeof(*proc), GFP_KERNEL);
	if (proc == NULL)
		return -ENOMEM;
	get_task_struct(current);
	proc->tsk = current;//将当前线程的task保存到binder_proc的tsk
	INIT_LIST_HEAD(&proc->todo);//初始化todo队列
	init_waitqueue_head(&proc->wait);//初始化wait队列
	proc->default_priority = task_nice(current);//将当前进程的nice值转换为进程优先级

	binder_lock(__func__);//同步锁，因为binder支持多线程访问

	binder_stats_created(BINDER_STAT_PROC);//binder_stats是binder的统计数据载体
	hlist_add_head(&proc->proc_node, &binder_procs);//将proc加入到binder_procs头部
	proc->pid = current->group_leader->pid;//进程id
	INIT_LIST_HEAD(&proc->delivered_death);//初始化已分发的死亡通知列表
	filp->private_data = proc;//file文件指针的private_data变量指向binder_proc数据

	binder_unlock(__func__);//释放同步锁

	if (binder_debugfs_dir_entry_proc) {
		char strbuf[11];

		snprintf(strbuf, sizeof(strbuf), "%u", proc->pid);
		proc->debugfs_entry = debugfs_create_file(strbuf, S_IRUGO,
			binder_debugfs_dir_entry_proc, proc, &binder_proc_fops);
	}

	return 0;
}
```

## 2.2 binder_mmap
上层应用调用mmap，传入binder驱动的文件描述符，就会调用到binder_mmap。

Binder驱动在内核空间，一个接收端B进程通过mmap将Binder驱动使用的物理内存映射到自己的用户空间，然后发送端A进程将自己用户空间的数据通过copy_from_user传输到Binder的内核空间，而这个内核空间已经被B映射到了用户空间，所以就一次复制完成了跨进程传输。至于为什么A不也映射到用户空间，主要是考虑到进程共享内存后的同步等问题。

下面分段分析：
```
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
	int ret;
	struct vm_struct *area;
	struct binder_proc *proc = filp->private_data;
	const char *failure_string;
	struct binder_buffer *buffer;
```
* vm_area_struct *vma 描述了一块供应用程序使用的虚拟内存，vma->vm_start和vma->vm_end分别表示此虚拟内存的起始位置。从mmap系统调用到这里驱动的mmap函数，内核已经做了很多工作，vma就是内核预先准备好的虚拟内存，在驱动中需要对这个虚拟内存做实际的物理内存映射。
* vm_struct *area Binder驱动中对虚拟内存的描述，内核空间
* binder_proc *proc binder_open中已接触过，它是Binder驱动为每个进程分配的一个数据结果，存储该进程的内存分配、线程管理等所有信息
```
	if (proc->tsk != current)
		return -EINVAL;

	if ((vma->vm_end - vma->vm_start) > SZ_4M)
		vma->vm_end = vma->vm_start + SZ_4M;
```
开始前先判断应用程序申请的内存大小是否超过4MB，最多4MB
```
	binder_debug(BINDER_DEBUG_OPEN_CLOSE,
		     "binder_mmap: %d %lx-%lx (%ld K) vma %lx pagep %lx\n",
		     proc->pid, vma->vm_start, vma->vm_end,
		     (vma->vm_end - vma->vm_start) / SZ_1K, vma->vm_flags,
		     (unsigned long)pgprot_val(vma->vm_page_prot));

	if (vma->vm_flags & FORBIDDEN_MMAP_FLAGS) {
		ret = -EPERM;
		failure_string = "bad vm_flags";
		goto err_bad_arg;
	}
	vma->vm_flags = (vma->vm_flags | VM_DONTCOPY) & ~VM_MAYWRITE;
```
判断vma中是否有禁止mmap的标志，没有的话，就添加其他标志
```
	mutex_lock(&binder_mmap_lock);
	if (proc->buffer) {
		ret = -EBUSY;
		failure_string = "already mapped";
		goto err_already_mapped;
	}

	area = get_vm_area(vma->vm_end - vma->vm_start, VM_IOREMAP);
	if (area == NULL) {
		ret = -ENOMEM;
		failure_string = "get_vm_area";
		goto err_get_vm_area_failed;
	}
	proc->buffer = area->addr;//映射后的地址
	proc->user_buffer_offset = vma->vm_start - (uintptr_t)proc->buffer;
	mutex_unlock(&binder_mmap_lock);
```
同步锁区域，先判断是否已经映射。get_vm_area将为Binder驱动获取一段可用的虚拟内存空间（此时还没有分配实际的物理内存），将proc->buffer指向这块内存开始的位置，并计算它与应用程序中相关联的虚拟内存地址的偏移量
```
#ifdef CONFIG_CPU_CACHE_VIPT
	if (cache_is_vipt_aliasing()) {
		while (CACHE_COLOUR((vma->vm_start ^ (uint32_t)proc->buffer))) {
			pr_info("binder_mmap: %d %lx-%lx maps %p bad alignment\n", proc->pid, vma->vm_start, vma->vm_end, proc->buffer);
			vma->vm_start += PAGE_SIZE;
		}
	}
#endif
```
```
	proc->pages = kzalloc(sizeof(proc->pages[0]) * ((vma->vm_end - vma->vm_start) / PAGE_SIZE), GFP_KERNEL);
	if (proc->pages == NULL) {
		ret = -ENOMEM;
		failure_string = "alloc page array";
		goto err_alloc_pages_failed;
	}
	proc->buffer_size = vma->vm_end - vma->vm_start;

	vma->vm_ops = &binder_vm_ops;
	vma->vm_private_data = proc;
```
kzalloc分配了pages数组的空间，pages声明为 struct page **pages，是一个二维指针，用于指示Binder申请的物理页面的状态

```
	if (binder_update_page_range(proc, 1, proc->buffer, proc->buffer + PAGE_SIZE, vma)) {
		ret = -ENOMEM;
		failure_string = "alloc small buf";
		goto err_alloc_small_buf_failed;
	}
```
binder_update_page_range开始真正申请物理页面，该函数各参数作用如下：

```
binder_update_page_range(struct binder_proc *poc, int allocate, void *start, void *end, struct vm_area_struct *vma);

proc：申请内存的进程所对应的binder_proc
allocate：1表示申请，2表示释放
start：Binder中虚拟内存起点
end：Binder中虚拟内存终点
vma：应用程序中虚拟内存的描述（函数内将完成内核内存空间和应用内存空间的映射）
```
上面传入的start和end相差一个PAGE_SIZE，因为开始不需要分配很多，后面有需要再增加即可。

```
	buffer = proc->buffer;
	INIT_LIST_HEAD(&proc->buffers);
	list_add(&buffer->entry, &proc->buffers);
	buffer->free = 1;
	binder_insert_free_buffer(proc, buffer);
	proc->free_async_space = proc->buffer_size / 2;
	barrier();
	proc->files = get_files_struct(current);
	proc->vma = vma;
	proc->vma_vm_mm = vma->vm_mm;

	/*pr_info("binder_mmap: %d %lx-%lx maps %p\n",
		 proc->pid, vma->vm_start, vma->vm_end, proc->buffer);*/
	return 0;
```
分配了物理内存后，这里开始将这一段内存与应用程序的虚拟内存关联，实现Binder驱动和应用程序共享内存。

```
err_alloc_small_buf_failed:
	kfree(proc->pages);
	proc->pages = NULL;
err_alloc_pages_failed:
	mutex_lock(&binder_mmap_lock);
	vfree(proc->buffer);
	proc->buffer = NULL;
err_get_vm_area_failed:
err_already_mapped:
	mutex_unlock(&binder_mmap_lock);
err_bad_arg:
	pr_err("binder_mmap: %d %lx-%lx %s failed %d\n",
	       proc->pid, vma->vm_start, vma->vm_end, failure_string, ret);
	return ret;
}
```

## 2.3 binder_ioctl
binder_ioctl部分的源码，在不了解具体使用场景的时候，还不易理解，所以这部分可以在了解serviceManager和进程交互的同时再来阅读。	

| 命令 | 说明 |
|---|---|
| BINDER_WRITE_READ | 用这个命令向Binder读写数据 |
| BINDER_SET_MAX_THREADS | 设置支持的Binder线程的最大数量 |
| BINDER_SET_CONTEXT_MGR | ServiceManager专用 |
| BINDER_THREAD_EXIT | 通知Binder 线程退出，Binder知道线程退出后可以释放相关资源 |
| BINDER_VERSION | 获取Binder版本 |

其中BINDER_WRITE_READ使用最多，它有很多子命令：
| BINDER_WRITE_READ子命令 | 说明 |
|---|---|
| BC_TRANSACTION<br/> BC_REPLY| 是BINDER_WRITE_READ中最关键的两个命令，Binder机制中的客户端和服务端的交互基本靠它们完成 |
|BC_ACQUIRE_RESULT<br>BC_ATTEMPT_ACQUIRE|未实现|
| BC_FREE_BUFFER | 用于Binder的buffer管理 |
| BC_INCREFS<br/>BC_ACQUIRE<br/>BC_RELEASE<br/>BC_DECREFS | 用于操作引用计数 |
|BC_INCREFS_DONE<br/>BC_ACQUIRE_DONE| 在BC_INCREFS和BC_ACQUIRE结束时发送 |
| BC_REGISTER_LOOPER<br/>BC_ENTER_LOOPER<br/>BC_EXIT_LOOPER |  |
| BC_REQUEST_DEATH_NOTIFICATION<br/>BC_CLEAR_DEATH_NOTIFICATION|  |
| BC_DEAD_BINDER_DONE |  |

> BC开头的命令为传给Binder的请求命令，BR开头的命令为从Binder收到的响应命令。

```
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	int ret;
	struct binder_proc *proc = filp->private_data;//从flip中取出在bind_open中创建的proc变量
	struct binder_thread *thread;
	unsigned int size = _IOC_SIZE(cmd);//命令的大小
	void __user *ubuf = (void __user *)arg;

    ...
	//第二个参数为false的话会阻塞
	ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
	if (ret)
		goto err_unlocked;
```
上面已经拿到之前创建的proc结构体，并计算了命令的大小，下面在临界保护区域执行：


```
	binder_lock(__func__);
	thread = binder_get_thread(proc);
	if (thread == NULL) {
		ret = -ENOMEM;
		goto err;
	}
```
`binder_get_thread`方法会在proc中的threads中查询是否已经添加了当前线程的节点，如果没有就插入一个新节点。用户态的线程在Linux内核中对应着一个task_struct, 线程实际也是一个进程，也就是说有自己的pid，所以threads是按pid大小排序的。

接着处理各种命令
```
	switch (cmd) {
	case BINDER_WRITE_READ://Binder读写命令
		ret = binder_ioctl_write_read(filp, cmd, arg, thread);
		if (ret)
			goto err;
		break;
	case BINDER_SET_MAX_THREADS://设置Binder的最大线程数量命令
		if (copy_from_user(&proc->max_threads, ubuf, sizeof(proc->max_threads))) {
			ret = -EINVAL;
			goto err;
		}
		break;
	case BINDER_SET_CONTEXT_MGR://设置Service Manager为上下文管理者
		ret = binder_ioctl_set_ctx_mgr(filp);
		if (ret)
			goto err;
		ret = security_binder_set_context_mgr(proc->tsk);
		if (ret < 0)
			goto err;
		break;
	case BINDER_THREAD_EXIT://线程退出时释放Binder线程
		binder_debug(BINDER_DEBUG_THREADS, "%d:%d exit\n",
			     proc->pid, thread->pid);
		binder_free_thread(proc, thread);
		thread = NULL;
		break;
	case BINDER_VERSION: {//获取Binder版本
		struct binder_version __user *ver = ubuf;

		if (size != sizeof(struct binder_version)) {
			ret = -EINVAL;
			goto err;
		}
		if (put_user(BINDER_CURRENT_PROTOCOL_VERSION,
			     &ver->protocol_version)) {
			ret = -EINVAL;
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
		thread->looper &= ~BINDER_LOOPER_STATE_NEED_RETURN;
	binder_unlock(__func__);
	wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
	if (ret && ret != -ERESTARTSYS)
		pr_info("%d:%d ioctl %x %lx returned %d\n", proc->pid, current->pid, cmd, arg, ret);
err_unlocked:
	trace_binder_ioctl_done(ret);
	return ret;
}
```

这里主要看处理`BINDER_WRITE_READ`命令，分段分析`binder_ioctl_write_read`函数：

### binder_ioctl_write_read
```
static int binder_ioctl_write_read(struct file *filp,
				unsigned int cmd, unsigned long arg,
				struct binder_thread *thread)
{
	int ret = 0;
	struct binder_proc *proc = filp->private_data;
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;
	struct binder_write_read bwr;

	if (size != sizeof(struct binder_write_read)) {
		ret = -EINVAL;
		goto out;
	}
	if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {//把用户空间数据ubuf复制到bwr
		ret = -EFAULT;
		goto out;
	}
```
将用户空间数据通过`copy_from_user`复制地址到bwr中，然后处理bwr中的数据

```
	...
	if (bwr.write_size > 0) {//如果有数据需要写
		ret = binder_thread_write(proc, thread,
					  bwr.write_buffer,
					  bwr.write_size,
					  &bwr.write_consumed);
		trace_binder_write_done(ret);
		if (ret < 0) {//错误处理
			bwr.read_consumed = 0;
			if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
				ret = -EFAULT;
			goto out;
		}
	}
```
上面代码的重点是判断有数据需要写后，调用`binder_thread_write`

```
	if (bwr.read_size > 0) {//如果有数据需要读
		ret = binder_thread_read(proc, thread, bwr.read_buffer,
					 bwr.read_size,
					 &bwr.read_consumed,
					 filp->f_flags & O_NONBLOCK);
		trace_binder_read_done(ret);
		if (!list_empty(&proc->todo))
			wake_up_interruptible(&proc->wait);
		if (ret < 0) {//错误处理
			if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
				ret = -EFAULT;
			goto out;
		}
	}
	...
	if (copy_to_user(ubuf, &bwr, sizeof(bwr))) { //执行完write和read后，把bwr复制到用户空间
		ret = -EFAULT;
		goto out;
	}
out:
	return ret;
}
```

下面看`binder_thread_write`函数的处理过程：

#### binder_thread_write
```
static int binder_thread_write(struct binder_proc *proc,
			struct binder_thread *thread,
			binder_uintptr_t binder_buffer, size_t size,
			binder_size_t *consumed)
{
	uint32_t cmd;
	void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
	void __user *ptr = buffer + *consumed;
	void __user *end = buffer + size;

	while (ptr < end && thread->return_error == BR_OK) {//循环处理buffer中所有命令
		if (get_user(cmd, (uint32_t __user *)ptr))
			return -EFAULT;
		ptr += sizeof(uint32_t);//跳过cmd的大小，这样就会指向下面需要处理的数据
		trace_binder_command(cmd);
		if (_IOC_NR(cmd) < ARRAY_SIZE(binder_stats.bc)) {
			binder_stats.bc[_IOC_NR(cmd)]++;
			proc->stats.bc[_IOC_NR(cmd)]++;
			thread->stats.bc[_IOC_NR(cmd)]++;
		}
		switch (cmd) {
		case BC_INCREFS:
		case BC_ACQUIRE:
		case BC_RELEASE:
		case BC_DECREFS:...
		case BC_INCREFS_DONE:
		case BC_ACQUIRE_DONE: ...
		case BC_ATTEMPT_ACQUIRE:...
		case BC_ACQUIRE_RESULT:...
		case BC_FREE_BUFFER: ...

		case BC_TRANSACTION:
		case BC_REPLY: {
			struct binder_transaction_data tr;

			if (copy_from_user(&tr, ptr, sizeof(tr)))
				return -EFAULT;
			ptr += sizeof(tr);
			binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
			break;
		}

		case BC_REGISTER_LOOPER:...
		case BC_ENTER_LOOPER:...
		case BC_EXIT_LOOPER:...
		case BC_REQUEST_DEATH_NOTIFICATION:
		case BC_CLEAR_DEATH_NOTIFICATION:...
		case BC_DEAD_BINDER_DONE: ...
        ...
		*consumed = ptr - buffer;
	}
	return 0;
}
```
`binder_transaction_data`数据结构是`IPCThreadState.writeTransactionData`中将Parcel包装后的数据，会被放到mOut中，然后放到binder_write_read bwr。这里的ptr就是读的bwr.write_buffer，也就是放到mOut中的位置。<br/>
然后`copy_from_user(&tr, ptr, sizeof(tr))`就会将数据地址从用户空间复制到Binder内核空间。然后将数据交给`binder_transaction`处理：


#### binder_transaction
```
static void binder_transaction(struct binder_proc *proc,
			       struct binder_thread *thread,
			       struct binder_transaction_data *tr, int reply)
{
	struct binder_transaction *t; //一个transaction操作
	struct binder_work *tcomplete; //表示一个未完成的操作。因为一个transaction操作涉及两个进程，A向B发送请求后，等待B执行，此时对于A就是未完成的操作
	binder_size_t *offp, *off_end;
	binder_size_t off_min;
	struct binder_proc *target_proc; //目标进程，比如Service Manager
	struct binder_thread *target_thread = NULL; //目标线程
	struct binder_node *target_node = NULL; //目标Binder节点
	struct list_head *target_list; //目标todo列表
	wait_queue_head_t *target_wait; //目标等待列表
	struct binder_transaction *in_reply_to = NULL;
	struct binder_transaction_log_entry *e;
	uint32_t return_error;

	e = binder_transaction_log_add(&binder_transaction_log);
	...//日志相关
```
```
	if (reply) { //BC_REPLY的情况，比如SM查询到server句柄后，回复到Binder驱动
        //SM被唤醒后，执行binder_thread_read，会把thread->transaction_stack设置为Binder客户端在binder_thread_write函数
		//中放到SM对应的todo列表的t，
		in_reply_to = thread->transaction_stack;
		...
		binder_set_nice(in_reply_to->saved_priority);
		...
		thread->transaction_stack = in_reply_to->to_parent;
		target_thread = in_reply_to->from;//获取目标所在线程
		...
		target_proc = target_thread->proc;//获取目标所在进程
	}
```


上面是对于`BC_REPLY`的处理，下面则是对于`BC_TRANSACTION`的处理

```
	 else { //BC_TRANSACTION的情况
		if (tr->target.handle) { //handle不为0的情况，也就是非Service Mangager
			struct binder_ref *ref;

			ref = binder_get_ref(proc, tr->target.handle);
			...//异常处理
			target_node = ref->node;
		} else { //handle为0的情况，也就是Service Mangager
			target_node = binder_context_mgr_node;
			...//异常处理
		}
```
`BC_TRANSACTION`处理的第一步：获取`target_node`。如果不是Service Mangager，则需要调用binder_get_ref来查找node，如果是Service Mangager，就直接使用全局变量`binder_context_mgr_node`（这个变量在SM启动的时候调用`binder_become_context_manager`时就创建了）

```
		e->to_node = target_node->debug_id;
		target_proc = target_node->proc;
		...
		if (!(tr->flags & TF_ONE_WAY) && thread->transaction_stack) {
			struct binder_transaction *tmp;
			tmp = thread->transaction_stack;
			...
			while (tmp) {
				if (tmp->from && tmp->from->proc == target_proc)
					target_thread = tmp->from;
				tmp = tmp->from_parent;
			}
		}
	}
```
`BC_TRANSACTION`处理的第二步：得到`target_proc`和`target_thread`


下面开始是处理`BC_REPLY`和`BC_TRANSACTION`的公共部分
```
	if (target_thread) {
		e->to_thread = target_thread->pid;
		target_list = &target_thread->todo;
		target_wait = &target_thread->wait;
	} else {
		target_list = &target_proc->todo;
		target_wait = &target_proc->wait;
	}
	e->to_proc = target_proc->pid;
```
`BC_REPLY`、`BC_TRANSACTION`处理的第三步：根据`target_proc`和`target_thread`，得到`target_list`和`target_wait`。

```

	/* TODO: reuse incoming transaction for reply */
	t = kzalloc(sizeof(*t), GFP_KERNEL);
	...
	binder_stats_created(BINDER_STAT_TRANSACTION);
```
`BC_REPLY`、`BC_TRANSACTION`处理的第四步：生成`binder_transaction`变量t，用于描述本次transaction（最后会加入对方的target_thread->todo中）。当目标对象被唤醒时，就会从这个队列中取出将要做的事务。

```
	tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
	...
	binder_stats_created(BINDER_STAT_TRANSACTION_COMPLETE);
	...
```
`BC_REPLY`、`BC_TRANSACTION`处理的第五步：生成`binder_work`变量tcomplete，说明当前调用者线程有一个transaction未完成，tcomplete将添加到本线程的todo队列

对于`BC_REPLY`，发起者是SM，target是Binder Client，对于`BC_TRANSACTION`，发起者是Binder Client，target是SM


```
	...
	if (!reply && !(tr->flags & TF_ONE_WAY))
		t->from = thread;
	else
		t->from = NULL;
	t->sender_euid = task_euid(proc->tsk);
	t->to_proc = target_proc;
	t->to_thread = target_thread;
	t->code = tr->code;
	t->flags = tr->flags;
	t->priority = task_nice(current);

	trace_binder_transaction(reply, t, target_node);

	t->buffer = binder_alloc_buf(target_proc, tr->data_size,
		tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));
	if (t->buffer == NULL) {
		return_error = BR_FAILED_REPLY;
		goto err_binder_alloc_buf_failed;
	}
	t->buffer->allow_user_free = 0;
	t->buffer->debug_id = t->debug_id;
	t->buffer->transaction = t;
	t->buffer->target_node = target_node;
	trace_binder_transaction_alloc_buf(t->buffer);
	if (target_node)
		binder_inc_node(target_node, 1, 0, NULL);
```
`BC_REPLY`、`BC_TRANSACTION`处理的第六步：对`binder_transaction`变量的各个数据赋值：
* t->from 表示transaction的发起者
* t->to_proc和t->to_thread 表示目标进程和线程
* t->code 对Binder Server的请求码。getService()服务对应的是GET_SERVICE_TRANSACATION
* t->buffer 为本次transaction申请的内存，从目标进程分配，是Binder驱动的mmap管理的内存区域


```
	offp = (binder_size_t *)(t->buffer->data +
				 ALIGN(tr->data_size, sizeof(void *)));
	// 直接复制到目标进程中
	if (copy_from_user(t->buffer->data, (const void __user *)(uintptr_t)
			   tr->data.ptr.buffer, tr->data_size)) {
		binder_user_error("%d:%d got transaction with invalid data ptr\n",
				proc->pid, thread->pid);
		return_error = BR_FAILED_REPLY;
		goto err_copy_data_failed;
	}
	if (copy_from_user(offp, (const void __user *)(uintptr_t)
			   tr->data.ptr.offsets, tr->offsets_size)) {
		binder_user_error("%d:%d got transaction with invalid offsets ptr\n",
				proc->pid, thread->pid);
		return_error = BR_FAILED_REPLY;
		goto err_copy_data_failed;
	}
	if (!IS_ALIGNED(tr->offsets_size, sizeof(binder_size_t))) {
		binder_user_error("%d:%d got transaction with invalid offsets size, %lld\n",
				proc->pid, thread->pid, (u64)tr->offsets_size);
		return_error = BR_FAILED_REPLY;
		goto err_bad_offset;
	}
```
`BC_REPLY`、`BC_TRANSACTION`处理的第七步：前面申请到t->buffer内存后，就把先前从调用者进程用户空间复制到内核空间的数据tr的data.ptr.offsets指向的用户空间数据复制到t->buffer指向的内存，t->buffer指向的内存空间和目标进程是共享的，所以也就是一次复制就可以把Binder客户端的数据复制到Binder服务端。


```
	off_end = (void *)offp + tr->offsets_size;
	off_min = 0;
	for (; offp < off_end; offp++) {
		struct flat_binder_object *fp;

		if (*offp > t->buffer->data_size - sizeof(*fp) ||
		    *offp < off_min ||
		    t->buffer->data_size < sizeof(*fp) ||
		    !IS_ALIGNED(*offp, sizeof(u32))) {
			binder_user_error("%d:%d got transaction with invalid offset, %lld (min %lld, max %lld)\n",
					  proc->pid, thread->pid, (u64)*offp,
					  (u64)off_min,
					  (u64)(t->buffer->data_size -
					  sizeof(*fp)));
			return_error = BR_FAILED_REPLY;
			goto err_bad_offset;
		}
		fp = (struct flat_binder_object *)(t->buffer->data + *offp);
		off_min = *offp + sizeof(struct flat_binder_object);
		switch (fp->type) {
		case BINDER_TYPE_BINDER:
		case BINDER_TYPE_WEAK_BINDER: {
			struct binder_ref *ref;
			//当前进程注册服务（注册自己）时，查询自己的binder_proc中是否已有自己的BpBinder对应的binder_node
			//fp->binder也就是BpBinder的引用
			struct binder_node *node = binder_get_node(proc, fp->binder);

			if (node == NULL) {
				//自己的BpBinder在bind_proc中的nodes中还没有时，就添加新的。fp->cookie是BpBinder的地址，
				node = binder_new_node(proc, fp->binder, fp->cookie);
				if (node == NULL) {
					return_error = BR_FAILED_REPLY;
					goto err_binder_new_node_failed;
				}
				node->min_priority = fp->flags & FLAT_BINDER_FLAG_PRIORITY_MASK;
				node->accept_fds = !!(fp->flags & FLAT_BINDER_FLAG_ACCEPTS_FDS);
			}
			...
			//注册服务的进程创建binder_ref，binder_ref中记录了服务的binder_node，binder_ref->proc设为SM
			//分配binder_ref.desc值，也就是句柄handle值，并放到SM对应的bind_proc的refs_by_desc中
			ref = binder_get_ref_for_node(target_proc, node);
			if (ref == NULL) {
				return_error = BR_FAILED_REPLY;
				goto err_binder_get_ref_for_node_failed;
			}
			if (fp->type == BINDER_TYPE_BINDER)
				fp->type = BINDER_TYPE_HANDLE;
			else
				fp->type = BINDER_TYPE_WEAK_HANDLE;
			fp->handle = ref->desc;//分配好的句柄
			binder_inc_ref(ref, fp->type == BINDER_TYPE_HANDLE,
				       &thread->todo);

			trace_binder_transaction_node_to_ref(t, node, ref);
			binder_debug(BINDER_DEBUG_TRANSACTION,
				     "        node %d u%016llx -> ref %d desc %d\n",
				     node->debug_id, (u64)node->ptr,
				     ref->debug_id, ref->desc);
		} break;
		case BINDER_TYPE_HANDLE://SM在回复Client查询服务时，设置type为BINDER_TYPE_HANDLE
		case BINDER_TYPE_WEAK_HANDLE: {
			//通过句柄，从SM对应的proc->refs_by_desc中查到binder_ref（Binder Service注册的时候设置的）
			struct binder_ref *ref = binder_get_ref(proc, fp->handle);
			...
			if (ref->node->proc == target_proc) {//如果是相同进程
				if (fp->type == BINDER_TYPE_HANDLE)
					fp->type = BINDER_TYPE_BINDER;//直接使用内存地址
				else
					fp->type = BINDER_TYPE_WEAK_BINDER;
				fp->binder = ref->node->ptr;
				fp->cookie = ref->node->cookie;
				binder_inc_node(ref->node, fp->type == BINDER_TYPE_BINDER, 0, NULL);
				...
			} else {//一般都不是一个进程
				struct binder_ref *new_ref;
				//SM创建一个binder_ref，引用之前注册服务的进程的BpBinder的node，分配handle，
				//放到目标进程对应的bind_proc的refs_by_desc中
				new_ref = binder_get_ref_for_node(target_proc, ref->node);
				...
				fp->handle = new_ref->desc;//设置handle
				binder_inc_ref(new_ref, fp->type == BINDER_TYPE_HANDLE, NULL);
				...
			}
		} break;

		case BINDER_TYPE_FD: {
			int target_fd;
			struct file *file;

			if (reply) {
				if (!(in_reply_to->flags & TF_ACCEPT_FDS)) {
					binder_user_error("%d:%d got reply with fd, %d, but target does not allow fds\n",
						proc->pid, thread->pid, fp->handle);
					return_error = BR_FAILED_REPLY;
					goto err_fd_not_allowed;
				}
			} else if (!target_node->accept_fds) {
				binder_user_error("%d:%d got transaction with fd, %d, but target does not allow fds\n",
					proc->pid, thread->pid, fp->handle);
				return_error = BR_FAILED_REPLY;
				goto err_fd_not_allowed;
			}

			file = fget(fp->handle);
			if (file == NULL) {
				binder_user_error("%d:%d got transaction with invalid fd, %d\n",
					proc->pid, thread->pid, fp->handle);
				return_error = BR_FAILED_REPLY;
				goto err_fget_failed;
			}
			if (security_binder_transfer_file(proc->tsk, target_proc->tsk, file) < 0) {
				fput(file);
				return_error = BR_FAILED_REPLY;
				goto err_get_unused_fd_failed;
			}
			target_fd = task_get_unused_fd_flags(target_proc, O_CLOEXEC);
			if (target_fd < 0) {
				fput(file);
				return_error = BR_FAILED_REPLY;
				goto err_get_unused_fd_failed;
			}
			task_fd_install(target_proc, target_fd, file);
			trace_binder_transaction_fd(t, fp->handle, target_fd);
			binder_debug(BINDER_DEBUG_TRANSACTION,
				     "        fd %d -> %d\n", fp->handle, target_fd);
			/* TODO: fput? */
			fp->handle = target_fd;
		} break;

		default:
			binder_user_error("%d:%d got transaction with invalid object type, %x\n",
				proc->pid, thread->pid, fp->type);
			return_error = BR_FAILED_REPLY;
			goto err_bad_object_type;
		}
	}
```
以上处理数据中的`binder_object`对象，包括查找、设置Binder Server的句柄等

```
	if (reply) {
		BUG_ON(t->buffer->async_transaction != 0);
		binder_pop_transaction(target_thread, in_reply_to);
	} else if (!(t->flags & TF_ONE_WAY)) {
		BUG_ON(t->buffer->async_transaction != 0);
		t->need_reply = 1; //1表示是同步事务，需要等待回复；0表示异步
		//设置本次binder_transaction的from_parent为当前binder_thread的transaction_stack（binder_transaction类型）
		t->from_parent = thread->transaction_stack;
		thread->transaction_stack = t;//设置当前binder_thread的transaction_stack为本次binder_transaction
	} else {
		BUG_ON(target_node == NULL);
		BUG_ON(t->buffer->async_transaction != 1);
		if (target_node->has_async_transaction) {
			target_list = &target_node->async_todo;
			target_wait = NULL;
		} else
			target_node->has_async_transaction = 1;
	}

```
1. if 分支：事务出栈
2. else if ：对于getService的情况，执行第二个分支。没有指定TF_ONE_WAY，就将`need_reply`设为1，并记录本次`binder_transaction`变量t，用于后期查询。
3. else 分支：异步


继续执行后续代码：
```
	t->work.type = BINDER_WORK_TRANSACTION;
	list_add_tail(&t->work.entry, target_list);//加入对方的处理队列中
	tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
	list_add_tail(&tcomplete->entry, &thread->todo);//表示当前线程有一个待完成操作
	if (target_wait)
		wake_up_interruptible(target_wait);//唤醒目标
	return;

...//错误处理
}
```
这里把t加入了目标的todo列表，把tcomplete加入到调用者自身的todo列表，然后开始唤醒目标对象，让它处理target_list中的任务。

对于getService的情况，到这里Service Manager被唤醒，然后系统分为Binder客户端和服务端分别执行。

对于SM响应的情况，到这里Binder客户端被唤醒


#### binder_thread_read
在`binder_ioctl_write_read`函数中，如果有数据需要写，就执行`binder_thread_write`后；如果有数据需要读就执行`binder_thread_read`。
```
static int binder_thread_read(struct binder_proc *proc,
			      struct binder_thread *thread,
			      binder_uintptr_t binder_buffer, size_t size,
			      binder_size_t *consumed, int non_block)
{
	void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
	void __user *ptr = buffer + *consumed;
	void __user *end = buffer + size;

	int ret = 0;
	int wait_for_proc_work;

	if (*consumed == 0) {
		if (put_user(BR_NOOP, (uint32_t __user *)ptr))
			return -EFAULT;
		ptr += sizeof(uint32_t);
	}
```
`proc`和`thread`分别是调用者进程和线程，`binder_buffer`是`binder_write_read bwr`中存储的`mIn`的地址，用于接收Binder驱动的回复数据。`size`和`consumed`也都是`bwr`中数据。

`put_user`函数是简单数据版本的`copy_to_user`。ptr和end是数据的起点和终点位置。如果bwr.read_consumed为0，则写入BR_NOOP

```
retry:
    //在binder_thread_write中会设置thread->transaction_stack
    wait_for_proc_work = thread->transaction_stack == NULL &&
				list_empty(&thread->todo);
	...
	if (wait_for_proc_work) {
		...
		binder_set_nice(proc->default_priority);
		if (non_block) {
			if (!binder_has_proc_work(proc, thread))
				ret = -EAGAIN;
		} else
		    //比如SM会在这里睡眠，然后等待客户端唤醒
			ret = wait_event_freezable_exclusive(proc->wait, binder_has_proc_work(proc, thread));
	} else {//对于getService，第二次执行`talkWithDriver()`，wait_for_proc_work为false，会执行这里
		if (non_block) {
			if (!binder_has_thread_work(thread))
				ret = -EAGAIN;
		} else
			ret = wait_event_freezable(thread->wait, binder_has_thread_work(thread));
			//进入等待，等待Service Manager唤醒，SM响应后，客户端在这里开始继续执行
	}

	binder_lock(__func__);

	if (wait_for_proc_work)
		proc->ready_threads--;
	thread->looper &= ~BINDER_LOOPER_STATE_WAITING;

	if (ret)
		return ret;
	...
	while (1) {
		uint32_t cmd;
		struct binder_transaction_data tr;
		struct binder_work *w;
		struct binder_transaction *t = NULL;

		if (!list_empty(&thread->todo)) {
			w = list_first_entry(&thread->todo, struct binder_work,
					     entry);
		} else if (!list_empty(&proc->todo) && wait_for_proc_work) {
			w = list_first_entry(&proc->todo, struct binder_work,
					     entry);
		} else {
			/* no data added */
			if (ptr - buffer == 4 &&
			    !(thread->looper & BINDER_LOOPER_STATE_NEED_RETURN))
				goto retry;
			break;
		}
```
在`binder_thread_write`的最后，把`binder_work`变量tcomplete放到了调用者的`thread->todo`中，所以这里得到类型为`BINDER_WORK_TRANSACTION_COMPLETE`的w。

```

		if (end - ptr < sizeof(tr) + 4)
			break;

		switch (w->type) {
		case BINDER_WORK_TRANSACTION: {
			//客户端在binder_transaction的最后会把type为BINDER_WORK_TRANSACTIONSM的t放到目标队列，服务端
			//在被唤醒后就会执行到这里。
			//对于服务端是SM然后会执行switch-case语句之后的代码，会把用户请求复制到SM中并对各种队列进行调整
			//结束后回到servicemanager/binder.c的。
			//对于发起端是SM，服务端是普通进程的情况，通过w在binder_transaction中的位置取得t
			t = container_of(w, struct binder_transaction, work);
		} break;
		case BINDER_WORK_TRANSACTION_COMPLETE: {
			cmd = BR_TRANSACTION_COMPLETE; //
			if (put_user(cmd, (uint32_t __user *)ptr)) //写入cmd到用户空间的mIn
				return -EFAULT;
			ptr += sizeof(uint32_t);
			binder_stat_br(proc, thread, cmd); //统计信息
			...
			list_del(&w->entry);//从事务队列中删除本次binder_work
			kfree(w);//释放内存
			binder_stats_deleted(BINDER_STAT_TRANSACTION_COMPLETE);
		} break;
		case BINDER_WORK_NODE: ...
		case BINDER_WORK_DEAD_BINDER:
		case BINDER_WORK_DEAD_BINDER_AND_CLEAR:
		case BINDER_WORK_CLEAR_DEATH_NOTIFICATION: ...
		}

		if (!t) //只有BINDER_WORK_TRANSACTION命令，即t不为空才能继续往下执行
			continue;

		BUG_ON(t->buffer == NULL);
		//判断事务t中是否有目标进程的Binder实体
		if (t->buffer->target_node) {
			struct binder_node *target_node = t->buffer->target_node;

			tr.target.ptr = target_node->ptr;
			tr.cookie =  target_node->cookie;
			t->saved_priority = task_nice(current);
			if (t->priority < target_node->min_priority &&
			    !(t->flags & TF_ONE_WAY))
				binder_set_nice(t->priority);
			else if (!(t->flags & TF_ONE_WAY) ||
				 t->saved_priority > target_node->min_priority)
				binder_set_nice(target_node->min_priority);
			cmd = BR_TRANSACTION;//设置BR_TRANSACTION命令
		} else {
			tr.target.ptr = 0;
			tr.cookie = 0;
			cmd = BR_REPLY;//设置BR_REPLY命令
		}
		tr.code = t->code;
		tr.flags = t->flags;
		tr.sender_euid = from_kuid(current_user_ns(), t->sender_euid);

		if (t->from) {
			struct task_struct *sender = t->from->proc->tsk;

			tr.sender_pid = task_tgid_nr_ns(sender,
							task_active_pid_ns(current));
		} else {
			tr.sender_pid = 0;
		}

		tr.data_size = t->buffer->data_size;
		tr.offsets_size = t->buffer->offsets_size;
		tr.data.ptr.buffer = (binder_uintptr_t)(
					(uintptr_t)t->buffer->data +
					proc->user_buffer_offset);//得到数据的用户空间地址
		tr.data.ptr.offsets = tr.data.ptr.buffer +
					ALIGN(t->buffer->data_size,
					    sizeof(void *));

        //将cmd和数据写回用户空间的mIn
		if (put_user(cmd, (uint32_t __user *)ptr))
			return -EFAULT;
		ptr += sizeof(uint32_t);
		//事务数据写回用户空间的mIn
		if (copy_to_user(ptr, &tr, sizeof(tr)))
			return -EFAULT;
		ptr += sizeof(tr);

		trace_binder_transaction_received(t);
		binder_stat_br(proc, thread, cmd);
		...

		list_del(&t->work.entry);
		t->buffer->allow_user_free = 1;
		if (cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)) {
			t->to_parent = thread->transaction_stack;
			t->to_thread = thread;
			thread->transaction_stack = t;
		} else {
			t->buffer->transaction = NULL;
			kfree(t);
			binder_stats_deleted(BINDER_STAT_TRANSACTION);
		}
		break;
	}

done:

	*consumed = ptr - buffer;
	if (proc->requested_threads + proc->ready_threads == 0 &&
	    proc->requested_threads_started < proc->max_threads &&
	    (thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
	     BINDER_LOOPER_STATE_ENTERED)) /* the user-space code fails to */
	     /*spawn a new thread if we leave this out */) {
		proc->requested_threads++;
		...
		//创建新线程
		if (put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer))
			return -EFAULT;
		binder_stat_br(proc, thread, BR_SPAWN_LOOPER);
	}
	return 0;
}
```
上面看到处理`BINDER_WORK_TRANSACTION_COMPLETE`的情况，向用户空间mIn中写入了`BR_TRANSACTION_COMPLETE`命令，这是Binder驱动回复调用者事务完成的命令。

ioctl把用户空间的bwr复制到内核空间后，在`binder_thread_write`和`binder_thread_read`中会对该数据读写，然后在`binder_ioctl_write_read`函数的最后把数据复制回用户空间。从`IPCThreadState::talkWithDriver`调用ioctl到这里就结束了，然后就回到了`IPCThreadState::waitForResponse`函数中的`talkWithDriver`的下一行。