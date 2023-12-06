---
title: "Binder Driver探索"
date:  2023-12-06T10:36:27+08:00
draft: true
categories: [android]
tags:
  - android
  - binder
---

## binder驱动的初始化

在binder.c中有以下一行代码；

```c
device_initcall(binder_init);
```

在Linux内核的启动过程中，一个驱动的注册用module_init调用，即device_initcall，它可以将驱动设备加载进内核中，以供后续使用。

在Android8.0之后，现在Binder驱动有三个：/dev/binder; /dev/hwbinder; /dev/vndbinder.

```c
static int __init binder_init(void)
{
  int ret;
  char *device_name, *device_tmp;
  struct binder_device *device;
  struct hlist_node *tmp;
  char *device_names = NULL;
  //初始化binder缓冲区分配
  ret = binder_alloc_shrinker_init();
  if (ret)
    return ret;
  // ~0U：无符号整型，对0取反。
  atomic_set(&binder_transaction_log.cur, ~0U);
  atomic_set(&binder_transaction_log_failed.cur, ~0U);
  // 创建/sys/kernel/debug/binder目录。
  binder_debugfs_dir_entry_root = debugfs_create_dir("binder", NULL);
  // 创建/sys/kernel/debug/binder/proc目录用于记录每个进程基本信息。
  if (binder_debugfs_dir_entry_root)
    binder_debugfs_dir_entry_proc = debugfs_create_dir("proc",
             binder_debugfs_dir_entry_root);

  if (binder_debugfs_dir_entry_root) {
    // 创建/sys/kernel/debug/binder/state文件用于记录状态信息，
    //并注册操作函数binder_state_fops。
    debugfs_create_file("state",
            0444,
            binder_debugfs_dir_entry_root,
            NULL,
            &binder_state_fops);
    // 创建/sys/kernel/debug/binder/stats文件用于记录统计信息，
    //并注册操作函数binder_stats_fops。
    debugfs_create_file("stats",
            0444,
            binder_debugfs_dir_entry_root,
            NULL,
            &binder_stats_fops);
     // 创建/sys/kernel/debug/binder/transactions文件用于记录transaction相关信息，
     //并注册操作函数binder_transactions_fops。
    debugfs_create_file("transactions",
            0444,
            binder_debugfs_dir_entry_root,
            NULL,
            &binder_transactions_fops);
    // 创建/sys/kernel/debug/binder/transaction_log文件用于记录transaction日志相关信息，
    //并注册操作函数binder_transaction_log_fops。
    debugfs_create_file("transaction_log",
            0444,
            binder_debugfs_dir_entry_root,
            &binder_transaction_log,
            &binder_transaction_log_fops);
    // 创建/sys/kernel/debug/binder/failed_transaction_log文件用于记录transaction失败日志相关信息，
    // 并注册操作函数binder_transaction_log_fops
    debugfs_create_file("failed_transaction_log",
            0444,
            binder_debugfs_dir_entry_root,
            &binder_transaction_log_failed,
            &binder_transaction_log_fops);
  }

  if (!IS_ENABLED(CONFIG_ANDROID_BINDERFS) &&
      strcmp(binder_devices_param, "") != 0) {
    /*
    * Copy the module_parameter string, because we don't want to
    * tokenize it in-place.
     */
    // kzalloc：分配不超过128KB的连续的物理内存映射区域。
      // GFP_KERNEL：内存分配器flags，无内存可用时可引起休眠，允许启动磁盘IO和文件系统IO。
      // binder_devices_param：binder，hwbinder，vndbinder。
    device_names = kstrdup(binder_devices_param, GFP_KERNEL);
    if (!device_names) {
      ret = -ENOMEM;
      goto err_alloc_device_names_failed;
    }
     // 创建binder设备
    device_tmp = device_names;
    while ((device_name = strsep(&device_tmp, ","))) {
      ret = init_binder_device(device_name);
      if (ret)
        goto err_init_binder_device_failed;
    }
  }
  //初始化binder文件系统
  ret = init_binderfs();
  if (ret)
    goto err_init_binder_device_failed;

  return ret;

err_init_binder_device_failed:
  hlist_for_each_entry_safe(device, tmp, &binder_devices, hlist) {
    misc_deregister(&device->miscdev);
    hlist_del(&device->hlist);
    kfree(device);
  }

  kfree(device_names);

err_alloc_device_names_failed:
  debugfs_remove_recursive(binder_debugfs_dir_entry_root);

  return ret;
}
```

`binder_init`函数主要内容是：

- 初始化binder缓冲区分配
- 创建了sys/kernel/debug/binder目录，以及其子目录或文件
- 注册misc设备，创建binder设备
- 把binder_device加入到全局链表binder_devices进行管理

那来看看`init_binder_device`是如何注册misc设备，创建binder设备，代码如下：

```c
static int __init init_binder_device(const char *name)
{
  int ret;
  struct binder_device *binder_device;

  binder_device = kzalloc(sizeof(*binder_device), GFP_KERNEL);
  if (!binder_device)
    return -ENOMEM;
  //miscdevice结构体
  binder_device->miscdev.fops = &binder_fops; //设备的文件操作结构，这是file_operations结构
  binder_device->miscdev.minor = MISC_DYNAMIC_MINOR;//次设备号 动态分配
  binder_device->miscdev.name = name;//设备名
  //binder设备的引用计数
  refcount_set(&binder_device->ref, 1);
  //默认binder驱动的上下文管理者
  binder_device->context.binder_context_mgr_uid = INVALID_UID;
  binder_device->context.name = name;
  mutex_init(&binder_device->context.context_mgr_node_lock);
  // 注册misc设备
  ret = misc_register(&binder_device->miscdev);
  if (ret < 0) {
    kfree(binder_device);
    return ret;
  }
  // 通过全局链表binder_devices管理binder_device。
  hlist_add_head(&binder_device->hlist, &binder_devices);

  return ret;
}
```

从`init_binder_device`函数看出binder驱动设备节点是通过`binder_device`结构体管理的；设置`binder_device`的`miscdev`参数，`miscdev`其实是`miscdevice`结构体，misc_register函数注册misc设备，`miscdevice`参数分别是：

1. 设备的文件操作结构，这是file_operations结构
2. 次设备号 动态分配
3. 设备名

`binder_device`结构体定义如下：

```c
struct binder_device {
  // 加入binder_devices全局链表的node。
  struct hlist_node hlist;
    // misc设备。
  struct miscdevice miscdev;
   // 获取service manager对应的binder_node。
  struct binder_context context;
  //属于bindfs挂载的超级块的根节点的inode。
  struct inode *binderfs_inode;
  //binder_device的引用计数
  refcount_t ref;
};
```

`file_operations`结构体,指定相应文件操作的方法

```c
const struct file_operations binder_fops = {
  .owner = THIS_MODULE,
  .poll = binder_poll,
  .unlocked_ioctl = binder_ioctl,
  .compat_ioctl = compat_ptr_ioctl,
  .mmap = binder_mmap,
  .open = binder_open,
  .flush = binder_flush,
  .release = binder_release,
};
```

用户态的程序调用Kernel层驱动是需要陷入内核态，进行系统调用(`syscall`)，比如打开Binder驱动方法的调用链为： open-> __open() -> binder_open()。通过`binder_fops`的定义得出以下调用规则；

![binder_syscall](./../img/binder_syscall.png)

[图片来源](http://gityuan.com/2015/11/01/binder-driver/)

## binder_open

之前已经提到每个进程都会单独创建自己的`ProcessState`，`ProcessState`是进程唯一的；在`ProcessState`创建时会调用`open`函数，那对应调用的就是binder驱动中的`binder_open`;

```c
static int binder_open(struct inode *nodp, struct file *filp)
{
  struct binder_proc *proc, *itr;
  struct binder_device *binder_dev;
  struct binderfs_info *info;
  struct dentry *binder_binderfs_dir_entry_proc = NULL;
  bool existing_pid = false;

  binder_debug(BINDER_DEBUG_OPEN_CLOSE, "%s: %d:%d\n", __func__,
         current->group_leader->pid, current->pid);
  //创建binder驱动中管理IPC和保存进程信息的根结构体
  proc = kzalloc(sizeof(*proc), GFP_KERNEL);
  if (proc == NULL)
    return -ENOMEM;
  // 初始化两个自旋锁。
    // inner_lock保护线程、binder_node以及所有与进程相关的的todo队列。
    // outer_lock保护binder_ref。
  spin_lock_init(&proc->inner_lock);
  spin_lock_init(&proc->outer_lock);
  // 获取当前进程组领头进程。
  get_task_struct(current->group_leader);
  proc->tsk = current->group_leader;//将当前进程的task_struct保存到binder_proc
  INIT_LIST_HEAD(&proc->todo);//初始化todo列表
  // 判断当前进程的调度策略是否支持，binder只支持SCHED_NORMAL(00b)、SCHED_FIFO(01b)、SCHED_RR(10b)、SCHED_BATCH(11b)。
    // prio为进程优先级，可通过normal_prio获取。一般分为实时优先级及静态优先级。
  if (binder_supported_policy(current->policy)) {
    proc->default_priority.sched_policy = current->policy;
    proc->default_priority.prio = current->normal_prio;
  } else {
    proc->default_priority.sched_policy = SCHED_NORMAL;
    proc->default_priority.prio = NICE_TO_PRIO(0);
  }

  /* binderfs stashes devices in i_private */
   // 通过miscdev获取binder_device。
  if (is_binderfs_device(nodp)) {
    binder_dev = nodp->i_private;
    info = nodp->i_sb->s_fs_info;
    binder_binderfs_dir_entry_proc = info->proc_log_dir;
  } else {
    binder_dev = container_of(filp->private_data,
            struct binder_device, miscdev);
  }
  //binder_device的引用计数加1
  refcount_inc(&binder_dev->ref);
  //初始化对应进程中的binder驱动上下文管理者
  proc->context = &binder_dev->context;
  // 初始化binder_proc的binder_alloc字段。
  binder_alloc_init(&proc->alloc);

  // binder驱动维护静态全局数组binder_stats，其中有一个成员数组obj_created。
    // 当binder_open调用时，obj_created[BINDER_STAT_PROC]将自增。该数组用来统计binder对象的数量。
  binder_stats_created(BINDER_STAT_PROC);
  //初始化binder_proc的pid为领头进程的pid值。
  proc->pid = current->group_leader->pid;
  // 初始化delivered_death及waiting_threads队列。
  INIT_LIST_HEAD(&proc->delivered_death);
  INIT_LIST_HEAD(&proc->waiting_threads);
  // private_data保存binder_proc对象。
  filp->private_data = proc;
  // 将binder_proc加入到全局队列binder_procs中,该操作必须加锁。
  mutex_lock(&binder_procs_lock);
  hlist_for_each_entry(itr, &binder_procs, proc_node) {
    if (itr->pid == proc->pid) {
      existing_pid = true;
      break;
    }
  }
  hlist_add_head(&proc->proc_node, &binder_procs);
  mutex_unlock(&binder_procs_lock);
    // 若/sys/kernel/binder/proc目录已经创建好，则在该目录下创建一个以pid为名的文件。
  if (binder_debugfs_dir_entry_proc && !existing_pid) {
    char strbuf[11];

    snprintf(strbuf, sizeof(strbuf), "%u", proc->pid);
    /*
     * proc debug entries are shared between contexts.
     * Only create for the first PID to avoid debugfs log spamming
     * The printing code will anyway print all contexts for a given
     * PID so this is not a problem.
     */
    // proc调试条目在上下文之间共享，如果进程尝试使用其他上下文再次打开驱动程序，则此操作将失败。
    proc->debugfs_entry = debugfs_create_file(strbuf, 0444,
      binder_debugfs_dir_entry_proc,
      (void *)(unsigned long)proc->pid,
      &proc_fops);
  }

  if (binder_binderfs_dir_entry_proc && !existing_pid) {
    char strbuf[11];
    struct dentry *binderfs_entry;

    snprintf(strbuf, sizeof(strbuf), "%u", proc->pid);
    /*
     * Similar to debugfs, the process specific log file is shared
     * between contexts. Only create for the first PID.
     * This is ok since same as debugfs, the log file will contain
     * information on all contexts of a given PID.
     */
    binderfs_entry = binderfs_create_file(binder_binderfs_dir_entry_proc,
      strbuf, &proc_fops, (void *)(unsigned long)proc->pid);
    if (!IS_ERR(binderfs_entry)) {
      proc->binderfs_entry = binderfs_entry;
    } else {
      int error;

      error = PTR_ERR(binderfs_entry);
      pr_warn("Unable to create file %s in binderfs (error %d)\n",
        strbuf, error);
    }
  }

  return 0;
}
```

从`binder_open`函数的主要工作是创建`binder_proc`结构体，并把当前进程等信息保存到`binder_proc`，初始化`binder_proc`中管理IPC所需的各种信息并创建其它相关的子结构体；再把`binder_proc`保存到文件指针`filp`，以及把`binder_proc`加入到全局链表`binder_procs`，这一个双向链表结构。

### binder_procs结构体

```c
struct binder_proc {
  //加入binder_procs全局链表的node节点。
  struct hlist_node proc_node;
  //记录执行传输动作的线程信息, binder_thread红黑树的根节点
  struct rb_root threads;
  //用于记录binder实体  ,binder_node红黑树的根节点，它是Server在Binder驱动中的体现
  struct rb_root nodes;
  //binder_ref红黑树的根节点(以handle为key
  struct rb_root refs_by_desc;
  //binder_ref红黑树的根节点（以ptr为key）
  struct rb_root refs_by_node;
  //该binder进程的线程池中等待处理binder_work的binder_thread链表
  struct list_head waiting_threads;
  //相应进程id
  int pid;
  //相应进程的task结构体
  struct task_struct *tsk;

  struct hlist_node deferred_work_node;
  int deferred_work;
  bool is_dead;

  //进程将要做的事
  struct list_head todo;
  //binder统计信息
  struct binder_stats stats;
  //已分发的死亡通知
  struct list_head delivered_death;
  //最大线程数
  int max_threads;
  //请求的线程数
  int requested_threads;
  //已启动的请求线程数
  int requested_threads_started;
  int tmp_ref;
  //默认优先级
  struct binder_priority default_priority;
  struct dentry *debugfs_entry;
   //进程通信数据内存分配相关
  struct binder_alloc alloc;
  //binder驱动的上下文管理者
  struct binder_context *context;
  spinlock_t inner_lock;
  spinlock_t outer_lock;
  struct dentry *binderfs_entry;
};
struct binder_alloc {
  struct mutex mutex;
  //指向进程虚拟地址空间的指针
  struct vm_area_struct *vma;
  //相应进程的内存结构体
  struct mm_struct *vma_vm_mm;
  // map 的地址就是这里了
  void __user *buffer;
  //所有的buffers列表
  struct list_head buffers;
  //只进行了预定，没有分配，按大小排序
  struct rb_root free_buffers;
  //已经分配了,按地址排序
  struct rb_root allocated_buffers;
  //用于异步请求的空间
  size_t free_async_space;
  //所有的pages指向物理内存页
  struct binder_lru_page *pages;
  //映射的内核空间大小
  size_t buffer_size;
  uint32_t buffer_free;
  int pid;
  size_t pages_high;
};
```

## binder_mmap

主要功能：

1. 为用户进程分配一块内核空间作为缓冲区，并把分配的缓冲区指针存放到binder_proc的buffer字段；
2. 分配pages空间，并内核分配一块同样页数的内核空间,并把它的物理内存和前面为用户进程分配的内存地址关联；
3. 分配的内存块加入用户进程内存链表；

```c
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
  //private_data保存了我们open设备时创建的binder_proc信息
  struct binder_proc *proc = filp->private_data;

  if (proc->tsk != current->group_leader)
    return -EINVAL;

  binder_debug(BINDER_DEBUG_OPEN_CLOSE,
         "%s: %d %lx-%lx (%ld K) vma %lx pagep %lx\n",
         __func__, proc->pid, vma->vm_start, vma->vm_end,
         (vma->vm_end - vma->vm_start) / SZ_1K, vma->vm_flags,
         (unsigned long)pgprot_val(vma->vm_page_prot));
  //mmap 的 buffer 禁止用户进行写操作。mmap 只是为了分配内核空间，传递数据通过 ioctl()
  if (vma->vm_flags & FORBIDDEN_MMAP_FLAGS) {
    pr_err("%s: %d %lx-%lx %s failed %d\n", __func__,
           proc->pid, vma->vm_start, vma->vm_end, "bad vm_flags", -EPERM);
    return -EPERM;
  }
  // 将 VM_DONTCOP 置起，禁止 拷贝，禁止 写操作
  vma->vm_flags |= VM_DONTCOPY | VM_MIXEDMAP;
  vma->vm_flags &= ~VM_MAYWRITE;

  vma->vm_ops = &binder_vm_ops;
  vma->vm_private_data = proc;
  // 再次完善 binder buffer allocator
  return binder_alloc_mmap_handler(&proc->alloc, vma);
}
int binder_alloc_mmap_handler(struct binder_alloc *alloc,
            struct vm_area_struct *vma)
{
  int ret;
  const char *failure_string;
  //每一次Binder传输数据时，都会先从Binder内存缓存区中分配一个binder_buffer来存储传输数据
  struct binder_buffer *buffer;
  //同步锁
  mutex_lock(&binder_alloc_mmap_lock);
  // 不需要重复mmap
  if (alloc->buffer_size) {
    ret = -EBUSY;
    failure_string = "already mapped";
    goto err_already_mapped;
  }
  //vma->vm_end, vma->vm_start 指向要 映射的用户空间地址, map size 不允许 大于 4M
  alloc->buffer_size = min_t(unsigned long, vma->vm_end - vma->vm_start,
           SZ_4M);
  mutex_unlock(&binder_alloc_mmap_lock);
  //指向用户进程内核虚拟空间的 start地址
  alloc->buffer = (void __user *)vma->vm_start;

  //分配物理页的指针数组，数组大小为vma的等效page个数
  alloc->pages = kcalloc(alloc->buffer_size / PAGE_SIZE,
             sizeof(alloc->pages[0]),
             GFP_KERNEL);
  if (alloc->pages == NULL) {
    ret = -ENOMEM;
    failure_string = "alloc page array";
    goto err_alloc_pages_failed;
  }
   //申请一个binder_buffer的内存
  buffer = kzalloc(sizeof(*buffer), GFP_KERNEL);
  if (!buffer) {
    ret = -ENOMEM;
    failure_string = "alloc buffer struct";
    goto err_alloc_buf_struct_failed;
  }

  //指向用户进程内核虚拟空间的 start地址，即为当前进程mmap的内核空间地址
  buffer->user_data = alloc->buffer;
  //将binder_buffer地址，加入到所属进程的buffers队列
  list_add(&buffer->entry, &alloc->buffers);
  buffer->free = 1;
  //将当前buffer加入到红黑树alloc->free_buffers中，表示当前buffer是空闲buffer
  binder_insert_free_buffer(alloc, buffer);
  // 将异步事务的空间大小设置为 整个空间的一半
  alloc->free_async_space = alloc->buffer_size / 2;
  binder_alloc_set_vma(alloc, vma);
  mmgrab(alloc->vma_vm_mm);

  return 0;

err_alloc_buf_struct_failed:
  kfree(alloc->pages);
  alloc->pages = NULL;
err_alloc_pages_failed:
  alloc->buffer = NULL;
  mutex_lock(&binder_alloc_mmap_lock);
  alloc->buffer_size = 0;
err_already_mapped:
  mutex_unlock(&binder_alloc_mmap_lock);
  binder_alloc_debug(BINDER_DEBUG_USER_ERROR,
         "%s: %d %lx-%lx %s failed %d\n", __func__,
         alloc->pid, vma->vm_start, vma->vm_end,
         failure_string, ret);
  return ret;
}
static void binder_insert_free_buffer(struct binder_alloc *alloc,
              struct binder_buffer *new_buffer)
{
  struct rb_node **p = &alloc->free_buffers.rb_node;
  struct rb_node *parent = NULL;
  struct binder_buffer *buffer;
  size_t buffer_size;
  size_t new_buffer_size;

  BUG_ON(!new_buffer->free);
  // 通过binder_alloc_buffer_size计算当前new_buffer的大小，之后将用于比较。
  new_buffer_size = binder_alloc_buffer_size(alloc, new_buffer);

  binder_alloc_debug(BINDER_DEBUG_BUFFER_ALLOC,
         "%d: add free buffer, size %zd, at %pK\n",
          alloc->pid, new_buffer_size, new_buffer);

  while (*p) {
    parent = *p;
    // 获取当前红黑树节点的buffer。
    buffer = rb_entry(parent, struct binder_buffer, rb_node);
    BUG_ON(!buffer->free);
     // 计算当前红黑树节点的buffer大小。
    buffer_size = binder_alloc_buffer_size(alloc, buffer);
    // 红黑树遵循二叉树规则：当新的buffer比当前节点的buffer小时，向左子节点继续重复，否则转向右子节点。
    if (new_buffer_size < buffer_size)
      p = &parent->rb_left;
    else
      p = &parent->rb_right;
  }
  // 找到合适位置后，将new_buffer插入到该位置。
  rb_link_node(&new_buffer->rb_node, parent, p);
  rb_insert_color(&new_buffer->rb_node, &alloc->free_buffers);
}
```

`binder_mmap`通过加锁，保证一次只有一个进程分配内存，保证多进程间的并发访问。将分配一块内核空间缓冲区buffer加入到红黑树alloc->free_buffers中，表示当前buffer是空闲buffer；每一次Binder传输数据时，都会先从Binder内存缓存区中分配一个binder_buffer来存储传输数据。

### binder_buffer结构体

```c
struct binder_buffer {
  //buffer实体的地址
  struct list_head entry; /* free and allocated entries by address */
  struct rb_node rb_node; /* free entry by size or allocated entry */
        /* by address */
  //标记是否是空闲buffer，占位1bit
  unsigned free:1;
  //是否允许用户释放，占位1bit
  unsigned clear_on_free:1;
  unsigned allow_user_free:1;
  unsigned async_transaction:1;
  unsigned debug_id:28;

  //该缓存区的需要处理的事务
  struct binder_transaction *transaction;

  //该缓存区所需处理的Binder实体
  struct binder_node *target_node;

   //数据大小
  size_t data_size;
  //数据偏移量
  size_t offsets_size;
  size_t extra_buffers_size;
  //用户数据
  void __user *user_data;
  int    pid;
};
struct binder_lru_page {
  struct list_head lru;//binder_alloc_lru中的条目
  struct page *page_ptr;//指向mmap空间中的物理页面的指针
  struct binder_alloc *alloc;//proc的binder_alloc
};
```

`binder_mmap`这里主要映射的内存只允许用户空间读，不允许用户空间写；`binder`驱动可以读写这块内存；存映射实现一次拷贝的概念是：

1. 用户空间给`binder`驱动传入信息时，都需要内存拷贝；
2. `binder`驱动给发送用户空间则不需要；

### 物理内存的分配和释放

`binder_update_page_range`函数主要的作用是分配和释放物理内存；



## binder_ioctl

 binder_ioctl()函数负责在两个进程间收发IPC数据和IPC reply数据。

 ioctl命令和数据类型是一体的，不同的命令对应不同的数据类型

| ioctl命令                | 数据类型                 | 操作                    |
| :----------------------- | :----------------------- | :---------------------- |
| **BINDER_WRITE_READ**    | struct binder_write_read | 收发Binder IPC数据      |
| BINDER_SET_MAX_THREADS   | __u32                    | 设置Binder线程最大个数  |
| BINDER_SET_CONTEXT_MGR   | __s32                    | 设置Service Manager节点 |
| BINDER_THREAD_EXIT       | __s32                    | 释放Binder线程          |
| BINDER_VERSION           | struct binder_version    | 获取Binder版本信息      |
| BINDER_SET_IDLE_TIMEOUT  | __s64                    | 没有使用                |
| BINDER_SET_IDLE_PRIORITY | __s32                    | 没有使用                |

这些命令中`BINDER_WRITE_READ`命令使用率最为频繁，也是ioctl最为核心的命令。

```c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
  int ret;
  //oepn的时候包在filp的private_data
  struct binder_proc *proc = filp->private_data;
  struct binder_thread *thread;
  unsigned int size = _IOC_SIZE(cmd);
  void __user *ubuf = (void __user *)arg;

  /*pr_info("binder_ioctl: %d:%d %x %lx\n",
      proc->pid, current->pid, cmd, arg);*/

  binder_selftest_alloc(&proc->alloc);

  trace_binder_ioctl(cmd, arg);

  //这里时一个阻塞代码，如果binder_stop_on_user_error 大于2这里会被休眠
  //这里先无视即可
  ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
  if (ret)
    goto err_unlocked;
  //获取当前线程对应的结构体，如果没有创建过，那么就创建一个放入到proc的红黑树
  thread = binder_get_thread(proc);
  if (thread == NULL) {
    ret = -ENOMEM;
    goto err;
  }

  switch (cmd) {
  case BINDER_WRITE_READ://进行binder的读写操作
    ret = binder_ioctl_write_read(filp, cmd, arg, thread);
    if (ret)
      goto err;
    break;
  case BINDER_SET_MAX_THREADS: {//设置binder最大支持的线程数
    int max_threads;

    if (copy_from_user(&max_threads, ubuf,
           sizeof(max_threads))) {
      ret = -EINVAL;
      goto err;
    }
    binder_inner_proc_lock(proc);
    proc->max_threads = max_threads;
    binder_inner_proc_unlock(proc);
    break;
  }
  case BINDER_SET_CONTEXT_MGR_EXT: {
   //设置Service Manager节点，带flag参数， servicemanager进程成为上下文管理者
    struct flat_binder_object fbo;

    if (copy_from_user(&fbo, ubuf, sizeof(fbo))) {
      ret = -EINVAL;
      goto err;
    }
    ret = binder_ioctl_set_ctx_mgr(filp, &fbo);
    if (ret)
      goto err;
    break;
  }
  case BINDER_SET_CONTEXT_MGR://成为binder的上下文管理者，不带flag参数，也就是ServiceManager成为守护进程
    ret = binder_ioctl_set_ctx_mgr(filp, NULL);
    if (ret)
      goto err;
    break;
  case BINDER_THREAD_EXIT://当binder线程退出，释放binder线程
    binder_debug(BINDER_DEBUG_THREADS, "%d:%d exit\n",
           proc->pid, thread->pid);
    binder_thread_release(proc, thread);
    thread = NULL;
    break;
  case BINDER_VERSION: {//获取binder的版本号
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
  case BINDER_GET_NODE_INFO_FOR_REF: {
    struct binder_node_info_for_ref info;

    if (copy_from_user(&info, ubuf, sizeof(info))) {
      ret = -EFAULT;
      goto err;
    }

    ret = binder_ioctl_get_node_info_for_ref(proc, &info);
    if (ret < 0)
      goto err;

    if (copy_to_user(ubuf, &info, sizeof(info))) {
      ret = -EFAULT;
      goto err;
    }

    break;
  }
  case BINDER_GET_NODE_DEBUG_INFO: {
    struct binder_node_debug_info info;

    if (copy_from_user(&info, ubuf, sizeof(info))) {
      ret = -EFAULT;
      goto err;
    }

    ret = binder_ioctl_get_node_debug_info(proc, &info);
    if (ret < 0)
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
    thread->looper_need_return = false;
  wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
  if (ret && ret != -ERESTARTSYS)
    pr_info("%d:%d ioctl %x %lx returned %d\n", proc->pid, current->pid, cmd, arg, ret);
err_unlocked:
  trace_binder_ioctl_done(ret);
  return ret;
}
```

### 获取binder线程

从binder_proc中查找binder_thread,如果当前线程已经加入到proc的线程队列则直接返回，如果不存在则创建binder_thread，并将当前线程添加到当前的proc

```c
static struct binder_thread *binder_get_thread(struct binder_proc *proc)
{
  struct binder_thread *thread;
  struct binder_thread *new_thread;

  binder_inner_proc_lock(proc);
   //从当前进程中获取线程
  thread = binder_get_thread_ilocked(proc, NULL);
  binder_inner_proc_unlock(proc);
  if (!thread) {
    //如果当前进程中没有线程，那么创建一个
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
}
```

### binder_thread结构体

`binder_thread`是当前binder操作所在的线程；

```c
struct binder_thread {
  struct binder_proc *proc;//线程所属的进程
  struct rb_node rb_node; //红黑树节点
  struct list_head waiting_thread_node;
  int pid;//线程pid
  int looper;              /* only modified by this thread */
  bool looper_need_return; /* can be written by other thread */
  struct binder_transaction *transaction_stack;//线程正在处理的事务
  struct list_head todo;//将要处理的链表
  bool process_todo;
  struct binder_error return_error;//write失败后，返回的错误码
  struct binder_error reply_error;
  wait_queue_head_t wait; //等待队列的队头
  struct binder_stats stats;//binder线程的统计信息
  atomic_t tmp_ref;
  bool is_dead;
  struct task_struct *task;
};
```

### ServiceManager是如何成binder驱动的上下文管理者

`ServiceManager`在native层中`ProcessState`的`becomeContextManager`函数通过`ioctl`发送`BINDER_SET_CONTEXT_MGR_EXT` 命令，让自身成为上下文管理者;然后在binder驱动中的`binder_ioctl`接收到了`BINDER_SET_CONTEXT_MGR_EXT`指令后；通过`binder_ioctl_set_ctx_mgr`函数进行处理；

**binder_ioctl_set_ctx_mgr()处理如下：**

1. 通过`filp->private_data`找到`binder_proc`;
2. 对当前进程进行是否具注册Context Manager的SELinux安全权限的检查
3. 进行uid检查，线程只能注册自己，且只能有一个线程设置为Context Manager
4. 设置当前线程euid作为ServiceManager的uid；
5. 创建一个binder实体binder_node，并加入到当前进程的nodes红黑树中；
6. 把新创建的binder_node,赋值给当前进程的binder_context_mgr_node，这样就约定该进程就成为了binder驱动的上下文的管理者；

```c

static int binder_ioctl_set_ctx_mgr(struct file *filp,
            struct flat_binder_object *fbo)
{
  int ret = 0;
   //filp->private_data 在open()binder驱动时，保存了一个创建的binder_proc，即是此时调用进程的binder_proc.
  struct binder_proc *proc = filp->private_data;
   //获得当前进程的context
  struct binder_context *context = proc->context;
  struct binder_node *new_node;
  kuid_t curr_euid = current_euid();

  mutex_lock(&context->context_mgr_node_lock);
  //保证只创建一次mgr_node对象
  if (context->binder_context_mgr_node) {
    pr_err("BINDER_SET_CONTEXT_MGR already set\n");
    ret = -EBUSY;
    goto out;
  }
  //检查当前进程是否具注册Context Manager的SEAndroid安全权限
  ret = security_binder_set_context_mgr(proc->tsk);
  if (ret < 0)
    goto out;
  //检查已的uid是否有效
  if (uid_valid(context->binder_context_mgr_uid)) {
    //uid有效但是与当前运行线程的效用户ID不相等，则出错。
    //即线程只能注册自己，且只能有一个线程设置为Context Manager
    if (!uid_eq(context->binder_context_mgr_uid, curr_euid)) {
      pr_err("BINDER_SET_CONTEXT_MGR bad uid %d != %d\n",
             from_kuid(&init_user_ns, curr_euid),
             from_kuid(&init_user_ns,
           context->binder_context_mgr_uid));
      ret = -EPERM;
      goto out;
    }
  } else {
    //设置当前线程euid作为ServiceManager的uid
    context->binder_context_mgr_uid = curr_euid;
  }
  //创建binder实体，并加入到当前进程的nodes红黑树中，我们这里可以是ServiceManager
  new_node = binder_new_node(proc, fbo);
  if (!new_node) {
    ret = -ENOMEM;
    goto out;
  }
  binder_node_lock(new_node);
  //更新new_node的相关强弱引用计数
  new_node->local_weak_refs++;
  new_node->local_strong_refs++;
  new_node->has_strong_ref = 1;
  new_node->has_weak_ref = 1;
  //new_node 赋值给进程的上下文管理节点，作为上下文管理者
  context->binder_context_mgr_node = new_node;
  binder_node_unlock(new_node);
  binder_put_node(new_node);
out:
  mutex_unlock(&context->context_mgr_node_lock);
  return ret;
}
```

### binder_node结构体

`binder_node`代表一个binder实体

```c
struct binder_node {
  int debug_id;//节点创建时分配，具有全局唯一性，用于调试使用
  spinlock_t lock;
  struct binder_work work;
  union {
    struct rb_node rb_node;//binder节点正常使用，union
    struct hlist_node dead_node;//binder节点已销毁，union
  };
  struct binder_proc *proc;//binder所在的进程
  struct hlist_head refs;//所有指向该节点的binder引用队列
  int internal_strong_refs;
  int local_weak_refs;
  int local_strong_refs;
  int tmp_refs;
  binder_uintptr_t ptr;//指向用户空间binder_node的指针，对应flat_binder_object.binder
  binder_uintptr_t cookie;//数据，对应flat_binder_object.cooki
  struct {
    /*
     * bitfield elements protected by
     * proc inner_lock
     */
    u8 has_strong_ref:1;
    u8 pending_strong_ref:1;
    u8 has_weak_ref:1;
    u8 pending_weak_ref:1;
  };
  struct {
    /*
     * invariant after initialization
     */
    u8 sched_policy:2;
    u8 inherit_rt:1;
    u8 accept_fds:1;
    u8 txn_security_ctx:1;
    u8 min_priority;
  };
  bool has_async_transaction;
  struct list_head async_todo;//异步todo队列
};
```

### binder_ioctl_write_read

`binder_ioctl_write_read`是binder数据交互的核心入口,在native层中通过`ioctl`发送`BINDER_WRITE_READ`命令执行该函数，`arg`是一个`binder_write_read`结构体;

```c
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
  //把用户空间数据ubuf拷贝到bwr
  if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
    ret = -EFAULT;
    goto out;
  }
  binder_debug(BINDER_DEBUG_READ_WRITE,
         "%d:%d write %lld at %016llx, read %lld at %016llx\n",
         proc->pid, thread->pid,
         (u64)bwr.write_size, (u64)bwr.write_buffer,
         (u64)bwr.read_size, (u64)bwr.read_buffer);

  if (bwr.write_size > 0) {
    //当写缓存中有数据，则执行binder写操作
    ret = binder_thread_write(proc, thread,
            bwr.write_buffer,
            bwr.write_size,
            &bwr.write_consumed);
    trace_binder_write_done(ret);
    if (ret < 0) {
      //binder_thread_write中有错误发生，则read_consumed设为0，表示kernel没有数据返回给进程
      bwr.read_consumed = 0;
      //将bwr返回给用户态调用者，bwr在binder_thread_write中会被修改
      if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
        ret = -EFAULT;
      goto out;
    }
  }
  if (bwr.read_size > 0) {
    //当读缓存中有数据，则执行binder读操作
    ret = binder_thread_read(proc, thread, bwr.read_buffer,
           bwr.read_size,
           &bwr.read_consumed,
           filp->f_flags & O_NONBLOCK);
    trace_binder_read_done(ret);
    binder_inner_proc_lock(proc);
    //读取完后，如果proc->todo链表不为空，则唤醒在proc->wait等待队列上的进程
    if (!binder_worklist_empty_ilocked(&proc->todo))
      binder_wakeup_proc_ilocked(proc);
    binder_inner_proc_unlock(proc);
    if (ret < 0) {
      //如果binder_thread_read返回小于0，可能处理一半就中断了，需要将bwr拷贝回进程的用户态地址
      if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
        ret = -EFAULT;
      goto out;
    }
  }
  binder_debug(BINDER_DEBUG_READ_WRITE,
         "%d:%d wrote %lld of %lld, read return %lld of %lld\n",
         proc->pid, thread->pid,
         (u64)bwr.write_consumed, (u64)bwr.write_size,
         (u64)bwr.read_consumed, (u64)bwr.read_size);
  //将内核数据bwr拷贝到用户空间ubuf
  if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
    ret = -EFAULT;
    goto out;
  }
out:
  return ret;
}
```

**流程:**

1. 用户空间数据ubuf拷贝到内核空间bwr;
2. 当bwr写缓存有数据，则执行binder_thread_write；当写失败则read_consumed设为0，并将bwr数据写回用户空间并退出；
3. 当bwr读缓存有数据，则执行binder_thread_read，读取完后，如果proc->todo链表不为空，则唤醒在proc->wait等待队列上的进程;当读失败则再将bwr数据写回用户空间并退出；
4. 把内核数据bwr拷贝到用户空间ubuf。

