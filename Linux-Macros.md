---
title: Linux-Macros
date: 2024-07-22 20:43:34
tags:
categories:
- [Linux相关]
---

# 常用宏

* WRITE_ONCE && READ_ONCE
    - 从__native_word的定义可以看出，所谓的原子读写并不能保证，只是尽可能地避免
    - 对一个10字节的数据进行读写，肯定要分成2次以上的内存访问操作
    - 如果为一个原始类型：char, short, int, long, long long，对其进行读写操作就只产生一次内存访问
    - 有些需要避免竞态的，需要加锁
    ```c
    /*
    * Yes, this permits 64-bit accesses on 32-bit architectures. These will
    * actually be atomic in some cases (namely Armv7 + LPAE), but for others we
    * rely on the access being split into 2x32-bit accesses for a 32-bit quantity
    * (e.g. a virtual address) and a strong prevailing wind.
    */
    #define compiletime_assert_rwonce_type(t)					\
        compiletime_assert(__native_word(t) || sizeof(t) == sizeof(long long),	\
            "Unsupported access size for {READ,WRITE}_ONCE().")

    /* Is this type a native word size -- useful for atomic operations */
    #define __native_word(t) \
        (sizeof(t) == sizeof(char) || sizeof(t) == sizeof(short) || \
        sizeof(t) == sizeof(int) || sizeof(t) == sizeof(long))
    
    /*
    * Use __READ_ONCE() instead of READ_ONCE() if you do not require any
    * atomicity. Note that this may result in tears!
    */
    #ifndef __READ_ONCE
    #define __READ_ONCE(x)	(*(const volatile __unqual_scalar_typeof(x) *)&(x))
    #endif

    #define READ_ONCE(x)							\
    ({									\
        compiletime_assert_rwonce_type(x);				\
        __READ_ONCE(x);							\
    })

    #define __WRITE_ONCE(x, val)						\
    do {									\
        *(volatile typeof(x) *)&(x) = (val);				\
    } while (0)

    #define WRITE_ONCE(x, val)						\
    do {									\
        compiletime_assert_rwonce_type(x);				\
        __WRITE_ONCE(x, val);						\
    } while (0)
    ```

# 标量类型（scalar type）

* 指那些不能被分解成更小的部分的数据类型，如整型（如 int、long、short 等）、浮点型（如 float、double 等）和字符型（如 char）。标量类型与复合类型相对，复合类型是由多个标量类型或其他复合类型组合而成的数据类型，如结构体（struct）、联合体（union）和数组。
    ```c
    /*
    * __unqual_scalar_typeof(x) - Declare an unqualified scalar type, leaving
    *			       non-scalar types unchanged.
    */
    /*
    * Prefer C11 _Generic for better compile-times and simpler code. Note: 'char'
    * is not type-compatible with 'signed char', and we define a separate case.
    */
    #define __scalar_type_to_expr_cases(type)				\
            unsigned type:	(unsigned type)0,			\
            signed type:	(signed type)0

    #define __unqual_scalar_typeof(x) typeof(				\
            _Generic((x),						\
                char:	(char)0,				\
                __scalar_type_to_expr_cases(char),		\
                __scalar_type_to_expr_cases(short),		\
                __scalar_type_to_expr_cases(int),		\
                __scalar_type_to_expr_cases(long),		\
                __scalar_type_to_expr_cases(long long),	\
                default: (x)))
    ```

## sparse 语义检查工具
这里描述的宏都是在使用sparse工具时才会定义/被sparse识别的：
* `# define __rcu		__attribute__((noderef, address_space(__rcu)))`
    - noderef: 意味着这是一个指针，且后续不能对其直接解引用；否则sparse检查器会报错
    - address_space: 这个地址在rcu地址空间中

* `#define __safe __attribute__(safe)`
    - 不用检查参数是否为控，可以直接取成员

* `#define __force __attribute__(force)`
    - 表示所定义的变量是可以做强制类型转换的