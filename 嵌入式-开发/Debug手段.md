---
title: Linux - 开发可以的debug手段
categories: 嵌入式-开发
---

## 一次真实的内存泄漏检测
### 参考链接
- [Linux如何查找内存泄露和内存占用过大？](https://zhuanlan.zhihu.com/p/715909392?utm_medium=social&utm_psn=1811515281429893123&utm_source=ZHShareTargetIDMore)
### 观察步骤
- `pidstat`观察某个进程的内存使用统计【随时间变化】
	- 还可以观察io/cpu/上下文切换等信息
- `pmap -x <pid>`显示程序的虚拟内存分布及虚拟内存映射情况，得到泄漏内存大小约为4K
- 查到可能是由于malloc或mmap系统调用分配内存导致的
- 使用`strace`跟踪系统调用，并`grep`过滤需要的信息


## 内存检测 -  valgrind
 - 它的一个被广泛使用的默认工具——'Memcheck'——可以拦截malloc()，new()，free()和delete()调用。换句话说，它在检测下面这些问题非常有用：
	- 内存泄露
	- 重释放
	- 访问越界
	- 使用未初始化的内存
	- 使用已经被释放的内存等
	但是也有明显的缺点
	- 增加了内存占用，会减慢你的程序
	- 它有时会造成误报和漏报
	- 它不能检测出***静态分配的数组***的访问越界问题
  ```shell
   // 直接命令行执行，test为用户程序 
   valgrind –tool=memcheck –leak-check=yes test
   ```

## 内存检测 - AddressSanitizer(ASan)工具

### 简要介绍
AddressSanitizer（ASan）是一种内存错误检测工具，属于Clang/LLVM编译器工具套件的一部分。它用于检测和调试C/C++程序中的内存问题，如缓冲区溢出、使用已释放或未初始化的内存等

ASan在编译时通过插入额外的代码来动态地检测程序运行过程中的内存访问错误。它会跟踪每个分配的内存块，并在程序执行期间监视对这些内存块的访问情况。如果发现任何非法操作，比如访问已释放或越界的内存，ASan将立即报告该错误，并提供有关问题位置和堆栈跟踪信息

它包括一个编译器instrumentation模块和一个`提供malloc()/free()替代项的运行时库`。从gcc 4.8开始，AddressSanitizer成为gcc的一部分。当然，要获得更好的体验，最好使用4.9及以上版本

### 使用方法
- 用`-fsanitize=address`选项编译和链接你的程序
- 用`-fno-omit-frame-pointer`编译，以得到更容易理解stack trace
- 可选择`-O1`或者更高的优化级别编译
	`gcc/clang++/g++ -fsanitize=address -o main -g main.c 
- 具体使用方法需要后续查找


## 内存泄漏 - 无法之法 - meminfo
上面的内存泄露是我们知道了具体的泄露的进程，然后再做详细分析。那么如果不知道哪里内存泄露了，有什么办法，可以通过分析`meminfo`文件，来观察泄露的类型

**meminfo文件**‌是Linux系统中用于显示内存使用情况的详细信息文件，它位于`/proc`目录下，提供了关于系统内存使用的全面信息。通过查看和分析meminfo文件的内容，可以了解系统的内存使用状况，包括总内存、空闲内存、缓存、交换分区等信息，这对于排查内存相关的问题非常有帮助

### 使用
`cat /proc/meminfo`

### 使用步骤
1. 首先，使用`cat /proc/meminfo`命令查看meminfo文件的内容，了解系统的整体内存使用情况。
2. 分析MemTotal和MemFree的值，了解系统的总内存和可用空闲内存。
3. 注意MemAvailable的值，它表示应用程序可用的内存，与MemFree的区别在于MemAvailable考虑了Buffers和Cached的大小，这些通常在系统需要时可以被回收。
4. 检查SwapUsage（虽然meminfo文件中没有直接显示SwapUsage，但可以通过SwapTotal和SwapFree计算得出），如果Swap空间被大量使用，可能意味着物理内存不足。
5. 注意Active、Inactive、Dirty和Writeback等值，这些指标可以帮助你了解系统当前的内存使用模式和可能的性能瓶颈。
6. 如果发现某些特定类型的内存使用异常高（如AnonPages、Shmem等），可能需要进一步调查这些类型的内存使用情况，以确定是否存在内存泄漏或其他问题。
7. 使用其他工具如`free`、`vmstat`、`top`或`htop`等命令提供的信息与meminfo文件的内容进行对比，以获得更全面的系统内存使用情况视图。


## coredump
### 参考链接
- [Linux下使用gdb调试core文件](https://cloud.tencent.com/developer/article/1177442)

### 简要介绍
1. coredump文件：当程序运行过程中出现Segmentation fault (core dumped)错误时，程序停止运行，并产生core文件。core文件是程序运行状态的内存映象。使用gdb调试core文件，可以帮助我们快速定位程序出现段错误的位置。当然，可执行程序==编译时应加上-g编译选项==，生成调试信息
2. 当程序访问的内存超出了系统给定的内存空间，就会产生Segmentation fault (core dumped)，因此，段错误产生的情况主要有：
	- 访问不存在的内存地址
	- 访问系统保护的内存地址
	- 数组访问越界等
3. 控制coredump文件的生成
	- 使用`ulimit -c`命令可查看core文件的生成开关
		- 若结果为0，则表示关闭了此功能，不会生成core文件
	- 使用`ulimit -c filesize`命令，可以限制core文件的大小（filesize的单位为KB）。如果生成的信息超过此大小，将会被裁剪，最终生成一个不完整的core文件。在调试此core文 件的时候，gdb会提示错误。比如：ulimit -c 1024
	- 使用ulimit -c unlimited，则表示core文件的大小不受限制。

4. 在终端通过命令`ulimit -c unlimited`只是临时修改，重启后无效 ，要想永久修改有三种方式：
	- 在`/etc/rc.local` 中增加一行 `ulimit -c unlimited`
	- 在`/etc/profile` 中增加一行 `ulimit -c unlimited`
	- 在`/etc/security/limits.conf`最后增加如下两行记录：
	 ```shell
		@root soft core unlimited
		@root hard core unlimited
	 ```
5. 默认生成：core默认的文件名称是`core.pid`，pid指的是产生段错误的程序的进程号。 默认路径是产生段错误的程序的当前目录

### GDB调试步骤
#### 方法一
1. 进入 `gdb [exec file] [core file]`
	- 值得注意的是，core文件中已经含有程序执行时给定的命令行参数，因此这里无需再指定命令行参数
	- [How do I analyze a program's core dump file with GDB when it has command-line parameters?](https://stackoverflow.com/questions/8305866/how-do-i-analyze-a-programs-core-dump-file-with-gdb-when-it-has-command-line-pa)
2. 查找段错误位置：`where` 或者 `bt`
	- 打印出错时的调用栈
3. 使用`up` and `down`上下切换调用栈，查看具体出错信息

#### 方法二
1. 进入`gdb --core=[core file]`
2. 进入gdb后指定==core文件对应的符号表==：`file [exec file]`
3. 查找段错误位置：`where` 或者 `bt`
