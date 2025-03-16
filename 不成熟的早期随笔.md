# 知识随笔

## Linux 相关
### 1. Linux常用命令
#### 1.1 ifconfig/ip
   - 常用格式
      ``` bash
         ifconfig
         ip
      ```
    
#### 1.2 netstat/ss

### 2. Linux网络系统
#### 2.1 为什么要有DMA技术
   - 没有之前，CPU在收到硬盘控制器缓冲区数据准备好的中断后，会将**硬盘控制器缓冲区数据**_逐字节_拷贝到**PageCache**，近而将数据从**PageCache**拷贝到**用户缓冲区**
     - **缺点**：CPU亲自参与搬运数据的过程极多，而且这个过程，CPU 是不能做其他事情的，造成了CPU资源过分占用
   - DMA 技术，也就是**直接内存访问（Direct Memory Access）** 技术，可以解决这一点
     - DMA技术将**数据搬运工作**（从硬盘控制器缓冲区到内核缓冲区）全部**交给DMA控制器**，CPU就可以去处理别的事物
     - 但应注意，从**内核拷贝到用户缓冲区**的工作还是由**CPU**完成
#### 2.2 文件传输模式的演进
   - 传统模式：将磁盘上的文件读取出来，然后通过网络协议发送给客户端。涉及到4次用户态/内核态上下文切换（1次系统调用产生2次上下文切换），以及4次拷贝过程（2次CPU拷贝，2次DMA拷贝）。
     - 代码如下，通过两次系统调用：
         ``` CXX
           read(file, tmp_buf, len);
           write(socket, tmp_buf, len);
         ```
     - <img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E9%9B%B6%E6%8B%B7%E8%B4%9D/%E4%BC%A0%E7%BB%9F%E6%96%87%E4%BB%B6%E4%BC%A0%E8%BE%93.png" alt="传统文件传输模式" width="500px" height="300px">
     - 缺点：只是**搬运一份数据**，结果却**搬运了 4 次**。过多的数据拷贝会消耗 CPU 资源，大大降低了系统性能。
  - 优化思路：
    - 要想减少上下文切换到次数，就要**减少系统调用**的次数；
    - 减少「数据**拷贝**」的次数。
  - 改进方案：
    - **`mmap() + write()`**：并不是最理想的方案，因为需要四次系统调用(map(), write())与三次拷贝（CPU1，DMA2）
       - 用 **`mmap()`** 替换 `read()` 系统调用函数, `mmap()` 系统调用函数会直接把**内核缓冲区**里的数据「映射」到**用户空间**，可以减少一次拷贝（-2+1）
       - <img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E9%9B%B6%E6%8B%B7%E8%B4%9D/mmap%20%2B%20write%20%E9%9B%B6%E6%8B%B7%E8%B4%9D.png" alt=" mmap + write 传输模式" width="500px" height="300px">
    - **`sendfile()`**：Linux 内核版本 2.1 中，提供了一个专门发送文件的系统调用函数 `sendfile()`，会发生两次上下文切换与三次数据拷贝
       - 它的前两个参数分别是**目的端**和**源端的文件描述符**，后面两个参数是**源端的偏移量**和**复制数据的长度**，返回值是**实际复制数据的长度**。
         ``` CXX
         #include <sys/socket.h>
         ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
         ```
       -  <img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E9%9B%B6%E6%8B%B7%E8%B4%9D/senfile-3%E6%AC%A1%E6%8B%B7%E8%B4%9D.png" alt=" mmap + write 传输模式" width="500px" height="300px">
   - **SG-DMA 技术**：他是真正的零拷贝技术，能将**文件传输性能提升一倍以上**，实现两次系统调用与两次拷贝（DMA拷贝*2）。
     - 什么是**零拷贝**呢？ 零拷贝（Zero-copy）：**没有**在**内存层面**去**拷贝数据**，**全程没有通过CPU 来搬运数据**，所有的数据都是通过 DMA 来进行传输的。
     - 查看网卡是否支持 SG-DMA（The Scatter-Gather Direct Memory Access）技术（和普通的 DMA 有所不同）
         ``` bash
         $ ethtool -k eth0 | grep scatter-gather
         scatter-gather: on
         ```
     - 如果支持，第一步，通过 DMA 将**磁盘**上的数据拷贝**到内核缓冲区**里；第二步，**缓冲区描述符**和**数据长度**传**到 socket 缓冲区**；第三步，网卡的 **SG-DMA 控制器**直接将**内核**缓存中的数据拷贝到**网卡的缓冲区**里，流程图下所示：
     - <img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E9%9B%B6%E6%8B%B7%E8%B4%9D/senfile-%E9%9B%B6%E6%8B%B7%E8%B4%9D.png" alt=" mmap + write 传输模式" width="500px" height="300px">
#### 2.3 什么是PageCache？它有什么用？
- **是什么**：无论哪种文件传输方式，第一步都会是**磁盘控制器缓冲区**拷贝到**内核缓冲区**。这里的**内核缓冲区**就是指**磁盘高速缓存**（**PageCache**）
- **有什么用**：读写磁盘极慢，我们希望能通过**读写内存**来代替**读写磁盘**。所以在读之前我们可以使用DMA将硬盘数据读到内存中，但又出现了一个问题，内存容量往往是比较小的，所以选择读取的数据很重要：
  - 利用程序运行**局部性**原理，将**最近被访问的数据**缓存在PageCache中，但空间不足时，**淘汰最久未被访问**的缓存。
  - 所以现在读写磁盘的逻辑是什么？先在PageCache中，找不到从磁盘中读，读到后更新PageCache即可
  - PageCache 使用了「**预读功能**」，来降低机械硬盘磁头转动所造大量物理耗时。
- PageCache 的**优点**主要是两个：
  - 缓存最近被访问的数据；
  - 预读功能；
- PageCache 的**缺点**：不适用于大文件
  - 大文件会迅速沾满PageCache，并且几乎享受不了预读的好处，还导致其他小文件没有空间
  - 并且大文件几乎享受不了预读的好处，但是造成了DMA搬运开销
  - 针对大文件，可以使用**异步 直接I/O**来代替**零拷贝技术**（与直接I/O相对的是使用PageCache的缓存I/O）
    - <img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E9%9B%B6%E6%8B%B7%E8%B4%9D/%E5%BC%82%E6%AD%A5%20IO%20%E7%9A%84%E8%BF%87%E7%A8%8B.png" alt=" 异步 直接I/O " width="500px" height="300px">
#### 2.4 I/O 多路复用：select/poll/epoll
##### 2.4.1 TCP 的 Socket 编程
- 服务端
   - 首先调用` socket()` 函数，创建网络协议为 IPv4，以及传输协议为 TCP 的 Socket；
   - 接着调用` bind() `函数，给这个 Socket 绑定一个 **IP 地址和端口**
     - 绑定**端口**的目的：内核收到 TCP 报文时，会根据报头的端口号来找到对应的应用程序，进而传递数据
     - 绑定**IP 地址**的目的：一台机器是可以有多个网卡的，每个**网卡都有对应的 IP** 地址。收到对应网卡IP的数据包，内核才会发给我们；
   - 调用` listen() `函数进行监听。 可以通过 `netstat` 命令查看对应的端口号是否有被监听
   - 通过调用 `accept() `函数，来从内核**获取客户端的连接**(阻塞等待客户端连接)。
      - 当 TCP 全连接队列不为空， `accept() `函数从队列取出一个 Socket 返回应用程序，这个 Socket 是**已连接Socket**
      - **注：**监听的 Socket 和真正用来传数据的 Socket 是两个
- 客户端
   - 创建 Socket
   - 调用` connect() `函数发起连接，该函数的参数要指明 **服务端** 的 IP 地址和端口号
   - (开始三次握手，看不到)
      - 没有完成三次握手的连接，此时服务端处于 `syn_rcvd `的状态, 客户端Socket位于服务器端「还没完全建立」连接的队列，称为 **TCP 半连接队列**
   - 确立全连接状态
      - 完成了三次握手的连接，此时服务端处于 `established `状态, 客户端Socket位于服务器端「已经建立」连接的队列，称为**TCP 全连接队列**

## C++ 相关
### 1. 回调与回调函数
  - 回调是一种C/C++语言特性，可以**将自己写的代码嵌入某些官方库函数**中。
  - 回调函数，就是普通函数，只不过会作为参数传递给某些官方函数调用。
  - 为什么要使用回调函数？
    - 首先，如果我们不需要在使用官方库函数时进行特异性操作，那么完全用不到回调特性；
    - 其次，如果我们想要在官方库函数的指定位置实现特异性功能，官方可不知道我们需要进行怎样的操作；
    - 因此，我们需要写好自己的函数，然后传递进去，让官方库函数去调用。这样就实现了既有官方通用功能（官方库函数），又有我们自己特质功能（我们自己写的回调函数）的函数啦。

## git 相关
### 1. 创建本地Git库，并上传至Github网页

### 2. clone 操作

### 3. 查看分支与更改分支名

## 参考：
### 1. 小林coding系列作品
