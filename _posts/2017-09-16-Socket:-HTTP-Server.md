---
layout: post
title: 'Socket: HTTP Server'
categories: programming
---

> 玩socket第三篇，使用TCP模拟简单的HTTP服务。

<!-- more -->

### HTTP
即超文本传输协议，其实就是一种通过TCP发送和接收信息的格式。如果服务端和客户端都按照这个格式来交换信息，它就变成了两者间的一种协议，也可以看作一种标准。  

例如请求头：  
```
GET / HTTP/1.1
Host: www.google.com
```
第一行先是指定请求方法，然后指明请求的资源路径，再说明所用的HTTP协议版本；  
第二行表示向指定主机发出请求。  

响应头：  
```
HTTP/1.1 200 OK
Content-Length: 3059
Server: GWS/2.0
Date: Sat, 11 Jan 2003 02:44:04 GMT
Content-Type: text/html
Cache-control: private
Set-Cookie: PREF=ID=73d4aef52e57bae9:TM=1042253044:LM=1042253044:S=SMCc_HRPCQiqy
X9j; expires=Sun, 17-Jan-2038 19:14:07 GMT; path=/; domain=.google.com
Connection: keep-alive
```
第一行是先说明所用的HTTP协议版本，再显示响应的状态，下面则是响应的信息。  
响应头之后紧跟着一个空行，然后由HTML格式的文本组成了实际请求的内容。  

参考：[维基百科：超文本传输协议](https://zh.wikipedia.org/wiki/%E8%B6%85%E6%96%87%E6%9C%AC%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE)

### 原理
浏览器会按照HTTP协议发送和接收内容。我们在服务端接收到浏览器的请求时，手动构建一个符合HTTP响应头的字符串，然后使用TCP方法返回给浏览器，浏览器就可以正确显示我们返回的内容。  
这里的服务端当获取到对“/”或者是“/index.html”的请求时，就以合适的响应头返回当前目录下index.html内的内容。否则返回404 Not Found的状态码。  

### 代码  
服务端：  
{% highlight cpp %}
/*************************************************************************
    > File Name: server.cpp
    > Author: nian
    > Blog: https://whoisnian.com
    > Mail: zhuchangbao2017@gmail.com
    > Created Time: 2017年08月24日 星期四 23时36分28秒
 ************************************************************************/
#include<cstdio>
#include<cstring>
#include<ctime>

//引入头文件
#include<sys/types.h> 
#include<sys/socket.h>
#include<netinet/in.h>
#include<arpa/inet.h>
#include<unistd.h>

#include<sys/stat.h>
#include<fcntl.h>

//设置服务器监听端口号
const unsigned long port = 80;

int main(void)
{
	//创建服务器端socket，地址族为AF_INET(IPv4)，传输方式为TCP
	int server_socket;
	server_socket = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);

	//初始化监听IP为本地所有IP，端口为已设置的port
	struct sockaddr_in server_addr;
	memset(&server_addr, 0, sizeof(server_addr));
	server_addr.sin_family = AF_INET;
	server_addr.sin_port = htons(port);
	server_addr.sin_addr.s_addr = inet_addr("0.0.0.0");
	
	//绑定socket与IP和端口
	bind(server_socket, (struct sockaddr*)&server_addr, sizeof(server_addr));

	//使socket进入监听模式
	listen(server_socket, 10);
	
	while(1)
	{
		//接收客户端请求
		int client_socket;
		struct sockaddr_in client_addr;
		socklen_t client_addr_len;
		client_addr_len = sizeof(client_addr);
		client_socket = accept(server_socket, (struct sockaddr*)&client_addr, &client_addr_len);

		//获取浏览器发送的请求
		char request[1000];
		read(client_socket, request, sizeof(request));
		printf("%s\n", inet_ntoa(client_addr.sin_addr));

		//获取当前时间
		time_t Time;
		char time_of_now[50];
		time(&Time);
		strftime(time_of_now, 50, "%a, %d %b %Y %H:%I:%S %Z", gmtime(&Time));
		
		char address[500];
		sscanf(request, "%*s%s", address);
		//如果不是请求”/“或”/index.html“，返回404
		if(strcmp(address, "/")&&strcmp(address, "/index.html"))
		{
			//初始化响应
			char response[100];
			memset(response, 0, sizeof(response));

			//设置响应头
			sprintf(response, "HTTP/1.1 404 Not Found\nDate: %s\nServer: C++ Server\n\n", time_of_now);

			//返回服务器响应
			write(client_socket, response, strlen(response));
		}
		//否则返回当前目录下index.html的内容
		else
		{
			//初始化响应
			char response[100000];
			char html[90000];
			memset(response, 0, sizeof(response));
			memset(html, 0, sizeof(html));

			//设置响应头
			sprintf(response, "HTTP/1.1 200 OK\nDate: %s\nServer: C++ Server\n\n", time_of_now);
		
			//设置返回的内容
			int fp = open("./index.html", O_RDONLY);
			read(fp, html, sizeof(html));
			close(fp);
			strcat(response, html);
		
			//返回服务器响应
			write(client_socket, response, strlen(response));
		}
		close(client_socket);
	}
	close(server_socket);
	return 0;
}
{% endhighlight %}
当前目录下index.html示例：  
{% highlight html %}
<html>
  <head>
	<title>
	  Permission denied
	</title>
  </head>

  <body>
	<br/><br/>
	<h1 style="font-family:verdana;font-size:10em;text-align:center;color:#303F9F">
	  Permission denied
	</h1>
  </body>
</html>
{% endhighlight %}

### 注意
* 直接`g++ server.cpp -o server`就可以编译，可执行程序需要与一个index.html文件处于相同目录下。  
* 服务端绑定的是80端口，在Linux上绑定1024以下端口时需要以管理员权限运行，例如`sudo ./server`，否则可能会提示无法访问。  
* server程序在本地运行后可以用浏览器访问[http://localhost](http://localhost)查看结果，可使用`Ctrl+C`来结束程序。  
* 此服务端每次接收请求并返回信息都需要TCP的三次握手，因此对于单个浏览器的连续多次请求效率可能会极低，而且对于多台客户端上浏览器同时访问极有可能会出现问题。  
