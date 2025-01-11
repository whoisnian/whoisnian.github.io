---
layout: post
title: 'Socket: UDP echo Server'
categories: programming
last_modified_at: 2017-08-29T09:06:46+08:00
---

> 玩socket第二篇，使用UDP实现简单的echo服务器。

<!-- more -->

在第一篇中使用TCP实现了简单的echo服务器，然后就想试试UDP。使用UDP的方法其实是比TCP要简单一些的。  

### 步骤
在使用socket中的UDP进行通信时，服务器进行需要以下几步：  
* `socket()` 创建服务器socket  
* `bind()` 绑定服务器socket与IP和指定的端口  
* `recvfrom()/sendto()` 对连接至客户端的套接字进行读写操作  
* `close()` 关闭所有socket  

而客户端则需要：  
* `socket()` 创建socket  
* `recvfrom()/sendto()` 对连接至服务器的套接字进行读写操作  
* `close()` 关闭使用的socket  

### 代码
使用时直接使用g++编译，先运行udp_server，再运行udp_client，然后在udp_client中输入内容发送后，就可以接收到udp_server返回的信息，当在udp_client中发送“end”后结束udp_server和udp_client。  

服务端：  
{% highlight cpp %}
/*************************************************************************
    > File Name: udp_server.cpp
    > Author: nian
    > Blog: https://whoisnian.com
    > Mail: zhuchangbao2017@gmail.com
    > Created Time: 2017年08月25日 星期五 20时29分46秒
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
    //创建服务器端套接字，地址族为AF_INET(IPv4)，传输方式为UDP
    int server_socket;
    server_socket = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);

    //初始化IP为本地127.0.0.1，端口为已设置的port
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(port);
    server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");

    //绑定套接字与IP和端口
    bind(server_socket, (struct sockaddr*)&server_addr, sizeof(server_addr));

    //一直接收客户端请求，直到接收到的内容为“end”时停止
    struct sockaddr_in client_addr;
    socklen_t client_addr_len;
    int flag = 1;
    while(flag)
    {
        char buf[1000] = {0};
        client_addr_len = sizeof(client_addr);
        recvfrom(server_socket, buf, sizeof(buf), 0, (struct sockaddr*)&client_addr, &client_addr_len);

        //如果内容是“end”就停止，否则返回给客户端输入的内容
        if(buf[0] == 'e'&&buf[1] == 'n'&&buf[2] == 'd'&&buf[3] == '\n')
            flag = 0;
        else
        {
            sendto(server_socket, buf, sizeof(buf), 0, (struct sockaddr*)&client_addr, client_addr_len);
        }
    }
    close(server_socket);
    return 0;
}
{% endhighlight %}
客户端：  
{% highlight cpp %}
/*************************************************************************
    > File Name: udp_client.cpp
    > Author: nian
    > Blog: https://whoisnian.com
    > Mail: zhuchangbao2017@gmail.com
    > Created Time: 2017年08月25日 星期五 20时39分54秒
 ************************************************************************/
#include<cstdio>
#include<cstring>

//引入头文件
#include<sys/types.h> 
#include<sys/socket.h>
#include<netinet/in.h>
#include<arpa/inet.h>
#include<unistd.h>

//设置服务器端监听端口号
const unsigned long port = 7637;

int main(void)
{
	//创建客户端套接字，地址族为AF_INET(IPv4)，传输方式为UDP
	int client_socket;
	client_socket = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);

	//初始化IP为服务器端127.0.0.1，端口为已设置的port
	struct sockaddr_in server_addr;
	memset(&server_addr, 0, sizeof(server_addr));
	server_addr.sin_family = AF_INET;
	server_addr.sin_port = htons(port);
	server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");

	//一直发送请求，直到发送的内容为“end”时停止
	int flag = 1;
	while(flag)
	{
		//获取要发送的内容
		char buf[1000];
		memset(buf, 0, sizeof(buf));
		printf("put: ");
		fgets(buf, sizeof(buf)-1, stdin);

		//发送
		socklen_t server_addr_len;
		server_addr_len = sizeof(server_addr);
		sendto(client_socket, buf, sizeof(buf), 0, (struct sockaddr*)&server_addr, server_addr_len);

		//如果内容是“end”就停止，否则读取服务器返回的内容
		if(buf[0] == 'e'&&buf[1] == 'n'&&buf[2] == 'd'&&buf[3] == '\n')
			flag = 0;
		else
		{
			recvfrom(client_socket, buf, sizeof(buf), 0, (struct sockaddr*)&server_addr, &server_addr_len);
			printf("get: %s\n", buf);
		}
	}
	close(client_socket);
	return 0;
}
{% endhighlight %}

### 函数
* 创建UDP类型的socket  
  `int udp_socket = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);`  
  或 `int udp_socket = socket(AF_INET, SOCK_DGRAM, 0);`  
  * AF_INET 指IPv4，AF_INET6 指IPv6  
  * SOCK_DGRAM 表示无连接的数据传输方式  
* 服务器绑定socket与IP和端口：  
  `int bind(int sock, struct sockaddr *addr, socklen_t addrlen);`  
* UDP接收socket收到的信息：  
  `int recvfrom(int sock, void *buf, int len, unsigned int flags, struct sockaddr *serv_addr, socklen_t *addrlen);`  
* UDP通过socket发送信息：  
  `int sendto(int sock, void *buf, int len, unsigned int flags, struct sockaddr *serv_addr, socklen_t addrlen);`  
