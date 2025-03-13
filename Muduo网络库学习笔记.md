## noncopyable模块
实现禁用拷贝构造与赋值运算符，广泛用于派生类构造限制，可复用

## 时间戳模块
输出当前时间的模块，复用性很高

## 日志模块
主要是通过单例设计模式与定义日志宏来实现muduo网络库的日志输出，复用性很高
### 日志头文件 Logger.h
```CXX
// 枚举定义日志级别
enum LogLevel
{
  INFO,  // 普通信息  0
  ERROR, // 错误信息  1
  FATAL, // core信息  2
  DEBUG, // 调试信息  3
};
// 日志类
class Logger : noncopyable
{
public:
  // 获取日志唯一实例对象
  static Logger& instance();
  // 设置日志级别
  void setLogLevel(int level);
  // 写日志
  void writeLog(std::string msg);
private:
  int logLevel_;
};
```
### 日志cc文件 Logger.cc
```CXX
```










