---
title: "grpc实践之路:03.Qt与grpc的集成使用"
date: 2025-07-08
cover: /images/cover/grpc_cover.jpeg
categories: 
  - 动手实践-三方库
  - grpc
tags:
  - C++
  - grpc
  - 进程间通信
---
## **前言**

在之前的文章中，我们从一个Qt应用的角度出发，探讨了怎么使用grpc作为进程间通信的手段，进行点对点的调用，并且讲解了怎么处理服务端产生的**流**数据。
但是，既然都没涉及到Qt相关的内容，本篇文章就讲解下在Qt中怎么去使用Grpc作为客户端的集成。

Qt作为一个主要应用在GUI中的应用框架，在其中最大的挑战就是：**如何避免阻塞 UI 主线程**。当然可以利用QThread和信号槽的机制去解决这个问题，
但是却有些繁琐，需要我们自己去进行线程管理和对象的生命周期。但是，好消息是自动 Qt6.9版本之后，**Qt 官方已经正式提供了对 gRPC 的原生支持** (Qt Grpc 模块)。这意味着我们不再需要手动创建线程来包装
gRPC 调用了。Qt 将 gRPC与它核心的事件循环和信号槽机制完美地融合在了一起。

本文，我们将探索这个“官方解决方案”，看看如何利用 Qt Grpc 模块，以一种前所未有的、极其优雅的方式，在 Qt 应用中实践 gRPC 通信。

<!--more-->
## **准备工作：CMake 与 Qt gRPC 模块**

要使用 Qt 官方的 gRPC 支持，首先你需要确保在安装 Qt 时，已经勾选并安装了 Qt Protobuf 和 Qt Grpc 这两个模块，并且安装好了Grpc和Protobuf。

并且Qt官方也提供了Grpc的函数去处理grpc的原生命令

**CMakeLists.txt**
```cmake
cmake_minimum_required(VERSION 3.16)  
project(MyGrpcQtProject)

#配置你的项目,保证先安装了grpc和protobuf

# 找到 Qt6，并确保 Protobuf 和 Grpc 组件可用  
find_package(Qt6 REQUIRED COMPONENTS Protobuf Grpc)  
qt_standard_project_setup()

# 添加你的可执行文件  
qt_add_executable(MyApp main.cpp clientguide.cpp) # 假设你的客户端逻辑在 clientguide.cpp

# Qt 提供的 magic command，自动处理 .proto 文件！  
# 它会调用 protoc 和 grpc_cpp_plugin 生成 Qt 风格的代码  
qt_add_grpc(MyApp CLIENT # 或者 SERVER, BIDI  
PROTO_FILES  
path/to/your/service.proto  
)

# 链接到 Qt 提供的库  
target_link_libraries(MyApp PRIVATE Qt6::Protobuf Qt6::Grpc)
```

qt_add_grpc 这个命令了它会自动处理代码生成和编译，也不需要去手动处理proto文件的生成c代码了

## **代码生成**

我们还是使用之前类似的 .proto 文件，但这次，由 qt_add_grpc 生成的文件会有些不同，它们是为 Qt 量身定做的。

**clientguide.proto**

```protobuf
syntax = "proto3";  
package client.guide; // enclosing namespace

message Request { /* ... */ }  
message Response { /* ... */ }

service ClientGuideService {  
rpc UnaryCall (Request) returns (Response);  
rpc ServerStreaming (Request) returns (stream Response);
}

```

生成的文件名会带有 .qpb.h 和 .grpc.qpb.h 的后缀。它们不仅包含了 Protobuf 消息的 C++ 类，更重要的是，生成的 gRPC Stub 和
Service 基类，其所有方法都**原生返回 Qt 的异步对象**，并**通过信号槽来通知结果**。

下面是生成Qt grpc中源码生成的示意图
![qt_grpc_plugin](/images/grpc/qt_add_grpc.png)

## **实践：一个纯 Qt 风格的体验**

现在，让我们看看 Qt 官方示例中的客户端代码是如何利用这些新工具的。

**clientguide.cpp (核心逻辑)**

### **1. 初始化：创建 Channel 和 Client Stub**

```c++
//引入Qt生成的头文件
#include "clientguide.qpb.h"
#include "clientguide_client.grpc.qpb.h"
//! [gen-includes]

#include <QtGrpc/QGrpcHttp2Channel>
#include <QtGrpc/QGrpcServerStream.h>

#include <QtCore/QCommandLineParser>
#include <QtCore/QCoreApplication>
#include <QtCore/QDateTime>
#include <QtCore/QProcess>
#include <QtCore/QThread>
#include <QtCore/QUrl>

#include <limits>
#include <memory>

using namespace client;

void startServerProcess();
QDebug operator<<(QDebug debug, const guide::Response &response);

class ClientGuide : public QObject
{
public:
    explicit ClientGuide(std::shared_ptr<QAbstractGrpcChannel> channel)
    {
        // 将 Qt 的 Channel 附加到 Qt 生成的 Client Stub 上 
        // 这一步和原生 gRPC 类似，但 Channel 和 Client 都是 Qt gRPC 提供的类型。
        m_client.attachChannel(std::move(channel));
    }

    static guide::Request createRequest(int32_t num, bool fail = false)
    {
        guide::Request request;
        request.setNum(num);
        request.setTime(fail ? std::numeric_limits<int64_t>::max()
                             : QDateTime::currentMSecsSinceEpoch());
        return request;
    }
   
    void unaryCall(const guide::Request &request)
    {
        std::unique_ptr<QGrpcCallReply> reply = m_client.UnaryCall(request);
        const auto *replyPtr = reply.get();
        QObject::connect(
            replyPtr, &QGrpcCallReply::finished, replyPtr,
            [reply = std::move(reply)](const QGrpcStatus &status) {
                if (status.isOk()) {
                    if (const auto response = reply->read<guide::Response>())
                        qDebug() << "Client (UnaryCall) finished, received:" << *response;
                    else
                        qDebug("Client (UnaryCall) deserialization failed");
                } else {
                    qDebug() << "Client (UnaryCall) failed:" << status;
                }
            },
            Qt::SingleShotConnection);
    }

    void serverStreaming(const guide::Request &initialRequest)
    {
        std::unique_ptr<QGrpcServerStream> stream = m_client.ServerStreaming(initialRequest);
        const auto *streamPtr = stream.get();

        QObject::connect(
            streamPtr, &QGrpcServerStream::finished, streamPtr,
            [stream = std::move(stream)](const QGrpcStatus &status) {
                if (status.isOk())
                    qDebug("Client (ServerStreaming) finished");
                else
                    qDebug() << "Client (ServerStreaming) failed:" << status;
            },
            Qt::SingleShotConnection);

        QObject::connect(streamPtr, &QGrpcServerStream::messageReceived, streamPtr, [streamPtr] {
            if (const auto response = streamPtr->read<guide::Response>())
                qDebug() << "Client (ServerStream) received:" << *response;
            else
                qDebug("Client (ServerStream) deserialization failed");
        });
    }
private:
    guide::ClientGuideService::Client m_client;
};

int main(int argc, char *argv[])
{
    QCoreApplication app(argc, argv);

    QCommandLineParser parser;
    QCommandLineOption enableUnary("U", "Enable UnaryCalls");
    QCommandLineOption enableSStream("S", "Enable ServerStream");

    parser.addHelpOption();
    parser.addOption(enableUnary);
    parser.addOption(enableSStream);
    parser.process(app);

    bool defaultRun = !parser.isSet(enableUnary) && !parser.isSet(enableSStream)
        && !parser.isSet(enableCStream) && !parser.isSet(enableBStream);

    qDebug("Welcome to the clientguide!");
    qDebug("Starting the server process ...");
    startServerProcess();

    //! [basic-0]
    auto channel = std::make_shared<QGrpcHttp2Channel>(
        QUrl("http://localhost:50056")
        /* without channel options. */
    );
    ClientGuide clientGuide(channel);
    
    if (defaultRun || parser.isSet(enableUnary)) {
        clientGuide.unaryCall(ClientGuide::createRequest(1));
        clientGuide.unaryCall(ClientGuide::createRequest(2, true)); // fail the RPC
        clientGuide.unaryCall(ClientGuide::createRequest(3));
    }

    if (defaultRun || parser.isSet(enableSStream)) {
        clientGuide.serverStreaming(ClientGuide::createRequest(3));
    }

    return app.exec();
}
```

### **2. Unary RPC (点对点调用)**

Qt生成的客户端可以让我们直接利用信号槽的机制，不必为了避免阻塞UI，自动手动处理线程

```c++
void ClientGuide::unaryCall(const guide::Request &request)  
{  
// 调用 RPC 方法，不再返回 Status，而是返回一个 QGrpcCallReply 指针！  
std::unique_ptr<QGrpcCallReply> reply = m_client.UnaryCall(request);

    // QGrpcCallReply 是一个 QObject，我们可以连接它的 finished 信号！  
    QObject::connect(reply.get(), &QGrpcCallReply::finished, this,  
        // Lambda 表达式作为槽函数  
        [reply = std::move(reply)](const QGrpcStatus &status) {  
            // 这个 Lambda 将在 Qt 的事件循环中被安全地调用，不会阻塞  
            if (status.isOk()) {  
                if (const auto response = reply->read<guide::Response>())  
                    qDebug() << "Client (UnaryCall) finished, received:" << *response;  
            } else {  
                qDebug() << "Client (UnaryCall) failed:" << status;  
            }  
        });  

}
```

m_client.UnaryCall 立即返回一个 QGrpcCallReply 对象，**完全不会阻塞**。我们只需要连接它的 finished
信号，就可以在未来的某个时刻，当 RPC 调用完成时，在槽函数中处理结果。整个过程是**纯异步、事件驱动**的，完美融入了 Qt 的体系。

下面的是调用过程的示意图:
![qt_client](/images/grpc/qt_client.png)
### **3. Server Streaming (服务端流)**

处理流式数据也变得异常简单。调用流式 RPC 方法会返回一个 QGrpcServerStream 对象。

```c++
void ClientGuide::serverStreaming(const guide::Request &initialRequest)  
{  
// 调用流式 RPC，返回一个 QGrpcServerStream 指针  
std::unique_ptr<QGrpcServerStream> stream = m_client.ServerStreaming(initialRequest);  
const auto *streamPtr = stream.get();

    // 连接 finished 信号，处理流结束事件  
    QObject::connect(streamPtr, &QGrpcServerStream::finished, this,  
        [stream = std::move(stream)](const QGrpcStatus &status) {  
            if (status.isOk())  
                qDebug("Client (ServerStreaming) finished");  
            else  
                qDebug() << "Client (ServerStreaming) failed:" << status;  
        });  
      
    // 连接 messageReceived 信号，处理每一条从服务端推送来的消息！  
    QObject::connect(streamPtr, &QGrpcServerStream::messageReceived, this, [streamPtr] {  
        if (const auto response = streamPtr->read<guide::Response>())  
            qDebug() << "Client (ServerStream) received:" << *response;  
    });  

}

```

我们无需编写 while 循环读取响应数据了，取而代之的是连接 messageReceived
信号。每当服务端推送一条新消息，这个信号就会被发射一次，我们的槽函数就会被调用一次。这正是我们梦寐以求的、真正的事件驱动的流式数据处理方式！

## **个人思考：Qt gRPC 封装的优雅之处**

Qt的接口设计和使用方式确实有很多可取之处,Qt gRPC 模块的封装，堪称教科书级别的“框架集成”：

1. **使用风格的统一：** 将grpc的调用过程统一成信号槽的方式与Qt的控件使用完全一致。
2. **原生信号槽的胜利：** 整个 gRPC 的生命周期——发起调用、收到消息、调用结束——都被映射成了 Qt 的信号。这使得 gRPC
   的异步事件能像按钮点击、鼠标移动一样，被无缝地集成到 Qt 的事件循环中。
3. **使用更加简单：** 开发者不再需要关心线程问题，只需关注业务逻辑。Qt 将这一切都隐藏在了
   QGrpcCallReply 和 QGrpcServerStream 对象的背后，提供了极其简洁和符合直觉的 API。

## **总结与展望**

本文，我们探索了 Qt 官方提供的 Qt Grpc 模块，见证了它如何将 gRPC 的强大功能与 Qt 优雅的信号槽机制完美结合。通过使用
QGrpcCallReply 和 QGrpcServerStream，我们以一种纯异步、事件驱动的方式，轻松地实现了 Unary 和 Streaming RPC，彻底解决了在 GUI
应用中进行网络通信的核心痛点。

这无疑是 Qt C++ 开发者的一大福音。它极大地降低了在桌面和移动应用中使用 gRPC 的门槛，让我们能更专注于创造富有价值的应用本身。
