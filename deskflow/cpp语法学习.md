
## 常量成员函数
`double Stopwatch::getTime() const`
- 表示该成员函数不会修改对象的状态，即不会修改对象的成员变量
### 重载决议规则
- 如果同时存在 `void f()` 和 `void f() const`，编译器根据对象是否 `const` 选择最佳匹配：
```cpp
struct A {
    void f()       { std::cout << "non-const\n"; }
    void f() const { std::cout << "const\n"; }
};

A a1;        a1.f(); // 输出: non-const
const A a2;  a2.f(); // 输出: const
```

