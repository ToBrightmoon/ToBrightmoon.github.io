---
title: "spdlog源码阅读:01.异步机制解析"
date: 2025-08-26
categories: 
  - 源码分析
  - spdlog
tags:
  - C++
  - spdlog
  - 日志系统
---

# 引言
在之前的工作中使用spdlog这个开源库封装了一个异步的日志模块供上层应用使用，并借着这个机会学习阅读了spdlog的源码，在使用和阅读的过程中有一些心得，也踩了一些坑，最近终于稍微闲暇下来，准备将自己阅读源码和分析源码过程记录下来，方便日后自己的学习和复盘。
<!-- more -->
## spdlog的优势

**便于集成**：提供了头文件模式，可以直接在源码中集成

**跨平台**： 支持windows,linux,android等多个平台

**功能丰富**： 提供了同步，异步日志等多种日志模式，并且提供了丰富的格式化选择和日志输出选择 

## 阅读导航
在阅读spdlog源码，分析spdlog是怎么使用同一套接口支持同步和异步日志，以及丰富的日志类型的机制的时候，当时是从spdlog提供的异步日志的demo出发，分析日志消息是怎么在各个类之间流转，从用户输入到日志打印都经历了那些类和函数。以下是我的阅读过程：
1. **从demo出发**：追踪日志消息的产生，处理和输出路径
2. **梳理类和函数的调用链**：通过调试和阅读源码，理清日志消息流转中设计的主要类和函数调用关系
3. **分析协作机制**：理解这个类是怎么通过消息传递和多态机制协作，完成日志功能
4. **可视化设计**：梳理出关键类之后，绘制出类图，进一步明确设计思路和拓展方式

以上内容是我阅读源码的思路，也是这篇文章的主要脉络，读者可以通过这个思路更好的理解这篇文章，也可以通过这个方式去自己了解想要理解spdlog源码中的其他部分

**注：本文分析的源码为spdlog的v1.15.1版本**
# spdlog异步机制解析
## 异步测试demo
spdlog官方提供了异步日志的使用demo,包括单文件和多文件的异步日志demo

```c++
#include "spdlog/async.h"
#include "spdlog/sinks/basic_file_sink.h"
void async_example()
{
    auto async_file = spdlog::basic_logger_mt<spdlog::async_factory>("async_file_logger", "logs/async_log.txt");

}
```
```c++
#include "spdlog/async.h"
#include "spdlog/sinks/stdout_color_sinks.h"
#include "spdlog/sinks/rotating_file_sink.h"
void multi_sink_example2()
{
    spdlog::init_thread_pool(8192, 1);
    auto stdout_sink = std::make_shared<spdlog::sinks::stdout_color_sink_mt >();
    auto rotating_sink = std::make_shared<spdlog::sinks::rotating_file_sink_mt>("mylog.txt", 1024*1024*10, 3);
    std::vector<spdlog::sink_ptr> sinks {stdout_sink, rotating_sink};
    auto logger = std::make_shared<spdlog::async_logger>("loggername", sinks.begin(), sinks.end(), spdlog::thread_pool(), spdlog::async_overflow_policy::block);
    spdlog::register_logger(logger);
}
```
我们从最简答的单文件的异步日志demo入手，使用这个单文件的异步日志进行静态的字符串打印
```c++
#include "spdlog/async.h"
#include "spdlog/sinks/basic_file_sink.h"
void async_example()
{
    auto async_file = spdlog::basic_logger_mt<spdlog::async_factory>("async_file_logger", "logs/async_log.txt");
    async_file->info("message");
}
```
## 日志消息流转
以info("message")这个过程为例进行测试，整个日志消息的流转过程如下：
### 日志生产
1. logger对象指针通过info这个模板函数根据参数类型进行匹配，调用log这个模板方法
```c++
template <typename T>
void info(const T &msg) {
     log(level::info, msg);
}
```
2. log这个模板方法根据参数类型进行匹配，调用log_it_这个函数
```c++
void log(source_loc loc, level::level_enum lvl, string_view_t msg) {
    bool log_enabled = should_log(lvl);
    bool traceback_enabled = tracer_.enabled();
    if (!log_enabled && !traceback_enabled) {
        return;
    }

    details::log_msg log_msg(loc, name_, lvl, msg);
    log_it_(log_msg, log_enabled, traceback_enabled);
}
```
3. 在log_it_这个方法里，会调用logger这个类的虚函数sink_it_
```c++
void logger::log_it_(const spdlog::details::log_msg &log_msg,
                                   bool log_enabled,
                                   bool traceback_enabled) {
    if (log_enabled) {
        sink_it_(log_msg);//实际调用的async_logger这个类的sink_it_方法
    }
    if (traceback_enabled) {
        tracer_.push_back(log_msg);
    }
}
```
4. 在async_logger的sink_it_函数中，日志消息最终被封装成async_msg，并加入mpmc_queue这个队列中
```c++
void spdlog::async_logger::sink_it_(const details::log_msg &msg) {
    if (auto pool_ptr = thread_pool_.lock()) {
        pool_ptr->post_log(shared_from_this(), msg, overflow_policy_);
    } else {
        throw_spdlog_ex("async log: thread pool doesn't exist anymore");
    }
}
//构造一个async_msg，包含日志消息和async_logger的弱指针（shared_from_this()），然后调用post_async_msg_
void thread_pool::post_log(async_logger_ptr &&worker_ptr, const details::log_msg &msg, async_overflow_policy overflow_policy) {
    async_msg async_m(std::move(worker_ptr), async_msg_type::log, msg);
    post_async_msg_(std::move(async_m), overflow_policy);
}

//将async_msg入队到多生产者单消费者队列（mpsc_que）中，支持不同的溢出策略（如阻塞、丢弃新消息或覆盖旧消息）
void thread_pool::post_async_msg_(async_msg &&new_msg, async_overflow_policy overflow_policy) {
    if (overflow_policy == async_overflow_policy::block) {
        q_.enqueue(std::move(new_msg));
    } else if (overflow_policy == async_overflow_policy::overrun_oldest) {
        q_.enqueue_nowait(std::move(new_msg));
    } else {
        q_.enqueue_if_have_room(std::move(new_msg));
    }
}
```
### 日志消费
1. 线程池的工作线程将不断从队列中取出async_msg消息，并根据异步日志的类型不同的处理
```c++
void thread_pool::worker_loop_() {
    while (process_next_msg_()) {}
}

//process_next_msg_从队列中取出async_msg，根据消息类型执行操作。
bool  thread_pool::process_next_msg_() {
    async_msg incoming_async_msg;
    q_.dequeue(incoming_async_msg);
    switch (incoming_async_msg.msg_type) {
        //对于log类型，调用async_logger::backend_sink_it
        case async_msg_type::log: {
            incoming_async_msg.worker_ptr->backend_sink_it_(incoming_async_msg);
            return true;
        }
        //对于flush类型，调用async_logger::backend_sink_it
        case async_msg_type::flush: {
            incoming_async_msg.worker_ptr->backend_flush_();
            return true;
        }
        
        //对于terminate类型，结束工作线程
        case async_msg_type::terminate: {
            return false;
        }

        default: {
            assert(false);
        }
    }
    return true;
}
```
2. 在async_logger::backend_sink_it_中，async_logger遍历其持有的所有sinks_，调用每个sink的log方法
```c++
void async_logger::backend_sink_it_(const details::log_msg &msg) {
    for (auto &sink : sinks_) {
        if (sink->should_log(msg.level)) {
            sink->log(msg);
        }
    }
    
    //强制日志进行输出
    if (should_flush_(msg)) {
        backend_flush_();
    }
}
```
3. spdlog中的sink类都是继承自base_sink这个类，在base_sink的log方法中，调用了sink_it_这个虚方法，进行了具体的打印操作
```c++
template <typename Mutex>
void base_sink<Mutex>::log(const details::log_msg &msg) {
    std::lock_guard<Mutex> lock(mutex_);
    sink_it_(msg);
}
```
4. 以basic_file_sink这个类为例，在这个方法中，使用格式化器格式化了日志，并且将日志消息写入文件
```c++
template <typename Mutex>
void basic_file_sink<Mutex>::sink_it_(const details::log_msg &msg) {
    memory_buf_t formatted;
    base_sink<Mutex>::formatter_->format(msg, formatted);
    file_helper_.write(formatted);
}
```
上述过程就是，就是整个异步过程中日志消息的整个流程，通过这个流程我们可以发现，spdlog的异步模式就是经典的生产者，消费者模式，前端通过logger的打印日志的log方法将日志消息写入队列，线程池中的后端线程不断从队列中取出异步消息，根据异步消息调用async_logger本身的方法进行处理，这样就实现了异步的日志写入。

可以参考下图更加直观的感受这个过程(图片来源: https://www.cnblogs.com/shuqin/p/12214439.html)
![spdlog_seq.png](/images/spdlog_seq.png)

### 强制刷新与结束线程
在上面线程中的工作中提到了三种异步日志消息，`log`，`flush`,`terminate`。log类型的消息是用户写日志的时候产生的，那么另外两种消息是什么时候产生的呢？

spdlog支持手动强制日志输出，用户调用logger->flush()时，spdlog强制刷新日志:
```c++
//async_logger::flush_生成一个flush类型的async_msg，投递到线程池
void async_logger::flush_() {
    if (auto pool_ptr = thread_pool_.lock()) {
        pool_ptr->post_flush(shared_from_this(), overflow_policy_);
    }
}

void thread_pool::post_flush(async_logger_ptr &&worker_ptr,
                                           async_overflow_policy overflow_policy) {
    post_async_msg_(async_msg(std::move(worker_ptr), async_msg_type::flush), overflow_policy);
}
```
线程池处理flush消息，调用async_logger::backend_flush_，最终触发每个sink的flush操作。

spdlog的线程池在调用析构函数的时候，会产生`terminate`消息，优雅的结束这个工作线程
```c++
thread_pool::~thread_pool() {
    for (size_t i = 0; i < threads_.size(); i++) {
        post_async_msg_(async_msg(async_msg_type::terminate), async_overflow_policy::block);
    }
    for (auto &t : threads_) {
        t.join();
    }
}
```

## 主要类
### 类层次结构
通过上面分析日志消息流转的过程中可以发现，实现异步日志功能的主要类包括，logger，async_logger,sink,base_sink，thread_pool和mpmc_blocking_queue这几个类。
他们的类关系图如下所示：
![spdlog_class.png](/images/spdlog.png)
**logger**：用户打印日志的接口基类

**formatter**：格式化日志消息的基类

**sink**: 日志输出的基类

spdlog通过让logger组合sink，sink组合formatter，并通过合理的职责划分和接口定义，实现了良好的可拓展性，用户只需要继承sink接口，实现sink_it_和flush方法就可以实现自定义sink的实现，实现formatter类的format函数就能实现自定义格式化器

### mpmc_blocking_queue分析

mpmc_blocking_queue是存储异步日志消息的关键类，使用互斥锁和条件变量保证线程安全，内部使用环形队列(circular_q.h)存储数据
```c++
template <typename T>
class mpmc_blocking_queue 
{
public:
    using item_type = T;
   
    //阻塞模式下调用
    void enqueue(T &&item) {
        {
            std::unique_lock<std::mutex> lock(queue_mutex_);
            pop_cv_.wait(lock, [this] { return !this->q_.full(); });
            q_.push_back(std::move(item));
        }
        push_cv_.notify_one();
    }

    //overrun_oldest 覆盖旧日志模式下使用
    void enqueue_nowait(T &&item) 
    {
        {
            std::unique_lock<std::mutex> lock(queue_mutex_);
            q_.push_back(std::move(item));
        }
        push_cv_.notify_one();
    }

    //覆盖新日志模式下使用
    void enqueue_if_have_room(T &&item) 
    {
        bool pushed = false;
        {
            std::unique_lock<std::mutex> lock(queue_mutex_);
            if (!q_.full()) {
                q_.push_back(std::move(item));
                pushed = true;
            }
        }

        if (pushed) {
            push_cv_.notify_one();
        } else {
            ++discard_counter_;
        }
    }
    void dequeue(T &popped_item)
    {
        {
            std::unique_lock<std::mutex> lock(queue_mutex_);
            push_cv_.wait(lock, [this] { return !this->q_.empty(); });
            popped_item = std::move(q_.front());
            q_.pop_front();
        }
        pop_cv_.notify_one();
    }

private:
    std::mutex queue_mutex_; //全局锁，保护线程安全
    std::condition_variable push_cv_; //针对消费者的条件变量，等待非空，通知可读
    std::condition_variable pop_cv_; //针对生产者的条件变量，等待不满，通知可写
    spdlog::details::circular_q<T> q_;
    std::atomic<size_t> discard_counter_{0};
};
```
通过相关的源代码可以看出来，只有在阻塞模式下，生产者线程才会等待队列进入可写状态，其他时候均当队列满的时候都会覆盖消息；但是无论在什么模式下，所有的生产者和消费者都会去争夺互斥锁，保证线程安全；
# 一些建议
在使用的时候，虽然spdlog支持多消费模式，但是理论上写日志这个操作是IO密集型的操作，性能的瓶颈不在cpu上，多线程读取是没必要的，还会增加锁的损耗，所以多线程的消费者是没必要。
并且多个消费者无法保证日志的输出顺序，在实际的测试中也发现，单个消费者的吞吐量是比多个消费者更高，所以建议将线程池的线程数设置为1







