# 重要参考
1.  [Rockchip RK3399 - 引导流程和准备工作](https://www.cnblogs.com/zyly/p/17380243.html "发布于 2023-05-07 21:45")
2. [Rockchip RK3399 - 官方固件方式加载uboot](https://www.cnblogs.com/zyly/p/17403323.html "发布于 2023-05-15 22:16")
3. [Rockchip RK3399 - TPL/SPL方式加载uboot](https://www.cnblogs.com/zyly/p/17389525.html "发布于 2023-05-11 01:02")
4. [Rockchip Linux partition definition](https://opensource.rock-chips.com/wiki_Partitions "Partitions")
5. [Rockchip Linux 软件开发指南](https://file.elecfans.com/web2/M00/80/9D/poYBAGONqvGAef3hAC9xhk_dlOw443.pdf)
- 

# 仓库
1. https://github.com/rockchip-linux/u-boot.git -> next_dev 分支
2. https://github.com/friendlyarm/rkbin

# u-boot编译
def config
```bash
make evb-rk3399_defconfig V=1
```
修改.config，uart 波特率为115200，uboot命令等待时间`CONFIG_BOOTDELAY`
```
CONFIG_BAUDRATE=115200
CONFIG_BOOTDELAY=10
```
编译及输出
```bash
make CROSS_COMPILE=aarch64-linux-gnu-

# u-boot.bin, spl/*spl.bin, tpl/*tpl.bin
```

# 多个镜像打包
## rk loader
`SoC`进入到`MASKROM`模式后，目标板子会运行`Rockusb`驱动程序。在`MASKROM`模式下，需要使用到`DDR`，因此需要下载固件进行`DDR`的初始化。
- `ddr.bin`：等价于`TPL`，用于初始化`DDR`；
- `usbplug.bin`：`Rockusb`驱动程序，用于将程序通过`usb`下载到`eMMC`；
- `miniloader.bin`：`Rockchip`修改的一个`bootloader`，等价于`SPL`，用于加载`uboot`；
```bash
cd rkbin
tools/boot_merger ./RKBOOT/RK3399MINIALL.ini
# out: rk3399_loader_v1.24.126.bin
```
## idbloader
### uboot官方TPL/SPL方式 -> 对应后续的u-boot.itb
基于`uboot`源码编译出`TPL/SPL`，其中`TPL`负责实现`DDR`初始化，`TPL`初始化结束之后会回跳到`BootROM`程序，`BootROM`程序继续加载`SPL`，`SPL`加载`u-boot.itb`文件，然后跳转到`uboot`执行。`idbloader.img`是由`tpl/u-boot-tpl.bin`和`spl/u-boot-spl.bin`文件生成。需要使用u-boot/tools/mkimage工具进行打包
```bash
cd rkbin
tools/mkimage -n rk3399 -T rksd -d tpl/u-boot-tpl.bin idbloader.img
#-T rksd: Rockchip SD卡启动映像类型
cat spl/u-boot-spl.bin >> idbloader.img
```
### rkchip独有的方式 -> 对应后续的u-boot.img & trust.img
```bash
cd rkbin
tools/mkimage -n rkxxxx -T rksd -d rkxx_ddr_vx.xx.bin idbloader.img
cat rkxx_miniloader_vx.xx.bin >> idbloader.img
```
### 总结
这里我们可以知道，tpl的作用就是初始化DDR；而SPL的作用就是加载u-boot到DDR，并跳转到u-boot执行
⚠️从下面的实验来看，似乎官方TPL具有rkchip DDR初始化的能力？-> 需要读代码看一下

## u-boot.img & trust.img
使用rkchip独有的生成idbloader模式时，需要把后续的镜像打包为rkchip miniloader能识别的格式
```bash
cd u-boot
tools/loaderimage --pack --uboot u-boot.bin uboot.img $SYS_TEXT_BASE
# out: u-boot.img
cd u-boot
tools/trustmerge ../rkbin/*/RKTRUST_RKXXXXTRUST.ini
# out: trust.img
```
这里的打包可以参考./make.sh脚本，做了封装

## u-boot.itb
在使用uboot官方方式的idbloader时，后续引导需要itb方式

`u-boot.itb`实际上是`u-boot.img`的另一个变种，也是通过`mkimage`构建出来的，依赖于`u-boot.its u-boot.dtb u-boot-nodtb.bin`这三个文件。

`mkimage`工具在`u-boot`源码目录下的`tools`目录中，不过由于`u-boot`官方原本的`FIT`功能无法满足实际产品需要，所以`RK`平台对`FIT` 功能进行了适配和优化，所以自然对`mkimage`工具的源代码进行了修改、优化；所以对于`RK`平台硬件，如果使用`FIT`格式镜像，必须使用`RK u-boot`源码编译生成的`mkimage`工具，不可使用`u-boot`原版的`mkimage`工具。

这里之前使用u-boot官方仓库编译时出现错误：rkchip提供了bl31.elf，bl32.bin；但是uboot官方程序在打包时处理不了.bin镜像的情况。

`FIT`是`flattened image tree`的简称，它采用了[`device tree source filse（DTS）`](https://www.cnblogs.com/zyly/p/17266960.html#_label2_0)的语法，生成的`image`文件也和`dtb`文件类似（称做`itb`）。

### mkimage -E选项
其中`-E`这个字段比较重要，它会影响生成的`itb`的文件布局；
- 如果没有指定该选项，其生成的`itb`文件格式和`dts`文件编译生成的`dtb`文件布局一样，包括`data`属性指定的`/incbin/("bl31_0x00040000.bin")`文件也会以二进制数据格式的形式放到`FIT`结构内；
- 如果指定了该选项，会为`data`属性指向的文件扩充`data-offset`（指定文件的偏移，这个偏移是以`FIT`结构结束地址下一扇区起始地址开始计算的，即相对于`fdt_blob`末尾的位置偏移量）、以及`data-size`（指定文件的大小）属性，而在`data`属性指向的二进制数据文件将会被追加到`FIT`结构的尾部（也是扇区对齐）
```bash
./tools/mkimage -f u-boot.its -E u-boot.itb
```

rkuboot仓库会动态生成u-boot.its，生成的脚本为`make_fit_atf.py`。比较麻烦的是：
1）这个py只能使用python2.7进行（目前使用Fedora43，官方包管理器已经废弃了python2的支持）
2）因此需要使用conda创建虚拟环境
3）依赖`pyelftools`模块，其在0.3版本后废弃了对python2的支持，因此pip安装时需要指定0.3之前的版本，例如0.29

因为rk3399是armv8-A架构，还需要提供ATF固件。这里有两种方式：
1）使用上游提供的开源ATF源码进行编译 -> 需要注意的是，这里只能生成bl31
2）使用rkbin中的elf二进制文件

### BL31
配置atf生成脚本如下，仅支持bl31.elf
```
CONFIG_SPL_FIT_GENERATOR="arch/arm/mach-rockchip/make_fit_atf.py"
```

### BL31 & BL32
配置如下时，还支持tee.bin(BL32)；目前这里只能使用rkbin中的*bl32*.bin
```
CONFIG_SPL_FIT_GENERATOR="arch/arm/mach-rockchip/make_fit_atf.sh"
```

### 编译生成u-boot.itb
需要把bl31.elf，tee.bin拷贝到u-boot根目录下
```bash
make u-boot.itb CROSS_COMPILE=aarch64-linux-gnu- 
```

# rkdevtool烧录
## 进入maskrom模式
长按boot按钮，然后reset上电，就会进入maskrom模式

## !!! 烧录rkloader
使用下载引导命令去使目标机器初始化`DD`R与运行`usbplug`（初始化`DDR`的原因是由于升级需要很大的内存，所以需要使用到`DDR`）
>这个步骤的目的！
>同rkloader小节模式：芯片进入maskrom模式时，将会运行usb驱动程序；后续在`MASKROM`模式下，需要使用到`DDR`，因此需要下载固件进行`DDR`的初始化
```bash
sudo rkdeveloptool db rkloader.bin
```
是的！后续烧录前都需要先执行
## 烧录后续的img
```bash
sudo rkdeveloptool wl 0x40 idbloader.img
sudo rkdeveloptool wl 0x4000 u-boot.itb
```
至于烧录的镜像的位置，需要参考：[Rockchip Linux partition definition](https://opensource.rock-chips.com/wiki_Partitions "Partitions")

## 分区要求
| Partition                      | Start Sector |              | Number of Sectors |              | Partition Size |           | PartNum in GPT | Requirements                             |
| ------------------------------ | ------------ | ------------ | ----------------- | ------------ | -------------- | --------- | -------------- | ---------------------------------------- |
| MBR                            | 0            | 00000000     | 1                 | 00000001     | 512            | 0.5KB     |                |                                          |
| Primary GPT                    | 1            | 00000001     | 63                | 0000003F     | 32256          | 31.5KB    |                |                                          |
| **loader1**                    | **64**       | **00000040** | **7104**          | **00001bc0** | **4096000**    | **2.5MB** | **1**          | **preloader (miniloader or U-Boot SPL)** |
| Vendor Storage                 | 7168         | 00001c00     | 512               | 00000200     | 262144         | 256KB     |                | SN, MAC and etc.                         |
| Reserved Space                 | 7680         | 00001e00     | 384               | 00000180     | 196608         | 192KB     |                | Not used                                 |
| reserved1                      | 8064         | 00001f80     | 128               | 00000080     | 65536          | 64KB      |                | legacy DRM key                           |
| U-Boot ENV                     | 8128         | 00001fc0     | 64                | 00000040     | 32768          | 32KB      |                |                                          |
| reserved2                      | 8192         | 00002000     | 8192              | 00002000     | 4194304        | 4MB       |                | legacy parameter                         |
| **loader2**                    | **16384**    | **00004000** | **8192**          | **00002000** | **4194304**    | **4MB**   | **2**          | **U-Boot or UEFI**                       |
| **trust**                      | **24576**    | **00006000** | **8192**          | **00002000** | **4194304**    | **4MB**   | **3**          | **trusted-os like ATF, OP-TEE**          |
| **boot（bootable must be set）** | **32768**    | **00008000** | **229376**        | **00038000** | **117440512**  | **112MB** | **4**          | **kernel, dtb, extlinux.conf, ramdisk**  |
| **rootfs**                     | **262144**   | **00040000** | **-**             | **-**        | **-**          | **-MB**   | **5**          | **Linux system**                         |
| Secondary GPT                  | 16777183     | 00FFFFDF     | 33                | 00000021     | 16896          | 16.5KB    |                |                                          |
⚠️⚠️⚠️
>1. If preloader is miniloader, loader2 partition available for uboot.img and trust partition available for trust.img; 
>2. if preloader is SPL without trust support, loader2 partition is available for u-boot.bin and trust partition not available;
>3. If preloader is SPL with trust support(ATF or OPTEE), loader2 is available for u-boot.itb(including u-boot.bin and trust binary) and trust partition not available.

### 似乎可以改变！！！
- [Write GPT partition table through rkdeveloptool](https://opensource.rock-chips.com/wiki_Partitions#Write_GPT_partition_table_through_rkdeveloptool)
- [Write GPT partition table through U-boot](https://opensource.rock-chips.com/wiki_Partitions#Write_GPT_partition_table_through_U-boot )
- Write GPT partition table through U-Boot's fastboot
	- The current upstream u-boot contains fastboot protocol support. And this version of fastboot supports 2 ways to modify gpt partition table:
