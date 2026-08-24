# execution_context
```cpp
class execution_context{

	template <typename Service>
	friend bool has_service(execution_context& e);
	
	template <typename Service>
	friend void add_service(execution_context& e, Service* svc);
	
	detail::service_registry* service_registry_;
}
```

## `service_registry_`
从下面的源码可以看出：
1. 每个具体类型的service具有相同的Service::id；从目前的代码看，他们的key相同
2. 且从do_add_service()的代码看，一个execution_context只允许注册一个同类型的服务
3. 所有的service通过单链表的形式挂在## `service_registry_->first_service_`上
4. `service_registry_`以及每个service类型内部都有execution_context的引用`owner_`
### 相关源码
```cpp
class service_registry_{
	// Mutex to protect access to internal data.
	mutable asio::detail::mutex mutex_;
	// The owner of this service registry and the services it contains.
	execution_context& owner_;
	// The first service in the list of contained services.
	execution_context::service* first_service_;
};

template <typename Service>
void service_registry::add_service(Service* new_service)
{
	execution_context::service::key key;
	init_key<Service>(key, 0);
	return do_add_service(key, new_service);
}

template <typename Service>
inline void service_registry::init_key(
execution_context::service::key& key, ...)
{
	init_key_from_id(key, Service::id);
}

#if !defined(ASIO_NO_TYPEID)
template <typename Service>
void service_registry::init_key(execution_context::service::key& key,
enable_if_t<is_base_of<typename Service::key_type, Service>::value>*)
{
	key.type_info_ = &typeid(typeid_wrapper<Service>);
	key.id_ = 0;
}

template <typename Service>
void service_registry::init_key_from_id(execution_context::service::key& key,
	const service_id<Service>& /*id*/)
{
	key.type_info_ = &typeid(typeid_wrapper<Service>);
	key.id_ = 0;
}

void service_registry::init_key_from_id(execution_context::service::key& key,
	const execution_context::id& id)
{
	key.type_info_ = 0;
	key.id_ = &id;
}

bool service_registry::keys_match(
	const execution_context::service::key& key1,
	const execution_context::service::key& key2)
{
	if (key1.id_ && key2.id_)
		if (key1.id_ == key2.id_)
			return true;
	if (key1.type_info_ && key2.type_info_)
		if (*key1.type_info_ == *key2.type_info_)
			return true;
	return false;
}

// `typeid` 是 C++ 的**运行时类型识别（RTTI）**运算符，用于获取类型的相关信息
template <typename T>
class typeid_wrapper {};

void service_registry::do_add_service(
const execution_context::service::key& key,
execution_context::service* new_service)
{
	if (&owner_ != &new_service->context())
		asio::detail::throw_exception(invalid_service_owner());

	asio::detail::mutex::scoped_lock lock(mutex_);

	// Check if there is an existing service object with the given key.
	execution_context::service* service = first_service_;
	while (service)
	{
		if (keys_match(service->key_, key))
			asio::detail::throw_exception(service_already_exists());
		service = service->next_;
	}
	
	// Take ownership of the service object.
	if (!new_service->destroy_)
		new_service->destroy_ = &service_registry::destroy_added;
	new_service->key_ = key;
	new_service->next_ = first_service_;
	first_service_ = new_service;
}
```

## service
```cpp
class execution_context::service{
	//...
	struct key
	{
		key() : type_info_(0), id_(0) {}
		const std::type_info* type_info_;
		const execution_context::id* id_;
	} key_;
	execution_context& owner_;
	service* next_;
	void (*destroy_)(service*);
}
```

## execution_context_service_base
```cpp
// Special derived service id type to keep classes header-file only.
template <typename Type>
class service_id : public execution_context::id
{
};

// Special service base class to keep classes header-file only.
template <typename Type>
class execution_context_service_base : public execution_context::service
{
public:
	static service_id<Type> id;

	// Constructor.
	execution_context_service_base(execution_context& e): execution_context::service(e)
	{
	}
};
```
