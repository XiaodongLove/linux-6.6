# AGENTS.md — Linux 6.6.0 内核源码阅读工程

## 工程性质

- 这是 Linux 内核 **6.6.0** 的原始源码树（`Makefile` 中 `VERSION=6 PATCHLEVEL=6 SUBLEVEL=0`），目录名 `linux6.6_read` 表明主要用途是**阅读与分析源码**，而不是日常开发或向上游提交补丁。
- 本地 git 仓库**仅从 6.6.0 快照开始**（单个初始提交），`git log/blame` 看不到上游历史。需要追溯某段代码的演变时，引用上游 `https://git.kernel.org/` 或 `https://elixir.bootlin.com/linux/v6.6/source/`。本地提交主要用于记录阅读笔记和个人改动。
- 根目录有一个约 1.1 GB 的 `tags` 文件（ctags 生成），不要读取它，用 `rg` / Grep 直接搜索源码即可。
- 回答源码问题时优先给出 **文件路径 + 函数名/行号** 的引用，并沿调用链解释，而不是只给结论。

## 目录布局

| 目录 | 内容 |
|---|---|
| `init/` | 内核启动入口：`start_kernel()` 在 `init/main.c` |
| `kernel/` | 核心子系统：调度 `kernel/sched/`、中断 `kernel/irq/`、时间 `kernel/time/`、锁 `kernel/locking/`、RCU `kernel/rcu/`、cgroup `kernel/cgroup/`、BPF `kernel/bpf/`、`fork.c`、`exit.c`、`signal.c`、`workqueue.c` |
| `mm/` | 内存管理：伙伴系统 `page_alloc.c`、slab `slub.c`、缺页 `memory.c`、VMA `mmap.c`、回收 `vmscan.c`、memcg `memcontrol.c`、folio/compaction/KSM 等 |
| `fs/` | VFS 层（`namei.c`、`dcache.c`、`inode.c`、`open.c`、`read_write.c`）及各文件系统子目录（`ext4/`、`xfs/`、`btrfs/`、`proc/`、`sysfs/`、`overlayfs/` 等） |
| `net/` | 网络栈：`core/`、`ipv4/`、`ipv6/`、`socket.c`、`netfilter/`、`sched/`（流控） |
| `drivers/` | 设备驱动，按总线/类别分子目录；驱动模型核心在 `drivers/base/` |
| `block/` | 块层、blk-mq、I/O 调度器 |
| `io_uring/` | io_uring 实现 |
| `ipc/` | System V / POSIX IPC |
| `security/` | LSM 框架及 SELinux、AppArmor 等 |
| `crypto/` | 加密 API |
| `lib/` | 通用库函数（链表、位图、`kobject`、`xarray`、`maple_tree` 等） |
| `include/linux/` | 内核公共头文件（结构体定义多数在这里） |
| `include/uapi/` | 用户态 ABI 头文件 |
| `include/asm-generic/` | 架构通用的 asm 默认实现 |
| `arch/<arch>/` | 架构相关代码；本工程以 `arch/arm64/` 为主，`arch/x86/` 作对照，其他架构可忽略 |
| `virt/kvm/` | KVM 架构无关部分 |
| `rust/` | Rust 支持（6.6 仍处于早期阶段） |
| `tools/` | 用户态工具：`perf/`、`bpf/`、`objtool/`、`testing/selftests/` |
| `scripts/` | 构建脚本与辅助工具（见下文） |
| `samples/` | 示例代码 |
| `Documentation/` | 内核文档（RST 格式） |

## 构建系统（Kbuild）

- 顶层 `Makefile` + `Kconfig` + 各目录 `Makefile`/`Kconfig`；Kbuild 规则在 `scripts/Makefile.build`、`scripts/Kbuild.include`。
- `obj-y` 编进内核，`obj-m` 编成模块，`obj-$(CONFIG_FOO)` 由配置决定。追踪某文件是否参与编译，看其所在目录 `Makefile` 中对应的 `CONFIG_*` 开关。
- 常用命令（需要 Linux 环境和交叉工具链；本机为 macOS，通常**只读代码不编译**）：
  - `make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig`（arm64 只有一个 `defconfig`，位于 `arch/arm64/configs/`）
  - `make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc) Image modules dtbs`
  - `make ARCH=arm64 menuconfig`
  - 对照 x86 时：`make x86_64_defconfig && make -j$(nproc)`
  - `make compile_commands.json`（生成 clangd 索引）
  - `make tags` / `make cscope`
- 配置项说明在对应 `Kconfig` 文件中，可用 `rg 'config FOO' --glob 'Kconfig*'` 定位。

## 编码规范

- 内核编码风格：`Documentation/process/coding-style.rst`；格式化配置 `.clang-format`，Rust 用 `.rustfmt.toml`。
- 要点：Tab 缩进（8 列宽）、函数左花括号另起一行、`struct` 不 typedef、优先使用 `list_head`/`rb_root`/`xarray` 等内核容器、错误用负 errno 返回。
- 补丁检查：`scripts/checkpatch.pl -f <file>`；语义补丁：`scripts/coccinelle/`、`make coccicheck`。
- 新文件首行须有 SPDX 许可证标识（`// SPDX-License-Identifier: GPL-2.0`）。
- 文档注释用 kernel-doc 格式（`/** ... */`），见 `Documentation/doc-guide/kernel-doc.rst`。

## 阅读源码的常用入口与技巧

- **系统调用**：定义用 `SYSCALL_DEFINEn(name, ...)` 宏，搜索 `rg 'SYSCALL_DEFINE[0-6]\(name'`；arm64 没有 `.tbl` 文件，系统调用号来自 `include/uapi/asm-generic/unistd.h`，调用表在 `arch/arm64/kernel/sys.c`（`sys_call_table`），入口汇编在 `arch/arm64/kernel/entry.S`，分发在 `arch/arm64/kernel/syscall.c`；x86 对照为 `arch/x86/entry/syscalls/syscall_64.tbl`。
- **arm64 关键路径**：启动 `arch/arm64/kernel/head.S` → `__primary_switched` → `start_kernel()`；异常向量 `arch/arm64/kernel/entry.S`、`entry-common.c`；页表/MMU `arch/arm64/mm/`（`mmu.c`、`fault.c`）；上下文切换 `arch/arm64/kernel/process.c` 的 `cpu_switch_to`（`entry.S`）；SMP 启动 `arch/arm64/kernel/smp.c`；内存布局文档 `Documentation/arch/arm64/memory.rst`。
- **初始化顺序**：`start_kernel()` → `rest_init()` → `kernel_init()`；`*_initcall()` 宏的等级定义在 `include/linux/init.h`。
- **查找维护者/子系统归属**：`MAINTAINERS` 文件，或 `scripts/get_maintainer.pl -f <path>`。
- **宏展开/间接调用**：`EXPORT_SYMBOL`、函数指针 ops 结构体（`file_operations`、`inode_operations`、`net_device_ops`、`sched_class`）是常见的间接层，追踪时先定位 ops 实例。
- **架构相关符号**：`include/asm-generic/` 给默认实现，`arch/<arch>/include/asm/` 覆盖。
- **调试工具文档**：`Documentation/dev-tools/`（KASAN、KCSAN、kmemleak、kgdb、ftrace 等）；`scripts/gdb/` 提供 GDB 内核调试脚本；`scripts/faddr2line`、`scripts/decode_stacktrace.sh` 解析崩溃栈。
- **文档索引**：`Documentation/core-api/`、`Documentation/mm/`、`Documentation/scheduler/`、`Documentation/filesystems/`、`Documentation/networking/`。

## 对 Agent 的约定

- 默认架构上下文为 **arm64**：涉及 `arch/` 代码、`asm/` 头文件、内存布局、异常/中断入口时一律以 `arch/arm64/` 为准；需要对照时再提 x86_64，不要主动展开其他架构。
- 这是源码阅读工程：除非用户明确要求，**不要修改内核源码**；可以在用户要求下新增笔记文件（建议放在根目录 `notes/` 之类的独立目录，避免与源码混淆）。
- 解释代码时注意区分**编译期配置**（`#ifdef CONFIG_*`）导致的不同路径，并指出对应的 Kconfig 选项。
- 大文件（如 `kernel/sched/fair.c`、`mm/page_alloc.c`、`fs/namei.c`）先用 Grep 定位，再按行范围读取，不要整文件读取。
- 不要读取根目录 `tags` 文件。
