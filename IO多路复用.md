# 讲述I/O多路复用基本知识与用法（文末附相关代码，来自B站UP中码农论坛）
- **多线程/多进程**：适合**CPU密集型**任务，如计算、数据处理等。
- **I/O多路复用**：适合**I/O密集型**任务，如网络服务器、实时通信等。
- 本笔记将简要介绍几种I/O多路复用方式，主要介绍`epoll`模式
- 
## 一、为什么要用I/O多路复用（对比多线/进程）
### 1. 资源占用与**效率**:
  - I/O 多路复用：I/O 多路复用允许在一个线程内同时**监控多个文件描述符**（或套接字）的 I/O 事件。通过一个线程就可以处理多个并发的 I/O 操作，在**处理大量连接但 I/O 活动并不频繁的场景**下，资源占用少，系统开销小，能更高效地利用系统资源。
  - 多线程 / 多进程：每一个线程或进程通常对应一个连接或任务。当有大量并发连接时，需要创建大量的线程或进程，这会**消耗大量的内存**等系统资源，线程或进程的频繁切换也会带来**较大的上下文切换开销**，降低系统效率。
    
### 2. **响应性能**与并发处理能力
  - I/O 多路复用：能在单个线程中迅速响应多个 I/O 事件，当有 I/O 事件就绪时，能立即进行处理，**不会因为**线程或进程的**调度延迟**而**影响响应速度**。在高并发场景下，能保持**较低的延迟和较高的吞吐量**，能同时处理大量的并发连接，不会因为连接数的增加而导致性能急剧下降。
  - 多线程 / 多进程：多线程或多进程在处理大量并发连接时，由于线程或进程的调度是由操作系统决定的，可能会出现线程或进程长时间得不到调度的情况，导致**响应延迟增加**，在高并发场景下，随着并发连接数的增加，性能可能会受到线程或进程调度、资源竞争等因素的影响而下降。

### 3. 编程复杂度与维护性
   
### 4. 稳定性与可靠性

## 二、不同复用方式与对比
I/O多路复用允许单线程或单进程同时监控多个I/O操作，常见的实现方式包括 select、poll、epoll（Linux）和 kqueue（BSD/macOS）。它们在性能、可扩展性和使用场景上有所不同。
### 1. select
  - 原理：通过一个文件描述符集合(位图-`bitmap`)监控多个I/O操作，当某个描述符就绪时返回。
  - 特点：
    - 跨平台支持（几乎所有操作系统都支持）。
    - 文件描述符数量有限（通常受 FD_SETSIZE 限制，默认1024）。
    - 每次调用需要重新传递文件描述符集合，效率较低。(传递两次，一次是用户态拷贝，一次是内核态拷贝)
    - 需要遍历所有描述符来检查就绪状态，时间复杂度为 O(n)。（可利用Max_fd剪枝，但在大连接数下仍然够呛）
  - 性能：适合少量连接，高并发场景下性能较差。

### 2. poll
  - 原理：与 select 类似，但使用链表存储文件描述符，没有数量限制。
  - 特点：
    - 文件描述符数量无硬性限制。（但是太大了遍历仍然有较高时间复杂度）
    - 每次调用仍需遍历所有描述符，时间复杂度为 O(n)。
    - 相比 select，接口更灵活。
  - 性能：比 select 稍好，但在高并发下仍存在性能瓶颈。(所以地位很尴尬呀，windows会用？，不过现在都是在Linux上部署吧？)

### 3. epoll（Linux）
  - 原理：**基于事件驱动**的I/O多路复用机制，通过回调函数通知就绪事件。
  - 特点：
    - 支持边缘触发（ET）和水平触发（LT）模式。
    - 文件描述符数量无限制。
    - 仅返回就绪的文件描述符，时间复杂度为 O(1)。
    - 内核维护事件表，无需每次传递文件描述符集合。
  - 性能：高并发场景下**性能优异**，适合大规模连接。

### 4. kqueue（BSD/macOS）
  - 性能：与 `epoll` 相当，适合高并发场景。
    
## 三、Epoll
这一部分将会介绍Epoll编程流程
### 1. 创建套接字链接
  - 语句：`int listensock = initserver(atoi(argv[1]))` listensock是一个套接字文件描述符
### 2. 创建epoll句柄(这里的句柄就是epoll的文件描述符)
  - 语句：`int epollfd = epoll_create(1)`
  - 作用：创建一个epoll实例（句柄），即是一个epoll文件描述符
  - 参数：Linux 2.6.3之后随便填，大于0即可
### 3. 为服务端的listensock准备读事件
```CXX
epoll_event ev;              // 声明事件的数据结构。
ev.data.fd = listensock;   // ev.data是一个union联合体，指定事件的自定义数据，会随着epoll_wait()返回的事件一并返回。
// ev.data.ptr = (void*)"超女";   // 指定事件的自定义数据，会随着epoll_wait()返回的事件一并返回。
ev.events = EPOLLIN;      // 打算让epoll监视listensock的读事件。  EPOLLOUT为写事件
```
### 4. 把需要监听的socket和事件加入epollfd中
  - 语句：`epoll_ctl(epollfd, EPOLL_CTL_ADD, listensock, &ev);`
  - 参数：epoll实例（句柄）、EPOLL_CTL_ADD（添加）、需要监听的**socket**、epoll_event地址（监听的**事件**）
### 5. 开辟epoll_event结构体数组存放epoll返回的事件
  - 语句：`epoll_event evs[10];`
  - 参数决定性能，可大可小
### 6. 准备工作结束，进入循环
  - 等待监视的socket有事件发生
    - 语句：`int infds = epoll_wait(epollfd, evs, 10, -1);`
    - 参数：epoll实例（句柄）、存放epoll返回的事件的结构体、结构体的大小、超时时间（非负的毫秒数，-1表示不启用）
    - 返回值：表示 events 数组中**有效的事件数量**。例如，如果返回值是 3，则表示 events 数组的**前 3 个元素**包含了发生 I/O 事件的文件描述符的相关信息。（带来遍历性能的提升？）
  - 遍历`ii = 0~infds`, 处理事件请求
```CXX
           if (evs[ii].data.fd==listensock) {
                struct sockaddr_in client;
                socklen_t len = sizeof(client);
                int clientsock = accept(listensock, (struct sockaddr*)&client, &len); // 从已连接队列中将客户端的socket取出
                printf ("accept client(socket=%d) ok.\n",clientsock);
                // 为新客户端准备读事件，并添加到epoll中。
                ev.data.fd = clientsock;
                ev.events = EPOLLIN;
                epoll_ctl(epollfd, EPOLL_CTL_ADD, clientsock, &ev);
            }
           else
            {
                // 如果是客户端连接的socke有事件，表示有报文发过来或者连接已断开。
                char buffer[1024]; // 存放从客户端读取的数据。
                memset(buffer,0,sizeof(buffer));
                if (recv(evs[ii].data.fd, buffer, sizeof(buffer), 0) <= 0){  // recv中的0表示，默认阻塞接收
                    // 如果客户端的连接已断开。
                    printf("client (eventfd = %d) disconnected.\n",evs[ii].data.fd);
                    close(evs[ii].data.fd);            // 关闭客户端的socket
                    // 从epollfd中删除客户端的socket，如果socket被关闭了，会自动从epollfd中删除，所以，以下代码不必启用。
                    // epoll_ctl(epollfd, EPOLL_CTL_DEL, evs[ii].data.fd, 0);     
                }
                else{
                    // 如果客户端有报文发过来。
                    printf("recv (eventfd = %d): %s\n", evs[ii].data.fd, buffer);
                    // 把接收到的报文内容原封不动的发回去。
                    send(evs[ii].data.fd, buffer, strlen(buffer), 0); // strlen(buffer)表示字符数组的实际长度；send中的0表示，默认阻塞接收，但发送缓冲区一般不会阻塞
                }
            }

```
## 四、Epoll代码
```CXX
/*
 * 程序名：tcpepoll.cpp，此程序用于演示采用epoll模型实现网络通讯的服务端。
 * 作者：吴从周
*/
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <sys/socket.h>
#include <sys/types.h>          
#include <arpa/inet.h>
#include <sys/fcntl.h>
#include <sys/epoll.h>

// 初始化服务端的监听端口。
int initserver(int port);

int main(int argc,char *argv[])
{
    if (argc != 2) { printf("usage: ./tcpepoll port\n"); return -1; }

    // 初始化服务端用于监听的socket。
    int listensock = initserver(atoi(argv[1]));
    printf("listensock=%d\n",listensock);

    if (listensock < 0) { printf("initserver() failed.\n"); return -1; }

    // 创建epoll句柄。
    int epollfd=epoll_create(1);

    // 为服务端的listensock准备读事件。
    epoll_event ev;              // 声明事件的数据结构。
    ev.data.fd=listensock;   // 指定事件的自定义数据，会随着epoll_wait()返回的事件一并返回。
    // ev.data.ptr=(void*)"超女";   // 指定事件的自定义数据，会随着epoll_wait()返回的事件一并返回。
    ev.events=EPOLLIN;      // 打算让epoll监视listensock的读事件。

    epoll_ctl(epollfd,EPOLL_CTL_ADD,listensock,&ev);     // 把需要监视的socket和事件加入epollfd中。

    epoll_event evs[10];      // 存放epoll返回的事件。

    while (true)        // 事件循环。
    {
        // 等待监视的socket有事件发生。
        int infds=epoll_wait(epollfd,evs,10,-1);

        // 返回失败。
        if (infds < 0)
        {
            perror("epoll() failed"); break;
        }

        // 超时。
        if (infds == 0)
        {
            printf("epoll() timeout.\n"); continue;
        }

        // 如果infds>0，表示有事件发生的socket的数量。
        for (int ii=0;ii<infds;ii++)       // 遍历epoll返回的数组evs。
        {
            // printf("ptr=%s,events=%d\n",evs[ii].data.ptr,evs[ii].events);

            // 如果发生事件的是listensock，表示有新的客户端连上来。
            if (evs[ii].data.fd==listensock)
            {
                struct sockaddr_in client;
                socklen_t len = sizeof(client);
                int clientsock = accept(listensock,(struct sockaddr*)&client,&len);

                printf ("accept client(socket=%d) ok.\n",clientsock);

                // 为新客户端准备读事件，并添加到epoll中。
                ev.data.fd=clientsock;
                ev.events=EPOLLIN;
                epoll_ctl(epollfd,EPOLL_CTL_ADD,clientsock,&ev);
            }
            else
            {
                // 如果是客户端连接的socke有事件，表示有报文发过来或者连接已断开。
                char buffer[1024]; // 存放从客户端读取的数据。
                memset(buffer,0,sizeof(buffer));
                if (recv(evs[ii].data.fd,buffer,sizeof(buffer),0)<=0)
                {
                    // 如果客户端的连接已断开。
                    printf("client(eventfd=%d) disconnected.\n",evs[ii].data.fd);
                    close(evs[ii].data.fd);            // 关闭客户端的socket
                    // 从epollfd中删除客户端的socket，如果socket被关闭了，会自动从epollfd中删除，所以，以下代码不必启用。
                    // epoll_ctl(epollfd,EPOLL_CTL_DEL,evs[ii].data.fd,0);     
                }
                else
                {
                    // 如果客户端有报文发过来。
                    printf("recv(eventfd=%d):%s\n",evs[ii].data.fd,buffer);

                    // 把接收到的报文内容原封不动的发回去。
                    send(evs[ii].data.fd,buffer,strlen(buffer),0);
                }
            }
        }
    }

  return 0;
}

// 初始化服务端的监听端口。
int initserver(int port)
{
    int sock = socket(AF_INET,SOCK_STREAM,0);
    if (sock < 0)
    {
        perror("socket() failed"); return -1;
    }

    int opt = 1; unsigned int len = sizeof(opt);
    setsockopt(sock,SOL_SOCKET,SO_REUSEADDR,&opt,len); // 设置套接字允许地址复用

    struct sockaddr_in servaddr;
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(port);

    if (bind(sock,(struct sockaddr *)&servaddr,sizeof(servaddr)) < 0 )
    {
        perror("bind() failed"); close(sock); return -1;
    }

    if (listen(sock,5) != 0 )
    {
        perror("listen() failed"); close(sock); return -1;
    }

    return sock;
}

```
## 五、Select

## 六、Poll

## 七、Select代码
```CXX

```

## 八、Poll代码
```CXX

```


# 水平触发(LT模式, EPOLLIN)与边沿触发(ET模式, EPOLLIN | EPOLLET)
## 水平触发
- 读事件: 如果`epoll_wait`触发了读事件，表示有数据可读.如果程序没有把数据读完，再次调用`epoll_wait`的时候，将立即再次触发读事件。
- 写事件: 如果发送缓冲区没有满，表示可以写入数据，只要缓冲区没有被写满，再次调用`epoll_wait`的时候，将立即再次触发写事件。
## 边沿触发
- 读事件:  `epoll_wait`触发读事件后，不管程序有没有处理读事件`epoll_wait`都不会再触发读事件.只有当新的数据到达时，才再次触发读事件。
- 写事件: `epoll_wait`触发写事件之后，如果发送缓冲区仍可以写(发送缓冲区没有满)，`epoll_wait`不会再次触发写事件.只有当发送缓冲区由**满**变成**不满**时，才再次触发写事件。
## 当EPOLL服务器使用边沿触发(ET模式, EPOLLIN | EPOLLET)
- **监听套接字**和**连接套接字**可以独立设置，**互不干扰**，一个LT一个ET都是可以的，但是要正常工作，必须是
- 将**监听套接字**设置为**非阻塞**（不设置不行哦~，会导致有些套接字在全连接队列出不来）
  - 如何处理事件？当**监听套接字**有新事件到达时，用`while(1)`循环处理`accept()`，直到 返回值异常（<0）&& errno==EAGAIN  退出
- 将**连接套接字**设置为**非阻塞**（不设置不行哦~，当数据过大时会导致有些发送/接收缓冲区数据被截断）
  - 如何处理事件？当**连接套接字**有新事件到达时，用`while(1)`循环处理`recv()`，直到 返回值异常（<0）&& errno==EAGAIN  退出循环


















