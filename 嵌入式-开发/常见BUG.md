---
title: 常见BUG
categories: 嵌入式-开发
---

#### 32位整形溢出
- 比较经典的是时间比较，需要考虑溢出情况
```c
if (lower - greater > 0x7FFFFFFF) {ok, smaller is true}
```