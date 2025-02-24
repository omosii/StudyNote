# Socket编程的服务器端与客户端讲解（文末附相关代码，来自B站UP主码农论坛）
Socket编程分为TCP与UDP
- TCP：有连接的，可靠的。有差错检测，重传等机制
- UDP：无连接的，非可靠的。优点是速度快，带宽大
- 以下介绍TCP的客户端与服务端模型
## 服务器端
- 服务器端是被动的接收客户端的请求，包括连接请求、业务请求、关闭连接请求
- 主要的流程是：
  - 创建套接字`socket()`：创建网络通信实体。Linux中套接字以文件形式存在
  - 绑定套接字`bind()`：配置套接字实体。指定服务器自身IP、端口、协议
  - 监听套接字`listen()`：监听套接字实体
  - 接收连接`accept()`：若有完成三次握手的套接字，`accept()`将其从全连接队列中取出，否则阻塞。**应注意这里的套接字是通信套接字，与监听套接字不同**
  - 网络通信`recv()/send()`：数据收发
  - 关闭套接字`close()`：发现客户端连接关闭，将自身监听套接字、客户端套接字关闭
## 客户端
- 客户端主动向服务器端发起连接请求，三次握手完成后即可发送/接收数据
- 主要的流程是：
  - 创建套接字`socket()`：创建网络通信实体。Linux中套接字以文件形式存在
    - 函数使用：`int sockfd = socket(AF_INET, SOCK_STREAM, 0);`
    - 返回值：失败返回-1，errno被设置。成功返回已连接套接字的文件描述符
    - 参数：1、通讯的协议家族 `AF_INET(IPv4)`、 `AF_INET6(IPv6)`   2、数据传输的类型  SOCK_STREAM（面向连接的socket）、 SOCK_DGRAM（无连接的socket）   3、最终使用的协议  不用管，填0即可，编译器会自动推到
  - 请求套接字连接`connect()`：指定服务器端IP、端口、协议，配置套接字，并通过其向服务器端发起连接请求
    - 函数使用：`connect(sockfd, (struct sockaddr *)&servaddr, sizeof(servaddr))!=0`
    - 返回值：失败返回-1，errno被设置。
    - 参数： 1、发起连接的套接字   2、存放服务端信息的结构体（需要注意**IP与端口**需要从**主机小端序**转为**网络大端序**）   3、存放服务端信息的结构体大小
      - 在客户端代码中，存放服务端信息的结构体可使用以下代码初始化, 这种方式可以**接收IP地址**也可以**接收域名**：
        ```CXX
         // 第2步：向服务器发起连接请求。 
          struct hostent* h;    // 用于存放服务端IP的结构体。
          if ( (h = gethostbyname(argv[1])) == 0 ){  // 把字符串格式的IP转换成结构体。  e.g. argv[1] = "192.168.137.1"
            cout << "gethostbyname failed.\n" << endl; close(sockfd); return -1;}
          struct sockaddr_in servaddr;              // 用于存放服务端IP和端口的结构体。
          memset(&servaddr,0,sizeof(servaddr));   
          servaddr.sin_family = AF_INET;
          memcpy(&servaddr.sin_addr,h->h_addr,h->h_length); // 指定服务端的IP地址。
          servaddr.sin_port = htons(atoi(argv[2]));         // 指定服务端的通信端口。
          
          if (connect(sockfd,(struct sockaddr *)&servaddr,sizeof(servaddr))!=0){  // 向服务端发起连接清求。
            perror("connect"); close(sockfd); return -1; }
        ```
      - 初始化`servaddr.sin_addr`时,也可以使用` servaddr.sin_addr.s_addr = inet_addr(const char *cp); `初始化
  - 网络通信`recv()/send()`：`connect()`成功后，可用这些函数进行数据收发
    - 函数使用：`iret = send(sockfd,buffer,strlen(buffer),0)` 成功返回**发送数据的大小**
    - 函数使用：`iret = recv(sockfd,buffer,sizeof(buffer),0)` 成功返回**接收数据的大小**
  - 关闭套接字`close()`:将已连接套接字关闭
    - 函数使用：`close(sockfd);` 
## 服务器端代码
```CXX

```
## 客户端代码
```CXX
/*
 * 程序名：demo1.cpp，此程序用于演示socket的客户端
*/
#include <iostream>
#include <cstdio>
#include <cstring>
#include <cstdlib>
#include <unistd.h>
#include <netdb.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
using namespace std;
 
int main(int argc,char *argv[])
{
  if (argc!=3)
  {
    cout << "Using:./demo1 服务端的IP 服务端的端口\nExample:./demo1 192.168.101.139 5005\n\n"; 
    return -1;
  }

  // 第1步：创建客户端的socket。  
  int sockfd = socket(AF_INET,SOCK_STREAM,0);
  if (sockfd==-1)
  {
    perror("socket"); return -1;
  }
 
  // 第2步：向服务器发起连接请求。 
  struct hostent* h;    // 用于存放服务端IP的结构体。
  if ( (h = gethostbyname(argv[1])) == 0 )  // 把字符串格式的IP转换成结构体。
  { 
    cout << "gethostbyname failed.\n" << endl; close(sockfd); return -1;
  }
  struct sockaddr_in servaddr;              // 用于存放服务端IP和端口的结构体。
  memset(&servaddr,0,sizeof(servaddr));
  servaddr.sin_family = AF_INET;
  memcpy(&servaddr.sin_addr,h->h_addr,h->h_length); // 指定服务端的IP地址。
  servaddr.sin_port = htons(atoi(argv[2]));         // 指定服务端的通信端口。
  
  if (connect(sockfd,(struct sockaddr *)&servaddr,sizeof(servaddr))!=0)  // 向服务端发起连接清求。
  { 
    perror("connect"); close(sockfd); return -1; 
  }
  
  // 第3步：与服务端通讯，客户发送一个请求报文后等待服务端的回复，收到回复后，再发下一个请求报文。
  char buffer[1024];
  for (int ii=0;ii<3;ii++)  // 循环3次，将与服务端进行三次通讯。
  {
    int iret;
    memset(buffer,0,sizeof(buffer));
    sprintf(buffer,"这是第%d个超级女生，编号%03d。",ii+1,ii+1);  // 生成请求报文内容。
    // 向服务端发送请求报文。
    if ( (iret=send(sockfd,buffer,strlen(buffer),0))<=0)
    { 
      perror("send"); break; 
    }
    cout << "发送：" << buffer << endl;

    memset(buffer,0,sizeof(buffer));
    // 接收服务端的回应报文，如果服务端没有发送回应报文，recv()函数将阻塞等待。
    if ( (iret=recv(sockfd,buffer,sizeof(buffer),0))<=0)
    {
       cout << "iret=" << iret << endl; break;
    }
    cout << "接收：" << buffer << endl;

    sleep(1);
  }
 
  // 第4步：关闭socket，释放资源。
  close(sockfd);
}

```
