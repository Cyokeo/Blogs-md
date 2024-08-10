---
title: Linux-aarch64-任务切换与内核栈
date: 2024-07-24 19:47:56
tags:
categories:
- [Linux相关]
---

# 任务切换

* task_struct->stack成员指向的内存空间就是内核栈

* 陷入内核时，用户任务上下文保存在其内核栈上：**且保存的位置也是固定的，就在内核栈空间最上方预留的的pt_regs区域**
    - 参考《vectors.md》种entry_handler的定义：`mov	x0, sp; bl	el\el\ht\()_\regsize\()_\label\()_handler`
    - 参考 `kernel_entry`宏的定义中，用户上下文的保存 `...; stp	x2, x3, [sp, #16 * 1]; ...`
    - 综合可知，用户任务上下文保存在**当前内核栈指针的上方**
    - 因此，进程工作在用户态时，其内核栈指针应始终指向内核栈空间最高处-sizeof(pt_regs)处？**这里可能会有些问题**
        - 刚创建该任务，而且还没有被调度运行时，应该是这样的
    - **但是当该任务被调度运行后**：
        1. 首先在内核态切换到该任务的内核栈：首次切换到内核栈初期【汇编码范围内】sp确实指向了`内核栈空间最高处-sizeof(pt_regs)`
        2. 接着执行部分内核态代码，此时会有入/出栈操作
            * 这里要注意：首次进入内核态后，之后会向普通函数调用那样，借助`lr`寄存器进行函数返回
            * 注意到任务创建时，任务结构体中固定位置处【保存内核态上下文的结构】将**pc = ret_from_fork**，因此任务首次被调度执行 -> 首次进入内核态时执行的第一个比较重要的函数就是`ret_from_fork`
        3. 如果任务在内核态被抢占 -> 仍然要把内核态上下文保存到任务结构体的固定位置处？
        4. 之后该任务恢复后，依然继续之前内核态的执行
        5. 再之后需要从内核态返回用户空间：`ret_to_user/ret_to_fork`: 这里面需要把内核栈上保存的用户空间上下文恢复
        6. 需要区分对待第一次返回到用户空间/第二次返回到用户空间：因为第一次返回时内核栈上没有保存的用户上下文？
            - 这里还需要再深入看一下创建任务时，有没有为内核栈模拟保存的用户空间上下文！！！？？？❗️❗️
            - 深入看一下`ret_from_fork`
        

* 任务切换时，内核态上下文保存在任务结构体的固定位置处：THREAD_CPU_CONTEXT
    - 新建任务时，要把其内核栈地址放在其任务结构体成员cpu_context.sp处；-> 再去看一下新建任务时，sp的赋值情况，就可以探索出内核栈的初始内存分配情况；
    - 根据上面的叙述：`要把用户上下文保存在内核栈上方【高地址】位置处`，因此在初始创建内核栈时，要注意这一点！！！
    - 要注意：创建任务时，其tsk->thread.cpu_context.sp要向内核栈空间最高地址 - 足够的空间容纳pt_regs：***与后面的代码走读匹配了***
    ```c
    DEFINE(THREAD_CPU_CONTEXT,	offsetof(struct task_struct, thread.cpu_context));
    struct task_struct {
        ......
        void   *stack;      // 通过查找，明确了这里为task的内核栈?栈顶（往下增长）

        /* CPU-specific state of this task: */
        // 这是一个架构相关的结构体
        // 这个成员位于任务结构体的末尾
        struct thread_struct		thread;

        /*
        * WARNING: on x86, 'thread_struct' contains a variable-sized
        * structure.  It *MUST* be at the end of 'task_struct'.
        *
        * Do not put anything below here!
        */
    };
    struct cpu_context {
        unsigned long x19;
        unsigned long x20;
        unsigned long x21;
        unsigned long x22;
        unsigned long x23;
        unsigned long x24;
        unsigned long x25;
        unsigned long x26;
        unsigned long x27;
        unsigned long x28;
        unsigned long fp;
        unsigned long sp;
        unsigned long pc;
    }; 
    ```

## 最终的寄存器、栈切换
```c

// in file: arch\arm64\kernel\entry.S
/*
* Register switch for AArch64. The callee-saved registers need to be saved
* and restored. On entry:
*   x0 = previous task_struct (must be preserved across the switch)
*   x1 = next task_struct
* Previous and next are guaranteed not to be the same.
*
*/
SYM_FUNC_START(cpu_switch_to)
    mov	x10, #THREAD_CPU_CONTEXT
    add	x8, x0, x10
    mov	x9, sp                  // 这里将prev的内核栈地址保存到x9寄存器
    stp	x19, x20, [x8], #16		// store callee-saved registers
    stp	x21, x22, [x8], #16
    stp	x23, x24, [x8], #16
    stp	x25, x26, [x8], #16
    stp	x27, x28, [x8], #16
    stp	x29, x9, [x8], #16      // 这里将内核栈当前指针保存到x8寄存器的值指示的地址处
    str	lr, [x8]                // 这里将lr寄存器的值保存到...
    add	x8, x1, x10
    ldp	x19, x20, [x8], #16		// restore callee-saved registers
    ldp	x21, x22, [x8], #16
    ldp	x23, x24, [x8], #16
    ldp	x25, x26, [x8], #16
    ldp	x27, x28, [x8], #16
    ldp	x29, x9, [x8], #16      // 从这里可以看出，新建任务时，要把其内核栈地址放在其任务结构体成员cpu_context.sp处
    ldr	lr, [x8]                // -> 再去看一下新建任务时，sp的赋值情况，就可以探索出内核栈的初始内存分配情况；
    mov	sp, x9                  // 这里将sp的值更新为待切换任务的内核栈
    msr	sp_el0, x1              // 这里将sp_el0的值更新为待切换任务的任务结构体，供后续current()使用
    ptrauth_keys_install_kernel x1, x8, x9, x10
    scs_save x0
    scs_load_current
    ret                         // 这里ret执行正常的借助lr的返回，因此返回到调用cpu_switch_to()的下一条指令
SYM_FUNC_END(cpu_switch_to)
NOKPROBE(cpu_switch_to)


// in file: arch\arm64\kernel\process.c
/*
* Thread switching.
*/
__notrace_funcgraph __sched
struct task_struct *__switch_to(struct task_struct *prev,
                struct task_struct *next)
{
    ......
    /* the actual thread switch */
    last = cpu_switch_to(prev, next);

    return last;
}

```

## 内核栈构建

```c
// in file: kernel\fork.c
static int alloc_thread_stack_node(struct task_struct *tsk, int node)
{
	unsigned long *stack;
	stack = kmem_cache_alloc_node(thread_stack_cache, THREADINFO_GFP, node);
	stack = kasan_reset_tag(stack);
	tsk->stack = stack;                 // 这里申请内核栈，并将地址赋给任务结构体的stack指针
	return stack ? 0 : -ENOMEM;
}

// in file: arch\arm64\include\asm\processor.h
#define task_pt_regs(p) \
	((struct pt_regs *)(THREAD_SIZE + task_stack_page(p)) - 1)

// in file: include\linux\sched\task_stack.h
#define task_stack_page(task)	((void *)(task)->stack)

// in file: arch\arm64\kernel\process.c
int copy_thread(struct task_struct *p, const struct kernel_clone_args *args)
{
    struct pt_regs *childregs = task_pt_regs(p);
    memset(&p->thread.cpu_context, 0, sizeof(struct cpu_context));

    ...

	p->thread.cpu_context.pc = (unsigned long)ret_from_fork;
	p->thread.cpu_context.sp = (unsigned long)childregs;

    ...
}
```


* **仅考虑栈向低地址增长**；结合前面两个宏可以知道：
    - tsk->stack 指向申请的内核栈空间的起始地址【低地址】
    - childregs 指向该内核栈空间的最高地址 - sizeof(struct pt_regs)
    - `p->thread.cpu_context.sp = (unsigned long)childregs`这一行就使得新创建任务的内核栈地址处在内核栈空间的高位，且其上方有一个(struct pt_regs)空间，用于存储用户上下文
