---
title: LW-DDS - OWCET
categories: 嵌入式-开发
---
## 一、owcet使用
1. -p 选项用于指定.osx文件中可配置参数，例如
	`owcet -s trivial dds.elf -p stages=2`

## 二、Processor类相关方法的调用顺序
  - `configure()`
  - `setup()`
  - `processWorkSpace()`
  - `cleanup()`
  - `void destroy(WorkSpace *ws)`-- the code processor is alive as long as its provided features are alive: this function is called when the feature is invalidated to release resources and properties of the provided features.

## 三、
1. 双指令发射（并行进入整数流水线和加载 / 存储流水线）
2. 仅用于循环指令的第三条流水线（零开销循环）
3. 将cdr测试均放在10ms任务中时，出现了任务切换导致的时延

## 四、代码阅读注意事项
1. void ParExeGraph::createNodes() // !!HUX!! Ensuring in-order for execute stage
2. 文献中的PD阶段是否对应repeat-cycle?
3. 由于I-cache line的存在，多条指令可以同时fetch，但是只能一条条的发射出去【repeat latency？】
4. ***还是要花大力气把这个源码框架走读一下***
5. `flowfact_FlowFactLoader.cpp`设置CPU的初始状态：寄存器，内存
6. `Identifier`机制确实有点复杂
7. 注意区分`Process`与`Processor`两种类

### 加载库并使用的方法
1. `dlopen()`打开库并获得句柄
2. 从库里搜索需要的函数/对象，进行使用

## 五、NMP語法
1. card表示无符号；coerce表示类型强转
2. `macro reverse16(v,rev)`这里感觉有些小问题
3. %s代表这是一个寄存器占位符?
4. `tricore_inst_t->ident`是对opcode进行比较复杂的解码最终得到的一个值，并不等于实际的opcode的值

## 六、Processor调用顺序
1. FlowFactLoader

2. LabelSetter: 遍历symbol，并添加到identifier中
	- FUNCTION_LABEL(i) = sym->name();
	- SYMBOL(i).add(\*sym);
	- LABEL(i) = sym->name();

3. Marker

4. CFGCollector {`AbstractCFGBuilder`}
	- `static Identifier<BasicBlock *> BB("", 0);`是很重要的一个identifier，以BB的起始inst为ID；这个Identifier记录了所有的BB
	- `AbstractCFGBuilder::processCFG`创建所有的BB
	- `AbstractCFGBuilder::buildEdges(CFGMaker& m)`在所有的BB间创建可能的edge
	- 这里一个marker代表一个函数入口！！！？call的入口
```cpp
void AbstractCFGBuilder::processWorkSpace(WorkSpace *ws) {
    for(int i = 0; i < makers.count(); i++)
        processCFG(makers[i].fst);
}
```

5. 会从给的入口处扫描BB，并创建edge；遇到call后，不递归处理，而是将其添加到marker中，等待延后的扫描BB，创建edge处理
	1. ！！！***从这里也能看出来，一个CFG就对应一个实际的call函数，而且其可能包含多个BB***
6. otawa::Dominance
7. otawa::LoopInfoBuilder
8. otawa::CacheConfigurationProcessor
9. otawa::MemoryProcessor
10. otawa::util::LBlockBuilder
	- 处理inst cache相关的
	- 使用`BB_LBLOCKS(bb)`记录所有bb的inst cache block信息
11. otawa::FirstLastBuilder
	- 一条CFG{Control Flow Graph}包含多个BB
	- inst cache line 相关
12. otawa::ACSBuilder
13. otawa::CAT2Builder
	- 分析inst的hit， miss情况
14. otawa::ipet::ILPSystemGetter
	- 需要好好看看
15. otawa::ipet::VarAssignment
	- 有关求解，需要好好看看

16. otawa::CAT2OnlyConstraintBuilder


17. otawa::clp::CLPAnalysis

18. otawa::dcache::CLPBlockBuilder
	- 需要读memory

## 七、otawa::ipet::WCETComputation求解调用
1. otawa::ipet::BasicConstraintsBuilder
	- otawa::ipet::VarAssignment：给BB，还有BB的out_edge添加对应的变量 - 供ILP时使用
	- 并创建基础的constraints
		1. 对于每个cfg，如果该cfg是elf的entry，则x = 1；否则该cfg是程序执行过程中的call调用产生的，则x = sum(x_caller)
		2. 对于cfg中的每个BB：如果该bb不是cfg的入口，那么x = sum(x_in)；如果该bb不是cfg的出口，那么x = sum(x_out)；
1. FlowFactConstraintBuilder::processBB
	- 用于处理作为循环体的的BB，假设给定循环体最大执行次数为max = 2；
	- 例：`e_0` -> bb -> bb_1 ->`e_1` -> bb; `e_2` -> bb -> bb_2 -> `e_3` -> bb
	- 则最终得到的约束为：
		- `X_e1 + X_e3 <= 2 X_e0 + 2 X_e2`
	- 因此可以看到，这里的max并不是该BB实际执行的总次数；代码里的`totol`才是总实际执行次数
3. `sys->addObjectFunction`
	1. 给目标函数添加变量
	2. EdgeTimeBuilder::contributeSplit
	3. ot::time cost = graph->analyze(); 计算耗时！！！
4. 从node【每个inst在pipeline的每个阶段都对应一个node】的latency中可以看到，node->latency都是在BBTimerTC16P中设置的！！！
	1. 某个bank的最差读取/写入时延为54，后续就全用这个计算？--> 太悲观了！！！

## 八、OWCET限制性较大
1. 只能分析不那么复杂的程序
	1. 用于分析cdr_serialize部分的时延
2. 其余部分还是采用测量的方式进行！！！
3. rtps_write的处理时延变化的几个可能
	1. data不同 -> 主要是这个，尤其在数据量比较大的情况下！！！
	2. History数量不同 -> 但是这个的影响已经比较小了
	3. 一次编译，rtps_write的不变量是基本保持不变的，变化的只有不同writer对应了不同的数据结构 -> 这个变化的部分使用静态数据分析的方式进行！vs 基于测量统计的方法
		1. 最终的误差的bounding有多大！可接受范围应该在`5us`级！！！
	4. `如何从基本的统计数据得到一个比较复杂数据结构的处理时延`
	5. 