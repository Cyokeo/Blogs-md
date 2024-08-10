---
title: 加减乘除与mod
date: 2024-06-25 22:16:57
tags:
categories:
- [刷题记录]
---

## 参考文章
- [力扣](https://leetcode.cn/circle/discuss/mDfnkW/)

## 记录要点

### 两个恒等式
- `(a + b) mod m = [(a mod m) + (b mod m)] mod m`
- `(a b) mod m = [(a mod m) (b mod m)] mod m`

### 幂运算与mod
- <font color=DC143C>指数不能随便取余</font>，如果指数在 64 位整数的范围内，可以使用[快速幂](https://leetcode.cn/problems/powx-n/description/)计算方法
> 注：如果指数超出 64 位整数的范围，需要用「欧拉降幂」处理。

### 负数与mod
- 如果`x`是负数，要采用`(x mod m + m) mod m`的形式，这样不用判断`x`是否为负数

### 除法与mod
- 参考上述链接


## 总结
代码实现时，上面的加减乘除通常这样写：
```cpp
MOD = 1_000_000_007

// 加
(a + b) % MOD

// 减
(a - b + MOD) % MOD

// 取模到 [0,MOD-1] 中，无论正负
(a % MOD + MOD) % MOD

// 乘
a * b % MOD

// 多个数相乘，要步步取模，防止溢出
a * b % MOD * c % MOD

// 除（MOD 是质数且 b 不是 MOD 的倍数）
a * qpow(b, MOD - 2, MOD) % MOD
```
其中 `qpow` 为快速幂函数。