---
title: cpp - weak_ptr
categories: cpp基础
---
## 参考博客
- [深入理解C++ weak_ptr](https://csguide.cn/cpp/memory/how_to_understand_weak_ptr.html#weak-ptr-%E6%98%AF%E4%BB%80%E4%B9%88)
- [关于std::weak_ptr使用的理解](https://0cch.com/2022/10/31/some-tips-about-weakptr/)



## 总结
- 抽象出一个对立的概念
	- ==内存所有者==：用于管理内存的创建/释放
	- ==内存观察者==：可以访问内存；
		- 但是访问前需要变为临时的内存所有者
		- 或者锁住对内存的创建/释放/修改等操作
- weak_ptr 和 shared_ptr一样，多线程使用时会涉及到数据竞争的问题
	- 