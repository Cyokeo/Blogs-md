1. 在模板重载决议过程中，如果某个模板的**参数替换失败**，这个模板会**从重载集中静默移除**，而不是导致编译错误；
2. C++ 模板只有在**真正被使用**时才会实例化

### 1. 类模板实例化阶段

`MyClass<int> a; // 只实例化类模板，不实例化成员函数`

此时编译器生成：

```
class MyClass<int> {
public:
// 只是声明存在，还没有实例化函数体
void process(constraint_t<std::is_integral_v<int>> = defaulted_constraint{});
void process(constraint_t<std::is_floating_point_v<int>> = defaulted_constraint{});
};
```

**注意**：此时还没有检查函数签名的有效性！

### 2. 成员函数实例化阶段

`a.process(); // 现在才实例化 process() 函数`

编译器尝试实例化两个重载：

#### 重载1：整数版本

```
void process(constraint_t<true> = defaulted_constraint{})
// 展开：constraint_t<true> → std::enable_if<true, defaulted_constraint>::type → defaulted_constraint
// 最终：void process(defaulted_constraint = defaulted_constraint{}) ✅
```

#### 重载2：浮点版本

```
void process(constraint_t<false> = defaulted_constraint{})
// 展开：constraint_t<false> → std::enable_if<false, defaulted_constraint>::type
// ❌ 替换失败！这个重载被移除
```

### 3. 最终结果

只有一个有效的重载：

`void process(defaulted_constraint = defaulted_constraint{}) // 整数版本`

## 验证示例

让我们添加一些调试输出：

```
#include <iostream>
#include <type_traits>

struct defaulted_constraint {
defaulted_constraint() { std::cout << "defaulted_constraint constructed\n"; }
};

template<bool B, typename T = defaulted_constraint>
using constraint_t = typename std::enable_if<B, T>::type;

template<typename T>
class MyClass {
public:
MyClass() { std::cout << "MyClass<" << typeid(T).name() << "> constructed\n"; }

void process(constraint_t<std::is_integral_v<T>> = defaulted_constraint{}) {
    std::cout << "Integral version called\n";
}

void process(constraint_t<std::is_floating_point_v<T>> = defaulted_constraint{}) {
    std::cout << "Floating point version called\n";
}
};

int main() {
    std::cout << "Creating MyClass<int>...\n";
    MyClass<int> a;  // 只调用构造函数

    std::cout << "Calling process()...\n";
    a.process();     // 现在才实例化 process() 函数
}
```

输出可能是：

```
Creating MyClass<int>...
MyClass<int> constructed  
Calling process()...
Integral version called
```

## 关键要点

1. **类模板实例化 ≠ 成员函数实例化**
2. **SFINAE 发生在函数模板实例化时**
3. **只有被调用的函数才会被实例化**
4. **未被调用的函数即使"有问题"也不会导致编译错误**