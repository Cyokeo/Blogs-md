# 参考文档
- https://ipotato.me/article/70

# 源码分析
```rust
#[inline(never)]
async fn x() -> i32 {
    5
}

async fn y() -> i32 {
    x().await
}
```

## 编译器中间产物HIR伪代码
```rust
#[inline(never)]
fn x() -> impl Future<Output = usize> {
    from_generator(move |mut _task_context| {
        let _t = 5;
        _t
    })
}

fn y() -> impl Future<Output = usize> {
    from_generator(move |mut _task_context| {
        let mut pinned = into_future(x());
        loop {
            match unsafe {
                Pin::new_unchecked(&mut pinned).poll(_task_context.get_context());
            } {
                Poll::Ready(result) => break result,
                Poll::Pending => {}
            }
            yield
        }
    })
}
```

# 举例2
## 源码
```rust
async fn myTest() {
	let fut_one = /* ... */;
	let fut_two = /* ... */;
	
	fut_one.await;
	fut_two.await;
}
```

## 编译器可能进行的代码处理与生成
### 生成状态机相关
```rust
struct AsyncFuture {
	fut_one: Fut_one,
	fut_two: FutTwo,
	state: State,
}

enum State {
	AwaitingFutOne,
	AwaitIngFutTwo,
	Done,
}
```

### 代码处理
```rust
impl Future for AsyncFuture {
	type OutPut = ();
	
	fn poll(mut self: Pin<&mut Self>, ctx: &mut Context<'_>) {
		loop {
			match self.state {
				State::AwaitingFutOne => match self.fut_one.poll() {
					Poll::Ready(()) => self.state = State::AwaitingTwo,
					PollPending => return Poll:Pending,
				}
				State::AwaitingFutTwo => match self.fut_two.poll() {
					Poll::Ready(()) => self.state = State::Done,
					Poll::Pending => return Poll::Pending,
				}
				State::Done => return PollReady(()),
			}
		}
	}
}
```

# 思考
1. 对于加了`async`前缀的函数，编译器会自动帮我们进行状态机、Future生成，并帮我们生成一个对于Future的poll函数
	1. 我们不需要添加额外的内容，就可以完成此异步编程任务
	2. 此自动生成的poll函数，能够根据不同的状态，对不同的Future进行poll操作
	3. 进而就形成了与异步函数调用链基本一致的poll链条
	4. 直到遇到非async函数，需要执行用户自定义Future类型的poll方法
2. 对于返回Future的非async函数，我们要自己实现针对该返回Future的poll函数的实现
	1. 此函数中变量的保存，后续要再研究一下
3. 需要注意的是，对于Future的poll，应该要在对应的Task被执行一次后，再进行链式的poll操作