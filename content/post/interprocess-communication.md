---
title: "Interprocess Communication"
description: 
categories:
  - "Linux"
tags:
  - "Linux"
date: 2022-06-07T01:18:12-07:00
slug: "interprocess-communication"
math: 
license: 
hidden: false
draft: false
---

# 引言

进程间通信即InterProcess Communication，简称IPC。其目的是实现不同进程之间的通信问题。

进程间通信方式可以归纳为以下几类：

- 管道：PIPE、FIFO；
- 系统IPC：消息队列、共享存储；
- 信号（Signal）
- 套接字：socket。

# 管道

## PIPE

PIPE是UNIX系统最古老的IPC方式，也叫**未命名管道**，PIPE有以下两种局限性：

- PIPE是半双工的，即数据只能在一个方向上流动。
- PIPE只能在具有公共祖先的两个进程之间使用。通常，一个进程创建一个管道，然后进程调用fork之后，这个管道就可以在父进程和子进程之间使用了。

PIPE通过调用pipe函数来创建：

```c
#include <unistd.h>
int pipe(int fd[2]);
//返回值：成功则返回0；出错则返回-1。
```

- 参数fd[2]为一个长度为2的文件描述符数组，fd[0]是读出端，fd[1]是写入端。当函数成功返回，则自动维护了一个从fd[1]到fd[0]的数据通道。

通常，进程会先调用pipe，接着调用fork，从而创建从父进程到子进程的IPC通道。如下图所示：

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/interprocess-communicatiom/after-fork.png)

fork之后做什么取决于我们想要的数据流的方向：

- 如果想要从父进程到子进程流动的PIPE，则关闭父进程的读端（fd[0]）和子进程的写端（fd[1]）,下图展示了这种情况；
- 如果想要从子进程到父进程流动的PIPE，则关闭父进程的写端（fd[1]）和子进程的读端（fd[0]）。

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/interprocess-communicatiom/from-father-to-son.png)

> PIPE的一端被关闭后，下列两条规则起作用：
>
> - 对写端关闭的PIPE，在所有数据都被读取（read）后，read返回0，表示文件结束；
> - 对读端关闭的PIPE，write时将产生SIGPIPE信号，如果忽略该信号或者捕捉该信号并从其处理程序返回，则write返回-1，errno设置为EPIPE。
>
> 以上图从父进程到子进程的管道为例：
>
> - 如果父进程已经关闭了fd[1]，那么在子进程读取完PIPE内所有数据后，read会返回0，表示文件结束；
> - 如果子进程已经关闭了fd[0]，父进程还在write时，会产生SIGPIPE信号。

因为上述创建数据通道的流程（首先要调用pipe创建PIPE，然后调用fork创建子进程，最后还要根据通道方向关闭父/子进程的读/写端）过于繁杂，所以标准I/O库提供了两个函数：popen和pclose，这两个函数实现的操作是：

- popen：创建一个PIPE，fork一个子进程，关闭未使用的管道端，执行一个shell命令；
- pclose：关闭由popen所建立的管道及文件指针；

```c
#include <stdio.h>
FILE *popen(const char *cmdstring, const char *type);
//返回值：若成功，返回文件指针；若出错，返回NULL
int pclose(FILE *fp);
//返回值：若成功，返回cmdstring的终止状态；若出错，返回-1
```

函数popen先执行fork，然后调用exec执行cmdstring，并且返回一个标准I/O文件指针。

> cmdstring由Bourne shell以下列方式执行：
>
> ```sh
> sh -c cmdstring
> ```

- 如果type是“r”，则文件指针连接到cmdstring的标准输出；

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/interprocess-communicatiom/r.png)

- 如果type是“w”，则文件指针连接到cmdstring的标准输入。

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/interprocess-communicatiom/w.png)

## FIFO

FIFO也叫**命名管道**。相比于PIPE，FIFO的最大优势在于：它在两个不相关的进程间也能交换数据。

FIFO是一种文件类型。FIFO的路径名存在于文件系统中。创建FIFO类似于创建文件：

```c
#include <sys/stat.h>
int mkfifo(const char *path, mode_t mode);
int mkfifoat(int fd, const char *path, mode_t mode);
//两个函数返回值：成功返回0；出错返回-1
```

- 若path为绝对路径，则fd参数被忽略，mkfifoat和mkfifo类似；
- 若path为相对路径，则fd参数是一个打开目录的有效文件描述符，路径名和目录有关；
- 若path为相对路径，且fd参数有一个特殊值AF_FDCWD，则路径名以当前目录开始，mkfifoat和mkfifo类似。

> mode参数用于指定新建的FIFO文件的访问权限；

与PIPE类似：

- 若write一个尚无进程为读而打开的FIFO，则产生SIGPIPE信号；
- 若FIFO的最后一个写进程关闭了该FIFO，则将为该FIFO的读进程产生一个文件结束标志。

# XSI IPC

> 《APUE》中提到了三种XSI IPC：消息队列、信号量、共享存储。

这里首先要说明一些东西：信号（Signal）和信号量（Semaphore）是两个不同的东西：

- 信号（Signal）是软件中断，由用户、系统或者进程发送给目标进程的信息，以通知目标进程某个状态的改变或系统异常；
- 信号量（Semaphore）是一种特殊变量，用于协调进程对共享资源的访问。

也就是说信号（Signal）是用来传递信息的，尽管它传递的信息量非常小，小到只有一个数字，但它确实在传递消息，所以信号（Signal）可以算是一种进程间通信方式；而信号量（Semaphore）是用来解决多进程或多线程对共享资源的互斥访问问题的，它应该属于进/线程同步机制。所以我会把信号量（Semaphore）放在下一篇讲进/线程同步问题的时候介绍，本篇只介绍XSI IPC里的消息队列和共享内存。

> 不过，《APUE》既然提到了三种XSI IPC：消息队列、信号量、共享存储。也就是说信号量同消息队列还有共享存储出自同一家标准，所以它们还是有些联系的。下面简单说一下。

每个IPC对象（消息队列、信号量、共享存储）都有两个名字：标识符和键。其中标识符是内部名，键是外部名。

- **标识符**不同于文件描述符，它不是小整数，而是连续加1，直到达到最大值之后又回转到0；
- 创建IPC对象时需要指定**键**，其类型为key_t，通常在头文件<sys/types.h>中被定义为长整型，内核负责把键变换成标识符。

键的指定有三种方式：

- IPC_PRIVATE；
- 自己定义；
- 用ftok函数。

XSI IPC为每一个IPC结构关联了一个权限结构体ipc_perm，该结构规定了权限和所有者，创建IPC对象后，可以通过***ctl函数来修改这个结构体里某些字段的值。

> XSI IPC的缺点：
>
> - IPC结构没有引用计数，终止之后也不会删除，需要通过调用msgctl或者执行命令ipcrm来删除；
> - IPC结构不在文件系统中，也没有文件描述符，所以不能用ls来查看，也不能对它们用select/poll等IO多路复用函数。

## 消息队列

消息队列是消息的链表，存储在内核中，由消息队列标识符标识。

```c
#include <sys/msg.h>
int msgget(key_t key, int flag);//成功返回消息队列ID；出错返回-1
int msgctl(int msqid, int cmd, struct msqid_ds *buf);//成功返回0；出错返回-1
int msgsnd(int msqid, const void *ptr, size_t nbytes, int flag);//成功返回0；出错返回-1
ssize_t msgrcv(int msqid, void *ptr, size_t nbytes, long type, int flag);//成功返回消息数据部分的长度；出错返回-1
```

- msgget函数用于创建一个新队列或打开一个现有队列，其中flag参数用于设置权限（ipc_perm中的mode成员）；
- msgctl函数用于修改ipc_perm结构的成员值，也用于删除消息队列以及仍在该队列中的所有数据；
- msgsnd函数用于讲新消息添加到队列尾端；
- msgrcv函数用于从队列中取消息。

> 并不一定要以先进先出顺序取消息，也可以按照消息的类型（type）字段取消息。

## 共享存储

共享存储允许两个或多个进程共享一个给定的存储区。因为数据不需要在进程之间复制，所以这是最快的一种IPC。

> 共享存储有两种：
>
> - 内存文件映射：即通过mmap函数，将同一个文件映射到多个进程的虚拟地址空间；
> - XSI共享存储：与内存文件映射相比，它没有相关文件，它的共享存储段是内存的匿名段。

### XSI 共享存储

```c
#include <sys/shm.h>
int shmget(key_t key, size_t size, int flag);//成功返回共享存储ID；出错返回-1
int shmctl(int shmid, int cmd, struct shmid_ds *buf);//成功返回0；出错返回-1
void *shmat(int shmid, const void *addr, int flag);//成功返回指向共享存储段的指针；出错返回-1
int shmdt(const void *addr);//成功返回0；出错返回-1
```

- shmget函数用于获得一个共享存储标识符，size为共享存储段的长度（以字节为单位），flag参数设置权限；
- shmctl函数用于修改ipc_perm结构的成员值，也用于删除共享存储以及仍在该共享存储中的所有数据；
- shmat函数用于将共享存储段与进程地址空间连接起来，addr参数用于指定连接地址，一般为0，指连接地址由系统决定；
- shmdt函数用于在对共享存储段的操作结束之后，将进程地址空间与该共享存储段分离，注意，仅分离，并不删除。

### 内存文件映射

```c
#include <sys/mman.h>
void *mmap(void *addr, size_t len, int prot, int flag, int fd, off_t off);
//成功返回映射区起始地址；出错返回MAP_FAILED
```

- addr参数用于指定映射区的起始地址，一般为0，指起始地址由系统决定；
- len参数为映射的字节数；
- fd参数为要映射的文件的文件描述符；
- off参数为要映射字节在文件中的起始偏移量；
- prot参数指定映射区的保护要求；
- flag参数影响映射区的多种属性，不详细展开。

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/interprocess-communicatiom/mmap.png)

# 信号

信号是软件中断。它只能用于通知进程某事件的发生，并不能传输数据。

通过kill -l命令可以看到目前Linux支持的全部信号，已经达到了62种（32号和33号没有）。

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/interprocess-communicatiom/xinhaonum.png)

每个信号的名字都是以SIG开头，这些信号名在头文件<signal.h>中被定义为正整数常量（信号编号）。

在某个信号出现时，可以告诉内核按下列三种方式之一处理：

- 忽略该信号；
- 捕捉信号。为了做到这一点，要通知内核在某种信号发生时，调用一个用户函数。在用户函数中，可执行用户希望对这种事件进行的处理；
- 执行系统默认动作。大多数信号的系统默认动作是终止该进程；

UNIX系统信号机制最简单的接口是signal函数：

```c
#include <signal.h>
void (*signal(int signo, void (*func) (int))) (int);
//成功返回以前的信号处理配置；出错返回SIG_ERR
```

signo是我们要处理的信号的信号名；func是我们对信号signo要进行的处理方式，有以下三种：

- 若为SIG_IGN则向内核表示忽略该信号（SIGKILL和SIGSTOP不能忽略）；
- 若为SIG_DFL则向内核表示接到此信号后的动作是系统默认动作；
- 若为函数地址，则在信号发生时，调用该函数。该函数则叫做信号处理程序。

**signal函数现在已基本被sigaction函数替代**：

```c
#include <signal.h>
int sigaction(int signo, 
              const struct sigaction *restrict act,
              struct sigaction *restrict oact);
//成功返回0；出错返回-1
```

sigaction函数的功能是检查或修改与指定信号相关联的处理动作：

- signo参数是要检测或修改其具体动作的信号编号；
- act用于指定新的信号处理方式；
- oact若不为NULL，则通过它返回之前的信号处理方式。

> sigaction结构体里的sa_handler成员指定信号处理方式。详见《APUE》。

# 套接字

## 套接字描述符

套接字实现通过网络相连的不同计算机之间的进程间通信。在UNIX系统中，套接字也是一种文件类型，故它也有自己的文件描述符，叫做套接字描述符。

```c
#include <sys/socket.h>
int socket(int domain, int type, int protocol);
//成功返回套接字（文件）描述符；出错返回-1
```

domain参数确定通信的特性，其常见取值为:

- AF_INET：代表IPv4；
- AF_INET6：代表IPv6。

type参数确定套接字类型，进一步确定通信特征，常见取值为：

- SOCK_DGRAM：固定长度、无连接、不可靠报文传递；
- SOCK_STREAM：有序、可靠、双向、面向连接的字节流。

protocol参数通常为0，表示为给定的domain和type选择默认协议：

- domain=AF_INET，type=SOCK_STREAM时，默认协议为TCP；
- domain=AF_INET，type=SOCK_DGRAM时，默认协议为UDP。

> 尽管套接字描述符本质上是文件描述符，但并非所有参数为文件描述符的函数都可以接受套接字描述符。如lseek就不能，因为套接字没有文件偏移量的概念。

## 寻址及关联

我们想跟另一台主机上的进程通信，首先要能够标识我们的目标通信进程，需要两个信息：

- IP地址：锁定要通信的计算机；
- 端口号：锁定要通信的进程。

> 可以称（IP，Port）为套接字地址。

但是有这样一个CPU架构特性——字节序，字节序分两种：

- 大端字节序：高位存在低地址，低位存在高地址；
- 小端字节序：低位存在低地址，高位存在高地址。

不同的计算机使用的CPU不同，那主机字节序就有可能不同，这个问题必须解决，要不然我这边是0x01020304，传到那边成0x04030201了，这是绝对不可以的。解决方案：

- 由网络协议指定一种字节序，称为网络字节序。通信双方收到数据后再转成自己的字节序。

> TCP/IP协议栈采用大端字节序。

UNIX提供了四个函数，用于完成主机字节序和网络字节序的相互转换：

```c
#include <arpa/inet.h>
uint32_t htonl(uint32_t hostint32);//32位整数的主机字节序到网络字节序的转换
uint16_t htons(uint16_t hostint16);//16位整数的主机字节序到网络字节序的转换
uint32_t ntohl(uint32_t netint32);//32位整数的网络字节序到主机字节序的转换
uint16_t ntohs(uint16_t netint16);//16位整数的网络字节序到主机字节序的转换
```

在IPv4因特网中，套接字地址用结构体sockaddr_in表示：

```c
struct in_addr{
	in_addr_t s_addr; //IPv4地址
};

struct sockaddr_in{
	sa_family_t    sin_family; //协议族
	in_port_t      sin_port;  //端口号
	struct in_addr sin_addr;  //IPv4地址
}
```

IP地址我们经常用点分十进制表示，但是计算机只能理解二进制格式，所以还有两个用来转换点分十进制和二进制地址的函数：

```c
#include <arpa/inet.h>
const char *inet_ntop(int domain,
                      const void *restrict addr,
                      char *restrict str,
                      socklen_t size);
//返回值：成功则返回点分十进制地址字符串的指针；出错返回NULL
int inet_pton(int domain,
              const char *restrict str,
              void *restrict addr);
//返回值：成功则返回1；若格式无效则返回0；出错返回-1
```

- inet_ntop函数将网络字节序二进制地址转换为点分十进制；
- inet_pton函数将点分十进制转换为网络字节序的二进制地址。

客户端程序的套接字由系统选择默认端口即可；而对于服务端程序，我们需要把其套接字关联到一个固定的套接字地址上，这个过程由bind函数完成：

```c
#include <sys/socket.h>
int bind(int sockfd, const struct sockaddr *addr, socklen_t len);
//返回值：成功返回0；出错返回-1
```

- sockfd参数为要关联地址的套接字描述符（socket函数的返回值）；
- addr参数为套接字地址结构体，对sockaddr_in结构体进行类型转换即可；
- len参数为addr变量的大小，可由sizeof()得出。

## 建立连接

对于面向连接的套接字类型（如SOCK_STREAM），在开始交换数据之前，需要在客户端和服务端之间建立连接，发起连接的一般为客户端，这由connect函数完成：

```c
#include <sys/socket.h>
int connect(int sockfd, const struct sockaddr *addr, socklen_t len);
//成功返回0；出错返回-1
```

- sock参数是要发起连接的套接字描述符（即客户端的套接字描述符）；
- addr参数是服务端的套接字（即对方的套接字地址结构）；
- len参数addr变量的大小，可由sizeof()得出。

> 因为我们通常不会用bind函数给客户端的套接字绑定一个地址（端口号），所以connect函数会给客户端套接字绑定一个默认地址，即客户端进程的端口号在这时产生。

服务器调用listen函数来宣告它愿意接受连接请求：

```c
#include <sys/socket.h>
int listen(int sockfd, int backlog);
//返回值：成功返回0；出错返回-1
```

- sockfd参数为服务端套接字描述符；
- backlog参数规定了内核应该为该套接字排队的最大连接个数。

> backlog参数牵扯到TCP连接的内容，以后细讲。

调用listen之后，服务端套接字就能接收连接请求，使用accept函数获得连接请求并建立连接：

```c
#include <sys/socket.h>
int accept(int sockfd, struct sockaddr *restrict addr, socklen *restrict len);
//成功返回新的套接字描述符；出错返回-1
```

- sockfd参数为服务端套接字描述符；
- 如果不关心客户端的身份，addr和len可以均设为NULL。

> accept函数返回一个新的套接字描述符，这个新的套接字与客户端套接字连接在一起，并用来交换数据；
>
> 这个新的套接字与原始套接字（sockfd）具有相同的套接字类型和地址族；
>
> 原始套接字继续保持可用状态并接收其他连接请求。

## 时序图

一图胜千言，下面是面向连接的TCP socket通信时序图：

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/interprocess-communicatiom/princple.png)

# 总结

> 《APUE》中明确指明，要尽量避免使用消息队列及信号量（Semaphore）。

**为什么共享存储是最快的IPC方式？**

使用PIPE、FIFO、消息队列如下图所示：

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/interprocess-communicatiom/IPC-slow.png)

使用PIPE、FIFO、消息队列从一个文件传输信息到另外一个文件需要复制4次：

1. 服务端将信息从相应的文件复制到服务端临时缓冲区中；
2. 从服务端临时缓冲区中复制到PIPE（FIFO/消息队列）；
3. 客户端将信息从PIPE（FIFO/消息队列）复制到客户端临时缓冲区中；
4. 从客户端临时缓冲区将信息复制到输出文件中。

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/interprocess-communicatiom/shared-memory-fast.png)

共享内存的消息复制只有两次：

1. 从输入文件到共享内存；
2. 从共享内存到输出文件。

PIPE、FIFO、消息队列都属于间接通信，它们要经过内核；而共享存储属于直接通信，共享存储段分配在虚拟地址空间中的用户那一部分，客户端和服务端共享，不经过内核而直接读写，所以快。

> 共享存储的缺点就是需要进行同步操作。