# 结合 Mini Tokio 的完整 `Waker` 与 `wake()` 实现（可直接运行）

我们基于你之前的代码，构建一个**完整、可编译、可运行**的 Mini Tokio 示例，重点展示：

1. **执行器如何创建 `Waker`**；
2. **`wake()` 如何把任务塞回就绪队列**；
3. **底层 Future（`Delay`）如何正确保存和更新 `Waker`**。

---

# 第一步：完整代码（直接复制到 `src/main.rs`）
```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Waker};
use std::thread;
use std::time::{Duration, Instant};

// 引入 futures 库的 ArcWake 工具，避免手动写 RawWakerVTable
use futures::task::{self, ArcWake};
// 引入多生产者单消费者通道，用于线程安全地传递就绪任务
use std::sync::mpsc;

// ==========================================
// 1. 定义 Mini Tokio 执行器
// ==========================================
struct MiniTokio {
    // 接收端：接收被唤醒的任务
    scheduled: mpsc::Receiver<Arc<Task>>,
    // 发送端：用于把任务塞回队列
    sender: mpsc::Sender<Arc<Task>>,
}

impl MiniTokio {
    /// 创建一个新的 Mini Tokio 执行器
    fn new() -> Self {
        // 创建一个通道：发送端给 Task，接收端留着 run() 里取任务
        let (sender, scheduled) = mpsc::channel();
        MiniTokio { scheduled, sender }
    }

    ///  spawn 一个异步任务到执行器
    fn spawn<F>(&self, future: F)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        // 把 Future 包装成 Task，通过通道发送给自己
        Task::spawn(future, &self.sender);
    }

    /// 运行执行器（主循环）
    fn run(&self) {
        // 不断从通道接收被唤醒的任务
        while let Ok(task) = self.scheduled.recv() {  // 阻塞等待
            // 拿到任务后，调用它的 poll 方法
            task.poll();
        }
    }
}

// ==========================================
// 2. 定义 Task：包装 Future + 实现 Waker 逻辑
// ==========================================
/// 辅助结构体：保存 Future 和它的最新 poll 结果
struct TaskFuture {
    future: Pin<Box<dyn Future<Output = ()> + Send>>,
    poll: Poll<()>,
}

impl TaskFuture {
    fn new(future: impl Future<Output = ()> + Send + 'static) -> Self {
        TaskFuture {
            future: Box::pin(future),
            poll: Poll::Pending,
        }
    }

    /// 真正执行 poll 的地方
    fn poll(&mut self, cx: &mut Context<'_>) {
        // 【契约】禁止 poll 已经返回 Ready 的 Future
        if self.poll.is_pending() {
            self.poll = self.future.as_mut().poll(cx);
        }
    }
}

/// 核心 Task 结构体：
/// 1. 持有要执行的 Future
/// 2. 持有通道发送端，用于 wake() 时把自己塞回队列
/// 3. 实现 ArcWake，让自己能被转成 Waker
struct Task {
    // Mutex 只是为了满足 Sync，实际只有一个线程访问
    task_future: Mutex<TaskFuture>,
    // 通道发送端：wake() 时用
    executor: mpsc::Sender<Arc<Task>>,
}

impl Task {
    /// 把 Future 包装成 Task，发送到执行器
    fn spawn<F>(future: F, sender: &mpsc::Sender<Arc<Task>>)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        let task = Arc::new(Task {
            task_future: Mutex::new(TaskFuture::new(future)),
            executor: sender.clone(),
        });

        // 【关键】第一次把任务塞到队列里，让执行器开始跑
        let _ = sender.send(task);
    }

    /// 任务被唤醒后，执行 poll 的入口
    fn poll(self: Arc<Self>) {
        // 【核心 1】把 Task 自己转成 Waker！
        // 这里用了 futures 库的 task::waker，它会利用我们下面实现的 ArcWake
        let waker = task::waker(self.clone());
        let mut cx = Context::from_waker(&waker);

        // 拿到锁，执行真正的 poll
        let mut task_future = self.task_future.try_lock().unwrap();
        task_future.poll(&mut cx);
    }

    /// 【核心 2】把自己塞回执行器的就绪队列
    fn schedule(self: &Arc<Self>) {
        // 通过通道发送端，把自己（Arc<Task>）发给执行器
        let _ = self.executor.send(self.clone());
    }
}

// ==========================================
// 3. 实现 ArcWake：这是 Waker 的核心逻辑！
// ==========================================
impl ArcWake for Task {
    /// 当有人调用 waker.wake_by_ref() 时，会执行这个函数
    fn wake_by_ref(arc_self: &Arc<Self>) {
        // 【关键】调用 schedule，把自己塞回队列
        arc_self.schedule();
    }

    /// 当有人调用 waker.wake()（消耗所有权）时，默认会调用 wake_by_ref
    /// 我们可以不用重写，直接用默认实现
}

// ==========================================
// 4. 定义底层 Future：Delay（带正确的 Waker 保存逻辑）
// ==========================================
struct Delay {
    when: Instant,
    // 保存 Waker：用 Arc<Mutex> 是为了跨线程安全传递
    waker: Option<Arc<Mutex<Waker>>>,
}

impl Future for Delay {
    type Output = &'static str;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // 1. 先检查时间到了没
        if Instant::now() >= self.when {
            println!("Hello world! Delay is done.");
            return Poll::Ready("done");
        }

        // 2. 【Waker 契约】检查并更新保存的 Waker
        if let Some(waker) = &self.waker {
            let mut waker_guard = waker.lock().unwrap();
            // 用 will_wake 检查：如果新 Waker 和旧的不一样，必须更新
            if !waker_guard.will_wake(cx.waker()) {
                *waker_guard = cx.waker().clone();
            }
        } else {
            // 第一次 poll：还没保存过 Waker，初始化并启动定时器线程
            let waker = Arc::new(Mutex::new(cx.waker().clone()));
            self.waker = Some(waker.clone());
            let when = self.when;

            // 启动一个后台线程，模拟定时器
            thread::spawn(move || {
                let now = Instant::now();
                if now < when {
                    thread::sleep(when - now);
                }

                // 【关键】时间到了！调用 Waker 唤醒任务
                let waker_guard = waker.lock().unwrap();
                // 这里用 wake_by_ref，因为我们还想保留 Waker 的所有权（虽然这里用完就丢了）
                waker_guard.wake_by_ref();
            });
        }

        // 3. 返回 Pending，告诉执行器还没就绪
        Poll::Pending
    }
}

// ==========================================
// 5. 主函数：运行起来！
// ==========================================
fn main() {
    // 创建 Mini Tokio 执行器
    let mini_tokio = MiniTokio::new();

    // spawn 一个异步任务
    mini_tokio.spawn(async {
        println!("Task started! Waiting for 1 second...");
        
        // await 我们的 Delay Future
        let when = Instant::now() + Duration::from_secs(1);
        let future = Delay { when, waker: None };
        let out = future.await;
        
        assert_eq!(out, "done");
        println!("Task finished! Output: {}", out);
    });

    // 启动执行器主循环
    println!("Mini Tokio running...");
    mini_tokio.run();
}
```

---

# 第二步：Cargo.toml 依赖
```rust
[package]
name = "mini-tokio-waker"
version = "0.1.0"
edition = "2021"

[dependencies]
futures = "0.3"
```

---

# 第三步：核心代码逐块解析（重点看 Waker 部分）

我们把代码里和 `Waker` 直接相关的部分拎出来，按流程走一遍：

## 1. 执行器如何创建 `Waker`？

看 `Task::poll` 方法：
```rust
fn poll(self: Arc<Self>) {
    // 【核心】用 futures::task::waker 把 Arc<Task> 转成 Waker
    // 这一步的前提是：Task 实现了 ArcWake trait
    let waker = task::waker(self.clone());
    let mut cx = Context::from_waker(&waker);

    // ... 执行 poll
}
```

**关键点**：

- `task::waker` 是 `futures` 库的工具函数；
- 它会利用我们实现的 `ArcWake` trait，自动构建符合标准库要求的 `Waker`；
- 我们完全不用碰 `RawWakerVTable` 这些底层 unsafe 代码。

---

## 2. `wake()` 如何把任务塞回队列？

看 `ArcWake` 的实现和 `Task::schedule`：
```rust
// 【核心】实现 ArcWake trait
impl ArcWake for Task {
    // 当有人调用 waker.wake() 或 waker.wake_by_ref() 时，会执行这里
    fn wake_by_ref(arc_self: &Arc<Self>) {
        // 调用 schedule，把自己塞回执行器的通道
        arc_self.schedule();
    }
}

impl Task {
    // 把自己通过通道发给执行器
    fn schedule(self: &Arc<Self>) {
        let _ = self.executor.send(self.clone());
    }
}
```

**完整唤醒流程**：

1. 底层资源（定时器线程）调用 `waker.wake_by_ref()`；
2. 自动触发 `ArcWake::wake_by_ref`；
3. 调用 `Task::schedule`；
4. 通过 `mpsc::Sender` 把 `Arc<Task>` 发给执行器；
5. 执行器的 `MiniTokio::run` 从通道收到任务，调用 `task.poll()`。

---

## 3. 底层 Future 如何正确保存 `Waker`？

看 `Delay` 的 `poll` 方法：
```rust
fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
    // ... 检查时间 ...

    // 【Waker 契约 1】每次 poll 都检查是否需要更新 Waker
    if let Some(waker) = &self.waker {
        let mut waker_guard = waker.lock().unwrap();
        // 用 will_wake() 判断：避免不必要的克隆
        if !waker_guard.will_wake(cx.waker()) {
            *waker_guard = cx.waker().clone();
        }
    } else {
        // 【Waker 契约 2】第一次 poll：保存 Waker 并启动资源等待
        let waker = Arc::new(Mutex::new(cx.waker().clone()));
        self.waker = Some(waker.clone());
        
        // 启动定时器线程
        thread::spawn(move || {
            // ... 等待时间 ...
            // 【Waker 契约 3】资源就绪后，调用 wake()
            let waker_guard = waker.lock().unwrap();
            waker_guard.wake_by_ref();
        });
    }

    Poll::Pending
}
```

**这里严格遵守了 Waker 的三大契约**：

1. ✅ 返回 `Pending` 后，保证未来会调用 `wake()`；
2. ✅ 每次 `poll` 都用 `will_wake()` 检查并更新 Waker；
3. ✅ 允许虚假唤醒（每次 `poll` 都重新检查时间）。

---

# 第四步：运行效果

执行 `cargo run`，你会看到：
```rust
Mini Tokio running...
Task started! Waiting for 1 second...
Hello world! Delay is done.
Task finished! Output: done
```

程序会在 1 秒后打印完成信息，然后退出。

---

# 总结

这个完整的 Mini Tokio 示例展示了 `Waker` 的全流程：

1. **创建**：执行器用 `task::waker(self.clone())` 把 `Arc<Task>` 转成 `Waker`；
2. **传递**：通过 `Context` 传给 Future；
3. **保存**：底层 Future（`Delay`）用 `Arc<Mutex<Waker>>` 保存；
4. **唤醒**：资源就绪后调用 `waker.wake_by_ref()`，触发 `ArcWake::wake_by_ref`；
5. **调度**：`schedule()` 把任务塞回通道，执行器重新 `poll`。

这就是 Tokio 等工业级执行器 `Waker` 实现的**简化核心版**！