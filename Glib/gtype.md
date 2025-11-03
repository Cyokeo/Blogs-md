## libnice - nice_agent
`G_DEFINE_TYPE (NiceAgent, nice_agent, G_TYPE_OBJECT);


## 简易案例
```c
// person.h - 头文件
#ifndef __PERSON_H__
#define __PERSON_H__

#include <glib-object.h>

G_BEGIN_DECLS

// 1. 定义类型宏
#define TYPE_PERSON            (person_get_type())
#define PERSON(obj)            (G_TYPE_CHECK_INSTANCE_CAST((obj), TYPE_PERSON, Person))
#define PERSON_CLASS(klass)    (G_TYPE_CHECK_CLASS_CAST((klass), TYPE_PERSON, PersonClass))
#define IS_PERSON(obj)         (G_TYPE_CHECK_INSTANCE_TYPE((obj), TYPE_PERSON))
#define IS_PERSON_CLASS(klass) (G_TYPE_CHECK_CLASS_TYPE((klass), TYPE_PERSON))
#define PERSON_GET_CLASS(obj)  (G_TYPE_INSTANCE_GET_CLASS((obj), TYPE_PERSON, PersonClass))

// 2. 前向声明
typedef struct _Person      Person;
typedef struct _PersonClass PersonClass;
typedef struct _PersonPrivate PersonPrivate;

// 3. 实例结构
struct _Person {
    GObject parent_instance;
    
    // 公共成员（不推荐直接访问）
    gchar *name;
    gint age;
    
    // 私有数据指针
    PersonPrivate *priv;
};

// 4. 类结构
struct _PersonClass {
    GObjectClass parent_class;
    
    // 虚函数指针
    void (*say_hello)(Person *self);
    void (*celebrate_birthday)(Person *self);
};

// 5. 公共API函数声明
GType person_get_type(void) G_GNUC_CONST;
Person* person_new(const gchar *name, gint age);
Person* person_new_empty(void);

// 访问器函数
void person_set_name(Person *self, const gchar *name);
const gchar* person_get_name(Person *self);
void person_set_age(Person *self, gint age);
gint person_get_age(Person *self);

// 方法
void person_say_hello(Person *self);
void person_celebrate_birthday(Person *self);

G_END_DECLS

#endif /* __PERSON_H__ */

// person.c - 实现文件
#include "person.h"
#include <stdio.h>

// 6. 私有结构定义
struct _PersonPrivate {
    gchar *email;
    gboolean is_adult;
};

// 7. 属性枚举
enum {
    PROP_0,
    PROP_NAME,
    PROP_AGE,
    PROP_EMAIL,
    N_PROPERTIES
};

static GParamSpec *obj_properties[N_PROPERTIES] = { NULL, };

// 8. 信号枚举
enum {
    SIGNAL_BIRTHDAY,
    SIGNAL_NAME_CHANGED,
    N_SIGNALS
};

static guint obj_signals[N_SIGNALS] = { 0, };

// 9. 使用G_DEFINE_TYPE_WITH_PRIVATE宏定义类型
G_DEFINE_TYPE_WITH_PRIVATE(Person, person, G_TYPE_OBJECT)

// 10. 前向声明内部函数
static void person_dispose(GObject *object);
static void person_finalize(GObject *object);
static void person_set_property(GObject *object, guint prop_id,
                               const GValue *value, GParamSpec *pspec);
static void person_get_property(GObject *object, guint prop_id,
                               GValue *value, GParamSpec *pspec);

// 虚函数的默认实现
static void person_real_say_hello(Person *self);
static void person_real_celebrate_birthday(Person *self);

// 11. 类初始化函数
static void person_class_init(PersonClass *klass)
{
    GObjectClass *object_class = G_OBJECT_CLASS(klass);
    
    // 设置对象生命周期函数
    object_class->dispose = person_dispose;
    object_class->finalize = person_finalize;
    object_class->set_property = person_set_property;
    object_class->get_property = person_get_property;
    
    // 设置虚函数的默认实现
    klass->say_hello = person_real_say_hello;
    klass->celebrate_birthday = person_real_celebrate_birthday;
    
    // 12. 安装属性
    obj_properties[PROP_NAME] = 
        g_param_spec_string("name",
                           "Name",
                           "Person's name",
                           NULL, /* default value */
                           G_PARAM_READWRITE |
                           G_PARAM_STATIC_STRINGS);
                           
    obj_properties[PROP_AGE] = 
        g_param_spec_int("age",
                        "Age", 
                        "Person's age",
                        0,     /* minimum */
                        150,   /* maximum */
                        0,     /* default */
                        G_PARAM_READWRITE |
                        G_PARAM_STATIC_STRINGS);
                        
    obj_properties[PROP_EMAIL] = 
        g_param_spec_string("email",
                           "Email",
                           "Person's email address",
                           NULL,
                           G_PARAM_READWRITE |
                           G_PARAM_STATIC_STRINGS);
    
    g_object_class_install_properties(object_class,
                                     N_PROPERTIES,
                                     obj_properties);
    
    // 13. 创建信号
    obj_signals[SIGNAL_BIRTHDAY] = 
        g_signal_new("birthday",
                    G_TYPE_FROM_CLASS(klass),
                    G_SIGNAL_RUN_LAST,
                    G_STRUCT_OFFSET(PersonClass, celebrate_birthday),
                    NULL, NULL,
                    g_cclosure_marshal_VOID__VOID,
                    G_TYPE_NONE, 0);
                    
    obj_signals[SIGNAL_NAME_CHANGED] = 
        g_signal_new("name-changed",
                    G_TYPE_FROM_CLASS(klass),
                    G_SIGNAL_RUN_FIRST,
                    0,
                    NULL, NULL,
                    g_cclosure_marshal_VOID__STRING,
                    G_TYPE_NONE, 1, G_TYPE_STRING);
}

// 14. 实例初始化函数
static void person_init(Person *self)
{
    // 获取私有数据
    self->priv = person_get_instance_private(self);
    
    // 初始化成员
    self->name = NULL;
    self->age = 0;
    self->priv->email = NULL;
    self->priv->is_adult = FALSE;
}

// 15. 对象销毁函数
static void person_dispose(GObject *object)
{
    Person *self = PERSON(object);
    
    // 释放引用计数对象
    // (在这个例子中我们没有持有其他GObject引用)
    
    G_OBJECT_CLASS(person_parent_class)->dispose(object);
}

static void person_finalize(GObject *object)
{
    Person *self = PERSON(object);
    
    // 释放内存
    g_free(self->name);
    g_free(self->priv->email);
    
    G_OBJECT_CLASS(person_parent_class)->finalize(object);
}

// 16. 属性访问函数
static void person_set_property(GObject *object, guint prop_id,
                               const GValue *value, GParamSpec *pspec)
{
    Person *self = PERSON(object);
    
    switch (prop_id) {
        case PROP_NAME:
            person_set_name(self, g_value_get_string(value));
            break;
        case PROP_AGE:
            person_set_age(self, g_value_get_int(value));
            break;
        case PROP_EMAIL:
            g_free(self->priv->email);
            self->priv->email = g_value_dup_string(value);
            break;
        default:
            G_OBJECT_WARN_INVALID_PROPERTY_ID(object, prop_id, pspec);
            break;
    }
}

static void person_get_property(GObject *object, guint prop_id,
                               GValue *value, GParamSpec *pspec)
{
    Person *self = PERSON(object);
    
    switch (prop_id) {
        case PROP_NAME:
            g_value_set_string(value, self->name);
            break;
        case PROP_AGE:
            g_value_set_int(value, self->age);
            break;
        case PROP_EMAIL:
            g_value_set_string(value, self->priv->email);
            break;
        default:
            G_OBJECT_WARN_INVALID_PROPERTY_ID(object, prop_id, pspec);
            break;
    }
}

// 17. 虚函数的默认实现
static void person_real_say_hello(Person *self)
{
    g_return_if_fail(IS_PERSON(self));
    
    printf("Hello, I'm %s, %d years old.\n", 
           self->name ? self->name : "Unknown", 
           self->age);
}

static void person_real_celebrate_birthday(Person *self)
{
    g_return_if_fail(IS_PERSON(self));
    
    self->age++;
    self->priv->is_adult = (self->age >= 18);
    
    printf("Happy birthday %s! You are now %d years old.\n",
           self->name ? self->name : "Unknown", 
           self->age);
}

// 18. 公共API实现
Person* person_new(const gchar *name, gint age)
{
    return g_object_new(TYPE_PERSON,
                       "name", name,
                       "age", age,
                       NULL);
}

Person* person_new_empty(void)
{
    return g_object_new(TYPE_PERSON, NULL);
}

void person_set_name(Person *self, const gchar *name)
{
    g_return_if_fail(IS_PERSON(self));
    
    if (g_strcmp0(self->name, name) != 0) {
        gchar *old_name = self->name;
        self->name = g_strdup(name);
        g_free(old_name);
        
        // 发射信号
        g_signal_emit(self, obj_signals[SIGNAL_NAME_CHANGED], 0, self->name);
        
        // 通知属性变化
        g_object_notify_by_pspec(G_OBJECT(self), obj_properties[PROP_NAME]);
    }
}

const gchar* person_get_name(Person *self)
{
    g_return_val_if_fail(IS_PERSON(self), NULL);
    return self->name;
}

void person_set_age(Person *self, gint age)
{
    g_return_if_fail(IS_PERSON(self));
    
    if (self->age != age) {
        self->age = age;
        self->priv->is_adult = (age >= 18);
        g_object_notify_by_pspec(G_OBJECT(self), obj_properties[PROP_AGE]);
    }
}

gint person_get_age(Person *self)
{
    g_return_val_if_fail(IS_PERSON(self), 0);
    return self->age;
}

void person_say_hello(Person *self)
{
    g_return_if_fail(IS_PERSON(self));
    
    PersonClass *klass = PERSON_GET_CLASS(self);
    if (klass->say_hello) {
        klass->say_hello(self);
    }
}

void person_celebrate_birthday(Person *self)
{
    g_return_if_fail(IS_PERSON(self));
    
    // 发射信号
    g_signal_emit(self, obj_signals[SIGNAL_BIRTHDAY], 0);
}

// 19. 使用示例
int main(void)
{
    // 初始化GObject类型系统
    g_type_init(); // 在较新版本的GLib中可能不需要
    
    // 创建对象
    Person *person1 = person_new("张三", 25);
    Person *person2 = person_new_empty();
    
    // 使用属性系统设置值
    g_object_set(person2, 
                "name", "李四",
                "age", 30,
                "email", "lisi@example.com",
                NULL);
    
    // 调用方法
    person_say_hello(person1);
    person_say_hello(person2);
    
    // 连接信号
    g_signal_connect(person1, "name-changed", 
                    G_CALLBACK(printf), "Name changed to: %s\n");
    
    // 触发属性变化
    person_set_name(person1, "张三丰");
    
    // 庆祝生日
    person_celebrate_birthday(person1);
    
    // 释放对象
    g_object_unref(person1);
    g_object_unref(person2);
    
    return 0;
}
```


