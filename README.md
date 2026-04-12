# OS Internship Log

开源社区实习日志仓库 - 贾一飞  

## 文档

- [StarryOS：EEVDF 调度与优先级接口实习报告](./docs/starryos-eevdf-class-scheduler-report.md)（与 StarryOS `docs/report.md` 同步归档；文件名保留历史命名）
- [实习计划](./实习计划.md)
- [第一阶段学习总结](./第一阶段学习总结.md)
- [自动测试系统部署文档](./自动测试系统部署文档.md)
- [多组织共享测试环境实施文档](./多组织共享测试环境实施文档.md) - 基于文件锁的 Runner 并发控制
- 例会报告： [2月7日例会（report_0207 仓库）](https://github.com/yoinspiration/report_0207)

### 上游文档快照（`docs/`）

> 与 axvisor / axci / github-runners 仓库同步的说明类文档，**权威版本以各源仓库为准**。

- [axci 依赖感知：工作原理](./docs/axci-工作原理.md)（源：`axci/docs/工作原理.md`）
- [AxVisor 与 axci 集成说明](./docs/axvisor-axci-integration.md)（源：`axvisor/doc/axci-integration.md`）
- [多组织部署完整指南](./docs/github-runners-多组织部署指南.md)（源：`github-runners/docs/多组织部署指南.md`）
- [github-runners：PR 贡献与仓库现状说明](./docs/github-runners-PR-contribution-summary.md)
- [视频演示：测试目标选择和多组织部署](https://www.bilibili.com/video/BV1d9ATzgEa9/)

## 周报

- [第1周（1月19日-1月25日）](./logs/week1.md) - 项目分析、环境搭建、aarch64 多 Guest 测试
- [第2周（1月26日-2月1日）](./logs/week2.md) - x86_64 NimbOS 验证、日志仓库整理
- [第3周（2月2日-2月8日）](./logs/week3.md) - 多组织共享并发控制、runner-wrapper、部署文档完善
- [第4周（2月9日-2月22日，含春节周）](./logs/week4.md) - 上游化准备、共享方案迭代、目标选择方案设计
- [第6周（2月23日-3月1日）](./logs/week6.md) - Cancel 自动解锁实现、无感集成改造
- [第7周（3月2日-3月8日）](./logs/week7.md) - 自动目标选择规则完善与 CI 接入
- [第8周（3月9日-3月15日）](./logs/week8.md) - EEVDF 实现推进与语义测试补充
- [第9周（3月16日-3月22日）](./logs/week9.md) - 统计可观测性增强与 nice 对照实验
- [第10周（3月23日-3月29日）](./logs/week10.md) - 3月技术报告撰写与链路收尾
- [第11周（3月30日-4月5日）](./logs/week11.md) - 周报补齐、索引完善与4月计划准备
- [第12周（4月6日-4月12日）](./logs/week12.md) - EEVDF 核心算法的实现和支持不同 CPU 使用不同的调度算法

## 月报

- [2月技术报告](./技术报告2月.md)（部署流程简化、多组织共享、依赖感知测试）
- [3月技术报告](./技术报告3月-贾一飞.md)（Cancel 自动解锁、依赖感知测试、EEVDF 调度实现）
