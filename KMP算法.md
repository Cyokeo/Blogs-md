---
title: KMP算法
date: 2024-07-06 15:46:03
tags:
categories:
- [刷题记录]
---

## 参考博客
- [leetcode 宫水三叶](https://leetcode.cn/problems/find-the-index-of-the-first-occurrence-in-a-string/solutions/575568/shua-chuan-lc-shuang-bai-po-su-jie-fa-km-tb86)
- [从next数组的求解解读KMP算法](https://leetcode.cn/problems/find-the-index-of-the-first-occurrence-in-a-string/solutions/2600821/kan-bu-dong-ni-da-wo-kmp-suan-fa-chao-qi-z1y0)
- [还是carl的好理解一些](https://programmercarl.com/0028.%E5%AE%9E%E7%8E%B0strStr.html#%E6%80%9D%E8%B7%AF)


## 解决场景

- 如何快速在「原字符串」中找到「匹配字符串」
- 时间复杂度：O(m + n)，其中m，n分别为原字符串，待匹配字符串的长度
- 其能在「非完全匹配」的过程中提取到有效信息进行复用，以减少「重复匹配」的消耗。


## 心得体会

- next数组是模式串的相同最长前后缀长度表
- next[j]表示模式串中，字串[0, j-1]的相同最长前后缀的长度
- “a”，“”，“abc”的最长前后缀长度均为0；“aabcaa”的最长相同前后缀的长度为2
- 当主串的第i个位置和模式串的第j个位置不相同时，模式串的匹配只需**回退**到 j' = next[j]的下一个位置。因为可以保证：`模式串的前j'个字符[0, j']与主串第i个位置前的j'个字符是相同的`


## 如何求模式串的next数组

- "abcabd"

- "abacabad"：
    ```
    -> b: next[0] = 0 所在的位置与b不同；由于next[0] = 0，所以next[1] = 0;
    -> a: next[1] = 0 所在的位置与a相同，因此next[2] = next[1] + 1 = 1;
    -> c: next[2] = 1 所在的位置与c不同；next[1] = 0 所在的位置与c不同；因此next[3] = 0;
    -> a: next[3] = 0 所在的位置与a相同，因此next[4] = next[3] + 1 = 1;
    -> b: next[4] = 1 所在的位置与b相同，因此next[5] = next[4] + 1 = 2;
    -> a: next[5] = 2 所在的位置与a相同，因此next[6] = next[5] + 1 = 3;
    -> d: next[6] = 3 所在的位置与d不同；next[next[6]] = next[3] = 0；因此next[7] = 0;
    ```

## 字符串哈希（https://leetcode.cn/problems/shortest-palindrome/solutions/1396220/by-flix-be4y)

- 构造next数组的算法：
    ```cpp
        int n = tmp.size();
        vector<int> next(n, 0);

        for (int i = 1; i < n; i++)
        {
            int k = i-1;
            while (k > 0 && tmp[next[k]] != tmp[i]) {k = next[k]-1;}
            if (k >= 0) next[i] = next[k] + 1;
            else next[i] = 0;
        }
    ```

## 典型题目
- [28. 找出字符串中第一个匹配项的下标](https://leetcode.cn/problems/find-the-index-of-the-first-occurrence-in-a-string/description/)
- [214. 最短回文串](https://leetcode.cn/problems/shortest-palindrome/solutions/392561/zui-duan-hui-wen-chuan-by-leetcode-solution/)