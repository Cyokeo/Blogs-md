---
title: " Linux - 线程资源&模型"
categories: 操作系统
---
## 参考文章
- [主线程退出对子线程的影响](https://originlee.com/2015/04/08/influence-of-main-threads-exiting-to-child-thread/)

## 线程资源
同一进程中的多个线程使用同一份进程地址空间，因此它们的大部分资源是共享的，我们只需要记住一些不被共享的资源即可：
- 线程ID  
- 一组寄存器  
- 栈  
- errno(错误码)  
- 信号屏蔽字(block)  
- 调度优先级

## 线程模型
实际上，posix线程和一般的进程不同，在概念上没有主线程和子线程之分（虽然在实际实现上还是有一些区分），如果仔细观察apue或者unp等书会发现基本看不到「主线程」或者「子线程」等词语，在csapp中甚至都是用「对等线程」一词来描述线程间的关系。

在Linux 2.6以后的posix线程都是由用户态的pthread库来实现的。在使用pthread库以后，在用户视角看来，每一个tast_struct就对应一个线程（tast_struct原本是内核对应一个进程的结构），而一组线程以及他们所共同引用的一组资源就是进程。从Linux 2.6开始，内核有了线程组的概念，***tast_struct结构中增加了一个`tgid`（thread group id）字段***。`getpid`（获取进程号）通过系统调用「返回的也是tast_struct中的`tgid`」，所以`tgid`其实就是进程号。而***tast_struct中的线程号pid字段***则由系统调用`syscall(SYS_gettid)`来获取。

当线程收到一个kill致命信号时，内核会将处理动作施加到整个线程组上。为了应付「发送给进程的信号」和「发送给线程的信号」，tast_struct里面*维护了两套signal_pending*，一套是线程组共用的，一套是线程独有的。通过***kill发送的信号被放在线程组共享的signal_pending中***，可以*任意由一个线程来处理*。而通过***pthread_kill发送的信号被放在线程独有的signal_pending中***，只能由本线程来处理。

关于线程与信号，apue有这么几句：

> 每个线程都有自己的信号屏蔽字，但是信号的处理是进程中所有线程共享的。这意味着尽管单个线程可以阻止某些信号，但当线程修改了与某个信号相关的处理行为以后，所有的线程都必须共享这个处理行为的改变。这样如果一个线程选择忽略某个信号，而其他的线程可以恢复信号的默认处理行为，或者是为信号设置一个新的处理程序，从而可以撤销上述线程的信号选择。
> 
> **如果信号的「默认处理动作」是终止该进程，那么*把信号传递给某个线程仍然会杀掉整个进程*。**

例如一个程序a.out创建了一个子线程，假设主线程的线程号为9601，子线程的线程号为9602（它们的tgid都是9601），因为默认没有设置信号处理程序，所以如果运行命令kill 9602的话，是可以把9601和9602这个两个线程一起杀死的。如果不知道Linux线程背后的故事，可能就会觉得遇到灵异事件了。

因为SIGKILL无法被忽略、捕获且不可改变其默认行为：杀死进程；因此最终无论哪个线程响应了该信号：都会导致进程被杀死。实际上对于这种无法自定义用户处理函数的信号，内核应该直接在内核态直接完成处理了？-> 得再看一下👀

另外*系统调用syscall(SYS_gettid)获取的线程号与pthread_self获取的线程号是不同的*，pthread_self获取的线程号仅仅在线程所依赖的进程内部唯一，在[pthread_self](http://linux.die.net/man/3/pthread_self)的man page中有这样一段话：

> Thread IDs are guaranteed to be unique only within a process. A thread ID may be reused after a terminated thread has been joined, or a detached thread has terminated.

所以在内核中唯一标识线程ID的线程号只能通过系统调用syscall(SYS_gettid)获取。

## 主线程退出对其余线程的影响
### 主线程先退出 
```cpp
#include <pthread.h>
#include <unistd.h>
#include <stdio.h>

void* func(void* arg)
{
	pthread_t main_tid = *static_cast<pthread_t*>(arg);
	pthread_cancel(main_tid);
	while (true)
	{
		printf("child loops\n");
	}
	return NULL;
}
int main(int argc, char* argv[])
{
	pthread_t main_tid = pthread_self();
	pthread_t tid = 0;
	pthread_create(&tid, NULL, func, &main_tid);
	while (true)
	{
		printf("main loops\n");
	}
	sleep(1);
	printf("main exit\n");
	return 0;
}
```

把主线程的线程号传给子线程，在子线程中通过`pthread_cancel`终止主线程使其退出。运行程序，可以发现在打印了一定数量的「main loops」之后程序就挂起了，但却没有退出。

主线程因为被子线程终止了，所有没有看到「main exit」的打印。子线程终止了主线程后进入了死循环while中，所以程序看起来像挂起了。如果我们让子进程while循环中的打印语句生效再运行就可以发现程序会一直打印「child loops」字样。

主线程被子线程终止了，***但他们所「依赖的进程」并没有退出***，所以子线程依然正常运转。

如果主线程使用`pthread_exit`主的退出，也不会导致「依赖的进程」退出，因此其余线程仍然处于运行状态

### 主线程随进程一起退出
之前看到一些人说如果主线程先退出了，子线程也会跟着退出，其实他们混淆了线程退出和进程退出的概念。下面这个例子代表了他们的观点: 
```c
#include <pthread.h>
#include <unistd.h>
#include <stdio.h>

void* func(void* arg)
{
	pthread_t main_tid = *static_cast<pthread_t*>(arg);
	while (true)
	{
		printf("child loops\n");
	}
	return NULL;
}
int main(int argc, char* argv[])
{
	pthread_t main_tid = pthread_self();
	pthread_t tid = 0;
	pthread_create(&tid, NULL, func, &main_tid);
	// while (true)
	{
		printf("main loops\n");
	}
	sleep(1);
	printf("main exit\n");
	return 0;
}
```

运行上面的代码，会发现程序在打印一定数量的「child loops」和一句「main exit」之后退出，并且在退出之前的最后一句打印是「main exit」。

按照他们的逻辑，你看，因为主线程在打印完「main exit」后退出了，然后子线程也跟着退出了，所以随后就没有子线程的打印了。

***但其实这里是混淆了「进程退出」和「线程退出」的概念***了。实际的情况是*主线程中的main函数执行完ruturn后弹栈*，***然后调用glibc库函数「exit」，exit进行相关清理工作后调用_exit系统调用退出该进程***。所以，这种情况*实际上是因为进程运行完毕退出导致所有的线程也都跟着退出了*，并非是因为主线程的退出导致子线程也退出。

### pthread_exit()
```c
/* Decrease the number of threads. We use an atomic operation to
make sure that only the last thread calls `exit'. */
if (atomic_fetch_add_relaxed (&__pthread_total, -1) == 1)
	/* We are the last thread. */
	exit (0);
```