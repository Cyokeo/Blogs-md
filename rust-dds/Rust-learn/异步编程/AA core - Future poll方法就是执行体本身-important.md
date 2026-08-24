> https://tokio.rs/tokio/tutorial/async

## 源码
```rust
async fn muFn() {
    let when = Instant::now() + Duration::from_millis(10);
    let future = Delay { when };
    let out = future.await;
    assert_eq!(out, "done");
}
```

## 编译器处理后
生成的状态机以及poll大致如下，可以得到一个结论
1. 把正常代码部分，分阶段不同，塞到了不同的位置
```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

enum MainFuture {
    // Initialized, never polled
    State0,
    // Waiting on `Delay`, i.e. the `future.await` line.
    State1(Delay),
    // The future has completed.
    Terminated,
}

impl Future for MainFuture {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        use MainFuture::*;

        loop {
            match *self {
                State0 => {
                    let when = Instant::now() + Duration::from_millis(10);   // ！！！可以看到，把正常代码塞到了poll函数中
                    let future = Delay { when };
                    *self = State1(future);
                }
                State1(ref mut my_future) => match Pin::new(my_future).poll(cx) {
                    Poll::Ready(out) => {
                        assert_eq!(out, "done");
                        *self = Terminated;
                        return Poll::Ready(());
                    }
                    Poll::Pending => {
                        return Poll::Pending;
                    }
                },
                Terminated => {
                    panic!("future polled after completion")
                }
            }
        }
    }
}
```

## Mini Tokio
```rust
use std::collections::VecDeque;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};
use futures::task;

fn main() {
    let mut mini_tokio = MiniTokio::new();

    mini_tokio.spawn(async {
        let when = Instant::now() + Duration::from_millis(10);
        let future = Delay { when };

        let out = future.await;
        assert_eq!(out, "done");
    });

    mini_tokio.run();
}

struct MiniTokio {
    tasks: VecDeque<Task>,
}

type Task = Pin<Box<dyn Future<Output = ()> + Send>>;

impl MiniTokio {
    fn new() -> MiniTokio {
        MiniTokio {
            tasks: VecDeque::new(),
        }
    }

    /// Spawn a future onto the mini-tokio instance.
    fn spawn<F>(&mut self, future: F)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        self.tasks.push_back(Box::pin(future));
    }

    fn run(&mut self) {
        let waker = task::noop_waker();
        let mut cx = Context::from_waker(&waker);

        while let Some(mut task) = self.tasks.pop_front() {
            if task.as_mut().poll(&mut cx).is_pending() {
                self.tasks.push_back(task);
            }
        }
    }
}
```

## update using Waker
```rust
use std::sync::mpsc;
use std::sync::Arc; 
struct MiniTokio {
	scheduled: mpsc::Receiver<Arc<Task>>,
	sender: mpsc::Sender<Arc<Task>>, } 
	
struct Task { // This will be filled in soon. }
```

```rust
use std::sync::{Arc, Mutex};
/// A structure holding a future and the result of
/// the latest call to its `poll` method.
struct TaskFuture { 
	future: Pin<Box<dyn Future<Output = ()> + Send>>,
	poll: Poll<()>, }

struct Task {
// The `Mutex` is to make `Task` implement `Sync`. Only
// one thread accesses `task_future` at any given time.
// The `Mutex` is not required for correctness. Real Tokio
// does not use a mutex here, but real Tokio has
// more lines of code than can fit in a single tutorial
// page.
	task_future: Mutex<TaskFuture>,
	executor: mpsc::Sender<Arc<Task>>, }
	
impl Task {
	fn schedule(self: &Arc<Self>) { 
		self.executor.send(self.clone());
	}
}
```

```rust
use futures::task::{self, ArcWake};
use std::sync::Arc;
impl ArcWake for Task { 
	fn wake_by_ref(arc_self: &Arc<Self>) { 
		arc_self.schedule(); 
	} 
}

impl MiniTokio {
    fn run(&self) {
        while let Ok(task) = self.scheduled.recv() {
            task.poll();
        }
    }

    /// Initialize a new mini-tokio instance.
    fn new() -> MiniTokio {
        let (sender, scheduled) = mpsc::channel();

        MiniTokio { scheduled, sender }
    }

    /// Spawn a future onto the mini-tokio instance.
    ///
    /// The given future is wrapped with the `Task` harness and pushed into the
    /// `scheduled` queue. The future will be executed when `run` is called.
    fn spawn<F>(&self, future: F)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        Task::spawn(future, &self.sender);
    }
}

impl TaskFuture {
    fn new(future: impl Future<Output = ()> + Send + 'static) -> TaskFuture {
        TaskFuture {
            future: Box::pin(future),
            poll: Poll::Pending,
        }
    }

    fn poll(&mut self, cx: &mut Context<'_>) {
        // Spurious wake-ups are allowed, even after a future has
        // returned `Ready`. However, polling a future which has
        // already returned `Ready` is *not* allowed. For this
        // reason we need to check that the future is still pending
        // before we call it. Failure to do so can lead to a panic.
        if self.poll.is_pending() {
            self.poll = self.future.as_mut().poll(cx);
        }
    }
}

impl Task {
    fn poll(self: Arc<Self>) {
        // Create a waker from the `Task` instance. This
        // uses the `ArcWake` impl from above.
        let waker = task::waker(self.clone());
        let mut cx = Context::from_waker(&waker);

        // No other thread ever tries to lock the task_future
        let mut task_future = self.task_future.try_lock().unwrap();

        // Poll the inner future
        task_future.poll(&mut cx);
    }

    // Spawns a new task with the given future.
    //
    // Initializes a new Task harness containing the given future and pushes it
    // onto `sender`. The receiver half of the channel will get the task and
    // execute it.
    fn spawn<F>(future: F, sender: &mpsc::Sender<Arc<Task>>)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        let task = Arc::new(Task {
            task_future: Mutex::new(TaskFuture::new(future)),
            executor: sender.clone(),
        });

        let _ = sender.send(task);
    }
}
```

# 思考
很明显，这里还有个关键问题没有说明：
>Future的poll中该如何保存Waker呢？

## 对于async声明的函数
1. 显然，编译器对其处理后，肯定是不会有保存Waker的代码的；=> 因为编译器不可能需要知道我们用的哪种运行时！！！
2. 因此，这种函数是不需要保存Waker的！！！
## 链式调用
显然，async函数，最终肯定有某个调用的函数是非async的，且其一定会返回某个Future。而程序员需要在这个Future的poll实现中进行
Waker保存的操作！！！
