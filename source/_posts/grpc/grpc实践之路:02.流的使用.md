---
title: "grpc实践之路:02.流的使用"
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

在上一篇文章《gRPC 实践之路（一）：从一个项目需求谈起》中，我们成功地搭建了一个 gRPC 服务，并让客户端通过一次“请求-响应”式的 Unary RPC 调用，获取到了服务端的版本号。这就像是我们去前台问了个问题，前台给了我们一个确切的答案，然后对话就结束了。

但是实际中,很多消息不是一个一次性产生完成的，可能需要你去持续的接受，比如 ：**数据源持续的产生日志，每条日志都要你去处理，但是日志产生的时间间隔比较长**

如果还用 Unary RPC，那么可能就是不断的去轮询，每隔几百毫秒就去调用一次获取日志的方法。这种轮询方式不仅效率低下，浪费网络和 CPU 资源，而且 UI 的实时性也得不到保证。

我需要的是一种更优雅的模式，就像订阅一份实时消息：我只需要订阅一次，然后只要有新的情况发生，你（Service）就主动把战报推送给我。

这，就是 gRPC **服务端流式 RPC (Server-side Streaming RPC)** 的核心思想。本文，我们将深入实践这种模式，模拟一个日志间隔产生，客户端订阅的方式。
<!-- more -->
## **什么是流 (Streaming)？**

在深入代码之前，我想先谈谈我对“流”的个人理解。

我认为，“流”特别适合处理这样一“大坨”数据，这坨数据有几个特点：

1. **数据的源头不是一次性产生完的**，而是随着时间的推移，源源不断地产生的。
2. 每次产生一小部分数据之间，**存在一定的时间间隔**。
3. **每一小部分独立产生的数据，本身就是有意义的**，可以直接使用或存储。

现在最火的例子就是大语言模型。当模型生成回答时，它不是瞬间完成的，而是一个字一个字或一个词一个词地“思考”和产生。为了让用户能尽快看到内容，而不是干等几十秒，大模型就会通过类似流式服务的技术，将产生的文字一点点地推送到你的聊天窗口里。

这种长连接、持续推送的场景，正是流式服务大展拳脚的地方。

下面是流的简单示意图:
![stream](/images/stream.png)

## **第一步：在 .proto 中定义流式服务**

我们首先需要回到我们的“代码合同”——.proto 文件中，定义一个新的流式服务接口。我们在 Controller 服务中实现一个 GetRealtimeLogs 方法。

关键在于，我们在返回类型 LogEntry 前面加上了 stream 关键字。

**controller.proto**
```protobuf
syntax = "proto3";

package controllerStream;

// 用于请求实时日志的消息
message GetLogsRequest {
  // 我们可以留空，或者未来加入一些过滤条件，比如日志级别
  string level_filter = 1;
}

// 定义单条日志的数据结构
message LogEntry {
  int64 timestamp = 1;
  string level = 2;
  string message = 3;
}

service Controller {
  // 定义一个服务端流式 RPC
  // 接收一个 GetLogsRequest，返回一个 LogEntry 的数据流
  rpc GetRealtimeLogs(GetLogsRequest) returns (stream LogEntry);
}
```

修改完 .proto 文件后，别忘了重新运行 protoc 命令来生成新的 C++ 代码。

## **第二步：服务端实现 —— 成为一个“数据源”**

实现流式服务的服务端，与 Unary RPC 的主要区别在于，方法的第三个参数不再是一个简单的响应对象指针，而是一个**写入器 (Writer)** 对象：grpc::ServerWriter<LogEntry>* writer。

这个 writer 就是我们向客户端推送数据的管道。我们可以通过不断调用 writer->Write(log_entry) 来发送一条条日志。

**server.cc**

```c++
#include <iostream>
#include <memory>
#include <string>
#include <thread> // 用于模拟日志生成
#include <chrono> // 用于 sleep

#include <grpcpp/grpcpp.h>
#include "grpc_gen/controllerStream.grpc.pb.h"

class ControllerServiceImpl final : public controllerStream::Controller::Service {
    // 实现 GetRealtimeLogs 方法
    grpc::Status GetRealtimeLogs(grpc::ServerContext* context,
                                 const controllerStream::GetLogsRequest* request,
                                 grpc::ServerWriter<controllerStream::LogEntry>* writer) override {

        std::cout << "Client subscribed for logs..." << std::endl;

        // 模拟一个持续产生日志的场景
        for (int i = 1; i <= 10; ++i) {
            // 错误处理：检查客户端是否已经取消了请求（例如，关闭了UI）
            if (context->IsCancelled()) {
                std::cout << "Client cancelled the request. Stopping the log stream." << std::endl;
                break;
            }

            controllerStream::LogEntry log_entry;
            log_entry.set_timestamp(std::chrono::system_clock::to_time_t(std::chrono::system_clock::now()));
            log_entry.set_level("INFO");
            log_entry.set_message("This is log message number " + std::to_string(i));

            // 通过 writer 将这条日志发送给客户端
            if (!writer->Write(log_entry)) {
                // 错误处理：如果写入失败（例如，连接断开），就退出循环
                std::cout << "Failed to write log to stream. Connection might be broken." << std::endl;
                break;
            }

            // 暂停一秒，模拟日志产生的间隔
            std::this_thread::sleep_for(std::chrono::seconds(1));
        }

        std::cout << "Finished sending logs." << std::endl;
        // 当函数返回时，gRPC 会自动关闭流，并向客户端发送一个 OK 状态，
        // 表示“我说完了，一切正常”。
        return grpc::Status::OK;
    }
};

void RunServer()
{
    std::string server_address("0.0.0.0:50051");
    ControllerServiceImpl service;

    grpc::ServerBuilder builder;
    builder.AddListeningPort(server_address, grpc::InsecureServerCredentials());
    builder.RegisterService(&service); // 注册我们的服务实现

    std::unique_ptr<grpc::Server> server(builder.BuildAndStart()); // 构建并启动服务器
    std::cout << "Server listening on " << server_address << std::endl;

    server->Wait(); // 阻塞等待服务器关闭
}

int main(int argc, char **argv)
{
    RunServer();
    return 0;
}
```
下面是流服务产生数据的过程:
![server](/images/server.png)
在真实的项目中，我们不会在 RPC 处理函数里用一个循环来阻塞线程。更优雅的方式是，将 writer 对象保存起来，然后由 Service 层的其他部分（比如一个真正的日志系统）在产生新日志时，异步地调用这个 writer 的 Write 方法。

## **第三步：客户端实现 —— 成为一个“订阅者”**

客户端的实现同样发生了变化。调用流式服务的 Stub 方法，不再直接返回一个 Status 和响应对象，而是返回一个**读取器 (Reader)** 对象：std::unique_ptr<grpc::ClientReader<controller::LogEntry>>。

我们需要在一个循环中不断调用 reader->Read(&log_entry)，直到它返回 false，这表示服务端已经关闭了数据流。

**client.cc**
```c++
#include <iostream>
#include <memory>
#include <string>

#include <grpcpp/grpcpp.h>
#include "grpc_gen/controllerStream.grpc.pb.h"

class ControllerClient
{
public:
    ControllerClient(std::shared_ptr<grpc::Channel> channel)
        : stub_(controllerStream::Controller::NewStub(channel))
    {
    }

    void GetRealtimeLogs()
    {
        controllerStream::GetLogsRequest request;

        controllerStream::LogEntry log_entry;
        grpc::ClientContext context;

        // 调用流式 RPC，获取一个 reader
        std::unique_ptr<grpc::ClientReader<controllerStream::LogEntry> > reader(
            stub_->GetRealtimeLogs(&context, request));

        std::cout << "Subscribed to log stream. Waiting for logs..." << std::endl;

        // 在循环中不断读取从服务端推送来的日志
        while (reader->Read(&log_entry))
        {
            std::cout << "[LOG RECEIVED] "
                    << "Time: " << log_entry.timestamp()
                    << ", Level: " << log_entry.level()
                    << ", Message: " << log_entry.message() << std::endl;
        }

        // 当 Read() 返回 false 后，通过 Finish() 获取最终的状态
        grpc::Status status = reader->Finish();
        // 错误处理：检查最终状态
        if (status.ok())
        {
            std::cout << "Log stream finished cleanly." << std::endl;
        }
        else
        {
            std::cout << "RPC failed with error: " << status.error_code() << ": " << status.error_message() <<
                    std::endl;
        }
    }

private:
    std::unique_ptr<controllerStream::Controller::Stub> stub_;
};

int main(int argc, char **argv)
{
    ControllerClient client(grpc::CreateChannel(
        "localhost:50051", grpc::InsecureChannelCredentials()));

    std::cout << "n--- Calling GetRealtimeLogs ---" << std::endl;
    client.GetRealtimeLogs();

    return 0;
}

```

下面是整个调用过程的时序图:
![client](/images/client.png)
## **关于错误处理的初步思考**

从上面的代码中，我们可以看到 gRPC 在流式通信中处理错误的基本方式：

* **服务端：**
    * 可以通过检查 context->IsCancelled() 来判断客户端是否已经主动断开了连接。
    * writer->Write() 的返回值可以判断写入是否成功。如果返回 false，通常意味着底层的连接已经出问题了。
* **客户端：**
    * reader->Read() 循环结束后，必须调用 reader->Finish() 来获取整个流的最终状态。
    * 如果 status.ok() 为 true，表示流正常结束。
    * 如果为 false，则表示流是因错误而中断的，我们可以从 status 对象中获取错误码和错误信息。

这只是最基础的错误处理。在真实的健壮应用中，我们还需要考虑网络抖动、超时、重试等更复杂的策略，这些都是后续可以深入的话题。

## **总结与展望**

本文我们成功地解决了项目中的一个核心需求——**实时数据推送**。通过学习和实践 gRPC 的服务端流式 RPC，我们已经从“一问一答”的通信模式，迈向了更灵活、更高效的“实时订阅”模式。

我们深入了解了如何在 .proto 中定义流式服务，以及如何在 C++ 中实现流的服务端（ServerWriter）和客户端（ClientReader）。

在下一篇文章中，我们将讲解 **在Qt应用中如何直接集成Grpc**

**最后，再次强调：** 本系列文章是我个人的学习笔记和思考，而 gRPC 的官方文档和 grpc/grpc GitHub 仓库中的例子是最好的、最权威的学习资料。**官方文档和示例远比我写得好，强烈建议大家去阅读！** 希望我的文章能作为一个有益的补充和不同的视角。