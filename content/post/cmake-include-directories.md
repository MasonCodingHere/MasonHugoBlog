---
title: "Cmake Include Directories"
description: 
categories:
  - "DefaultCategory"
tags:
  - "Default-Tag"
date: 2024-10-13T02:23:23-07:00
slug: "cmake-include-directories"
math: 
license: 
hidden: false
draft: false
---

#### 命令格式

```cmake
include_directories ([AFTER|BEFORE] [SYSTEM] dir1 [dir2 ...])
```

将指定目录添加到编译器的头文件搜索路径之下，指定的目录被解释成当前源码路径的相对路径。

#### 命令解析

默认情况下，`include_directories`命令会将目录添加到列表最后，可以通过命令设置`CMAKE_INCLUDE_DIRECTORIES_BEFORE`变量为`ON`来改变它默认行为，将目录添加到列表前面。也可以在每次调用`include_directories`命令时使用`AFTER`或`BEFORE`选项来指定是添加到列表的前面或者后面。如果使用`SYSTEM`选项，会把指定目录当成系统的搜索目录。该命令作用范围只在当前的CMakeLists.txt。

#### `include_directories`命令的基本行为

```cmake
#CMakeLists.txt
cmake_minimum_required(VERSION 3.18.2)
project(include_directories_test)

include_directories(sub) 
include_directories(sub2) #默认将sub2添加到列表最后
include_directories(BEFORE sub3) #可以临时改变行为，添加到列表最前面

get_property(dirs DIRECTORY ${CMAKE_SOURCE_DIR} PROPERTY INCLUDE_DIRECTORIES)
message(">>> include_dirs=${dirs}") #打印一下目录情况

set(CMAKE_INCLUDE_DIRECTORIES_BEFORE ON) #改变默认行为，默认添加到列表前面
include_directories(sub4)
include_directories(AFTER sub5) #可以临时改变行为，添加到列表的最后
get_property(dirs DIRECTORY ${CMAKE_SOURCE_DIR} PROPERTY INCLUDE_DIRECTORIES)
message(">>> SET DEFAULT TO BEFORE, include_dirs=${dirs}")
```

```cmake
#输出
>>> include_dirs=/XXX/XXX/sub3;/XXX/XXX/sub;/XXX/XXX/sub2
>>> SET DEFAULT TO BEFORE, include_dirs=/XXX/XXX/sub4;/XXX/XXX/sub3;/XXX/XXX/sub;/XXX/XXX/sub2;/XXX/XXX/sub5
```

#### 结合实际说明用法

创建的文件和目录结构及说明如下：

```cmake
├── CMakeLists.txt    #最外层的CMakeList.txt
├── main.cpp    #源文件，包含被测试的头文件
├── sub    #子目录
 └── test.h    #测试头文件，是个空文件，被外层的main,cpp包含
```

##### **场景1**：不使用`include_directories`包含子目录`sub`,直接在`main.cpp`里面包含`"test.h"`。

```cmake
# CMakeList.txt
cmake_minimum_required(VERSION 3.18.2)
project(include_directories_test)
add_executable(test main.cpp)
```

```cmake
//main.cpp
#include "test.h"
#include <stdio.h>
int main(int argc, char **argv)
{
    printf("hello, world!\n");
    return 0;
}
```

执行`cmake --build .`，会提示找不到头文件的错误:

```cmake
fatal error: 'test.h' file not found 
#include "test.h"
```

##### **场景2**：使用`include_directories`包含子目录`sub`,并在`main.cpp`里面包含`"test.h"`。

```cmake
# CMakeList.txt
cmake_minimum_required(VERSION 3.18.2)
project(include_directories_test)
include_directories(sub) #与上个场景不同的地方在于此处
add_executable(test main.cpp)
```

```cmake
//main.cpp
#include "test.h"
#include <stdio.h>
int main(int argc, char **argv)
{
    printf("hello, world!\n");
    return 0;
}
```

执行`cmake --build .`，会生成可执行文件`test`，使用`./test`执行后会输出打印`hello, world!`。

> 当然，不使用`include_directories(sub)`，在`main.cpp`中直接使用`#include "sub/test.h"`也是可以的。
