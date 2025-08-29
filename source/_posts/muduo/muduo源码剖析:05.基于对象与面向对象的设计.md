
---
title: "muduo源码剖析:05.基于对象与面向对象的设计"
date: 2025-08-27
categories: 
  - 源码分析
  - muduo
tags:
  - C++
  - muduo
  - 网络库
---
## 前言：

在前几篇文章中，我们依次剖析了：

* `EventLoop` 的单线程事件循环模型（One Loop Per Thread）
* `TcpConnection` 的连接生命周期与“三个半事件”机制
* `TimerQueue` 在事件驱动框架下的使用
* `Buffer` 模块在高性能数据处理中的角色

至此，我们已经基本掌握了陈硕大佬在《Linux 多线程服务端编程》中提出的网络编程核心思想及其在 muduo 中的具象实现。

在这篇文章中，我们不再分析具体功能的实现细节，而是谈一谈陈硕大佬对 C++ 进行开发的一个理念：**muduo 并未采用传统的“面向对象”编程风格，而是秉持“基于对象”的理念进行系统设计。**

我们要具体分析这两者在使用上，功能实现上的区别。
<!-- more -->
---

## 使用场景对比：基于对象 vs 面向对象

我们首先回顾 muduo 中的一个经典的 `EchoServer` 示例：

```cpp
class EchoServer {
public:
  EchoServer(muduo::net::EventLoop* loop, const muduo::net::InetAddress& listenAddr)
    : server_(loop, listenAddr, "EchoServer") {
    server_.setConnectionCallback(
      std::bind(&EchoServer::onConnection, this, std::placeholders::_1));
    server_.setMessageCallback(
      std::bind(&EchoServer::onMessage, this, std::placeholders::_1, std::placeholders::_2, std::placeholders::_3));
  }

  void start() { server_.start(); }

private:
  void onConnection(const muduo::net::TcpConnectionPtr& conn);
  void onMessage(const muduo::net::TcpConnectionPtr& conn, muduo::net::Buffer* buf, muduo::Timestamp time);
  muduo::net::TcpServer server_;
};
```

陈硕大佬并未提供一个 `TcpConnectionListener` 的抽象接口让用户去继承实现 `onConnect` / `onMessage`，而是允许用户**通过组合对象、注册回调函数**的方式，将自己的逻辑注入库中。

### 对比：常见的面向对象的网络库实现

如果有 C# 和 Java 的开发经验，可能非常熟悉下面的模式：

```cpp
class SocketEventListener {
public:
  virtual void handleRead() = 0;
  virtual void handleClose() = 0;
};

class MyConnection : public SocketEventListener {
public:
  void handleRead() override {
    // handle logic
  }

  void handleClose() override {
    // clean up
  }
};
```

这种做法的本意是具体约束行为，让使用者去处理一个网络连接的完整生命周期，但也带来以下问题：

* 实现门槛高：即便只是写个简单 demo，也需实现多个空函数
* 接口变更代价大：增加一个虚函数会影响所有子类
* 无法真正约束开发者的设计质量: 很多使用者只是简单的打下日志

---

## 基于对象的设计：以 Channel 为例

在 muduo 中，`Channel` 是绑定文件描述符与事件处理器的核心组件。其应用遍布各个模块：

### 服务端监听

```cpp
class Acceptor : noncopyable {
  EventLoop* loop_;
  Socket acceptSocket_;
  Channel acceptChannel_;
  NewConnectionCallback newConnectionCallback_;
  bool listenning_;
  int idleFd_;
};
```

### 客户端连接处理

```cpp
class TcpConnection : noncopyable, public std::enable_shared_from_this<TcpConnection> {
  std::unique_ptr<Socket> socket_;
  std::unique_ptr<Channel> channel_;
  const InetAddress localAddr_;
  const InetAddress peerAddr_;
  ConnectionCallback connectionCallback_;
  MessageCallback messageCallback_;
  WriteCompleteCallback writeCompleteCallback_;
  HighWaterMarkCallback highWaterMarkCallback_;
  CloseCallback closeCallback_;
  size_t highWaterMark_;
  Buffer inputBuffer_;
  Buffer outputBuffer_;
  std::any context_;
};
```

### 唤醒机制

```cpp
class EventLoop : noncopyable {
  int wakeupFd_;
  std::unique_ptr<Channel> wakeupChannel_;
  std::any context_;
  ChannelList activeChannels_;
  Channel* currentActiveChannel_;
  mutable MutexLock mutex_;
  std::vector<Functor> pendingFunctors_;
};
```

### 定时器

```cpp
class TimerQueue : noncopyable {
  const int timerfd_;
  Channel timerfdChannel_;
  TimerList timers_;
  ActiveTimerSet activeTimers_;
  bool callingExpiredTimers_; 
  ActiveTimerSet cancelingTimers_;
};
```

### Channel 的统一处理方式

```cpp
channel->setReadCallback(std::bind(&TcpConnection::handleRead, this));
channel->setWriteCallback(std::bind(&TcpConnection::handleWrite, this));
```

这就是“基于对象”的关键点：**不搞子类体系，而用统一类 + 函数注入来处理多样行为。**

---

## 回调机制的优越性：组合优于继承

muduo 的核心组件几乎都是：**具体类 + 回调注入**

* `TcpServer` 组合 `Acceptor`、`EventLoopThreadPool`、用户回调
* `TcpConnection` 内部注册回调响应事件
* `Channel` 统一管理事件分发

**类图示意：**

![TcpServer](/images/TcpServer.png)

**组合优于继承，行为通过注入，而非继承重写。**

```cpp
TcpConnection::TcpConnection(...) {
  channel_->setReadCallback(std::bind(&TcpConnection::handleRead, this));
}
```

对比传统 OOP：

```cpp
class BaseHandler {
  virtual void onRead() = 0;
};

class MyHandler : public BaseHandler {
  void onRead() override { ... }
};
```

muduo 的方式降低抽象复杂度，提升扩展灵活性。

---

## 生命周期管理：RAII + 智能指针 + tie()

陈硕大佬强调：C++ 的最大优势是**确定性析构**。muduo 的生命周期管理机制如下：

* 所有 socket 封装于 RAII 类
* `TcpConnection` 使用 `shared_ptr` + `enable_shared_from_this`
* `Channel::tie()` 实现弱引用锁定，防止悬垂指针

```cpp
void TcpConnection::connectEstablished() {
  channel_->tie(shared_from_this());
}
```

事件处理过程：

```cpp
void Channel::handleEvent(...) {
  std::shared_ptr<void> guard;
  if (tied_) {
    guard = tie_.lock();
    if (guard) handleEventWithGuard(...);
  } else {
    handleEventWithGuard(...);
  }
}
```

**时序图：**

![RAII](/images/raii.png)

保障了：

* 对象析构后无悬垂调用
* 生命周期由 `shared_ptr` 统一管理，清晰可控

---

## 尾声：从 muduo 中我们应该学些什么？对面向对象的一点思考

关于面向对象的争议其实一直很大，批评也很多，但是面向对象这种思想既然可以存在，并且依然得到哪么多人的拥护，肯定有可取之处。

咱们先关心从muduo源码中我们应该学点什么知识。

muduo 告诉我们：

* RAII + 智能指针是 C++ 的精髓
* 无虚函数，回调更灵活
* 强调组合，提升模块化与可维护性

我对使用面向对象想法是：

> “让行业大佬设计接口，让普通开发者以简单朴实的方式逐步演进。”

普通开发者：

* 从组合和回调入手，快速实现业务逻辑
* 在需求稳定后，再抽象出通用接口
* 通过测试保证软件质量

这种自顶向下的设计方式，其实很适合那些具有非常丰富行业经验的人去设计出一套接口来。
因为他们实际上知道了，这套接口就是足够好的抽象。
像普通的开发人员，就做到简单，朴实就好了。

最后：
**设计应服务于目标，而非固守范式。**

作为程序员，应该具备多种设计手法，并在适当时刻选择最合适的方式。

