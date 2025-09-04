---
title: "coredump的那些事:03.调试信息的生成"
date: 2025-09-04
cover: /images/cover/coredump_cover.jpeg
categories:
   - 程序员的自我修养
   - coredump
tags:
  - coredump
  - linux
  - 问题分析
---
## 前言

在之前的文章中，我们已经深入的的讲解了**coredump文件的生成过程**以及**coredump的使用**。我们也得到了一个核心的结论：调试程序的时候，只需要个三个关键的信息
* 可执行的程序
* coredump文件
* 调试信息

针对其中的coredump的相关内容，我们已经进行了详细的讲解，这篇文章我们将详细的讲解关于调试信息的相关内容。

在本篇文章中你将了解到:
* **DWARF调试格式**
* **为什么strip之后gdb什么都看不到**
* **调试信息的生成与使用**

**注:本文使用的Linux是ubuntu22.04，gcc版本是13.1**
## 问题的引出
### 无符号问题

现在很多同学都会遇到这种情况,自己本地写程序，出现死机问题了，直接IDE本地调试，然后去查IDE为什么能调试，知道是使用了 gdb，也了解了coredump。

等到工作了，遇到了这种 coredump问题，被安排去练手，因为有coredump文件的问题其实难度是比较低的，然后根据自己的经验使用 gdb调试，结果发现：

* **没有源码行号**

* **没有调用栈**

* **没有变量名**

一脸懵逼，甚至怀疑自己用的是“假的 gdb”。

实际上，这并不是 gdb 的问题，而是因为生产环境里的程序几乎总是：

* **Release 模式编译**

* **strip 过符号**

去减小文件大小和提升性能。

此时生成的ELF文件缺少了调试信息，gdb只能通过coredump文件的PT_NODE和PT_LOAD，没有办法映射到源代码。

本地开发的时候，很多人默认使用的是Debug模式，此时默认就保留了调试信息，所以一切正常。自己使用ide的时候开启release模式，无法直接使用断点调试与这个问题其实是一致的，都是缺少了调试信息 。

### -g参数的作用

还是从一个实际的例子出发，我们沿用上一篇文章的coredump代码

// crash.cpp
```c++
void corrupt_heap()
{
    int *p = nullptr;
    *p = 10;   // 崩溃点
}

int main()
{
    corrupt_heap();
    return 0;
}
```

使用如下命令去产生带调试信息和不带调试信息的可执行文件:
```bash
## 产生带调试信息的
g++ -g -o crash_debug crash.cpp

## 产生不带调试信息的
g++ -o crash_no_debug crash.cpp
```

使用 `readelf -h` 命令分别查看两个文件的程序头

```text

readelf -h crash_debug

## 输出如下
ELF 头：
  Magic：   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  类别:                              ELF64
  数据:                              2 补码，小端序 (little endian)
  ......
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         13
  Size of section headers:           64 (bytes)
  Number of section headers:         35
  Section header string table index: 34
```

```text
readelf -h crash_no_debug

ELF 头：
  Magic：   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  类别:                              ELF64
  数据:                              2 补码，小端序 (little endian)
  ......
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         13
  Size of section headers:           64 (bytes)
  Number of section headers:         29
  Section header string table index: 28
```

通过比较可以发现二者的输出结果主要区别在于大小和`Number of section headers`,因为程序头主要是用来加载可执行程序的，有无调试信息二者的区别并不大。

然后，我们通过`readelf -S`命令去查看节点头(sections)

```text
readelf -S crrash_no_debug

There are 29 section headers, starting at offset 0x3690:

节头：
  [号] 名称              类型             地址              偏移量
       大小              全体大小          旗标   链接   信息   对齐
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  .....
  [26] .symtab           SYMTAB           0000000000000000  00003040
       0000000000000360  0000000000000018          27    18     8
  [27] .strtab           STRTAB           0000000000000000  000033a0
       00000000000001de  0000000000000000           0     0     1
  [28] .shstrtab         STRTAB           0000000000000000  0000357e
       000000000000010c  0000000000000000           0     0     1
.....
```

```text
readelf -S crash_debug

There are 35 section headers, starting at offset 0x39a0:

节头：
  [号] 名称              类型             地址              偏移量
       大小              全体大小          旗标   链接   信息   对齐
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  .......
  [26] .debug_aranges    PROGBITS         0000000000000000  0000303b
       0000000000000030  0000000000000000           0     0     1
  [27] .debug_info       PROGBITS         0000000000000000  0000306b
       000000000000008a  0000000000000000           0     0     1
  [28] .debug_abbrev     PROGBITS         0000000000000000  000030f5
       000000000000006d  0000000000000000           0     0     1
  [29] .debug_line       PROGBITS         0000000000000000  00003162
       000000000000005c  0000000000000000           0     0     1
  [30] .debug_str        PROGBITS         0000000000000000  000031be
       00000000000000b4  0000000000000001  MS       0     0     1
  [31] .debug_line_str   PROGBITS         0000000000000000  00003272
       000000000000008a  0000000000000001  MS       0     0     1
  [32] .symtab           SYMTAB           0000000000000000  00003300
       0000000000000360  0000000000000018          33    18     8
  [33] .strtab           STRTAB           0000000000000000  00003660
       00000000000001de  0000000000000000           0     0     1
  [34] .shstrtab         STRTAB           0000000000000000  0000383e
       000000000000015c  0000000000000000           0     0     1
.....

```

可以发现二者的主要区别就在于，带调试信息的ELF文件多了很多 `debug_*` 段，这些段就包含了调试信息，内核在加载的时候会忽略这些，但是gdb调试器会读取这些信息。

## DWARF 调试信息讲解
之前一直使用调试信息这个通俗的叫法，实际上这个调试信息有专有的名词：DWARF（Debugging With Attributed Record Formats），是Linux ELF的标准调试格式。

为什么strip之后gdb调试器就不能直接显示源码行数了？就是因为这些段被剥离了。

在这里简单的介绍DWARF一些常用的段,更加详细，权威的描述参考 https://dwarfstd.org/doc/DWARF5.pdf。

* debug_info：核心 DIE 树，描述编译单元、函数（DW_TAG_subprogram）、变量（DW_TAG_variable）、类型（DW_TAG_class_type for C++ 类）。作用：GDB 解析类型/符号，显示变量值（如 std::vector 的长度/容量）。
* debug_abbrev：缩写表，压缩 .debug_info 的 DIE 格式。作用：加速 GDB 解析，减少冗余。
* debug_line：行号表（状态机编码），映射地址到源行/文件。作用：GDB 用 RIP（从 core）找源代码行，支持断点/步进。
* debug_frame：CFI，描述栈帧布局/寄存器恢复。作用：GDB 栈回溯（bt），展开调用链。
* debug_str：字符串表（名称/路径）。作用：提供可读字符串，GDB 显示函数/变量名。

## 调试信息的处理
在这个小节就简单介绍，处理符号文件的几个简单的命令。符号文件的专业管理
### 调试符号的剥离

```bash
objcopy --only-keep-debug crash crash.dbg  # 提取符号到crash.dbg
objcopy --strip-debug crash                # 剥离crash中的调试信息
```
crash为生产用精简二进制，crash.dbg为独立符号文件。objcopy优于strip，因为它保留build-id（位于.note.gnu.build-id），便于GDB匹配。

C++程序因模板和类膨胀严重，可用-gsplit-dwarf生成.dwo文件，进一步分离符号。压缩符号文件：dwz crash.dbg可减少50%体积。自动化建议：在CI/CD（如Jenkins）中加入脚本，确保每次构建生成匹配的符号文件：

```bash
# 示例脚本
objcopy --only-keep-debug crash crash.dbg
objcopy --strip-debug crash
dwz crash.dbg
```
### 调试信息的链接
符号文件因为包含了很多敏感信息，实际上都是被管理起来的，比如debuginfod这个开源符号管理器。

可以通过在本地配置debuginfod的服务器地址，可以自动拉取这个符号文件。
```c++
export DEBUGINFOD_URLS="https://your.debuginfod.server"
```

当然，你要是很喜欢手动操作一切,可以直接使用。
```bash
objcopy --only-keep-debug crash crash.dbg
objcopy --strip-debug --add-gnu-debuglink=crash.dbg crash
```
将符号文件去嵌入到可执行程序中。

## 源码解读
在这一小节我们就通过gcc的源码和gdb的源码简单的看一下DWARF文件的生成过程与使用过程。
### GCC 如何生成 DWARF

在 GCC 中，调试信息的生成发生在 **后端阶段**，核心代码在 `gcc/dwarf2out.c`。

#### 整体流程

1. **前端 (parser)**：构建 AST，每个变量/函数是一个 `tree` 节点。
2. **后端 (expand/final)**：在生成汇编时调用 `dwarf2out_*` 系列函数。
3. **dwarf2out.c**：将 `tree` 转换为 **DIE（Debugging Information Entry）**，写入 `.debug_info` 等段。

#### 关键函数片段

**变量/函数声明：**

```C++
static void
dwarf2out_decl (tree decl)
{
  dw_die_ref context_die = comp_unit_die ();

  // 处理各种情况,函数，变量，类型等等
  switch (TREE_CODE (decl))
    {
    case ERROR_MARK:
      return;

    case FUNCTION_DECL:
      ....
      if (early_dwarf
	  && decl_function_context (decl)
	  && debug_info_level > DINFO_LEVEL_TERSE)
	context_die = NULL;
      break;

    case VAR_DECL:
      if (local_function_static (decl))
	context_die = lookup_decl_die (DECL_CONTEXT (decl));
     
      if (debug_info_level < DINFO_LEVEL_TERSE
	  || (debug_info_level == DINFO_LEVEL_TERSE
	      && !TREE_PUBLIC (decl)))
	return;
      break;
    ......
    default:
      return;
    }
  
  // 生成调试信息
  gen_decl_die (decl, NULL, NULL, context_die);

  ......
}
```

每一个 C++ 里的变量/函数声明，最终都会被转成一个 **DIE 节点**，并写到 `.debug_info`。

**行号信息：**

```c++
static void
dwarf2out_source_line (unsigned int line, unsigned int column,
		       const char *filename,
                       int discriminator, bool is_stmt)
{
  unsigned int file_num;
  dw_line_info_table *table;
  static var_loc_view lvugid;

  // 如果 debug 信息等级太低，或者 DWARF 没启用，直接返回，不输出任何行号信息
  if (debug_info_level < DINFO_LEVEL_TERSE || !dwarf_debuginfo_p ())
    return;

  table = cur_line_info_table;

  ......
  // 更新行号信息
  table->file_num = file_num;
  table->line_num = line;
  table->column_num = column;
  table->discrim_num = discriminator;
  table->is_stmt = is_stmt;
  table->in_use = true;
}
```

建立 **地址 → 源码行号** 的映射，写到 `.debug_line`，这是为什么 gdb 能显示 `list` 和断点对应到源码行。

### GDB 如何使用 DWARF

GDB在调试时读取ELF中的DWARF信息，主要由gdb/dwarf2/read.c负责。该文件实现了DWARF解析器，将节加载到内存，构建符号表、行号表等，用于断点、变量检查等。

#### 整体流程

1. **加载 ELF**：使用 `bfd` 打开 ELF，找到 `.debug_*` 段。
2. **dwarf2read.c**：解析 `.debug_info`、`.debug_line`，构建符号表。
3. **symtab.c**：提供 `lookup_symbol` 等接口，供用户操作（如 `break main`、`print a`）。

#### 关键函数片段

**初始化符号表：**

```C++
// gdb/dwarf2/read.c
bool
dwarf2_initialize_objfile (struct objfile *objfile,
			   const struct dwarf2_debug_sections *names,
			   bool can_copy)
{
  // 检查是否具有可用的调试
  if (!dwarf2_has_info (objfile, names, can_copy))
    return false;

  // 解析使用的上下文对象
  dwarf2_per_objfile *per_objfile = get_dwarf2_per_objfile (objfile);
  dwarf2_per_bfd *per_bfd = per_objfile->per_bfd;

  dwarf_read_debug_printf ("called");
   
   // 具体的解析函数与策略
  if ((objfile->flags & OBJF_READNOW))
  ......

  return true;
}
```

加载 ELF 文件时，GDB 会调用这个函数，开始读取 DWARF。

**符号查找：**

```c++
// gdb/symtab.c
// 从一个作用域中找符号
static struct block_symbol
lookup_symbol_aux (const char *name, symbol_name_match_type match_type,
		   const struct block *block,
		   const domain_search_flags domain, enum language language,
		   struct field_of_this_result *is_a_field_of_this)
{
  ......

  // 局部作用域

  result = lookup_local_symbol (name, match_type, block, domain, language);
  if (result.symbol != NULL)
    {
      symbol_lookup_debug_printf
	("found symbol @ %s (using lookup_local_symbol)",
	 host_address_to_string (result.symbol));
      return result;
    }

  langdef = language_def (language);

  // 当成成员变量去找
  if (is_a_field_of_this != NULL && (domain & SEARCH_STRUCT_DOMAIN) == 0)
    {
      result = lookup_language_this (langdef, block);

      if (result.symbol)
	{
	  ......

	  if (check_field (t, name, is_a_field_of_this))
	    {
	      symbol_lookup_debug_printf ("no symbol found");
	      return {};
	    }
	}
    }
  
  // 全局去找
  result = langdef->lookup_symbol_nonlocal (name, block, domain);
  if (result.symbol != NULL)
    {
     ....
      return result;
    }
  
  // 静态符号兜底
  result = lookup_static_symbol (name, domain);
  symbol_lookup_debug_printf
    ("found symbol @ %s (using lookup_static_symbol)",
     result.symbol != NULL ? host_address_to_string (result.symbol) : "NULL");
  return result;
}
```

当你在 gdb 输入 `break main` 或 `print a` 时，GDB 就利用这个函数查找符号。

## 总结

本篇文章围绕 **`-g` 参数与调试信息** 展开：

从gdb 看不到行号、变量名这一问题出发，说明了：
* 生产环境中默认编译成 release 并 strip，导致 ELF 缺少 `.debug_*` 段

并通过实际的例子，讲解了-g参数的作用：
* `-g` 参数告诉编译器生成 **DWARF 调试信息**，写入 ELF 的 `.debug_info`、`.debug_line` 等段，供调试器使用

并介绍了符号处理的几个命令:
*  使用 `readelf -S` 可以验证调试段是否存在；
* `strip` 会移除调试段；
* `objcopy --only-keep-debug` 可以分离调试符号文件，并配合 `--add-gnu-debuglink` 或 debuginfod 使用。

最后，简单看一下 gcc和gdb的相关源码：
* **GCC** 在 `dwarf2out.c` 中生成调试信息，将变量、函数、行号写入 DWARF 段；
* **GDB** 在 `dwarf2read.c` 中解析调试信息，构建 DIE 树，支持 `bt`、`print`、`break` 等命令。

从开发者的角度看：
- **没有 `-g`：** gdb 只能看到裸地址和汇编。
- **有 `-g`：** gdb 能直接映射到源码行、变量、函数名。
- **最佳实践：** 在生产环境保留分离式符号文件，在本地或符号服务器上调试，兼顾安全与可用性。

> **一句话总结：**
> `-g` 参数是 gdb 从“只能看十六进制地址”到“直接跳到源码行”的根本原因，背后依赖的就是 DWARF 格式调试信息。





