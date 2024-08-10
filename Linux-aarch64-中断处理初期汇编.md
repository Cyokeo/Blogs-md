---
title: Linux-aarch64-中断处理初期汇编
date: 2024-07-23 20:33:47
tags:
categories:
- [Linux相关]
---

## 中断处理初期汇编码&dup2分析

### 参考博客

- [ARM-GNU常用汇编伪指令](https://www.cnblogs.com/niezhongle/p/11088658.html)
- [ARM64底层中断处理](https://blog.csdn.net/jasonactions/article/details/115689462)

### 总结

* kernel_ventry : el 0 64 sync
    * el0t_64_sync
        * kernel_entry \el -> 保存异常处理的现场[x0-29, lr, sp(sp_el0/1), pc(elr_el1), pstate(spsr_el1)]到当前sp`sp_el1，即当前task的内核栈`指向的地址处。因此，可以知道 -> 在进行任务切换时，内核应该会把sp_el0的值设置为待切换到的task的内核栈地址。此外，这里还会使用sp_el0存储当前用户态任务的task_struct数据的地址，供get_current()使用
            * el0t_64_sync_handler(struct pt_regs* regs), 这里的参数就使用当前的sp+sizeof(pt_regs)了


### Linux dup2()的实现中，current返回的为什么是task_struct?
* 根据下面的分析可知
    - 异常发生时，在第一阶段的处理`macro: kernel_entry`中，将当前用户态任务结构体的地址保存到了sp_el0寄存器中

    - 根据之后的分析可知：dup2的作用为：把当前进程文件描述符表中oldfd的文件表项指针复制到newfd所在的文件描述符表项

- 相关代码
    ```c
    SYSCALL_DEFINE2(dup2, unsigned int, oldfd, unsigned int, newfd)
    {
        if (unlikely(newfd == oldfd)) { /* corner case */
            struct files_struct *files = current->files;
            struct file *f;
            int retval = oldfd;

            rcu_read_lock();
            f = __fget_files_rcu(files, oldfd, 0);
            if (!f)
                retval = -EBADF;
            rcu_read_unlock();
            if (f)
                fput(f);
            return retval;
        }
        return ksys_dup3(oldfd, newfd, 0);
    }


    /*
    * We don't use read_sysreg() as we want the compiler to cache the value where
    * possible.
    */
    static __always_inline struct task_struct *get_current(void)
    {
        unsigned long sp_el0;
        // 这里使用mrs寄存器读sp_el0，它的当前值为什么指向了被中断的task?
        // 按照目前的理解：sp_el0为用户态的栈指针，是在变化的；而对应的task_struct的地址应该是一个确定的值！
        // 两者是怎么对应起来的？
        asm ("mrs %0, sp_el0" : "=r" (sp_el0));

        return (struct task_struct *)sp_el0;
    }

    #define current get_current()
    ```
- 疑惑探索
    - .req 汇编伪指令
        ```asm
        /* GPRs used by entry code */
        tsk	.req	x28		// current thread_info
        // 伪汇编：name .req register_name -> 为寄存器定义一个别名
        ```

    - stp指令：stp reg1, reg2, ptr_: 将寄存器reg1, reg2的值存储到ptr_指示的地址中

    - ldr_this_cpu 汇编宏: \arch\arm64\include\asm\assembler.h
        作用为加载percpu的变量值到dst
        ```asm
        /*
        * @dst: Result of READ_ONCE(per_cpu(sym, smp_processor_id()))
        * @sym: The name of the per-cpu variable
        * @tmp: scratch register
        */
        .macro ldr_this_cpu dst, sym, tmp
        adr_l	\dst, \sym
        get_this_cpu_offset \tmp
        ldr	\dst, [\dst, \tmp]
        .endm
        ```

    - __entry_task: percpu变量，用于记录当前执行的用户task_struct
        ```c
        /*
        * We store our current task in sp_el0, which is clobbered by userspace. Keep a
        * shadow copy so that we can restore this upon entry from userspace.
        *
        * This is *only* for exception entry from EL0, and is not valid until we
        * __switch_to() a user task.
        */
        DEFINE_PER_CPU(struct task_struct *, __entry_task);

        static void entry_task_switch(struct task_struct *next)
        {
            // 这里可以看到，每次切换任务时，都会将该percpu变量的值更新为待切换任务的task_struct地址
            __this_cpu_write(__entry_task, next);
        }
        ```

    - pt_regs：软件保护的异常现场结构：arch\arm64\include\asm\ptrace.h
        ```c
        /*
        * This struct defines the way the registers are stored on the stack during an
        * exception. Note that sizeof(struct pt_regs) has to be a multiple of 16 (for
        * stack alignment). struct user_pt_regs must form a prefix of struct pt_regs.
        */
        struct pt_regs {
            union {
                struct user_pt_regs user_regs;
                struct {
                    u64 regs[31];
                    u64 sp;
                    u64 pc;
                    u64 pstate;
                };
            };
            u64 orig_x0;
        #ifdef __AARCH64EB__
            u32 unused2;
            s32 syscallno;
        #else
            s32 syscallno;
            u32 unused2;
        #endif
            u64 sdei_ttbr1;
            /* Only valid when ARM64_HAS_GIC_PRIO_MASKING is enabled. */
            u64 pmr_save;
            u64 stackframe[2];

            /* Only valid for some EL1 exceptions. */
            u64 lockdep_hardirqs;
            u64 exit_rcu;
        };
        ```


    - kernel_entry宏解析: 位于el0/1t_64_sync/irq中
        ```asm
        .macro	kernel_entry, el, regsize = 64
        .if	\el == 0
        alternative_insn nop, SET_PSTATE_DIT(1), ARM64_HAS_DIT
        .endif
        .if	\regsize == 32
        mov	w0, w0				// zero upper 32 bits of x0
        .endif
        stp	x0, x1, [sp, #16 * 0]   // 从用户态陷入内核时触发el0t_sync/irq，因此当前的sp已经指向了sp_el1
        stp	x2, x3, [sp, #16 * 1]   // 接上，此时sp_el1应该指向当前任务的内核栈：应该有一种机制在任务切换时，将sp_el1设置为待切换任务的内核栈
        stp	x4, x5, [sp, #16 * 2]
        stp	x6, x7, [sp, #16 * 3]
        stp	x8, x9, [sp, #16 * 4]
        stp	x10, x11, [sp, #16 * 5]
        stp	x12, x13, [sp, #16 * 6]
        stp	x14, x15, [sp, #16 * 7]
        stp	x16, x17, [sp, #16 * 8]
        stp	x18, x19, [sp, #16 * 9]
        stp	x20, x21, [sp, #16 * 10]
        stp	x22, x23, [sp, #16 * 11]
        stp	x24, x25, [sp, #16 * 12]
        stp	x26, x27, [sp, #16 * 13]
        stp	x28, x29, [sp, #16 * 14]

        .if	\el == 0
        clear_gp_regs
        mrs	x21, sp_el0                             // 这里获取用户态的栈指针
        ldr_this_cpu	tsk, __entry_task, x20      // 加载percpu变量，用于记录当前执行的用户task_struct
        msr	sp_el0, tsk                             // 将tsk的值加载到sp_el0，供后续get_current()使用
        ...
        scs_load_current
        .else                       // 注意这里.else处理异常发生在内核态的情况
        add	x21, sp, #PT_REGS_SIZE  // 栈没有发生切换，因此栈底直接+保存的pt_regs的大小即可
        get_current_task tsk        // 获取当前正在执行的任务；直接取sp_el0获取，因为之前一定发生过用户态陷入内核态的操作
        .endif /* \el == 0 */
        mrs	x22, elr_el1            // 异常发生时，硬件将异常返回地址保存到了elr_elx寄存器中。同步异常：当前地址；异步：next
        mrs	x23, spsr_el1           // 异常发生时，硬件将PSTATE值保存到spsr_elx，用于异常返回时，恢复PE的状态
        stp	lr, x21, [sp, #S_LR]    // 对于用户态陷入内核的情况，此时x21保存的是sp_el0，即用户态栈地址。lr保存到x[30], x21保存到sp
        ...
        stp	x22, x23, [sp, #S_PC]   // 保存elr_el1, spsr_el1到内核栈的相应栈帧位置处
        ...
        /*
        * Registers that may be useful after this macro is invoked:
        *
        * x20 - ICC_PMR_EL1
        * x21 - aborted SP
        * x22 - aborted PC
        * x23 - aborted PSTATE
        */
        .endm
        ```
    


