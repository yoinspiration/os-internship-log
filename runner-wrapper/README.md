# Runner Wrapper - 多组织共享硬件锁

基于文件锁的 GitHub Actions Runner 包装脚本，用于多组织共享同一硬件测试环境时的并发控制。

## 快速使用

```bash
chmod +x runner-wrapper.sh
export RUNNER_RESOURCE_ID=hardware-test-1
export RUNNER_SCRIPT=/path/to/run.sh
./runner-wrapper.sh
```

## 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `RUNNER_RESOURCE_ID` | `default-hardware` | 锁资源 ID，相同 ID 的 Runner 串行执行 |
| `RUNNER_SCRIPT` | `./run.sh` | 实际 Runner 脚本路径 |
| `RUNNER_LOCK_DIR` | `/tmp/github-runner-locks` | 锁文件目录 |

## 依赖

- `flock`（通常随 util-linux 提供）
- Bash

## 详细文档

参见 [多组织共享测试环境实施文档.md](../多组织共享测试环境实施文档.md)。
