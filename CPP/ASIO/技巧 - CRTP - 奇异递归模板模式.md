## 代码
- 这还是一个两层CRTP
```cpp
template <typename Protocol>
class reactive_socket_service :
	public execution_context_service_base<reactive_socket_service<Protocol>>,
	public reactive_socket_service_base
{
	//...
}
```

## 什么是 CRTP？
**CRTP** 是一种模板元编程技术，其中**一个类继承自以自己为模板参数的模板基类**
- 可以实现静态多态
	- 即基类可以访问子类中的方法
```cpp
// 基本模式
template<typename Derived>
class Base {
    // 基类可以访问 Derived 的方法
};

class Derived : public Base<Derived> {  // 🔥 关键：把自己作为模板参数！
    // 派生类
};
```

## CRTP 的工作原理

### 1. **静态多态（编译时多态）**
```cpp
template<typename Derived>
class Base {
public:
    void interface() {
        // 静态向下转型（编译时）
        static_cast<Derived*>(this)->implementation();
    }
    
    void common_operation() {
        // 公共实现
        std::cout << "Common operation in Base\n";
    }
};

class Derived1 : public Base<Derived1> {
public:
    void implementation() {
        std::cout << "Derived1 implementation\n";
    }
};

class Derived2 : public Base<Derived2> {
public:
    void implementation() {
        std::cout << "Derived2 implementation\n";
    }
};

int main() {
    Derived1 d1;
    Derived2 d2;
    
    d1.interface();  // 调用 Derived1::implementation()
    d2.interface();  // 调用 Derived2::implementation()
}
```

### 2. **模板类的静态成员变量在头文件中定义不会造成ODR【multiple def】问题**
```cpp
#include <iostream>

// 模板定义（在头文件中）
template<typename T>
class MyTemplate {
public:
    static void func() {
        std::cout << "func called" << std::endl;
    }
    
    static int count;
};

// 静态成员变量也需要定义（C++17 前）
template<typename T>
int MyTemplate<T>::count = 0;

// file1.cpp 编译后生成的符号
// MyTemplate<int>::func() [弱符号/weak symbol]
// MyTemplate<int>::count [弱符号]

// file2.cpp 编译后生成的符号  
// MyTemplate<int>::func() [弱符号]
// MyTemplate<int>::count [弱符号]

// 链接器看到多个弱符号时：保留一个，丢弃其他的（不报错）
```
#### 2.1 cpp标准规定
```cpp
// C++ 标准的 ODR 例外情况：

1. 模板的实例化可以在多个翻译单元中定义
2. 内联函数可以在多个翻译单元中定义
3. constexpr 函数可以在多个翻译单元中定义

// 条件：这些定义必须完全相同（token-by-token identical）
```