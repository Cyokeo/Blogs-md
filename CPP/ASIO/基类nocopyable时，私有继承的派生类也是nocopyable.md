## 基本示例
```cpp
#include <iostream>

// 基类：不可拷贝
class NonCopyable {
protected:
    NonCopyable() = default;
    ~NonCopyable() = default;
    
public:
    NonCopyable(const NonCopyable&) = delete;
    NonCopyable& operator=(const NonCopyable&) = delete;
};

// 私有继承
class Derived : private NonCopyable {
public:
    Derived() = default;
    int value = 42;
};

int main() {
    Derived d1;
    
    // 尝试拷贝
    Derived d2 = d1;  // 编译错误！
    // error: use of deleted function 'Derived::Derived(const Derived&)'
    
    Derived d3;
    d3 = d1;  // 编译错误！
    // error: use of deleted function 'Derived& Derived::operator=(const Derived&)'
    
    return 0;
}
```

## 为什么私有继承也会禁止拷贝？
### 1. **编译器生成的拷贝构造函数**
```cpp
class NonCopyable {
public:
    NonCopyable(const NonCopyable&) = delete;
    NonCopyable& operator=(const NonCopyable&) = delete;
};

class Derived : private NonCopyable {
    // 编译器尝试生成默认拷贝构造函数：
    // Derived(const Derived& other) 
    //     : NonCopyable(other)  // 需要调用基类的拷贝构造函数
    //     , /* 拷贝成员变量 */
    // { }
    
    // 但是！NonCopyable 的拷贝构造函数被删除了
    // 所以编译器无法生成 Derived 的拷贝构造函数
    // 结果：Derived 的拷贝构造函数也被隐式删除
};
```

### 2. 显式尝试定义拷贝构造函数
```cpp
class NonCopyable {
public:
    NonCopyable() = default;
    NonCopyable(const NonCopyable&) = delete;
};

class Derived : private NonCopyable {
public:
    Derived() = default;
    
    // 尝试显式定义拷贝构造函数
#if 0
    Derived(const Derived& other) 
        : NonCopyable(other)  // 错误！无法调用删除的函数
    {
    }
#endif
	// 但是下面这样定义是可以的
	Derived(const Derived& other) 
    {
	    // 即不调用父类的拷贝构造
    }
};

// 编译错误：
// error: use of deleted function 'NonCopyable::NonCopyable(const NonCopyable&)'
```