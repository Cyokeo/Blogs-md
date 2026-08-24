## 1. **基础架构设计**

### 1.1 **事件循环（Event Loop）**
```cpp
// async_event_loop.hpp
#pragma once

#include <unordered_map>
#include <functional>
#include <mutex>
#include <atomic>
#include <thread>
#include <vector>
#include <memory>
#include <sys/epoll.h>  // Linux
// 或 #include <sys/event.h>  // BSD/macOS
// 或 #include <winsock2.h>   // Windows

class AsyncEventLoop {
public:
    using Callback = std::function<void(int fd, uint32_t events)>;
    
    AsyncEventLoop();
    ~AsyncEventLoop();
    
    // 注册/取消事件监听
    void add_fd(int fd, uint32_t events, Callback callback);
    void remove_fd(int fd);
    
    // 运行事件循环
    void run();
    void stop();
    
    // 立即执行任务（线程安全）
    void post(std::function<void()> task);
    
private:
    int epoll_fd_;
    std::atomic<bool> running_{false};
    std::thread loop_thread_;
    
    struct FdContext {
        Callback callback;
        uint32_t events;
    };
    
    std::unordered_map<int, FdContext> fd_contexts_;
    std::mutex contexts_mutex_;
    
    // 任务队列
    std::vector<std::function<void()>> task_queue_;
    std::mutex task_mutex_;
    int event_fd_;  // 用于唤醒事件循环
    
    void process_events();
    void process_tasks();
};
```

### 1.2 **事件循环实现**
```cpp
// async_event_loop.cpp
#include "async_event_loop.hpp"
#include <unistd.h>
#include <fcntl.h>
#include <sys/eventfd.h>
#include <cstring>
#include <iostream>

AsyncEventLoop::AsyncEventLoop() {
    // 创建 epoll 实例
    epoll_fd_ = epoll_create1(0);
    if (epoll_fd_ < 0) {
        throw std::runtime_error("Failed to create epoll");
    }
    
    // 创建 eventfd 用于唤醒
    event_fd_ = eventfd(0, EFD_NONBLOCK);
    if (event_fd_ < 0) {
        close(epoll_fd_);
        throw std::runtime_error("Failed to create eventfd");
    }
    
    // 监听 eventfd
    add_fd(event_fd_, EPOLLIN, [this](int fd, uint32_t events) {
        uint64_t value;
        read(fd, &value, sizeof(value));
        process_tasks();
    });
}

AsyncEventLoop::~AsyncEventLoop() {
    stop();
    if (event_fd_ >= 0) close(event_fd_);
    if (epoll_fd_ >= 0) close(epoll_fd_);
}

void AsyncEventLoop::add_fd(int fd, uint32_t events, Callback callback) {
    std::lock_guard<std::mutex> lock(contexts_mutex_);
    
    epoll_event ev{};
    ev.events = events;
    ev.data.fd = fd;
    
    if (epoll_ctl(epoll_fd_, EPOLL_CTL_ADD, fd, &ev) < 0) {
        throw std::runtime_error("Failed to add fd to epoll");
    }
    
    fd_contexts_[fd] = {std::move(callback), events};
}

void AsyncEventLoop::remove_fd(int fd) {
    std::lock_guard<std::mutex> lock(contexts_mutex_);
    
    if (fd_contexts_.erase(fd)) {
        epoll_ctl(epoll_fd_, EPOLL_CTL_DEL, fd, nullptr);
    }
}

void AsyncEventLoop::post(std::function<void()> task) {
    {
        std::lock_guard<std::mutex> lock(task_mutex_);
        task_queue_.push_back(std::move(task));
    }
    
    // 唤醒事件循环
    uint64_t value = 1;
    write(event_fd_, &value, sizeof(value));
}

void AsyncEventLoop::run() {
    running_ = true;
    
    while (running_) {
        process_events();
    }
}

void AsyncEventLoop::stop() {
    running_ = false;
    uint64_t value = 1;
    write(event_fd_, &value, sizeof(value));
    
    if (loop_thread_.joinable()) {
        loop_thread_.join();
    }
}

void AsyncEventLoop::process_events() {
    constexpr int MAX_EVENTS = 64;
    epoll_event events[MAX_EVENTS];
    
    int n = epoll_wait(epoll_fd_, events, MAX_EVENTS, -1);
    if (n < 0) {
        if (errno != EINTR) {
            throw std::runtime_error("epoll_wait failed");
        }
        return;
    }
    
    for (int i = 0; i < n; ++i) {
        int fd = events[i].data.fd;
        
        std::lock_guard<std::mutex> lock(contexts_mutex_);
        auto it = fd_contexts_.find(fd);
        if (it != fd_contexts_.end()) {
            it->second.callback(fd, events[i].events);
        }
    }
}

void AsyncEventLoop::process_tasks() {
    std::vector<std::function<void()>> tasks;
    
    {
        std::lock_guard<std::mutex> lock(task_mutex_);
        tasks.swap(task_queue_);
    }
    
    for (auto& task : tasks) {
        task();
    }
}
```
## 2. **异步 Socket 封装**

### 2.1 **Socket 类**
```cpp
// async_socket.hpp
#pragma once

#include "async_event_loop.hpp"
#include <string>
#include <memory>
#include <functional>

class AsyncSocket {
public:
    using ConnectCallback = std::function<void(int error_code)>;
    using ReadCallback = std::function<void(const char* data, size_t size)>;
    using WriteCallback = std::function<void(size_t bytes_written)>;
    
    AsyncSocket(std::shared_ptr<AsyncEventLoop> loop);
    ~AsyncSocket();
    
    // 同步操作
    int create(int domain = AF_INET, int type = SOCK_STREAM, int protocol = 0);
    int bind(const std::string& ip, uint16_t port);
    int listen(int backlog = SOMAXCONN);
    std::shared_ptr<AsyncSocket> accept();
    
    // 异步操作
    void async_connect(const std::string& ip, uint16_t port, 
                       ConnectCallback callback, int timeout_ms = 0);
    void async_read(size_t size, ReadCallback callback);
    void async_write(const char* data, size_t size, WriteCallback callback);
    
    // 工具方法
    int get_fd() const { return fd_; }
    void close();
    bool is_open() const { return fd_ >= 0; }
    
private:
    int fd_{-1};
    std::shared_ptr<AsyncEventLoop> loop_;
    
    // 连接超时处理
    struct ConnectContext {
        ConnectCallback callback;
        std::chrono::steady_clock::time_point start_time;
        int timeout_ms;
        bool completed{false};
    };
    
    std::unique_ptr<ConnectContext> connect_context_;
    
    void handle_connect_event(int fd, uint32_t events);
    void check_connect_timeout();
};
```

### 2.2 **Socket 实现**
```cpp
// async_socket.cpp
#include "async_socket.hpp"
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <fcntl.h>
#include <cstring>
#include <chrono>
#include <iostream>

AsyncSocket::AsyncSocket(std::shared_ptr<AsyncEventLoop> loop)
    : loop_(std::move(loop)) {}

AsyncSocket::~AsyncSocket() {
    close();
}

int AsyncSocket::create(int domain, int type, int protocol) {
    fd_ = socket(domain, type, protocol);
    if (fd_ < 0) return -1;
    
    // 设置为非阻塞
    int flags = fcntl(fd_, F_GETFL, 0);
    fcntl(fd_, F_SETFL, flags | O_NONBLOCK);
    
    return 0;
}

void AsyncSocket::close() {
    if (fd_ >= 0) {
        loop_->remove_fd(fd_);
        ::close(fd_);
        fd_ = -1;
    }
}

int AsyncSocket::bind(const std::string& ip, uint16_t port) {
    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);
    
    if (inet_pton(AF_INET, ip.c_str(), &addr.sin_addr) <= 0) {
        return -1;
    }
    
    return ::bind(fd_, reinterpret_cast<sockaddr*>(&addr), sizeof(addr));
}

int AsyncSocket::listen(int backlog) {
    return ::listen(fd_, backlog);
}

std::shared_ptr<AsyncSocket> AsyncSocket::accept() {
    sockaddr_in client_addr{};
    socklen_t addr_len = sizeof(client_addr);
    
    int client_fd = ::accept(fd_, 
                           reinterpret_cast<sockaddr*>(&client_addr),
                           &addr_len);
    
    if (client_fd < 0) {
        return nullptr;
    }
    
    auto client_socket = std::make_shared<AsyncSocket>(loop_);
    client_socket->fd_ = client_fd;
    
    // 设置为非阻塞
    int flags = fcntl(client_fd, F_GETFL, 0);
    fcntl(client_fd, F_SETFL, flags | O_NONBLOCK);
    
    return client_socket;
}

void AsyncSocket::async_connect(const std::string& ip, uint16_t port,
                                ConnectCallback callback, int timeout_ms) {
    if (fd_ < 0) {
        create();
    }
    
    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);
    
    if (inet_pton(AF_INET, ip.c_str(), &addr.sin_addr) <= 0) {
        loop_->post([callback]() { callback(EINVAL); });
        return;
    }
    
    // 发起非阻塞连接
    int result = ::connect(fd_, 
                          reinterpret_cast<sockaddr*>(&addr),
                          sizeof(addr));
    
    if (result == 0) {
        // 立即连接成功（本地连接）
        loop_->post([callback]() { callback(0); });
        return;
    }
    
    if (errno != EINPROGRESS) {
        // 立即失败
        loop_->post([callback, err = errno]() { callback(err); });
        return;
    }
    
    // 连接进行中，注册事件监听
    connect_context_ = std::make_unique<ConnectContext>();
    connect_context_->callback = std::move(callback);
    connect_context_->timeout_ms = timeout_ms;
    connect_context_->start_time = std::chrono::steady_clock::now();
    
    // 监听可写事件（连接完成时会变为可写）
    loop_->add_fd(fd_, EPOLLOUT, 
        [this](int fd, uint32_t events) {
            handle_connect_event(fd, events);
        });
    
    // 设置超时检查
    if (timeout_ms > 0) {
        loop_->post([this]() { check_connect_timeout(); });
    }
}

void AsyncSocket::handle_connect_event(int fd, uint32_t events) {
    if (!connect_context_ || connect_context_->completed) {
        return;
    }
    
    connect_context_->completed = true;
    loop_->remove_fd(fd);
    
    // 检查连接结果
    int error = 0;
    socklen_t len = sizeof(error);
    getsockopt(fd, SOL_SOCKET, SO_ERROR, &error, &len);
    
    // 调用回调
    connect_context_->callback(error);
    connect_context_.reset();
}

void AsyncSocket::check_connect_timeout() {
    if (!connect_context_ || connect_context_->completed) {
        return;
    }
    
    auto now = std::chrono::steady_clock::now();
    auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(
        now - connect_context_->start_time).count();
    
    if (elapsed >= connect_context_->timeout_ms) {
        // 超时
        connect_context_->completed = true;
        loop_->remove_fd(fd_);
        connect_context_->callback(ETIMEDOUT);
        connect_context_.reset();
    } else {
        // 继续检查
        loop_->post([this]() { check_connect_timeout(); });
    }
}

void AsyncSocket::async_read(size_t size, ReadCallback callback) {
    auto buffer = std::make_shared<std::vector<char>>(size);
    
    loop_->add_fd(fd_, EPOLLIN, 
        [this, buffer, callback, size](int fd, uint32_t events) mutable {
            ssize_t n = read(fd, buffer->data(), size);
            
            if (n > 0) {
                callback(buffer->data(), n);
                loop_->remove_fd(fd);  // 一次性读取
            } else if (n == 0) {
                // 连接关闭
                loop_->remove_fd(fd);
            } else if (errno != EAGAIN && errno != EWOULDBLOCK) {
                // 错误
                loop_->remove_fd(fd);
            }
            // 否则继续等待
        });
}

void AsyncSocket::async_write(const char* data, size_t size, 
                              WriteCallback callback) {
    auto shared_data = std::make_shared<std::vector<char>>(data, data + size);
    
    loop_->add_fd(fd_, EPOLLOUT,
        [this, shared_data, callback, size](int fd, uint32_t events) mutable {
            ssize_t n = write(fd, shared_data->data(), shared_data->size());
            
            if (n >= 0) {
                callback(n);
                loop_->remove_fd(fd);
            } else if (errno != EAGAIN && errno != EWOULDBLOCK) {
                // 错误
                loop_->remove_fd(fd);
                callback(0);
            }
            // 否则继续等待
        });
}
```

## 3. **异步 Connect 的高级功能**

### 3.1 **支持多个端点（Endpoint Iterator）**
```cpp
// async_connect_ex.hpp
#pragma once

#include "async_socket.hpp"
#include <vector>
#include <functional>
#include <memory>
#include <chrono>

namespace my_async {

struct Endpoint {
    std::string ip;
    uint16_t port;
};

template<typename Iterator>
void async_connect(Iterator begin, Iterator end,
                   std::shared_ptr<AsyncSocket> socket,
                   std::function<void(int error_code, Iterator current)> callback,
                   int timeout_per_endpoint_ms = 5000) {
    
    struct ConnectState {
        Iterator current;
        Iterator end;
        std::function<void(int, Iterator)> callback;
        int timeout_ms;
        std::chrono::steady_clock::time_point start_time;
    };
    
    auto state = std::make_shared<ConnectState>();
    state->current = begin;
    state->end = end;
    state->callback = std::move(callback);
    state->timeout_ms = timeout_per_endpoint_ms;
    state->start_time = std::chrono::steady_clock::now();
    
    // 内部递归连接函数
    std::function<void()> try_next;
    
    try_next = [state, socket, &try_next]() {
        if (state->current == state->end) {
            // 所有端点都尝试失败
            state->callback(ENETUNREACH, state->end);
            return;
        }
        
        const auto& endpoint = *state->current;
        
        socket->async_connect(endpoint.ip, endpoint.port,
            [state, socket, &try_next, current_it = state->current]
            (int error_code) {
                if (error_code == 0) {
                    // 连接成功
                    state->callback(0, current_it);
                } else {
                    // 尝试下一个端点
                    ++state->current;
                    
                    // 检查总超时
                    auto now = std::chrono::steady_clock::now();
                    auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(
                        now - state->start_time).count();
                    
                    if (elapsed < state->timeout_ms) {
                        try_next();
                    } else {
                        state->callback(ETIMEDOUT, state->current);
                    }
                }
            },
            state->timeout_ms);
    };
    
    try_next();
}

} // namespace my_async
```

### 3.2 **连接条件支持**
```cpp
// connect_condition.hpp
#pragma once

#include <functional>

namespace my_async {

// 连接条件接口
template<typename Iterator>
class ConnectCondition {
public:
    virtual ~ConnectCondition() = default;
    
    // 返回 true 表示尝试连接当前端点
    virtual bool should_try(int last_error, 
                           typename Iterator::value_type endpoint) = 0;
    
    // 返回下一个应该尝试的迭代器
    virtual Iterator next_iterator(int last_error,
                                  Iterator current,
                                  Iterator end) = 0;
};

// 默认连接条件：尝试所有端点
template<typename Iterator>
class DefaultConnectCondition : public ConnectCondition<Iterator> {
public:
    bool should_try(int last_error,
                   typename Iterator::value_type endpoint) override {
        return true;  // 总是尝试
    }
    
    Iterator next_iterator(int last_error,
                          Iterator current,
                          Iterator end) override {
        if (current != end) {
            return ++current;
        }
        return end;
    }
};

// 带连接条件的异步连接
template<typename Iterator, typename ConnectCondition>
void async_connect_with_condition(
    Iterator begin, Iterator end,
    std::shared_ptr<AsyncSocket> socket,
    ConnectCondition&& condition,
    std::function<void(int error_code, Iterator current)> callback,
    int timeout_per_endpoint_ms = 5000) {
    
    struct ConnectState {
        Iterator current;
        Iterator end;
        ConnectCondition condition;
        std::function<void(int, Iterator)> callback;
        int timeout_ms;
        std::chrono::steady_clock::time_point start_time;
    };
    
    auto state = std::make_shared<ConnectState>();
    state->current = begin;
    state->end = end;
    state->condition = std::forward<ConnectCondition>(condition);
    state->callback = std::move(callback);
    state->timeout_ms = timeout_per_endpoint_ms;
    state->start_time = std::chrono::steady_clock::now();
    
    std::function<void()> try_next;
    
    try_next = [state, socket, &try_next]() {
        // 跳过不应该尝试的端点
        while (state->current != state->end &&
               !state->condition.should_try(0, *state->current)) {
            state->current = state->condition.next_iterator(
                0, state->current, state->end);
        }
        
        if (state->current == state->end) {
            state->callback(ENETUNREACH, state->end);
            return;
        }
        
        const auto& endpoint = *state->current;
        
        socket->async_connect(endpoint.ip, endpoint.port,
            [state, socket, &try_next, current_it = state->current]
            (int error_code) {
                if (error_code == 0) {
                    // 连接成功
                    state->callback(0, current_it);
                } else {
                    // 根据条件决定下一个端点
                    state->current = state->condition.next_iterator(
                        error_code, state->current, state->end);
                    
                    // 检查总超时
                    auto now = std::chrono::steady_clock::now();
                    auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(
                        now - state->start_time).count();
                    
                    if (elapsed < state->timeout_ms) {
                        try_next();
                    } else {
                        state->callback(ETIMEDOUT, state->current);
                    }
                }
            },
            state->timeout_ms);
    };
    
    try_next();
}

} // namespace my_async
```

## 4. **使用示例**

### 4.1 **基本使用**
```cpp
// example.cpp
#include "async_socket.hpp"
#include "async_connect_ex.hpp"
#include <iostream>
#include <vector>

int main() {
    // 创建事件循环
    auto loop = std::make_shared<AsyncEventLoop>();
    
    // 在工作线程中运行事件循环
    std::thread loop_thread([loop]() {
        loop->run();
    });
    
    // 创建 socket
    auto socket = std::make_shared<AsyncSocket>(loop);
    
    // 准备端点列表
    std::vector<my_async::Endpoint> endpoints = {
        {"127.0.0.1", 8080},
        {"127.0.0.1", 8081},
        {"127.0.0.1", 8082}
    };
    
    // 异步连接
    my_async::async_connect(
        endpoints.begin(), endpoints.end(),
        socket,
        [socket](int error_code, auto current) {
            if (error_code == 0) {
                std::cout << "Connected to " 
                          << current->ip << ":" << current->port 
                          << std::endl;
                
                // 发送数据
                std::string message = "Hello Server!";
                socket->async_write(message.data(), message.size(),
                    [](size_t bytes_written) {
                        std::cout << "Sent " << bytes_written 
                                  << " bytes" << std::endl;
                    });
            } else {
                std::cerr << "Connect failed with error: " 
                          << strerror(error_code) << std::endl;
            }
        },
        3000);  // 每个端点超时3秒
    
    // 等待连接完成
    std::this_thread::sleep_for(std::chrono::seconds(10));
    
    // 清理
    loop->stop();
    loop_thread.join();
    
    return 0;
}
```

### 4.2 **自定义连接条件**
```cpp
// 自定义条件：只尝试特定端口的端点
class PortFilterCondition {
    uint16_t target_port_;
    
public:
    PortFilterCondition(uint16_t port) : target_port_(port) {}
    
    template<typename Iterator>
    bool should_try(int last_error, typename Iterator::value_type endpoint) {
        return endpoint.port == target_port_;
    }
    
    template<typename Iterator>
    Iterator next_iterator(int last_error, Iterator current, Iterator end) {
        auto next = current;
        while (++next != end) {
            if (next->port == target_port_) {
                return next;
            }
        }
        return end;
    }
};

// 使用自定义条件
PortFilterCondition condition(443);  // 只尝试 443 端口
my_async::async_connect_with_condition(
    endpoints.begin(), endpoints.end(),
    socket, condition,
    [](int error_code, auto current) {
        // 处理结果
    });
```

## 5. **跨平台考虑**

### 5.1 **平台抽象层**
```cpp
// platform_selector.hpp
#pragma once

#ifdef _WIN32
    #include <winsock2.h>
    #include <ws2tcpip.h>
    #pragma comment(lib, "ws2_32.lib")
    
    #define SOCKET_ERROR_NONBLOCKING WSAEWOULDBLOCK
    #define close_socket closesocket
    using socket_t = SOCKET;
    constexpr socket_t INVALID_SOCKET_VALUE = INVALID_SOCKET;
    
#else
    #include <sys/socket.h>
    #include <netinet/in.h>
    #include <arpa/inet.h>
    #include <unistd.h>
    #include <fcntl.h>
    
    #define SOCKET_ERROR_NONBLOCKING EAGAIN
    #define close_socket close
    using socket_t = int;
    constexpr socket_t INVALID_SOCKET_VALUE = -1;
    
#endif

// 统一的 socket 操作封装
class PlatformSocket {
public:
    static socket_t create(int domain, int type, int protocol);
    static int set_nonblocking(socket_t fd);
    static int connect(socket_t fd, const sockaddr* addr, socklen_t len);
    static int get_error(socket_t fd);
    // ... 其他封装
};
```

## 6. **性能优化**

### 6.1 **连接池**
```cpp
// connection_pool.hpp
class ConnectionPool {
    struct PooledConnection {
        std::shared_ptr<AsyncSocket> socket;
        std::chrono::steady_clock::time_point last_used;
        bool in_use{false};
    };
    
    std::vector<PooledConnection> pool_;
    std::shared_ptr<AsyncEventLoop> loop_;
    size_t max_pool_size_;
    
public:
    void async_get_connection(const std::string& ip, uint16_t port,
                             std::function<void(std::shared_ptr<AsyncSocket>)> callback) {
        // 1. 尝试从池中获取可用连接
        // 2. 如果没有，创建新连接
        // 3. 异步连接
        // 4. 返回给用户
    }
    
    void release_connection(std::shared_ptr<AsyncSocket> socket) {
        // 放回连接池或关闭
    }
};
```

## 7. **总结**

实现异步 `connect` 的关键点：

1. **事件循环**：核心是 `epoll`/`kqueue`/`IOCP` 的封装
    
2. **非阻塞 socket**：所有操作都设为非阻塞
    
3. **回调机制**：使用 `std::function` 处理异步完成
    
4. **超时处理**：通过事件循环定时检查
    
5. **错误处理**：正确处理各种错误码
    
6. **资源管理**：使用智能指针管理生命周期
    

这个实现虽然简化，但包含了异步网络编程的核心概念。实际生产环境中还需要：

- 线程安全优化
    
- 更完善的错误处理
    
- 内存池优化
    
- 日志和监控
    
- 更高效的缓冲区管理

通过这个从零开始的实现，你可以深入理解 Asio 等网络库的内部工作原理。