# 【ChatServer】04 - 项目目录结构


## 目录

```bash
[xushun@localhost chat]$ tree .
.
├── bin
├── build
├── CMakeLists.txt
├── include
│   ├── client
│   └── server
├── src
│   ├── client
│   └── server
└── thirdparty
    └── json.hpp

9 directories, 2 files
```
- `bin`：可执行文件
- `build`：中间生成文件
- `CMakeLists.txt`：cmake管理项目
- `include`：头文件
- `src`：源文件
- `thirdparty`：第三方


## CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.0)
project(chat)

# 编译选项
set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS} -g)

# 可执行文件输出目录
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)

# 头文件搜索路径
include_directories(${PROJECT_SOURCE_DIR}/include)
include_directories(${PROJECT_SOURCE_DIR}/include/server)
include_directories(${PROJECT_SOURCE_DIR}/thirdparty)

# add src
aux_source_directory(src/server SERVER_SRC_LIST)
aux_source_directory(src/client CLIENT_SRC_LIST)

# 添加可执行文件
add_executable(chatserver ${SERVER_SRC_LIST})
# 依赖库文件
target_link_libraries(chatserver muduo_net muduo_base pthread)

```

可能还有修改。
