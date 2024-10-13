---
title: "Build Echo Project With Cmake"
description: 
categories:
  - "DefaultCategory"
tags:
  - "Default-Tag"
date: 2024-10-13T02:26:15-07:00
slug: "build-echo-project-with-cmake"
math: 
license: 
hidden: false
draft: false
---


### Linux下CMake构建echo服务器项目

> 昨天测试CSAPP第十一章网络编程中的echo服务器时，想到顺便学着用CMake构建一下项目。

#### 关于CMake

> 这里来讲一下CMake是什么？以及我们为什么需要CMake？

#### CMake是什么？

在Windows下编程的时候，大概率会用到像Windows Visual Studio这样的集成开发环境（IDE），在这种IDE工作环境中，程序员只负责写程序，而像编译、链接这些工作是由IDE来自动完成的。

> 实际上，编译和链接是相当复杂的一些工作，各种依赖关系想想都令人头大，只是Windows下IDE帮我们承担了这项任务。

而Linux下是缺少这种IDE的。因为Linux下编程常常是服务端的编程，可能是在云主机上，只有命令行而没有GUI。

那Linux下编译、链接这些工作谁来做呢？

如果项目很小，源文件很少时，我们直接用编译命令就好了。比如

```shell
gcc -o hello helloworld.c
```

但是，一旦遇到大工程，可能有成百上千个源文件，这时候各个源文件之间、源文件与库之间的依赖关系会非常复杂，还是用编译命令来编译、链接的话，将会非常低效且易错。

这时候，make应运而生。没错，是make，还不是CMake。

使用make需要有一个Makefile文件，Makefile文件定义了工程的依赖关系、编译参数等，Makefile文件告诉make命令需要怎样的去编译和链接程序，并且make会智能地根据当前的文件修改的情况来确定哪些文件需要重编译，从而自己编译所需要的文件和链接目标程序。

但是对于一个大工程，编写Makefile也是件复杂的事，于是人们又想，为什么不设计一个工具，读入所有源文件之后，自动生成Makefile呢，于是就出现了CMake工具，它能够输出各种各样的makefile或者project文件,从而帮助程序员减轻负担。

> CMakeLists是需要程序员自己写的。

### 我们的项目

项目组织结构如下图，这是一个比较正规的工程组织结构。src文件夹存放项目源文件，如.c .cpp等；include文件夹存放头文件；bin文件夹用来存放最终生成的可执行文件；build文件夹用来存放构建项目时生产的中间文件。

示例项目是一个echo服务器。客户端发送字符串给服务端，服务端像回声一样再把同样的字符串发回客户端，并在服务端显示服务端接收了多少字节。

> caspp.h是CSAPP提供的一个头文件，csapp.c是其实现文件。
>
> echoclient.c是客户端的实现文件。
>
> echoserveri.c是服务端的实现文件。
>
> echo.c是用来显示接收字节数的函数实现文件。

所以该项目最终会生成两个可执行文件，一个客户端，一个服务端。

### CMake工作原理

一个刚才会有多个CMakeLists，项目最外层一个CMakeLists，然后每个存放源文件的文件夹里要有一个CMakeLists。

针对我们的echo项目，则一共要写两个CMakeLists文件：最外层需要写一个，称之为外层CMakeLists；src文件夹内需要写一个，称之为内层CMakeLists。

#### 外层CMakeLists

```cmake
cmake_minimum_required (VERSION 2.8)
project (MyEchoServer)
option (CLIENT "For building echoClient" ON)
set (EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
add_subdirectory (./src)
```

一共五条语句：

1. 第一条语句指明要求的CMake最低版本。

2. 第二条语句提供项目名称等信息。

3. 第三条是控制选项，这条比较重要。

   因为我们编译客户端的时候不需要echo.c和echoserveri.c这两个文件的参与。这里定义一个名为CLIENT的选项，用来给内层CMakeLists一个标识，如果这个选项是ON，就编译客户端。是OFF则编译服务端。

4. 第四条语句设置了可执行文件的输出目录。

   EXECUTABLE_OUTPUT_PATH是cmake内置的变量，意为可执行文件输出路径。

   PROJECT_SOURCE_DIR也是cmake内置的变量，意为工程的根目录。

   这条语句含义就是把可执行文件输出路径设置为工程根目录下的bin文件夹。

5. 第五条语句也很重要。

   add_subdirectory命令向当前工程添加存放源文件的子目录。src文件夹存放着项目源文件，所以这里传递src。

> 外层CMakeLists是总的CMakeLists，它控制着整个构建流程。它通过add_subdirectory来进入各个装有源文件的子目录，并执行该目录里的内层CMakeLists，直到处理完再出来继续处理下一个add_subdirectory。
>
> 我们这里只有一个源文件夹src，所以就只有一条add_subdirectory命令。

#### 内层CMakeLists

```cmake
set (CLIENT_SRC_LIST
	./csapp.c
	./echoclient.c)
set (SERVER_SRC_LIST
	./csapp.c
	./echo.c
	./echoserveri.c)
include_directories (../include)
find_package (Threads)
if (CLIENT)
	add_executable (echoclient ${CLIENT_SRC_LIST})
	target_link_libraries (echoclient ${CMAKE_THREAD_LIBS_INIT})
else()
	add_executable (echoserver ${SERVER_SRC_LIST})
	target_link_libraries (echoserver ${CMAKE_THREAD_LIBS_INIT})
endif()
set (EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
```

当外层CMakeLists通过add_subdirectory(./src)进入内层CMakeLists时，就开始执行内层CMakeLists，直到处理完才回到外层CMakeLists。

在内层CMakeLists，我们首先定义了CLIENT_SRC_LIST和SERVER_SRC_LIST两个变量，前者指明了编译客户端所需的源文件，后者则指定了编译服务端需要的源文件。

接下来的include_directories (../include)指明该项目所需头文件的路径。

在接下来就比较重要了，因为echo项目用到了多线程，所以编译时需要加上-lpthread这个选项，当我们用gcc直接编译时，应该这样：

```shell
gcc -o echoclient csapp.c echoclient.c -lpthread
```

但是，用CMake时，发生一些变化。我们需要用find_package(Threads)来寻找线程库。

接下来是if-else语句，如果CLIENT是ON的，则生成名为echoclient的客户端可执行文件，其依赖文件为CLIENT_SRC_LIST变量包含的源文件列表。否则，生成名为echoserver的服务端可执行文件，其依赖文件为SERVER_SRC_LIST变量包含的源文件列表。target_link_libraries命令负责将目标文件与库文件进行链接，这里的库文件就是Threads库。

最后一条语句就是设置可执行文件输出路径，前边已解释过。

#### 测试

加上这两个CMakelists之后，我们的项目组织结构图如下。

我们进入build文件夹进行构建测试。在build文件夹内输入以下命令

```shell
cmake ..
make
cmake .. -DCLIENT=OFF
make
```
