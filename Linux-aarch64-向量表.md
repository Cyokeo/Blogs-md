---
title: Linux-aarch64-向量表
date: 2024-07-10 19:02:57
tags:
categories:
- [Linux相关]
---

# arm64(aarch64)的中断入口

- 总结
    - 部分汇编码，由宏kernel_ventry定义
    - el0t_64_irq
    - el0t_64_irq_handler
    - ret_to_user/kernel

- 相关两个宏定义：下面 \el，对宏参数展开

    ```asm
        .macro entry_handler el:req, ht:req, regsize:req, label:req
    SYM_CODE_START_LOCAL(el\el\ht\()_\regsize\()_\label)
        kernel_entry \el, \regsize
        mov	x0, sp
        bl	el\el\ht\()_\regsize\()_\label\()_handler      // 这里先调用el0t_64_irq_handler[在entry-common.c中定义]，bl是带返回的跳转[lr存储返回地址]
        .if \el == 0
        b	ret_to_user                                    // 返回用户空间
        .else
        b	ret_to_kernel                                  // 返回内核空间
        .endif
    SYM_CODE_END(el\el\ht\()_\regsize\()_\label)
        .endm
    
    // 这里使用上面的宏，定义了不同等级的异常入口：例如，el0t_64_irq， el0t_32_sync
    /*
    * Early exception handlers
    */
        entry_handler	1, t, 64, sync
        entry_handler	1, t, 64, irq
        entry_handler	1, t, 64, fiq
        entry_handler	1, t, 64, error

        entry_handler	1, h, 64, sync
        entry_handler	1, h, 64, irq
        entry_handler	1, h, 64, fiq
        entry_handler	1, h, 64, error

        entry_handler	0, t, 64, sync
        entry_handler	0, t, 64, irq
        entry_handler	0, t, 64, fiq
        entry_handler	0, t, 64, error

        entry_handler	0, t, 32, sync
        entry_handler	0, t, 32, irq
        entry_handler	0, t, 32, fiq
        entry_handler	0, t, 32, error
    
    // 下面的宏使用，定义了向量表

    /*
    * Exception vectors.
    */
        .pushsection ".entry.text", "ax"

        .align	11
    SYM_CODE_START(vectors)
        kernel_ventry	1, t, 64, sync		// Synchronous EL1t
        kernel_ventry	1, t, 64, irq		// IRQ EL1t
        kernel_ventry	1, t, 64, fiq		// FIQ EL1t
        kernel_ventry	1, t, 64, error		// Error EL1t

        kernel_ventry	1, h, 64, sync		// Synchronous EL1h
        kernel_ventry	1, h, 64, irq		// IRQ EL1h
        kernel_ventry	1, h, 64, fiq		// FIQ EL1h
        kernel_ventry	1, h, 64, error		// Error EL1h

        kernel_ventry	0, t, 64, sync		// Synchronous 64-bit EL0
        kernel_ventry	0, t, 64, irq		// IRQ 64-bit EL0
        kernel_ventry	0, t, 64, fiq		// FIQ 64-bit EL0
        kernel_ventry	0, t, 64, error		// Error 64-bit EL0

        kernel_ventry	0, t, 32, sync		// Synchronous 32-bit EL0
        kernel_ventry	0, t, 32, irq		// IRQ 32-bit EL0
        kernel_ventry	0, t, 32, fiq		// FIQ 32-bit EL0
        kernel_ventry	0, t, 32, error		// Error 32-bit EL0
    SYM_CODE_END(vectors)

        .macro kernel_ventry, el:req, ht:req, regsize:req, label:req
        .align 7
    .Lventry_start\@:
        .if	\el == 0
        /*
        * This must be the first instruction of the EL0 vector entries. It is
        * skipped by the trampoline vectors, to trigger the cleanup.
        */
        b	.Lskip_tramp_vectors_cleanup\@
        .if	\regsize == 64
        mrs	x30, tpidrro_el0
        msr	tpidrro_el0, xzr
        .else
        mov	x30, xzr
        .endif
    .Lskip_tramp_vectors_cleanup\@:
        .endif

        sub	sp, sp, #PT_REGS_SIZE
    #ifdef CONFIG_VMAP_STACK
        /*
        * Test whether the SP has overflowed, without corrupting a GPR.
        * Task and IRQ stacks are aligned so that SP & (1 << THREAD_SHIFT)
        * should always be zero.
        */
        add	sp, sp, x0			// sp' = sp + x0
        sub	x0, sp, x0			// x0' = sp' - x0 = (sp + x0) - x0 = sp
        tbnz	x0, #THREAD_SHIFT, 0f
        sub	x0, sp, x0			// x0'' = sp' - x0' = (sp + x0) - sp = x0
        sub	sp, sp, x0			// sp'' = sp' - x0 = (sp + x0) - x0 = sp
        b	el\el\ht\()_\regsize\()_\label                                  // 前面如果成功的话，会跳转到相应的前面定义的elnt_size_name：el0t_64_irq

    0:
        ......
    ```

