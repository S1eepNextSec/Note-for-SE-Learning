# CMake

## Start it Naively

### 最简单的CMake文件编写

```cmake
├── CMakeLists.txt
├── function.hpp
├── main.cpp
└── function.cpp

#指定cmake的最低版本
#cmake_minimum_required (VERSION <版本号>)
cmake_minimum_required (VERSION 3.0)

#声明工程最终编译后会生成一个可执行文件
#add_executable(<可执行文件名> <源文件> <源文件> ...)
#add_executable(<可执行文件名> <源文件>;<源文件>;...)
add_executable(app function.cpp main.cpp)
```

###  利用CMakeLists.txt生成Make文件

```shell
$cmake CMakeLists.txt
$make
```

```Shell
$tree
.
├── app
├── CMakeCache.txt
├── CMakeFiles
│   ├── 3.16.3
│   ├── 一大堆文件....
├── cmake_install.cmake
├── CMakeLists.txt
├── function.cpp
├── function.hpp
├── main.cpp
└── Makefile

8 directories, 36 files
#其中
#cmake <CMakeLists.txt文件路径> 命令
#生成目录&文件有 CMakeFiles | cmake_install.cmake | CMakeCache.txt
```



### 混乱的文件管理

* 按照上述方式简单的构建项目，CMake相关目录&文件、Make相关目录&文件扎堆在生成在项目的根目录下，在编译过程中同时生成大量的中间文件，根目录的文件管理变得很臃肿
* 灾难一般的文件管理

### 笨方法

* 创建一个build文件夹
* 在build文件夹中生成Make文件后进行编译

## 指定可执行文件生成位置

```cmake
#[预定义变量]
#EXECUTABLE_OUTPUT_PATH:最终生成的可执行文件所在的位置
#PROJECT_SOURCE_DIR:项目的根目录
#指定生成的可执行文件的路径
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR})
```

## 指定编译参数

### 添加编译选项

```cmake
add_compile_options(-Wall -O3 ...)
-Wall : 启用警告
-O3 : 启用优化
-std=C++xx : 指定标准
-l : 指定链接的库文件
```



## 罗列源文件太麻烦？

* 搜索全部

```cmake
#利用命令查找指定路径下的所有源文件
#将搜索到的源文件列表存储到一个变量中
#aux_source_directory(<指定搜索路径> <指定存储变量>)
aux_source_directory(${PROJECT_SOURCE_DIR}/src SRC)
#搜索根目录下的src目录 并将源文件列表存储到SRC变量中
```

* 按条件搜索

```cmake
#file(GLOB/GLOB_RECURSE <变量> <指定搜索路径&限定文件类型>)
#GLOB:仅搜索文件夹路径下的文件，不会搜索文件夹中的文件夹
#GLOB_RECURSE:递归的搜索，会搜索文件夹里的文件夹
file(GLOB SRC ${PROJECT_SOURCR_DIR}/src/*.cpp)
#仅搜索"项目根目录"-"src文件夹"-所有cpp文件，文件名都存储到SRC变量中
```

```cmake
#文件结构
├── CMakeLists.txt
├── build
└── src
    ├── function.cpp
    ├── function.hpp
    └── main.cpp
    
#CMakeLists.txt [version-1]
cmake_minimum_required (VERSION 3.0)
aux_source_directory(${PROJECT_SOURCE_DIR}/src SRC)
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR})
add_executable (app ${SRC})

#CMakeLists.txt [version-2]
cmake_minimum_required (VERSION 3.0)
file(GLOB SRC_CPP ${PROJECT_SOURCE_DIR}/src/*.cpp)
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR})
add_executable (app ${SRC_CPP})

```

## 头文件与源文件分离时

* 需要指出头文件的位置

```cmake
include_directories(<头文件位置>)
```

```cmake
#文件结构
├── build
├── CMakeLists.txt
├── include
│   └── function.hpp
└── src
    ├── function.cpp
    └── main.cpp
    
#CMakeLists.txt
cmake_minimum_required (VERSION 3.0)
aux_source_directory(${PROJECT_SOURCE_DIR}/src SRC)
include_directories(${PROJECT_SOURCE_DIR}/include)#指定头文件位置
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR})
add_executable (app ${SRC})

```

## 文件散落世界各地

### 字符串拼接

```cmake
set(src ${SRC_1} ${SRC_2} ${SRC_3} ...)
```

## 打印信息

```cmake
message(<参数> <消息>)
```

