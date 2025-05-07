---
title: 论文（四）- CDR_Delay
categories: 嵌入式-开发
---
## CPU时钟改变
1. 通过`Ifx_Cfg.h`文件进行

## 计划-步骤
1. 搭建数据集【自动化】采集框架
2. MLP
3. 参考X-Types定义所有相关的特征
	1. 还有float类型
4. 使用otawa-old下面的otawa/test进行估计
	1. 且需要自定义序列化函数，不然可能运行不了；参考sim_data文件，该文件中自定义序列化函数做了简化；但是otawa估计方法的估计仍然偏大
5. 放在20ms的任务里为84，而放在5ms的任务里为81
	1. 可能测量的81里面也包含了一些打断的操作；需要重点关注
	2. 在放到主线程的回调里测试一下

## 细节
1. 如何处理string类型的特征，因为可能包含多个string，且每个string的长度不同
	1. 两个特性：string出现的次数，string的平均长度（或者总长度）
	2. 长度标准差？
	3. 模型的数据输入，也以CSV文件的方式进行吧！！！
2. 不同类型的损失函数
	1. 平均绝对误差`(1/n)*sum(|y_i_real - y_i_pred|)`
	2. 均方差，样本方差的平均值
3. Huber 损失
4. 学习率【参数调整系数/变化率】调整、权重衰减系数【防止过拟合】调整原则
5. 相同的种类数量，但是不同的前后顺序 -> 检测是否与前后顺序相关！！！
6. string使用的memcpy()函数，在4字节对齐/不对齐的情况下，延时差距很大
	1. 如果string全部与4字节对齐，则延时基本只与string_cnt，string_len【总长度】相关
	2. 如果string未与4字节对齐，则时延还与未对齐的string的长度有关
7. 系统时钟配置为1ms时，确实抖动要小很多
	1. 系统时钟配置为10ns时，-> 也是一样比较小的抖动
	2. 还是要看系统中有没有其它可能打断的操作，以及操作有多少！
	3. 我加个MCU的接收后，就能看到比较多的抖动了！！！

## 损失函数
1. nn.L1Loss() -> 平均绝对值误差损失
2. nn.MSELoss() -> 均方误差损失
3. 自定义损失函数：-> 当误差大于某个阈值时，加大惩罚
```python
import torch
import torch.nn as nn
class CustomLoss(nn.Module):
	def __init__(self, threshold=1.0):
		super(CustomLoss, self).__init__()
		self.threshold = threshold
	def forward(self, y_pred, y_true):
		diff = y_pred - y_true
		mask = torch.abs(diff) > self.threshold
		weighted_diff = torch.where(mask, 2 * diff, diff)
		loss = torch.mean(weighted_diff ** 2)
		return loss
```
4. 自定义损失函数代码解释：
	-  **类定义和初始化**：
	    - 首先定义一个名为 `CustomLoss` 的类，它继承自 `nn.Module`，这是 PyTorch 中定义神经网络模块（包括损失函数）的基类。在 `__init__` 方法中，接收一个可选参数 `threshold`（默认为 1.0），用于设定差异的阈值。
	- **前向传播方法（`forward`）**：
	    - 计算预测值 `y_pred` 和真实值 `y_true` 之间的差值 `diff`。
	    - 创建一个布尔掩码 `mask`，用于标记差值的绝对值大于阈值的位置。
	    - 使用 `torch.where` 函数根据掩码 `mask` 对差值进行加权。如果差值大于阈值，将差值乘以 2，否则保持不变。
	    - 最后，计算加权差值的平方的平均值作为损失值并返回。
