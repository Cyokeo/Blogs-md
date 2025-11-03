---
title: " Hash优化"
categories: 刷题记录
---
## 题库
- 给定相同长度的两个数组，问有几个区间，使得两个数组在这个区间中的异或值相同？
	- 目标是al^…^ar = bl^…br， 可以得出 al^…ar^bl^…br = 0， 于是可以定义 ci = ai^bi，有cl^…^cr = 0， 然后是比较熟悉的问题了， ***用map 存c的前缀异或出现的次数***；


## 字符串哈希操作
- [2430. 对字母串可执行的最大删除数](https://leetcode.cn/problems/maximum-deletions-on-a-string/)
	- 通过hash快速判断两个字符串是否相等
```cpp
using ull = unsigned long long;
const int N = 40001, P = 31;
ull h[N];
ull p[N];
ull shash(string& s, int l, int r)
{
	return h[r] - (l == 0 ? 0 : h[l - 1]*p[r - l + 1]);
}
p[0] = 1;
h[0] = s[0];

for(int i = 1; i < s.size(); i++)
{
	h[i] = h[i - 1]*P + s[i];
	p[i] = p[i - 1]*P;
}
```