# 实习三个月技术总结报告

## 自动测试系统

### 1. AxVisor QEMU CI 稳定性改进

- 链接：[PR #363](https://github.com/arceos-hypervisor/axvisor/pull/363)
- 合并状态：已合并（Merged）
- 贡献规模：26 commits，18 个文件变更，`+678 / -19`

#### 背景问题

在 AxVisor 的 QEMU 自动化测试中，存在以下痛点：
- 不同 Guest（ArceOS / Linux / NimbOS）环境准备流程分散，维护成本高；
- NimbOS 场景依赖交互式输入，CI 中容易出现“测试通过但任务失败”的误报；
- 失败信号不够明确，panic 等异常无法被尽早识别。

#### 关键工作

1. 统一环境准备入口
- 新增 `scripts/setup_qemu.sh`，统一支持 `arceos` / `linux` / `nimbos` 三类 Guest；
- 自动执行镜像下载、配置 patch、rootfs 准备，减少手工步骤和路径错误。

2. 补齐文档与上手路径
- 新增中英文 QEMU 快速上手文档，沉淀从依赖安装到三类 Guest 运行的完整流程；
- 在 README 中增加快速入口，降低新成员接入成本。

3. 增强 NimbOS 测试可自动化能力
- 新增 `scripts/ci_run_qemu_nimbos.py`，通过 PTY 方式启动子进程，保障 CI 环境下输入可正确透传；
- 识别 shell 提示后自动触发 `usertests`，并在命中 `usertests passed!` 时返回正确退出码。

4. 提升 CI 失败可观测性与可诊断性
- 在 QEMU 配置中补充 `fail_regex`（如 `panicked at`），尽早暴露 guest panic；
- 对 NimbOS 启动依赖（`axvm-bios.bin`）进行前置校验，避免隐式失败。

#### 结果与价值

- 测试稳定性：修复了 NimbOS 在 CI 中的交互与退出码问题，显著降低误报失败；
- 工程效率：统一 setup 脚本后，减少重复脚本与人工排障成本；
- 可维护性：流程与文档标准化后，跨场景测试可复用性更高；
- 团队协作：将经验固化为脚本与文档，便于后续同学复用与扩展。

### 2. 多组织 GitHub Runner 硬件锁机制建设（github-runners）

- PR 汇总入口：[GitHub PR 列表](https://github.com/arceos-hypervisor/github-runners/pulls?q=is%3Apr+author%3Ayoinspiration+is%3Aclosed)
- 合并成果：5 个 PR 已合并（[#2](https://github.com/arceos-hypervisor/github-runners/pull/2)、[#3](https://github.com/arceos-hypervisor/github-runners/pull/3)、[#4](https://github.com/arceos-hypervisor/github-runners/pull/4)、[#11](https://github.com/arceos-hypervisor/github-runners/pull/11)、[#13](https://github.com/arceos-hypervisor/github-runners/pull/13)）

#### 背景问题

在多组织共享开发板资源时，Runner 缺乏统一锁机制，容易出现并发抢占、取消后资源未释放、锁粒度不一致等问题，导致 CI 任务互相干扰、排队时间增加、故障定位困难。

#### 实现原理（文件锁）

- 方案基于 Linux 文件锁（`flock`）实现互斥：将每块开发板抽象为一个 lock file，同一时刻仅允许一个 Runner 持有该文件的独占锁；
- 锁粒度为 per-board：同一块板上的任务互斥，不同开发板对应不同锁文件，可被不同 Runner 并行使用；
- 任务启动时先尝试获取独占锁，获取成功后进入执行阶段，失败则等待或退出，避免多任务并发抢占同一硬件；
- 在正常结束、失败或 Cancel 路径统一执行解锁与清理，降低异常中断后残留“僵尸锁”的概率。

#### 关键工作（按迭代演进）

1. 建立锁包装能力（PR [#2](https://github.com/arceos-hypervisor/github-runners/pull/2)、[#3](https://github.com/arceos-hypervisor/github-runners/pull/3)）
- 为多组织共享硬件引入 `runner-wrapper` 锁包装脚本；
- 将锁能力集成进 `runner.sh` 工作流，并补齐使用文档，形成可落地的基础方案。

2. 标准化锁标识与隔离策略（PR [#4](https://github.com/arceos-hypervisor/github-runners/pull/4)）
- 将板子锁 ID 收敛到 per-board 默认策略；
- 将容器命名自动拼入 org/repo 维度，降低跨组织任务冲突概率。

3. 修复并发竞态与取消场景（PR [#11](https://github.com/arceos-hypervisor/github-runners/pull/11)、[#13](https://github.com/arceos-hypervisor/github-runners/pull/13)）
- 加固多组织 Runner 锁机制，修复 cancel 场景下的并行竞态问题；
- 将 cancel watcher 与 `docker compose` 生命周期集成，随 Runner 一起启动/回收，对使用者基本无感；
- 支持在 Cancel 路径自动释放开发板锁，减少“僵尸锁”导致的资源阻塞；
- 持续补充文档，降低维护门槛并提升团队可复用性。

#### 结果与价值

- 资源利用率：降低开发板被异常占用的概率，提升共享硬件可用性；
- 流程鲁棒性：在取消、失败等非理想路径下也能保证锁释放；
- 并发安全性：减少跨组织并行任务互相抢占与串扰；
- 可运维性：锁策略、命名规范和文档沉淀后，问题定位更快、迁移成本更低。

### 3. axci 规则驱动自动目标选择与测试链路重构

- 链接：[PR #9](https://github.com/arceos-hypervisor/axci/pull/9)
- 当前状态：Open（待合并）
- 变更规模：39 commits，21 个文件变更，`+4071 / -397`

#### 背景问题

在 CI 全量测试模式下，存在“变更范围小但测试范围大”的问题，导致执行耗时长、资源利用率偏低；同时，测试脚本长期演进后出现结构耦合，扩展与维护成本上升。

#### 实现原理（依赖感知自动选目标）

- 基于 `git diff` 获取变更文件，并结合 `cargo metadata` 将变更路径映射到对应 workspace crate；
- 从直接变更 crate 出发，在反向依赖图上做 BFS 扩散，得到受影响 crate 集合（`affected_crates`）；
- 按规则文件（路径规则、crate 规则、全量触发规则）求值得到逻辑目标 key 列表（`targets`）；
- 在 CI detect 阶段按 `target_key` 过滤预置候选矩阵，生成最终并行 job，避免无关目标全量执行。

#### 接入方式（落地步骤）

- 引用方式：在组件仓库 workflow 中显式拉取 `arceos-hypervisor/axci`（固定分支或 commit），复用其 `axci-affected` 与规则处理逻辑；
- 在仓库测试入口（如 `tests.sh`）接入 `--auto-target` 与 `--base-ref` 参数，支持按基线分支自动选择目标；
- 在 workflow（如 `.github/workflows/test.yml`）增加 detect 阶段：先计算 `targets`，再按 `target_key` 过滤预置矩阵并输出 JSON；
- 在执行阶段使用 `matrix.include: ${{ fromJson(...) }}` 并行运行目标任务，`skip_all` 时直接跳过无关 job；
- 保留回退路径：`axci-affected` 不可用时回退到 shell 规则匹配，保证 CI 可用性与渐进迁移。

#### 规则自定义（可配置能力）

- 规则文件默认位于 `configs/test-target-rules.json`；组件仓库可在 `.github/axci-test-target-rules.json` 放置自定义规则，无需改动 axci 主仓代码；
- 组件侧规则可按仓库测试拓扑覆盖目标映射（如新增/删除 target key、调整 `target_order`、补充路径或 crate 触发条件）；
- 可按目录/文件模式定义 `selection_rules`，将路径变更映射到测试目标；
- 可按 crate 维度定义 `crate_rules`（含 `direct_only`），区分仅直接变更还是包含依赖扩散影响；
- 可通过 `run_all_patterns`、`run_all_crates` 定义“全量触发条件”，并用 `non_code` 规则跳过纯文档类变更；
- 通过 `target_order` 统一目标输出顺序，保证选择结果稳定、可预期、便于回归对比。

#### 关键工作

1. 合并模块化重构并统一测试入口
- 将 `main` 分支的脚本模块化重构合入当前分支，形成 `lib/*.sh` 分层结构；
- 保持 `tests.sh` 作为统一入口，兼容已有流程并提升后续可维护性。

2. 引入规则驱动自动目标选择
- 在 `tests.sh` 增加 `--auto-target`、`--base-ref` 能力；
- 新增 `configs/test-target-rules.json`，将路径匹配与依赖规则配置化；
- 优先使用 `axci-affected` 引擎做影响范围分析，失败时回退到 shell 规则匹配，保证可用性。

3. 增强 CI 可观测性与稳定性
- 在 `test.yml` 增加 `test_targets=auto` 相关输入与 `detect-targets` 检测链路；
- 输出自动选择决策摘要（selection mode、auto reason、target list）到 `GITHUB_STEP_SUMMARY`；
- 补充 git 网络抗抖动参数、checkout 超时与关键依赖检查，降低网络和环境抖动带来的不确定失败。

4. 加固 Starry 测试链路
- 将测试触发改为 `scripts/ci-test.py`，缓解长驻命令导致的超时风险；
- 在运行前增加 `disk.img` 检查与软链兜底逻辑，减少镜像路径问题导致的无效失败。

#### 阶段性价值

- 效率收益：为“按影响范围执行测试”打通主链路，预期可显著减少无关测试开销；
- 工程收益：测试能力从脚本硬编码向“规则配置 + 引擎分析”演进；
- 质量收益：自动选择过程具备可解释输出，便于排障与规则迭代；
- 扩展收益：模块化后更便于后续新增 target、suite 与规则。

#### 相关代码（速查）

- 依赖感知引擎：[axci-affected/src/engine.rs](https://github.com/yoinspiration/axci/blob/test/auto-target-regression/axci-affected/src/engine.rs)
- 回退脚本：[scripts/affected_crates.sh](https://github.com/yoinspiration/axci/blob/test/auto-target-regression/scripts/affected_crates.sh)
- 规则配置：[configs/test-target-rules.json](https://github.com/yoinspiration/axci/blob/test/auto-target-regression/configs/test-target-rules.json)
- CI 检测与矩阵编排：[.github/workflows/test.yml](https://github.com/yoinspiration/axci/blob/test/auto-target-regression/.github/workflows/test.yml)
- 测试入口与自动目标选择：[tests.sh](https://github.com/yoinspiration/axci/blob/test/auto-target-regression/tests.sh)

## EEVDF

### 代表性成果：StarryOS 中 EEVDF 调度器实现与验证

- 仓库链接：[StarryOS feat/eevdf-scheduler](https://github.com/yoinspiration/StarryOS/tree/feat/eevdf-scheduler)
- 当前状态：功能完成并形成文档、单测与演示脚本闭环

#### 背景问题

在操作系统调度中，需要同时满足两类目标：
- 公平性：不同优先级任务应按权重获得合理 CPU 份额；
- 响应性：交互任务应尽快获得服务，避免高负载下长尾延迟。

传统仅按时间片轮转或仅按 vruntime 最小选择，难以同时兼顾“公平份额”与“截止期驱动响应”。因此在 StarryOS 上实现 EEVDF（Earliest Eligible Virtual Deadline First）调度器，验证其在可解释性、公平性和可观测性上的工程价值。

#### 关键工作

1. 完成 per-task EEVDF 核心调度逻辑
- 在 `crates/axsched/src/eevdf.rs` 中实现 `EevdfScheduler` 与 `EevdfEntity`；
- 任务实体维护 `vruntime`、`deadline`、`nice`、`slice` 等关键元数据；
- 采用 Linux 兼容 nice->weight 映射（-20..19）并据此计算 vruntime 增量与 deadline。

2. 设计双索引结构，兼顾选择效率与资格判断
- `ready_queue`：按 `(deadline, id)` 排序，快速获得最早 deadline 任务；
- `vrt_set`：按 `(vruntime, id)` 排序，用于 `vruntime <= V` 的 eligible 范围查询；
- `id_to_deadline`：连接两套索引，保障在慢路径下仍可高效定位候选任务。

3. 完成 EEVDF 选取与抢占策略
- `pick_next_task`：优先走快路径（最早 deadline 且 eligible），否则走慢路径筛选 eligible 中 deadline 最小任务；
- 当无 eligible 任务时启用 fallback（直接取最早 deadline）保证系统可推进；
- `task_tick` 中实现 deadline 驱动抢占：若队首任务 eligible 且 deadline 更早，则触发抢占。

4. 完成与运行队列集成及可观测性建设
- 在 `crates/axtask/src/run_queue.rs` 接入 tick 调度与抢占路径；
- 增加 EEVDF 统计项（picks、preempt、slice_expired、fallback）及周期日志输出；
- 提供 `scripts/demo-eevdf-stats.sh`、`scripts/bench-regression-eevdf.sh`、`scripts/parse-eevdf-stats-log.sh`，形成“采集-解析-回归”自动化链路。

5. 支持多 CPU 指定调度算法（per-CPU 异构调度）
- 设计并实现调度器元数据分离，支持不同 CPU 绑定不同调度算法，避免全局单策略耦合；
- 引入 `CPU_SCHED` 编译期配置，支持按 CPU 维度声明调度策略；
- 补充跨调度器迁移路径的设计与验证要点，保证任务迁移过程的状态一致性与可预期行为。
- 与 Linux 现状相比，当前方案仍以编译期静态指定为主：尚未覆盖运行时动态策略切换、成熟的跨 CPU 负载均衡协同以及更完整的调度域/拓扑感知能力。

6. 补齐文档与验证闭环
- 输出概念与实现文档：`docs/report/eevdf-concept.md`、`docs/starry-scheduling.md`；
- 沉淀测试与演示报告：`docs/report/eevdf-unit-tests-summary.md`、`docs/report/eevdf-nice-demo-summary.md`；
- 给出从理论到实测的一体化说明，降低后续同学接手成本。

#### 实验结果

1. 单元测试结果
- 等权重公平性测试：3 个 nice=0 任务长期运行后，CPU 占比误差控制在预期范围内；
- 加权公平性测试：nice -5/0/+5 场景下，CPU 占比与权重比一致性良好；
- 抢占与 deadline 修正测试：覆盖“时间片耗尽”和“提前抢占后剩余时间片重算”路径；
- fallback 场景测试：在强制无 eligible 条件下，兜底逻辑与统计计数行为符合预期。

2. QEMU 实测表现
- 在 StarryOS 压测场景中，`preempt_by_deadline` 与 `slice_expired` 均有稳定触发；
- `fallback_no_eligible` 在常规负载下接近 0，说明系统大多数时刻可由 eligible 主路径完成调度；
- 从日志观察到调度行为可解释、趋势稳定，支持后续回归比较与参数调优。

#### 结果与价值

- 算法落地：将 EEVDF 从概念层落到可运行、可测试、可观测的内核实现；
- 工程可维护性：双索引 + 统计设计使问题定位路径清晰，便于持续迭代；
- 测试体系收益：建立了单测、演示、回归脚本三层验证，减少调度改动引入回归风险；
- 团队协作收益：文档化沉淀完整，便于新成员快速理解调度设计与验证方法。

#### 相关代码（速查）

- EEVDF 核心实现：`crates/axsched/src/eevdf.rs`
- 调度器抽象与 per-CPU 支持：`crates/axsched/src/per_cpu.rs`
- 运行队列与调度接入：`crates/axtask/src/run_queue.rs`
- 多 CPU 策略注入：`axruntime-patched/build.rs`、`axruntime-patched/src/lib.rs`
- 编译期配置触发：`kernel/build.rs`
- 观测与回归脚本：`scripts/demo-eevdf-stats.sh`、`scripts/bench-regression-eevdf.sh`、`scripts/parse-eevdf-stats-log.sh`

#### 经验复盘

1. 调度算法实现不仅是“选下一个任务”，更关键是数据结构设计与状态一致性维护；
2. 可观测性应与算法实现同步建设，否则难以在真实负载下解释行为差异；
3. 对“无 eligible”这类边界路径提前设计 fallback 与测试，能显著降低线上不确定性；
4. 通过“文档 + 脚本 + 单测”三位一体沉淀，能让调度改动从个人经验升级为团队资产。