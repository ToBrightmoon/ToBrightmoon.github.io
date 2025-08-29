---
title: "spdlog源码阅读:04.format格式化引擎分析"
date: 2025-05-11
categories: 
  - 源码分析
  - spdlog
tags:
  - C++
  - spdlog
  - 日志系统
---
## 引言

在本系列的前三篇文章中，我们依次探讨了 spdlog 的核心异步日志机制、两种常用的内建文件 Sink（`daily_file_sink` 和 `rotating_file_sink`），以及如何通过实现自定义 Sink（以 `compressed_file_sink` 为例）来扩展其功能。至此，我们已经对日志消息的产生、处理、流转以及最终输出有了较为深入的理解。

现在我们开始剖析spdlog日志中的最后一个组件 `formmatter`。

`spdlog` 提供了强大的日志格式化功能，允许用户通过模式字符串 (pattern string) 精确控制日志输出的每一个细节，例如时间戳、日志级别、线程 ID、源代码位置以及日志消息本身。这一核心功能主要由 `pattern_formatter` 类负责。

本文将聚焦于 `spdlog` 的**核心格式化引擎 `pattern_formatter`**，剖析其如何将用户定义的模式字符串解析、编译，并最终应用于日志消息，生成格式化的输出。通过本文，你将理解：

* `pattern_formatter` 的两阶段工作机制：“编译”与“执行”。
* 模式字符串是如何被解析成一系列格式化单元 (`flag_formatter`) 的。
* `spdlog` 如何支持丰富的内置格式标志（如 `%Y`, `%m`, `%l`, `%v` 等）。
* `spdlog` 格式化引擎的扩展性：如何实现并注册自定义的格式标志。

**注：本文分析的源码基于 spdlog v1.15.1。**
<!-- more -->
## `pattern_formatter` 工作原理：**编译与执行**

`pattern_formatter` 的核心思路是将格式化的过程分为两个阶段：

1.  **编译阶段 (Compilation Phase):** 在创建 `pattern_formatter` 对象或设置新的模式字符串时执行。它会解析模式字符串，将其转换(或称“编译”)成一个内部的、由多个小型格式化单元 (`flag_formatter` 对象) 组成的序列。
2.  **执行阶段 (Execution/Formatting Phase):** 在每次需要格式化一条具体的日志消息 (`log_msg`) 时执行。它会按顺序执行“编译”阶段生成好的格式化单元序列，每个单元负责输出模式字符串中的一部分内容，最终拼接成完整的格式化日志。

下面我们详细分析这两个阶段。

## 编译阶段：解析模式串

编译阶段的目标是将用户提供的模式字符串（如 `"%Y-%m-%d %H:%M:%S.%e [%l] %v"`）转换成一个 `std::vector<std::unique_ptr<flag_formatter>>` 对象（即 `formatters_` 成员变量）。这个过程由 `pattern_formatter::compile_pattern_` 私有方法完成。

其主要逻辑如下：

1.  **遍历模式串:** 从头到尾逐个字符地扫描模式字符串。
```c++
void pattern_formatter::compile_pattern_(const std::string &pattern) {
    auto end = pattern.end();
    std::unique_ptr<details::aggregate_formatter> user_chars;
    formatters_.clear();
    //遍历整个模式字符串
    for (auto it = pattern.begin(); it != end; ++it) {
        ......
    }
    if (user_chars) 
    {
        formatters_.push_back(std::move(user_chars));
    }
}
```
2.  **处理普通字符:** 如果当前字符不是 `%`，则将其视为普通文本。
    * 创建一个 `details::aggregate_formatter` 实例（如果尚不存在）。
    * 调用 `aggregate_formatter::add_ch()` 方法将该普通字符追加到其内部字符串 `str_` 中。
    * 连续的普通字符会被追加到同一个 `aggregate_formatter` 实例。
```c++
void pattern_formatter::compile_pattern_(const std::string &pattern) {
    ......
    for (auto it = pattern.begin(); it != end; ++it) {
        if (*it == '%') {
           ......
        } else  //处理普通字符
        {
            if (!user_chars) {
                user_chars = details::make_unique<details::aggregate_formatter>();
            }
            user_chars->add_ch(*it);
        }
    }
    if (user_chars)  
    {
        formatters_.push_back(std::move(user_chars));
    }
}
```
3.  **处理模式标志 (`%`):** 如果当前字符是 `%`：
    * 首先，如果之前存在收集普通字符的 `aggregate_formatter` 实例，则将其添加到 `formatters_` 列表中。
    * 然后，尝试解析 `%` 后面的填充与对齐说明（padding spec），由 `handle_padspec_` 完成。
    * 接着，读取 `%` 后面的**标志字符** (flag character)，例如 `l`, `t`, `v`, `Y` 等。
    * 调用 `handle_flag_` 方法处理这个标志字符和解析出的填充信息。
```c++
void pattern_formatter::compile_pattern_(const std::string &pattern) {
    auto end = pattern.end();
    std::unique_ptr<details::aggregate_formatter> user_chars;
    formatters_.clear();
    for (auto it = pattern.begin(); it != end; ++it) {
        //处理模式字符
        if (*it == '%') {
            if (user_chars)  //先将之前的普通字符对象加入进去
            {
                formatters_.push_back(std::move(user_chars));
            }

            auto padding = handle_padspec_(++it, end);

            if (it != end) {
                if (padding.enabled()) {
                    handle_flag_<details::scoped_padder>(*it, padding);
                } else {
                    handle_flag_<details::null_scoped_padder>(*it, padding);
                }
            } else {
                break;
            }
        } else  
           ......
        }
    }
    if (user_chars)  
    {
        formatters_.push_back(std::move(user_chars));
    }
}
```
4.  **`handle_flag_` 的逻辑:**
    * **检查自定义标志:** 首先在 `custom_handlers_` (一个存储用户自定义标志处理器的 map) 中查找该标志字符。
        * 如果找到，说明用户注册了针对该字符的自定义格式化器。创建一个该自定义格式化器的克隆实例，设置好填充信息，并将其添加到 `formatters_` 列表中。**注意：用户自定义标志的优先级高于内置标志。**
    * **处理内置标志:** 如果不是自定义标志，则进入一个巨大的 `switch` 语句，根据标志字符匹配对应的内置 `flag_formatter` 子类。
        * 例如，`case 'l'` 会创建一个 `details::level_formatter` 实例；`case 'v'` 会创建一个 `details::v_formatter` 实例；各种时间相关的标志（`Y`, `m`, `d`, `H`, `M`, `S`, `e`, `f`, `F` 等）会创建对应的 `X_formatter` 实例。
        * 根据是否需要填充，会选择性地使用 `details::scoped_padder` 或 `details::null_scoped_padder` 作为模板参数。
        * 创建好对应的 formatter 实例后，将其添加到 `formatters_` 列表中。
    * **处理未知标志:** 如果标志字符既不是自定义的，也不在内置 `switch` 语句中，`spdlog` 默认会将其视为普通文本（连同前面的 `%` 一起）添加到 `aggregate_formatter` 中。

```c++
template <typename Padder>
SPDLOG_INLINE void pattern_formatter::handle_flag_(char flag, details::padding_info padding) {
    //处理自定义的模式字符，遇到直接退出，overrider原本定义的模式字符
    auto it = custom_handlers_.find(flag);
    if (it != custom_handlers_.end()) {
        auto custom_handler = it->second->clone();
        custom_handler->set_padding_info(padding);
        formatters_.push_back(std::move(custom_handler));
        return;
    }

    switch (flag) {
        case ('+'):  
            formatters_.push_back(details::make_unique<details::full_formatter>(padding));
            need_localtime_ = true;
            break;

        case 'n':  
            formatters_.push_back(details::make_unique<details::name_formatter<Padder>>(padding));
            break;

        case 'l':  
            formatters_.push_back(details::make_unique<details::level_formatter<Padder>>(padding));
            break;
        ......
        default:  
            auto unknown_flag = details::make_unique<details::aggregate_formatter>();

            if (!padding.truncate_) {
                unknown_flag->add_ch('%');
                unknown_flag->add_ch(flag);
                formatters_.push_back((std::move(unknown_flag)));
            }
        
            else {
                padding.truncate_ = false;
                formatters_.push_back(
                    details::make_unique<details::source_funcname_formatter<Padder>>(padding));
                unknown_flag->add_ch(flag);
                formatters_.push_back((std::move(unknown_flag)));
            }

            break;
    }
}
```
5.  **结束处理:** 遍历完整个模式字符串后，如果最后还有未添加的 `aggregate_formatter` 实例（表示模式串以普通字符结尾），则将其添加到 `formatters_` 列表末尾。

```c++
void pattern_formatter::compile_pattern_(const std::string &pattern) {
    ......
    //末尾的普通字符要保持
    if (user_chars)  
    {
        formatters_.push_back(std::move(user_chars));
    }
}
```
经过这个编译阶段，模式字符串就被有效地转换成了一个由 `flag_formatter` 对象组成的、有序的“格式化指令列表” `formatters_`。

## 执行阶段：格式化日志消息

当调用 `pattern_formatter::format(const details::log_msg &msg, memory_buf_t &dest)` 方法来格式化一条具体的日志消息时，执行阶段开始。

这个过程相对简单：

1.  **遍历 `formatters_` 列表:** 按顺序迭代编译阶段生成的 `formatters_` 向量中的每一个 `std::unique_ptr<flag_formatter>`。
2.  **调用 `format` 方法:** 对每一个 `flag_formatter` 对象，调用其**虚函数 `format(const details::log_msg &msg, const std::tm &tm_time, memory_buf_t &dest)`**。
    * `spdlog` 会预先计算好日志消息的时间戳对应的 `std::tm` 结构（如果模式中包含时间相关标志），并传递给 `format` 方法。
    * 每个具体的 `flag_formatter` 子类会实现自己的 `format` 方法，根据其职责从 `msg` 或 `tm_time` 中提取所需信息（如日志级别、线程 ID、格式化的时间部分、日志消息文本等），进行必要的处理和填充，并使用 `fmt_helper` 中的函数将结果**追加 (append)** 到传入的目标缓冲区 `dest` 中。
3.  **完成格式化:** 当 `formatters_` 列表中的所有对象都执行完其 `format` 方法后，`dest` 缓冲区中就包含了根据原始模式字符串生成的、完整的、格式化好的日志输出。

```c++
void pattern_formatter::format(const details::log_msg &msg, memory_buf_t &dest) {
    ......
    for (auto &f : formatters_) {
        f->format(msg, cached_tm_, dest);
    }
    details::fmt_helper::append_string_view(eol_, dest);
}
```
## 关键类与设计

`pattern_formatter` 的设计体现了良好的面向对象思想：

* **`flag_formatter` (基类):** 定义了所有格式化单元的统一接口（主要是 `format` 虚函数），是实现多态的基础。
* **`aggregate_formatter` (子类):** 处理模式串中的普通文本部分。
* **众多具体的 `X_formatter` (子类):** 如 `level_formatter`, `v_formatter`, `Y_formatter`, `H_formatter` 等，每个类负责处理一个特定的 `%` 格式标志，实现了单一职责原则。
* **`pattern_formatter` (协调者):** 负责解析模式串（编译过程），管理 `flag_formatter` 对象列表，并在需要时按顺序调用它们（执行过程）。

这种设计可以看作是**策略模式 (Strategy Pattern)** 的应用：每个 `%` 标志对应一种格式化策略，由一个具体的 `flag_formatter` 子类实现。`pattern_formatter` 在编译时根据模式串选择并组合这些策略，在执行时应用它们。同时，`formatters_` 列表也体现了**组合模式 (Composite Pattern)** 的思想，将简单的格式化单元组合成复杂的格式化逻辑。

## 扩展性：自定义格式标志

`spdlog` 的格式化引擎不仅功能丰富，还具有良好的扩展性，允许用户添加自己定义的格式标志。

实现步骤如下：

1.  **创建自定义 Formatter 类:** 创建一个新类，继承自 `spdlog::custom_flag_formatter`。
2.  **实现 `format` 方法:** 在新类中重写 `format` 方法，实现自定义的格式化逻辑。你可以从 `log_msg` 对象获取信息，进行处理，并将结果追加到 `dest` 缓冲区。
3.  **实现 `clone` 方法:** 实现一个 `clone` 方法，用于在编译阶段创建自定义 formatter 的实例。通常是返回 `std::make_unique<YourCustomFormatter>(*this)`。
4.  **注册自定义标志:** 获取 `pattern_formatter` 对象（或者通过 `spdlog::set_formatter` 设置一个新的），调用其 `add_flag<YourCustomFormatter>(flag_char)` 方法，将你的自定义 formatter 类与一个未被使用的字符（作为新的标志字符）关联起来。

**spdlog中的github上的示例**
```c++
#include "spdlog/pattern_formatter.h"
class my_formatter_flag : public spdlog::custom_flag_formatter
{
public:
    void format(const spdlog::details::log_msg &, const std::tm &, spdlog::memory_buf_t &dest) override
    {
        std::string some_txt = "custom-flag";
        dest.append(some_txt.data(), some_txt.data() + some_txt.size());
    }

    std::unique_ptr<custom_flag_formatter> clone() const override
    {
        return spdlog::details::make_unique<my_formatter_flag>();
    }
};

void custom_flags_example()
{    
    auto formatter = std::make_unique<spdlog::pattern_formatter>();
    formatter->add_flag<my_formatter_flag>('*').set_pattern("[%n] [%*] [%^%l%$] %v");
    spdlog::set_formatter(std::move(formatter));
}

```

完成注册后，`pattern_formatter` 在编译阶段遇到你指定的 `flag_char` 时，就会优先创建并使用你的 `YourCustomFormatter` 实例。如前所述，**自定义标志的优先级高于内置标志**，这意味着你可以用自定义实现覆盖掉 `spdlog` 的默认行为（但不建议覆盖常用标志，最好选择未使用或特殊的字符）。

## 总结

`spdlog` 的 `pattern_formatter` 通过巧妙的“编译-执行”两阶段机制，将用户定义的模式字符串高效地转换并应用于日志消息。其核心在于将模式串解析为一系列 `flag_formatter` 对象，每个对象负责处理模式的一部分。这种基于策略模式和组合模式的设计不仅实现了丰富的功能，还通过 `custom_flag_formatter` 提供了优秀的扩展性。

至此，我们已经完成了对 `spdlog` 核心组件——异步机制、内建 sink、自定义 sink 扩展以及核心格式化引擎 `pattern_formatter` 的剖析。



