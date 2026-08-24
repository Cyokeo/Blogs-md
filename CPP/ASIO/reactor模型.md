**Reactor 模型**是高性能网络编程中的核心设计模式，特别适合处理大量并发连接。让我详细解释：
## 1. **基本概念**

### 1.1 **什么是 Reactor？**
Reactor（反应器）模式是一种**事件驱动的设计模式**，用于处理多个并发请求。它通过**事件分发器**监听多个事件源，当事件发生时，将其分发给相应的事件处理器。

### 1.2 **核心组件**
```cpp
// Reactor 模式的四个核心组件：
1. 事件（Event）：I/O 操作、定时器、信号等
2. 事件源（Event Source）：产生事件的实体（如socket）
3. 事件分发器（Demultiplexer）：等待事件发生（如epoll_wait）
4. 事件处理器（EventHandler）：处理事件的回调函数
```

## 2. **工作原理**

### 2.1 **同步 vs Reactor**
```cpp
// 传统同步模型（一个连接一个线程）
void handle_client(int client_fd) {
    while (true) {
        char buffer[1024];
        ssize_t n = read(client_fd, buffer, sizeof(buffer));  // 阻塞
        if (n > 0) {
            process(buffer, n);
            write(client_fd, response, response_len);  // 阻塞
        }
    }
}

// Reactor 模型（所有连接在一个线程中）
void reactor_loop() {
    while (true) {
        // 1. 等待事件（非阻塞）
        events = demultiplexer.wait();
        
        // 2. 分发事件
        for (event in events) {
            handler = get_handler(event.fd);
            handler.handle_event(event.type);
        }
    }
}
```

### 2.2 **工作流程图**
```txt
┌─────────────────────────────────────────────┐
│                Reactor 线程                  │
│                                             │
│  ┌─────────────┐    ┌───────────────────┐   │
│  │ 等待事件     │    │  事件分发         │   │
│  │ demultiplex │───▶│   dispatch        │   │
│  │   (epoll)   │    │                   │   │
│  └─────────────┘    └──────────┬────────┘   │
│                                 │            │
│                                 ▼            │
│  ┌───────────────────────────────────────┐   │
│  │         事件处理器 EventHandler        │   │
│  │  ┌──────┐  ┌──────┐  ┌──────┐       │   │
│  │  │ Acceptor │  │ Handler │  │ Handler │  ... │
│  │  └──────┘  └──────┘  └──────┘       │   │
│  └───────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

## 3. **具体实现示例**

### 3.1 **Reactor 核心类**
```cpp
// reactor.hpp
#pragma once

#include <unordered_map>
#include <functional>
#include <memory>
#include <vector>
#include <sys/epoll.h>

class Reactor {
public:
    using EventType = uint32_t;
    using Handler = std::function<void(int fd, EventType events)>;
    
    enum {
        READ_EVENT = EPOLLIN,
        WRITE_EVENT = EPOLLOUT,
        ERROR_EVENT = EPOLLERR,
        CLOSE_EVENT = EPOLLRDHUP
    };
    
    Reactor();
    ~Reactor();
    
    // 注册/取消事件
    void register_handler(int fd, EventType events, Handler handler);
    void unregister_handler(int fd);
    void modify_handler(int fd, EventType events);
    
    // 运行事件循环
    void run();
    void stop();
    
    // 定时器支持
    using TimerCallback = std::function<void()>;
    int add_timer(int milliseconds, TimerCallback callback, bool repeat = false);
    void cancel_timer(int timer_id);
    
private:
    int epoll_fd_{-1};
    std::atomic<bool> running_{false};
    
    struct EventContext {
        Handler handler;
        EventType events;
    };
    
    std::unordered_map<int, EventContext> handlers_;
    
    // 定时器相关
    struct TimerInfo {
        int id;
        int interval_ms;
        std::chrono::steady_clock::time_point expires_at;
        TimerCallback callback;
        bool repeat;
    };
    
    std::unordered_map<int, TimerInfo> timers_;
    int next_timer_id_{0};
    int timer_fd_{-1};  // 用于定时器的事件fd
    
    void process_events(int timeout_ms = -1);
    void process_timers();
    void update_timer_fd();
};
```
### 3.2 **Acceptor 组件**
```cpp
// acceptor.hpp
#pragma once

#include "reactor.hpp"
#include <functional>

class Acceptor {
public:
    using ConnectionCallback = std::function<void(int client_fd, 
                                                 sockaddr_in client_addr)>;
    
    Acceptor(Reactor& reactor, const std::string& ip, uint16_t port);
    ~Acceptor();
    
    void start();
    void stop();
    void set_connection_callback(ConnectionCallback callback);
    
private:
    Reactor& reactor_;
    int listen_fd_{-1};
    ConnectionCallback connection_callback_;
    
    void handle_accept_event(int fd, Reactor::EventType events);
    void setup_listen_socket(const std::string& ip, uint16_t port);
};
```
### 3.3 **具体实现**
```cpp
// reactor.cpp
#include "reactor.hpp"
#include <unistd.h>
#include <sys/timerfd.h>
#include <cstring>
#include <chrono>
#include <iostream>

Reactor::Reactor() {
    epoll_fd_ = epoll_create1(0);
    if (epoll_fd_ < 0) {
        throw std::runtime_error("Failed to create epoll");
    }
    
    // 创建定时器fd
    timer_fd_ = timerfd_create(CLOCK_MONOTONIC, TFD_NONBLOCK);
    if (timer_fd_ >= 0) {
        register_handler(timer_fd_, READ_EVENT,
            [this](int fd, EventType events) {
                uint64_t expirations;
                read(fd, &expirations, sizeof(expirations));
                process_timers();
            });
    }
}

Reactor::~Reactor() {
    stop();
    if (timer_fd_ >= 0) close(timer_fd_);
    if (epoll_fd_ >= 0) close(epoll_fd_);
}

void Reactor::register_handler(int fd, EventType events, Handler handler) {
    epoll_event ev{};
    ev.events = events;
    ev.data.fd = fd;
    
    if (epoll_ctl(epoll_fd_, EPOLL_CTL_ADD, fd, &ev) < 0) {
        throw std::runtime_error("Failed to register handler");
    }
    
    handlers_[fd] = {std::move(handler), events};
}

void Reactor::unregister_handler(int fd) {
    if (handlers_.erase(fd)) {
        epoll_ctl(epoll_fd_, EPOLL_CTL_DEL, fd, nullptr);
    }
}

void Reactor::run() {
    running_ = true;
    
    while (running_) {
        // 计算最近的定时器超时时间
        int timeout_ms = -1;  // 默认无限等待
        if (!timers_.empty()) {
            auto now = std::chrono::steady_clock::now();
            auto next_expire = timers_.begin()->second.expires_at;
            auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(
                next_expire - now);
            timeout_ms = std::max(0, static_cast<int>(duration.count()));
        }
        
        process_events(timeout_ms);
    }
}

void Reactor::process_events(int timeout_ms) {
    constexpr int MAX_EVENTS = 64;
    epoll_event events[MAX_EVENTS];
    
    int n = epoll_wait(epoll_fd_, events, MAX_EVENTS, timeout_ms);
    if (n < 0) {
        if (errno != EINTR) {
            throw std::runtime_error("epoll_wait failed");
        }
        return;
    }
    
    for (int i = 0; i < n; ++i) {
        int fd = events[i].data.fd;
        
        auto it = handlers_.find(fd);
        if (it != handlers_.end()) {
            try {
                it->second.handler(fd, events[i].events);
            } catch (const std::exception& e) {
                std::cerr << "Handler error: " << e.what() << std::endl;
            }
        }
    }
}

int Reactor::add_timer(int milliseconds, TimerCallback callback, bool repeat) {
    TimerInfo timer;
    timer.id = ++next_timer_id_;
    timer.interval_ms = milliseconds;
    timer.expires_at = std::chrono::steady_clock::now() + 
                      std::chrono::milliseconds(milliseconds);
    timer.callback = std::move(callback);
    timer.repeat = repeat;
    
    timers_[timer.id] = std::move(timer);
    update_timer_fd();
    
    return timer.id;
}

void Reactor::process_timers() {
    auto now = std::chrono::steady_clock::now();
    std::vector<int> expired_timers;
    
    // 找出所有过期的定时器
    for (const auto& [id, timer] : timers_) {
        if (timer.expires_at <= now) {
            expired_timers.push_back(id);
        }
    }
    
    // 执行回调并更新重复的定时器
    for (int id : expired_timers) {
        auto it = timers_.find(id);
        if (it != timers_.end()) {
            TimerInfo& timer = it->second;
            
            try {
                timer.callback();
            } catch (const std::exception& e) {
                std::cerr << "Timer callback error: " << e.what() << std::endl;
            }
            
            if (timer.repeat) {
                // 重置定时器
                timer.expires_at = now + 
                                  std::chrono::milliseconds(timer.interval_ms);
            } else {
                // 一次性定时器，删除
                timers_.erase(it);
            }
        }
    }
    
    update_timer_fd();
}
```
### 3.4 **连接处理器**
```cpp
// connection_handler.hpp
class ConnectionHandler {
    Reactor& reactor_;
    int client_fd_;
    std::string buffer_;
    
public:
    ConnectionHandler(Reactor& reactor, int client_fd)
        : reactor_(reactor), client_fd_(client_fd) {
        
        // 注册读事件
        reactor_.register_handler(client_fd_, Reactor::READ_EVENT,
            [this](int fd, Reactor::EventType events) {
                handle_read(fd, events);
            });
    }
    
    ~ConnectionHandler() {
        reactor_.unregister_handler(client_fd_);
        close(client_fd_);
    }
    
private:
    void handle_read(int fd, Reactor::EventType events) {
        char temp_buf[4096];
        ssize_t n = read(fd, temp_buf, sizeof(temp_buf));
        
        if (n > 0) {
            buffer_.append(temp_buf, n);
            process_buffer();
        } else if (n == 0) {
            // 连接关闭
            delete this;  // 自销毁
        } else {
            if (errno != EAGAIN && errno != EWOULDBLOCK) {
                delete this;  // 错误，自销毁
            }
        }
    }
    
    void process_buffer() {
        // 处理接收到的数据
        // 例如：解析HTTP请求、处理协议等
    }
};
```
## 4. **Reactor 的变体**

### 4.1 **单 Reactor 单线程**
```txt
// 最简单的模型，适合轻量级应用
┌─────────────────────┐
│   Reactor 线程       │
│ 监听 + 分发 + 处理    │
└─────────────────────┘
// 缺点：处理耗时操作会阻塞整个事件循环
```

### 4.2 **单 Reactor 多线程**
```txt
// 主流模型：Netty、Redis
┌─────────────────────┐
│   Reactor 线程       │
│   监听 + 分发        │
└──────────┬──────────┘
           │
    ┌──────▼──────┐
    │  线程池      │
    │  处理耗时操作  │
    └─────────────┘
// I/O在Reactor线程，业务处理在线程池
```

### 4.3 **多 Reactor 多线程**
```txt
// 最高性能模型：Nginx、Memcached
┌─────────────────────┐
│  Main Reactor       │
│  仅处理accept        │
└──────────┬──────────┘
           │
    ┌──────▼──────┐ ┌──────┐ ┌──────┐
    │ Sub Reactor  │ │ Sub  │ │ Sub  │
    │   线程1      │ │线程2 │ │线程3 │ ...
    └─────────────┘ └──────┘ └──────┘
// 每个Sub Reactor有独立的事件循环
// 连接被均匀分配到各个Sub Reactor
```

