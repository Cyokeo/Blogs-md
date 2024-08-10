---
title: Linux-aarch64-系统调用-2
date: 2024-07-10 19:13:48
tags:
categories:
- [Linux相关]
---

## Linux系统调用的处理
- 注意本系列以Linux6.9为例
- 这里为aarch64

### 总结：

* dup2() of POSIX interface: defined in glibc
    * INLINE_SYSCALL_CALL() -> __INLINE_SYSCALL2() -> INLINE_SYSCALL(): 针对不同的UNIX实现，定义了不同的宏实现
        * for Linux: INLINE_SYSCALL() -> INTERNAL_SYSCALL(): 针对不同的架构，定义了不同的宏实现
            * for aarch64: INTERNAL_SYSCALL_RAW()
                - 将系统调用号放在x8寄存器，将调用参数放到x0-x6寄存器中
                - `svc	0` 触发SVC异常，陷入内核

* 陷入内核后：（SVC异常属于同步异常）
    * 根据vectors，进入el0，64，sync处理流程
        * 执行部分汇编码 -> b(无条件，不返回)跳转到 el0t_64_sync
            * el0t_64_sync (由entry_handler宏定义，汇编标号) -> el0t_64_sync_handler (c函数)
                * el0t_64_sync_handler -> el0_svc -> do_el0_svc
                    * do_el0_svc(从x8获取系统调用号) -> el0_svc_common
                        * el0_svc_common: 从调用表中拿到Linux定义的系统调用函数__arm64_sym_sys_dup2()
                            * 进入执行 SYSCALL_DEFINE2(dup2, unsigned int, oldfd, unsigned int, newfd): in file.c

### 分步梳理

- 参考《vectors.md》可知，系统调用触发软件中断，为同步异常，因此会进入el0t_64_sync的处理流程

- 必要的汇编 -> el0t_64_sync -> el0t_64_sync_handler

- el0t_64_sync_handler
    ```c
    asmlinkage void noinstr el0t_64_sync_handler(struct pt_regs *regs)
    {
        unsigned long esr = read_sysreg(esr_el1);

        switch (ESR_ELx_EC(esr)) {
        case ESR_ELx_EC_SVC64:
            el0_svc(regs);          // 这里处理 svc 同步异常
            break;
        case ESR_ELx_EC_DABT_LOW:
            el0_da(regs, esr);
            break;

        // ......
        }
    }
    ```

- el0_svc -> do_el0_svc
    ```c
    static void noinstr el0_svc(struct pt_regs *regs)
    {
        enter_from_user_mode(regs);
        cortex_a76_erratum_1463225_svc_handler();
        fp_user_discard();
        local_daif_restore(DAIF_PROCCTX);
        do_el0_svc(regs);
        exit_to_user_mode(regs);
    }
    ```
- do_el0_svc -> el0_svc_common -> invoke_syscall
    ```c
    void do_el0_svc(struct pt_regs *regs)
    {
        // 这里从x8中获取调用号; sys_call_table为系统调用表
        el0_svc_common(regs, regs->regs[8], __NR_syscalls, sys_call_table);
    }

    // sys_call_table的“创建”, in file: arch\arm64\kernel\sys.c
    /*
    * Wrappers to pass the pt_regs argument.
    */
    #define __arm64_sys_personality		__arm64_sys_arm64_personality

    #undef __SYSCALL
    #define __SYSCALL(nr, sym)	asmlinkage long __arm64_##sym(const struct pt_regs *);
    #include <asm/unistd.h>

    #undef __SYSCALL
    #define __SYSCALL(nr, sym)	[nr] = __arm64_##sym,

    const syscall_fn_t sys_call_table[__NR_syscalls] = {
        [0 ... __NR_syscalls - 1] = __arm64_sys_ni_syscall,
    #include <asm/unistd.h>
        [23] = __arm64_sym_sys_dup,     // 在《syscall-1.md》的分析中，已经见到了该函数的定义
        [24] = __arm64_sym_sys_dup3
    };

    // 上面两个#include <asm/unistd.h>，最终会包含linux\include\uapi\asm-generic\unistd.h 这个重要文件
    ...
    #define __NR_dup 23
    __SYSCALL(__NR_dup, sys_dup)
    #define __NR_dup3 24
    __SYSCALL(__NR_dup3, sys_dup3)
    #define __NR3264_fcntl 25
    __SC_COMP_3264(__NR3264_fcntl, sys_fcntl64, sys_fcntl, compat_sys_fcntl64)
    ...
    ```

- invoke_syscall：从调用表中所引到某个表项，取得syscall_fn_t，调用即可
    - 对于POSIX标准规范的dup()
        - 调用到__arm64_sym_sys_dup()
            - __se_sys_dup()
                - __do_sys_dup()
                    - file.c 中 dup 的定义：`SYSCALL_DEFINE2(dup2, unsigned int, oldfd, unsigned int, newfd)`

