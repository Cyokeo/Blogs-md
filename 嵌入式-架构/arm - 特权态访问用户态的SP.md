
# 参考
https://stackoverflow.com/questions/324704/arm-access-user-r13-and-r14-from-supervisor-mode/324928#324928

When doing a STM, if r15 isn't one of the operands then ^ gives access to user-mode registers. However, autoincrementing doesn't seem to work within the instruction, and a nop is required afterwards if you want to access the register bank.
Something like
```asm
stmfd r13, {r13-r14}^ ;store r13 and r14 usermode
nop
sub r13, r13, #8 ;update stack pointer
```

# 总结
1. 在STM/LDM指令中，如果寄存器列表中没有r15「即pc」，则^意味着访问user mode的寄存器
2. 如果含有r15，则表示同时加载/或保存SPSR寄存器