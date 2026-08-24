1. 参考板工程`rockpro64`
2. u-boot暂时使用`go`指令进行kernel镜像跳转
3. 可以使用build.ninja查看每个文件的构建命令
# entry
## elfloader-tool
入口：`tools/seL4/elfloader-tool/src/arch-arm/64/crt0.S:_start`
### 关键汇编代码分析
```asm
#ifdef CONFIG_IMAGE_BINARY
	/* Store our original arguments before calling subroutines */
    stp     x0, x1, [sp, #-16]!  // 从这里看到，x0，x1存储了重要信息 <- 需要前一级加载器传入！！！
	// ...
	// 如果elfloader的镜像格式为bin，则加载地址与链接地址可能不一致，需要自我搬运[注意这里不是重定位]到正确的位置：如果搬运失败 -> 
	// 因为bin格式一定为位置相关码
#endif

b main
```
### main关键代码分析
`sel4test/tools/seL4/elfloader-tool/src/arch-arm/sys_boot.c`
```c
int initialise_devices(void)
{
	for (unsigned int i = 0; i < ARRAY_SIZE(elfloader_devices); i++) {
		int ret = init_device(&elfloader_devices[i]);
		if (ret) {
			return ret;
		}
	}
	return 0;
}

void main(UNUSED void *arg)
{
	initialise_devices(); // 最重要的是UART初始化
	
#if defined(CONFIG_IMAGE_UIMAGE)
/* U-Boot passes a DTB. Ancient bootloaders may pass atags. When booting via
* bootelf argc is NULL.
*/
	if (arg && (DTB_MAGIC == *(uint32_t *)arg)) {
		bootloader_dtb = arg;
	}
#endif
}
```

#### 从DTB生成devices
有两个关键文件：
- tools/hardware.yml：架构默认设备
- tools/hardware_schema.yml：SoC级设备定义schema
- kernel/tools/hardware_gen.py：文件生成py脚本
kernel/config.cmake会根据这两个文件以及kernel.dtb生成`devices_gen.h`，其中就包含了`elfloader_devices`的定义

```python
kernel/tools/hardware_gen.py --hardware-config tools/hardware.yml --hardware-schema tools/hardware_schema.yml --elfloader
 --elfloader-out build/elfloader/gen_headers/devices_gen.h
```

在kernel.dts中包含platform配置，其中使用下面的配置指示elfloader需要使用到的devices
```dts
/ {
	chosen {
		seL4,elfloader-devices =
		    "serial2",
		    &{/psci},
		    &{/timer};
		seL4,kernel-devices =
		    "serial2",
		    &{/interrupt-controller@fd400000},
		    &{/timer};
	};
};
```

elfloader需要的所有device-drivers使用`ELFLOADER_DRIVER`进行定义，通过常见的list_start/list_end链接符号包裹起来

显然 -> 可能需要实现一个uart driver！！！
查看u-boot和sel4中driver的compatible字段，都有`snps,dw-apb-uart`，因此可以直接使用sel4中的uart驱动了！

-> 可以跑一下了，应该有输出了。注意，使用的`printf`实现也在elfloader中！！！

## kernel
```c
define PHYS_BASE_RAW 0x10000000 // kernel的物理基地址
```

# 修改点
1. `sel4test/tools/seL4/cmake-tool/helpers/application_settings.cmake`，把rockpro64加入binary list；并添加下面的设置。设置elfloader的起始地址。-> 后续Kernel的起始地址，根据DTS来看，也是被设置到了x010000000；这里elfloader原来起始地址为0x10000000，会与后续DTB的加载地址有重叠，导致sel4 kernel启动失败
```cmake
if(KernelPlatformRockpro64)
	set(IMAGE_START_ADDR
	0xc000000 # 0x10000000
	CACHE INTERNAL "" FORCE
)
endif()
```
2. 上述修改做完，把sel4test/build/images/sel4test-driver-image-arm-rockpro64加载到0xc000000后，再使用u-boot的go指令，就可以成功进入sel4的世界了
## uboot 启动参数


# elfloader使用哪一个serial
在`sel4test/kernel/src/plat/rockpro64/overlay-rockpro64.dts`文件中有定义
```dts
/ {
	chosen {
		seL4,elfloader-devices =
		    "serial2",
		    &{/psci},
		    &{/timer};
		seL4,kernel-devices =
		    "serial2",
		    &{/interrupt-controller@fee00000},
		    &{/timer};
	};
};
```

### uboot使用哪一个
对应的，在uboot的dts中会使用`stdout-path`进行指定；在`u-boot/drivers/serial/serial-uclass.c::serial_init()->serial_find_console_or_panic()->serial_check_stdout()`中会尝试从DTB中获取