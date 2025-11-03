
## experssion
###  include(application_settings)
- **`include()`**: CMake 的内置命令，用于包含其他 CMake 脚本文件
- **`application_settings`**: 要包含的文件名（不带 `.cmake` 扩展名）
```cmake
include(application_settings)
# 搜索顺序：
# 1. ${CMAKE_CURRENT_SOURCE_DIR}/application_settings.cmake
# 2. ${CMAKE_CURRENT_BINARY_DIR}/application_settings.cmake  
# 3. CMake 模块路径（如 /usr/share/cmake/Modules/）
# 4. CMAKE_MODULE_PATH 变量指定的路径
```

### file(GLOB project_modules "/workspace/projects/\*")
-  **`file(GLOB ...)`**: 文件通配命令
```cmake
# 如果目录结构为：
# /workspace/projects/
#   ├── module_a/
#   ├── module_b/
#   └── config.txt

# project_modules 将包含：
# /workspace/projects/module_a
# /workspace/projects/module_b  
# /workspace/projects/config.txt
```

### 变量取值优先级
在 CMake 中，**命令行通过 `-D` 设置的值会覆盖 `.cmake` 脚本中设置的值**。
```txt
命令行 -D 参数 → 最高优先级
    ↓
CMakeCache.txt 缓存值 → 中等优先级  
    ↓
.cmake 脚本中的 set() → 最低优先级
    ↓
CMake 默认值 → 最低优先级
```

#### 值类型
```cmake
# 普通变量（会被覆盖）
set(MY_VAR "value")

# 缓存变量（有默认值，可被命令行覆盖）
set(MY_CACHE_VAR "default" CACHE STRING "Description")

# 强制缓存变量（不能被覆盖）
set(MY_FORCED_VAR "fixed" CACHE STRING "Description" FORCE)
```

### debug
- 会打印出查包细节
```cmake
-DCMAKE_FIND_DEBUG_MODE=ON
```