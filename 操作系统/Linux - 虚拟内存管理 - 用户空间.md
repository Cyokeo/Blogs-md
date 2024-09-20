---
title: Linux - 虚拟内存管理
categories: 操作系统
---
## 参考文章
- [bin的技术小屋](https://www.cnblogs.com/binlovetech/p/16824522.html)

## 阅读总结
#### 用户空间
应用程序在用户空间时，只能看到用户空间的虚拟地址：无需关注具体对应的物理内存；其虚拟地址内存空间到物理内存空间的映射由内核管理
陷入内核后，程序只能看到内核空间的虚拟地址；由于内核态管理整个物理内存空间，因此处于内核态时一定要能看到整个物理内存空间。内核虚拟空间大小有可能小于实际的物理内存空间大小「32位系统内核空间只有1G，而物理内存空间最大可以到4G」，因此必须要采用动态映射技术使内核态可以访问到所有的物理内存空间。

#### 区分虚拟内存空间 🆚 物理内存空间
- 虚拟空间多大都没事：只要开启了虚拟机制、swap机制
- 建立好虚拟内存 -> 物理内存的映射后：只是建立好了页表，并没有分配物理内存空间
- 父子进程虚拟内存空间
	- 通过`vfork/clone`系统调用创建出的子进程（线程）共享父进程的虚拟内存空间：即直接将父进程的虚拟内存空间结构体「`struct mm_struct`」的地址赋值给子进程中的虚拟内存空间结构体指针，并增加父进程虚拟内存空间结构体的引用计数；即子进程不含有「`struct mm_struct`」的内存
	- 通过`fork`系统调用创建出的子进程，则将父进程的虚拟内存空间以及相关页表拷贝到拷贝到子进程的「`struct mm_struct`」中，即子进程含有自己的「`struct mm_struct`」结构内存 -> 此时子进程虚拟内存对应的物理内存应该是与父进程相同的！！！❌ -> 共享代码段，但是数据段不共享，可以采取写时复制技术
- 子进程共享了父进程的虚拟内存空间，这样子进程就变成了我们熟悉的线程，**是否共享地址空间几乎是进程和线程之间的本质区别。Linux 内核并不区别对待它们，线程对于内核来说仅仅是一个共享特定资源的进程而已**
- 「内核线程」和「用户态线程」的区别就是内核线程没有相关的内存描述符 mm_struct ，***内核线程对应的 task_struct 结构中的 mm 域指向 NULL***，所以内核线程之间调度是不涉及地址空间切换的。
- 当一个内核线程被调度时，它会发现自己的虚拟地址空间为 Null，虽然它不会访问用户态的内存，但是它会访问内核内存，聪明的内核会*将调度之前的上一个用户态进程的虚拟内存空间 mm_struct 直接赋值给内核线程*，因为内核线程不会访问用户空间的内存，它仅仅只会访问内核空间的内存，所以直接复用上一个用户态进程的虚拟地址空间就可以避免为内核线程分配 mm_struct 和相关页表的开销，以及避免内核线程之间调度时地址空间的切换开销。

> 父进程与子进程的区别，进程与线程的区别，以及内核线程与用户态线程的区别其实都是围绕着这个 mm_struct 展开的。


## Linux虚拟内存空间布局
### 用户空间布局
用户空间最上方：栈 -> 「待分配区域」| 文件映射区 | 「待分配区域」<- 堆 <->「 BSS」、「数据段」-> 「不可访问区域（仅限64位系统）」 <-「代码段」、「保留区」。
- malloc : 从堆区分配内存
- mmap : 从文件映射区分配内存
- 文件映射区：还会存放动态库中的代码段、数据段、BSS段

### `mm_struct`与虚拟内存管理
```c
struct mm_struct {
    unsigned long task_size;    /* size of task vm space */
    unsigned long start_code, end_code, start_data, end_data;
    unsigned long start_brk, brk, start_stack;
    unsigned long arg_start, arg_end, env_start, env_end;
    unsigned long mmap_base;  /* base of mmap area */
    unsigned long total_vm;    /* Total pages mapped */
    unsigned long locked_vm;  /* Pages that have PG_mlocked set */
    unsigned long pinned_vm;  /* Refcount permanently increased */
    unsigned long data_vm;    /* VM_WRITE & ~VM_SHARED & ~VM_STACK */
    unsigned long exec_vm;    /* VM_EXEC & ~VM_WRITE & ~VM_STACK */
    unsigned long stack_vm;    /* VM_STACK */

       ...... 省略 ........
}
```

`task_size `表示进程用户空间的大小，用来划分用户态虚拟内存空间和内核态虚拟内存空间

mm_struct 结构体中的 total_vm 表示在进程虚拟内存空间中总共与物理内存映射的页的总数。

> 注意映射这个概念，它表示只是将*虚拟内存与物理内存建立关联关系*，「***并不代表真正的分配物理内存***」

当内存吃紧的时候，有些页可以换出到硬盘上，而有些页因为比较重要，不能换出。***locked_vm*** 就是被「锁定不能换出」的内存页总数，***pinned_vm*** 表示「既不能换出，也不能移动」的内存页总数。

data_vm 表示数据段中映射的内存页数目，exec_vm 是代码段中存放可执行文件的内存页数目，stack_vm 是栈中所映射的内存页数目，这些变量均是表示进程虚拟内存空间中的「虚拟内存使用情况」。

### vm_area_struct
一个虚拟内存区域就是一次分配后的结果，其可能包含多个虚拟内存页，例如在上述的堆区进行内存申请。
一个虚拟内存区域就对应前述的某个VMA，例如堆、栈、代码区等 ☑️
```c
struct vm_area_struct {
	unsigned long vm_start;		/* Our start address within vm_mm. */
	unsigned long vm_end;		/* The first byte after our end address
					   within vm_mm. */
	/*
	 * Access permissions of this VMA.
	 */
	pgprot_t vm_page_prot;  // 决定页表中内存页访问权限
	unsigned long vm_flags;	// 整个虚拟内存区域的访问权限及行为规范：读/写/执行等
							// 这块虚拟内存区域映射的物理内存是否可以多进程共享
	struct anon_vma *anon_vma;	/* Serialized by page_table_lock */
    struct file * vm_file;		/* File we map to (can be NULL). */
	unsigned long vm_pgoff;		/* Offset (within vm_file) in PAGE_SIZE
					   units */	
	void * vm_private_data;		/* was vm_pte (shared mem) */
	/* Function pointers to deal with this struct. */
	const struct vm_operations_struct *vm_ops;
}
```

每个 vm_area_struct 结构对应于虚拟内存空间中的唯一虚拟内存区域 VMA，vm_start 指向了这块虚拟内存区域的起始地址（最低地址），vm_start 本身包含在这块虚拟内存区域内。vm_end 指向了这块虚拟内存区域的结束地址（最高地址），而 vm_end 本身包含在这块虚拟内存区域之外，所以 vm_area_struct 结构描述的是` [vm_start，vm_end)` 这样一段左闭右开的虚拟内存区域，每个区域是连续的，且可以包含很多虚拟内存页。

通过这一系列的介绍，我们可以看到` vm_flags` 就是定义整个虚拟内存区域的访问权限以及行为规范，而内存区域中内存的最小单位为页（4K），虚拟内存区域中包含了很多这样的虚拟页，对于虚拟内存区域 VMA 设置的访问权限也会全部复制到区域中包含的内存页中

接下来的三个属性 anon_vma，vm_file，vm_pgoff 分别和虚拟内存映射相关，虚拟内存区域可以映射到物理内存上，也可以映射到文件中，映射到物理内存上我们称之为匿名映射，映射到文件中我们称之为文件映射。

#### vm_ops
struct vm_area_struct 结构中还有一个 vm_ops 用来指向针对虚拟内存区域 VMA 的相关操作的函数指针：
```c
struct vm_operations_struct {
	void (*open)(struct vm_area_struct * area);
	void (*close)(struct vm_area_struct * area);
    vm_fault_t (*fault)(struct vm_fault *vmf);
    vm_fault_t (*page_mkwrite)(struct vm_fault *vmf);

    ..... 省略 .......
}
```

- 当指定的虚拟内存区域被加入到进程虚拟内存空间中时，open 函数会被调用
- 当虚拟内存区域 VMA 从进程虚拟内存空间中被删除时，close 函数会被调用
- 当进程访问虚拟内存时，访问的页面不在物理内存中，可能是未分配物理内存也可能是被置换到磁盘中，这时就会产生缺页异常，fault 函数就会被调用。
- 当一个只读的页面将要变为可写时，page_mkwrite 函数会被调用。

### 内核如何组织虚拟内存区域
```c
struct vm_area_struct {

	struct vm_area_struct *vm_next, *vm_prev;
	struct rb_node vm_rb;
    struct list_head anon_vma_chain; 
	struct mm_struct *vm_mm;	/* The address space we belong to. */
	
	......
}
```
可以看到，是通过双向链表来组织管理所有`vm_area_struct`的。双向链表的头指针存储在内存描述符 struct mm_struct 结构中的 mmap 中，正是这个 mmap 串联起了整个虚拟内存空间中的虚拟内存区域。
```c
struct mm_struct {
	...
    struct vm_area_struct *mmap;		/* list of VMAs */
    ...
}
```
内核中关于这些虚拟内存区域的操作除了遍历之外还有许多需要根据特定虚拟内存地址在虚拟内存空间中查找特定的虚拟内存区域。

尤其在进程虚拟内存空间中包含的内存区域 VMA 比较多的情况下，使用红黑树查找特定虚拟内存区域的时间复杂度是 O( logN ) ，可以显著减少查找所需的时间。

所以在内核中，***同样的内存区域 vm_area_struct 会有两种组织形式***，一种是「双向链表」用于高效的遍历，另一种就是「红黑树」用于高效的查找。

### load_elf_binary
内核中完成这个映射过程的函数是 load_elf_binary ，这个函数的作用很大，加载内核的是它，启动第一个用户态进程 init 的是它，fork 完了以后，调用 exec 运行一个二进制程序的也是它。当 exec 运行一个二进制程序的时候，除了解析 ELF 的格式之外，另外一个重要的事情就是建立上述提到的内存映射。
```c
static int load_elf_binary(struct linux_binprm *bprm)
{
      ...... 省略 ........
  // 设置虚拟内存空间中的内存映射区域起始地址 mmap_base
  setup_new_exec(bprm);

     ...... 省略 ........
  // 创建并初始化栈对应的 vm_area_struct 结构。
  // 设置 mm->start_stack 就是栈的起始地址也就是栈底，并将 mm->arg_start 是指向栈底的。
  retval = setup_arg_pages(bprm, randomize_stack_top(STACK_TOP),
         executable_stack);

     ...... 省略 ........
  // 将二进制文件中的代码部分映射到虚拟内存空间中
  error = elf_map(bprm->file, load_bias + vaddr, elf_ppnt,
        elf_prot, elf_flags, total_size);

     ...... 省略 ........
 // 创建并初始化堆对应的的 vm_area_struct 结构
 // 设置 current->mm->start_brk = current->mm->brk，设置堆的起始地址 start_brk，结束地址 brk。 起初两者相等表示堆是空的
  retval = set_brk(elf_bss, elf_brk, bss_prot);

     ...... 省略 ........
  // 将进程依赖的动态链接库 .so 文件映射到虚拟内存空间中的内存映射区域
  elf_entry = load_elf_interp(&loc->interp_elf_ex,
              interpreter,
              &interp_map_addr,
              load_bias, interp_elf_phdata);

     ...... 省略 ........
  // 初始化内存描述符 mm_struct
  current->mm->end_code = end_code;
  current->mm->start_code = start_code;
  current->mm->start_data = start_data;
  current->mm->end_data = end_data;
  current->mm->start_stack = bprm->p;

     ...... 省略 ........
}
```