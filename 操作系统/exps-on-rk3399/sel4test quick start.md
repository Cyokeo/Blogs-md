# references
- https://docs.sel4.systems/projects/buildsystem/host-dependencies.html

# 构建体系
## elfloader-tool
### 重要编译参数
1. 如果前一级是EFI格式的加载器，则编译为位置无关码；前一级EFI Loader会进行重定位
2. 如果前一级是Bare Loader，则需要编译为位置相关码，即最终的elf文件中没有重定位段，使用编译选项`-fno-pic, -fno-pie`

elfloader中有一个cpio archive，其中包含两个image：kernel，rootserver

### 链接脚本
elfloader的cmake脚本中会根据模版生成一个链接脚本
tools/seL4/elfloader-tool/src/linker.lds -> build_aarch64/elfloader/linker.lds_pp

## kernel

## kernel dtb
需要在板级配置文件夹的config.cmake中为变量`KernelDTSList`添加值，即添加板级的dts文件

也可以从外部，给变量`KernelCustomDTS`赋值以覆盖默认的板级dts文件

此外，还可以从外部，给变量`KernelCustomDTSOverlay`赋值以对板级dts文件进行补充

对上述三部分文件进行编译，获得最终的dtb二进制文件，并写入`KernelDTBPath`中

如果配置`ElfloaderIncludeDtb`为ON，则`KernelDTBPath`最终也会被加入kernel，rootserver所在的cpio archive中

## rootserver
rootserver在编译时，会打包所有app的image到一个cpio archive中并将其链接到rootserver的image中

# 关键配置
- `ElfloaderImage`：elfloader的镜像格式Elf、Bin、EFi等
- `IMAGE_START_ADDR`：elfloader的入口链接地址；在sel4test中，不同板级有默认配置
