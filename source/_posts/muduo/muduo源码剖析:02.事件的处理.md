---
title: "muduo源码剖析:02.事件的处理"
date: 2025-05-31
cover: /images/cover/muduo_cover.png
categories: 
  - 源码分析
  - muduo
tags:
  - C++
  - muduo
  - 网络库
---
## **前言**

在上一篇文章《muduo 源码剖析（一）：深入 'One Loop Per Thread' 与 EventLoop 的实现》中，我们分析了 muduo 如何通过 EventLoopThreadPool 和 EventLoopThread 实现 "One Loop Per Thread" 的并发模型，以及 EventLoop 如何通过 eventfd 机制支持跨线程的任务提交。这些是 muduo 事件驱动框架的基石。

本文我们将聚焦于网络编程的核心——TCP 连接，以及陈硕大佬提出的“三个半事件”中的核心部分：**新连接的建立与分发、连接上数据的接收与发送、以及连接的关闭与资源回收**。我们将深入探讨在 muduo 中，这些事件是如何被优雅地融入其基于对象的事件驱动框架中的。
<!-- more -->
## **EventLoop::loop()：事件处理的核心驱动**

在深入具体事件之前，我们首先需要理解 EventLoop 的主循环 loop() 是如何驱动事件处理的。其核心逻辑非常直观：

1. 调用 Poller::poll() (底层通常是 epoll_wait) 等待 I/O 事件的发生，或者等待被 wakeupFd_ 唤醒。
2. 获取到活跃的 Channel 列表 (activeChannels_)。
3. 遍历 activeChannels_，对每个 Channel 调用其 handleEvent() 方法。
4. 执行所有通过 runInLoop 或 queueInLoop 提交的待处理任务 (doPendingFunctors())。

```c++
void EventLoop::loop()  
{  
assert(!looping_);  
assertInLoopThread();  
looping_ = true;  
quit_ = false;  // FIXME: what if someone calls quit() before loop() ?  
LOG_TRACE << "EventLoop " << this << " start looping";

while (!quit_)  
{  
activeChannels_.clear();  
// 1. 等待事件发生  
pollReturnTime_ = poller_->poll(kPollTimeMs, &activeChannels_);  
++iteration_;  
if (Logger::logLevel() <= Logger::TRACE)  
{  
printActiveChannels();  
}  
eventHandling_ = true;  
// 2. 处理活跃 Channel 的事件  
for (Channel* channel : activeChannels_)  
{  
currentActiveChannel_ = channel;  
currentActiveChannel_->handleEvent(pollReturnTime_);  
}  
currentActiveChannel_ = NULL;  
eventHandling_ = false;  
// 3. 执行 EventLoop 内部任务队列中的任务  
doPendingFunctors();  
}

LOG_TRACE << "EventLoop " << this << " stop looping";  
looping_ = false;  
}  
```

`Poller` (以 `EPollPoller` 为例) 的 `poll()` 方法负责调用 `epoll_wait`，并将返回的就绪事件填充到 `activeChannels` 列表中。

```c++
// EPollPoller.cc  
Timestamp EPollPoller::poll(int timeoutMs, ChannelList* activeChannels)  
{  
LOG_TRACE << "fd total count " << channels_.size();  
int numEvents = ::epoll_wait(epollfd_,  
&*events_.begin(), // events_ 是 epoll_event 数组  
static_cast<int>(events_.size()),  
timeoutMs);  
int savedErrno = errno;  
Timestamp now(Timestamp::now());  
if (numEvents > 0)  
{  
LOG_TRACE << numEvents << " events happened";  
fillActiveChannels(numEvents, activeChannels); // 将就绪事件转换为 Channel  
if (implicit_cast<size_t>(numEvents) == events_.size()) // 如果 epoll_event 数组满了，则扩容  
{  
events_.resize(events_.size()*2);  
}  
}  
else if (numEvents == 0)  
{  
LOG_TRACE << "nothing happened";  
}  
else // 出错处理  
{  
if (savedErrno != EINTR)  
{  
errno = savedErrno;  
LOG_SYSERR << "EPollPoller::poll()";  
}  
}  
return now;  
}

void EPollPoller::fillActiveChannels(int numEvents,  
ChannelList* activeChannels) const  
{  
assert(implicit_cast<size_t>(numEvents) <= events_.size());  
for (int i = 0; i < numEvents; ++i)  
{  
// 注册时将 Channel* 存放在 epoll_event 的 data.ptr 中  
Channel* channel = static_cast<Channel*>(events_[i].data.ptr);  
// ... (省略 NDEBUG 下的断言检查) ...  
channel->set_revents(events_[i].events); // 设置 Channel 实际发生的事件  
activeChannels->push_back(channel);  
}  
}
``` 

当 EventLoop 获取到活跃的 Channel 后，会调用 Channel::handleEvent()。
这个方法是事件分发的枢纽，它根据 Channel 上实际发生的事件类型 (revents_)，调用相应的回调函数。

```c++
// Channel.cc  
void Channel::handleEvent(Timestamp receiveTime)  
{  
std::shared_ptr<void> guard;  
if (tied_) // 如果 Channel 与某个对象（通常是 TcpConnection）绑定了生命周期  
{  
guard = tie_.lock(); // 尝试获取对象的 shared_ptr  
if (guard) // 对象仍然存活  
{  
handleEventWithGuard(receiveTime);  
}  
// 如果 guard 为空，说明对象已销毁，Channel 不再处理事件  
}  
else // 未绑定生命周期  
{  
handleEventWithGuard(receiveTime);  
}  
}

void Channel::handleEventWithGuard(Timestamp receiveTime)  
{  
eventHandling_ = true;  
LOG_TRACE << reventsToString();  
// 对端关闭连接 (POLLHUP)，并且没有可读数据 (POLLIN)  
if ((revents_ & POLLHUP) && !(revents_ & POLLIN))  
{  
if (logHup_)  
{  
LOG_WARN << "fd = " << fd_ << " Channel::handle_event() POLLHUP";  
}  
if (closeCallback_) closeCallback_(); // 执行关闭回调  
}

if (revents_ & POLLNVAL) // 无效的请求，通常是 fd 已关闭  
{  
LOG_WARN << "fd = " << fd_ << " Channel::handle_event() POLLNVAL";  
}

// 错误事件 (POLLERR) 或无效请求 (POLLNVAL)  
if (revents_ & (POLLERR | POLLNVAL))  
{  
if (errorCallback_) errorCallback_(); // 执行错误回调  
}  
// 可读事件 (POLLIN)、高优先级可读 (POLLPRI)、对端关闭连接且仍有数据可读 (POLLRDHUP)  
if (revents_ & (POLLIN | POLLPRI | POLLRDHUP))  
{  
if (readCallback_) readCallback_(receiveTime); // 执行读回调  
}  
// 可写事件 (POLLOUT)  
if (revents_ & POLLOUT)  
{  
if (writeCallback_) writeCallback_(); // 执行写回调  
}  
eventHandling_ = false;  
}
```

下面的就是loop事件循环的时序图:

![loop循环](/images/muduo/loop循环.png)

理解了 EventLoop 的核心驱动逻辑和 Channel 的事件分发机制后，我们就可以具体分析“三个半事件”的处理了。

## **事件一：新连接的建立与分发**

学习过 Linux 网络编程的都知道，服务器接受新连接的基本流程是 socket -> bind -> listen -> accept。muduo 将这个过程优雅地封装在 Acceptor 和 TcpServer 类中。

### **1. Acceptor：新连接的接收者**

TcpServer 在构造时会创建一个 Acceptor 对象，并将其与主 EventLoop 关联。
Acceptor 负责创建监听套接字、绑定地址、并设置当有新连接到来时（监听套接字可读）的回调函数为 Acceptor::handleRead。

```c++
// TcpServer.cc (构造函数部分)  
TcpServer::TcpServer(EventLoop* loop,  
const InetAddress& listenAddr,  
const string& nameArg,  
Option option)  
: loop_(CHECK_NOTNULL(loop)),  
acceptor_(new Acceptor(loop, listenAddr, option == kReusePort)),  
// ...  
{  
acceptor_->setNewConnectionCallback(  
std::bind(&TcpServer::newConnection, this, _1, _2));  
}

// Acceptor.cc (构造函数部分)
Acceptor::Acceptor(EventLoop* loop, const InetAddress& listenAddr, bool reuseport)
: loop_(loop),  
acceptSocket_(sockets::createNonblockingOrDie(listenAddr.family())),  
acceptChannel_(loop, acceptSocket_.fd()), // 为监听套接字创建 Channel  
// ...  
{  
// ...  
acceptSocket_.bindAddress(listenAddr);  
acceptChannel_.setReadCallback(  
std::bind(&Acceptor::handleRead, this)); // 设置 Channel 的读回调  
}  
```

调用`TcpServer::start()` 时，会通过 `loop_->runInLoop()` 调用 `Acceptor::listen()`，该方法会调用 `listen()` 系统调用并使 `acceptChannel_` 开始关注读事件。

```c++  
// Acceptor.cc  
void Acceptor::listen()  
{  
loop_->assertInLoopThread();  
listenning_ = true;  
acceptSocket_.listen();  
acceptChannel_.enableReading(); // 将 acceptChannel_ 加入 Poller 监听  
}
```

当新连接到达时，acceptChannel_ 的 handleRead 被触发：

```c++
// Acceptor.cc  
void Acceptor::handleRead()  
{  
loop_->assertInLoopThread();  
InetAddress peerAddr;  
int connfd = acceptSocket_.accept(&peerAddr); // 接受新连接  
if (connfd >= 0)  
{  
if (newConnectionCallback_)  
{  
newConnectionCallback_(connfd, peerAddr); // 调用 TcpServer::newConnection  
}  
else  
{  
sockets::close(connfd);  
}  
}  
// ... (错误处理和 EMFILE 处理) ...  
}
```

### **2. TcpServer::newConnection：连接的分发**

Acceptor 将新接受的 connfd 和对端地址传递给 TcpServer::newConnection。此方法的核心职责是：

1. 从 EventLoopThreadPool 中通过轮询选择一个 I/O EventLoop。
2. 为新连接创建一个 TcpConnection 对象，并将选择的 I/O EventLoop 传递给它。
3. 设置 TcpConnection 的各种回调（连接状态、消息到达、写完成、关闭）。
4. 将 TcpConnection::connectEstablished 方法提交到选定的 I/O EventLoop 中执行。

```c++
// TcpServer.cc  
void TcpServer::newConnection(int sockfd, const InetAddress& peerAddr)  
{  
loop_->assertInLoopThread(); // 确保在主 EventLoop 中  
EventLoop* ioLoop = threadPool_->getNextLoop(); // 轮询选择 I/O Loop  
// ... (生成连接名 connName) ...  
TcpConnectionPtr conn(new TcpConnection(ioLoop, // 将 ioLoop 传递给 TcpConnection  
connName,  
sockfd,  
localAddr,  
peerAddr));  
connections_[connName] = conn; // 保存连接  
// ... (设置各种回调) ...  
conn->setCloseCallback(  
std::bind(&TcpServer::removeConnection, this, _1));  
// 将连接建立的后续操作交给 ioLoop 执行  
ioLoop->runInLoop(std::bind(&TcpConnection::connectEstablished, conn));  
}
```

### **3. TcpConnection::connectEstablished：连接的最终建立**

此方法在选定的 I/O EventLoop 线程中执行，完成连接的最后步骤：

```c++
// TcpConnection.cc  
void TcpConnection::connectEstablished()  
{  
loop_->assertInLoopThread(); // 确保在 ioLoop 中  
assert(state_ == kConnecting);  
setState(kConnected);  
channel_->tie(shared_from_this()); // 绑定生命周期  
channel_->enableReading(); // 开始关注该连接上的读事件  
connectionCallback_(shared_from_this()); // 调用用户设置的连接建立回调  
}
```

至此，新连接的建立和分发完成，后续该连接上的所有 I/O 事件都将在其被分配到的 I/O EventLoop 线程中处理。

这是建立连接的时序图：

![建立新连接](/images/muduo/建立新连接.png)
## **事件二：收到消息 (MessageCallback)**

当客户端发送数据时，TcpConnection 对应的 channel_ 会触发读事件，进而调用 TcpConnection::handleRead。

```c++
// TcpConnection.cc (构造函数中设置)  
channel_->setReadCallback(  
std::bind(&TcpConnection::handleRead, this, _1));

// TcpConnection.cc  
void TcpConnection::handleRead(Timestamp receiveTime)  
{  
loop_->assertInLoopThread();  
int savedErrno = 0;  
ssize_t n = inputBuffer_.readFd(channel_->fd(), &savedErrno); // 从 socket 读取数据到 inputBuffer_  
if (n > 0)  
{  
// 调用用户在 TcpServer 中设置的 messageCallback_  
messageCallback_(shared_from_this(), &inputBuffer_, receiveTime);  
}  
else if (n == 0) // 对端关闭  
{  
handleClose();  
}  
else // 错误  
{  
errno = savedErrno;  
LOG_SYSERR << "TcpConnection::handleRead";  
handleError(); // 通常也会调用 handleClose  
}  
}
```


用户通过 TcpServer::setMessageCallback 设置的回调函数会在这里被调用，参数包括 TcpConnectionPtr、存有接收数据的 Buffer* 以及时间戳。用户可以在回调中从 Buffer 中取出数据进行业务处理。

这是处理读事件的时序图:

![收到消息](/images/muduo/处理消息.png)

## **事件三：消息发送完成 (WriteCompleteCallback)**

当用户调用 TcpConnection::send() 发送数据时：

1. 如果当前线程不是连接所属的 ioLoop，则将发送任务 sendInLoop 提交到 ioLoop 执行。
2. sendInLoop 会尝试直接 write() 数据到 socket。
    * 如果数据一次性全部写完，且用户设置了 writeCompleteCallback_，则将其提交到 ioLoop 的任务队列中执行。
    * 如果数据没有一次性写完（例如内核发送缓冲区满），则将剩余数据存入 outputBuffer_，并使 channel_ 开始关注写事件 (enableWriting())。
3. 当 socket 变为可写时，channel_ 的写事件回调 TcpConnection::handleWrite 被触发。
4. handleWrite 会继续从 outputBuffer_ 中发送数据。如果所有数据都发送完毕，则取消对写事件的关注 (disableWriting())，并调用 writeCompleteCallback_。

```c++
// TcpConnection.cc (sendInLoop 核心逻辑)  
void TcpConnection::sendInLoop(const void* data, size_t len)  
{  
loop_->assertInLoopThread();  
ssize_t nwrote = 0;  
size_t remaining = len;  
bool faultError = false;  
if (state_ == kDisconnected) { /* ... return ... */ }

// 如果输出队列为空，尝试直接发送  
if (!channel_->isWriting() && outputBuffer_.readableBytes() == 0)  
{  
nwrote = sockets::write(channel_->fd(), data, len);  
if (nwrote >= 0)  
{  
remaining = len - nwrote;  
if (remaining == 0 && writeCompleteCallback_) // 全部发送完毕  
{  
loop_->queueInLoop(std::bind(writeCompleteCallback_, shared_from_this()));  
}  
}  
else // nwrote < 0 (错误)  
{  
// ... 错误处理 ...  
}  
}

if (!faultError && remaining > 0) // 如果还有数据未发送  
{  
// ... (检查高水位回调 HighWaterMarkCallback) ...  
outputBuffer_.append(static_cast<const char*>(data)+nwrote, remaining); // 存入输出缓冲区  
if (!channel_->isWriting())  
{  
channel_->enableWriting(); // 开始关注写事件  
}  
}  
}

// TcpConnection.cc (构造函数中设置)  
channel_->setWriteCallback(  
std::bind(&TcpConnection::handleWrite, this));

// TcpConnection.cc  
void TcpConnection::handleWrite()  
{  
loop_->assertInLoopThread();  
if (channel_->isWriting())  
{  
ssize_t n = sockets::write(channel_->fd(),  
outputBuffer_.peek(),  
outputBuffer_.readableBytes());  
if (n > 0)  
{  
outputBuffer_.retrieve(n);  
if (outputBuffer_.readableBytes() == 0) // 输出缓冲区已空  
{  
channel_->disableWriting(); // 不再关注写事件  
if (writeCompleteCallback_)  
{  
loop_->queueInLoop(std::bind(writeCompleteCallback_, shared_from_this()));  
}  
if (state_ == kDisconnecting) // 如果正在关闭连接  
{  
shutdownInLoop();  
}  
}  
}  
else  
{  
LOG_SYSERR << "TcpConnection::handleWrite";  
}  
}  
else  
{  
LOG_TRACE << "Connection fd = " << channel_->fd()  
<< " is down, no more writing";  
}  
}
```

“消息发送完成”这个“半个事件”是通过 WriteCompleteCallback 来体现的，它在输出缓冲区的数据全部成功写入内核后被调用。

因为写数据是用户关注的，如果一次性写完成就不必关心了，写不完就先放到buffer中，等到可写的时候，处理下写事件，将数据再次发送，发送完就不必关注了，这样也可以避免busyloop了

这是上述过程的时序图

![消息写完](/images/muduo/消息写完.png)

## **事件四：连接关闭**

连接关闭的触发点有多种：对端关闭（handleRead 读到 EOF）、本地主动关闭（用户调用 TcpConnection::shutdown()）、或发生错误（handleError） 。
这些路径通常都会汇聚到 TcpConnection::handleClose()。

TcpConnection 在构造时设置了其 channel_ 的关闭回调为 handleClose。

```c++
// TcpConnection.cc (构造函数中设置)  
channel_->setCloseCallback(  
std::bind(&TcpConnection::handleClose, this));  
```

`handleClose()` 的核心逻辑：  
1.  断言在正确的 `ioLoop` 中执行。  
2.  将连接状态设为 `kDisconnected`。  
3.  调用 `channel_->disableAll()`，使该 `Channel` 不再关注任何事件。  
4.  调用用户设置的 `connectionCallback_`（此时连接状态已变为 `kDisconnected`）。  
5.  调用 `closeCallback_`（在 `TcpServer::newConnection` 中被绑定到 `TcpServer::removeConnection`）。

```c++  
// TcpConnection.cc  
void TcpConnection::handleClose()  
{  
loop_->assertInLoopThread();  
LOG_TRACE << "fd = " << channel_->fd() << " state = " << stateToString();  
assert(state_ == kConnected || state_ == kDisconnecting);  
setState(kDisconnected);  
channel_->disableAll(); // 如果是那种建立过连接的文件描述符，在这一步就会被从poller移除了

TcpConnectionPtr guardThis(shared_from_this()); // 确保回调期间对象存活  
connectionCallback_(guardThis);  
closeCallback_(guardThis); // 调用 TcpServer::removeConnection  
}  
```
`TcpServer::removeConnection` 会将实际的移除操作（从 `connections_` map 中删除）提交到 `TcpServer` 的主 `EventLoop` 中执行（`removeConnectionInLoop`），以保证线程安全。之后，它会将最终的销毁操作 `TcpConnection::connectDestroyed` 提交回该连接所属的 I/O `EventLoop` 中执行。

```c++  
// TcpServer.cc  
void TcpServer::removeConnection(const TcpConnectionPtr& conn)  
{  
loop_->runInLoop(std::bind(&TcpServer::removeConnectionInLoop, this, conn));  
}

void TcpServer::removeConnectionInLoop(const TcpConnectionPtr& conn)  
{  
loop_->assertInLoopThread();  
LOG_INFO << "TcpServer::removeConnectionInLoop [" << name_  
<< "] - connection " << conn->name();  
connections_.erase(conn->name()); // 从 TcpServer 管理的 map 中移除  
EventLoop* ioLoop = conn->getLoop();  
// 将最后的清理工作交给连接所属的 ioLoop  
ioLoop->queueInLoop(  
std::bind(&TcpConnection::connectDestroyed, conn));  
}  
``` 
`TcpConnection::connectDestroyed` 负责将 `Channel` 从其 `EventLoop` 的 `Poller` 中移除。

```c++  
// TcpConnection.cc  
void TcpConnection::connectDestroyed()  
{  
loop_->assertInLoopThread();  
if (state_ == kConnected) // 如果之前是连接状态，则先更新状态并调用回调  
{  
setState(kDisconnected);  
channel_->disableAll();  
connectionCallback_(shared_from_this());  
}  
channel_->remove(); // 从 Poller 中移除 Channel  
}

// Channel.cc  
void Channel::remove()  
{  
assert(isNoneEvent());  
addedToLoop_ = false;  
loop_->removeChannel(this); // 通知 EventLoop 从 Poller 中移除  
}
```

EventLoop::removeChannel 会调用 Poller::removeChannel，最终通过 epoll_ctl(EPOLL_CTL_DEL, ...) 将文件描述符从 epoll 实例中移除。

这是关闭连接的时序图:

![连接关闭](/images/muduo/连接关闭.png)
当 TcpConnectionPtr 的最后一个 shared_ptr 引用（通常是在 connectDestroyed 的 std::bind 对象析构时）消失后，TcpConnection 对象及其拥有的 Socket（会在析构时关闭 fd）和 Channel 对象会被自动析构，完成资源的彻底回收。

这一套精心设计的流程，严格遵守了“对象生命周期管理”和“线程封闭”的原则，确保了连接关闭的正确性和资源的有效释放。

## **总结**

通过对 muduo 中新连接建立、数据收发、连接关闭这“三个半事件”处理流程的分析，我们可以看到陈硕大佬是如何将这些网络编程中的核心操作，优雅地融入其基于对象的事件驱动框架中的。

* **事件的统一处理入口：** EventLoop::loop() 是所有事件处理的起点。
* **Channel 作为事件分发的核心：** 它封装了文件描述符和相关的事件回调。
* **职责明确的类设计：** Acceptor 专注于接受新连接，TcpServer 负责管理和分发连接，TcpConnection 负责处理单个连接的生命周期和业务逻辑。
* **线程安全的保证：** 通过 "One Loop Per Thread" 和 runInLoop/queueInLoop 机制，确保了所有对象的操作都在其所属的线程中执行。
* **回调机制的广泛应用：** 将具体的业务逻辑与框架逻辑解耦，提高了灵活性和可扩展性。

通过这些操作，在遵循陈硕大佬的 `基于对象` 的理念的前提下，完成了对“三个半事件”的处理。