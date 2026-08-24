## 参考文档
- https://rust-lang.github.io/async-book/part-guide/async-await.html

## Tokio 的技术架构
```text
┌─────────────────────────────────────────┐
│          您的应用程序                    │
│  (使用 async/await)                     │
├─────────────────────────────────────────┤
│           Tokio 运行时                  │
│  ├─────────────────────────────────────┤
│  │  执行器 (Executor)                  │
│  │  调度器 (Scheduler)                 │
│  │  异步 I/O 抽象                      │
│  │  定时器 (Timer)                     │
│  └─────────────────────────────────────┤
│           MIO                          │
│  ├─────────────────────────────────────┤
│  │  事件通知系统                       │
│  │  跨平台 I/O 多路复用                │
│  │  (epoll/kqueue/IOCP 封装)           │
│  └─────────────────────────────────────┤
│          操作系统 API                   │
│  (epoll/kqueue/IOCP/...)               │
└─────────────────────────────────────────┘
```

There are a lot of subtleties to be considered, though. I strongly recommend reading the [tokio tutorial](https://tokio.rs/tokio/tutorial), it explains many of those concepts. After that, I recommend reading the [async book](https://rust-lang.github.io/async-book).

For example, some important subtleties:

- Do **not** block a task using std's synchronization primitives, under any circumstance. Blocking a task will block **everything**, because async scheduling is **non-preemptive**, meaning, the scheduler cannot unschedule a task. It can only switch tasks at `.await` points, so whenever you are waiting for something, make sure it's inside of an `.await` point. (exception: short-lived `std::sync::Mutex`, see [here](https://tokio.rs/tokio/tutorial/shared-state#on-using-stdsyncmutex))
- Don't use async tasks for heavy computation. While it technically isn't blocking, a heavy computation introduces a long time between two `.await` points. Instead, off-load it to a real thread using [`spawn_blocking`](https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html), which introduces an `.await` point to the worker and performs the actual computation on a different threadpool.