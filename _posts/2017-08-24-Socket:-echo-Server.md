---
layout: post
title: 'Socket: echo Server'
categories: Programming
---

> 玩socket第一篇，简单的echo服务器。

<!-- more -->

### Socket
socket在我的认识中就是用于不同进程间的通讯，甚至包括了不同主机上的不同进程，通讯的两端就是两个socket。  

根据Linux下“一切皆文件”的原则，用socket进行通讯就是对创建并连接好的socket进行读（read）写（write）操作，客户端对连接到服务器的socket进行write就是发送信息，进行read就是接收信息；服务器则是对连接到客户端的socket进行读写操作。  

### TCP和UDP
通过socket传输信息的方式主要有两种：TCP和UDP。
* TCP的话，需要客户端先发送一个消息，询问服务器在不在线；服务器接收到之后发出回应，回答自己在线；然后客户端收到之后再发送一次消息，告诉服务器准备接收自己的信息。在这三次握手之后才是客户端向服务器真正发送信息的过程。  
关于为什么是三次握手，而不是看起来更快的两次握手的问题，可以这样想：原则上任何数据传输都是不能保证完全可靠的，TCP就是要为通讯双方建立一个尽量可靠的连接，只有当连接尽量可靠时双方才开始通讯。  
  * 如果是两次握手，Client先向Server发送请求，Server收到，于是返回请求，接着Server就进入连接状态了，然而Client并不一定可以收到Server返回的请求，在Server到Client的连接不可靠时Server就进入了连接状态。  
  * 如果是三次握手，Client先发送请求，Server接收后返回，如果Client收不到Server返回的消息，两方就都不会进入连接状态，只有在Client收到Server的返回之后，Client再发送一个请求并且Server成功收到，此时Client到Server和Server到Client的通道都已经相对可靠了，双方才会进入连接状态。  
* UDP的话，就是直接向目标地址发送真正的信息，不去考虑目标是否可以收到。和面向连接的TCP相比，UDP是非连接的。  

两者相比，TCP的可靠性明显比UDP要高，但UDP的传输速度则要比TCP高很多。在网络状况比较差时，TCP的可靠连接难以建立，反而不如使用UDP不停地发送信息效果好。TCP的速度虽然难以超过UDP，但在单次传输的数据量越来越大时，TCP的速度也会越来越接近UDP。  

### 步骤
在使用C语言中的socket进行通信时，服务器进行需要以下几步：  
* `socket()` 创建服务器socket  
* `bind()` 绑定服务器socket与IP和指定的端口  
* `listen()` 服务器socket进入监听状态  
* `accept()` 得到连接上客户端的socket（此时已经完成了三次握手）  
* `read()/write()` 对连接至客户端的套接字进行读写操作  
* `close()` 关闭所有socket  

而客户端则需要：  
* `socket()` 创建socket  
* `connect()` 将创建的socket连接至服务器  
* `read()/write()` 对连接至服务器的套接字进行读写操作  
* `close()` 关闭使用的socket  

### 代码
使用时直接使用g++编译，先运行server，再运行client，然后在client中输入内容发送后，就可以接收到server返回的信息，当在client中发送“end”后结束server和client。  

服务端：  
{% highlight cpp %}
/*************************************************************************
    > File Name: server.cpp
    > Author: nian
    > Blog: https://whoisnian.com
    > Mail: zhuchangbao2017@gmail.com
    > Created Time: 2017年08月23日 星期三 22时29分46秒
 ************************************************************************/
#include<cstdio>
#include<cstring>

//引入头文件
#include<sys/types.h>
#include<sys/socket.h>
#include<netinet/in.h>
#include<arpa/inet.h>
#include<unistd.h>

//设置服务器监听端口号
const unsigned long port = 7637;

int main(void)
{
	//创建服务器端socket，地址族为AF_INET(IPv4)，传输方式为TCP
	int server_socket;
	server_socket = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);

	//初始化IP为本地127.0.0.1，端口为已设置的port
	struct sockaddr_in server_addr;
	memset(&server_addr, 0, sizeof(server_addr));
	server_addr.sin_family = AF_INET;
	server_addr.sin_port = htons(port);
	server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");

	//绑定socket与IP和端口
	bind(server_socket, (struct sockaddr*)&server_addr, sizeof(server_addr));

	//使socket进入监听模式
	listen(server_socket, 10);

	//等待接收客户端请求
	int client_socket;
	struct sockaddr_in client_addr;
	socklen_t client_addr_len;
	client_addr_len = sizeof(client_addr);
	client_socket = accept(server_socket, (struct sockaddr*)&client_addr, &client_addr_len);

	//一直获取内容，直到获取到的内容为“end”时停止
	int flag = 1;
	while(flag)
	{
		char buf[1000];
		read(client_socket, buf, sizeof(buf));

		//如果内容是“end”就停止，否则返回给客户端输入的内容
		if(buf[0] == 'e'&&buf[1] == 'n'&&buf[2] == 'd'&&buf[3] == '\n')
			flag = 0;
		else
		{
			write(client_socket, buf, sizeof(buf));
		}
	}
	close(client_socket);
	close(server_socket);
	return 0;
}
{% endhighlight %}
客户端：  
{% highlight cpp %}
/*************************************************************************
    > File Name: client.cpp
    > Author: nian
    > Blog: https://whoisnian.com
    > Mail: zhuchangbao2017@gmail.com
    > Created Time: 2017年08月23日 星期三 23时39分54秒
 ************************************************************************/
#include<cstdio>
#include<cstring>

//引入头文件
#include<sys/types.h>
#include<sys/socket.h>
#include<netinet/in.h>
#include<arpa/inet.h>
#include<unistd.h>

//设置服务器监听端口号
const unsigned long port = 7637;

int main(void)
{
	//创建服务器socket，地址族为AF_INET(IPv4)，传输方式为TCP
	int server_socket;
	server_socket = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);

	//初始化IP为服务器127.0.0.1，端口为已设置的port
	struct sockaddr_in server_addr;
	memset(&server_addr, 0, sizeof(server_addr));
	server_addr.sin_family = AF_INET;
	server_addr.sin_port = htons(port);
	server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");

	//客户端连接服务器
	connect(server_socket, (struct sockaddr*)&server_addr, sizeof(server_addr));

	//一直发送请求，直到发送的内容为“end”时停止
	int flag = 1;
	while(flag)
	{
		//发送内容
		char buf[1000];
		memset(buf, 0, sizeof(buf));
		printf("put: ");
		fgets(buf, sizeof(buf)-1, stdin);
		write(server_socket, buf, sizeof(buf));

		//如果内容是“end”就停止，否则读取并输出服务器返回的内容
		if(buf[0] == 'e'&&buf[1] == 'n'&&buf[2] == 'd'&&buf[3] == '\n')
			flag = 0;
		else
		{
			read(server_socket, buf, sizeof(buf));
			printf("get: %s\n", buf);
		}
	}
	close(server_socket);
	return 0;
}
{% endhighlight %}

### 函数
* 创建TCP类型的socket  
  `int tcp_socket = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);`  
  或 `int tcp_socket = socket(AF_INET, SOCK_STREAM, 0);`  
  * AF_INET 指IPv4，AF_INET6 指IPv6  
  * SOCK_STREAM 表示面向连接的数据传输方式  
* 创建UDP类型的socket  
  `int udp_socket = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);`  
  或 `int udp_socket = socket(AF_INET, SOCK_DGRAM, 0);`  
  * AF_INET 指IPv4，AF_INET6 指IPv6  
  * SOCK_DGRAM 表示无连接的数据传输方式  
* 服务器绑定socket与IP和端口：  
  `int bind(int sock, struct sockaddr *addr, socklen_t addrlen);`  
* 客户端连接服务器：  
  `int connect(int sock, struct sockaddr *serv_addr, socklen_t addrlen);`  
* 服务器端socket进入监听状态：  
  `int listen(int sock, int backlog);`  
  * `sock` 需要进入监听状态的socket  
  * `backlog` 请求队列最大长度  
* 处于监听状态的socket接受请求：  
  `int accept(int sock, struct sockaddr *addr, socklen_t *addrlen);`  
  * `accept()` 返回一个新的socket来和客户端通信  
  * `listen()` 只是让socket进入监听状态，并没有真正接收客户端请求。  

### 注意
* 一些端口号被固定的服务占用，如22的SSH，80的http等，所以一般分配2000-65535内的端口。  
* `bind()`，`connect()`，以及`accept()`中的sockaddr结构体内IP地址和端口在一起存储，无法直接分开读入，所以通常构造sockaddr_in结构体，然后在使用时转换为sockaddr类型，两种类型大小相等，防止了转换中出现问题。  
  {% highlight c %}
  struct sockaddr_in{
    sa_family_t     sin_family;   //地址族，即地址类型，与创建套接字时保持一致
    uint16_t        sin_port;     //16位的端口号，要用htons()转换
    struct in_addr  sin_addr;     //32位IP地址
    char            sin_zero[8];  //不使用，一般用0填充
  };

  struct in_addr{
    in_addr_t  s_addr;  //32位的IP地址，要用inet_addr()将字符串类型的IP地址转换为数字
  };

  struct sockaddr{
    sa_family_t  sin_family;   //地址族（Address Family），也就是地址类型
    char         sa_data[14];  //IP地址和端口号
  };
  {% endhighlight %}
* `htons()`将主机字节顺序转换为网络字节顺序，因为不同计算机存储数据时有两种字节顺序，而在通过网络传输时则需要统一为相同的字节顺序。  
  ```
  htonl()--"Host to Network Long int" 32Bytes  
  ntohl()--"Network to Host Long int" 32Bytes  
  htons()--"Host to Network Short int" 16Bytes  
  ntohs()--"Network to Host Short int " 16Bytes  
  ```
