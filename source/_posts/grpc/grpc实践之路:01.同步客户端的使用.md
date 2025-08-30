---
title: "grpc实践之路:01.同步客户端的使用"
date: 2025-06-30
categories: 
  - 动手实践-三方库
  - grpc
tags:
  - C++
  - grpc
  - 进程间通信
---
## **前言：**

因为最近的个人需要,想要自己多做一点实践，因此我决定 ：**开发一个带 GUI 的、用于管理和监控一个外部核心服务 (Core Service) 的跨平台桌面应用。** 

在我的设想中，这个应用的架构是分层的：UI 层（我选择了 Qt）负责界面展示和用户交互，而 Service 层（我将用 C++ 实现）则负责与那个外部核心服务进行通信，并处理所有复杂的后端逻辑。

很自然，我决定将 UI 和 Service 设计成两个独立的进程。这样不仅能让架构更清晰、权责更分明，也能避免任何一端的崩溃影响到另一端。于是，第一个核心问题就摆在了面前：**UI 进程和 Service 进程之间，该如何通信？**
<!-- more -->
## **思考：我的进程间通信（IPC）技术选型**

常见的 IPC 手段有很多，比如命名管道、共享内存、套接字，或者更高层的 RESTful API。因为我之前在公司的项目中写过 IPC 相关的代码（虽然不是 gRPC），并且我设想的这个“外部核心服务”有**大量实时状态需要流式传输**给 UI，所以我很快将目光锁定在了更现代的 RPC 框架上。

最终，我选择了 gRPC。理由如下：

1. **流式传输 (Streaming) 是刚需：** 这是决定性因素。我需要将 Service 层的实时日志和连接状态源源不断地推送到 UI。gRPC 原生支持服务端流和双向流，这完美地解决了我的核心痛点。
2. **跨语言的未来：** 虽然现在 UI 和 Service 都是 C++ 生态（Qt + C++），但 gRPC 的跨语言特性给了我极大的灵活性。未来，如果我想把 UI 换成其他语言实现，或者想用 Python/Go 写一些简单的测试脚本来调用我的 Service，gRPC 都能轻松应对。
3. **性能与效率：** 基于 HTTP/2 和 Protocol Buffers，gRPC 在性能和数据压缩方面都优于传统的 REST+JSON 组合。
4. **强类型契约：** 通过 .proto 文件定义服务，避免了手写协议和序列化的繁琐与易错。

这个系列文章，就是我对自己学习和实践 gRPC 过程的笔记与复盘。我计划把如何简单地使用 gRPC 的**同步模式、流模式**，以及**如何与 Qt 框架优雅地集成**都记录下来。后续如果精力允许，可能还会去剖析一些 gRPC 的核心组件内部原理。

## **准备工作：gRPC 的安装**

在开始之前，有一个非常重要的建议：**gRPC 的手动编译安装有些麻烦，强烈建议使用 vcpkg 这个包管理器来安装。**

vcpkg 是微软推出的 C++ 包管理器，可以极大地简化第三方库的安装和集成过程。你只需要执行简单的命令，它就会自动帮你下载源码、处理依赖、编译并安装好 gRPC 及其所需的 protobuf, openssl, zlib 等所有依赖项。

```bash
# 克隆 vcpkg  
git clone https://github.com/microsoft/vcpkg.git  
cd vcpkg

# 运行引导脚本  
./bootstrap-vcpkg.sh # Linux / macOS  
# 或者 .bootstrap-vcpkg.bat (Windows)

# 安装 gRPC  
./vcpkg install grpc
```

安装完成后，你可以通过 CMake 的 toolchain 文件非常方便地在你的项目中使用它。**不要自己手动去编译，相信我，这会为你省下大量的时间和精力，让你能专注于 gRPC 本身。**

## **“Hello, gRPC!”：核心使用流程**

下面，我们来走一遍 gRPC 的核心使用流程，实现一个最简单的客户端-服务端通信。

### **第一步：定义“代码合同” (.proto 文件)**

gRPC 的一切都始于 .proto 文件，它使用 Protocol Buffers 语法来定义服务接口和消息结构。

对于我的项目，我先定义一个最简单的服务 Controller，它只有一个“点对点”的 RPC 调用，用于获取核心服务的版本号。

**controller.proto**
```c++
// 指定使用 proto3 语法
syntax = "proto3";

// 定义包名，避免命名冲突
package controller;

// GetVersion 服务的请求消息
message GetVersionRequest {
  // 消息字段，这里为空，因为获取版本不需要参数
}

// GetVersion 服务的响应消息
message GetVersionResponse {
  string version = 1; // 1 是字段的唯一编号，不是值
}

// 定义我们的核心服务
service Controller {
  // 定义一个 Unary RPC (一元 RPC，即点对点调用)
  // 方法名叫 GetVersion，接收 GetVersionRequest，返回 GetVersionResponse
  rpc GetVersion(GetVersionRequest) returns (GetVersionResponse);
}
```

### **第二步：生成代码与文件解析**

定义好 .proto 文件后，我们使用 protoc 编译器和 gRPC C++ 插件来生成 C++ 代码。

# 假设你的 .proto 文件在 protos 目录下，输出到当前目录  
protoc -I=./protos --cpp_out=./grpc_gen --grpc_out=./grpc_gen --plugin=protoc-gen-grpc=`which grpc_cpp_plugin` ./protos/controller.proto

当你使用vcpkg安装grpc的时候，你会将 `protos`工具和 `protoc-gen-grpc`工具一并安装，安装目录一般在 `/home/username/.vcpkg/vcpkg/installed/x64-linux/tools` 的 `protobuf`目录和 `grpc`目录下


执行后，会生成四个关键文件：

* **controller.pb.h / controller.pb.cc：** 这两个文件是 **Protocol Buffers** 生成的。它们包含了你在 .proto 中定义的 message（如 GetVersionRequest）所对应的 C++ 类，以及这些类的序列化和反序列化方法。**它们只负责数据的载体。**
* **controller.grpc.pb.h / controller.grpc.pb.cc：** 这两个文件是 **gRPC 插件**生成的。它们包含了你在 .proto 中定义的 service（如 Controller）相关的 C++ 代码。这里面有：
  * 一个用于**服务端实现**的抽象基类 (Controller::Service)。
  * 一个用于**客户端调用**的桩代码类 (Controller::Stub)。

理解这四个文件的分工非常重要，它体现了 gRPC 的分层设计：数据层 (Protobuf) 和通信层 (gRPC) 是分离的。

### **第三步：编写服务端**

我们需要创建一个类，继承自生成的 Controller::Service，并重写 .proto 中定义的 GetVersion 虚函数。

**server.cc**
```c++
#include <iostream>
#include <memory>
#include <string>

#include <grpcpp/grpcpp.h>
#include "grpc_gen/controller.grpc.pb.h" // 引入 gRPC 生成的头文件

// 继承自生成的服务基类
class ControllerServiceImpl final : public controller::Controller::Service
{
    // 重写 GetVersion 方法
    grpc::Status GetVersion(grpc::ServerContext *context,
                            const controller::GetVersionRequest *request,
                            controller::GetVersionResponse *response) override
    {
        std::string version_str = "Core Service v1.0.0";
        std::cout << "Received GetVersion request. Responding with: " << version_str << std::endl;

        response->set_version(version_str); // 设置响应消息
        return grpc::Status::OK; // 返回 OK 状态
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

### **第四步：编写客户端**

客户端通过生成的 Controller::Stub 来调用远程方法，这个过程被封装得就像调用本地函数一样简单。

**client.cc**
```c++
#include <iostream>
#include <memory>
#include <string>

#include <grpcpp/grpcpp.h>
#include "grpc_gen/controller.grpc.pb.h" // 同样引入

class ControllerClient
{
public:
    ControllerClient(std::shared_ptr<grpc::Channel> channel)
        : stub_(controller::Controller::NewStub(channel))
    {
    } // 通过 Channel 创建 Stub

    std::string GetVersion()
    {
        controller::GetVersionRequest request;
        controller::GetVersionResponse response;
        grpc::ClientContext context;

        // 发起 RPC 调用，就像调用一个本地方法
        grpc::Status status = stub_->GetVersion(&context, request, &response);

        if (status.ok())
        {
            return response.version();
        }
        else
        {
            std::cout << status.error_code() << ": " << status.error_message() << std::endl;
            return "RPC failed";
        }
    }

private:
    std::unique_ptr<controller::Controller::Stub> stub_;
};

int main(int argc, char **argv)
{
    std::string target_str = "localhost:50051";
    // 创建一个到服务端的 Channel
    ControllerClient client(grpc::CreateChannel(target_str, grpc::InsecureChannelCredentials()));

    std::string version = client.GetVersion();
    std::cout << "Controller version received: " << version << std::endl;

    return 0;
}
```

### **第五步：运行！**

先启动服务端，再启动客户端，你就能看到客户端成功获取到了服务端返回的版本号信息。一个完整的 RPC 调用就这么简单地完成了！

下面就是这个服务的简单调用时序图
![version_call](/images/grpc/version_call.png)


## **总结与展望**

本文作为 gRPC 系列的开篇，我们从一个真实的个人项目需求出发，探讨了为什么选择 gRPC 作为我们的 IPC 方案。接着，我们通过定义一个简单的 .proto 文件和实现一个完整的“Hello, gRPC!”示例，迈出了实践的第一步。

我们看到，gRPC 借助 Protocol Buffers 和代码生成，极大地简化了网络通信应用的开发，让我们能更专注于业务逻辑本身。

当然，我们目前只接触了最简单的 Unary RPC 和同步调用。这只是冰山一角。在接下来的文章中，我们将深入探讨：

* **如何利用 gRPC 强大的流式通信能力，实现实时数据推送？**
* **如何在 GUI 应用（如 Qt）中优雅地使用 gRPC，避免界面卡死？**
* **服务端和客户端的核心组件是如何工作的？**

**最后，有一个强烈的建议：** 本系列文章是我个人的学习笔记和思考，而 gRPC 的官方文档和 GitHub 仓库中的例子是最好的、最权威的学习资料。**官方文档和示例远比我写得好，强烈建议大家去阅读！** 希望我的文章能作为一个有益的补充和不同的视角。