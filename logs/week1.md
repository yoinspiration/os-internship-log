# 开发日志 - 第1周（1月20日 - 1月26日）

**记录人**：贾一飞  
**实习方向**：AxVisor 自动测试系统开发与完善

---

## 本周工作内容

### 1. 项目结构分析与文档阅读

- 完成 AxVisor 项目结构分析，输出文档 `AxVisor项目结构分析.md`
- 阅读核心文档：
  - [AxVisor 集成测试环境文档](https://arceos-hypervisor.github.io/axvisorbook/docs/design/test)
  - [集成测试环境完善](https://github.com/orgs/arceos-hypervisor/discussions/346)
  - [开发日志管理规范](https://github.com/orgs/arceos-hypervisor/discussions/197)
- 分析现有硬件平台测试流程（ROC-RK3568-PC、飞腾派）

### 2. 本地开发环境搭建

- 安装 QEMU（qemu-system-aarch64、qemu-system-x86_64）
- 安装 Rust 工具链和交叉编译工具链
- Fork/Clone AxVisor 仓库，创建开发分支

### 3. QEMU 环境测试方案设计

- 研究 QEMU 虚拟化配置
- 设计 QEMU 环境下的测试方案
- 对比硬件平台和虚拟化平台的差异
- 输出文档 `QEMU测试方案设计.md`

### 4. 多 Guest 测试验证（aarch64）

#### ArceOS Guest 测试
- 使用 `cargo xtask image download qemu_aarch64_arceos` 下载镜像
- 配置 `configs/vms/arceos-aarch64-qemu-smp1.toml`
- 执行测试命令，成功看到 `Hello, world!`
- **测试通过** ✅

#### Linux Guest 测试
- 使用 `cargo xtask image download qemu_aarch64_linux` 下载镜像
- 配置 `configs/vms/linux-aarch64-qemu-smp1.toml`
- 执行测试命令，成功看到 `test pass!`
- **测试通过** ✅

### 5. 文档输出

- `docs/QEMU-aarch64部署文档.md` - QEMU aarch64 环境部署指南
- `docs/QEMU测试方案设计.md` - 测试方案设计文档
- `docs/多Guest测试记录.md` - 多 Guest 测试详细记录
- `docs/核心文档阅读总结.md` - 核心文档阅读笔记
- `docs/环境搭建状态.md` - 环境搭建状态记录
- `docs/硬件平台测试流程分析.md` - 硬件平台测试流程分析
- `docs/测试系统深入分析.md` - 测试系统深入分析

---

## 遇到的问题与解决方案

### 问题1：rootfs.img 路径问题
- **问题**：QEMU 配置中 rootfs 路径与实际路径不一致
- **解决**：将镜像复制到 `tmp/rootfs.img`，与 QEMU 配置保持一致

### 问题2：VM 配置中 kernel_path 需要修改
- **问题**：默认 VM 配置的 `kernel_path` 指向不存在的路径
- **解决**：修改为实际下载的镜像路径 `/tmp/axvisor/qemu_aarch64_*/qemu-aarch64`

---

## 下周计划（2月3日 - 2月9日）

1. **x86_64 NimbOS 测试**
   - 在支持 VT-x 的环境中验证 NimbOS 测试流程
   
2. **第一阶段学习总结**
   - 整理学习笔记
   - 准备阶段总结报告

3. **合作沟通**
   - 联系蒋奇润，讨论后续合作分工

4. **测试配置扩展**
   - 扩展 qemu-aarch64.toml 的 success_regex、fail_regex

---

## 学习收获

1. 理解了 AxVisor 的整体架构和组件化设计
2. 掌握了 `cargo xtask` 测试工具链的使用方法
3. 理解了 ostool 的测试流程（构建 → 启动 QEMU → 捕获输出 → 判定结果）
4. 了解了硬件平台（U-Boot loady/TFTP）和 QEMU 虚拟化环境的差异

---

**日志创建时间**：2026年2月3日
