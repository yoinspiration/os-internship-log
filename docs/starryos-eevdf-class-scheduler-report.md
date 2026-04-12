# StarryOS：EEVDF 调度与 Linux 风格优先级接口

**分支：** `feat/eevdf-class-scheduler`（历史名称；实现上已**不再使用**「EEVDF-Class」多类两层框架，而以 **EEVDF 单队列调度器**及可选 **per-CPU 异构调度**为主。）

**源码：** [https://github.com/yoinspiration/StarryOS/tree/feat/eevdf-class-scheduler](https://github.com/yoinspiration/StarryOS/tree/feat/eevdf-class-scheduler)

> **归档说明：** 与 StarryOS 仓库中 `docs/report.md` 内容一致；修改请以 StarryOS 侧为权威，本文件便于在实习日志仓库中查阅与提交。

## 目标

在 StarryOS 上实现 **EEVDF 风格**的抢占式调度（基于 vruntime、eligible 与 deadline），将 `nice` 与 `getpriority` / `setpriority` 接到内核任务层，明确权限与作用域，并具备可观测性与回归测试能力；可选地支持 **per-CPU** 为不同 CPU 选择 EEVDF、FIFO 或 RR（编译期 `CPU_SCHED` 配置）。

## 主要工作

### 调度框架与算法

- 在 `axsched` 中实现 **`EevdfScheduler` / `EevdfEntity`**：单就绪队列语义下，按 eligible 与最小 deadline 选任务，配合时间片与抢占；`nice` 通过 Linux 风格权重映射参与公平性与截止期计算。
- 可选 **`sched-per-cpu`**：每 CPU 一个 `PerCpuScheduler`，在 **EEVDF、FIFO、RR** 间按配置选择；调度元数据放在调度器侧表项中，任务结构可与算法解耦，便于跨调度器迁移同一 `Arc<Task>`。
- 提供调度统计（选中次数、抢占、时间片耗尽、fallback 等），并支持 **启动期统计演示**（`eevdf-stats-demo`）等可观测能力。

### 与 Linux 用户态接口对齐

- 实现或完善 `getpriority` / `setpriority`：
  - **`PRIO_PROCESS`**：支持当前进程及约定的目标语义。
  - **`PRIO_PGRP` / `PRIO_USER`** 等作用域按策略返回无权限或错误，与文档声明的边界一致。
- `getpriority` 返回值与常见 Linux 编码一致（nice 与返回值的映射）。
- **权限模型**：区分 **uid / euid**；支持自己调自己、特权（euid 为 0）、与目标同 uid 等规则，贴近 Linux 的分阶段实现思路。

### 工程与质量保障

- 更新根目录与 `kernel` 的 Cargo 依赖，完成与现有内核的集成。
- 编写 EEVDF / nice 相关文档与回归脚本（基准、权限、`syscall` 冒烟），便于重复验证。

**主要代码路径（便于对照）：** `crates/axsched/src/eevdf.rs`、`crates/axsched/src/per_cpu.rs`、`crates/axtask/`、`kernel/src/syscall/task/schedule.rs`。

### 调度器选择与 RR、EEVDF、CFS 等对比

StarryOS 通过 **`axfeat` / `axtask` 的编译期特性**选定调度器（见 `crates/axtask/src/api.rs`）。**优先级顺序**为：`sched-per-cpu` → `sched-eevdf` → `sched-rr` → `sched-cfs`；若均未启用，则回退为 **FIFO**（协作式，无抢占时间片）。

**本仓库当前默认（`kernel/Cargo.toml`）：** 为 `axfeat` 打开 **`sched-per-cpu`**，因此默认构建走 **per-CPU 调度包装**（具体每 CPU 算法由构建时 `CPU_SCHED` 等配置决定），**不再**默认使用已移除的「EEVDF-Class」两层类调度。若需与纯 **RR** 或纯 **`sched-eevdf`**（单队列 EEVDF、无 per-CPU 包装）对照实验，需在构建中调整特性组合，避免与当前默认混淆。

| 维度 | RR（Round-Robin） | EEVDF（`sched-eevdf`，`EevdfScheduler`） |
| --- | --- | --- |
| 队列结构 | **单条**就绪队列，全体任务按时间片轮转 | **单条**逻辑就绪结构；在 **eligible** 任务中选 **deadline** 最小者（辅以快路径与 fallback） |
| 时间片 | 每任务递减时间片，耗尽则切换 | 时间片与 vruntime、deadline 更新联动；可被更早 deadline 的 eligible 任务抢占 |
| 公平性含义 | 任务之间**近似均分** CPU（同权时） | **加权**公平（nice → 权重），并强调延迟相关语义（相对 CFS 的改进点之一） |
| 与 `nice` | 无内置多档语义（除非另行扩展） | **nice → 权重**，与 `getpriority` / `setpriority` 联动 |
| 典型用途 | 教学与基线抢占调度 | 单队列下对接 Linux 风格优先级与 EEVDF 启发式选路 |

**关系说明：** 当前实现**没有**「先选调度类、再在类内 RR」的 **EEVDF-Class 第二层**；EEVDF 即为 **同一就绪集合上**的 eligible / deadline 规则。**`sched-per-cpu`** 则是在 SMP 上为**每个 CPU** 独立挂载 EEVDF、FIFO 或 RR 之一（见 `PerCpuScheduler`、`SchedulerKind`），与「多类队列」不是同一概念。

## 参考与致谢

- 调度框架以 **ArceOS** 生态的 [axsched](https://github.com/arceos-org/axsched) 为基础；本仓库在 `crates/axsched` 中扩展 **EEVDF** 与 **per-CPU** 实现。
- **Linux EEVDF/CFS** 及官方或教材中的相关说明，作为调度语义与设计参照。

## 局限与后续

- **`PRIO_PGRP`、`PRIO_USER`** 等未按 Linux 全功能实现，属明确裁剪；跨进程 `setpriority` 语义以当前分支实现为准。
- 与真实 Linux **EEVDF** 在 SMP、负载跟踪、细节参数上仍可能有差异，本阶段侧重**教学/原型级**集成与接口语义。
- 后续可扩展：多核行为、更完整的跨进程优先级策略、与性能实验数据的结合。

## 摘要

本工作在 StarryOS 上集成 **EEVDF 单队列调度**与 **Linux 风格 `nice` / `getpriority` / `setpriority`**，明确 `PRIO_PROCESS` 等作用域与 uid/euid 规则；可选 **per-CPU** 异构调度（EEVDF / FIFO / RR），并补充统计与回归脚本。**已不再采用**历史上的「EEVDF-Class」多类两层调度描述，相关叙述以本文与 StarryOS 源码为准。
