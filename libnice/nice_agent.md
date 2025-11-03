agent为一个g_object -> glib有一套完整的运行时*类型系统* 
`gtype.h`

可以使用glib的相关宏定义一个类型；会为该类型定义特定的函数；
1. 类型初始化函数；
2. 类型获取函数；
3. 在首次获取类型时，会调用once_init函数，对类型相关的局部static变量进行once初始化
	1. 这里会进行类型注册
	2. 类型名得使用字符串进行存储
	3. 注册相关函数为：g_type_register_static_simple()->g_type_register_static()
	4. 返回类型为：new type identifier *`GType`*
	5. type会按照父子关系进行存储；且所有类型信息都会存储在一张以type name进行hash的hashTable中

## 创建时初始化
也需要且glib中也有这种行为！

### 方法
G_DEFINE_TYPE (NiceAgent, nice_agent, G_TYPE_OBJECT);
使用该宏进行定义，并与glib进行交互！！！
例如：`type_name##_init()`函数就会在类型首次在glib的类型系统初始化时进行注册