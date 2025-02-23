# 线程池思想与框架

## 为什么要用线程池
- 通过线程池化，**避免**线程资源的**反复申请与释放**，避免不必要的创建开销
- 便于**统一管理线程资源**，比如说设置线程优先级、生命周期、回收线程资源等；可以控制线程的数量，避免并发线程过多导致程序不稳定、服务器压力大等
- 线程池中有任务队列与工作队列，可以实现**任务缓冲处理**

## 线程池框架
### 线程池结构体（类）
```c
typedef struct NWORKQUEUE {
	struct NWORKER *workers;
	struct NJOB *waiting_jobs;
	pthread_mutex_t jobs_mtx;
	pthread_cond_t jobs_cond;
} nWorkQueue;

typedef nWorkQueue nThreadPool;
```
#### 1. 工作队列指针
- 存放工作结构体队列的头指针，每个工作结构体都维护一个工作线程ID、线程终止信号
#### 2. 任务队列指针
- 存放任务结构体队列的头指针，每个任务结构体都维护着自己的回调函数指针、回调函数参数
#### 3. 条件等待变量
- `pthread_cond_t jobs_cond` 提供**等待、通知**机制
- 初始化方式：
  - **动态初始化**方法（推荐）：
    ```c
    // 使用pthread_cond_init动态初始化条件变量
    if (pthread_cond_init(&workqueue->jobs_cond, NULL) != 0) {
        // 处理错误
        return -1;
    }
    ```
  - 静态初始化方法（不太推荐）：
    ```c
    pthread_cond_t blank_cond = PTHREAD_COND_INITIALIZER;
    memcpy(&workqueue->jobs_cond, &blank_cond, sizeof(workqueue->jobs_cond));
    ```
- **等待任务的工作线程**使用`pthread_cond_wait(&worker->workqueue->jobs_cond, &worker->workqueue->jobs_mtx)`
  - 这个函数使用条件等待变量、互斥锁的引用作为变量
  - 进入等待后会阻塞，并释放互斥锁（**这样才能让其他线程访问共享资源并提供等待终止信号**）
  - 收到`pthread_cond_signal()` or `pthread_cond_broadcast()`的信号后获取互斥锁，结束阻塞
  - 继续执行后续工作
-  **分配任务的线程**在加入任务后，给出信号，解除`pthread_cond_wait()`的等待
  - 实现：
    - `pthread_cond_signal(&workqueue->jobs_cond)` 结束某个线程的等待
    - `pthread_cond_broadcast(&workqueue->jobs_cond)` 结束所有线程的等待
  
#### 4. 互斥锁
- `pthread_mutex_t jobs_mtx` 对线程间**共享的资源**（任务队列）进行**访问控制**
- 初始化方式：
  - **动态初始化**方法（推荐）：
    ```c
    // pthread_mutex_t jobs_mtx; 在workqueue中声明
    // 使用pthread_mutex_init动态初始化条件变量
    if (pthread_mutex_init(&workqueue->jobs_mtx, NULL) != 0) {
        // 清理已初始化的条件变量
        pthread_cond_destroy(&workqueue->jobs_cond);
        return -1;
    }
    ```
  - 静态初始化方法（不太推荐）：
    ```c
    pthread_mutex_t blank_mutex = PTHREAD_MUTEX_INITIALIZER;
    memcpy(&workqueue->jobs_mtx, &blank_mutex, sizeof(workqueue->jobs_mtx));
    ```








