# kclaw Development Guide

日常开发、调试、测试、验证完整工作流。基于 Fork 锚点版本 `openclaw@2026.5.19`。

## 1. 环境搭建

```bash
# 前置条件
node -v                            # 必须 >= 22.19.0
corepack enable                    # 启用 pnpm@11.1.0

# 安装依赖
pnpm install

# ARM64 / 国产平台 (麒麟 V10 / 统信 UOS) sharp 编译失败时
SHARP_IGNORE_GLOBAL_LIBVIPS=1 pnpm install
```

**关键配置**:

- `pnpm-workspace.yaml` 使用 `nodeLinker: hoisted` —— 扁平化 `node_modules` 布局，非 pnpm 默认隔离模式
- `package.json` `packageManager` 字段锁定 `pnpm@11.1.0`
- 项目使用 `tsdown/rolldown` 构建，`tsconfig.json` 仅用于 IDE 支持（`noEmit: true`）

## 2. 编译与运行

### 2.1 首次启动完整流程

```bash
# Step 1: 安装依赖
pnpm install

# Step 2: 编译
pnpm build

# Step 3: 初始化配置
pnpm openclaw setup                # 写入 ~/.openclaw/ 本地配置和工作区

# Step 4: 启动 Gateway
pnpm gateway:dev                   # 跳过通道加载的快速启动
# 或完整启动: pnpm openclaw gateway --port 18789 --bind loopback

# Step 5: 验证
curl http://127.0.0.1:18789/healthz   # → {"status":"ok"}
```

> **国产平台 ARM64**: 编译原生模块失败时 `SHARP_IGNORE_GLOBAL_LIBVIPS=1 pnpm install`。

### 2.2 日常开发循环

```bash
# 编译
pnpm build                         # 完整构建 → dist/
pnpm build:strict-smoke            # SDK/出口边界变更时必须跑
                                   #   = build + plugin-sdk:dts + export checks
pnpm tsgo:prod                     # core + extensions 全量类型检查 (tsgo, 非 tsc)
pnpm tsgo:test                     # 测试文件类型检查

# 运行
pnpm openclaw ...                  # CLI 入口 (调用 node scripts/run-node.mjs)
pnpm dev                           # 等价于 pnpm openclaw
pnpm gateway:dev                   # 跳过通道加载的快速启动
pnpm gateway:watch                 # 文件变更自动重启 (tmux-based)
pnpm gateway:watch:raw             # 文件变更自动重启 (裸 process)

# 常用诊断
pnpm openclaw doctor               # 系统诊断
pnpm openclaw channels list        # 查看通道
pnpm openclaw models list          # 查看模型
pnpm openclaw gateway status       # Gateway 状态
```

**构建组成** (`pnpm build` = `node scripts/build-all.mjs`):

- `plugins:assets:build` — 插件静态资源
- `tsdown` — rolldown 打包
- `check-cli-bootstrap-imports` — CLI 启动导入验证
- `runtime-postbuild` / `build-stamp` / `runtime-postbuild-stamp` — 运行时标记

**环境变量覆盖**:

```bash
OPENCLAW_CONFIG_DIR=/tmp/kclaw-test pnpm openclaw gateway --port 18790
OPENCLAW_CHILD_OOM_SCORE_ADJ=0 pnpm openclaw gateway
NODE_COMPILE_CACHE=/var/tmp/kclaw-cache pnpm openclaw gateway
```

### 2.3 首次运行常见问题

**问题 1: `pnpm dev` 启动的是 Crestodian 而非 Gateway**

`pnpm dev` 调用内置配置助手 Crestodian。如果提示 "config missing" 或 "Gateway not reachable"，是因为尚未初始化配置和启动 Gateway。按 [2.1 首次启动完整流程](#21-首次启动完整流程) 从 Step 3 开始执行即可。Gateway 启动后 Crestodian 会自动探测到并进入就绪状态。

**问题 2: Control UI 提示需要认证令牌**

Gateway 启动后通过浏览器访问 Control UI 页面时可能提示需要令牌。在 Gateway 主机上查看配置文件中已有的 token：

```bash
cat ~/.openclaw/openclaw.json | grep token
```

将输出的 token 值粘贴到 Control UI 的认证输入框中点击 Connect 即可。

**问题 3: Control UI 提示需要设备配对**

输入令牌后，新浏览器首次连接可能需要设备批准：

```bash
pnpm openclaw devices approve <request-id>
```

request-id 来自 Control UI 的错误提示中。也可先运行 `pnpm openclaw devices list` 查看待批准设备列表。批准后在 Control UI 中点击 Connect 完成配对。

## 3. 测试

### 运行测试

```bash
# 单文件 (最常用)
pnpm test src/security/secret-equal.test.ts

# 模式匹配
pnpm test "auth-rate-limit"

# 快速回归
pnpm test:unit:fast                # 快速子集 (vitest.unit-fast.config.ts)
pnpm test:unit                     # 快速 + 完整单元测试

# 变更文件 (vs origin/main)
pnpm test:changed

# 全量
pnpm test                          # node scripts/test-projects.mjs
pnpm test:serial                   # 单线程全量 (KCLAW_VITEST_MAX_WORKERS=1)

# 扩展测试
pnpm test extensions               # 所有扩展
pnpm test extensions/<id>          # 单扩展 (例: extensions/telegram)

# 覆盖率
pnpm test:coverage                 # 全量单元覆盖率
pnpm test:coverage:changed         # 仅变更文件覆盖率
```

### 安全模块测试

```bash
# M01 - 恒时校验
pnpm test src/security/secret-equal.test.ts

# M03 - 挂载管控
pnpm test src/agents/sandbox/validate-sandbox-security.test.ts

# M04 - 权限最小化
kclaw exec "cat /proc/1/status | grep CapEff"     # → 0000000000000000

# M10 - 会话加密
hexdump -C ~/.kclaw/workspace/sessions/latest.json | head -1  # → 4f 43 45 43

# M09 - 熔断 (需模拟)
# 连续 5 次失败 → 第 6 次立即拒绝 → 30s 后半开探测
```

### 内存与性能

```bash
KCLAW_VITEST_MAX_WORKERS=1 pnpm test <path>       # 单线程 (断点调试)
KCLAW_LIVE_TEST=1 pnpm test:live                  # 实时测试
KCLAW_LIVE_TEST_QUIET=0 pnpm test:live            # 详细实时输出
KCLAW_VITEST_IMPORT_DURATIONS=1 pnpm test          # 导入耗时分析
```

**重要**: 所有测试必须通过 `node scripts/run-vitest.mjs` 或 `pnpm test` 入口运行，禁止直接执行 `vitest` 命令。不要同时运行多个独立的 `pnpm test` 进程——Vitest 缓存会触发 `ENOTEMPTY` 竞争。

## 4. 门禁检查（提交前）

```bash
# 推荐: 变更文件全量门禁
pnpm check:changed                 # lint + typecheck + test

# 暂存文件
pnpm check:changed --staged        # 仅 staged 变更

# 单独检查
pnpm format:check                  # 格式化检查 (oxfmt, print width 100)
pnpm lint:core                     # lint src/ui/packages (oxlint)
pnpm lint:all                      # 全量 lint
pnpm tsgo:prod                     # 类型检查

# 全量门禁
pnpm check                         # node scripts/check.mjs
```

**oxfmt 格式化规则**:

- print width: **100** 字符 (非默认 80)
- 使用分号、双引号、tab width 2
- 忽略目录: `apps/`, `dist/`, `patches/`, `vendor/`, `pnpm-lock.yaml/`

## 5. 调试

### 日志

```bash
journalctl --user -u kclaw-gateway -f -n 200     # Gateway 日志
./scripts/clawlog.sh                              # 日志快捷脚本
grep -i "sk-\|AIza\|ghp_" ~/.kclaw/state/*.jsonl # 脱敏检查 (应无输出)
```

### Node.js 调试

```bash
# Chrome DevTools
node --inspect openclaw.mjs gateway --port 18789

# 启动时暂停等待附加
node --inspect-brk openclaw.mjs gateway --port 18789

# 堆快照
node --heap-prof openclaw.mjs gateway
```

### 常见问题排查

| 症状             | 检查步骤                                                         |
| ---------------- | ---------------------------------------------------------------- |
| Gateway 启动失败 | `pnpm openclaw doctor` → `journalctl --user -u kclaw-gateway -e` |
| 飞书收不到消息   | `pnpm openclaw channels status` → 检查 Webhook 地址可达性        |
| 模型调用超时     | `curl -v <provider-api-url>` → 检查 `HTTPS_PROXY`                |
| OOM 崩溃         | `journalctl` 查看 OOM 记录 → 检查 `--max-old-space-size`         |
| 离线安装失败     | `uname -m` 确认架构 → `ldd --version` 检查 glibc                 |

## 6. Git 工作流

```bash
# 提交 (必须用项目脚本, 禁止手动 git commit)
scripts/committer "<msg>" <file...>

# 查看变更
git status
git diff
git log --oneline -10

# 推送前
pnpm check:changed                    # 先过门禁
git pull --rebase origin main         # rebase 到最新 main
git push

# 约定
# - 提交信息: conventional-ish, 简洁, 分组
# - main 分支: 禁止 merge commit, 必须 rebase
# - 不要手动 stash/autostash
# - commit: 你的变更; commit all: 所有变更分组提交
# - push: 可以先 git pull --rebase
# - ship it: 更新 changelog (如需要) → commit → pull --rebase → push
```

---

## 附录: kclaw 特定注意事项

### 重命名前后对照

| 项目     | Fork 前 (当前)          | Fork 后 (kclaw)      |
| -------- | ----------------------- | -------------------- |
| CLI      | `pnpm openclaw`         | `pnpm kclaw`         |
| 入口文件 | `openclaw.mjs`          | `kclaw.mjs`          |
| 环境变量 | `OPENCLAW_*`            | `KCLAW_*`            |
| 配置目录 | `~/.openclaw/`          | `~/.kclaw/`          |
| 配置文件 | `openclaw.json`         | `kclaw.json`         |
| 导入路径 | `openclaw/plugin-sdk/*` | `kclaw/plugin-sdk/*` |
| 清单文件 | `openclaw.plugin.json`  | `kclaw.plugin.json`  |

### 国内网络问题

国内环境访问 `registry.npmjs.org` 可能遇到下载失败或超时，常见错误码包括：

- `error (23)` — 网络连接不稳定导致数据传输中断
- `UND_ERR_SOCKET` — TCP 连接被重置
- `The operation was aborted due to timeout` — 请求超时

大型二进制包（如 `@openai/codex` 88MB、`@anthropic-ai/claude-agent-sdk` 78MB）更容易触发此类问题。

**解决方案（按优先级尝试）**:

1. 切换国内镜像源：

   ```bash
   pnpm config set registry https://registry.npmmirror.com
   pnpm install
   ```

2. 增大超时时间（镜像源仍不稳定时）：

   ```bash
   pnpm config set fetch-timeout 600000
   pnpm install
   ```

3. 配置代理（企业内网环境）：
   ```bash
   HTTPS_PROXY=http://<代理地址>:<端口> pnpm install
   ```

### 重命名验证

```bash
grep -r "OPENCLAW" src/ --include="*.ts" --include="*.mjs"  # 应无结果
grep -r "openclaw/plugin-sdk" src/ --include="*.ts"          # 应无结果
```

### 不重命名范围

- `extensions/` — 保持与 OpenClaw 插件生态兼容
- `src/gateway/protocol/` — 网关协议版本号保留
- `LICENSE`, `CONTRIBUTORS` — 法律文件保持原样

### 版本对照

| 说明           | 版本                                 |
| -------------- | ------------------------------------ |
| Fork 锚点      | `openclaw@2026.5.19`                 |
| 最新 GHSA 修复 | `2026.5.26` (3 条未包含在本 Fork 中) |

### 额外资源

- 项目计划: `docs/reference/kclaw-project-plan.md`
- 全局架构: `AGENTS.md`
- 测试指南: `docs/reference/test.md`
- 构建/测试脚本: `scripts/AGENTS.md`
- 扩展指南: `extensions/AGENTS.md`
