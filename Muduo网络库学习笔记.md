## noncopyable模块
实现禁用拷贝构造与赋值运算符，广泛用于派生类构造限制，可复用

## 时间戳模块
输出当前时间的模块，复用性很高

## 日志模块
主要是通过单例设计模式与定义日志宏来实现muduo网络库的日志输出，复用性很高
### 日志头文件 Logger.h
```CXX
#pragma once // 通过文件路径确保只编译一次，#ifndef #define #endif 这种每次都会进入编译，再做判断

#include <string>

#include "noncopyable.h"
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
  void printLog(std::string msg);
private:
  int logLevel_;
};
```
### 日志cc文件 Logger.cc
```CXX
#include "Logger.h"
#include "Timestamp.h"

#include <iostream>

// 获取唯一实例
Logger& Logger::instance()
{
  static Logger logger; // 每次调用都会查看 Logger静态对象logger 是否创建，没有则在静态区创建，创建则不执行当前语句，返回对象引用
  return logger;
}
// 设置日志级别
void Logger::setLogLevel(int level)
{
  logLevel_ = level;
}
// 打印日志 打印格式：  [日志级别] 时间 : 信息
void Logger::printLog(std::string msg)
{
  // [日志级别]
  switch(logLevel_)
  {
    case INFO:  // 普通信息  0
      std::cout << "[INFO]" ;
      break;
    case ERROR: // 错误信息  1
      std::cout << "[ERROR]" ;
      break;
    case FATAL: // core信息  2
      std::cout << "[FATAL]" ;
      break;
    case DEBUG: // 调试信息  3
      std::cout << "[DEBUG]" ;
      break;
    default: 
      break;
  }
  // 时间 : 信息
    std::cout << Timestamp::now().toString() << " : " << msg << std::endl;
}
```
### 日志如何创建唯一实例？ static! - 静态成员变量与静态成员函数
- 头文件中使用`static`关键字修饰成员函数（此时函数中的第一个参数将不再是this对象指针），表示不需要实例化对象即可调用
- 每次调用`Logger::instance()`都会查看 Logger 的静态对象 logger 是否创建
  - 没有 则在静态区创建，返回对象引用
  - 已创建 则直接返回对象引用
```CXX
// .h头文件中
  // 获取日志唯一实例对象
  static Logger& instance();
// .cc文件中
  Logger& Logger::instance()
  {
    static Logger logger; // 
    return logger;
  }
```
### 日志宏的实现
- 日志宏提供给开发者便捷的日志输出语句，使用格式化字符串的方式大大减少了日志输出的工作量
- 日志宏提供四种日志输出，分别为 `LOG_INFO(logmsgFormat, ...)`、`LOG_ERROR(logmsgFormat, ...)`、`LOG_FATAL(logmsgFormat, ...)`、`LOG_DEBUG(logmsgFormat, ...)`  可将其定义在 Logger.h 头文件中。需要使用日志宏时直接包含头文件即可
  - `LOG_INFO(logmsgFormat, ...)`如下，其余同理
```CXX
#define LOG_INFO(logmsgFormat, ...)\    // ...代表任意多个参数
  do                                                        \
  {                                                         \
    Logger& logger = Logger::instance();                    \  // 获取 唯一实例
    logger.setLogLevel(INFO);                               \  // 设置等级
    char buf[1024] = {0};                                   \  // 准备字符串数组
    snprintf(buf, 1024, logmsgFormat, ##__VA_ARGS__);       \  // 填充字符串数组
    logger.printLog(buf);                                   \  // 打印buf内容，char* 隐式转换为 std::string
  }while(0)                                                 \
```
- `LOG_INFO(logmsgFormat, ...)`如下，其余同理







