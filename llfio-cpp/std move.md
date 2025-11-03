The primary purpose of `std::move` is **to enable move semantics**. By **casting an lvalue to an rvalue reference**, `std::move` signals to the compiler that the object being referenced is about to be "moved from

cpp reference: `std::move`用于标记一个对象可以被移走（即资源可以被转移），它生成一个将亡值（xvalue）表达式，标识其参数t。它**等价于一个到右值引用类型的静态转换**。

## 本质实现
```cpp
template <typename T>
constexpr typename std::remove_reference<T>::type&& move(T&& t) noexcept {
    return static_cast<typename std::remove_reference<T>::type&&>(t);
}
```
实际等价于
```cpp
static_cast<MyType&&>(t);  // 强制转换为右值引用
```

`std::move` 本质是**编译器指令**而非运行时操作：
- 编译期作用：改变表达式值类别
- 运行时作用：启用移动操作
- 性能影响：0 开销抽象（编译后无额外指令）