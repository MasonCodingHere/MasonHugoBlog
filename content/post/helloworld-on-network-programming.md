---
title: "网络编程领域的HelloWorld程序"
description: 
categories:
  - "Linux"
tags:
  - "Socket"
  - "C++"
date: 2021-05-14T06:44:38-07:00
slug: "helloworld-on-network-programming"
math: 
license: 
hidden: false
draft: false
---

# 引言

Echo客户端/服务端程序应该是网络编程领域的入门首选，可以视为网络编程领域的HelloWorld程序。

为了深入学习网络编程，我写了这样一个程序，姑且叫它Simplest_Socket。这确实是最简单的socket通信程序。与一般的Echo服务器不同，Simplest_Socket会把客户端传来的英文字符串转换为大写再返回给客户端；而不像Echo服务器那样原样返回。

这样设计的目的在于体现服务器的“服务”功能，尽管只是把小写转为大写，但这确实是一项服务。

# 思路

TCP套接字通信由一个四元组确定一个端到端通信，即：

**（客户端IP地址，客户端端口号，服务端IP地址，服务端端口号）**

整体的时序图如下：

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/helloworld-on-network-programming/princeple-of-comm.png)

> 上图中数据传输采用了read/write函数，套接字是一种文件类型，所以用这两个文件IO来读写套接字描述符也可以；但是套接字与普通文件又有些不同，在<sys/socket.h>中声明了专门用于socket的IO函数：send函数和recv函数。Simplest_Socket采用这两个函数来读写套接字。

## 客户端

### 指定服务端套接字地址结构

客户端作为连接的**主动**发起方，它需要知道它要连到哪个服务器的哪个端口，所以在客户端程序中，首先要定义服务端的套接字地址结构，在**IPv4因特网**中，套接字地址结构由结构体`sockaddr_in`定义，它声明在头文件`<arpa/inet.h>`中。在这个结构体中：

- sin_family用于指定使用的网络层协议IPv4，取值为AF_INET；
- in_port_t被定义为uint16_t，sin_port指定端口号，由于是无符号16位整型，所以其取值范围为0~65535；
- sin_addr用于指定IPv4地址，它也是一个结构体in_addr，in_addr_t被定义为uint32_t，即无符号32位整型，正好与IPv4地址的大小对应。

```c
#include <arpa/inet.h>
struct in_addr{
	in_addr_t s_addr; //IPv4地址
};
struct sockaddr_in{
	sa_family_t    sin_family; //协议族
	in_port_t      sin_port;  //端口号
	struct in_addr sin_addr;  //IPv4地址
}
```

我们所定义的存放IP地址和端口号的变量均已**主机字节序**存放在我们的本地主机上，当进行网络通信时，需要将它们转化为**网络字节序**，这些工作由头文件`<arpa/inet.h>`中声明的htons函数和inet_pton函数来完成：

- htons函数用于把将无符号16位整型数据由主机字节序转为网络字节序，端口号正好适用；
- inet_pton函数用于将主机字节序的点分十进制IP地址转换为网络字节序的二进制IP地址，IPv4地址和IPv6地址均可用该函数。

```c
#include <arpa/inet.h>
uint16_t htons(uint16_t hostint16);
int inet_pton(int domain,
              const char *restrict str,
              void *restrict addr);
```

所以对于我们的客户端，我们可以定义如下套接字地址结构：

```c
#include <arpa/inet.h>
#include <string.h>
const char* server_ip = "127.0.0.1"; //服务端IP地址
const uint16_t SERVER_PORT = 2021; //服务端监听端口号

struct sockaddr_in server_sockaddr;
memset(&server_sockaddr, 0, sizeof(server_sockaddr));
server_sockaddr.sin_family = AF_INET; //指定IPv4
server_sockaddr.sin_port = htons(SERVER_PORT);
inet_pton(AF_INET, server_ip, &server_sockaddr.sin_addr);
```

> 声明在<string.h>中的memset函数可以用于初始化新申请的空间，将其置为指定值。

### 创建套接字

既然要使用套接字，当然第一件事就是调用socket函数创建套接字。

```c
#include <sys/socket.h>
int socket(int family, int type, int protocol);
//成功返回非负套接字描述符，出错返回-1
```

对于我们的客户端：

- family：设置协议域为AF_INET，即IPv4；
- type：设置套接字类型为SOCK_STREAM，即字节流套接字；
- protocol：设置该参数为0，表示选择根据family和type组合系统提供的默认传输层协议。对于AF_INET和SOCK_STREAM组合，默认协议为TCP。

使用perror函数将错误原因输出到**标准错误**（stderr）：

```c
#include <stdio.h>
void perror(const char *str);
//输出格式为"str:错误原因",错误原因依照全局变量errno的值来决定要输出的字符串
```

所以在客户端中这样创建客户端的套接字：

```c
#include <sys/socket.h>
#include <stdio.h>

int client_fd = socket(AF_INET, SOCK_STREAM, 0); //指定TCP协议
if(client_fd < 0){
	perror("socket");
	exit(1);
}
```

### 发起连接

创建完套接字之后，作为主动方的客户端要做的就是发起连接，这由connect函数完成：

```c
#include <sys/socket.h>
int connect(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen);
//成功返回0；出错返回-1
```

- sockfd参数即客户端socket函数返回的套接字描述符；
- servaddr参数指向一个指明了服务端IP地址和端口号的套接字地址结构，即我们之前创建的server_sockaddr；
- addrlen参数是servaddr参数指向的地址结构的大小，可由sizeof()运算得到。

所以，在客户端中这样发起连接：

```c
#include <sys/socket.h>
#include <stdio.h>

if(connect(client_fd, (struct sockaddr*) &server_sockaddr, sizeof(server_sockaddr)) < 0){
	perror("connect");
	exit(1);
}
```

> 调用connect函数将激发TCP的三路握手过程，详情以后分析TCP状态机的时候再讲。

### 数据传输

客户端发起连接之后，服务端接收连接，双方完成TCP的三路握手之后，通信链路就建立起来了，客户端可以传输数据了。

在传输数据之前，我们定义了一个发送缓冲区sendbuf和一个接收缓冲区recvbuf。

对于Simplest_Socket来说，客户端传输的数据就是用户输入的英文字符串，所以while函数的条件我们设置为fgets函数，fgets函数是一个声明在<stdio.h>的标准IO库函数：

```c
#include <stdio.h>
char *fgets(char *restrict buf, int n, FILE *restrict fp);
//成功返回buf；若已到达文件尾或出错返回NULL
```

- fgets函数从指定的流fp读取字符送到长度为n的缓冲区buf，一直读到下一个换行符为止，但不会超过n-1个字符。

> fgets函数：缓冲区buf以null字节结尾。如果该行（包括换行符）的字符数超过n-1，则fgets只返回一个不完整的行，但是buf总是以null结尾。对fgets的下一次调用会继续读该行。

我们从标准输入（stdin）读取用户输入的数据到sendbuf。然后就可以使用send函数把sendbuf中的数据通过客户端的套接字传输给服务端，

```c
#include <sys/socket.h>
ssize_t send(int sockfd, const void *buf, size_t nbytes, int flags);
//成功返回发送的字节数；出错返回-1
```

- send函数将buf中的nbytes个字节的数据通过套接字描述符sockfd发送给服务端，flags参数一般置0。

我们可以用strlen函数获取sendbuf中实际数据的长度：

```c
size_t strlen(const char* str);
```

- strlen函数从字符串的开头位置依次向后计数，直到遇见`\0`，然后返回计时器的值。最终统计的字符串长度不包括`\0`。

我们定义如果用户输入的是`Q\n`，表示退出。这个用strcmp函数来完成：

```c
#include <string.h>
int strcmp(const char *str1, const char *str2)
```

- strcmp函数返回值为0则表示参与比较的两个字符串相等。

发送完数据之后要做的就是要接收服务端返回的数据，这由recv函数来完成：

```c
#include <sys/socket.h>
ssize_t recv(int sockfd, void *buf, size_t nbytes, int flags);
//返回值：返回数据的字节长度；若无可用数据或对等方已经按序结束，返回0；出错返回-1
```

- recv函数通过套接字描述符sockfd接收数据至buf，nbytes参数指定buf的大小，flags参数一般置0。

接收完数据后，通过fputs函数把recvbuf的内容输出到标准输出（stdout）。

```c
#include <stdio.h>
int fputs(const char *restrict str, FILE *restrict fp);
```

- fputs函数将一个以null字节终止的字符串str写到指定的流fp，尾端的终止符null不写出。

在准备发送和接收下一次的新数据之前，我们用memset函数将sendbuf和recvbuf的空间全部置0：

```c
#include <string.h>
void *memset(void *str, int c, size_t n)
```

- memset函数将参数 str 所指向的字符串的前 n 个字符置为值c。

所以在客户端这样写数据传输部分：

```c
#include <stdio.h>
#include <sys/socket.h>
#include <string.h>

const int BUFFER_SIZE = 1024; //定义缓冲区大小

char sendbuf[BUFFER_SIZE];
char recvbuf[BUFFER_SIZE];
while(fgets(sendbuf, sizeof(sendbuf), stdin)){ //从stdin读入待传输数据至sendbuf
	send(client_fd, sendbuf, strlen(sendbuf), 0); //将sendbuf中的数据通过client_fd套接字传输
    if(strcmp(sendbuf, "Q\n") == 0) //输入Q表示退出
		break;
	recv(client_fd, recvbuf, sizeof(recvbuf), 0); //从client_fd套接字接收数据，保存至recvbuf
	fputs(recvbuf, stdout);//将recvbuf中的数据输出至stdout
	memset(sendbuf, 0, sizeof(sendbuf));//将sendbuf的内存值置0
	memset(recvbuf, 0, sizeof(recvbuf));//将recvbuf的内存值置0
}
```

### 关闭连接

当用户输入`Q\n`，表示要退出连接。

因为套接字也是一种文件类型，所以我们可以像关闭普通的文件描述符一样，用close函数来关闭它。

```c
#include <unistd.h>
int close(int fd);
//成功返回0；出错返回-1
```

所以在客户端这样写关闭连接部分：

```c
#include <unistd.h>

close(client_fd);
```

## 服务端

服务端部分内容与客户端一致，重复部分不再赘述。

### 定义服务端套接字地址结构

这一部分与客户端相同，也是定义服务端的套接字地址结构。

### 创建监听套接字

服务端作为被动接收连接的一方，需要创建一个监听套接字，这个也跟客户端创建套接字一样。

### 绑定

对于服务端，我们一般还会指定一个固定的端口号，并且这个端口号还应该让想用这个服务器的客户端知道，也就是服务端的监听套接字要绑定一个固定的套接字地址结构，这样客户端在想要连接到这个服务端时，才可以知道我应该连接到哪个套接字地址结构，如果你的端口号一直变，那客户端就比较难受了。

这个绑定工作由bind函数完成。

```c
#include <sys/socket.h>
int bind(int sockfd, const struct sockaddr *myaddr, socklen_t addrlen);
//成功返回0；出错返回-1
```

- bind函数将套接字sockfd绑定到套接字地址结构myaddr，addrlen为myaddr的长度，可由sizeof()得到。

### 转化为被动套接字

由socket函数创建的套接字是一个主动套接字，即它是一个会调用connect函数发起连接的客户端套接字。

那问题就出来了，我服务端的套接字可不是要主动发起连接的，而是要被动接受连接的。那么就需要把socket函数创建的主动套接字转化为被动套接字，这个工作由listen函数来完成：

```c
#include <sys/socket.h>
int listen(int sockfd, int backlog);
//成功返回0；出错返回-1
```

- listen函数把主动套接字sockfd转化为被动套接字，指示内核应该接受指向该套接字的连接请求。
- backlog参数规定了内核应该为sockfd套接字排队的最大连接个数。

> 调用listen函数会使TCP服务器状态由ClOSED转为LISTEN。

### 接收请求

接下来就是接收客户端的连接请求，该工作由accept函数完成。

```c
#include <sys/socket.h>
int accept(int sockfd, struct sockaddr *cliaddr, socklen_t *addrlen);
```

- accept函数接收监听套接字描述符，并返回一个已连接套接字描述符。
- cliaddr参数和addrlen用于返回客户的套接字地址结构及其大小，若不关心客户端的身份，将二者设为NULL即可。

### 数据传输

数据传输过程也与客户端没有太大不同。

### 关闭连接

关闭连接也与客户端相同，只是服务端要关闭两个套接字：

- 监听套接字
- 已连接套接字

# 源程序

## 客户端

```c
#include <sys/socket.h> //socket系列函数头文件
#include <arpa/inet.h> //sockaddr_in结构体、inet_pton函数和htons函数头文件
#include <string.h> //strlen函数、strcmp函数和memset函数头文件
#include <unistd.h> //close函数头文件
#include <stdlib.h> //exit函数头文件
#include <stdio.h> //fgets函数、fputs函数、perror函数头文件

const int BUFFER_SIZE = 1024; //定义缓冲区大小
const char* server_ip = "127.0.0.1"; //服务端IP地址
const uint16_t SERVER_PORT = 2021; //服务端监听端口号

int main(){
	//定义服务端套接字地址结构
	struct sockaddr_in server_sockaddr;
	memset(&server_sockaddr, 0, sizeof(server_sockaddr));
	server_sockaddr.sin_family = AF_INET; //指定IPv4
	server_sockaddr.sin_port = htons(SERVER_PORT);
	inet_pton(AF_INET, server_ip, &server_sockaddr.sin_addr);

	//创建客户端套接字描述符
	int client_fd = socket(AF_INET, SOCK_STREAM, 0); //指定TCP协议
	if(client_fd < 0){
		perror("socket");
		exit(1);
	}

	//发起连接
	if(connect(client_fd, (struct sockaddr*) &server_sockaddr, sizeof(server_sockaddr)) < 0){
		perror("connect");
		exit(1);
	}

	//开始数据传输
	char sendbuf[BUFFER_SIZE];
	char recvbuf[BUFFER_SIZE];
	while(fgets(sendbuf, sizeof(sendbuf), stdin)){ //从stdin读入待传输数据至sendbuf
		send(client_fd, sendbuf, strlen(sendbuf), 0); //将sendbuf中的数据通过client_fd套接字传输
        if(strcmp(sendbuf, "Q\n") == 0) //输入Q表示退出
			break;
		recv(client_fd, recvbuf, sizeof(recvbuf), 0); //从client_fd套接字接收数据，保存至recvbuf
		fputs(recvbuf, stdout);//将recvbuf中的数据输出至stdout
		memset(sendbuf, 0, sizeof(sendbuf));//将sendbuf的内存值置0
		memset(recvbuf, 0, sizeof(recvbuf));//将recvbuf的内存值置0
	}

	//关闭连接
	close(client_fd);
	return 0;
}
```

## 服务端

```c
#include <sys/socket.h> //socket系列函数头文件
#include <arpa/inet.h> //sockaddr_in结构体、inet_pton函数、htons函数头文件
#include <string.h> //strlen函数、strcmp函数和memset函数头文件
#include <unistd.h> //close函数头文件
#include <stdlib.h> //exit函数头文件
#include <stdio.h> //perror函数头文件
#include <ctype.h> //toupper函数头文件

const char* server_ip = "127.0.0.1"; //指定服务端IP地址
const uint16_t SERVER_PORT = 2021;//指定监听端口号
const int QUEUE = 1024; //用于listen函数第二个参数，指定内核应为相应套接字排队的最大连接数
const int BUFFER_SIZE = 1024;//指定缓冲区大小

int main(){
	//定义服务端套接字地址结构并赋值
	struct sockaddr_in server_sockaddr;
	memset(&server_sockaddr, 0, sizeof(server_sockaddr));
	server_sockaddr.sin_family = AF_INET; //指定IPv4
	server_sockaddr.sin_port = htons(SERVER_PORT);
	inet_pton(AF_INET, server_ip, &server_sockaddr.sin_addr);

	//创建一个监听套接字描述符
	int listen_fd = socket(AF_INET, SOCK_STREAM, 0); //指定TCP协议
	if(listen_fd < 0){
		perror("socket");
		exit(1);
	}

	//将监听套接字描述符绑定到套接字地址结构
	if(bind(listen_fd, (struct sockaddr*) &server_sockaddr, sizeof(server_sockaddr)) < 0){
		perror("bind");
		exit(1);
	}

	//开始监听
	if(listen(listen_fd, QUEUE) < 0){
		perror("listen");
		exit(1);
	}

	//接受连接请求，创建 已连接套接字描述符
	int conn_fd = accept(listen_fd, NULL, NULL);
	if(conn_fd < 0){
		perror("accept");
		exit(1);
	}

	//开始数据传输
	char sendbuf[BUFFER_SIZE];
	char recvbuf[BUFFER_SIZE];
	while(1){
		memset(recvbuf, 0, sizeof(recvbuf));
		memset(sendbuf, 0, sizeof(sendbuf));
		
		recv(conn_fd, recvbuf, sizeof(recvbuf), 0); //从conn_fd套接字接收数据，保存至recvbuf
		if(strcmp(recvbuf, "Q\n") == 0) //如果收到的数据是Q，表示退出
			break;
		fputs(recvbuf, stdout); //将收到的数据原样输出至stdout
	
		//将recvbuf中的小写字母转为大写，新数据保存至sendbuf
		for(int i = 0; i < strlen(recvbuf); ++i){
			if(islower(recvbuf[i]))
				sendbuf[i] = toupper(recvbuf[i]);
			else
				sendbuf[i] = recvbuf[i];
		}

		send(conn_fd, sendbuf, strlen(sendbuf), 0); //将sendbuf中的数据通过conn_fd套接字传输
		
	}

	//关闭连接及监听描述符
	close(conn_fd);
	close(listen_fd);
	return 0;
}
```

# 数据流通图

![Simplest_Socket](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/helloworld-on-network-programming/Simplest_Socket.png)