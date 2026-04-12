# 多组织共享硬件 Runner：个人贡献与仓库现状说明

本文档用于**成果整理**，说明作者（yoinspiration）在 [arceos-hypervisor/github-runners](https://github.com/arceos-hypervisor/github-runners) 上提交的系列 PR，以及**当前 `main` 分支**的实现形态。技术细节以 Git 历史为准。

---

## 1. 项目背景（贡献发生时的问题）

在同一台物理机上，多个 GitHub 组织各自部署 **self-hosted Actions Runner** 时，若 Runner 需要访问**同一套独占硬件**（串口、上电控制、开发板等），会出现：

- 多作业并行访问硬件导致冲突；
- 工作流 **Cancel** 后，基于文件锁的串行机制若释放不当，可能造成**误释放**或**并行竞态**。

当时的目标是在**不改变「每组织独立 Runner 容器」**的前提下，用 **Job 级串行 + 可审计的锁语义**，让多组织安全共享硬件。

---

## 2. 个人 PR 系列概览

| PR | 合并时间（约） | 主题 |
|----|----------------|------|
| [#2](https://github.com/arceos-hypervisor/github-runners/pull/2) | 2026-02-10 | 引入 `runner-wrapper`：Pre/Post Job 脚本 + `flock` 文件锁，Job 级串行 |
| [#3](https://github.com/arceos-hypervisor/github-runners/pull/3) | 2026-02-11 | `runner.sh` 集成 wrapper；`.env.example` 与 README 说明 |
| [#4](https://github.com/arceos-hypervisor/github-runners/pull/4) | 2026-02-24 | 板子锁 ID 默认值（per-board）；容器名/Compose 默认拼入 `ORG`/`REPO`；`verify-changes.sh` |
| [#11](https://github.com/arceos-hypervisor/github-runners/pull/11) | 2026-02-26 | Cancel 场景锁释放加固（runner / run-id / attempt 校验、独立 release 语义）；文档与 compose 侧修复 |
| [#13](https://github.com/arceos-hypervisor/github-runners/pull/13) | 2026-03-27 | Board lock watcher：监控并配合释放；单进程多板、与 `runner.sh`/Compose 生命周期集成；依赖与文档 |

---

## 3. 各 PR 的技术要点（贡献内容）

### PR #2 — `runner-wrapper` 与 Job 级锁

- 新增 `runner-wrapper/`：`runner-wrapper.sh`、`pre-job-lock.sh`、`post-job-lock.sh`。
- 使用 **`flock` 文件锁**在**单个 Job** 粒度串行化对共享资源的访问；支持多 Runner 空闲、有 Job 时排队执行。
- `Dockerfile` 中为脚本赋予可执行权限，便于镜像内挂载或内嵌使用。

### PR #3 — 与 `runner.sh` 集成及配置面

- 将 wrapper 接入主脚本路径，完善 **`.env.example`** 与 **README / README_CN**，使运维可按文档启用多组织共享硬件方案。

### PR #4 — 命名与锁 ID 规则

- **板子锁 ID**：优先使用显式配置，否则回退到 **per-board 默认值**（如 `board-phytiumpi`、`board-roc-rk3568-pc`），避免错误混用全局 `RUNNER_RESOURCE_ID`。
- **容器名 / Compose 文件**：未设置前缀时自动拼入 **`ORG` / `REPO`**，降低多组织、多副本同机部署时的命名冲突。
- 提供 **`verify-changes.sh`** 做关键逻辑回归检查。

### PR #11 — Cancel 与并行安全

- 针对 **用户取消工作流** 等场景，加固 **锁释放条件**（与 runner、run id、attempt 等上下文一致），避免 cancel 后错误释放导致**下一 Job 与仍存活进程并行**。
- 同步处理 **Compose 代理注入**、wrapper **热更新生效**等问题，并补充多组织共享 Runner 的使用说明类文档。

### PR #13 — Board lock watcher 与运维闭环

- 引入 **lock-watcher**：在 cancel 等异常路径下辅助保证**板级文件锁**可被安全释放，并与多组织场景对齐。
- 演进为**单进程监控多块板**、与 **Compose** 的 start/stop 联动；补充 **libclang/bindgen**、锁目录校验、权限类排障文档及多组织部署指南等。

---

## 4. 后续仓库演进：为何当前 `main` 不再包含上述实现

2026-04 起，维护者引入多笔 **Revert**（针对 #2、#3、#4、#11 等相关提交），随后通过 **[PR #18](https://github.com/arceos-hypervisor/github-runners/pull/18)**（说明含 *revert commit about flock*）**移除了** `runner-wrapper/` 目录、watcher 及相关文档，整体方向改为**不再采用基于 flock 的 wrapper/watcher 栈**。

因此：

- 上述 PR **仍保留在 Git 历史中**，可完整溯源、对比与复现；
- **当前 `main` 上的可运行代码**已**不依赖**这些 PR 中的脚本与文档路径；若写「当前生产部署方案」，应以**当前分支 README 与 `runner.sh`** 为准。

这不是否定 PR 的设计与实现价值，而是仓库在**架构选型上的一次切换**（从「wrapper + 文件锁 + watcher」转为后续由维护者主导的脚本与配置模型）。

### 4.1 为何会做这次切换（依据与推断）

**仓库内可直接核对的事实：**

- [PR #18](https://github.com/arceos-hypervisor/github-runners/pull/18) 合并说明为 **`revert commit about flock`**，即维护者明确要**移除基于 `flock` 的那套实现**，而非在原架构上小幅修补。
- 更早的多组织相关提交中，已有 **「移除通用 runner 资源锁逻辑，仅保留板卡专用锁」** 一类重构（见 `git log` 中 `877910a` 等），说明路线在**收束锁的适用范围**：从「通用资源锁」收束到「板卡相关」，与后来**整体删掉 wrapper 栈**方向一致。

**基于常见工程实践的合理推断（非官方长文，若需原话请查 PR #18 的 Review/讨论）：**

- 旧方案组件多（wrapper、pre/post、watcher、锁目录与权限），**运维与排障路径更长**，与官方 `run.sh` 行为耦合更深。
- 上游若优先 **降低脚本栈复杂度**、把并发控制更多交给 **Runner 实例划分 + `runs-on` 标签路由**，往往会倾向拿掉跨 Job 的全局文件锁方案。

### 4.2 当前实现相对旧方案的侧重点（优越性 / 取舍）

以下是对 **当前 `main`**（去掉 wrapper 后、以 `runner.sh` 动态生成 Compose、`BOARD_RUNNERS` 等为主，如 `93712c3`「板子 runner 配置泛化为环境变量驱动的动态生成」）的归纳：

| 侧重点 | 说明 |
|--------|------|
| **栈更薄** | 容器内仍以官方 **`/home/runner/run.sh`** 为入口，不再包一层 Job 级锁脚本；无长期驻留的 lock-watcher，**故障面更少**。 |
| **隔离方式** | 通过 **`RUNNER_NAME_PREFIX`、按 ORG/REPO 的 Compose/缓存文件** 等区分多组织；板卡 workload 通过 **`BOARD_RUNNERS` 与标签** 路由，**依赖「实例 + 标签」策略** 而非「全局文件锁串行」。 |
| **配置维护** | **环境变量驱动**生成 Compose，减少独立脚本目录与文档分叉，后续易与 **KVM GID** 等能力在同一脚本内扩展。 |

**取舍说明：** 新方案**不自动等价于**「硬件层面绝对互斥」——若标签与 Runner 数量配置不当，仍可能出现多 Job 争用；旧方案用 **`flock` 强串行**换更强约束。评审时应表述为：**上游在正确性与复杂度之间选择了更薄的部署模型**，而非简单判定新旧孰优。

---

## 5. 当前 `main` 实现要点（与本人 PR 的关系）

当前公开文档（如 `README_CN.md`）描述的能力主要包括：

- 使用 **Docker Compose** 管理多个 **官方/自定义 `actions-runner` 镜像**容器；
- **组织级 / 仓库级** Runner（`REPO`、`ORG` 等）；
- **`BOARD_RUNNERS`**：为开发板类实例配置独立标签与设备映射；
- **容器命名前缀**、**Compose 文件名**、**注册令牌缓存文件**等按 `ORG`/`REPO` 区分，避免同机多组织冲突；
- 自定义 **`Dockerfile`**（如 QEMU、交叉工具链等）与镜像重建检测。

这些能力中，**多组织隔离、命名与文件按组织区分**等与 PR #4 及后续 multi 系列演进**在目标上一脉相承**；但**不再包含** PR #2/#3/#11/#13 中的 **wrapper、flock Job 锁、lock-watcher** 等具体组件。

---

## 6. 成果表述建议（简历 / 答辩 / 材料）

可将本系列工作概括为：

1. **问题建模**：多组织 self-hosted Runner 共享独占硬件时的**串行与取消语义**问题。  
2. **方案设计**：Job 级 `flock`、上下文相关的释放校验、独立 watcher 与 Compose 生命周期配合。  
3. **工程落地**：Bash 脚本、`Dockerfile`、文档、验证脚本及多轮 Code Review 反馈（如 per-board 锁 ID、命名规则）。  
4. **客观现状**：后续上游选择移除该栈；贡献保留在 **Git 历史**中，可随仓库永久引用。答辩时可结合上文 **§4.1–§4.2** 说明「为何切换」与「新方案取舍」，避免评审只记住「被 revert」。

如需向评审展示代码，可直接给出 **合并提交的父级范围** 或 PR 链接，并说明「当前主干已改版，本段为历史贡献」。

---

## 7. 参考链接

- 仓库：<https://github.com/arceos-hypervisor/github-runners>  
- 相关讨论（PR #2 引用）：组织内 discussion *#341*（见提交 `83149ff` 说明）

---

*文档根据本地 `git log` 与 README 整理；若上游分支有更新，请以远程 `main` 为准。*
