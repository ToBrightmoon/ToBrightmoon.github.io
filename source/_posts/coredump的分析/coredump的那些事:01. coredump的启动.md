---
title: "coredump的那些事:01.coredump的启动"
date: 2025-08-20
categories:
   - 程序员的自我修养
   - coredump
tags:
  - coredump
  - linux
  - 问题分析
---
## 前言

在使用c/c++进行程序开发，或者类型IL2CPP这种将Unity的应用编译成原生应用的时候，不管开发人员多么小心，不可避免的会出现的 *段错误* 导致的死机现象。当出现这种问题的时候，

一个最常用的手段就是利用coredump辅助日志文件进行问题分析，这篇文章就来讲一讲如何修改Linux中的一些配置，让你的系统可以产生coredump文件以及相关的知识。

*注：本文的一些终端操作均在ubuntu22.04下进行*
<!--more-->
## coredump介绍

### coredump简介

关于coredump的介绍， 可以使用 `man 5 core` 命令，在终端 shell中查看 coredump 的手册
```text

       The  default  action of certain signals is to cause a process to terminate and produce a core dump file, a file containing an image of the process's memory at the time
       of termination.	This image can be used in a debugger (e.g., gdb(1)) to inspect the state of the program at the time that it terminated.	 A list of the	signals	 which
       cause a process to dump core can be found in signal(7).
       
       ...
```
上面的英文，使用中文简单总结就是:

*当一个程序没有处理收到的信号导致异常退出时，操作系统会生成一个该进程的内存镜像，这个内存镜像中保存着可以供gdb调试器调试的信息*

从上面的介绍中可以得到一些信息:

1. 没有处理一些信号会导致进程终止并产生coredump文件
2. coredump是操作系统产生的
3. coredump可以供gdb调试器使用

核心信息是内存镜像（栈、寄存器、堆等状态），它的体积可能非常大，所以系统默认不开启。

### 产生coredump生成的信号

那么到底那些信号会导致coredump的产生呢? 我们使用 `man 7 signal` 这个命令去查看信号的手册，在 `Standard signals` 小节中有着相关的内容，相关的内容如下
```text
 Standard signals
       Linux  supports  the  standard signals listed below.  
       ......

       Signal      Standard   Action   Comment
       ────────────────────────────────────────────────────────────────────────
       SIGABRT      P1990      Core    Abort signal from abort(3)
       SIGBUS       P2001      Core    Bus error (bad memory access)
       SIGFPE       P1990      Core    Floating-point exception
       SIGILL       P1990      Core    Illegal Instruction
       SIGIOT         -        Core    IOT trap. A synonym for SIGABRT
       SIGQUIT      P1990      Core    Quit from keyboard
       SIGSEGV      P1990      Core    Invalid memory reference
       SIGSYS       P2001      Core    Bad system call (SVr4);
                                       see also seccomp(2)
       SIGTRAP      P2001      Core    Trace/breakpoint trap
       SIGUNUSED      -        Core    Synonymous with SIGSYS[已经废弃]
       SIGXCPU      P2001      Core    CPU time limit exceeded (4.2BSD);
                                       see setrlimit(2)
       SIGXFSZ      P2001      Core    File size limit exceeded (4.2BSD);
                                       see setrlimit(2)
```
这些信号的默认处理方式就是 Core Dump，比如 SIGSEGV（非法内存访问）、SIGABRT（abort 触发），而 SIGKILL、SIGTERM 等则不会生成 core dump。

具体的coredump的产生的细节，可以参考知乎上的一篇文章
![coredump生成原理](https://zhuanlan.zhihu.com/p/626026606)

## coredump的配置

之前的部分介绍了coredump的信息，这个部分正式进入本篇文章的主题，介绍如何生成修改Linux中的相关配置。
### coredump生成的限制
Linux系统默认情况下一个进程崩掉的时候，是不会产生coredump文件，至于具体原因，可以同样使用 `man 5 core`这个命令去查看coredump的手册，去找到相关的内容:

```text
       A process can set its soft RLIMIT_CORE resource limit to place an upper limit on the size of the core dump file that will be produced if it receives a "core dump" sig‐
       nal; see getrlimit(2) for details.

       There are various circumstances in which a core dump file is not produced:

       *  The process does not have permission to write the core file.	(By default, the core file is called core or core.pid, where pid is the ID of the process that	dumped
	  core, and is created in the current working directory.  See below for details on naming.)  Writing the core file fails if the directory in which it is to be created
	  is not writable, or if a file with the same name exists and is not writable or is not a regular file (e.g., it is a directory or a symbolic link).

       *  A (writable, regular) file with the same name as would be used for the core dump already exists, but there is more than one hard link to that file.

       ......
```

将手册中的内容简单使用中文总结就是:

有多种情况会导致 **不生成 core dump 文件**：

* 进程没有权限写 core 文件（默认文件名为 `core` 或 `core.pid`，存放在当前工作目录）。
* 目标文件已存在并有多个硬链接。
* 文件系统写满 / inode 用尽 / 挂载为只读 / 用户超出磁盘配额。
* 目录不存在。
* **RLIMIT_CORE** 或 **RLIMIT_FSIZE** 被设置为 0。
* 可执行文件没有读取权限（安全措施）。
* 程序以 set-user-ID / set-group-ID 或带有文件 capabilities 运行（除非通过 **prctl(2) PR_SET_DUMPABLE** 和 `/proc/sys/fs/suid_dumpable` 允许）。
* `/proc/sys/kernel/core_pattern` 为空 且 `/proc/sys/kernel/core_uses_pid` = 0。
* 内核编译时未启用 **CONFIG_COREDUMP**（Linux 3.7+）。
* 进程使用了 `madvise(MADV_DONTDUMP)` 屏蔽了部分地址空间。
* 在 systemd 系统中，core dump 可能由 **systemd-coredump(8)** 接管。

那么想要修改操作系统中的配置，让进程崩溃时能够产生coredump文件简单的说就是要让进程运行的环境中避免这种限制:

针对系统本身:

1. 磁盘空间足够，支持生成coredump。
2. 生成coredump的大小在受限范围内。
3. 生成文件的目录存在。

针对进程：

1. 进程未被systemd接管，没有屏蔽 地址空间
2. 有相关位置的读写权限。

我们大部分时候需要操作的，就是修改下系统的coredump大小限制和coredump的生成位置(/proc/sys/kernel/core_pattern)，保证进程有权限在相关的位置下进行读写。更加复杂的情况，请读者根据手册上的相关内容自行探索。

此外，linux还支持通过自定义coredump的name和利用管道对coredump进行相关操作，相关的内容
### 操作步骤
#### 临时开启
linux提供了`ulimit`这个命令去修改coredump的限制，相关的内容可以通过 `man ulimit`去查阅。

```bash
## 打开终端,修改系统限制为不限制 coredumop大小
## 不要关闭终端，因为ulimit命令只在当前终端下生效。
ulimit -c unlimited

## 将当前的coredump输出目录修改成你的进程有权限读写的位置
## 其中 %e和%p 是linux默认支持的将coredump的文件名命名为 程序名和进程id的占位符
sudo bash -c 'echo "/path/%e-%p.core" > /proc/sys/kernel/core_pattern'

## 在此终端中的启动的进程输出的coredump就会最终保存在你设置的路径下
```
#### 长久开启

想要长久的配置coredump文件的启动，需要修改 `/etc/security/limits.conf`和 `/etc/sysctl.conf` 文件，并且配置一个systemd服务，保证系统重启的时候依旧有效

```bash
# 使用gedit工具打开limits.conf文件 
sudo gedit /etc/security/limits.conf 

# 在文件的最后一行增加如下内容,注意*不能省略
*               soft    core            unlimited 

# 编辑/etc/sysctl.conf文件
sudo gedit /etc/sysctl.conf

# 在文件的最后一行增加如下内容

kernel.core_pattern=/path/%e-%p.core
```

配置启动服务，每次开机后使得/etc/sysctl.conf配置文件发挥作用

创建服务文件:
```bash
sudo gedit /etc/systemd/system/sysctl-reload.service
```

文件内容如下所示
```systemd
[Unit]
Description=Reload sysctl settings after all services have started
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/sysctl -p /etc/sysctl.conf
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

执行如下命令配置服务
```bash
# 确保服务可以开机自动启动
sudo systemctl enable sysctl-reload.service

# 重新加载“systemd”配置
sudo systemctl daemon-reload

# 启动服务
sudo systemctl start sysctl-reload.service

# 查看服务启动的结果
sudo systemctl status sysctl-reload.service
```

### 配套的脚本
读到这里的时候，我相信其实你已经基本知道了coredump配置的基本原理，能够*知其然,也知其所以然*,针对自己需要的情况进行配置。但是为了方便读者使用，本文提供了一个脚本去开启本机的coreumdp设置。

这个脚本能够长久的配置coredump文件以 程序名-进程id-coredump产生时间的格式保留在 `~/coredump` 目录下。

```bash
#!/bin/bash

# 获取当前用户的用户名
USERNAME=$(whoami)
CORE_PATH="/home/$USERNAME/coredump"

# 创建coredump存储目录
mkdir -p $CORE_PATH

# 定义新脚本的文件名和路径
SCRIPT_PATH="$CORE_PATH/rename_core.sh"

# 使用重定向符号将内容写入新脚本
cat << EOF > $SCRIPT_PATH
#!/bin/bash
COREFILE_DIR="/home/$USERNAME/coredump"
EOF

cat << 'EOF' >> $SCRIPT_PATH
TIMESTAMP="$(date +%Y-%m-%d-%H-%M-%S)"
COREFILE_NAME="${1}-${2}-$TIMESTAMP.core"
# 创建coredump文件
cat > "$COREFILE_DIR/$COREFILE_NAME"
EOF

chmod +x $SCRIPT_PATH

# 永久设置coredump文件大小限制
sudo bash -c  'echo "* hard core unlimited" >> /etc/security/limits.conf'
sudo bash -c 'echo "* soft core unlimited" >> /etc/security/limits.conf'


# 确保设置在重启后仍然有效
echo "Making changes persistent..."

sudo bash -c 'echo "kernel.core_pattern=|'$SCRIPT_PATH' %e %p" >> /etc/sysctl.conf'

SERVICE="/etc/systemd/system/sysctl-reload.service"

sudo bash -c 'echo "[Unit]
Description=Reload sysctl settings after all services have started
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/sysctl -p /etc/sysctl.conf
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target" > '$SERVICE''



# 确保服务可以开机自动启动
sudo systemctl enable sysctl-reload.service

# 重新加载“systemd”配置
sudo systemctl daemon-reload

# 启动服务
sudo systemctl start sysctl-reload.service

# 查看服务启动的结果
sudo systemctl status sysctl-reload.service
```

## 相关部分的部分内核源码解读
之前查看手册看了那么多coredump生成的限制和，coredump生成的修改方式，这一小节我们就从源码中看一下，Linux内核中负责生成coredump的函数 do_coredump到底做了生成。

这个函数的位置是 linux内核中的 `fs/coredump.c`中。我查看的源码的分支是 v6.9,读者也可以从 `https://elixir.bootlin.com/linux/v6.9/source/fs/coredump.c#L635` 这个网址查看。

### 初始化参数

```c++
struct mm_struct *mm = current->mm; // 进程的内存描述符，保存了进程的虚拟内存信息
struct linux_binfmt * binfmt;
......
struct coredump_params cprm = {
	.siginfo = siginfo,
	.limit = rlimit(RLIMIT_CORE), // coredump 文件大小限制，来自 RLIMIT_CORE（也就是 ulimit -c 设置的值）
	.mm_flags = mm->flags,
	.vma_meta = NULL,
	.cpu = raw_smp_processor_id(),
};
```

这一段代码初始化了生成coredump需要的一些基础信息，rlimit这个函数会读取当前任务的coredump大小的限制

### 简单的检查

如果进程没有合法的 binfmt，或者没有实现 core_dump 方法，直接失败
```c++
	binfmt = mm->binfmt;
	if (!binfmt || !binfmt->core_dump)
		goto fail;
	if (!__get_dumpable(cprm.mm_flags))
		goto fail;
```

### 输出方式的决定

如果 /proc/sys/kernel/core_pattern 以 | 开头，那么内核会通过管道把 core dump 交给用户态的 helper 程序处理。 否则，就会直接在文件系统里创建一个 core 文件并写入
```c++
ispipe = format_corename(&cn, &cprm, &argv, &argc);

if (ispipe) {
   ......
    /* 管道模式，把 core dump 交给用户态程序处理 */
    sub_info = call_usermodehelper_setup(helper_argv[0],
                        helper_argv, NULL, GFP_KERNEL,
                        umh_pipe_setup, NULL, &cprm);
    retval = call_usermodehelper_exec(sub_info, UMH_WAIT_EXEC);
} else {
    .....
    // 检查文件的大小和suid模式下是否是绝对路径
    if (cprm.limit < binfmt->min_coredump)
			goto fail_unlock;

		if (need_suid_safe && cn.corename[0] != '/') {
			printk(KERN_WARNING "Pid %d(%s) can only dump core "\
				"to fully qualified path!\n",
				task_tgid_vnr(current), current->comm);
			printk(KERN_WARNING "Skipping core dump\n");
			goto fail_unlock;
		}
    /* 文件模式，直接写入 core 文件 */
    cprm.file = filp_open(cn.corename, open_flags, 0600);
}

```

```c++
static int format_corename(struct core_name *cn, struct coredump_params *cprm,
			   size_t **argv, int *argc)
{
	....
	int ispipe = (*pat_ptr == '|'); // 就是通过core_pattern 是否是| 决定的
	bool was_space = false;
	.....

	// 一些coredump name的输出配置
	while (*pat_ptr) {
		...
		if (*pat_ptr != '%') {
			err = cn_printf(cn, "%c", *pat_ptr++);
		} else {
			switch (*++pat_ptr) {
			/* single % at the end, drop that */
			case 0:
				goto out;
			/* Double percent, output one percent */
			case '%':
				err = cn_printf(cn, "%c", '%');
				break;
			/* pid */
			case 'p':
				pid_in_pattern = 1;
				err = cn_printf(cn, "%d",
					      task_tgid_vnr(current));
				break;
			...
		}

		....
	}

out:
    ......
	return ispipe;
}
```
这部分源码解释了为什么 core_pattern 的配置能决定 core dump 的“去向”

### 写入内存镜像
遍历进程的虚拟内存映射，收集要写的内存段（堆、栈、代码段等,并且使用core_dump这个方法将coredump文件按照需要的格式保存到文件中
```c++
if (!dump_vma_snapshot(&cprm))
		goto close_fail;

	file_start_write(cprm.file);
	core_dumped = binfmt->core_dump(&cprm);
	if (cprm.to_skip) {
		cprm.to_skip--;
		dump_emit(&cprm, "", 1);
	}
	file_end_write(cprm.file);
	free_vma_snapshot(&cprm);
```

### 收尾
关闭一些文件,回收些资源
```c++
if (cprm.file)
filp_close(cprm.file, NULL);

coredump_finish(core_dumped);
revert_creds(old_cred);
put_cred(cred);
```

## 总结

本文围绕 **coredump 的生成与配置** 展开，主要内容如下：

1. **coredump 的本质**
    - 操作系统在进程异常退出时生成的内存镜像文件
    - 可用于 gdb 等工具还原程序当时的运行状态

2. **生成条件与限制**
    - 信号触发：SIGSEGV、SIGABRT、SIGBUS 等
    - 限制因素：`RLIMIT_CORE`、权限、文件系统、`core_pattern` 等

3. **配置方法**
    - 临时配置：`ulimit -c unlimited` + 修改 `/proc/sys/kernel/core_pattern`
    - 长期配置：编辑 `/etc/security/limits.conf`、`/etc/sysctl.conf`，配合 systemd 保持生效

4. **内核实现**
    - `do_coredump()` 是核心函数
    - 负责检查限制、解析 core_pattern、最终写入 core 文件
    - 从源码角度验证了手册与实际操作的正确性

