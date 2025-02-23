# 线程池思想与框架（涉及回调函数知识）

## 为什么要用线程池
- 通过线程池化，**避免**线程资源的**反复申请与释放**，避免不必要的创建开销
- 便于**统一管理线程资源**，比如说设置线程优先级、生命周期、回收线程资源等；可以控制线程的数量，避免并发线程过多导致程序不稳定、服务器压力大等
- 线程池中有任务队列与工作队列，可以实现**任务缓冲处理**

## 线程池框架
### 一、线程池管理结构体（类）
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
-  **分配任务的线程**在加入任务后，给出信号，解除`pthread_cond_wait()`的等待，实现：
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
### 二、工作队列结构体（类）
```c
typedef struct NWORKER {
	pthread_t thread;
	int terminate;
	struct NWORKQUEUE *workqueue;
	struct NWORKER *prev;
	struct NWORKER *next;
} nWorker;
```
#### 1. 线程ID
- 线程创建时：
  ```c
  // 参数：PID、NULL、线程回调函数、线程参数
  int ret = pthread_create(&worker->thread, NULL, ntyWorkerThread, (void *)worker);
  ```
#### 2. 终止信号
- 线程通过参数`(void *)worker`获取`terminate`
- 检测到`terminate`为终止值时：
  ```c
  free(worker);// 释放该线程对应的结构体
  pthread_exit(NULL); // 退出线程
  ```
#### 3. 线程池指针
- 让该工作结构体能够获取线程池中的共享资源与互斥锁、条件变量等

### 三、任务队列结构体（类）
```c
typedef struct NJOB {
	void (*job_function)(struct NJOB *job);
	void *user_data;
	struct NJOB *prev;
	struct NJOB *next;
} nJob;
```
#### 1. 回调函数
- C 中的回调函数指针：
	- 定义：`返回值类型 (*指针名)(参数列表)`,e.g.`void (*job_function)(struct NJOB *job);`
	- 赋值: ` 指针名 = 函数名 `  ` job->job_function = king_counter; `
	```c
	void king_counter(nJob *job) {}
	```
    - 回调函数经典用法：
      ```c
      	#include <stdio.h>
		// 定义回调函数原型
		typedef int (*CallbackFunction)(int, int);
		// 实现回调函数
		int add(int a, int b) {
		    return a + b;
		}
		// 实现接收回调函数的函数
		int perform_operation(CallbackFunction callback, int a, int b) {
		    return callback(a, b);
		}
		int main() {
		    int result = perform_operation(add, 3, 5);
		    printf("The result of the operation is: %d\n", result);
		    return 0;
		}
      ```
- C++ 中的回调函数常见用法： 1. 普通函数作为回调函数  2. 类的静态成员函数作为回调函数  3. 使用std::function和std::bind  4. 使用 lambda 表达式作为回调函数
#### 2. 用户数据
- 保存任务内容，可自行设计

### 四、线程池创建函数
- 作用：创建线程与工作结构体
  - 根据最大线程数，创建工作结构体；
  - 创建线程，将PID传入工作结构体，将结构体加入工作结构体队列
  - 将线程分离，方便后续子线程回收自己的资源
### 五、线程池删除函数
- 内容：
  - 遍历线程工作结构体，向每个线程发送终止信号
  - 要广播条件变量，让所有`pthread_cond_wait()`停止等待
### 六、工作线程函数
- 作用：线程的工作内容，负责查看任务队列，并从中取任务执行
- 是什么：是线程池创建时，每个线程在回调函数位置传入的**参数**
- 参数：它的参数是管理该线程的**工作队列**中的结构体（`nWorker *worker`）
### 七、线程池运行逻辑
- 声明线程池管理结构体，定义最大线程数与最大任务数
- 创建线程池，每个线程回调函数轮询访问任务队列，执行任务
- 获取任务，并加入任务队列，通知此时有任务
- 终止时，释放工作线程资源、退出线程















