# 阻塞与非阻塞IO
- **阻塞**: 在进/线程中，发起一个调用时，在调用返回之前，进/线程会被**阻塞等待**，等待中的进/线**让出CPU的使用权**。
- **非阻塞**: 在进/线程中，发起一个调用时，会**立即返回**。在IO复用中，只能采用这种方案
- 在网络编程中，会产生阻塞的四个系统调用：
  - `connect()、accept()、send()、recv()`
### 一、send()/recv()


### 二、connect()
- 客户端调用此函数发起连接请求
- 会有**三次握手**的过程（SYN、SYN+ACK、ACK），这是**需要时间**的
- 如果填一个**错误的IP、端口号**，`connect()`也会阻塞，十多秒后返回错误标志
- 怎么办？
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

