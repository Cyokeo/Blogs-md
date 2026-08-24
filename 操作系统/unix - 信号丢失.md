信号丢失是 Unix/Linux 信号机制的一个经典问题。让我详细解释原因和机制

## 1. **信号的基本特性**

### 1.1 **信号是异步的**
```cpp
// 信号处理函数在"任意时刻"被调用
void signal_handler(int sig) {
    // 这个函数可能在主程序的任何位置被中断执行
}

int main() {
    signal(SIGINT, signal_handler);  // 注册信号处理
    
    while (true) {
        // 任意时刻可能被 signal_handler 中断
        do_something();
    }
}
```

### 1.2 **信号的"不可靠"本质**
```cpp
// 传统 Unix 信号（不可靠信号，1-31）
// 1. 信号可能丢失
// 2. 信号不排队
// 3. 信号处理期间，同类信号被阻塞

// POSIX 可靠信号（34-64）
// 1. 支持排队
// 2. 但仍有丢失的可能
```
## 2. **信号丢失的具体原因**

### 2.1 **原因一：信号不排队**
```cpp
// 假设场景：快速连续发送多个相同信号
kill(pid, SIGUSR1);  // 第一次
kill(pid, SIGUSR1);  // 第二次（很快）
kill(pid, SIGUSR1);  // 第三次（很快）

// 结果：可能只收到一个 SIGUSR1！
// 内核：哦，进程已经有这个信号待处理了，不再添加
```
### 2.2 **原因二：信号处理期间屏蔽同类信号**
```cpp
// 信号处理函数执行时，自动屏蔽同类信号
void handler(int sig) {
    // 这里执行时间较长...
    sleep(3);  // 3秒
    
    // 在这3秒内，如果收到多个相同信号：
    // 1. 第一个信号：正在处理（就是当前这个）
    // 2. 第二个信号：被阻塞，但标记为"待处理"
    // 3. 第三个及更多：直接被丢弃！
}

int main() {
    struct sigaction sa;
    sa.sa_handler = handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    
    sigaction(SIGINT, &sa, NULL);  // 注册
    
    while (1) pause();
    return 0;
}
```
### 2.3 **原因三：实时信号的优先级**
```cpp
// 如果有多个不同类型的信号：
// 1. 实时信号（34-64）优先于标准信号（1-31）
// 2. 低编号信号优先于高编号信号

// 场景：信号队列满时
// 队列大小：/proc/sys/kernel/rtsig-max （默认1024）
// 如果队列满了，新信号直接被丢弃！
```

## 4. **系统限制导致的丢失**

### 4.1 **查看系统限制**
```bash
# 查看信号队列大小限制
$ cat /proc/sys/kernel/rtsig-max
1024  # 默认值，最多排队1024个实时信号

# 查看 pending 信号限制
$ ulimit -i
63156  # 可以挂起的信号数量
```
### 4.2 **队列满时的行为**
```cpp
// 当信号队列满时，不同信号的命运：

// 1. 标准信号（1-31）：直接丢弃
// 内核：这个进程已经有这个信号在队列里了，不重复添加

// 2. 实时信号（34-64）：
// a) 如果队列未满：加入队列
// b) 如果队列已满：返回 EAGAIN 错误
//    但很多程序不检查这个错误！

int send_signal_with_check(pid_t pid, int sig, int value) {
    union sigval sv;
    sv.sival_int = value;
    
    int ret = sigqueue(pid, sig, sv);
    if (ret < 0) {
        if (errno == EAGAIN) {
            std::cerr << "Signal queue full! Signal lost!" << std::endl;
        }
        return -1;
    }
    return 0;
}
```
## 5. **多线程环境下的信号丢失**

### 5.1 **信号的目标线程**
```cpp
// 多线程中，信号可以发给：
// 1. 整个进程：kill(pid, sig)
// 2. 特定线程：pthread_kill(thread_id, sig)

// 问题：如果没有指定目标线程，信号可能被任意线程处理
// 可能导致处理线程"错过"信号
```
### 5.2 **信号掩码的复杂性**
```cpp
// 每个线程有自己的信号掩码
void* worker_thread(void* arg) {
    sigset_t mask;
    sigemptyset(&mask);
    sigaddset(&mask, SIGUSR1);
    
    // 线程阻塞 SIGUSR1
    pthread_sigmask(SIG_BLOCK, &mask, NULL);
    
    // 这个线程永远不会收到 SIGUSR1
    // 如果主线程把信号发到这里，信号就被"丢失"了
    while (1) sleep(1);
    return NULL;
}

int main() {
    pthread_t thread;
    pthread_create(&thread, NULL, worker_thread, NULL);
    
    sleep(1);
    
    // 发送给特定线程（会被阻塞）
    pthread_kill(thread, SIGUSR1);
    
    // 发送给进程（可能被主线程处理）
    kill(getpid(), SIGUSR1);
    
    pthread_join(thread, NULL);
    return 0;
}
```

## 6. **信号与系统调用的交互**

### 6.1 **慢系统调用被信号中断**
```cpp
// 某些系统调用可能被信号中断
int read_data(int fd) {
    char buffer[1024];
    
    // read() 可能被信号中断
    ssize_t n = read(fd, buffer, sizeof(buffer));
    
    if (n < 0 && errno == EINTR) {
        // 被信号中断！需要重新读取
        // 如果程序不处理这个情况，数据就"丢失"了
        std::cerr << "read interrupted by signal" << std::endl;
        return -1;
    }
    
    return n;
}
```
### 6.2 **自动重启的系统调用**
```cpp
// 使用 SA_RESTART 标志
struct sigaction sa;
sa.sa_handler = handler;
sigemptyset(&sa.sa_mask);
sa.sa_flags = SA_RESTART;  // 关键：自动重启被中断的系统调用

sigaction(SIGINT, &sa, NULL);

// 现在 read(), write(), accept() 等被 SIGINT 中断后会自动重启
// 但：不是所有系统调用都支持重启！
```