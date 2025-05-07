---
title: 利用queue拆解比较简单的递归
categories: 算法刷题
---
#### 题目1
给定一个整数 `N`，找到最小的正整数 `M`，使得 `N * M` 的十进制表示仅由数字 `0` 和 `1` 组成。
我们需要找到最小的 `M`，使得 `N * M` 是一个仅包含 `0` 和 `1` 的数字。这个问题可以转化为 **搜索由 `0` 和 `1` 组成的数字是否能被 `N` 整除**，并找到最小的那个。
##### 方法：BFS（广度优先搜索）
```python
from collections import deque

def find_smallest_M(N):
    if N == 0:
        return -1  # 0 * M = 0，但题目可能要求 M > 0
    visited = set()  # 记录已经出现过的余数
    queue = deque()
    queue.append(1)  # 初始数字是 1（因为 M > 0，所以 X 至少是 1）
    visited.add(1 % N)
    
    while queue:
        current = queue.popleft()
        if current % N == 0:
            return current // N
        # 生成下一个数字：在后面添加 0 或 1
        for digit in [0, 1]:
            next_num = current * 10 + digit
            remainder = next_num % N
            # 这一步的处理需要体会一下！
            if remainder not in visited:
                visited.add(remainder)
                queue.append(next_num)
    return -1  # 理论上一定存在解（如 M = 111...111 / N）
```