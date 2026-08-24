# ShecduleIo
> /Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tokio-1.49.0/src/runtime/io/scheduled_io.rs

每个socket都有一个ShceduleIo实例

## Registration
> /Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tokio-1.49.0/src/runtime/io/registration.rs

每个ScheduleIo会有一个Registration实例：Associates an I/O resource with the reactor instance that drives it.
```rust
/// A registration represents an I/O resource registered with a Reactor such
/// that it will receive task notifications on readiness. This is the lowest
/// level API for integrating with a reactor.
```
