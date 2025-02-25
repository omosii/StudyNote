# 阻塞与非阻塞IO
- **阻塞**: 在进/线程中，发起一个调用时，在调用返回之前，进/线程会被**阻塞等待**，等待中的进/线**让出CPU的使用权**。
- **非阻塞**: 在进/线程中，发起一个调用时，会**立即返回**。在IO复用中，只能采用这种方案
- 在网络编程中，会产生阻塞的四个系统调用：
  - `connect()、accept()、send()、recv()`

### 一、connect()  （一般客户端阻塞就阻塞了，但了解以下处理方式更好）
- 客户端调用此函数发起连接请求
- 会有**三次握手**的过程（SYN、SYN+ACK、ACK），这是**需要时间**的
- 如果填一个**错误的IP、端口号**，`connect()`也会阻塞，十多秒后返回错误标志
- 怎么办？将其设置为非阻塞IO！
```CXX
  int setnonblocking(int fd){
      int flags;
      // 获取fd的状态。
      if  ((flags = fcntl(fd, F_GETFL, 0))==-1)
          flags = 0;
      return fcntl(fd, F_SETFL, flags|O_NONBLOCK );
  }
  setnonblocking(sockfd);          // 把sockfd设置成非阻塞。
```
- 不过应注意，设置非阻塞之后，不论连接成功与否，函数立即返回错误信息。如需判断连接成功，只需检查一下socket是否可写就行(毕竟当一个套接字连接成功建立后，它就会变为可写状态)：
```CXX
  // 使用了 poll 函数来监测套接字的可写状态  struct pollfd {
  pollfd fds;                       //       int fd;         /* 文件描述符 */
  fds.fd = sockfd;                   //      short events;     /* 要监测的事件 */
  fds.events = POLLOUT;             //       short revents;    /* 实际发生的事件 */
  poll(&fds, 1,- 1);                 //     };    // 1：表示要监测的文件描述符的数量, -1：表示 timeout 参数，设置为 -1 意味着 poll 函数会无限期阻塞
  if (fds.revents == POLLOUT)
      printf("connect ok.\n");
  else
      printf("connect failed.\n");
```

### 二、accept()
- 如果不设置监听套接字非阻塞，当没有全连接队列，就会**阻塞**，这对服务器端不友好
- 设置监听套接字非阻塞。当没有全连接队列，accept()立即返回，`errno == EAGAIN`
  - 设置方式同`connect()`:   setnonblocking(sockfd);    // 把sockfd设置成非阻塞。

### 三、send()/recv()
- `send()`:**发送**缓冲区**满**时阻塞
  - 对非阻塞IO调用`send()`,如果socket不可写（**发送**缓冲区**满**）。`send()`立即返回，`errno == EAGAIN`
- `recv()`：**接收**缓冲区**空**时阻塞
  - 对非阻塞IO调用`recv()`,如果socket不可读（**接收**缓冲区**空**）。`recv()`立即返回，`errno == EAGAIN`











