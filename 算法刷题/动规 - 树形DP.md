---
title: 动规 - 树形DP
categories: 刷题记录
---
## 难点一
- 从上往下计算：dfs时要传入多个参数
- 从下往上计算：dfs返回多个值

### 网友评论
这种题的话，需要实现节点之间信息的交流，二叉树有3种遍历，那就有3种信息交流的方式，***前序的「自上而下」***，***后序的「自下而上」***，***中序的「左上右下」***

## 从下往上 - 后序遍历
##### [968. 监控二叉树](https://leetcode.cn/problems/binary-tree-cameras/)
- [代码随想录](https://leetcode.cn/problems/binary-tree-cameras/solutions/423212/968-jian-kong-er-cha-shu-di-gui-shang-de-zhuang-ta)
- 根据题目要求，这题选用后续遍历是传递信息的最佳选择。需要传递的信息有三种：1.要求父节点安装相机；2.说自己有相机；3.自己不需要相机或空节点；那么，每个节点在接受了两个子节点的信息后，还要判断自己要向父节点传递哪种信息，如此层层向上传递，便解决了问题



## 从上往下 - 前序遍历
##### [129. 求根节点到叶节点数字之和](https://leetcode.cn/problems/sum-root-to-leaf-numbers/)

##### [257. 二叉树的所有路径](https://leetcode.cn/problems/binary-tree-paths/)
