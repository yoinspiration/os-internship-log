# 开发日志 - 第12周（4月6日 - 4月12日）

**记录人**：贾一飞  
**实习方向**：StarryOS EEVDF 调度相关实现

---

## 本周工作内容

### EEVDF 核心算法实现与多 CPU 调度策略支持

- 整理 EEVDF 与 CFS 的差异、eligible 与 deadline 的直觉说明，以及 StarryOS 中的数据结构、选路、抢占与入队语义。
- 实现 EEVDF 核心算法，并完成关键路径联调与验证。
- 补充 per-CPU 异构调度（元数据分离、`CPU_SCHED` 编译期配置、跨调度器迁移）的设计与验证要点。
- **正文与详细推导、实测数据见**：[StarryOS `docs/starry-scheduling.md`（分支 `feat/eevdf-scheduler`）](https://github.com/yoinspiration/StarryOS/blob/feat/eevdf-scheduler/docs/starry-scheduling.md)。

