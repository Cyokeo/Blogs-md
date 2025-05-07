---
title: seL4 - 中断处理
categories: 操作系统
---
## 异常处理c入口
- `c_handle_syscall()` -> 不同架构下，这个函数的名字是不变的，但是具体实现可能会有差别
	- `handleSyscall()` -> 架构无关的实现
		- SysSend
		- SysNBSend
		- SysCall
		- SysRecv
		- ============== for non MCS
		- SysReply
		- SysReplyRecv
		- ============== for MCS
		- SysWait
		- SysNBWait
		- SysReplyRecv
		- SysNBSendRecv
		- SysNBSendWait
		- ============== end
		- SysNBRecv
		- SysYield
- `c_handle_interrupt()` -> 中断