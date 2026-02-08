# 多组织 Runner 锁验证操作指南

本文档提供完整的一步步操作指导，用于在**无组织管理员权限**的情况下，使用自己的 GitHub 仓库验证 runner-wrapper 的锁机制是否生效。

---

## 前提条件

- 拥有 GitHub 账号
- 服务器已安装 Docker 和 Docker Compose
- 可 SSH 登录到服务器

---

## 第一步：准备两个带 CI 的仓库

### 方式 A：新建两个仓库

1. 在 GitHub 新建仓库 `runner-test-a`（可添加 README）
2. 新建仓库 `runner-test-b`
3. 在两个仓库中创建 `.github/workflows/test.yml`：

```yaml
name: Test
on:
  push:
    branches: [main]
  workflow_dispatch:
jobs:
  test:
    runs-on: [self-hosted]
    steps:
      - run: echo "Job started at $(date)"
      - run: sleep 30
      - run: echo "Job finished at $(date)"
```

提交并推送到各自的 `main` 分支。

### 方式 B：Fork 两个已有仓库

1. Fork 任意两个带 Actions 的仓库到你的账号
2. 确保 workflow 中有 `runs-on: [self-hosted]`，否则修改后提交

---

## 第二步：在服务器上创建两个 Runner 目录

```bash
# SSH 登录服务器
ssh user@your-server

# 创建工作目录
mkdir -p ~/runner-validation
cd ~/runner-validation
mkdir -p repo-a repo-b
```

---

## 第三步：部署第一个 Runner（repo-a）

```bash
cd ~/runner-validation/repo-a

# 克隆 github-runners
git clone https://github.com/arceos-hypervisor/github-runners.git .

# 创建 .env
cat > .env << 'EOF'
ORG=你的GitHub用户名
REPO=runner-test-a
GH_PAT=你的_GitHub_PAT
RUNNER_NAME_PREFIX=repo-a-
RUNNER_BOARD=0
EOF

# 若仓库在组织下，则 ORG=组织名

chmod +x runner.sh
./runner.sh init -n 1
```

`-n 1` 表示创建 1 个普通 runner。

---

## 第四步：创建 PAT 并注册 repo-a 的 Runner

1. GitHub → Settings → Developer settings → Personal access tokens
2. 新建 **Classic PAT**，勾选 `repo`（若用组织仓库则需 `admin:org` 下的 `read:org`）
3. 将生成的 token 填入 `.env` 的 `GH_PAT`
4. 执行：

```bash
cd ~/runner-validation/repo-a
./runner.sh register
./runner.sh start
```

在 GitHub 仓库的 Settings → Actions → Runners 中应能看到该 Runner。

---

## 第五步：部署第二个 Runner（repo-b）

```bash
cd ~/runner-validation/repo-b

git clone https://github.com/arceos-hypervisor/github-runners.git .

cat > .env << 'EOF'
ORG=你的GitHub用户名
REPO=runner-test-b
GH_PAT=你的_GitHub_PAT
RUNNER_NAME_PREFIX=repo-b-
RUNNER_BOARD=0
EOF

chmod +x runner.sh
./runner.sh init -n 1
./runner.sh register
./runner.sh start
```

---

## 第六步：配置 runner-wrapper 共享锁

### 6.1 创建共享锁目录

```bash
sudo mkdir -p /tmp/github-runner-locks
sudo chmod 1777 /tmp/github-runner-locks
```

### 6.2 修改 repo-a 的 docker-compose.yml

```bash
cd ~/runner-validation/repo-a
nano docker-compose.yml   # 或 vim
```

找到 `runner-1`（或 `xxx-runner-1`）的配置，做以下修改：

```yaml
# 将 command 从：
command: ["/home/runner/run.sh"]

# 改为：
command: ["/home/runner/runner-wrapper/runner-wrapper.sh"]

# 在 environment 中添加：
environment:
  RUNNER_RESOURCE_ID: "shared-lock"
  RUNNER_SCRIPT: "/home/runner/run.sh"
  # ... 其他已有变量

# 在 volumes 中添加：
volumes:
  - /tmp/github-runner-locks:/tmp/github-runner-locks
  # ... 其他已有 volumes
```

**若使用官方镜像**（无内置 runner-wrapper），需挂载脚本：

```yaml
volumes:
  - ./runner-wrapper:/home/runner/runner-wrapper:ro
  - /tmp/github-runner-locks:/tmp/github-runner-locks
  # ...
```

### 6.3 修改 repo-b 的 docker-compose.yml

对 repo-b 的 runner-1 做**相同修改**，确保：

- `RUNNER_RESOURCE_ID: "shared-lock"`（与 repo-a 一致）
- 挂载 `/tmp/github-runner-locks`

### 6.4 重启两个 Runner

```bash
cd ~/runner-validation/repo-a
./runner.sh restart

cd ~/runner-validation/repo-b
./runner.sh restart
```

---

## 第七步：确认 workflow 的 runs-on

两个仓库的 workflow 中 `runs-on` 需与 Runner 标签匹配。若使用默认配置，通常为 `[self-hosted, linux, docker]` 或 `[self-hosted]`。按需修改后提交：

```yaml
 runs-on: [self-hosted]   # 或 [self-hosted, linux, docker]
```

---

## 第八步：验证锁机制

### 8.1 同时触发两个 CI

1. 打开 `你的用户名/runner-test-a` → Actions → 选择 Test workflow → Run workflow
2. 打开 `你的用户名/runner-test-b` → Actions → 选择 Test workflow → Run workflow
3. 在几秒内依次点击 Run workflow

### 8.2 查看 Runner 日志

```bash
# 终端 1
cd ~/runner-validation/repo-a
./runner.sh log repo-a-runner-1

# 终端 2
cd ~/runner-validation/repo-b
./runner.sh log repo-b-runner-1
```

### 8.3 预期现象

- **先获得锁的 Runner**：输出 `Acquired lock for shared-lock`，然后执行任务
- **后到的 Runner**：输出 `Waiting for lock: shared-lock`，待前者完成后才执行

若看到上述行为，说明锁机制工作正常。

---

## 快速检查清单

| 检查项 | 说明 |
|--------|------|
| PAT 权限 | 需具备 `repo` 或 `admin:org` |
| RUNNER_RESOURCE_ID | 两个 Runner 必须相同（如 `shared-lock`） |
| 共享锁目录 | 两个 Runner 都挂载 `/tmp/github-runner-locks` |
| command | 两个 Runner 都改为 `runner-wrapper.sh` |
| runs-on | workflow 中的标签与 Runner 实际标签一致 |

---

## 参考资料

- [多组织共享测试环境实施文档](多组织共享测试环境实施文档.md)
- [github-runners 仓库](https://github.com/arceos-hypervisor/github-runners)
- [Discussion #341](https://github.com/orgs/arceos-hypervisor/discussions/341)
