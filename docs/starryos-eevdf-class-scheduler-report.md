# StarryOS：EEVDF 调度类集成与 Linux 风格优先级接口

**分支：** `feat/eevdf-class-scheduler`

**源码：** [https://github.com/yoinspiration/StarryOS/tree/feat/eevdf-class-scheduler](https://github.com/yoinspiration/StarryOS/tree/feat/eevdf-class-scheduler)

> **归档说明：** 与 StarryOS 仓库中 `docs/report.md` 内容一致；修改请以 StarryOS 侧为权威，本文件便于在实习日志仓库中查阅与提交。

## 目标

在 StarryOS 上实现按调度类划分的 EEVDF 风格调度器，将 `nice` 与 `getpriority` / `setpriority` 接到内核任务层，明确权限与作用域，并具备可观测性与回归测试能力。

## 主要工作

### 调度框架与算法

- 引入 `axsched`、`axtask` 等模块，在运行队列层实现 **EEVDF 类调度**：任务划分为交互 / 普通 / 后台等类（与 **nice** 区间对应），类间按权重与时间片策略调度，类内维护时间片与就绪队列。
- 提供调度统计（各类被选中次数、计费 tick 等），并支持**窗口化统计**，便于分析长期行为。

### 与 Linux 用户态接口对齐

- 实现或完善 `getpriority` / `setpriority`：
  - **`PRIO_PROCESS`**：支持当前进程及约定的目标语义。
  - **`PRIO_PGRP` / `PRIO_USER`** 等作用域按策略返回无权限或错误，与文档声明的边界一致。
- `getpriority` 返回值与常见 Linux 编码一致（nice 与返回值的映射）。
- **权限模型**：区分 **uid / euid**；支持自己调自己、特权（euid 为 0）、与目标同 uid 等规则，贴近 Linux 的分阶段实现思路。

### 工程与质量保障

- 更新根目录与 `kernel` 的 Cargo 依赖，完成与现有内核的集成。
- 编写 EEVDF / nice 相关文档与回归脚本（基准、权限、`syscall` 冒烟），便于重复验证。

**主要代码路径（便于对照）：** `crates/axsched/`（含 `eevdf_class.rs`）、`crates/axtask/`、`kernel/src/syscall/task/schedule.rs`。

### 调度器选择与 RR、EEVDF-Class 对比

StarryOS 通过 **`axfeat` 的编译期特性**在 `axtask` 中选定调度器（见 `crates/axtask/src/api.rs`）：`sched-rr` 为时间片轮转，`sched-eevdf-class` 为本文的 EEVDF 类调度，`sched-cfs` 为 CFS；若三者均未启用，则回退为 **FIFO**（协作式，无抢占时间片）。

**本分支默认配置：** `kernel/Cargo.toml` 中为 `axfeat` 打开了 **`sched-eevdf-class`**，因此**按默认构建的内核**使用的是 **EEVDF-Class**，不是 RR。若要与 RR 行为对照实验，需在构建中改为启用 **`sched-rr`**（并去掉 `sched-eevdf-class`），否则会与当前分支默认不一致。

| 维度 | RR（Round-Robin） | EEVDF-Class（本仓库 `eevdf_class.rs`） |
| --- | --- | --- |
| 队列结构 | **单条**就绪队列，全体任务轮转 | **两层**：先按「调度类」选队列，再在类内轮转 |
| 时间片 | 每任务递减时间片，耗尽则切换 | **类内**与 RR 相同的时间片轮转；**类间**按虚拟运行时间与权重决定下一类 |
| 公平性含义 | 任务之间**近似均分** CPU（不考虑业务优先级） | **类间**按权重与 EEVDF 风格截止时间分配比例；**类内**仍公平轮转 |
| 与 `nice` | 无内置多档语义（除非另行扩展） | **nice 映射到类**（交互 / 普通 / 后台），并与 `getpriority` / `setpriority` 联动 |
| 实现与调试 | 结构简单、易验证 | 需理解类、权重、`vruntime` / deadline 等状态，但可统计各类 pick 与 tick |
| 典型用途 | 教学与基线抢占调度 | 在单套 RR 之上区分交互与后台、对接 Linux 风格优先级 |

**关系说明：** EEVDF-Class 的**第二层**在代码注释中明确为「各类内部与 `RRScheduler` 相同的按时间片轮转」；因此可把它理解为「在全局 RR 之前增加了一层**带权重的类选择**」，而不是替代 RR 的另一种一维队列算法。

## 参考与致谢

- 调度框架以 **ArceOS** 生态的 [axsched](https://github.com/arceos-org/axsched) 为基础；EEVDF 类调度逻辑在本仓库 `eevdf_class.rs` 中实现。
- 类间公平与 **EEVDF-inspired** 的两层调度思路，参考了社区同类设计说明，例如 clockworker 对「多队列 + 权重 + EEVDF 启发」的描述；本实现面向**内核任务调度**，类内策略为时间片/轮转，与 clockworker 在 async 执行器中队列内 **FIFO** 的细节并不相同。
- **Linux EEVDF/CFS** 及官方或教材中的相关说明，作为调度语义与设计参照。

## 局限与后续

- **`PRIO_PGRP`、`PRIO_USER`** 等未按 Linux 全功能实现，属明确裁剪；跨进程 `setpriority` 语义以当前分支实现为准。
- 与真实 Linux **EEVDF** 在 SMP、负载跟踪、细节参数上仍可能有差异，本阶段侧重**教学/原型级**集成与接口语义。
- 后续可扩展：多核行为、更完整的跨进程优先级策略、与性能实验数据的结合。

## 摘要

本工作在 `feat/eevdf-class-scheduler` 分支上完成基于 EEVDF 调度类的 CPU 调度集成，并通过 `axsched` / `axtask` 实现与 nice 相关的多类调度及类间加权；内核侧对接 `getpriority` / `setpriority`，明确 `PRIO_PROCESS` 等作用域与 uid/euid 规则，并补充统计与回归脚本，提升内核对 Linux 优先级语义与类调度策略的覆盖，为后续性能与公平性实验提供基础。
