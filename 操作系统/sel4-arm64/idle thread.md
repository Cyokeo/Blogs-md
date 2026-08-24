## 创建
- 每个CPU node都有一个idle thread
- 线程入口：`idle_thread`
## 源码分析
`kernel/boot.c::create_idle_thread()`
```c
pptr = (pptr_t) &ksIdleThreadTCB[SMP_TERNARY(i, 0)];
NODE_STATE_ON_CORE(ksIdleThread, i) = TCB_PTR(pptr + TCB_OFFSET);
// 从这里也能看出ThreadTCBTCB控制块前面有一部分预留空间信息

BOOT_CODE void Arch_configureIdleThread(tcb_t *tcb)
{
setRegister(tcb, SPSR_EL1, PSTATE_IDLETHREAD);
// 这里可以看到idle thread的线程入口为 idle_thread()
setRegister(tcb, ELR_EL1, (word_t)&idle_thread);
// 这里将其写到ELR_EL1寄存器中，后续会使用eret进行线程resume
}

void setThreadState(tcb_t *tptr, _thread_state_t ts)
{
thread_state_ptr_set_tsType(&tptr->tcbState, ts);
scheduleTCB(tptr);
}

```
