# 几种不同的镜像格式
1. EFI：前一级加载器需要是符合EFI标准的
2. uImage：legacy u-boot image
3. binary：
4. ...

u-boot支持最新的FIT格式，即所有的镜像以类似DTS的格式打包成一个镜像



# ELF Loader执行完成后内存布局
开始时，所有镜像都在ELF Loader内部的CPIO包中；ELF Loader启动后开始进行load images，包括：kernel image，user images(per core)，DTB

## kernel elf image
kernel image是一个elf格式的文件，因此其包含load address和virtual address；而user-space images后续将会运行在虚拟地址空间，因此其被加载到的物理地址无关紧要；ELF loader会将user-space images加载到kernel后面

## DTB
ELF Loader的CPIO内可选包含DTB；DTB也可来自上一级加载器到ELF Loader的传入参数；如果有DTB，则将DTB重新加载到kernel后面


# Relocate
准备开启MMU；确保ELF Loader在kernel虚拟地址空间的下方；因此在开启MMU后，ELF Loader能继续正确执行
此种情况仅在ELF Loader是EFI格式的才会进行


# continue_boot
1. 如果当前处在Hyp模式「EL2」，但是当前没有开始Hypervisor支持的情况下，会主动退出Hyp模式
2. 准备Virtual Space
3. kick other CPUs by arm PSCI interface(SMC call)
4. 启用MMU
	1. 如果是Hyp模式，仅设置boot_pgd_down到ttbr0_el2寄存器上
	2. 如果不是Hyp模式，则设置boot_pgd_down到ttbr0_el1；设置boot_pgd_up到ttbr1_el1；后续CPU/MMU会根据虚拟地址的情况自动选用ttbr0_el1或者ttbr1_el1
5. 跳转到内核虚拟地址

```c
((init_arm_kernel_t)kernel_info.virt_entry)(user_info.phys_region_start,  // user image physical start addr
											user_info.phys_region_end,  // user image physical end addr
											user_info.phys_virt_offset, // user image physical/virtual offset
											user_info.virt_entry, // virtual entry addr
											(word_t)dtb, // physical addr of dtb
											dtb_size);
```

# SMP 多核启动 - `smp_boot`
非boot core会后续执行->`core_entry()`，且boot core会等到各从核起来之后再继续后续的执行
平台相关的core具体启动函数`plat_cpu_on()`；在初始化时，要调用`smp_register_handler()`注册smp_ops。
例如arm平台通用的psci：`sel4test/tools/seL4/elfloader-tool/src/arch-arm/drivers/smp-psci.c`。以设备树的形式进行组织：
1. elfloader阶段的elfloader_devices中可能包含`psci`设备信息，在elfloader设备树驱动初始化过程中，根据compat字段进行驱动匹配。
2. 驱动初始化时调用`smp_psci_init`
3. 最终调用驱动`ops->cpu_on = smp_psci_cpu_on()->psci_cpu_on()`
## PSCI
参考：[PSCI的安全唤醒与TrustZone的密室协约](https://kernel.meizu.com/2025/04/10/Linux%20SMP%E5%90%AF%E5%8A%A8%E7%BD%97%E6%9B%BC%E5%8F%B2(%E4%B8%8B):PSCI%E7%9A%84%E5%AE%89%E5%85%A8%E5%94%A4%E9%86%92%E4%B8%8ETrustZone%E7%9A%84%E5%AF%86%E5%AE%A4%E5%8D%8F%E7%BA%A6/)
1. psci是运行在安全世界的EL3固件，非安全世界通过`SMC`（secure monitor call）调用陷入其中，实现ARM核的安全状态与非安全状态之间的切换，执行PSCI调用
2. 陷入Hypervisior的指令为`HVC`
3. 陷入EL1的指令为`SVC`
4. 需要注意的是EL3固件运行在MMU关闭的环境

从具体的PSCI调用可以看到，AP的执行入口为`secondary_startup()->core_entry()->non_boot_main()`，最后跳转到内核入口

# EL执行等级与安全态/非安全态的区别
在ARMv8架构中使用执行等级（Execution Level，EL）**EL0～EL3来定义ARM核的运行等级**，其中***EL0～EL2等级分为安全态和非安全态***。ARMv8架构与ARMv7架构中ARM核运行权限的对应关系如图所示。EL3只有安全态。
需要注意的是：作为一个可选特性，*Armv8.4-A增加了安全世界下EL2的支持*。支持安全世界EL2的处理器，需配置EL3下的SCR_EL3.EEL2比特位来开启这一特性。设置了这一比特位，才允许使用安全状态下的虚拟化功能。
1. 非安全世界想要切换到安全世界，必须执行SMC指令，陷入到EL3。安全世界状态和正常世界状态之间的切换是由bl31的固件完成
![[Pasted image 20260223123038.png]]

