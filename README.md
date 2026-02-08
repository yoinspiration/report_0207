# 2月7日例会

这段时间围绕「QEMU 自动测试」和「多组织共享硬件测试环境」主要完成了三件事：
1. 在 AxVisor 仓库中完成 QEMU 自动测试脚本和配置改动，并提交上游 [PR #363](https://github.com/arceos-hypervisor/axvisor/pull/363)。
2. 系统性完善了[自动测试系统部署文档](自动测试系统部署文档.md)，沉淀为可复用的操作手册。
3. 为多组织共享同一套硬件测试环境设计并实现了 Runner 锁包装方案，并在 github-runners 仓库提交 [PR #2](https://github.com/arceos-hypervisor/github-runners/pull/2)。

## 一、在 QEMU 环境下复现已有的自动测试功能

在 [axvisor](https://github.com/yoinspiration/axvisor/tree/dev/qemu-test-enhancement) 仓库的 `scripts` 目录下添加了 **AxVisor QEMU 测试环境的统一准备脚本** `setup_qemu.sh`，用于在运行 QEMU 测试前自动完成镜像下载、配置修改和 rootfs 准备（对应上游 [PR #363](https://github.com/arceos-hypervisor/axvisor/pull/363)）。

### 1.1 功能概览

| 步骤 | 作用 |
|------|------|
| 1. 下载 Guest 镜像 | 若 `/tmp/axvisor/<镜像名>/` 不存在，通过 `cargo xtask image download` 下载 |
| 2. 修改 VM 配置 | 将 VM 配置中的 `kernel_path` 改为实际镜像路径 |
| 3. 准备 rootfs | 复制 rootfs 到 `tmp/rootfs.img`；NimbOS 还需把 kernel 注入 rootfs |

### 1.2 支持的 Guest

| 参数 | 架构 | Guest | 成功标志 |
|------|------|-------|----------|
| `arceos` | aarch64 | ArceOS | `Hello, world!` |
| `linux` | aarch64 | Linux | `test pass!` |
| `nimbos` | x86_64 | NimbOS | `usertests passed!`（需 VT-x/KVM） |

### 1.3 使用方式

```bash
./scripts/setup_qemu.sh arceos
./scripts/setup_qemu.sh linux
./scripts/setup_qemu.sh nimbos
```

### 1.4 执行完成后

脚本会输出对应的 `cargo xtask qemu` 命令，例如：

```bash
cargo xtask qemu \
  --build-config configs/board/qemu-aarch64.toml \
  --qemu-config .github/workflows/qemu-aarch64.toml \
  --vmconfigs configs/vms/arceos-aarch64-qemu-smp1.toml
```

### 1.5 NimbOS 特殊处理

NimbOS 使用 `image_location=fs`，kernel 需放在 rootfs 内。脚本会依次尝试以下方式挂载并注入 kernel：

- `mount -o loop` 挂载整个磁盘
- `mount -o loop,offset=...` 挂载分区
- `guestmount`（libguestfs-tools）挂载

任一种成功即可完成 kernel 注入。

**当前状态**：三个 Guest（ArceOS、Linux、NimbOS）在本地和 CI 环境均已跑通，对应改动已通过上游 [PR #363](https://github.com/arceos-hypervisor/axvisor/pull/363) 提交 AxVisor。  

---

## 二、细化和完善「自动测试系统部署文档」

完善了 [自动测试系统部署文档](自动测试系统部署文档.md)，主要更新包括：

- **QEMU 与硬件测试对照表**：对比 QEMU 与硬件平台（ROC-RK3568-PC、飞腾派等）的测试能力
- **x86_64 特殊要求**：明确 AxVisor 仅支持 Intel VT-x，补充 WSL2 嵌套虚拟化及 AMD 兼容性说明
- **Guest 镜像来源与版本**：说明 axvisor-guest 仓库、下载地址及镜像列表
- **统一 setup 脚本用法**：集成 `setup_qemu.sh` 的快速开始说明
- **配置文件说明**：补充 `success_regex` / `fail_regex` 的配置与扩展建议
- **CI 配置说明**：细化触发条件、测试矩阵、Runner 要求及镜像修补逻辑
- **故障排查**：新增 8 类常见问题（含 Intel/AMD、WSL2、KVM 权限等）

**当前状态**：文档已覆盖从环境要求、部署步骤到故障排查的一整套流程，可直接支撑新人或其他组织在本地/CI 复用现有测试体系。  

---

## 三、多组织共享测试环境（对应上游 [PR #2](https://github.com/arceos-hypervisor/github-runners/pull/2)）

### 3.1 问题项与状态

| 问题项 | 状态 | 说明 |
|--------|------|------|
| 跨组织任务调度 | 🟡 部分完成 | 通过[锁机制](多组织共享测试环境实施文档.md)实现多组织 Runner 共享同一硬件，执行时串行 |
| 并发控制机制 | ✅ 已完成 | 基于 `flock` 的 [runner-wrapper 锁机制](多组织共享测试环境实施文档.md)，防止资源竞争 |
| 统一任务队列 | ❌ 未完成 | GitHub 架构限制，各组织 Runner 独立接收任务，无法统一队列 |

### 3.2 已实现方案

采用 [Discussion #341](https://github.com/orgs/arceos-hypervisor/discussions/341) 中的「修改 Runner 程序」方案，实现 `runner-wrapper.sh`（详见 [多组织共享测试环境实施文档](多组织共享测试环境实施文档.md)）：

- **锁机制**：`flock` 排他锁，按 `RUNNER_RESOURCE_ID` 区分，同一资源 ID 的 Runner 串行执行
- **部署方式**：替换 Runner 的 `run.sh` 调用为 `runner-wrapper.sh`，零外部依赖

### 3.3 剩余限制

- **GitHub Actions 多租户隔离**：Runner 仍按组织注册，无法跨组织共享同一 Runner 实例
- **单机锁**：文件锁仅在同一主机有效，多机部署需 Redis/etcd 等分布式方案
- **无优先级**：先到先得，无法为特定组织设置任务优先级

**当前状态**：基于文件锁的单机场景方案已在 github-runners 中落地，并通过本地多组织模拟验证，可以用于现阶段的大部分共享硬件测试场景。  
