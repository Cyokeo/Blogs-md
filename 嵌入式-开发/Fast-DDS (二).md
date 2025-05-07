---
title: Fast-DDS (二)
categories: 嵌入式-开发
---
## 重要方法
1. DataReaderImpl::InnerDataReaderListener::on_data_available
	1. 这个listener是rtps层的reader在处理新数据样本，接收函数最后会调用的！！
	2. 调用 -> on_new_cache_change_added()，这个函数内部会进行deadline/lifespan策略的检查

## QoS刷新理解
1. Deadline QoS
	1. 每个instance都要周期性更新；维护一个定时器，超时触发
	2. 每次接收到一个新的数据样本，如果数据样本与发起deadline定时器操作的是同一个instance，则重置定时器
	3. 如果在定时时间内都没有收到该instance的新数据样本，则触发超时
	4. instance的下一次超时时间设置为当前新数据样本接收【监听回调处理时使用now()获取】时间+deadline_duration
2. DDS 定义了四种 `Durability QoS` 级别，从最低到最高依次为：

	1. **VOLATILE_DURABILITY_QOS**：
	    
	    - **行为**：不保存任何历史数据。
	        
	    - **适用场景**：仅适用于实时数据传输，订阅者只能接收到订阅后发布的数据。
	        
	    - **示例**：传感器实时数据流，不需要历史数据。
        
	1. **TRANSIENT_LOCAL_DURABILITY_QOS**：
    
	    - **行为**：发布者会保存数据的历史记录，并在订阅者加入时将这些数据发送给订阅者。但数据不会在发布者离线后持久化。
	        
	    - **适用场景**：订阅者需要获取发布者在订阅之前发送的数据，但不需要在发布者离线后仍然保留数据。
	        
	    - **示例**：实时控制系统，订阅者需要获取发布者的最新状态。
        
	1. **TRANSIENT_DURABILITY_QOS**：
    
	    - **行为**：数据会在发布者离线后仍然保留在 DDS 的全局数据空间中（通常由 DDS 的持久化服务管理），直到订阅者获取。
	        
	    - **适用场景**：需要跨发布者生命周期的数据持久化。
	        
	    - **示例**：分布式日志系统，数据需要在发布者重启后仍然可用。
        
	1. **PERSISTENT_DURABILITY_QOS**：
    
	    - **行为**：数据会持久化到磁盘或其他非易失性存储中，即使系统重启，数据仍然可用。
	        
	    - **适用场景**：需要长期保存数据的应用。
	        
	    - **示例**：金融交易记录、医疗数据存档。
3. LatencyBudget QoS
	1. 