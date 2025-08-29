
---
title: "muduo源码剖析:04.Buffer设计分析"
date: 2025-06-10
categories: 
  - 源码分析
  - muduo
tags:
  - C++
  - muduo
  - 网络库
---
## **前言**

在本系列之前的源码剖析中，我们已经分析了 muduo “一个线程一个EventLoop”的实现方式、网络连接事件的处理（三个半事件）、以及 TimerQueue 如何将定时器纳入事件循环框架。
至此，我们对 muduo 的事件驱动逻辑和核心调度机制已经有了深入的理解。

在正式进入 Buffer 的剖析前，让我们先重新考虑下一下陈硕大佬在《Linux高性能网络编程》一书的 6.4.1 小节中提出的关于为什么要使用微应用层缓冲区的问题：

1. 假设应用程序需要发送 40kB 数据，但是操作系统的 TCP 发送缓冲区只有 25kB 剩余空间，那么剩下的 15kB 数据怎么办？如果等待 OS 缓冲区可用，会阻塞当前线程…
2. 如果应用程序随后又要发送 50kB 数据，而此时应用层发送缓冲区中尚有未发送的数据…那么网络库应该将这 50kB 数据追加到发送缓冲区的末尾，而不能立刻尝试 write()，因为这样有可能打乱数据的顺序。
3. 在非阻塞网络编程中，为什么要使用应用层接收缓冲区？假如一次读到的数据不够一个完整的数据包，那么这些已经读到的数据是不是应该先暂存在某个地方…？
4. 在非阻塞网络编程中，如何设计并使用缓冲区？一方面我们希望减少系统调用，一次读的数据越多越划算…另一方面，我们希望减少内存占用…muduo 用 readv(2) 结合栈上空间巧妙地解决了这个问题。

这些问题实际上在说一个事情：**如何在非阻塞 I/O 模型下，高效、安全地处理数据的收发和内存管理。** 

在计算机界有一个经典的论断：`程序 = 数据结构 + 算法 `,muduo::net::Buffer就是陈硕大佬解决此问题的数据结构，也是他此问题的答案。

本文，我们将深入剖析 muduo::net::Buffer 的设计与实现，理解其如何通过精巧的内部结构和操作，实现高效的内存管理和数据处理，从而为 muduo 的高性能网络 I/O 提供坚实基础。
<!-- more -->
## **muduo::Buffer 概览与设计目标**

muduo::net::Buffer 的核心目标是提供一个可动态增长的缓冲区，用于暂存网络套接字读写的数据。其设计深受 Netty ChannelBuffer 的启发，采用了经典的三段式内存布局：

+-------------------+------------------+------------------+  
| prependable bytes |  readable bytes  |  writable bytes  |  
|                   |     (CONTENT)    |                  |  
+-------------------+------------------+------------------+  
|                   |                  |                  |  
0      <=      readerIndex   <=   writerIndex    <=     size

* **prependable bytes (可预置空间):** 位于缓冲区的最前端。它的一个妙用是在已有数据前方便地添加协议头（如消息长度），而无需移动现有数据。muduo 默认预留了 kCheapPrepend = 8 字节。
* **readable bytes (可读数据区):** 存储了从网络接收到或准备发送的实际有效数据，从 readerIndex_ 开始，到 writerIndex_ 结束。
* **writable bytes (可写空间):** 位于可读数据之后，用于追加新的数据，从 writerIndex_ 开始，到缓冲区末尾结束。

这种设计使得 Buffer 在处理网络协议和进行数据读写时非常灵活和高效。

## **核心数据成员与内存布局**

Buffer 类的核心主要由以下三个成员构成：

```c++
class Buffer : public muduo::copyable  
{  
public:  
static const size_t kCheapPrepend = 8;  
static const size_t kInitialSize = 1024;

explicit Buffer(size_t initialSize = kInitialSize)
: buffer_(kCheapPrepend + initialSize), // 内部使用 std::vector<char> 存储数据  
readerIndex_(kCheapPrepend),          // 读指针，初始指向预留空间之后  
writerIndex_(kCheapPrepend)           // 写指针，初始与读指针相同  
{  
assert(readableBytes() == 0);  
assert(writableBytes() == initialSize);  
assert(prependableBytes() == kCheapPrepend);  
}  
// ...  
private:  
std::vector<char> buffer_; // 底层存储  
size_t readerIndex_;       // 读指针索引  
size_t writerIndex_;       // 写指针索引  
// ...  
};
```

* buffer_: 一个 std::vector<char>，作为实际存储数据的底层容器。其初始大小为 kCheapPrepend + initialSize。
* readerIndex_: size_t 类型，标记可读数据的起始位置。
* writerIndex_: size_t 类型，标记可读数据的结束位置，同时也是可写空间的起始位置。

通过这两个索引，我们可以方便地计算出三段空间的大小，并获取可读数据的指针：

```c++
size_t readableBytes() const  
{ return writerIndex_ - readerIndex_; } // 可读数据长度

size_t writableBytes() const  
{ return buffer_.size() - writerIndex_; } // 可写空间长度

size_t prependableBytes() const  
{ return readerIndex_; } // 可预置空间长度

const char* peek() const  
{ return begin() + readerIndex_; }
```

## **基本操作：数据读取 (Retrieve) 与追加 (Append)**

### **1. 数据读取 (Retrieve) - 逻辑上的消耗**

当数据被应用程序消耗后，需要从 Buffer 中“取出”这部分数据。muduo::Buffer 并不立即删除内存，而是通过移动 readerIndex_ 来高效地完成这个操作：

```c++
void Buffer::retrieve(size_t len)  
{  
assert(len <= readableBytes());  
if (len < readableBytes())  
{  
readerIndex_ += len; // 简单地将读指针后移  
}  
else // 如果取出的长度等于或超过可读数据长度，则全部取出  
{  
retrieveAll();  
}  
}

void Buffer::retrieveAll()  
{  
// 将读写指针都重置到预留空间之后，表示缓冲区已空。  
// 之前已读的数据空间（0 到 readerIndex_）被逻辑上回收，成为新的 prependable 空间。  
readerIndex_ = kCheapPrepend;  
writerIndex_ = kCheapPrepend;  
}

string Buffer::retrieveAsString(size_t len)  
{  
assert(len <= readableBytes());  
string result(peek(), len); // 从可读区构造字符串  
retrieve(len); // 更新读指针  
return result;  
}
```

核心思想是通过增加 readerIndex_ 来“丢弃”已处理的数据，这些数据在物理上并未立即从 buffer_ 中删除，只是逻辑上变为不可读。当 retrieveAll() 被调用时，读写指针会重置，为下一次写入腾出大量空间。

### **2. 数据追加 (Append) - 解决系统缓冲区满的问题**

向 Buffer 中写入新数据是通过 append 系列方法实现的，这会移动 writerIndex_。这正是解决文章开头提出的“系统发送缓冲区满，数据怎么办”问题的答案——先将数据追加到应用层的 Buffer 中。

```c++
void Buffer::append(const char* /*restrict*/ data, size_t len)  
{  
ensureWritableBytes(len); // 确保有足够的可写空间  
std::copy(data, data+len, beginWrite()); // 拷贝数据到可写区  
hasWritten(len); // 更新写指针  
}

void Buffer::hasWritten(size_t len) // 更新写指针  
{  
assert(len <= writableBytes());  
writerIndex_ += len;  
}

```

在追加数据前，会调用 ensureWritableBytes(len) 来确保有足够的空间。如果空间不足，则会触发 makeSpace(len) 逻辑。

## **空间管理与扩容：makeSpace 的智慧**

makeSpace 是 Buffer 内存管理的核心，它智能地采取两种策略来获取更多可写空间：

1. **内部腾挪 (空间复用)：** 如果 writableBytes() + prependableBytes()（即总的空闲空间）足够大，它会选择将当前可读数据 (readerIndex_ 到 writerIndex_ 之间的内容) **向前移动**到 kCheapPrepend 位置，从而将 prependableBytes 的已读空间转化为新的 writableBytes。这种方式**避免了 std::vector 的重新分配和数据拷贝**（如果 resize 导致了重新分配），效率极高。
2. **外部扩容 (内存增长)：** 如果总空闲空间也不够用，说明缓冲区确实需要增长。此时，只能通过 buffer_.resize(writerIndex_ + len) 来扩展底层 std::vector 的大小。

```c++
void Buffer::makeSpace(size_t len)  
{  
// 条件：总空闲空间不足以容纳 len 和 kCheapPrepend  
if (writableBytes() + prependableBytes() < len + kCheapPrepend)  
{  
// 只能扩容 vector  
buffer_.resize(writerIndex_ + len);  
}  
else // 总空闲空间足够，通过移动数据来腾出可写空间  
{  
assert(kCheapPrepend < readerIndex_); // 确保 prependableBytes 区域确实有已读空间  
size_t readable = readableBytes();  
// 将 [readerIndex_, writerIndex_) 的数据拷贝到 [kCheapPrepend, kCheapPrepend + readable)  
std::copy(begin() + readerIndex_,  
begin() + writerIndex_,  
begin() + kCheapPrepend);  
readerIndex_ = kCheapPrepend; // 更新读指针  
writerIndex_ = readerIndex_ + readable; // 更新写指针  
assert(readable == readableBytes());  
}  
}
```

这种“优先内部腾挪，实在不行再扩容”的策略，完美兼顾了效率和空间利用率。

## **高效的 Socket 读操作：readFd 与 readv 的绝妙配合**

muduo 通常工作在 LT (电平触发) 模式下，为了避免因数据未读完而导致的事件重复触发，需要一次性将 socket 缓冲区的数据尽可能读完。Buffer::readFd 正是为此设计的，它通过 readv (分散读) 系统调用和栈上临时缓冲区 extrabuf，巧妙地解决了这个问题，同时优化了性能。

```c++
ssize_t Buffer::readFd(int fd, int* savedErrno)  
{  
char extrabuf[65536]; // 在栈上分配一个较大的临时缓冲区 (64KB)  
struct iovec vec[2];  
const size_t writable = writableBytes(); // Buffer 内部当前可写空间

// 第一块 iovec 指向 Buffer 内部的可写空间  
vec[0].iov_base = begin() + writerIndex_;  
vec[0].iov_len = writable;  
// 第二块 iovec 指向栈上的 extrabuf  
vec[1].iov_base = extrabuf;  
vec[1].iov_len = sizeof extrabuf;

// 如果 Buffer 内部可写空间较小，则同时使用两块 iovec 进行读操作  
const int iovcnt = (writable < sizeof extrabuf) ? 2 : 1;  
const ssize_t n = sockets::readv(fd, vec, iovcnt); // 一次系统调用，最多读取 writable + 64KB 数据

if (n < 0)  
{  
*savedErrno = errno;  
}  
else if (implicit_cast<size_t>(n) <= writable) // 读取的数据全部放入了 Buffer 的可写区  
{  
writerIndex_ += n;  
}  
else // 读取的数据量大于 Buffer 内部可写空间，说明 extrabuf 被用到了  
{  
writerIndex_ = buffer_.size(); // Buffer 内部可写空间已写满  
append(extrabuf, n - writable); // 将 extrabuf 中多余的数据追加到 Buffer (此时会触发 makeSpace 扩容)  
}  
return n;  
}  
```

`readFd` 的设计有几个显著优点：  
1.  **减少系统调用：** 通过 `readv` 和 `extrabuf`，即使 `Buffer` 内部当前可写空间不大，也能尝试一次性从内核读取更多数据，避免了多次 `read` 系统调用。  
2.  **避免内存浪费：** 解决了文章开头提出的“为每个连接分配大的缓冲区导致内存浪费”的问题。`Buffer` 可以以较小的初始大小启动，`readFd` 利用栈上临时空间来处理突发的大量数据，只有在确认需要时才真正扩容 `Buffer`。  
3.  **避免 `ioctl(FIONREAD)`：** 它没有先查询有多少数据可读，而是直接尝试读取，更加高效。  
4.  **栈上缓冲区：** `extrabuf` 分配在栈上，避免了额外的堆内存分配开销。

## **其他实用特性**

* **网络字节序处理：** `Buffer` 提供了一系列 `appendIntXX`、`readIntXX`、`peekIntXX` 方法，方便地处理网络字节序（Big Endian）的整型数据。  
* **头部预置 (`prepend`)：** 利用 `prependableBytes_` 空间，在可读数据前高效地添加数据，非常适合用于封装协议头。  
* **查找与收缩：** `findCRLF()` / `findEOL()` 辅助解析文本协议；`shrink()` 用于在 `Buffer` 容量远大于实际数据时回收内存。

## **总结与设计启示**

`muduo::net::Buffer` 通过其精巧的三段式内存布局、灵活的读写指针以及高效的空间管理和 I/O 策略，完美地回答了文章开头提出的所有关于网络编程中缓冲区设计的难题。 它为 `muduo` 网络库提供了坚实的数据缓冲基础。

其核心设计启示包括：

1.  **空间复用优于频繁分配：** `kCheapPrepend` 的设计和 `makeSpace` 中优先移动数据的策略，体现了对内存分配和数据拷贝的优化。  
2.  **减少系统调用是关键：** `readFd` 中使用 `readv` 和栈上大缓冲区，是典型的用少量计算换取大量 I/O 系统调用开销的性能优化范例。  
3.  **设计要贴合场景：** `Buffer` 的所有设计都紧密围绕 TCP 网络编程的特点和痛点，如协议头添加、避免阻塞等。  
4.  **接口的完备性与易用性：** 提供了丰富的辅助函数，使得上层业务处理数据和解析协议变得更加简单和安全。

理解了 `Buffer` 的设计，我们就能更好地理解 `muduo` 中数据是如何在网络连接中高效流转的。至此，我们对 `muduo` 核心的并发模型、事件处理、时间管理以及数据缓冲都有了深入的认识。