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
     - 如果支持，**第一步**，通过 DMA 将**磁盘**上的数据拷贝**到内核缓冲区**里；第二步，**缓冲区描述符**和**数据长度**传**到 socket 缓冲区**；第三步，网卡的 SG-DMA 控制器直接将内核缓存中的数据拷贝到网卡的缓冲区里。
     - <img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E9%9B%B6%E6%8B%B7%E8%B4%9D/senfile-%E9%9B%B6%E6%8B%B7%E8%B4%9D.png" alt=" mmap + write 传输模式" width="500px" height="300px">
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
