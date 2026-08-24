> 该类型表示能力槽cte_t相对于线程自身cnode空间的索引值，可能包含多级索引结构！

## 关键结构
`cspacepath_t`
capPtr x is a concatenated value of offsets. and cspacepath_t is used to find it's slot in a cnode

## Q& A
1. capptr_t 与cap类型的关系是什么？
	- capPtr: The address/pointer to a capability.