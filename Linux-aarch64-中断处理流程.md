---
title: Linux-aarch64-中断处理流程
date: 2024-07-10 19:06:40
tags:
categories:
- [Linux相关]
---

## 中断处理流程

### 总结
* 
    * 
        * 

### aarch64

* 参考《vectors.md》可知，中断响应时，执行部分汇编码[kernel_ventry宏]后，进入`el0t_64_irq`[in file entry.S，用entry_handler宏定义]，紧接着调用
`el0t_64_irq_handler`[in file: entry-common.c]，进行后续的处理

* el0t_64_irq_handler -> __el0_irq_handler_common -> el0_interrupt(regs, handle_arch_irq);
    
    - 这里要注意：`handle_arch_irq`是一个全局的变量，定义在/arch/arm64/kernel/irq.c文件中
        ```c
        void (*handle_arch_irq)(struct pt_regs *) __ro_after_init = default_handle_irq;     // __ro_after_init 是编译器提供的属性
        void (*handle_arch_fiq)(struct pt_regs *) __ro_after_init = default_handle_fiq;

        int __init set_handle_irq(void (*handle_irq)(struct pt_regs *))
        {
            if (handle_arch_irq != default_handle_irq)
                return -EBUSY;

            handle_arch_irq = handle_irq;
            pr_info("Root IRQ handler: %ps\n", handle_irq);
            return 0;
        }
        ``` 
    - 结合`__init set_handle_irq`可知，在系统启动阶段，irqchip的驱动会调用set_handle_irq，根据使用的中断控制器，将handle_arch_irq设置为对应的处理函数

    - 对应的中断处理函数有：gic_handel_irq -> GIC控制器

* gic_handle_irq
    ```c
    // in file: drivers/irqchip/irq-gic-v3.c
    static asmlinkage void __exception_irq_entry gic_handle_irq(struct pt_regs *regs)
    {
        if (unlikely(gic_supports_nmi() && !interrupts_enabled(regs)))
            __gic_handle_irq_from_irqsoff(regs);
        else
            __gic_handle_irq_from_irqson(regs);
    }
    ```

    - 如果不是NMI中断，则__gic_handle_irq_from_irqson -> __gic_handle_irq -> generic_handle_domain_irq. 注意这里的irqnr为从相应寄存器[iar]读出的硬件中断号
        ```c
        static void __gic_handle_irq(u32 irqnr, struct pt_regs *regs)
        {
            if (gic_irqnr_is_special(irqnr))
                return;

            gic_complete_ack(irqnr);

            if (generic_handle_domain_irq(gic_data.domain, irqnr)) {
                WARN_ONCE(true, "Unexpected interrupt (irqnr %u)\n", irqnr);
                gic_deactivate_unhandled(irqnr);
            }
        }
        ```

    - generic_handle_domain_irq : irqdomain 管理hwirq 到 Linux irqnr的映射，有四种映射类型[Documentation\translations\zh_CN\core-api\irq\irq-domain.rst]
        ```c
        // in file: /kernel/irq/irqdesc.c
        int generic_handle_domain_irq(struct irq_domain *domain, unsigned int hwirq)
        {
            return handle_irq_desc(irq_resolve_mapping(domain, hwirq));
        }
        ```

    - handle_irq_desc
        ```c
        int handle_irq_desc(struct irq_desc *desc)
        {
            struct irq_data *data;

            if (!desc)
                return -EINVAL;

            data = irq_desc_get_irq_data(desc);
            if (WARN_ON_ONCE(!in_hardirq() && handle_enforce_irqctx(data)))
                return -EPERM;

            generic_handle_irq_desc(desc);
            return 0;
        }
        ```
    - generic_handle_irq_desc：根据描述符中的内容，执行相应的回调
        ```c
        static inline void generic_handle_irq_desc(struct irq_desc *desc)
        {
            desc->handle_irq(desc);
        }
        ```

### x86-64
