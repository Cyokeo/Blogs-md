
### sel4的几种启动方式
1. 由elfloader工具拉起 -> 而elfloader之前也可以是bootloader等
2. (sysboot.c -> kernel_info.virt_entry)(): 进入内核入口
	1. head.S: `_start`
	2. 接着进入c函数init_kernel


### 内核的第一个C函数 init_kernel

1. 其中创建idle_thread
2. 创建initial_thread，其入口为elf-loader阶段获取的userelf【例如rootserver】的入口


### rootserver
1. 在sel4官方tutorials的ipc中，rootserver为capdl-loader