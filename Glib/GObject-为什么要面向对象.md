
## 相关链接
- [GTYPE类型系统分析](http://os.is-programmer.com/posts/40285.html)
- [GType类型系统（上）](https://brionas.github.io/2014/06/14/GType-1/)
- [Glib之GObject简介（翻译）](https://www.cnblogs.com/silvermagic/p/9087883.html "发布于 2018-05-25 12:11")

## 案例

```c
/* =================================================================
   问题1：传统C语言实现GUI组件的困难
   ================================================================= */

// 传统C语言方式 - 问题重重的实现
// ========================================

// button.h - 传统方式
typedef struct {
    int x, y, width, height;
    char* label;
    void (*click_handler)(void* data);
    void* user_data;
} Button;

typedef struct {
    int x, y, width, height;
    char* text;
    int cursor_pos;
    void (*text_changed)(char* new_text, void* data);
    void* user_data;
} TextEntry;

// 问题1：每个组件都要重复定义位置、大小等公共属性
// 问题2：无法统一处理不同类型的组件
// 问题3：没有继承，代码大量重复

void draw_button(Button* btn) {
    // 绘制按钮的代码...
    printf("Drawing button '%s' at (%d,%d)\n", btn->label, btn->x, btn->y);
}

void draw_text_entry(TextEntry* entry) {
    // 绘制文本框的代码...
    printf("Drawing text entry '%s' at (%d,%d)\n", entry->text, entry->x, entry->y);
}

// 问题4：无法统一处理事件
void handle_click_traditional(void* widget, char* type) {
    if (strcmp(type, "button") == 0) {
        Button* btn = (Button*)widget;
        if (btn->click_handler) {
            btn->click_handler(btn->user_data);
        }
    } else if (strcmp(type, "text_entry") == 0) {
        // TextEntry不处理点击...但这里已经很混乱了
    }
    // 需要为每种类型写不同的处理逻辑！
}

/* =================================================================
   解决方案：GObject方式 - 优雅的面向对象解决方案
   ================================================================= */

// GObject方式 - 优雅的解决方案
// ================================

#include <glib-object.h>

// 1. 基类Widget - 所有GUI组件的共同祖先
#define TYPE_WIDGET (widget_get_type())
G_DECLARE_DERIVABLE_TYPE(Widget, widget, , WIDGET, GObject)

struct _WidgetClass {
    GObjectClass parent_class;
    
    // 虚函数 - 子类可以重写
    void (*draw)(Widget* self);
    void (*handle_event)(Widget* self, int event_type);
    
    // 信号槽
    void (*clicked)(Widget* self);
};

// 2. Button类继承自Widget
#define TYPE_BUTTON (button_get_type())
G_DECLARE_FINAL_TYPE(Button, button, , BUTTON, Widget)

// 3. TextEntry类也继承自Widget
#define TYPE_TEXT_ENTRY (text_entry_get_type())
G_DECLARE_FINAL_TYPE(TextEntry, text_entry, , TEXT_ENTRY, Widget)

// Widget基类实现
G_DEFINE_TYPE(Widget, widget, G_TYPE_OBJECT)

enum {
    PROP_WIDGET_0,
    PROP_X,
    PROP_Y,
    PROP_WIDTH,
    PROP_HEIGHT,
    PROP_VISIBLE,
    N_WIDGET_PROPS
};

enum {
    SIGNAL_CLICKED,
    SIGNAL_FOCUS_IN,
    SIGNAL_FOCUS_OUT,
    N_SIGNALS
};

static GParamSpec* widget_props[N_WIDGET_PROPS] = {NULL,};
static guint widget_signals[N_SIGNALS] = {0,};

static void widget_class_init(WidgetClass* klass) {
    GObjectClass* object_class = G_OBJECT_CLASS(klass);
    
    // 默认虚函数实现
    klass->draw = widget_default_draw;
    klass->handle_event = widget_default_handle_event;
    
    // 定义所有Widget共有的属性
    widget_props[PROP_X] = g_param_spec_int("x", "X", "X coordinate",
        0, G_MAXINT, 0, G_PARAM_READWRITE);
    widget_props[PROP_Y] = g_param_spec_int("y", "Y", "Y coordinate", 
        0, G_MAXINT, 0, G_PARAM_READWRITE);
    widget_props[PROP_WIDTH] = g_param_spec_int("width", "Width", "Widget width",
        1, G_MAXINT, 100, G_PARAM_READWRITE);
    widget_props[PROP_HEIGHT] = g_param_spec_int("height", "Height", "Widget height",
        1, G_MAXINT, 30, G_PARAM_READWRITE);
    widget_props[PROP_VISIBLE] = g_param_spec_boolean("visible", "Visible", "Is widget visible",
        TRUE, G_PARAM_READWRITE);
    
    g_object_class_install_properties(object_class, N_WIDGET_PROPS, widget_props);
    
    // 定义信号
    widget_signals[SIGNAL_CLICKED] = g_signal_new("clicked",
        G_TYPE_FROM_CLASS(klass), G_SIGNAL_RUN_LAST,
        G_STRUCT_OFFSET(WidgetClass, clicked),
        NULL, NULL, g_cclosure_marshal_VOID__VOID,
        G_TYPE_NONE, 0);
}

static void widget_init(Widget* self) {
    // 初始化默认值通过属性系统自动处理
}

static void widget_default_draw(Widget* self) {
    g_print("Drawing generic widget at (%d,%d)\n", 
            widget_get_x(self), widget_get_y(self));
}

static void widget_default_handle_event(Widget* self, int event_type) {
    if (event_type == 1) { // 假设1是点击事件
        g_signal_emit(self, widget_signals[SIGNAL_CLICKED], 0);
    }
}

// Button类实现
G_DEFINE_TYPE(Button, button, TYPE_WIDGET)

static void button_draw(Widget* self) {
    Button* btn = BUTTON(self);
    g_print("Drawing button '%s' at (%d,%d)\n", 
            button_get_label(btn), widget_get_x(self), widget_get_y(self));
}

static void button_class_init(ButtonClass* klass) {
    WidgetClass* widget_class = WIDGET_CLASS(klass);
    
    // 重写父类的虚函数
    widget_class->draw = button_draw;
    
    // Button特有的属性
    g_object_class_install_property(G_OBJECT_CLASS(klass), PROP_LABEL,
        g_param_spec_string("label", "Label", "Button label",
                           NULL, G_PARAM_READWRITE));
}

static void button_init(Button* self) {
    // Button特有的初始化
}

// TextEntry类实现
G_DEFINE_TYPE(TextEntry, text_entry, TYPE_WIDGET)

static void text_entry_draw(Widget* self) {
    TextEntry* entry = TEXT_ENTRY(self);
    g_print("Drawing text entry '%s' at (%d,%d)\n",
            text_entry_get_text(entry), widget_get_x(self), widget_get_y(self));
}

static void text_entry_class_init(TextEntryClass* klass) {
    WidgetClass* widget_class = WIDGET_CLASS(klass);
    
    // 重写虚函数
    widget_class->draw = text_entry_draw;
    
    // TextEntry特有属性
    g_object_class_install_property(G_OBJECT_CLASS(klass), PROP_TEXT,
        g_param_spec_string("text", "Text", "Entry text",
                           "", G_PARAM_READWRITE));
}

static void text_entry_init(TextEntry* self) {
    // TextEntry特有的初始化
}

/* =================================================================
   对比使用效果 - GObject的巨大优势
   ================================================================= */

// 使用传统方式的痛苦
void traditional_gui_example() {
    Button btn1 = {10, 10, 100, 30, "OK", NULL, NULL};
    Button btn2 = {120, 10, 100, 30, "Cancel", NULL, NULL};
    TextEntry entry = {10, 50, 200, 25, "Enter text...", 0, NULL, NULL};
    
    // 问题1：无法统一处理
    draw_button(&btn1);
    draw_button(&btn2);
    draw_text_entry(&entry);  // 不同的函数！
    
    // 问题2：无法放在同一个容器中统一管理
    // Button* widgets[] = {&btn1, &btn2, &entry}; // 编译错误！
    
    // 问题3：事件处理复杂
    handle_click_traditional(&btn1, "button");
    handle_click_traditional(&entry, "text_entry");
}

// 使用GObject的优雅方式
void gobject_gui_example() {
    // 创建不同类型的组件
    Widget* btn1 = g_object_new(TYPE_BUTTON, 
                               "x", 10, "y", 10, 
                               "width", 100, "height", 30,
                               "label", "OK", 
                               NULL);
                               
    Widget* btn2 = g_object_new(TYPE_BUTTON,
                               "x", 120, "y", 10,
                               "width", 100, "height", 30, 
                               "label", "Cancel",
                               NULL);
                               
    Widget* entry = g_object_new(TYPE_TEXT_ENTRY,
                                "x", 10, "y", 50,
                                "width", 200, "height", 25,
                                "text", "Enter text...",
                                NULL);
    
    // 优势1：统一处理！可以放在同一个数组中
    Widget* widgets[] = {btn1, btn2, entry};
    int widget_count = sizeof(widgets) / sizeof(widgets[0]);
    
    // 优势2：多态 - 统一的接口，不同的行为
    for (int i = 0; i < widget_count; i++) {
        widget_draw(widgets[i]);  // 同一个函数调用！
    }
    
    // 优势3：统一的事件处理
    for (int i = 0; i < widget_count; i++) {
        widget_handle_event(widgets[i], 1);  // 统一处理点击事件
    }
    
    // 优势4：信号系统 - 松耦合的事件处理
    g_signal_connect(btn1, "clicked", G_CALLBACK(on_ok_clicked), NULL);
    g_signal_connect(btn2, "clicked", G_CALLBACK(on_cancel_clicked), NULL);
    
    // 优势5：属性系统 - 统一的属性访问
    g_object_set(btn1, "visible", FALSE, NULL);  // 隐藏按钮
    
    // 优势6：引用计数自动内存管理
    g_object_unref(btn1);
    g_object_unref(btn2);
    g_object_unref(entry);
}

// 优势7：轻松扩展 - 创建新的组件类型
#define TYPE_IMAGE_BUTTON (image_button_get_type())
G_DECLARE_FINAL_TYPE(ImageButton, image_button, , IMAGE_BUTTON, Button)

// ImageButton继承Button的所有功能，只需要添加图片相关的特性
static void image_button_draw(Widget* self) {
    // 先调用父类的绘制
    WIDGET_CLASS(image_button_parent_class)->draw(self);
    
    // 再绘制图片
    ImageButton* img_btn = IMAGE_BUTTON(self);
    g_print("Drawing image: %s\n", image_button_get_icon_path(img_btn));
}

/* =================================================================
   实际使用中的强大威力 - GTK的例子
   ================================================================= */

void real_world_example() {
    // 这就是为什么GTK这样工作的原因：
    
    // 所有GTK组件都继承自GtkWidget
    GtkWidget* window = gtk_window_new(GTK_WINDOW_TOPLEVEL);
    GtkWidget* button = gtk_button_new_with_label("Click me");
    GtkWidget* entry = gtk_entry_new();
    GtkWidget* label = gtk_label_new("Hello World");
    
    // 统一的容器管理
    GtkWidget* box = gtk_box_new(GTK_ORIENTATION_VERTICAL, 5);
    gtk_container_add(GTK_CONTAINER(box), button);  // 同一个函数
    gtk_container_add(GTK_CONTAINER(box), entry);   // 处理不同类型
    gtk_container_add(GTK_CONTAINER(box), label);   // 的组件！
    
    // 统一的属性系统
    g_object_set(button, "sensitive", FALSE, NULL);
    g_object_set(window, "title", "My App", "default-width", 300, NULL);
    
    // 统一的信号系统
    g_signal_connect(button, "clicked", G_CALLBACK(on_button_clicked), NULL);
    g_signal_connect(window, "destroy", G_CALLBACK(gtk_main_quit), NULL);
    
    // 这种统一性和扩展性是传统C语言无法提供的！
}

/* =================================================================
   总结：GObject解决的核心问题
   ================================================================= */

/*
1. 代码重用问题：
   - 传统C：每个组件重复定义公共属性和方法
   - GObject：继承机制，一次定义，到处重用

2. 多态问题：
   - 传统C：需要switch-case或函数指针表，容易出错
   - GObject：虚函数机制，编译时类型安全

3. 内存管理：
   - 传统C：手动malloc/free，容易内存泄漏
   - GObject：引用计数，自动管理生命周期

4. 事件处理：
   - 传统C：回调函数指针，紧耦合
   - GObject：信号系统，松耦合，支持多个监听者

5. 配置和状态管理：
   - 传统C：每个结构体不同的字段访问方式
   - GObject：统一的属性系统，支持通知、验证等

6. 类型安全：
   - 传统C：void*到处飞，运行时才发现类型错误
   - GObject：编译时类型检查，运行时类型验证

7. 扩展性：
   - 传统C：修改现有代码才能添加功能
   - GObject：继承和接口，无需修改原代码
*/
```