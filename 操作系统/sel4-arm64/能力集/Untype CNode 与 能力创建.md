# createNewObjects
invokeUntyped_Retype() -> createNewObjects() -> createObject() + insertNewCap

srcSlot是untype的，调用Retype将其转化为某个类型的内核对象。新的内核对象的内存就来自srcSlot->cap的Untype内存。
需要注意：一个Untype含有的内存可能有很多，其需要维护剩余内存量；其可以转化为多个内核对象

创建完内核对象后，其对应的cap_t会被写入到dst_slot中；且会将该dst_slot的mdbNode插入到srcSlot的mdbNode双向链表中
总是插入到head节点和head->next之间。

# 总结
后续主动创建的所有内核对象，都是从untype cnode 经过Retype得到的；其mdbnode在untype conde的mdbnode双向链表中

# 内核启动过程中关注一下untype cnode