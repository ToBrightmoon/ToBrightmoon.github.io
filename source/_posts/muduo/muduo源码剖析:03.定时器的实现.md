
---
title: "muduo源码剖析:03.定时器的实现"
date: 2025-08-27
categories: 
  - 源码分析
  - muduo
tags:
  - C++
  - muduo
  - 网络库
---
## **前言**

在muduo源码剖析的前两篇文章中，我们深入探讨了 muduo 的核心并发模型——"One Loop Per Thread" 的实现，以及 TCP
连接从建立、数据收发到关闭的完整生命周期管理。

这些内容实际上已经对陈硕大佬在网络库上的设计设计思想体现的很清晰了。但是仅有这些还是不够的， 除了处理网络 I/O
事件，一个完备的网络库还需要处理时间相关的事件。
例如，在固定的时间点执行某个任务（runAt）、在一段延迟之后执行任务（runAfter），或者以固定的时间间隔重复执行任务（runEvery）。这些都离不开一个高效且精准的定时器机制。

正如陈硕大佬在《Linux多线程服务端编程》一书中所强调的，muduo 的一个重要设计选择是**利用 timerfd_create 这个 Linux
系统调用，将时间事件也转化为文件描述符事件**，从而能够被 EventLoop 的 Poller (通常是 epoll) 统一管理和调度。

本文，我们将聚焦于 muduo::net::TimerQueue 类，深入剖析其如何巧妙地利用 timerfd 实现了一个高效、线程安全的定时器队列，并与
EventLoop 的事件驱动模型完美融合。
<!-- more -->
## **从 Printer 示例看 TimerQueue 的应用**

在深入源码之前，我们先来看一个 muduo 提供的简单示例，它展示了如何使用 EventLoop 提供的定时器接口：

```c++
#include <muduo/net/EventLoop.h>  
#include <muduo/base/Timestamp.h> // 虽然示例中没直接用，但 runAfter 内部会用  
#include <muduo/base/Logging.h>   // 虽然示例中没直接用，但 TimerQueue 内部会用  
#include <iostream>  
#include <functional>

class Printer : muduo::noncopyable  
{  
public:  
Printer(muduo::net::EventLoop* loop)  
: loop_(loop),  
count_(0)  
{  
// 注意：对于这种周期性任务，loop->runEvery() 是更好的选择。  
// 这里使用 runAfter 来演示其基本用法和递归调用自身实现重复。  
loop_->runAfter(1.0, std::bind(&Printer::print, this)); // 1秒后执行 print  
}

~Printer()  
{  
std::cout << "Final count is " << count_ << "n";  
}

void print()  
{  
if (count_ < 5)  
{  
std::cout << count_ << "n";  
++count_;
// 再次调度自己，1秒后执行  
loop_->runAfter(1.0, std::bind(&Printer::print, this));  
}  
else  
{  
    loop_->quit(); // 打印5次后退出 EventLoop  
}  
}

private:  
muduo::net::EventLoop* loop_;  
int count_;  
};

//int main() { ... EventLoop loop; Printer printer(&loop); loop.loop(); ... }
```

这个 Printer 类通过 EventLoop::runAfter 接口，实现了每隔 1 秒打印一次计数，共打印 5 次后退出 EventLoop 的功能。
EventLoop 提供的 runAt, runAfter, runEvery 以及 cancel 定时器接口，其底层实现都委托给了 TimerQueue 对象。

## **TimerQueue 的核心职责与设计概览**

TimerQueue 的核心职责是管理一系列的定时器 (Timer 对象)，并在它们到期时执行其回调函数。其整体设计思路如下：

1. **timerfd 作为时间事件的统一入口：**
    * 在 TimerQueue 构造时，会通过 timerfd_create(CLOCK_MONOTONIC, TFD_NONBLOCK | TFD_CLOEXEC) 创建一个
      timerfd。CLOCK_MONOTONIC 保证了时间是单调递增的，不受系统时间修改的影响。
    * 这个 timerfd 被封装成一个 Channel 对象 (timerfdChannel_)，并注册到其所属的 EventLoop 中，监听其可读事件。
2. **按到期时间排序的定时器列表：**
    * TimerQueue 内部使用 std::set<std::pair<Timestamp, Timer*>> (即 TimerList timers_) 来存储所有活动的定时器。std::set
      会自动根据 Timestamp (到期时间) 和 Timer* (指针地址，用于时间相同时保证唯一性) 进行排序，使得 timers_.begin()
      始终指向最早到期的那个定时器。
3. **动态设置 timerfd 的超时：**
    * 每当添加新的定时器或有定时器到期后，TimerQueue 会检查 timers_ 列表中最早到期的定时器的时间戳。
    * 然后，它会调用 timerfd_settime 系统调用，将 timerfd_ 的下一次超时时间设置为这个最早到期时间点。
4. **事件驱动的定时器处理：**
    * 当 timerfd_ 因设置的超时时间到达而变为可读时，EventLoop 的 Poller 会检测到这个事件，并通过 timerfdChannel_
      调用其注册的读回调函数，即 TimerQueue::handleRead。
    * TimerQueue::handleRead 负责：
        * 读取 timerfd_ 以清除事件通知。
        * 从 timers_ 中找出所有已经到期的定时器。
        * 执行这些到期定时器的回调函数。
        * 对于需要重复执行的定时器，重新计算其下一次到期时间并将其插回 timers_。
        * 根据 timers_ 中新的最早到期时间，重新设置 timerfd_。

## **关键数据结构与成员**

TimerQueue 内部使用了几个关键的数据结构来管理定时器：

```c++
class TimerQueue : noncopyable  
{  
public:  
// ... (构造与析构) ...  
private:  
// Entry 定义为一个 pair，包含到期时间戳和 Timer 指针  
typedef std::pair<Timestamp, Timer*> Entry;  
// TimerList 使用 std::set 存储 Entry，利用 set 的自动排序特性  
typedef std::set<Entry> TimerList;

// ActiveTimer 用于在取消时快速查找 Timer，通过 Timer* 和其序列号唯一标识  
typedef std::pair<Timer*, int64_t> ActiveTimer;  
typedef std::set<ActiveTimer> ActiveTimerSet;

EventLoop* loop_;          // 所属的 EventLoop  
const int timerfd_;        // timerfd_create() 返回的文件描述符  
Channel timerfdChannel_;   // 用于将 timerfd_ 纳入 EventLoop 管理的 Channel  
TimerList timers_;         // 按到期时间排序的定时器列表

// for cancel()  
ActiveTimerSet activeTimers_;      // 存储所有活跃的 Timer，用于高效取消  
bool callingExpiredTimers_;      // 标记是否正在调用已到期定时器的回调  
ActiveTimerSet cancelingTimers_; // 存储在调用已到期定时器回调期间，请求取消的定时器  
};
```

* loop_: 指向所属的 EventLoop 对象。
* timerfd_: 通过 detail::createTimerfd() 创建的文件描述符。
* timerfdChannel_: 将 timerfd_ 封装成一个 Channel，其读回调设置为 TimerQueue::handleRead。
* timers_ (TimerList): 一个 std::set<std::pair<Timestamp, Timer*>>，按到期时间升序存储定时器。timers_.begin()
  始终指向最早到期的定时器。
* activeTimers_ (ActiveTimerSet): 一个 std::set<std::pair<Timer*, int64_t>>，用于通过 Timer*
  和其序列号快速取消定时器。timers_ 和 activeTimers_ 中的 Timer* 应该是一一对应的，它们的 size() 应该始终相等。
* callingExpiredTimers_: 布尔标记，指示当前是否正在执行已到期定时器的回调。
* cancelingTimers_ (ActiveTimerSet): 用于处理在执行回调期间发生的取消请求。

## **TimerQueue 的初始化**

TimerQueue 在构造时，会创建 timerfd_，并初始化 timerfdChannel_，将其注册到 EventLoop 中。

```c++
// TimerQueue.cc
TimerQueue::TimerQueue(EventLoop* loop)
: loop_(loop),  
timerfd_(detail::createTimerfd()), // 调用辅助函数创建 timerfd  
timerfdChannel_(loop, timerfd_),   // 创建 Channel，并与 loop_ 关联  
timers_(),  
callingExpiredTimers_(false)  
{  
timerfdChannel_.setReadCallback(  
std::bind(&TimerQueue::handleRead, this)); // 设置读回调  
// 即使没有定时器，也使能读事件。  
// timerfd 的实际超时是通过 timerfd_settime 设置的。  
// 如果没有活动的定时器，timerfd_settime 会将其超时设为一个不会触发的状态  
// (例如，it_value 设为0，表示 disarm) 或一个极大的未来时间。  
// muduo 的做法是，如果 nextExpire 无效，则不调用 resetTimerfd。  
timerfdChannel_.enableReading();  
}

// muduo/net/TimerQueue.cc (detail 命名空间内)  
int createTimerfd()  
{  
int timerfd = ::timerfd_create(CLOCK_MONOTONIC,  
TFD_NONBLOCK | TFD_CLOEXEC);  
if (timerfd < 0)  
{  
LOG_SYSFATAL << "Failed in timerfd_create";  
}  
return timerfd;  
}
```

## **添加定时器：EventLoop 接口与 TimerQueue 实现**

用户通常通过 EventLoop 提供的接口（runAt, runAfter, runEvery）来添加定时器。这些接口最终都会调用到 TimerQueue::addTimer。

```c++
// EventLoop.cc  
TimerId EventLoop::runAt(Timestamp time, TimerCallback cb)  
{  
return timerQueue_->addTimer(std::move(cb), time, 0.0); // interval 为 0 表示非重复  
}

TimerId EventLoop::runAfter(double delay, TimerCallback cb)  
{  
Timestamp time(addTime(Timestamp::now(), delay)); // 计算绝对到期时间  
return runAt(time, std::move(cb));  
}

TimerId EventLoop::runEvery(double interval, TimerCallback cb)  
{  
Timestamp time(addTime(Timestamp::now(), interval)); // 首次到期时间  
return timerQueue_->addTimer(std::move(cb), time, interval); // interval > 0 表示重复  
}  
```

`TimerQueue::addTimer` 方法由于可能被其他线程调用，它会将实际的添加操作 `addTimerInLoop` 通过 `loop_->runInLoop()` 提交到
`TimerQueue` 所属的 `EventLoop` 线程中执行，以保证线程安全。

```c++  
// TimerQueue.cc  
TimerId TimerQueue::addTimer(TimerCallback cb,  
Timestamp when,  
double interval)  
{  
Timer* timer = new Timer(std::move(cb), when, interval); // 创建 Timer 对象  
loop_->runInLoop( // 保证在 loop_ 线程中执行  
std::bind(&TimerQueue::addTimerInLoop, this, timer));  
return TimerId(timer, timer->sequence()); // 返回 TimerId 用于取消  
}

void TimerQueue::addTimerInLoop(Timer* timer)  
{  
loop_->assertInLoopThread(); // 确保在正确的线程  
bool earliestChanged = insert(timer); // 将 Timer 插入内部列表

if (earliestChanged) // 如果新插入的定时器成为了最早到期的  
{  
// 重置 timerfd 的超时时间为这个新定时器的到期时间  
detail::resetTimerfd(timerfd_, timer->expiration());  
}  
}  
```

`insert(Timer* timer)` 方法负责将 `Timer` 对象同时插入到 `timers_` (按时间排序) 和 `activeTimers_` (用于取消) 两个
`std::set` 中，并返回新插入的定时器是否改变了“最早到期时间”。

```c++  
// TimerQueue.cc  
bool TimerQueue::insert(Timer* timer)  
{  
loop_->assertInLoopThread();  
assert(timers_.size() == activeTimers_.size());  
bool earliestChanged = false;  
Timestamp when = timer->expiration();  
TimerList::iterator it = timers_.begin();  
// 如果 timers_ 为空，或者新定时器的到期时间早于当前最早的定时器  
if (it == timers_.end() || when < it->first)  
{  
earliestChanged = true;  
}

timers_.insert(Entry(when, timer));  
activeTimers_.insert(ActiveTimer(timer, timer->sequence()));

assert(timers_.size() == activeTimers_.size());  
return earliestChanged;  
}

```

如果 earliestChanged 为 true，则需要调用 detail::resetTimerfd 来更新 timerfd_ 的超时设置。detail::resetTimerfd 内部调用
timerfd_settime，并将 newValue.it_value 设置为从当前时间到目标到期时间的相对时间。

```c++
// muduo/net/TimerQueue.cc (detail 命名空间内)  
void resetTimerfd(int timerfd, Timestamp expiration)  
{  
struct itimerspec newValue;  
memZero(&newValue, sizeof newValue);  
newValue.it_value = howMuchTimeFromNow(expiration); // 计算相对超时时间  
int ret = ::timerfd_settime(timerfd, 0, &newValue, NULL); // 0 表示相对时间，不关心 oldValue  
if (ret)  
{  
LOG_SYSERR << "timerfd_settime()";  
}  
}

struct timespec howMuchTimeFromNow(Timestamp when)  
{  
int64_t microseconds = when.microSecondsSinceEpoch()  
- Timestamp::now().microSecondsSinceEpoch();  
if (microseconds < 100) // 最小超时设为 100 微秒，避免过于频繁或立即触发  
{  
microseconds = 100;  
}  
struct timespec ts;  
ts.tv_sec = static_cast<time_t>(  
microseconds / Timestamp::kMicroSecondsPerSecond);  
ts.tv_nsec = static_long_cast( // 使用 muduo 的类型安全转换  
(microseconds % Timestamp::kMicroSecondsPerSecond) * 1000);  
return ts;  
}
```

## **处理定时器到期：TimerQueue::handleRead**

当 timerfd_ 因设置的超时时间到达而变为可读时，EventLoop 会调用 timerfdChannel_ 的读回调，即 TimerQueue::handleRead。

```c++
// TimerQueue.cc  
void TimerQueue::handleRead()  
{  
loop_->assertInLoopThread();  
Timestamp now(Timestamp::now());  
detail::readTimerfd(timerfd_, now); // 1. 读取 timerfd，清空事件，避免重复触发

// 2. 获取所有在 'now' 时刻之前或同时到期的定时器  
std::vector<Entry> expired = getExpired(now);

callingExpiredTimers_ = true; // 标记正在调用回调  
cancelingTimers_.clear();    // 清空上次调用期间的取消列表

// 3. 遍历所有到期的定时器并执行其回调  
for (const Entry& it : expired)  
{  
it.second->run(); // Timer::run() 会调用用户设置的 TimerCallback  
}  
callingExpiredTimers_ = false; // 标记回调调用结束

// 4. 重置重复的定时器，并设置 timerfd 的下一次超时时间  
reset(expired, now);  
}

```

detail::readTimerfd 简单地读取 timerfd 中的 uint64_t 值（表示自上次成功读取以来发生的超时次数），主要是为了清除 timerfd
的可读状态。

getExpired(Timestamp now) 方法负责从 timers_ 和 activeTimers_ 中找出并**移除**所有在 now 时刻之前（包括 now）到期的定时器。

```c++
// TimerQueue.cc  
std::vector<TimerQueue::Entry> TimerQueue::getExpired(Timestamp now)  
{  
assert(timers_.size() == activeTimers_.size());  
std::vector<Entry> expired;  
// 构造一个哨兵 Entry，其时间戳为 now，Timer* 为一个不可能的地址 (UINTPTR_MAX)  
// std::set::lower_bound 会找到第一个不小于 sentry 的元素  
// 由于 pair 的比较是先比较 first 再比较 second，  
// UINTPTR_MAX 确保了在时间戳相同时，sentry 比任何有效的 Timer* 都大。  
// 因此，end 将指向第一个到期时间严格大于 now 的定时器，或者 timers_.end()。  
Entry sentry(now, reinterpret_cast<Timer*>(UINTPTR_MAX));  
TimerList::iterator end = timers_.lower_bound(sentry);  
assert(end == timers_.end() || now < end->first);

// 将 [timers_.begin(), end) 范围内的元素（即所有已到期的）拷贝到 expired 向量  
std::copy(timers_.begin(), end, std::back_inserter(expired));  
// 从 timers_ 中移除这些已到期的元素  
timers_.erase(timers_.begin(), end);

// 同时从 activeTimers_ 中移除这些已到期的元素  
for (const Entry& it : expired)  
{  
ActiveTimer timer(it.second, it.second->sequence());  
size_t n = activeTimers_.erase(timer);  
assert(n == 1); (void)n;  
}

assert(timers_.size() == activeTimers_.size());  
return expired;  
}
```

下面是定时器到期处理的时序图,展示了 timerfd 触发后，TimerQueue 如何处理到期定时器的完整流程：

![定时器到期处理](/images/定时器到期处理.png)

## **重置与重新调度：TimerQueue::reset**

在处理完一批到期的定时器后，reset 方法负责：

1. 对于那些是重复执行 (it.second->repeat()) 且在回调执行期间未被 cancelingTimers_ 标记为取消的定时器，调用 Timer::
   restart(now) 更新其下一次到期时间，并将其重新调用 insert() 方法插入到 timers_ 和 activeTimers_ 中。
2. 对于非重复的或已被取消的定时器，则 delete it.second 释放 Timer 对象内存。
3. 根据 timers_ 中新的最早到期时间（如果列表不为空），调用 detail::resetTimerfd 重新设置 timerfd_ 的下一次超时。

```c++
// TimerQueue.cc  
void TimerQueue::reset(const std::vector<Entry>& expired, Timestamp now)  
{  
Timestamp nextExpire;

for (const Entry& it : expired)  
{  
ActiveTimer timer(it.second, it.second->sequence());  
// 如果是重复定时器，并且在回调执行期间没有被加入到 cancelingTimers_ 列表  
if (it.second->repeat()  
&& cancelingTimers_.find(timer) == cancelingTimers_.end())  
{  
it.second->restart(now); // 更新下次到期时间  
insert(it.second);       // 重新插入队列 (insert 会处理 earliestChanged)  
}  
else // 非重复或已被取消  
{  
// 作者在此处留下了 // FIXME: no delete please 的注释，  
// 暗示未来可能考虑使用对象池 (free list) 来复用 Timer 对象，  
// 以减少频繁 new 和 delete 带来的开销和内存碎片。  
delete it.second;   
}  
}
if (!timers_.empty()) // 如果还有未到期的定时器  
{  
nextExpire = timers_.begin()->second->expiration(); // 获取下一个最早到期时间  
}

if (nextExpire.valid()) // 如果存在下一个有效到期时间  
{  
resetTimerfd(timerfd_, nextExpire); // 重置 timerfd  
}  
}
```

## **取消定时器：TimerQueue::cancel 与 cancelInLoop**

用户通过 EventLoop::cancel(TimerId) 来取消定时器，该方法最终调用 TimerQueue::cancel。与 addTimer 类似，cancel 也会将实际的取消操作
cancelInLoop 提交到 loop_ 线程执行。

```c++
// EventLoop.cc  
void EventLoop::cancel(TimerId timerId)  
{  
return timerQueue_->cancel(timerId);  
}

// TimerQueue.cc  
void TimerQueue::cancel(TimerId timerId)  
{  
loop_->runInLoop(  
std::bind(&TimerQueue::cancelInLoop, this, timerId));  
}

void TimerQueue::cancelInLoop(TimerId timerId)  
{  
loop_->assertInLoopThread();  
assert(timers_.size() == activeTimers_.size());

// 使用 TimerId 中的 Timer* 和 sequence_ 构造 ActiveTimer 用于查找  
ActiveTimer timer(timerId.timer_, timerId.sequence_);  
ActiveTimerSet::iterator it = activeTimers_.find(timer);

if (it != activeTimers_.end()) // 如果在 activeTimers_ 中找到了该定时器  
{  
// 从 timers_ 中移除对应的 Entry  
// 注意：timers_ 的 key 是 pair<Timestamp, Timer*>，需要用其到期时间和指针来构造  
size_t n = timers_.erase(Entry(it->first->expiration(), it->first));  
assert(n == 1); (void)n; // 应该能精确找到并删除一个  
delete it->first; // 释放 Timer 对象内存  
activeTimers_.erase(it); // 从 activeTimers_ 中移除  
}  
else if (callingExpiredTimers_) // 如果没找到，且当前正在执行回调  
{  
// 这意味着要取消的定时器可能是一个刚刚到期并在被处理的重复定时器。  
// 将其加入 cancelingTimers_ 集合。  
// reset() 方法在重新插入重复定时器前会检查这个 cancelingTimers_ 集合，  
// 如果在其中，则不会重新插入，从而达到取消的目的。  
cancelingTimers_.insert(timer);  
}  
assert(timers_.size() == activeTimers_.size());  
}
```

这种处理方式确保了即使在定时器回调执行期间尝试取消该（重复的）定时器，也能正确处理。

下面是取消操作的时序图,展示了通过 EventLoop::cancel 取消定时器的流程，包括线程安全处理和回调期间的取消逻辑:

![取消操作](/images/取消操作.png)

## **Timer类与重复执行**

Timer 类封装了定时器的基本信息：回调函数、到期时间、重复间隔、是否重复以及一个唯一的序列号。

```c++
// Timer.h (部分)  
class Timer : noncopyable  
{  
public:  
Timer(TimerCallback cb, Timestamp when, double interval)  
: callback_(std::move(cb)),  
expiration_(when),  
interval_(interval),  
repeat_(interval > 0.0), // interval > 0.0 表示是重复定时器  
sequence_(s_numCreated_.incrementAndGet()) // 原子生成的唯一序列号  
{ }

void run() const // 执行回调  
{  
callback_();  
}

Timestamp expiration() const  { return expiration_; }  
bool repeat() const { return repeat_; }  
int64_t sequence() const { return sequence_; }

void restart(Timestamp now) // 重新计算下次到期时间 (用于重复定时器)  
{  
assert(repeat_);  
expiration_ = addTime(now, interval_);  
}

// ...  
private:  
const TimerCallback callback_;  
Timestamp expiration_;  
const double interval_;  
const bool repeat_;  
const int64_t sequence_; // 用于唯一标识 Timer 实例，配合 Timer* 使用

static AtomicInt64 s_numCreated_; // 用于生成 sequence_  
};
```

runEvery 接口在添加定时器时，会将 interval 参数设置为大于 0 的值，从而 Timer 对象的 repeat_ 成员为 true。在 TimerQueue::
reset 方法中，如果一个定时器的 repeat() 为 true 且未被取消，就会调用 restart() 更新其 expiration_ 并重新插入队列。

下面是添加重复执行任务的时序图,包括 Timer 对象的创建和 timerfd 的超时设置:

![重复任务](/images/重复任务.png)

## **总结**

muduo::net::TimerQueue 通过精巧的设计，实现了高效且与 EventLoop事件驱动模型完美集成的定时器管理机制：

1. **timerfd 的妙用：** 将时间事件转化为文件描述符事件，统一由 Poller 处理，使得定时器事件的处理与网络
   I/O 事件的处理路径一致。
2. **std::set 管理定时器：** 利用 std::set<std::pair<Timestamp, Timer*>>
   自动按到期时间排序的特性，使得获取最早到期定时器 (timers_.begin()) 和查找指定范围的到期定时器 (lower_bound) 非常高效。
3. **双集合管理 (timers_ 和 activeTimers_)：** timers_ 用于按时间排序和获取到期任务，activeTimers_ (以 Timer*
   和序列号为键) 用于高效地取消定时器。通过断言 timers_.size() == activeTimers_.size() 保证两者的一致性。
4. **线程安全：** 所有对 TimerQueue 内部状态的修改都通过 loop_->runInLoop() 保证在其所属的 EventLoop 线程中执行，确保了线程安全。
5. **处理回调期间的取消：** 通过 callingExpiredTimers_ 标志和 cancelingTimers_
   集合，优雅地处理了在执行定时器回调期间，这些定时器（特别是重复定时器）又被用户请求取消的复杂情况。
6. **资源管理：** Timer 对象通过 new 创建，并在不再需要时（非重复到期、或被取消、或 TimerQueue 析构时）通过 delete
   释放。

