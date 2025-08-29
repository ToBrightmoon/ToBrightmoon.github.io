---
title: "grpc实践之路:04.grpc异步回调接口的使用"
date: 2025-08-27
categories: 
  - 动手实践-三方库
  - grpc
tags:
  - C++
  - grpc
  - 进程间通信
---
## **前言**

在之前的文章中，我们实现的服务端模型都有一个共同的特点：它们是**同步**的。无论是 Unary RPC 还是流式 RPC，我们的服务端实现在 RPC 处理函数中都会阻塞，直到该次请求处理完成。这意味着，为了同时服务多个客户端，gRPC 的同步服务器不得不在其内部的线程池中为每一个并发请求分配一个线程。这种“一个请求一个线程”的模式，在并发量不高时简单有效。但试想一下，当成千上万的客户端同时涌入时，服务器的线程资源会被迅速耗尽，系统将不堪重负。

一个真正的高性能服务，必须具备“一人多能”的本领——用少量的核心线程，处理成千上万的并发请求。要做到这一点，我们必须**拥抱异步编程**。

本文，我们将深入 gRPC C++ 最核心、也是官方最新推荐的异步服务模型——**Callback API（回调式 API）**。它也被称为“**Reactor 模式**”，因为它与我们熟悉的 muduo 等网络库的事件驱动思想如出一辙。我们将通过实践，构建一个真正意义上的高性能 gRPC 异步服务。

<!--more-->
---

## **异步服务的核心理念：事件、队列与回调**

在深入代码之前，我们必须理解 gRPC 异步模型的三个核心概念：

* **事件驱动 (Event-Driven)**: 服务器不再为每个请求阻塞等待，而是以非阻塞的方式发起所有操作（如接收请求、读写数据、发送响应）。它只关心“事件”的发生，比如“一个新的 RPC 请求到来了”、“数据发送完成了”。
* **完成队列 (grpc::ServerCompletionQueue)**: 这是 gRPC C++ 异步编程的心脏。所有异步操作的“完成事件”最终都会被放入这个队列中。我们的工作线程只需要从这个队列里取出已完成的事件进行处理即可。
* **回调 (Callback)**: 对于每一个完成的事件，我们应该执行什么操作？这就是通过回调函数来定义的。

幸运的是，从 gRPC v1.60 版本开始，官方引入了全新的 **Callback API**，极大地简化了异步服务的编写。我们不再需要手动管理复杂的完成队列和“标签”，而是可以通过实现一系列的回调函数来响应事件。


## **第一步：重新认识服务实现**

在 Callback API 模式下，我们的服务实现类依然继承自 gRPC 生成的 Service 基类，但有一个关键区别：我们不再直接重写 RPC 方法本身，而是重写一个对应的 Request + RPC 方法名 的方法。

这个方法的作用，不再是处理业务逻辑，而是**创建一个“反应堆 (Reactor)”对象，并将其返回给 gRPC 框架。**

我们以一个简单的 Unary RPC 为例：

```C++
// server.cc (新的单次 RPC 服务实现)  
#**include** <iostream>  
#**include** <memory>  
#**include** <string>  
#**include** <thread>  
#**include** <chrono>

#**include** <grpcpp/grpcpp.h>  
#**include** "controller.grpc.pb.h" // 假设这是你 gRPC 生成的头文件

// 1. 我们需要为每个 RPC 方法创建一个 Reactor 类  
//    这个类继承自 gRPC 提供的 ServerUnaryReactor  
class GreeterReactor : public grpc::ServerUnaryReactor {  
public:  
// 构造函数接收请求和响应对象  
GreeterReactor(const controller::GetVersionRequest* request, controller::GetVersionResponse* response)  
: request_(request), response_(response) {

        // 在这里可以开始处理业务逻辑  
        std::cout << "Reactor created. Processing GetVersion request..." << std::endl;  
        std::string version_str = "Async Core Service v2.0.0";  
        response_->set_version(version_str);

        // 当业务逻辑处理完毕，调用 Finish() 告诉 gRPC 框架可以发送响应了  
        Finish(grpc::Status::OK);  
    }

    // 2. 当整个 RPC 调用（包括响应发送）彻底完成时，gRPC 会调用这个方法  
    void OnDone() override {  
        std::cout << "Reactor is done. Deleting self." << std::endl;  
        // 在这里，我们可以安全地释放资源，比如删除自身  
        delete this;  
    }

    // 你也可以实现其他回调，如 OnCancel 或 OnSendInitialMetadataDone  
    void OnCancel() override {  
        std::cout << "GreeterReactor::OnCancel" << std::endl;  
    }

    void OnSendInitialMetadataDone(bool ok) override {  
        std::cout << "GreeterReactor::OnSendInitialMetadataDone: " << (ok ? "Success" : "Failed") << std::endl;  
    }

private:  
const controller::GetVersionRequest* request_;  
controller::GetVersionResponse* response_;  
};

class ControllerServiceImpl final : public controller::Controller::CallbackService {  
public:  
// 3. 注意！我们不再重写 GetVersion，而是重写返回 ServerUnaryReactor* 的 GetVersion 方法  
//    （对于 CallbackService，它直接对应 RPC 方法名）。  
grpc::ServerUnaryReactor* GetVersion(grpc::CallbackServerContext* context,  
const controller::GetVersionRequest* request,  
controller::GetVersionResponse* response) override {  
// 这个方法的核心职责就是：创建一个新的 Reactor 实例，然后返回它  
// gRPC 框架会接管这个 Reactor 的生命周期  
return new GreeterReactor(request, response);  
}  
};
```
---
![reactor](/images/reactor_workflow.png)
## **第二步：理解新的异步工作流**

这个基于 Reactor 的新模型，其工作流程是这样的：

1. **注册服务**： 我们将 ControllerServiceImpl 的一个实例注册到 ServerBuilder 中。
2. **等待请求**： gRPC 框架的 I/O 线程在后台监听新请求。
3. **创建 Reactor**： 当一个 GetVersion 请求到来时，gRPC 框架会自动调用我们实现的 ControllerServiceImpl::GetVersion 方法。
4. **返回 Reactor**： 我们在这个方法里 new 一个 GreeterReactor 对象并返回。我们只负责创建，不负责销毁。
5. **业务处理与响应**： 在 GreeterReactor 的构造函数中，我们处理业务逻辑，填充 response，然后调用 Finish(Status::OK)。这个 Finish 调用是非阻塞的，它只是告诉 gRPC 框架：“我的活儿干完了，你可以把响应发出去了。”
6. **完成与销毁**： 当 gRPC 框架真正将响应发送完毕，并完成了所有清理工作后，它会调用 GreeterReactor::OnDone()。我们在这个回调函数中，安全地 delete this，完成 Reactor 对象的生命周期闭环。


![com](/images/completion_queue_sequence-0.png)
### **扩展：如何实现异步的流式服务？**

Reactor 模式对于流式 RPC 同样强大。让我们看看如何用它来实现一个服务端流式 RPC —— GetRealtimeLogs。

```c++
// [新增] Reactor 类，用于处理一个 GetRealtimeLogs 请求的生命周期  
class LogStreamReactor : public grpc::ServerWriteReactor<controllerStream::LogEntry> {  
public:  
// 构造函数接收请求，并立即开始发送第一条日志  
LogStreamReactor(const controllerStream::GetLogsRequest* request)  
: request_(*request) // 保存请求的副本（如果需要的话）  
{  
std::cout << "Client subscribed for logs (Async Reactor)." << std::endl;  
// 启动第一次写入  
SendLog();  
}

    // 当流完成时（无论成功、失败或取消），此方法被调用  
    void OnDone() override {  
        std::cout << "LogStreamReactor: OnDone called. Cleaning up." << std::endl;  
        // 在这里释放所有与此 Reactor 相关的资源  
        delete this;  
    }

    // 当客户端取消流时，此方法被调用  
    void OnCancel() override {  
        std::cout << "LogStreamReactor: Client cancelled the request." << std::endl;  
        // OnCancel 之后通常会调用 OnDone，所以主要清理逻辑放在 OnDone  
    }

    // 当上一次的 StartWrite() 操作完成时，此方法被调用  
    void OnWriteDone(bool ok) override {  
        // 'ok' 为 false 表示写入失败（例如，连接断开）  
        if (!ok) {  
            std::cout << "LogStreamReactor: Failed to write to stream. Connection might be broken." << std::endl;  
            // 不需要手动调用 Finish，当 OnWriteDone 返回后，gRPC 会自动处理清理  
            // 最终会调用 OnDone  
            return;  
        }

        // 检查是否还有更多日志要发送  
        if (log_counter_ <= 10) {  
            // 如果上一次写入成功，就继续发送下一条  
            SendLog();  
        } else {  
            // 所有日志都已发送完毕  
            std::cout << "LogStreamReactor: Finished sending all logs." << std::endl;  
            // 发送一个 OK 状态来正常关闭流  
            Finish(grpc::Status::OK);  
        }  
    }

private:  
void SendLog() {  
// 准备下一条日志  
log_entry_.set_timestamp(std::chrono::system_clock::to_time_t(std::chrono::system_clock::now()));  
log_entry_.set_level("INFO");  
log_entry_.set_message("This is async log message number " + std::to_string(log_counter_));

        log_counter_++;

        // 模拟日志生成的延迟  
        // 重要提示：在异步模型中，长时间的 sleep 会阻塞处理其他事件的线程！  
        // 在真实应用中，这里应该是快速的非阻塞操作。  
        // 为了演示，我们仍然使用 sleep，但要意识到它的影响。  
        std::this_thread::sleep_for(std::chrono::seconds(1));

        // 发起异步写入操作。我们传递 log_entry_ 的地址。  
        // gRPC 会在 OnWriteDone 被调用之前保证这个地址是有效的。  
        StartWrite(&log_entry_);  
    }

    const controllerStream::GetLogsRequest request_;  
    controllerStream::LogEntry log_entry_; // 重用 LogEntry 对象以避免重复分配内存  
    int log_counter_ = 1;  
};

class ControllerStreamServiceImpl final : public controllerStream::Controller::CallbackService {  
// 注意：服务类继承自 ...::CallbackService 而不是 ...::Service

    // 实现 GetRealtimeLogs 方法  
    // 它不再返回 Status，而是返回一个 ServerWriteReactor*  
    grpc::ServerWriteReactor<controllerStream::LogEntry>* GetRealtimeLogs(  
        grpc::CallbackServerContext* context,  
        const controllerStream::GetLogsRequest* request) override {

        // 创建一个新的 Reactor 来处理这个请求  
        // gRPC 框架将拥有这个指针的所有权，并在 OnDone 中由我们自己 delete  
        return new LogStreamReactor(request);  
    }  
};
```
---

## **第三步：启动异步服务器**

最令人惊喜的是，启动一个异步服务器的主函数部分，与同步服务器几乎一模一样，甚至更简单。我们不再需要自己管理线程池。

C++
```c++
/ server_main.cc  
void RunServer() {  
std::string server_address("0.0.0.0:50051");  
// 这里根据你的服务类型选择正确的实现类，例如 ControllerServiceImpl 或 ControllerStreamServiceImpl  
ControllerServiceImpl service;

    grpc::ServerBuilder builder;  
    builder.AddListeningPort(server_address, grpc::InsecureServerCredentials());

    // 注册回调服务（异步模式）  
    builder.RegisterService(&service);

    std::unique_ptr<grpc::Server> server(builder.BuildAndStart());  
    std::cout << "Async Reactor Server listening on " << server_address << std::endl;

    server->Wait(); // 阻塞主线程，等待服务器关闭  
}

int main(int argc, char** argv) {  
RunServer();  
return 0;  
}
```

gRPC 框架内部已经为我们处理好了线程池、完成队列、以及事件分发的所有复杂工作。我们只需要提供一个个小巧、独立的 Reactor 对象来处理具体的业务逻辑即可。

---

## **个人思考：新旧异步模型的对比与 Reactor 的优雅**

如果你之前了解过 gRPC 旧的、基于 CompletionQueue 和 void* tag 的异步 API，你会发现这个新的 Callback API (Reactor 模式) 是一个巨大的进步：

* **告别手动状态管理**： 在旧模型中，我们需要在一个庞大的 switch(tag) 语句中手动管理一次 RPC 的不同阶段（CREATE, PROCESS, FINISH），非常容易出错。现在，这些状态被封装在了 Reactor 对象内部，逻辑更清晰。
* **无需手动管理 CompletionQueue**： 我们不再需要编写 while(cq_->Next()) 这样的事件循环，gRPC 框架为我们代劳了。
* **更符合现代 C++ 的回调思想**： 这种“注册回调，等待框架调用”的模式，与 muduo 的事件处理、Qt 的信号槽都非常相似，更符合现代 C++ 开发者的直觉。

---

## **总结与展望**

本文，我们迈出了从“能用”到“高性能”的关键一步。通过学习和实践 gRPC 最新的 Callback API (Reactor 模式)，我们构建了一个真正意义上的异步服务端。

我们深入理解了如何通过实现 ServerUnaryReactor 和 ServerWriteReactor，将业务逻辑与 gRPC 的事件循环解耦，从而让我们的服务能够用少量的线程处理海量的并发请求。
