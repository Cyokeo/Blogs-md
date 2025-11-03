```cpp
m_thread = new Thread(new TMethodJob<SocketMultiplexer>(this, &SocketMultiplexer::serviceThread));
```

可以看到这里取函数SocketMultiplexer::serviceThread的地址，由于成员函数占用的内存不依赖某一个具体实例化的类，即其已经有具体的占用内存空；因此可以直接取地址

```
因此，可以将类的成员函数看作一个具名的普通C函数
```

### 成员函数指针的用法
```cpp
SocketMultiplexer obj;

void (SocketMultiplexer::*pmethod)(void*) = &SocketMultiplexer::serviceThread;

(obj.*pmethod)(nullptr); // 这时才需要实例
```

因为成员函数调用时，需要隐式传入this（对象指针），因此调用时需要实例化对象