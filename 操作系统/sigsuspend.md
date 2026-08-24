## 1. **基本功能**

### 1.1 **原子操作**
```cpp
// sigsuspend 的关键：原子性！
// 这两个操作是原子的（不可中断）：
// 1. 临时设置新的信号掩码
// 2. 进入等待状态（pause()）

// 非原子方式（可能丢失信号）：
sigprocmask(SIG_SETMASK, &newmask, &oldmask);  // 步骤1
pause();                                        // 步骤2（可能在这里丢失信号！）
sigprocmask(SIG_SETMASK, &oldmask, NULL);      // 步骤3

// 原子方式（使用 sigsuspend）：
sigsuspend(&newmask);  // 步骤1和2原子完成
// 返回时自动恢复原掩码
```

### 1.2 **等待信号的"正确方式"**
```cpp
// 错误方式：直接 pause()
pause();  // 问题：如果在调用 pause() 之前信号就到达了，
          // 那么 pause() 会永远等待！

// 正确方式：使用 sigsuspend()
sigset_t emptymask;
sigemptyset(&emptymask);
sigsuspend(&emptymask);  // 解除所有阻塞，等待信号
```

## 2. **工作原理详解**

### 2.1 **三阶段操作**
```cpp
int sigsuspend(const sigset_t *sigmask) {
    // 内部伪代码：
    1. 保存当前信号掩码
    2. 设置新的信号掩码（sigmask指向的掩码）
    3. 进入睡眠，等待任何未被新掩码阻塞的信号!!!
    4. 信号到达，处理信号处理函数
    5. 信号处理函数返回后，恢复原来的信号掩码
    6. 返回 -1，设置 errno = EINTR
    
    // 注意：总是返回 -1，errno = EINTR
    // 正常返回表示被信号中断，不是错误！
}
```