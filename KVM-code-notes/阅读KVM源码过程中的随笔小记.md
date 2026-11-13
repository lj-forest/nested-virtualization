# 阅读KVM源码过程中的随笔小记

阅读KVM源码方法：

使用宿主机（Windows11）上的vscode远程连接虚拟机（unbuntu 22.04），在宿主机上的vscode上进行阅读。

之前，我学习过我们课题组出的一本书《深入探析Linux系统虚拟化原理、实现与安全》，并做了笔记，知道虚拟化整个流程。然后跟着每个流程，一个一个函数学习，先让LLM按照功能模块解释每段代码，了解整体流程，然后详细了解每行代码的意思。并且自己完成一个流程的学习之后，会结合自己在学习源码过程中绘制的流程图、之前学习书记得笔记，以及我们课题组的书再重新梳理一遍。

> 但是学习时间有限，目前只完成了VCPU运行和部分VCPU退出的源码学习

用的1提示词（经多次修改后得出最满意的😎）：

```
简要解释一下这部分代码，每个代码的作用是什么？标准输出：每行代码+简要作用+涉及到的简要的基础知识。如果是注释的话，标准输出为：简要说明一下注释的内容。
```

为了实现可以跳转到方法的实现/具体应用位置需要完成以下配置（以下配置过程由DeepSeek Model辅助生成）：

（1）在虚拟机内安装clangd与编译内核数据库

```bash
sudo apt install -y clangd
clangd --version   # 确认版本，建议 ≥ 13

cd /path/to/linux          # 进入内核源码根目录
make defconfig             # 最小化配置，足以生成编译命令记录
make -j$(nproc)            # 实际编译；首次可能较慢

./scripts/clang-tools/gen_compile_commands.py		# 生成 compile_commands.json，clangd 的精确跳转依赖于此
ls -lh compile_commands.json		# 验证生成结果
```

（2）VS Code clangd 配置与冲突处理

clangd 与 Microsoft 的 C/C++ 插件会在 IntelliSense、代码补全和跳转上产生直接冲突，两者同时启用会导致“跳转时灵时不灵”甚至完全失效。

在远程环境的扩展面板中，找到 C/C++（或 C/C++ Extension Pack），点击齿轮图标，选择 Disable (Workspace) 或直接 Disable，然后按提示重新加载窗口。

在 VS Code 扩展面板搜索 clangd，注意安装时选择 Install in SSH: kvm-ubuntu（远程），而不是仅安装在本地。clangd 插件必须运行在远程侧，才能就近使用虚拟机中的 clangd 二进制和源码索引。

打开远程 `settings.json`（命令面板 → `Preferences: Open Remote Settings (JSON)`），添加：

```json
{
  "clangd.arguments": [
    "--compile-commands-dir=${workspaceFolder}",
    "--background-index",
    "--completion-style=detailed",
    "--header-insertion=never",
    "--log=info"
  ]
}
```

------

# KVM源码一些变量/函数/……

1. **EXIT\_FASTPATH\_NONE**：告诉调用者：“别尝试重进 Guest 了，也不要走快速通道，老老实实走慢速路径去让上层处理这个模拟请求。”（无法走快速通道，必须走慢速路径）

2. **PLE（PAUSE\-Loop Exiting）**：当 Guest 内部执行 `PAUSE` 指令（通常用于自旋锁）时，硬件可以通过配置 PLE 窗口，让 Guest 在空转一段时间后自动退出（VM\-Exit），把 CPU 让给其他线程，防止 Guest 白白浪费物理 CPU 算力。

3. **WARN\_ON\_ONCE\(\)**：检查一个条件，如果成立（说明代码出 Bug 了），就打印一条警告日志和调用栈，但系统不会崩溃，继续往下运行。

4. **\.noinstr\.text** 是 Linux 内核为了防止 Kprobes、ftrace 等追踪工具插桩（instrumentation）而保留的安全区。

> **插桩（Instrumentation）**，简单来说就是：在原本干净的代码中，强行“插”入一些额外的监控代码或指令（就像打入一根探针），用来收集运行数据、追踪错误或分析性能。

5. **`nested_run_pending`** 是一个标志位，表示 L1 的 `VMLAUNCH` 指令正在被 L0 处理中。

6. **KVM\_ISA\_VMX**：表示当前的硬件虚拟化架构是 Intel 的 VMX（Virtual Machine Extensions，即 VT\-x）

> KVM\_ISA：KVM Instruction Set Architecture（KVM 指令集架构）。
> 
> VMX：Intel 的虚拟化扩展技术。
> 

7. `native_write_cr2()`是直接读写硬件的内联汇编，不经过任何虚拟化间接层（paravirt）



# 退出原因

1. `EXIT_REASON_MCE_DURING_VMENTRY`：在进入虚拟机（VM\-Entry）期间发生了机器检查异常（MCE）

> `MCE`（Machine Check Exception）是 CPU 检测到硬件级别的严重错误（如内存 ECC 错误、总线错误）。



# 寄存器

1. **指令指针寄存器（RIP）**，即下一条要执行的代码地址

2. **栈指针寄存器（RSP）**，指向当前程序运行所使用的“栈（Stack）”的顶部，告诉你当前函数的数据写到哪个位置了

3. **CR4** 包含了各种 CPU 功能开关（比如虚拟化开关、浮点支持等）

4. **硬件调试寄存器（DR0\-DR7）**

5. **模型特定寄存器（MSR，Model\-Specific Register）** 是 x86 架构 CPU 内部的一组特殊控制/状态寄存器。它是用来配置和控制 CPU 各种高级特性的“后台控制面板”（eg\. 性能监控（PMU）、时钟与电源管理、系统调用、硬件虚拟化、温度与保护）。

> - **普通寄存器（如 RAX, RBX, RIP）**：给应用程序用的，用来做加减乘除、存指针。
> 
> - **MSR 寄存器**：给操作系统内核或底层软件用的。普通程序无权访问，必须通过特定的特权指令（`rdmsr` 读，`wrmsr` 写）配合一个编号（索引）来访问。
> 

6. **调试控制寄存器（****`DEBUGCTLMSR`****）**，它像一个“开关面板”，用不同的比特位（bit）来独立开启或关闭CPU内置的多种硬件调试与追踪功能。

7. 现代 CPU 有哪几个**段寄存器**？x86 架构提供了 6 个段寄存器：CS（代码段）、DS（数据段）、SS（栈段）、ES/FS/GS（附加数据段）

    现代用途：现代 Linux 等系统都使用“平坦模式（Flat Model）”，段的基址几乎全是 0，相当于隐身了。那为什么还要留着它？ 因为硬件需要它来记录当前的特权级（Ring 0\~3）。比如，`CS` 寄存器里就存着当前代码段是运行在“内核态（Ring 0）”还是“用户态（Ring 3）”

    特殊用途（TLS）：在 64 位下，`FS` 和 `GS` 寄存器被保留下来，专门用来指向“线程局部存储（TLS）”。



# 漏洞

1. **MDS（Microarchitectural Data Sampling）** 是一类漏洞的统称，允许攻击者从 CPU 的填充缓冲区（Fill Buffer）、加载端口（Load Port） 等微架构结构中采样瞬态数据。

2. **L1TF（L1 Terminal Fault） 漏洞**，该漏洞允许 guest 通过推测执行读取 L1 数据缓存中宿主机的残留数据



# 其它技术

## SRCU（Sleepable Read\-Copy Update）

它是一种允许多个读者同时无锁访问数据，并且允许读者在读取期间“睡觉（阻塞）”的同步保护机制。

> “睡觉（阻塞）”：让出CPU
> 

机制：

- 对于读者（Reader）：调用 `srcu_read_lock` 和 `srcu_read_unlock`。在这两个函数之间，你可以随便睡眠。

- 对于写者（Writer）：如果写者要修改或销毁数据，它会等待所有正在“读”的临界区都结束（哪怕那些读者中间睡了一大觉）。

## **LBR（Last Branch Record，最近分支记录）**

LBR 是 PMU （性能监控单元）的一部分，用于记录最近几十条跳转指令的地址，常用于性能瓶颈分析。

## **硬件抢占定时器（Preemption Timer）**

Intel VT\-x 提供的一个硬件倒计时器。时间一到，硬件会强制把 CPU 从 Guest 踢回宿主机（VM\-Exit），防止 Guest 在死循环里霸占 CPU。

> 抢占定时器就是为了配置和启动这个功能的
> 

## APIC**（Local Advanced Programmable Interrupt Controller）**

**本地 APIC** 是 CPU 内部（或紧贴着 CPU）的一个中断控制器模块。每个 CPU 核心都有一个自己的本地 APIC。

**本地 APIC 定时器**，就是这个模块里自带的一个“闹钟”。它的主要作用是给当前这个 CPU 核心发送时钟中断（比如用于操作系统的进程调度、时间片轮转）。

## **eVMCS（Enlightened VMCS，启明版 VMCS）**

微软 Hyper\-V 提供的一种“半虚拟化优化通道”，专门用来加速在 Hyper\-V 之上运行的嵌套虚拟机。

核心：用一个存放在 Guest 物理内存中的数据结构，替代传统上存放在硬件内部、访问极慢的 VMCS。

## FRED（Flexible Return and Event Delivery）

FRED 是 Intel 较新的中断/异常交付机制，用统一的栈帧和事件交付路径替代传统的 IDT \+ `iret` 方式

## PML：Page Modification Log

用于记录被修改的 GPA 页；PML full 表示 PML 缓冲区满导致的 VM\-Exit。

# 阅读QEMU源码方法：

相对于KVM整体方法不变

配置过程：

（1）确保你的 Ubuntu 虚拟机中已经安装了必要的工具。如果之前配置 KVM 环境时已安装，可跳过此步。

```bash
sudo apt update
sudo apt install -y clangd bear build-essential pkg-config libglib2.0-dev libpixman-1-dev ninja-build
```

> `clangd` 是语言服务器，`bear` 用于跟踪编译命令，其余是编译 QEMU 的基础依赖。

（2）通过 QEMU 的 `configure` 脚本

QEMU 的 `configure` 脚本在配置构建系统时，可以直接生成编译数据库。

**进入源码根目录**：

```bash
cd /path/to/qemu-6.2+dfsg
```

**创建独立的构建目录**（推荐，保持源码整洁）：

```bash
mkdir build && cd build
```

**运行 `configure`**：使用 `--target-list` 指定你关心的架构，这能显著减少索引范围，加快 clangd 的分析速度。例如，只关注 x86_64 系统虚拟化：

```bash
../configure --target-list=x86_64-softmmu
```

> 如果你想索引所有架构，可以省略 `--target-list` 参数，但后续 clangd 的索引时间会非常长。

执行后，`compile_commands.json` 文件应该已经生成在 `build/` 目录下了。你可以用 `ls compile_commands.json` 确认一下。

（3）配置 VS Code 与 clangd

**打开 QEMU 源码文件夹**：在 VS Code 中，通过 **File → Open Folder** 选择 QEMU 源码的**根目录**（即 `qemu-6.2+dfsg` 文件夹）作为工作区。

**禁用 Microsoft C/C++ 插件**：与配置 KVM 时一样，在远程扩展中，找到 **C/C++** 插件，选择 **Disable (Workspace)**，避免与 clangd 冲突。

**创建 clangd 配置文件**：在 QEMU 源码**根目录**下，新建一个名为 `.clangd` 的文件，填入以下内容：

```yaml
CompileFlags:
  CompilationDatabase: "build" # 指向 compile_commands.json 所在的目录
  Add: [-w, -ferror-limit=0] # 忽略所有警告，防止大量波浪线干扰阅读
Diagnostics:
  Suppress: ["*"] # 抑制所有诊断信息
ClangTidy:
  Remove: ["*"] # 关闭 Clang-Tidy 检查
Index:
  Background: Build # 后台构建索引
```
