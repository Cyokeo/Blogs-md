`timerfd` 注册到 `epoll` 中监听读事件的设计非常巧妙

## 1. **timerfd 的工作原理**

### 1.1 **timerfd 是什么？**
```cpp
#include <sys/timerfd.h>

// 创建一个定时器文件描述符
int timer_fd = timerfd_create(CLOCK_MONOTONIC, TFD_NONBLOCK);

// 设置定时器
struct itimerspec new_value {
    .it_interval = {1, 0},  // 重复间隔：1秒
    .it_value = {1, 0}      // 首次超时：1秒后
};
timerfd_settime(timer_fd, 0, &new_value, nullptr);
```
### 1.2 **为什么是"文件描述符"？**
`timerfd` 的设计哲学：**一切都是文件**
- 定时器 → 特殊的"文件"
- 超时 → "文件"变为可读
- 读取超时次数 → 读取"文件"

## 2. **读事件的含义**

### 2.1 **timerfd 的读行为**
```cpp
// 当定时器超时时：
// 1. timerfd 变为"可读"
// 2. 读取 timerfd 获得超时次数
uint64_t expirations;
ssize_t n = read(timer_fd, &expirations, sizeof(expirations));

if (n == sizeof(expirations)) {
    // expirations = 超时发生的次数
    // 例如：如果处理慢了，可能 expirations > 1
    std::cout << "Timer expired " << expirations << " times" << std::endl;
}
```
### 2.2 **与普通文件描述符的类比**
```cpp
// 普通文件描述符：
// 数据到达 → fd可读 → read()获取数据

// timerfd：
// 时间到达 → fd"可读" → read()获取超时次数

// 这样统一了事件模型：
// epoll_wait(epoll_fd, events, ...)
// 无论是socket数据到达，还是定时器超时
// 都通过"可读事件"通知
```
## 3. **设计优势**

### 3.1 **统一事件模型**
```cpp
// 传统定时器 vs timerfd+epoll

// 传统方式（复杂）：
void event_loop() {
    while (true) {
        // 1. 计算最近超时时间
        int timeout = calculate_timeout();
        
        // 2. 等待事件
        int n = epoll_wait(epoll_fd, events, max_events, timeout);
        
        // 3. 处理事件
        for (int i = 0; i < n; ++i) {
            handle_event(events[i]);
        }
        
        // 4. 检查定时器（需要额外逻辑）
        check_timers();
    }
}

// timerfd方式（简洁）：
void event_loop() {
    while (true) {
        // 1. 直接无限等待（timerfd会唤醒）
        int n = epoll_wait(epoll_fd, events, max_events, -1);   // !!!
        
        // 2. 统一处理（包括定时器）
        for (int i = 0; i < n; ++i) {
            if (events[i].data.fd == timer_fd) {
                handle_timer();
            } else {
                handle_event(events[i]);
            }
        }
    }
}
```
### 3.2 **避免忙等待**
```cpp
// 传统方式的问题：
int timeout = get_next_timer_timeout();  // 比如 1000ms
epoll_wait(..., timeout);  // 最多等待1秒

// 如果在等待期间添加了新定时器（500ms后超时）：!!!
// 需要重新计算超时时间，或者等到1秒后才处理

// timerfd方式：
// timerfd_settime() 会自动更新内核中的超时时间
// epoll_wait() 会立即被重新调度
```

## 4. **实际使用示例**

### 4.1 **精确的定时器实现**
```cpp
class PreciseTimer {
    int timer_fd_;
    Reactor& reactor_;
    
public:
    PreciseTimer(Reactor& reactor) : reactor_(reactor) {
        timer_fd_ = timerfd_create(CLOCK_MONOTONIC, TFD_NONBLOCK);
        
        // 注册读事件到 epoll
        reactor_.register_handler(timer_fd_, EPOLLIN,
            [this](int fd, uint32_t events) {
                handle_timeout(fd, events);
            });
    }
    
    void start(int interval_ms, bool repeat = false) {
        struct itimerspec spec{};
        
        // 首次超时时间
        spec.it_value.tv_sec = interval_ms / 1000;
        spec.it_value.tv_nsec = (interval_ms % 1000) * 1000000;
        
        // 重复间隔（如果不重复则为0）
        if (repeat) {
            spec.it_interval = spec.it_value;
        }
        
        timerfd_settime(timer_fd_, 0, &spec, nullptr);
    }
    
private:
    void handle_timeout(int fd, uint32_t events) {
        uint64_t expirations;
        ssize_t n = read(fd, &expirations, sizeof(expirations));
        
        if (n == sizeof(expirations)) {
            for (uint64_t i = 0; i < expirations; ++i) {
                on_timeout();
            }
        }
    }
    
    virtual void on_timeout() = 0;
};
```
### 4.2 **多个定时器管理**
```cpp
class TimerManager {
    Reactor& reactor_;
    int timer_fd_;
    
    // 使用最小堆管理定时器
    struct Timer {
        int64_t id;
        std::chrono::steady_clock::time_point expires_at;
        std::function<void()> callback;
        bool repeat;
        int interval_ms;
        
        bool operator>(const Timer& other) const {
            return expires_at > other.expires_at;
        }
    };
    
    std::priority_queue<Timer, std::vector<Timer>, 
                       std::greater<Timer>> timer_heap_;
    
public:
    TimerManager(Reactor& reactor) : reactor_(reactor) {
        timer_fd_ = timerfd_create(CLOCK_MONOTONIC, TFD_NONBLOCK);
        
        reactor_.register_handler(timer_fd_, EPOLLIN,
            [this](int fd, uint32_t events) {
                handle_timer_event();
            });
    }
    
    int add_timer(int delay_ms, std::function<void()> callback, bool repeat = false) {
        static int64_t next_id = 0;
        
        Timer timer{
            .id = ++next_id,
            .expires_at = std::chrono::steady_clock::now() + 
                         std::chrono::milliseconds(delay_ms),
            .callback = std::move(callback),
            .repeat = repeat,
            .interval_ms = delay_ms
        };
        
        timer_heap_.push(timer);
        update_timer_fd();  // 更新最近的超时时间
        
        return timer.id;
    }
    
private:
    void update_timer_fd() {
        if (timer_heap_.empty()) {
            // 禁用定时器
            struct itimerspec spec{};
            timerfd_settime(timer_fd_, 0, &spec, nullptr);
            return;
        }
        
        const auto& next_timer = timer_heap_.top();
        auto now = std::chrono::steady_clock::now();
        
        if (next_timer.expires_at <= now) {
            // 立即超时
            struct itimerspec spec{};
            spec.it_value.tv_sec = 0;
            spec.it_value.tv_nsec = 1;  // 1纳秒，尽快触发
            timerfd_settime(timer_fd_, 0, &spec, nullptr);
        } else {
            auto duration = next_timer.expires_at - now;
            auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(duration);
            
            struct itimerspec spec{};
            spec.it_value.tv_sec = ms.count() / 1000;
            spec.it_value.tv_nsec = (ms.count() % 1000) * 1000000;
            
            timerfd_settime(timer_fd_, 0, &spec, nullptr);
        }
    }
    
    void handle_timer_event() {
        uint64_t expirations;
        read(timer_fd_, &expirations, sizeof(expirations));
        
        auto now = std::chrono::steady_clock::now();
        std::vector<Timer> expired_timers;
        
        // 收集所有过期的定时器
        while (!timer_heap_.empty() && 
               timer_heap_.top().expires_at <= now) {
            expired_timers.push_back(timer_heap_.top());
            timer_heap_.pop();
        }
        
        // 执行回调
        for (auto& timer : expired_timers) {
            try {
                timer.callback();
            } catch (const std::exception& e) {
                std::cerr << "Timer callback error: " << e.what() << std::endl;
            }
            
            // 如果是重复定时器，重新加入堆
            if (timer.repeat) {
                timer.expires_at = now + 
                                  std::chrono::milliseconds(timer.interval_ms);
                timer_heap_.push(std::move(timer));
            }
        }
        
        // 更新 timerfd
        update_timer_fd();
    }
};
```
## 6. **与其他定时器方式的对比**

### 6.1 **传统 signal 方式**
```cpp
// 使用 SIGALRM：
signal(SIGALRM, handler);  // 信号处理函数
ualarm(100000, 100000);    // 设置定时器（微秒）

// 问题：
// 1. 信号处理函数限制多
// 2. 信号可能丢失
// 3. 多线程中的信号处理复杂
```
### 6.2 **select/poll 的 timeout**
```cpp
// select 方式：
struct timeval timeout {1, 0};  // 1秒
select(nfds, &read_fds, &write_fds, &except_fds, &timeout);

// 问题：
// 1. timeout 精度低
// 2. 每次循环需要重新计算
// 3. 与事件等待耦合
```
### 6.3 **timerfd 的优势**
```cpp
// timerfd + epoll：
// 1. 统一事件模型
// 2. 高精度
// 3. 线程安全
// 4. 支持多个独立定时器
// 5. 避免信号处理问题
```

