---
title: seL4 - 内核栈【切换】与任务切换
categories: 操作系统
---
## 内核栈存储？
1. 在不开启Hypervisior的情况下，kernel stack存储在tpid_el1寄存器中；否则存储在tpid_rl2中
2. 响应异常时，根据需要先将用户态上下文保存在`tcb->tcbArch.tcbContext.registers`指向的地址处；
	- 需要注意的是：在切换任务时，会将sp设置为`tcb->tcbArch.tcbContext.registers`【也即设置了sp_el1】；因此，在陷入内核时（el1），sp会自动链接到sp_el1，此时sp指向的也就是当前线程的`tcb->tcbArch.tcbContext.registers`结构处
	- 但是响应异常时，执行c代码之前，会在汇编代码中使用`lsp_i`再次更改sp的值【sp_el1，sp_el2】为真正的内核栈地址 -> 而该内核栈地址的值存储在tpid_el1/2寄存器中
3. 内存布局与Linux中thread_info存储在task_struct中，kernel stack存储在其它位置的布局[[Linux-aarch64-任务切换与内核栈]]很像，但是也有差别：
	- Linux内核栈地址存储在thread_info中 vs seL4的内核栈地址存储在tpid_el1/2中
	- 陷入内核时：Linux的sp已经更新到了内核栈；而seL4的sp更新到`tcb->tcbArch.tcbContext.registers`，后续通过显示的汇编代码`lsp_i`将sp的值更改为tpid_el1/2中的值
	- 任务切换时：Linux要先切内核栈，之后再通过部分代码，切换到用户态；seL4任务切换时，直接将sp的值更新为待切换任务的`tcb->tcbArch.tcbContext.registers`，之后恢复用户态上下文到相应的寄存器中，之后就通过`eret`从内核态返回了
4. seL4只有一个公共的内核栈 vs Linux每个task有自己的内核栈
	1. boot阶段调用`setKernelStack(stack_top);`【aarch64】进行设置