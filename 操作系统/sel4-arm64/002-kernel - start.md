> sel4test/kernel/src/arch/arm/64/head.S

# `_start`汇编
1. 设置栈；
2. 设置SCTLR寄存器
3. 如果配置了SMP，将栈指针结合elf-loader中设置的tpidr_el1，重新设置tpidr_el1/2(hyp)
	1. 需要注意，elf-loader中设置的为逻辑cpu_id，其顺序始终为0，1，2，3，...
	2. tpidr_elx的低12Bit用于存储逻辑CPU ID

# init_kernel
```asm
/* Call bootstrapping implemented in C with parameters:
* x0: user image physical start address
* x1: user image physical end address
* x2: user image physical/virtual offset
* x3: user image virtual entry address
* x4: DTB physical address (0 if there is none)
* x5: DTB size (0 if there is none)
*/

bl init_kernel

/* Thread 恢复执行 */ 
b restore_user_context 
```

## init_kernel
```c
BOOT_CODE VISIBLE void init_kernel(
	paddr_t ui_p_reg_start,
	paddr_t ui_p_reg_end,
	sword_t pv_offset,
	vptr_t v_entry,
	paddr_t dtb_addr_p,
	uint32_t dtb_size
)
{
	try_init_kernel();

	// 调度
	schedule();
	activateThread();
}
```

SMP的情况下：
- CPU 0 继续执行`try_init_kernel`
- 其余CPU执行`try_init_kernel_secondary_core`

# try_init_kernel
```c
static BOOT_CODE bool_t try_init_kernel(
	paddr_t ui_p_reg_start, // user image physical start address
	paddr_t ui_p_reg_end,
	sword_t pv_offset,      // user image physical/virtual offset
	vptr_t v_entry,         // user image virtual entry address
	paddr_t dtb_phys_addr, 
	word_t dtb_size
) {


	map_kernel_window();  // [PPTR_BASE, PPTR_TOP](pptr physical addr -> virt addr window) -> [PADDR_BASE, PADDR_TOP], 2M Page
	map_kernel_devices(); // build/kernel/gen_headers/plat/machine/devices_gen.h, such as serial/interrupt-controller/
	activate_kernel_vspace(); // 设置内核MMU页表基地址
	setKernelStack(stack_top); // 设置kernel stack为一全局数组
	setVtable((pptr_t)arm_vector_table); // 设置异常向量表
	
	cpu_initLocalIRQController(); // 中断控制器初始化
		{
			gicr_init();
			cpu_iface_init();
		}
		
	init_plat();
		{
			initIRQController(void); // -> dist_init() Route all global IRQs to this boot CPU
		}
		
	arch_init_freemem(); // 在pptr window中预留出kernel image区间、DTB区间
	// avail_p_regs，设备可用的所有物理内存区域，由gen_headers/plat/machine/devices_gen.h中定义
	// 最终所有的memory都被ndks_boot_t结构管理：预留区域，free区域
	
	init_freemem();
	// 预留 rootserver 内存区域，包括：[image、ipcbuff、bootInfo、extra_bootinfo]、Cspace[item = slot]、TCB、IPC、BOOTInfo、ASIDPool、VSpace
	// 从ndks_boot.freemem[]的最后一片区域开始，找到一个能存放rootserver所有内容的区域
	create_rootserver_objects();
	// 先后创建的内容为：[对齐pad] rootserver.cnode->rootserver.vspace->rootserver.asid_pool->rootserver.ipc_buf->rootserver.boot_info
	// ->rootserver.tcb->rootserver.paging[页表]; 这里的类型均为pptr_t
	
	/* create the root cnode */
	root_cnode_cap = create_root_cnode();
	// 从下面create_root_cnode()的实现可以看到，这里是创建了cap_t，而其对应的CSpace实际在的内存空间就是rootserver.cnode
	// 且该能力被写到了CSpace的seL4_CapInitThreadCNode下标所在的slot
	
	create_domain_cap(root_cnode_cap); // thread domains releated to schedule, inserted at root CSpace[seL4_CapDomain]
	
	/* initialise the IRQ states and provide the IRQ control cap */
	init_irqs(root_cnode_cap); // [seL4_CapIRQControl]
	
#ifdef CONFIG_ALLOW_SMC_CALLS
	/* secutiry monitor call which will be routed to EL3 */
	init_smc(root_cnode_cap); // [seL4_CapSMC]
#endif

	populate_bi_frame(); //填充rootserver.boot_info中的部分字段
	// 还有两个关键赋值：ndks_boot.bi_frame = rootserver.boot_info; ndks_boot.slot_pos_cur = seL4_NumInitialCaps;
	// bi->ipcBuffer = rootserver user image 后面的那个ipcbuffer；而不是TCB后面的那个！！！
	
	// 将DTB放在rootserver.extra_bi地址处，存储格式为 seL4_BootInfoHeader_1[id(seL4_BootInfoID), len] + data; seL4_BootInfoHeader_2 + data; +...
	
	it_pd_cap = create_it_address_space(root_cnode_cap, it_v_reg)；
	// 新建一个vspace cap，其capVSBasePtr = rootserver.vspace，并放在[seL4_CapInitThreadVSpace]下标处
	// create_it_pud/pd/pt_cap创建各级页表，管理的虚拟内存空间为[image、ipcbuff、bootInfo]
	// 创建页表：从rootserver.paging.start开始的内存空间获取一个页
	// 中间页表cap管理的映射关系为vptr->pptr[注意，不是写到最终页表中的映射关系]；最终写到页表项时，会将pptr转换为paddr
	// 创建的中间页表cap会被放在[ndks_boot.slot_pos_cur]下标处，并将ndks_boot.slot_pos_cur++ <- provide_cap()
	// 更新ndks_boot.bi_frame->userImagePaging，记录了[image、ipcbuff、bootInfo]虚拟内存对的页表caps的起始/结束slot下标
	// 注意这里的页表，还没有创建一个frame cap
	
	/* Create and map bootinfo frame cap */
	create_bi_frame_cap(root_cnode_cap, it_pd_cap, bi_frame_vptr);
	// [image、ipcbuff、bootInfo]中的bootInfo = bi_frame_vptr -> rootserver.boot_info(pptr)，将该frame cap放在[seL4_CapBootInfoFrame]下标处
	// map_it_frame_cap()，将frame cap对应的内存映射【vptr->pptr】转换为vptr->paddr，并纳入到it_pd_cap的页表体系中去，
	// 即填充对应的pt entry表项内容，包括读、写、执行属性等 --> 这就与具体的硬件架构有关系了
	
	// create and map extra bootinfo region ，即创建extra boot info region的映射页表项，可能超过一个页的大小
	// [image、ipcbuff、bootInfo、extra_bootinfo]中的extra_bootinfo区域虚拟地址，映射到rootserver.extra_bi（pptr）对应的物理地址
	// 返回也是这段内存区间frame caps在root CSpace中对应的slot下标区间
	create_frames_of_region()
	
	/* create the initial thread's IPC buffer */
	ipcbuf_cap = create_ipcbuf_frame_cap(root_cnode_cap, it_pd_cap, ipcbuf_vptr);
	// 虚拟地址区间中的[image、ipcbuff、bootInfo]中的ipcbuff -> rootserver.ipc_buf(pptr_t)
	// 创建的frame cap下到seL4_CapInitThreadIPCBuffer对应下标的slot中
	
	/* create all userland image frames */
	create_frames_ret = create_frames_of_region();
	// 虚拟地址区间[image、ipcbuff、bootInfo]中的image -> 对应的pptr -> paddr
	// 记录ndks_boot.bi_frame->userImageFrames = create_frames_ret.region;
	
	/* create/initialise the initial thread's ASID pool */
	create_it_asid_pool();
	// 创建的asid_poll cap记录了rootserver.asid_pool的pptr地址
	// 保存到seL4_CapInitThreadASIDPool下标对应的slot
	// 同时创建asid control cap（cap_asid_control_cap），放在seL4_CapASIDControl下标对应的slot
	
	write_it_asid_pool();
	// 记录当前it_pd_cap和asid_poll_cap信息，以asid_map_t的方式，记录到全局变量armKSASIDTable中
	
	// 创建idle线程
	create_idle_thread(); // 执行入口为 idle_thread
	// idle thread的内存区域在全局数组ksIdleThreadTCB[]
	// 内核使用ksIdleThread全局变量，记录idle thread的TCB，TCB_OFFSET
	// 设置idle thread上下文的寄存器区域，SPSR_EL1 = PSTATE_IDLETHREAD；ELR_EL1 = idle_thread「即idle thread的执行入口」
	
	create_initial_thread(); // 创建初始线程，也即rootserver对应的线程
	
	tcb_t *initial = create_initial_thread(
						root_cnode_cap, // 
						it_pd_cap,      // 
						v_entry,        // 
						bi_frame_vptr,  //
						ipcbuf_vptr,    // 
						ipcbuf_cap);    //
	// init thread对应的tcb内存区域就是rootserver.tcb，因此tcb_t *tcb = TCB_PTR(rootserver.tcb + TCB_OFFSET);
	
	NODE_STATE(ksSchedulerAction) = initial;
	NODE_STATE(ksCurThread) = NODE_STATE(ksIdleThread);
	// 这两个变量是内核进行线程调度的关键
	
	/* create all of the untypeds. Both devices and kernel window memory */
	create_untypeds(root_cnode_cap);
	// 为ndks_boot.freemem[i]的所有区域创建cap_untyped_cap，
	// 并放到ndks_boot.slot_pos_cur下标指向的root CSpace后续slot中，之后ndks_boot.slot_pos_cur++;
	// 此外ndks_boot.bi_frame->untypedList[i]还会记录所有untyped的信息，包括：
	// pptr、size、是否为device memory
	// 还会重新利用kernel image的部分boot区域「其只在boot阶段使用，因此这里写为untyped，以便后续重复利用」
	// ndks_boot.bi_frame->untyped = {.start = first_untyped_slot, .end = ndks_boot.slot_pos_cur}记录了untyped的slot区间
	
	/* finalise the bootinfo frame */
	bi_finalise();
	// ndks_boot.bi_frame->empty = {.start = ndks_boot.slot_pos_cur, BIT(CONFIG_ROOT_CNODE_SIZE_BITS)}
	
	/* initialize BKL before booting up other cores */
	SMP_COND_STATEMENT(clh_lock_init());
	SMP_COND_STATEMENT(release_secondary_cpus()); // 写标志位，运行其它核完成boot初始化，并等待其它核完成，
	// 「其它核完成后会ksNumCPUs++，这里循环判断ksNumCPUs == CONFIG_MAX_NUM_NODES」
}
```

## create_root_cnode
```c
BOOT_CODE cap_t
create_root_cnode(void)
{
	cap_t cap = cap_cnode_cap_new(
		CONFIG_ROOT_CNODE_SIZE_BITS, /* radix */
		wordBits - CONFIG_ROOT_CNODE_SIZE_BITS, /* guard size */
		0, /* guard */
		rootserver.cnode); /* pptr */
	/* write the root CNode cap into the root CNode */
	write_slot(SLOT_PTR(rootserver.cnode, seL4_CapInitThreadCNode), cap);
	// 这里rootserver.cnode，即为root_cnode_cap代表的CSpace的实际内存空间，其含有多个slot存放不同的cap
	return cap;
}
```

## create_initial_thread
```c
tcb_t *initial = create_initial_thread(
					root_cnode_cap, // 
					it_pd_cap,      // 
					v_entry,        // 
					bi_frame_vptr,  //
					ipcbuf_vptr,    // 
					ipcbuf_cap);    //

/* derive a copy of the IPC buffer cap for inserting */

/* initialise TCB (corresponds directly to abstract specification) */
cteInsert(...); // 拷贝并写到新的dest slot；并通过MDBNode双向链表，将dest slot插入到src slot MDBNode的双向链表中进行管理
// 将seL4_CapInitThreadCNode、seL4_CapInitThreadVSpace、seL4_CapInitThreadIPCBuffer下标对应的cap分别derive一份，
// 并存放到SLOT_PTR(rootserver.tcb, tcbCTable/tcbVTable/tcbBuffer)对应的slot中
// 从这里可以看到，TCB相关的几个关键cap slot就在TCB内存地址起始处；而TCB内存地址起始处偏移TCB_OFFSET才是tcb_t结构所在位置
// 此处，rootserver.tcb即 init_thread的tcb，因此init_thread可以访问所有的rootserver caps
// 从create_root_cnode()看，root_cnode_cap表示的CSpace的seL4_CapInitThreadCNode下标处的cap即为root_cnode_cap

tcb->tcbIPCBuffer = ipcbuf_vptr;
setRegister(tcb, capRegister, bi_frame_vptr); // 这个可能是关键参数？
setNextPC(tcb, ui_v_entry); //对于aarch64来说，为 #define NEXT_PC_REG ELR_EL1/2

// 。。。

setupReplyMaster(tcb);
// 在没配置MCS的情况下进行：创建cap_reply_cap【其记录了thread的TCB地址】，并放在tcbReply偏移处的slot中

setThreadState(tcb, ThreadState_Running);
ksCurDomain = ksDomSchedule[ksDomScheduleIdx].domain;

/* create initial thread's TCB cap */
cap_t cap = cap_thread_cap_new(TCB_REF(tcb));
write_slot(SLOT_PTR(pptr_of_cap(root_cnode_cap), seL4_CapInitThreadTCB), cap);
// TCB内存区域开始的slots是不可能记录TCB cap的

setThreadName(tcb, "rootserver");


```

## 调度相关
```c
	schedule();
	activateThread();
	restore_user_context();
```

### schedule()
```c
#define SchedulerAction_ResumeCurrentThread ((tcb_t*)0)
#define SchedulerAction_ChooseNewThread ((tcb_t*) 1)

/* Values of 0 and ~0 encode ResumeCurrentThread and ChooseNewThread
* respectively; other values encode SwitchToThread and must be valid
* tcb pointers */
UP_STATE_DEFINE(tcb_t *, ksSchedulerAction);
// 其它值表示切到该线程

void schedule(void) {

#ifdef CONFIG_KERNEL_MCS
	awaken();
	checkDomainTime();
#endif

	if (NODE_STATE(ksSchedulerAction) != SchedulerAction_ResumeCurrentThread) {
		
	}
	
	NODE_STATE(ksSchedulerAction) = SchedulerAction_ResumeCurrentThread;
#ifdef ENABLE_SMP_SUPPORT
	// 多核调度
	doMaskReschedule(ARCH_NODE_STATE(ipiReschedulePending));
	ARCH_NODE_STATE(ipiReschedulePending) = 0;
#endif /* ENABLE_SMP_SUPPORT */

#ifdef CONFIG_KERNEL_MCS
	switchSchedContext();
	if (NODE_STATE(ksReprogram)) {
		setNextInterrupt();
		NODE_STATE(ksReprogram) = false;
	}
#endif
}
```

### activateThread()
```c
// 将恢复ksCurThread变量指向的当前线程
ThreadState_Running, // 如何理解这几个状态？

ThreadState_Restart, // 会将Fault_IP对应的内容写到ELR_EL1/（或ELR_EL2 Hyp开启)寄存器对应的上下文位置处
// 似乎是从Fault_IP寄存器指示的位置处重新执行，
// 例如缺页异常等，就得重新读写操作
// 而中断等直接继续后续的执行就可以，不需要执行被中断的指令「因为上一条指令一定是执行完的」


ThreadState_IdleThreadState,
```

### restore_user_context()
```c
// 恢复ksCurThread变量指向的当前线程
```

三类地址相关重要类型
```c
paddr_t; // raw physcial address
pptr; // virtual address in kernel world
```
# restore_user_context
- 恢复执行root_server



# 补充知识
## aarch64异常向量表内容解析
向量可以分为两大类，四种
1. Exception from Low Level，从低特权级来的异常
	- 低特权级的执行状态为aarch32
	- 低特权级的执行状态为aarch64
2. Exception from Current level
	- 选择使用SP_EL0来处理异常
	- 选择使用SP_ELx来处理异常（即使用当前特权级自己的栈）

