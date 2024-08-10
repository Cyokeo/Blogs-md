---
title: c/cpp小技巧
date: 2024-07-06 21:45:20
tags:
- [c/cpp]
categories:
- [编程技巧]
---

## 如何计算负数的补码
- 负数一般以补码的形式存储
- 如果位数为(8), 则负数a的补码为`pow(2,n) - abs(a)`
- 另外，负数a的补码也可根据：abs(a)的反码 + 1 得到

## lambda把自身作为形参
```cpp
// bool u_compare(string& l, string* r)
// {

// }

class Solution {
public:
    vector<vector<string>> accountsMerge(vector<vector<string>>& accounts) {
        unordered_map<string, vector<int>> mp;
        vector<vector<string>> ans;

        for (int i = 0; i < accounts.size(); i++)
        {
            for (int j = 1; j < accounts[i].size(); j++)
            {
                mp[accounts[i][j]].push_back(i);
            }
        }

        vector<bool> vis(accounts.size(), false);
        unordered_set<string> mails;
        auto dfs = [&](auto&& dfs, int idx) -> void {
            if (vis[idx] == false)
            {
                vis[idx] = true;
               for (int i = 1; i < accounts[idx].size(); i++)
               {
                    mails.insert(accounts[idx][i]);
                    for (int it : mp[accounts[idx][i]])
                    {
                        dfs(dfs, it);
                    }
               } 
            }
        };
        for (int i = 0; i < vis.size(); i++)
        {
            if (!vis[i])
            {
                mails.clear();
                dfs(dfs, i);
                vector<string> tmp{accounts[i][0]};
                tmp.insert(tmp.end(), mails.begin(), mails.end());
                sort(tmp.begin()+1, tmp.end());
                ans.push_back(tmp);
            }
        }
        
        return ans;
    }
};

```