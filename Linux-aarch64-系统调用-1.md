---
title: Linux-aarch64-系统调用-1
date: 2024-07-10 19:10:03
tags:
categories:
- [Linux相关]
---

## Linux系统调用定义相关

- 相关代码： 
    - for general arch: sys_\*name -> aliased to __se_sys_\*name -> call __do_sys_\*name to do actual work
    - for arm64 arch: __arm64_sys_* -> call  __se_sys_* -> call __do_sys_\*name to do actual work

- 因此，实际的系统调用函数为__do_sys_*()

- asmlinkage：声明函数传参不使用寄存器，而使用栈进行。后续为Linux源码中的系统调用
    ```c
    // in file : syscalls.h
    #define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)

    #define SYSCALL_DEFINE2(name, ...) SYSCALL_DEFINEx(2, _##name, __VA_ARGS__)

    #define SYSCALL_METADATA(sname, nb, ...)

    /*
    * __MAP - apply a macro to syscall arguments
    * __MAP(n, m, t1, a1, t2, a2, ..., tn, an) will expand to
    *    m(t1, a1), m(t2, a2), ..., m(tn, an)
    * The first argument must be equal to the amount of type/name
    * pairs given.  Note that this list of pairs (i.e. the arguments
    * of __MAP starting at the third one) is in the same format as
    * for SYSCALL_DEFINE<n>/COMPAT_SYSCALL_DEFINE<n>
    */
    #define __MAP0(m,...)
    #define __MAP1(m,t,a,...) m(t,a)
    #define __MAP2(m,t,a,...) m(t,a), __MAP1(m,__VA_ARGS__)
    #define __MAP3(m,t,a,...) m(t,a), __MAP2(m,__VA_ARGS__)
    #define __MAP4(m,t,a,...) m(t,a), __MAP3(m,__VA_ARGS__)
    #define __MAP5(m,t,a,...) m(t,a), __MAP4(m,__VA_ARGS__)
    #define __MAP6(m,t,a,...) m(t,a), __MAP5(m,__VA_ARGS__)
    #define __MAP(n,...) __MAP##n(__VA_ARGS__)
    #define __SC_DECL(t, a)	t a

    #define SYSCALL_DEFINEx(x, sname, ...)				\
        SYSCALL_METADATA(sname, x, __VA_ARGS__)			\
        __SYSCALL_DEFINEx(x, sname, __VA_ARGS__)

    // in file：arch\arm64\include\asm\syscall_wrapper.h
    #define __SYSCALL_DEFINEx(x, name, ...)						\   // arm64定义了自己的__SYSCALL_DEFINEx宏
	asmlinkage long __arm64_sys##name(const struct pt_regs *regs);		\
	ALLOW_ERROR_INJECTION(__arm64_sys##name, ERRNO);			\
	static long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__));		\
	static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__));	\
	asmlinkage long __arm64_sys##name(const struct pt_regs *regs)		\
	{									\
		return __se_sys##name(SC_ARM64_REGS_TO_ARGS(x,__VA_ARGS__));	\
	}									\
	static long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__))		\
	{									\
		long ret = __do_sys##name(__MAP(x,__SC_CAST,__VA_ARGS__));	\
		__MAP(x,__SC_TEST,__VA_ARGS__);					\
		__PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));		\
		return ret;							\
	}									\
	static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))       // 这一行看着很奇怪，需要结合一个具体的系统调用定义才能看明白

    // 这里提取x0-x6
    #define SC_ARM64_REGS_TO_ARGS(x, ...)				\
	__MAP(x,__SC_ARGS					\
	      ,,regs->regs[0],,regs->regs[1],,regs->regs[2]	\
	      ,,regs->regs[3],,regs->regs[4],,regs->regs[5])

    // in file: file.c
    SYSCALL_DEFINE2(dup2, unsigned int, oldfd, unsigned int, newfd)
    {
        if (unlikely(newfd == oldfd)) { /* corner case */
            struct files_struct *files = current->files;
            struct file *f;
            int retval = oldfd;

            rcu_read_lock();
            f = __fget_files_rcu(files, oldfd, 0);
            if (!f)
                retval = -EBADF;
            rcu_read_unlock();
            if (f)
                fput(f);
            return retval;
        }
        return ksys_dup3(oldfd, newfd, 0);
    }

    /*
    * The asmlinkage stub is aliased to a function named __se_sys_*() which
    * sign-extends 32-bit ints to longs whenever needed. The actual work is
    * done within __do_sys_*().
    */
    #ifndef __SYSCALL_DEFINEx       // 不同架构可能定义了自己的__SYSCALL_DEFINEx宏
    #define __SYSCALL_DEFINEx(x, name, ...)					\
        __diag_push();							\
        __diag_ignore(GCC, 8, "-Wattribute-alias",			\
                "Type aliasing is used to sanitize syscall arguments");\
        asmlinkage long sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))	\
            __attribute__((alias(__stringify(__se_sys##name))));	\
        ALLOW_ERROR_INJECTION(sys##name, ERRNO);			\
        static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__));\
        asmlinkage long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__));	\
        asmlinkage long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__))	\
        {								\
            long ret = __do_sys##name(__MAP(x,__SC_CAST,__VA_ARGS__));\
            __MAP(x,__SC_TEST,__VA_ARGS__);				\
            __PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));	\
            return ret;						\
        }								\
        __diag_pop();							\
        static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))
    #endif /* __SYSCALL_DEFINEx */
    ```
- 实际使用中会直接调用dup2()，由其触发svc同步异常，最终调用Linux定义的系统调用“服务函数”。其函数原型在相应的头文件[unistd.h]中有声明，但其定义在哪里？
    - 显然，在Linux中，它直接使用GNU的C库，因此dup2()的定义在GNU的C库源文件中。下载glibc的源码后，可以看到dup2()的定义
    - 可以看到，对于不同的标准/UNIX实现，glibc中也定义了不同的dup2()实现
        ```c
        // in file: sysdeps/posix/dup2.c
            int
        __dup2 (int fd, int fd2)
        {
        int save;

        if (fd2 < 0
        #ifdef OPEN_MAX
            || fd2 >= OPEN_MAX
        #endif
        )
            {
            __set_errno (EBADF);
            return -1;
            }

        /* Check if FD is kosher.  */
        if (fcntl (fd, F_GETFL) < 0)
            return -1;

        if (fd == fd2)
            return fd2;

        /* This is not atomic.  */

        save = errno;
        (void) close (fd2);
        __set_errno (save);

        return fcntl (fd, F_DUPFD, fd2);
        }
        libc_hidden_def (__dup2)
        weak_alias (__dup2, dup2)

        // in file: sysdeps/unix/sysv/linux/dup2.c
        int
        __dup2 (int fd, int fd2)
        {
        #ifdef __NR_dup2
        return INLINE_SYSCALL_CALL (dup2, fd, fd2);
        #else
        /* For the degenerate case, check if the fd is valid (by trying to
            get the file status flags) and return it, or else return EBADF.  */
        if (fd == fd2)
            return __libc_fcntl (fd, F_GETFL, 0) < 0 ? -1 : fd;

        return INLINE_SYSCALL_CALL (dup3, fd, fd2, 0);
        #endif
        }
        libc_hidden_def (__dup2)
        weak_alias (__dup2, dup2)
        ```
    - INLINE_SYSCALL_CALL 宏展开
        ```c
        // in file: sysdeps/unix/sysdep.h

        /* Issue a syscall defined by syscall number plus any other argument
        required.  Any error will be handled using arch defined macros and errno
        will be set accordingly.
        It is similar to INLINE_SYSCALL macro, but without the need to pass the
        expected argument number as second parameter.  */
        #define INLINE_SYSCALL_CALL(...) \
        __INLINE_SYSCALL_DISP (__INLINE_SYSCALL, __VA_ARGS__)

        #define __INTERNAL_SYSCALL_DISP(b,...) \
        __SYSCALL_CONCAT (b,__INTERNAL_SYSCALL_NARGS(__VA_ARGS__))(__VA_ARGS__)

        #define __SYSCALL_CONCAT_X(a,b)     a##b
        #define __SYSCALL_CONCAT(a,b)       __SYSCALL_CONCAT_X (a, b)
        // 以上几个宏的作用就是：根据参数个数，展开为宏__INLINE_SYSCALLn()

        // 接着将参数展开，如下所示
        #define __INLINE_SYSCALL0(name) \
        INLINE_SYSCALL (name, 0)
        #define __INLINE_SYSCALL1(name, a1) \
        INLINE_SYSCALL (name, 1, a1)
        #define __INLINE_SYSCALL2(name, a1, a2) \
        INLINE_SYSCALL (name, 2, a1, a2)
        #define __INLINE_SYSCALL3(name, a1, a2, a3) \
        INLINE_SYSCALL (name, 3, a1, a2, a3)
        #define __INLINE_SYSCALL4(name, a1, a2, a3, a4) \
        INLINE_SYSCALL (name, 4, a1, a2, a3, a4)
        #define __INLINE_SYSCALL5(name, a1, a2, a3, a4, a5) \
        INLINE_SYSCALL (name, 5, a1, a2, a3, a4, a5)
        #define __INLINE_SYSCALL6(name, a1, a2, a3, a4, a5, a6) \
        INLINE_SYSCALL (name, 6, a1, a2, a3, a4, a5, a6)
        #define __INLINE_SYSCALL7(name, a1, a2, a3, a4, a5, a6, a7) \
        INLINE_SYSCALL (name, 7, a1, a2, a3, a4, a5, a6, a7)

        // in file: sysdeps/unix/sysdep.h
        /* Wrappers around system calls should normally inline the system call code.
        But sometimes it is not possible or implemented and we use this code.  */
        #ifndef INLINE_SYSCALL
        #define INLINE_SYSCALL(name, nr, args...) __syscall_##name (args)
        #endif

        // in file: sysdeps/unix/sysv/linux/sysdep.h
        /* Define a macro which expands into the inline wrapper code for a system
        call.  It sets the errno and returns -1 on a failure, or the syscall
        return value otherwise.  */
        #undef INLINE_SYSCALL
        #define INLINE_SYSCALL(name, nr, args...)				\
        ({									\
            long int sc_ret = INTERNAL_SYSCALL (name, nr, args);		\
            __glibc_unlikely (INTERNAL_SYSCALL_ERROR_P (sc_ret))		\
            ? SYSCALL_ERROR_LABEL (INTERNAL_SYSCALL_ERRNO (sc_ret))		\
            : sc_ret;								\
        })

        // 宏 INTERNAL_SYSCALL 是专属于**sysv/linux**实现的，而且不同的硬件架构，其具体定义不同
        // for x86-64, in file: sysdeps/unix/sysv/linux/x86_64/sysdep.h
        #undef INTERNAL_SYSCALL
        #define INTERNAL_SYSCALL(name, nr, args...)				\
            internal_syscall##nr (SYS_ify (name), args)

        #undef internal_syscall0
        #define internal_syscall0(number, dummy...)				\
        ({									\
            unsigned long int resultvar;					\
            asm volatile (							\
            "syscall\n\t"							\
            : "=a" (resultvar)							\
            : "0" (number)							\
            : "memory", REGISTERS_CLOBBERED_BY_SYSCALL);			\
            (long int) resultvar;						\
        })
        // 这里可以看到，x86_64使用 syscall触发软中断，调用号应该也是放到寄存器中："0"(number);参数分别放在r9,r8,r10,rdx,rsi,rdi中
        #undef internal_syscall1
        #define internal_syscall1(number, arg1)					\
        ({									\
            unsigned long int resultvar;					\
            TYPEFY (arg1, __arg1) = ARGIFY (arg1);			 	\
            register TYPEFY (arg1, _a1) asm ("rdi") = __arg1;			\
            asm volatile (							\
            "syscall\n\t"							\
            : "=a" (resultvar)							\
            : "0" (number), "r" (_a1)						\
            : "memory", REGISTERS_CLOBBERED_BY_SYSCALL);			\
            (long int) resultvar;						\
        })

        // for aarch64, in file: sysdeps/unix/sysv/linux/aarch64/sysdep.h
        # undef INTERNAL_SYSCALL
        # define INTERNAL_SYSCALL(name, nr, args...)			\
            INTERNAL_SYSCALL_RAW(SYS_ify(name), nr, args)
        
        // 这里，SYS_ify将系统调用名，扩展为具体的系统调用号
        #undef SYS_ify
        #define SYS_ify(syscall_name)	(__NR_##syscall_name)

        // 这里可以看到使用 svc 0 触发软中断，陷入内核，并在x8寄存器中存储系统调用号
        # undef INTERNAL_SYSCALL_RAW
        # define INTERNAL_SYSCALL_RAW(name, nr, args...)		\
        ({ long _sys_result;						\
            {								\
            LOAD_ARGS_##nr (args)					\
            register long _x8 asm ("x8") = (name);			\
            asm volatile ("svc	0	// syscall " # name     \
                    : "=r" (_x0) : "r"(_x8) ASM_ARGS_##nr : "memory");	\
            _sys_result = _x0;					\
            }								\
            _sys_result; })
        
        // 可以看到，系统调用使用x0-6这7个寄存器进行参数传递
        # define LOAD_ARGS_0()				\
        register long _x0 asm ("x0");
        # define LOAD_ARGS_1(x0)			\
        long _x0tmp = (long) (x0);			\
        LOAD_ARGS_0 ()				\
        _x0 = _x0tmp;
        ```


