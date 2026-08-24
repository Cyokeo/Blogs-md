- https://course.rs/advance/async/future-excuting.html#%E5%BA%95%E5%B1%82%E6%8E%A2%E7%A7%98-future-%E6%89%A7%E8%A1%8C%E5%99%A8%E4%B8%8E%E4%BB%BB%E5%8A%A1%E8%B0%83%E5%BA%A6

## Future 特征
`Future` 特征是 Rust 异步编程的核心，毕竟异步函数是异步编程的核心，而 `Future` 恰恰是异步函数的返回值和被执行的关键。
首先，来给出 `Future` 的定义：它是一个能产出值的异步计算(虽然该值可能为空，例如 `()` )。光看这个定义，可能会觉得很空洞，我们来看看一个简化版的 `Future` 特征:
```rust
trait SimpleFuture {
    type Output;
    fn poll(&mut self, wake: fn()) -> Poll<Self::Output>;
}

enum Poll<T> {
    Ready(T),
    Pending,
}
```
在上一章中，我们提到过 `Future` 需要被执行器`poll`(轮询)后才能运行，诺，这里 `poll` 就来了，通过调用该方法，可以推进 `Future` 的进一步执行，直到被切走为止( 这里不好理解，但是你只需要知道 `Future` 并不能保证在一次 `poll` 中就被执行完，后面会详解介绍)。

若在当前 `poll` 中， `Future` 可以被完成，则会返回 `Poll::Ready(result)` ，反之则返回 `Poll::Pending`， 并且安排一个 `wake` 函数：当未来 `Future` 准备好进一步执行时， 该函数会被调用，然后管理该 `Future` 的执行器(例如上一章节中的`block_on`函数)会再次调用 `poll` 方法，此时 `Future` 就可以继续执行了。

如果没有 `wake` 方法，那执行器无法知道某个 `Future` 是否可以继续被执行，除非执行器定期的轮询每一个 `Future`，确认它是否能被执行，但这种作法效率较低。而有了 `wake`，`Future` 就可以主动通知执行器，然后执行器就可以精确的执行该 `Future`。 这种“事件通知 -> 执行”的方式要远比定期对所有 `Future` 进行一次全遍历来的高效。
```rust
pub struct SocketRead<'a> {
    socket: &'a Socket,
}

impl SimpleFuture for SocketRead<'_> {
    type Output = Vec<u8>;

    fn poll(&mut self, wake: fn()) -> Poll<Self::Output> {
        if self.socket.has_data_to_read() {
            // socket有数据，写入buffer中并返回
            Poll::Ready(self.socket.read_buf())
        } else {
            // socket中还没数据
            //
            // 注册一个`wake`函数，当数据可用时，该函数会被调用，
            // 然后当前Future的执行器会再次调用`poll`方法，此时就可以读取到数据
            self.socket.set_readable_callback(wake);
            Poll::Pending
        }
    }
}
```

## 使用 Waker 来唤醒任务
对于 `Future` 来说，第一次被 `poll` 时无法完成任务是很正常的。但它需要确保在未来一旦准备好时，可以通知执行器再次对其进行 `poll` 进而继续往下执行，该通知就是通过 `Waker` 类型完成的。
`Waker` 提供了一个 `wake()` 方法可以用于告诉执行器：相关的任务可以被唤醒了，此时执行器就可以对相应的 `Future` 再次进行 `poll` 操作。

执行器会管理一批 `Future` (最外层的 `async` 函数)，然后通过不停地 `poll` 推动它们直到完成。 最开始，执行器会先 `poll` 一次 `Future` ，后面就不会主动去 `poll` 了，而是等待 `Future` 通过调用 `wake` 函数来通知它可以继续，它才会继续去 `poll` 。这种 **wake 通知然后 poll** 的方式会不断重复，直到 `Future` 完成。

## 执行器和系统 IO
前面我们一起看过一个使用 `Future` 从 `Socket` 中异步读取数据的例子:
该例子中，`Future` 将从 `Socket` 读取数据，若当前还没有数据，则会让出当前线程的所有权，允许执行器去执行其它的 `Future` 。当数据准备好后，会调用 `wake()` 函数将该 `Future` 的任务放入任务通道中，等待执行器的 `poll` 。

关于该流程已经反复讲了很多次，相信大家应该非常清楚了。然而该例子中还有一个疑问没有解决：

- `set_readable_callback` 方法到底是怎么工作的？怎么才能知道 `socket` 中的数据已经可以被读取了？

关于第二点，其中一个简单粗暴的方法就是使用一个新线程不停的检查 `socket` 中是否有了数据，当有了后，就调用 `wake()` 函数。该方法确实可以满足需求，但是性能着实太低了，需要为每个阻塞的 `Future` 都创建一个单独的线程！

在现实世界中，该问题往往是通过操作系统提供的 `IO` 多路复用机制来完成，例如 `Linux` 中的 **`epoll`**，`FreeBSD` 和 `macOS` 中的 **`kqueue`** ，`Windows` 中的 **`IOCP`**, `Fuchisa`中的 **`ports`** 等(可以通过 Rust 的跨平台包 `mio` 来使用它们)。借助 IO 多路复用机制，可以实现一个线程同时阻塞地去等待多个异步 IO 事件，一旦某个事件完成就立即退出阻塞并返回数据。相关实现类似于以下代码：

