
# 底层通信资源与DDS的对应关系
1. 从***RTPS Message Receiver***的描述来看，当收到一个新的RTPS消息时，应该能够知道该消息是属于哪个participant的！！！
2. 从socket的角度来看：socket与participant是1:1的关系；但是一个participant可能有多个socket

# tokio vs mio
`tokio-proto` 和 `tokio-service` 的 crates 提供了编写协议和服务的框架，但我认为 `tokio` 仅仅是 `mio` 和 `futures,` 的合并，为 `mio`提供更高级别但零成本的抽象。整个 tokio 项目都是基于此构建 Tokio 解决方案的。
