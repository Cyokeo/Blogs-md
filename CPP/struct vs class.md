---
title: struct vs class
categories: cpp基础
tags:
  - 面经
---
## 参考
- [# C++ 中 class 和 struct 区别](https://csguide.cn/cpp/basics/class_and_struct.html#google_vignette)

## 相同与不同
cpp中struct 与 class基本是通用的，只有几个细节不一样
- class 中类中的成员默认都是 private 属性的
	- struct 中结构体中的成员默认都是 public 属性的
- class 继承默认是 private 继承
	- struct 继承默认是 public 继承
- class 可以用于定义模板参数：`template <class T>` -> 正确的
	- struct 不能用于定义模板参数: `template <struct T>` -> 错误的


## 使用习惯
实际使用中，struct 我们通常用来定义一些 POD(plain old data)
POD是 C++ 定义的一类数据结构概念，比如 int、float 等都是 POD 类型的

Plain 代表它是一个普通类型，Old 代表它是旧的，与几十年前的 C 语言兼容，那么就意味着可以使用 memcpy() 这种最原始的函数进行操作。

两个系统进行交换数据，如果没有办法对数据进行语义检查和解释，那就只能以非常底层的数据形式进行交互，而拥有 POD 特征的类或者结构体通过二进制拷贝后依然能保持数据结构不变。

也就是说，能用 C 的 memcpy() 等函数进行操作的类、结构体就是 POD 类型的数据。

而 class 用于定义一些 非 POD 的对象，面向对象编程。

