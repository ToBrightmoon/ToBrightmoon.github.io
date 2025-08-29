---
title: "spdlog源码阅读:02.sink分析"
date: 2025-08-26
categories: 
  - 源码分析
  - spdlog
tags:
  - C++
  - spdlog
  - 日志系统
---
## 引言
上一篇文章讲解了主要spdlog的异步模式及其实现方式，其中讲到了spdlog中负责将日志输出到具体的地方的类是sink。这篇文章就会具体的分析daily_file_sink和rotating_file_sink的部分源码，分析下spdlog是怎么实现按日分割和按尺寸分割日志的。
<!-- more -->
# daily_file_sink
在每天特定的时间点创建新的日志文件
## 基础配置
在使用daily_file_sink的时候，有几个重要的构造函数的参数

**rotation_hour**:分割的小时

**rotation_minute**：分割的分钟

**truncate**:是否截断文件

**max_files**：最大的文件数目，设定为0的时候不限制文件个数，否则保留设置的max_files文件个数
## 实现原理
daily_file_sink是通过被动触发日志文件的分割的，当用户写入日志的时候，daily_file_sink会通过日志消息的时间和计算的需要转轮的时间判断是否需要将日志输出到新的文件中，并通过一个环形队列管理这些日志文件，当文件数量超过设定数目的时候，删除旧的文件。
### 计算分割时间点
```c++
log_clock::time_point next_rotation_tp_() {
        auto now = log_clock::now(); //获取当前时间
        tm date = now_tm(now);
        
        //使用配置的时间作为轮转日志的时间
        date.tm_hour = rotation_h_; 
        date.tm_min = rotation_m_;
        date.tm_sec = 0;
        auto rotation_time = log_clock::from_time_t(std::mktime(&date));
        //当前时间还未到计算出的轮转时间
        if (rotation_time > now) {
            return rotation_time;
        }
        //当前时间超过了设定的轮转时间，直接换到下一点时间点作为轮转时间
        return {rotation_time + std::chrono::hours(24)};
    }
```
每次sink初始化或者发生日志轮转的时候，就会调用这个函数计算下一次轮转日志的时间点。这个函数会使用用户配置的rotation_h_和rotation_m_，如果这个时间点过去了，就会那么就是明天的同一时间轮转。
比如，下午16点启动日志，但是设定是每天6点分割，那么下一次轮转的时间就是第二天的6点，本次不会分割。
### 文件管理
daily_file_sink通过max_files参数限制保留的旧日志文件的个数,并且通过filenames_q_队列维护这些文件，在初始化阶段使用init_filenames_q_去填充这个队列

```c++
void init_filenames_q_() {
        using details::os::path_exists;

        //环形队列存储文件名
        filenames_q_ = details::circular_q<filename_t>(static_cast<size_t>(max_files_));
        std::vector<filename_t> filenames;
        auto now = log_clock::now();
        
        //按照时间倒序查找现有的日志文件
        while (filenames.size() < max_files_) {
            //按照时间计算文件名
            auto filename = FileNameCalc::calc_filename(base_filename_, now_tm(now));
            
            //当前文件不存在，立刻停止
            if (!path_exists(filename)) {
                break;
            }
            filenames.emplace_back(filename);
            now -= std::chrono::hours(24);
        }
        // 将找到的文件名按反向顺序（最旧的在前）添加到队列中
        // 这确保了当队列满时，最旧的文件名位于队列前端，准备被删除。
        for (auto iter = filenames.rbegin(); iter != filenames.rend(); ++iter) {
            filenames_q_.push_back(std::move(*iter));
        }
    }
```
这个初始化过程巧妙地查找了前几天的现有日志文件，并将它们添加到队列中，使得找到的最旧的文件位于队列的前面，以便在队列满时首先被移除。
### 写入与转轮
当写入的时候，会根据日志消息的时间判断是否需要轮转，并且根据时间生成新的文件名，生成新的日志文件，写入日志消息，并且清理旧文件
```c++
 void sink_it_(const details::log_msg &msg) override {
        //根据日志时间进行判断是否需要转轮
        auto time = msg.time;
        bool should_rotate = time >= rotation_tp_;
        if (should_rotate) {
            auto filename = FileNameCalc::calc_filename(base_filename_, now_tm(time));
            
            //打开新文件，重新计算轮转时间
            file_helper_.open(filename, truncate_);
            rotation_tp_ = next_rotation_tp_();
        }
        memory_buf_t formatted;
        base_sink<Mutex>::formatter_->format(msg, formatted);
        file_helper_.write(formatted);

        // 轮转之后，并且max_files_>0，清理老文件
        if (should_rotate && max_files_ > 0) {
            delete_old_();
        }
    }
```

```c++
void delete_old_() {
        using details::os::filename_to_str;
        using details::os::remove_if_exists;

        filename_t current_file = file_helper_.filename();
        //队列满了，进行操作，删除最老的的文件
        if (filenames_q_.full()) {
            auto old_filename = std::move(filenames_q_.front());
            filenames_q_.pop_front(); //从队列中删除最老的
            bool ok = remove_if_exists(old_filename) == 0; //从磁盘删除
            if (!ok) {
                filenames_q_.push_back(std::move(current_file));
                throw_spdlog_ex("Failed removing daily file " + filename_to_str(old_filename),
                                errno);
            }
        }
        //当前的最新文件加入队列
        filenames_q_.push_back(std::move(current_file));
    }
```

本质上，daily_file_sink 确保日志按天分隔，在指定时间启动新文件，并可选地清理超过设定天数的旧文件。
# rotating_file_sink
日志文档达到固定的大小后，输出生成新的日志文件
## 基础配置
**max_size**：文件最大的字节数

**max_files**：最大的文件个数，0的时候，一直只有一个文件存在
## 实现原理
通过日志的写入被动触发日志文件的转轮，当原文件的大小和要写入的字节大小总和超过设定值的时候触发日志的轮转，生成新文件，并通过将日志文件重名维持文件个数
### 初始化
创建日志的时候，rotating_file_sink会打开基础的文件，它计算初始大小，并且如果 rotate_on_open 为 true 且文件不为空，可以选择立即执行一次轮转。
```c++
template <typename Mutex>
rotating_file_sink<Mutex>::rotating_file_sink(
    filename_t base_filename,
    std::size_t max_size,
    std::size_t max_files,
    bool rotate_on_open,
    const file_event_handlers &event_handlers)
    : base_filename_(std::move(base_filename)),
      max_size_(max_size),
      max_files_(max_files),
      file_helper_{event_handlers} {
      
      //配置参数的检查
    if (max_size == 0) {
        throw_spdlog_ex("rotating sink constructor: max_size arg cannot be zero");
    }

    if (max_files > 200000) {
        throw_spdlog_ex("rotating sink constructor: max_files arg cannot exceed 200000");
    }
    
    //为0的时候是原名打开日志文件
    file_helper_.open(calc_filename(base_filename_, 0));
    current_size_ = file_helper_.size(); 
    
    //打开的时候如果配置true并且文件大小不为0，开始轮转
    if (rotate_on_open && current_size_ > 0) {
        rotate_();
        current_size_ = 0;
    }
}
```
### 文件重命名
日志的轮转逻辑的核心在rotate_函数中，主要是处理重命名逻辑，根据max_files_参数维持备份文件
比如是3,日志文件的原始名是log.txt，会存在log.txt日志文件和log.1.txt,log.2.txt,log.3.txt等备份文件
```c++
// 示例: base_filename="log.txt", max_files=3
// 轮转顺序:
// 1. 删除 log.3.txt (如果存在) --> 实际是第2步重命名时覆盖，或者rename前删除
// 2. 将 log.2.txt 重命名为 log.3.txt
// 3. 将 log.1.txt 重命名为 log.2.txt
// 4. 将 log.txt 重命名为 log.1.txt
// 5. 重新打开 log.txt (截断) 以写入新日志

template <typename Mutex>
void rotating_file_sink<Mutex>::rotate_() {
    using details::os::filename_to_str;
    using details::os::path_exists;

    file_helper_.close(); // 首先关闭当前日志文件

    // 从最旧的备份文件索引开始，向下迭代到当前文件 (索引 0)
    for (auto i = max_files_; i > 0; --i) {
        filename_t src = calc_filename(base_filename_, i - 1); // 例如 log.1.txt (当 i=2), log.txt (当 i=1)
        if (!path_exists(src)) {
            continue; // 如果源文件不存在，则跳过
        }
        filename_t target = calc_filename(base_filename_, i); // 例如 log.2.txt (当 i=2), log.1.txt (当 i=1)

        // 将 src 重命名为 target。如果 target 已存在，会先删除它。
        if (!rename_file_(src, target)) {
            // 处理重命名失败 (重试逻辑并抛出异常)
            // ... (错误处理如提供的代码所示) ...
            file_helper_.reopen(true); // 即使重命名失败，也要截断日志文件以防超出限制！
            current_size_ = 0;
            throw_spdlog_ex("rotating_file_sink: failed renaming " + filename_to_str(src) + " to " + filename_to_str(target), errno);
        }
    }
    // 重新打开基础日志文件，并进行截断，以便重新开始写入
    file_helper_.reopen(true);
}
```
轮转是向后工作的：最旧的文件 (base_filename.max_files.ext) 被删除， 然后 base_filename.(max_files-1).ext 被重命名为 base_filename.max_files.ext，依此类推，直到当前的 base_filename.ext 被重命名为 base_filename.1.ext 。最后，base_filename.ext 被重新打开为一个空文件。
### 写入与轮转
当日志消息准备写入的时候，会首先格式化日志消息，然后计算新大小，检查是否应该进行轮转。需要轮转时flush文件然后轮转日志文件。完成之后写入日志消息。
```c++
template <typename Mutex>
void rotating_file_sink<Mutex>::sink_it_(const details::log_msg &msg) {
    memory_buf_t formatted;
    base_sink<Mutex>::formatter_->format(msg, formatted);
    auto new_size = current_size_ + formatted.size();

    if (new_size > max_size_) {
        file_helper_.flush();
        // 仅当文件非空时才轮转，避免对空文件的轮转
        if (file_helper_.size() > 0) { 
            rotate_();
            new_size = formatted.size(); // 新文件的大小就是这条消息的大小
        }
    }
    file_helper_.write(formatted);
    current_size_ = new_size;
}
```
此 sink 确保单个日志文件不会超过特定大小，通过保留分布在多个文件中的最新日志数据的滚动窗口来管理磁盘空间。
# 差异对比

| 特性             | daily_file_sink                   |         rotating_file_sink         |
|:-----------------|:---------------------------------:|:----------------------------------:|
| **触发条件**      | 固定时间点（如每日 00:00）          |       文件大小达到 `max_size` 时触发        |
| **文件名规则**    | 按时间生成（如 `app-2023-09-15.log`） |      基础名 + 序号（如 `app.log.1`）       |
| **旧文件处理**    | 删除超过 `max_files` 天数的文件      |         重命名后循环覆盖（保留固定数量备份）         |
| **时间关联性**    | 强（按时间归档）                   |             弱（按空间需求处理）             |
| **核心功能**      | 时间驱动的日志归档                 |           磁盘空间控制            |

