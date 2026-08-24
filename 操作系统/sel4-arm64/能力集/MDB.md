## MDB节点的数据结构

从代码中可以看出MDB节点包含：

c

mdb_node_t {
    mdbNext;      // 后向指针
    mdbPrev;      // 前向指针  
    mdbRevocable; // 是否可撤销
    mdbFirstBadged; // 是否是第一个badged能力
}

## MDB维护的关系网络

### 1. **双向链表结构**

text

srcSlot ←→ destSlot ←→ nextSlot

### 2. **具体维护步骤**

#### a. 初始化新MDB节点

c

newMDB = mdb_node_set_mdbPrev(srcMDB, CTE_REF(srcSlot));  // 设置prev指向源槽
newMDB = mdb_node_set_mdbRevocable(newMDB, newCapIsRevocable);
newMDB = mdb_node_set_mdbFirstBadged(newMDB, newCapIsRevocable);

#### b. 验证目标槽位为空

c

assert(cap_get_capType(destSlot->cap) == cap_null_cap);  // 目标能力必须为空
assert(mdb_node_get_mdbNext(destSlot->cteMDBNode) == NULL &&  // MDB节点必须为空
       mdb_node_get_mdbPrev(destSlot->cteMDBNode) == NULL);

#### c. 更新链表关系

c

// 1. 设置目标槽位的能力和MDB
destSlot->cap = newCap;
destSlot->cteMDBNode = newMDB;

// 2. 源槽位的next指向目标槽位
mdb_node_ptr_set_mdbNext(&srcSlot->cteMDBNode, CTE_REF(destSlot));

// 3. 如果新节点有next，更新那个节点的prev指向新节点
if (mdb_node_get_mdbNext(newMDB)) {
    mdb_node_ptr_set_mdbPrev(
        &CTE_PTR(mdb_node_get_mdbNext(newMDB))->cteMDBNode,
        CTE_REF(destSlot));
}

## 关系维护的语义

### 1. **父子关系**

- `srcSlot` 是父节点（源能力）
    
- `destSlot` 是新创建的子节点（复制的能力）
    
- 通过 `mdbPrev` 字段建立从子到父的链接
    

### 2. **兄弟关系**

- 同一父节点的所有子节点通过 `mdbNext` 连接成链表
    
- 新插入的 `destSlot` 成为链表的头部
    

### 3. **撤销关系**

c

newCapIsRevocable = isCapRevocable(newCap, srcCap);

- 根据能力和源能力类型决定是否可撤销
    
- 影响能力撤销机制的操作
    

## 示例场景

假设已有结构：

text

父能力A → 子能力B → 子能力C

执行 `cteInsert(D, A, 空槽)` 后：

text

父能力A → 新能力D → 子能力B → 子能力C
            ↑
        prev指向A

## 在能力系统中的作用

### 1. **能力派生跟踪**

- 跟踪所有从同一原始能力派生的能力
    
- 支持能力撤销时找到所有相关能力
    

### 2. **撤销机制基础**

- 可撤销标志决定是否参与撤销操作
    
- 双向链表支持高效遍历
    

### 3. **内存安全**

- 确保MDB结构的完整性
    
- 防止重复插入和循环引用
    

### 4. **未满能力管理**

c

setUntypedCapAsFull(srcCap, newCap, srcSlot);

- 当从未满能力创建子能力时，标记父能力为"已满"
    
- 防止能力空间的重复分配
    

这种MDB维护机制是seL4能力系统能够安全、高效管理能力派生和撤销的关键基础设施。