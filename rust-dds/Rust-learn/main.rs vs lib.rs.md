1. 如果crate中定义了其他的非src/main.rs的可执行文件入口，则一定要有src/lib.rs文件进行mod tree的构建；
2. 此时，main.rs可以省去mod tree的构建，直接使用即可
但还是有区别：
3. 使用lib.rs时，use net::server::UdpServer
4. 使用main.rs时，其内部使用crate::server::UdpServer