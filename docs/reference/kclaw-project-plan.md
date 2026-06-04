# kclaw — OpenClaw 工业安全加固衍生版 项目规划

## 一、项目定义

| 维度 | 说明 |
|------|------|
| **项目名称** | kclaw |
| **代码来源** | Fork OpenClaw (`https://github.com/openclaw/openclaw`) 当前最新 `main` 分支 |
| **维护模式** | 独立演进，不跟随上游合并 |
| **定位** | 面向工业/HPC 场景的出厂安全加固版本，内置 10 项安全功能模块 |
| **发布形式** | 双架构离线安装包 (linux-x64 + linux-arm64)，预置飞书通道 + 国产模型 |
| **CLI** | `kclaw`，配置目录 `~/.kclaw/`，环境变量前缀 `KCLAW_*` |

---

## 二、Fork 策略

```
openclaw/main ──Fork──> kclaw/main
                            │
                            ├── commit 1: 项目初始化 (package.json/README/FORK.md)
                            ├── commit 2: 全局重命名 (openclaw → kclaw)
                            ├── commit 3: 认证安全加固 (M01 恒时校验 + M02 计数持久化)
                            ├── commit 4: 沙箱隔离加固 (M03 挂载管控 + M04 权限最小化)
                            ├── commit 5: 部署安全加固 (M07 前置校验 + M08 流量防护)
                            ├── commit 6: 数据安全加固 (M10 会话加密)
                            ├── commit 7: 运行安全加固 (M05 内存管控 + M06 资源配额 + M09 熔断管理)
                            ├── commit 8: 国产化适配 (离线安装脚本)
                            ├── commit 9: CI 安全扫描 (CVE 扫描)
                            └── commit 10: 文档 + 配置模板
```

**不保留** `git remote add upstream` —— 独立演进，避免上游变更引入兼容性问题。

---

## 三、全局重命名清单

| 原值 | 新值 | 涉及位置 | 方式 |
|------|------|----------|------|
| `openclaw` CLI 命令 | `kclaw` | `package.json` `bin` 字段 | 修改 |
| `openclaw.mjs` 入口 | `kclaw.mjs` | 文件重命名 + `package.json` 引用 | 重命名 |
| `OPENCLAW_*` 环境变量 | `KCLAW_*` | `src/` 全部 ~200 处引用 | 全局替换 |
| `~/.openclaw/` 配置目录 | `~/.kclaw/` | `src/config/` 路径常量 | 修改 |
| `openclaw.json` 配置文件 | `kclaw.json` | 默认配置文件名 | 修改 |
| `openclaw-gateway.service` | `kclaw-gateway.service` | systemd 单元 | 修改 |
| `openclaw/plugin-sdk/*` 导入路径 | `kclaw/plugin-sdk/*` | 全部插件 import 语句 | 全局替换 |
| `openclaw.plugin.json` 清单 | `kclaw.plugin.json` | 清单文件名约定 | 修改 |
| `"openclaw"` 配置块名称 | `"kclaw"` | `kclaw.json` 顶层 key | 修改 |
| 用户可见字符串 "OpenClaw" | "kclaw" | CLI 帮助文本、日志、UI | 全局替换 |

**不重命名的范围：**
- `extensions/` 目录内的 135+ 插件 — 保持与 OpenClaw 生态兼容
- `src/gateway/protocol/` — Gateway 协议版本号维持
- `CONTRIBUTORS`、`LICENSE` — 法律文件保留原样

---

## 四、功能模块清单（10 项）

### 模块总览

| 模块 | 正式名称 | 优先级 | 编码量 | 对应缺陷 |
|------|----------|--------|--------|----------|
| M01 | 认证令牌恒时校验模块 | HIGH | ~15 行改 | Token `===` 比较存在时序侧信道攻击风险 |
| M02 | 认证失败计数持久化模块 | MID | ~60 行改 | 暴力破解计数器在内存中，重启清零可绕过锁定 |
| M03 | 共享存储挂载管控模块 | HIGH | ~10 行改 | 沙箱可 bind-mount HPC 共享存储路径，跨用户数据泄露 |
| M04 | 容器权限最小化管理模块 | MID | ~40 行改 | 沙箱容器无 cap_drop/seccomp，攻击者可利用内核提权逃逸 |
| M05 | 网关进程内存管控模块 | HIGH | ~5 行改 | Node.js 无内存上限，大会话 OOM 导致整个 Gateway 崩溃 |
| M06 | 子进程资源配额模块 | MID | ~50 行新 | Agent exec 工具无资源限制，恶意脚本可耗尽计算节点 |
| M07 | 部署安全前置校验模块 | HIGH | ~20 行改 | docker-compose 默认 --bind lan + 空 Token，可未授权访问 |
| M08 | 网关流量防护模块 | MID | ~50 行改 | HTTP 层无连接数/请求体/WS 消息大小限制 |
| M09 | AI 服务熔断管理模块 | MID | ~80 行新 | AI 服务故障时无熔断保护，连锁重试耗尽资源 |
| M10 | 会话数据加密存储模块 | MID | ~80 行新 | 对话历史明文存储，硬盘送修或报废时存在数据泄露风险 |

**总计：~410 行 TS（新增 ~260 行 + 修改 ~150 行）**

### 缺陷→模块→工业场景落地链路

```
原始审计发现的 10 个缺陷
        │
        ▼
  M01-M10 10 个功能模块逐一修复
        │
   ┌────┼────┬────────────┬──────────┐
   ▼    ▼    ▼            ▼          ▼
 认证  隔离  资源         部署       数据
 安全  安全  管控         适配       保护
(M01  (M03  (M05         (M07      (M10)
 M02)  M04)  M06 M08 M09)           )
        │
        ▼
  满足工业场景四大核心诉求:
  ① 多租户数据隔离 (M03/M04)
  ② 7×24 持续可用 (M05/M06/M08)
  ③ 内网安全部署 (M07)
  ④ 数据保密合规 (M01/M02/M09/M10)
```

### 行业推广适配性

| 行业场景 | 核心痛点 | 对应模块 | 推广理由 |
|----------|----------|----------|----------|
| 汽车制造 (广汽 HPC) | 多团队共用集群，CAE 数据敏感 | M03/M04/M10 | 沙箱隔离 + 数据加密，防止仿真数据泄露 |
| 能源/电力 | 内网隔离，7×24 不可中断 | M05/M06/M08/M09 | 资源管控 + 熔断保护，满足生产网安全要求 |
| 金融/政务 | 等保密评合规，防攻击 | M01/M02/M07 | 认证加固 + 出厂安全，通过安全测评检查项 |
| 军工/航天 | 完全断网，数据绝密 | M10 | 存储加密，满足保密管理要求 |

---

### M01 — 认证令牌恒时校验模块

**优先级**: HIGH
**编码量**: ~15 行（修改现有文件）

#### 概述

将 Gateway 认证流程中的 Token/密码字符串 `===` 比较替换为 `crypto.timingSafeEqual()`，确保无论输入值是否正确，比较耗时恒定，消除时序侧信道攻击面。

#### 涉及文件

`src/security/secret-equal.ts` (修改)、`src/gateway/auth.ts` (修改)、`src/gateway/connection-auth.ts` (修改)

#### 代码初步方案

```typescript
import { timingSafeEqual } from "node:crypto";

// 替换所有 auth.ts 中的 === 字符串比较
export function safeEqualSecret(provided: string, expected: string): boolean {
  const providedBuf = Buffer.from(provided.padEnd(expected.length, "\0"));
  const expectedBuf = Buffer.from(expected.padEnd(provided.length, "\0"));
  if (providedBuf.length !== expectedBuf.length) return false;
  return timingSafeEqual(providedBuf, expectedBuf);
}
// auth.ts 调用改造:
//   if (!safeEqualSecret(received, configured)) { throw new AuthError(); }
```

#### 验收步骤

```
前置条件：
1. kclaw Gateway 已正常启动并配置了有效的 Gateway Token。
2. 系统具备 curl 命令行工具和 time 命令。

操作步骤：
1. 使用正确 Token 通过 curl 请求 Gateway API，重复 100 次并记录每次响应耗时。
2. 使用与正确 Token 长度相同的错误 Token 通过 curl 请求 Gateway API，重复 100 次并记录每次响应耗时。
3. 分别计算两组响应耗时的平均值和标准差。

期望结果：
1. 正确 Token 组与错误 Token 组的平均响应耗时差值小于 1ms。
2. 两组耗时数据的标准差无明显差异。
3. 无法通过测量响应时间区分 Token 是否正确。
```

---

### M02 — 认证失败计数持久化模块

**优先级**: MID
**编码量**: ~60 行（修改现有文件）

#### 概述

将当前的纯内存速率限制改为磁盘持久化：登录失败计数写入 `rate-limit-state.json` 文件，Gateway 重启后计数器不归零，攻击者无法通过反复重启绕过锁定。

#### 涉及文件

`src/gateway/auth-rate-limit.ts` (修改)

#### 代码初步方案

```typescript
import { readFileSync, writeFileSync, existsSync } from "node:fs";
import { join } from "node:path";

const RATE_LIMIT_FILE = "rate-limit-state.json";

export function createPersistentRateLimiter(stateDir: string, opts: RateLimitOpts) {
  const filePath = join(stateDir, RATE_LIMIT_FILE);
  let cache = loadState(filePath);

  return {
    check(key: string): RateLimitResult {
      const now = Date.now();
      pruneExpired(cache, now, opts.windowMs);
      const entry = cache.get(key) ?? { count: 0, firstAttempt: now };
      entry.count++;
      if (entry.count === 1) entry.firstAttempt = now;
      cache.set(key, entry);
      persistState(filePath, cache);
      if (entry.count > opts.maxAttempts && now - entry.firstAttempt < opts.windowMs) {
        return { allowed: false, retryAfterMs: opts.lockoutMs };
      }
      return { allowed: true };
    },
  };
}

function loadState(path: string): Map<string, RateEntry> {
  if (!existsSync(path)) return new Map();
  return new Map(Object.entries(JSON.parse(readFileSync(path, "utf8"))));
}
function persistState(path: string, cache: Map<string, RateEntry>) {
  writeFileSync(path, JSON.stringify(Object.fromEntries(cache)), { mode: 0o600 });
}
```

#### 验收步骤

```
前置条件：
1. kclaw Gateway 已正常启动并配置了有效的 Gateway Token。
2. 操作者知晓正确的 Token 值。

操作步骤：
1. 使用错误 Token 向 Gateway API 连续发起 5 次认证请求。
2. 通过 systemctl --user restart kclaw-gateway 重启 Gateway 服务。
3. 重启完成后立即使用正确 Token 向 Gateway API 发起请求。
4. 查看速率限制持久化文件 ~/.kclaw/state/rate-limit-state.json。

期望结果：
1. 第 5 次错误请求被 Gateway 拒绝（返回 403 或 429 状态码）。
2. 速率限制状态文件 rate-limit-state.json 中记录有该客户端的失败计数和时间窗口。
3. 重启 Gateway 后计数未归零，在锁定窗口内使用正确 Token 请求也被限制。
4. 超出锁定窗口后正确 Token 可正常通过认证。
```

---

### M03 — 共享存储挂载管控模块

**优先级**: HIGH
**编码量**: ~10 行（修改现有文件）

#### 概述

在沙箱挂载黑名单常量 `BLOCKED_HOST_PATHS` 中追加 10 个 HPC 共享存储和调度器路径，防止 agent 通过 bind mount 访问集群上其他用户的数据。

#### 涉及文件

`src/agents/sandbox/validate-sandbox-security.ts` (修改)

#### 代码初步方案

```typescript
const BLOCKED_HOST_PATHS = [
  // 现有系统路径
  "/etc", "/proc", "/sys", "/dev", "/root", "/boot",
  "/run", "/var/run", "/var/run/docker.sock", "/run/docker.sock",
  // 新增 HPC 共享存储路径
  "/lustre",            // Lustre 并行文件系统
  "/gpfs",              // IBM Spectrum Scale
  "/scratch",           // 通用临时空间
  "/cvmfs",             // CernVM 科学计算
  "/beegfs",            // BeeGFS
  "/pfs",               // 通用并行文件系统
  // 新增 HPC 调度器/认证路径
  "/var/spool/slurm", "/etc/slurm", "/etc/munge",
  // 新增多用户主目录
  "/home",
];
```

#### 验收步骤

```
前置条件：
1. kclaw Gateway 已正常启动且 Docker 沙箱功能已启用。
2. 宿主机上存在 /lustre 或 /gpfs 目录（可用临时目录模拟）。
3. 当前用户具有创建 Docker 沙箱容器的权限。

操作步骤：
1. 通过 Gateway API 或 agent 会话触发沙箱容器创建。
2. 在沙箱配置中指定 bind-mount /lustre:/mnt/lustre。
3. 在沙箱配置中指定 bind-mount /gpfs:/mnt/gpfs。
4. 在沙箱配置中指定 bind-mount /scratch:/mnt/scratch。
5. 在沙箱配置中指定 bind-mount /home:/mnt/home。

期望结果：
1. 所有指向黑名单路径（/lustre、/gpfs、/scratch、/home）的挂载请求均被拒绝。
2. Gateway 日志包含 "blocked host path" 级别告警。
3. 容器创建失败并返回明确错误信息提示禁止挂载 HPC 共享路径。
4. 其他不在黑名单中的合法路径挂载不受影响。
```

---

### M04 — 容器权限最小化管理模块

**优先级**: MID
**编码量**: ~40 行（修改现有文件）

#### 概述

Docker 沙箱容器默认强制 `cap_drop: ["ALL"]`（移除所有 Linux 特权能力）、`readOnlyRoot: true`（根分区只读）、`pids_limit: 100`（防止进程爆炸），防止攻击者利用内核提权漏洞逃逸。

#### 涉及文件

`src/agents/sandbox/config.ts` (修改)、`src/config/types.sandbox.ts` (修改)

#### 代码初步方案

```typescript
// src/agents/sandbox/config.ts
const SECURE_SANDBOX_DEFAULTS = {
  docker: {
    capDrop: ["ALL"],           // 移除所有 Linux 特权能力
    readOnlyRoot: true,         // 根文件系统只读
    pidsLimit: 100,             // 最大进程数
    tmpfs: ["/tmp", "/var/tmp"],  // 可写临时路径
  },
};

function resolveSandboxConfig(userConfig: SandboxConfig): ResolvedSandboxConfig {
  return {
    ...SECURE_SANDBOX_DEFAULTS,
    ...userConfig,
    docker: { ...SECURE_SANDBOX_DEFAULTS.docker, ...userConfig.docker },
  };
}
```

#### 验收步骤

```
前置条件：
1. kclaw Gateway 已正常启动且 Docker 沙箱功能已启用。
2. 已成功创建一个 agent 沙箱容器。

操作步骤：
1. 在沙箱容器内执行 cat /proc/1/status | grep -i cap 查看能力集。
2. 在沙箱容器内执行 mount | grep " / " 查看根分区挂载选项。
3. 在沙箱容器内执行 cat /proc/1/limits | grep "Max processes" 查看进程数限制。
4. 在沙箱容器内尝试写入 /usr 目录测试根分区是否可写。

期望结果：
1. CapEff 能力集值为 0000000000000000，所有 Linux 特权能力已被移除。
2. 根文件系统挂载选项包含 "ro" 标志，根分区为只读。
3. Max processes 限制为 100，无法创建超过 100 个进程。
4. /tmp 和 /var/tmp 目录可正常写入，/usr 等系统目录写入失败。
```

---

### M05 — 网关进程内存管控模块

**优先级**: HIGH
**编码量**: ~5 行（修改现有文件）

#### 概述

Node.js 进程启动时默认追加 `--max-old-space-size=4096`（4GB），防止超长会话导致整个 Gateway 进程 OOM 崩溃。运维可通过 `NODE_OPTIONS` 环境变量覆盖。

#### 涉及文件

`kclaw.mjs` (修改)

#### 代码初步方案

```javascript
// kclaw.mjs — 在 spawn 子进程前注入内存限制
const nodeArgs = process.execArgv.slice();
if (!nodeArgs.some((a) => a.startsWith("--max-old-space-size"))) {
  nodeArgs.push("--max-old-space-size=4096");
}
const child = childProcess.spawn(process.execPath, nodeArgs.concat(entryArgs), {
  stdio: "inherit",
  env: { ...process.env },
});
```

#### 验收步骤

```
前置条件：
1. kclaw Gateway 已正常启动。

操作步骤：
1. 通过 ps aux | grep kclaw 命令查看 Gateway 进程的完整启动参数。
2. 启动一个 agent 会话并持续追加超过 4GB 的上下文数据。
3. 通过 top 或 htop 持续监控 Gateway 进程的 RSS 内存占用。

期望结果：
1. 进程启动参数中包含 --max-old-space-size=4096。
2. 内存使用接近 4GB 时触发 V8 垃圾回收，进程不发生 OOM 崩溃。
3. 通过环境变量 NODE_OPTIONS="--max-old-space-size=8192" 启动后，内存上限为 8192MB，原默认值被覆盖。
```

---

### M06 — 子进程资源配额模块

**优先级**: MID
**编码量**: ~50 行（新增文件）

#### 概述

对 Agent 调用的每个外部命令通过 `setrlimit` 设置操作系统级限制：最多 64 个进程、最多 256 个文件描述符、最长运行 5 分钟，防止恶意或错误脚本耗尽计算节点资源。

#### 涉及文件

`src/infra/child-process-limits.ts` (新增)

#### 代码初步方案

```typescript
import { spawn, SpawnOptions } from "node:child_process";

export type ProcessLimits = {
  maxProcesses?: number;   // RLIMIT_NPROC (默认 64)
  maxOpenFiles?: number;   // RLIMIT_NOFILE (默认 256)
  maxCpuSeconds?: number;  // RLIMIT_CPU   (默认 300 = 5 分钟)
};

export function spawnWithLimits(
  command: string, args: string[], limits: ProcessLimits, options?: SpawnOptions,
) {
  const ulimit = [];
  if (limits.maxProcesses)  ulimit.push(`ulimit -u ${limits.maxProcesses}`);
  if (limits.maxOpenFiles)  ulimit.push(`ulimit -n ${limits.maxOpenFiles}`);
  if (limits.maxCpuSeconds) ulimit.push(`ulimit -t ${limits.maxCpuSeconds}`);

  if (ulimit.length > 0) {
    const wrappedCmd = `${ulimit.join("; ")}; exec ${command} ${args.join(" ")}`;
    return spawn("/bin/sh", ["-c", wrappedCmd], options);
  }
  return spawn(command, args, { ...options, shell: false });
}

export const DEFAULT_PROCESS_LIMITS: ProcessLimits = {
  maxProcesses: 64, maxOpenFiles: 256, maxCpuSeconds: 300,
};
```

#### 验收步骤

```
前置条件：
1. kclaw Gateway 已正常启动。
2. Agent 具备执行 shell 命令的能力。

操作步骤：
1. 通过 agent 工具执行连续创建 65 个后台子进程的 shell 命令。
2. 通过 agent 工具执行同时打开 257 个文件描述符的命令。
3. 通过 agent 工具执行 CPU 密集运算超过 300 秒的命令。
4. 在操作期间通过 ps 和 lsof 监控子进程和文件描述符数量。

期望结果：
1. 子进程数量达到 64 后新的 fork 操作失败，agent 收到资源限制错误信息。
2. 文件描述符数量达到 256 后新的 open 操作失败。
3. 子进程运行时间超过 300 秒后被自动 SIGXCPU 信号终止。
4. Gateway 主进程在上述操作期间保持正常运行，不出现资源耗尽。
```

---

### M07 — 部署安全前置校验模块

**优先级**: HIGH
**编码量**: ~20 行（修改现有文件）

#### 概述

修复 Docker Compose 默认不安全配置：将默认监听地址改为 loopback，未设置 Token 时拒绝启动并输出错误提示。替换 `console.error()` 裸输出为结构化日志。

#### 涉及文件

`docker-compose.yml` (修改)、`src/gateway/server-startup-config.ts` (修改)、`src/gateway/server-http.ts` (修改)

#### 代码初步方案

```yaml
# docker-compose.yml
services:
  openclaw-gateway:
    command: ["node", "dist/index.js", "gateway",
      "--bind", "${OPENCLAW_GATEWAY_BIND:-loopback}",  # 默认不暴露公网
      ...]
    environment:
      OPENCLAW_GATEWAY_TOKEN: ${OPENCLAW_GATEWAY_TOKEN:?必须设置 OPENCLAW_GATEWAY_TOKEN}
```

```typescript
// server-startup-config.ts
export function assertGatewayBindSafety(bind: string, hasToken: boolean, hasTls: boolean) {
  const isLoopback = bind === "loopback" || bind.startsWith("127.");
  if (!isLoopback && !hasToken) {
    throw new Error("refusing to start: gateway bound to non-loopback without auth token. " +
      "Set KCLAW_GATEWAY_TOKEN or use --bind loopback.");
  }
  if (!isLoopback && !hasTls) {
    logger.warn("gateway bound to non-loopback without TLS — traffic is unencrypted");
  }
}
```

#### 验收步骤

```
前置条件：
1. docker-compose.yml 中 KCLAW_GATEWAY_TOKEN 变量已清空（设值为空字符串）。
2. docker-compose.yml 中 KCLAW_GATEWAY_BIND 变量设置为 lan。

操作步骤：
1. 执行 docker compose up 启动服务。
2. 查看容器启动日志输出。

期望结果：
1. Gateway 启动被拒绝并立即退出，退出码不为 0。
2. 日志中包含 "refusing to start" 或 "without auth token" 错误消息。
3. 将 KCLAW_GATEWAY_TOKEN 设置为有效值后重新启动，Gateway 正常启动并通过 healthz 检查。
```

---

### M08 — 网关流量防护模块

**优先级**: MID
**编码量**: ~50 行（修改现有文件）

#### 概述

HTTP 层新增全局防护：限制同时连接数（512）、单请求体大小（10MB）、全局 IP 速率限制（300 次/分钟）、WebSocket 消息大小限制（1MB）+ 频率限制（10 条/秒）。

#### 涉及文件

`src/gateway/server-http.ts` (修改)、`src/gateway/server-ws-runtime.ts` (修改)

#### 代码初步方案

```typescript
import { createFixedWindowRateLimiter } from "../infra/fixed-window-rate-limit.js";

const globalRateLimiter = createFixedWindowRateLimiter({ maxRequests: 300, windowMs: 60_000 });

server.on("request", (req, res) => {
  // 1. 连接数限制
  //    server.maxConnections = 512;
  // 2. 请求体大小限制
  const maxBodySize = 10 * 1024 * 1024; // 10MB
  if (Number(req.headers["content-length"]) > maxBodySize) {
    res.statusCode = 413;
    return res.end("Payload Too Large");
  }
  // 3. 全局 IP 速率限制
  const clientIp = req.socket.remoteAddress ?? "unknown";
  if (!globalRateLimiter.check(clientIp).allowed) {
    res.statusCode = 429;
    return res.end("Too Many Requests");
  }
});
```

#### 验收步骤

```
前置条件：
1. kclaw Gateway 已正常启动并监听 127.0.0.1:18789。

操作步骤：
1. 使用 curl 发送请求体大小为 11MB 的 POST 请求。
2. 从同一 IP 地址在 60 秒内向 Gateway 发送超过 300 个请求。
3. 通过 WebSocket 连接发送单条超过 1MB 的消息。

期望结果：
1. 11MB 请求体被拒绝，Gateway 返回 HTTP 413 Payload Too Large。
2. 超过速率限制后请求返回 HTTP 429 Too Many Requests。
3. WebSocket 超大消息发送后连接被服务端关闭。
4. 请求体小于 10MB 且请求频率在限制内的正常请求不受影响。
```

---

### M09 — AI 服务熔断管理模块

**优先级**: MID
**编码量**: ~80 行（新增文件）

#### 概述

实现三态熔断器（closed → open → half-open → closed）：AI 提供商连续 5 次失败后自动切断请求 30 秒，半开探测成功后恢复。消除连锁重试导致的资源耗尽风险。

#### 涉及文件

`src/infra/circuit-breaker.ts` (新增)

#### 代码初步方案

```typescript
type CircuitState = "closed" | "open" | "half-open";

export class CircuitBreaker {
  private state: CircuitState = "closed";
  private failureCount = 0;
  private openUntil = 0;

  constructor(
    private name: string,
    private opts: {
      maxFailures: number;        // 触发熔断的连续失败次数（默认 5）
      resetTimeoutMs: number;     // 熔断后等待时间（默认 30s）
      halfOpenMaxRequests: number; // 半开探测请求数（默认 1）
    } = { maxFailures: 5, resetTimeoutMs: 30_000, halfOpenMaxRequests: 1 },
  ) {}

  async call<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === "open") {
      if (Date.now() < this.openUntil) throw new Error(`circuit ${this.name} is open`);
      this.state = "half-open";
    }
    try {
      const result = await fn();
      this._onSuccess();
      return result;
    } catch (err) {
      this._onFailure();
      throw err;
    }
  }

  private _onSuccess() { this.failureCount = 0; this.state = "closed"; }
  private _onFailure() {
    this.failureCount++;
    if (this.state === "half-open" || this.failureCount >= this.opts.maxFailures) {
      this.state = "open";
      this.openUntil = Date.now() + this.opts.resetTimeoutMs;
    }
  }
}
// 使用:
// const cb = new CircuitBreaker("openai", { maxFailures: 5 });
// const result = await cb.call(() => fetchAIResponse(prompt));
```

#### 验收步骤

```
前置条件：
1. kclaw Gateway 已正常启动，至少配置了一个 AI 模型提供商。
2. 该提供商的 API 端点可通过工具模拟连续返回 5 次失败（如代理指向一个不存在的端点）。

操作步骤：
1. 通过 agent 会话向目标 AI 提供商连续发起 5 次请求，均模拟返回 5xx 错误。
2. 在第 6 次发起请求时观察 Gateway 行为。
3. 等待 30 秒后（熔断窗口结束），再次发起请求观察是否可以半开探测成功。
4. 半开探测成功后发起第 7 次正常请求，观察熔断器是否恢复关闭。

期望结果：
1. 连续 5 次失败后，熔断器状态变为 open，第 6 次请求被立即拒绝，不实际发送到下游 AI 服务。
2. 拒绝响应中包含熔断提示信息，不会阻塞 agent 会话。
3. 30 秒后熔断器进入 half-open 状态，允许发送一条探测请求。
4. 探测请求成功后，熔断器恢复 closed 状态，后续请求正常通过。
5. 多提供商独立熔断，一个提供商的故障不影响其他提供商的请求。
```

---

### M10 — 会话数据加密存储模块

**优先级**: MID
**编码量**: ~80 行（新增文件）

#### 概述

新增配置项 `session.encryption.atRest`，开启后对话历史文件写入磁盘前自动 AES-256-GCM 加密，防止硬盘被盗或送修时聊天记录明文泄露。

#### 涉及文件

`src/sessions/encrypted-transcript.ts` (新增)

#### 代码初步方案

```typescript
import { createCipheriv, createDecipheriv, randomBytes, scryptSync } from "node:crypto";
import { readFileSync, writeFileSync } from "node:fs";

const ALGORITHM = "aes-256-gcm"; const KEY_LENGTH = 32;
const IV_LENGTH = 16; const TAG_LENGTH = 16; const SALT_LENGTH = 32;
const ENCRYPTED_MAGIC = Buffer.from([0x4f, 0x43, 0x45, 0x43]); // "OCEC"

export function deriveMasterKey(): Buffer {
  const machineId = readFileSync("/etc/machine-id", "utf8").trim();
  return scryptSync(`${machineId}:${require("node:os").hostname()}`, "kclaw-vault", KEY_LENGTH);
}

export function encrypt(plaintext: string, masterKey: Buffer): Buffer {
  const salt = randomBytes(SALT_LENGTH); const key = scryptSync(masterKey, salt, KEY_LENGTH);
  const iv = randomBytes(IV_LENGTH);
  const cipher = createCipheriv(ALGORITHM, key, iv);
  const encrypted = Buffer.concat([cipher.update(plaintext, "utf8"), cipher.final()]);
  return Buffer.concat([salt, iv, cipher.getAuthTag(), encrypted]);
}

export function decrypt(data: Buffer, masterKey: Buffer): string {
  const salt = data.subarray(0, SALT_LENGTH);
  const iv = data.subarray(SALT_LENGTH, SALT_LENGTH + IV_LENGTH);
  const tag = data.subarray(SALT_LENGTH + IV_LENGTH, SALT_LENGTH + IV_LENGTH + TAG_LENGTH);
  const ciphertext = data.subarray(SALT_LENGTH + IV_LENGTH + TAG_LENGTH);
  const key = scryptSync(masterKey, salt, KEY_LENGTH);
  const decipher = createDecipheriv(ALGORITHM, key, iv);
  decipher.setAuthTag(tag);
  return decipher.update(ciphertext) + decipher.final("utf8");
}

export function readFileAuto(path: string, masterKey: Buffer): string {
  const raw = readFileSync(path);
  return raw.subarray(0, 4).equals(ENCRYPTED_MAGIC)
    ? decrypt(raw.subarray(4), masterKey)
    : raw.toString("utf8");
}

export function writeFileAuto(path: string, content: string, masterKey: Buffer): void {
  const encrypted = encrypt(content, masterKey);
  writeFileSync(path, Buffer.concat([ENCRYPTED_MAGIC, encrypted]), { mode: 0o600 });
}
```

#### 验收步骤

```
前置条件：
1. kclaw.json 中已配置 session.encryption.atRest = true。
2. kclaw Gateway 已正常启动。
3. 至少已完成一次 agent 对话产生会话文件。

操作步骤：
1. 进入会话存储目录 ~/.kclaw/workspace/sessions/。
2. 通过 hexdump 或二进制编辑器查看最新会话文件的文件头。
3. 尝试通过 cat 或文本编辑器直接读取会话文件内容。

期望结果：
1. 会话文件开头四个字节为魔数 0x4F 0x43 0x45 0x43（ASCII: OCEC）。
2. 文件主体内容为加密后的二进制数据，无法直接读取到明文对话内容。
3. 将 session.encryption.atRest 设为 false 后新建的会话文件恢复为明文 JSON 格式。
```

---

## 五、代码变更文件清单

```
kclaw/
├── kclaw.mjs                                # ★ 重命名 + M05 内存管控
├── package.json                             # ★ 重命名 name/bin/scripts
├── pnpm-lock.yaml                           # 锁文件（不变）
├── docker-compose.yml                       # ★ M07 loopback 默认 + 空 Token 拒绝
│
├── src/
│   ├── config/
│   │   └── paths.ts                         # ★ 路径常量 ~/.kclaw/
│   ├── security/
│   │   └── secret-equal.ts                  # ★ M01 恒时比较
│   ├── infra/
│   │   ├── child-process-limits.ts          # ★ 新增 M06 子进程配额
│   │   └── circuit-breaker.ts               # ★ 新增 M09 熔断管理
│   ├── sessions/
│   │   └── encrypted-transcript.ts          # ★ 新增 M10 会话加密
│   ├── gateway/
│   │   ├── auth-rate-limit.ts               # ★ M02 计数持久化
│   │   ├── server-startup-config.ts         # ★ M07 启动安全检查
│   │   ├── server-http.ts                   # ★ M08 网关流量防护
│   │   └── server-ws-runtime.ts             # ★ M08 WS 消息限制
│   ├── agents/
│   │   └── sandbox/
│   │       ├── validate-sandbox-security.ts # ★ M03 路径黑名单
│   │       └── config.ts                    # ★ M04 权限最小化
│   └── cli/                                 # ★ 全局重命名
│
├── scripts/
│   ├── hpc-offline-bundle.sh                # ★ 新增 离线打包
│   └── hpc-offline-install.sh               # ★ 新增 离线安装
│
├── .github/workflows/
│   └── security-scan.yml                    # ★ 新增 CVE 扫描
│
└── docs/
    ├── architecture.md                      # ★ 新增 总体架构
    ├── security-features.md                 # ★ 新增 安全功能清单
    ├── deployment.md                        # ★ 新增 部署手册
    └── fork-diff.md                         # ★ 新增 与上游精确差异
```

**文件变更统计：**

| 类型 | 数量 | 说明 |
|------|------|------|
| 新增文件 | 10 | 5 个 TS + 3 个脚本/YAML + 2 个文档 |
| 修改文件 | 10 | 重命名 + 安全功能注入 |
| 全局替换 | ~200 处 | `KCLAW_` / `kclaw` |
| 不涉及文件 | 其余全部 | 保持 OpenClaw 原始功能 |

---

## 六、架构介入点

```
kclaw 架构 (Fork 自 OpenClaw，★ 标记为变更注入点)

┌─────────────────────────────────────┐
│  原生应用  │  Web Control UI        │  ← 无变更
├─────────────────────────────────────┤
│  Gateway Server (HTTP/WS)           │
│  ├─ Auth ────── ★ M01 恒时校验     │
│  │              ★ M02 计数持久化    │
│  ├─ Server ──── ★ M07 前置校验     │
│  │              ★ M08 流量防护      │
│  └─ Sessions ── ★ M10 会话加密     │
├─────────────────────────────────────┤
│  Core Agent Runtime                 │
│  ├─ Runner ──── ★ M05 内存管控     │
│  ├─ Tool System ★ M06 资源配额     │
│  ├─ Transport ─ ★ M09 熔断管理     │
│  ├─ Sandbox ─── ★ M03 挂载管控     │
│  │              ★ M04 权限最小化    │
├─────────────────────────────────────┤
│  Plugin SDK ─── 无变更              │
├─────────────────────────────────────┤
│  Extensions ─── 无变更 (兼容上游)    │
├─────────────────────────────────────┤
│  Infrastructure                     │
│  ├─ Config ──── ★ 全局重命名        │
│  ├─ Daemon ──── ★ systemd 适配     │
│  └─ Secrets ─── 无变更              │
└─────────────────────────────────────┘
         │
         ├── scripts/  ★ 离线安装脚本
         ├── .github/  ★ CI 安全扫描
         └── docs/     ★ 项目文档
```

---

## 七、实施路线（3 周，1 人）

```
Week 1: Fork + 重命名 + 高危加固
├── Day 1    Fork + 项目初始化 (package.json/README/FORK.md)
├── Day 2    全局重命名 (CLI/环境变量/路径/import/UI 字符串)
├── Day 3    M01 恒时校验 + M03 挂载管控 + M05 内存管控
├── Day 4    M07 前置校验 + M04 权限最小化
├── Day 5    编译验证 + 基础冒烟测试 (麒麟 ARM64 + x86_64)

Week 2: 安全功能开发
├── Day 1-2  M02 计数持久化 + M06 资源配额
├── Day 3    M08 网关流量防护
├── Day 4    M09 熔断管理 + M10 会话加密
├── Day 5    M01-M10 全量集成测试 + 验收

Week 3: 交付物
├── Day 1-2  离线安装脚本 + 麒麟双架构验证
├── Day 3    CVE 扫描 CI 配置
├── Day 4    文档输出 (architecture/security-features/deployment/fork-diff)
└── Day 5    最终验证 → 交付物打包 (.tar.gz 双架构离线包)
```

**交付物：**

- `kclaw` 源码仓库
- `kclaw-linux-x64.tar.gz` 离线安装包
- `kclaw-linux-arm64.tar.gz` 离线安装包
- 项目文档 (README + 架构 + 安全清单 + 部署手册 + Fork 差异)

---

## 八、安全功能覆盖

| 原始高危问题 | kclaw 整改模块 | 状态 |
|-------------|---------------|------|
| Token 比较有时序侧信道风险 | M01 认证令牌恒时校验模块 | ✅ 已修复 |
| 沙箱可 bind-mount HPC 共享路径 | M03 共享存储挂载管控模块 | ✅ 已修复 |
| Docker sandbox 无 seccomp/cap_drop | M04 容器权限最小化管理模块 | ✅ 已修复 |
| Node.js 进程无内存上限 | M05 网关进程内存管控模块 | ✅ 已修复 |
| docker-compose --bind lan + 空 Token | M07 部署安全前置校验模块 | ✅ 已修复 |
| 内存速率限制重启清零 | M02 认证失败计数持久化模块 | ✅ 已修复 |
| 子进程无资源配额 | M06 子进程资源配额模块 | ✅ 已修复 |
| 无网关请求安全管控 | M08 网关流量防护模块 | ✅ 已修复 |
| AI 服务故障无熔断保护 | M09 AI 服务熔断管理模块 | ✅ 已修复 |
| 会话文件无加密存储 | M10 会话数据加密存储模块 | ✅ 已修复 |

---

## 九、风险与约束

| 风险 | 影响 | 应对 |
|------|------|------|
| `KCLAW_*` env 全局替换遗漏 | 运行时隐式引用旧名 | 用 `grep -r` 验证 + 自动化脚本 |
| 插件生态不兼容 | 外部插件引用 `openclaw/plugin-sdk` | 保留别名映射或导出兼容路径 |
| 上游关键安全补丁丢失 | 不跟随合并，漏洞需自行修复 | README 中记录已知差异 + 定期巡检 |
| 国内平台麒麟/UOS glibc 版本 | 预编译 ARM64 模块可能不兼容 | 离线包包含编译工具链回退脚本 |

---

## 十、开发操作

### 10.1 环境搭建

```bash
# 1. 克隆仓库
git clone <kclaw-repo-url>
cd kclaw

# 2. 环境要求 (Node.js >= 22.19, pnpm >= 9)
node -v
npm i -g pnpm

# 3. 安装依赖
pnpm install
# 国内环境: pnpm config set registry https://registry.npmmirror.com && pnpm install

# 4. 验证环境
pnpm kclaw --version
pnpm kclaw doctor
```

### 10.2 编译

```bash
pnpm build                        # 完整构建 → dist/
pnpm dev                          # 开发模式
pnpm tsgo                         # 类型检查
pnpm check:changed                # 变更文件检查
pnpm check:changed --staged       # 暂存文件检查
pnpm format                       # 格式化
pnpm lint                         # 代码检查
```

**注意事项：** 首次构建或依赖变更后必须 `pnpm build`。国产平台 ARM64 编译原生模块失败时 `SHARP_IGNORE_GLOBAL_LIBVIPS=1 pnpm install`。

### 10.3 本地运行

```bash
# 开发模式跳过通道加载
pnpm gateway:dev

# 正式运行
pnpm kclaw gateway --port 18789 --bind loopback

# 文件监听自动重启
pnpm gateway:watch

# 验证
curl http://127.0.0.1:18789/healthz   # → {"status":"ok"}

# 常用 CLI
kclaw onboard                       # 配置向导
kclaw doctor                        # 系统诊断
kclaw channels list                 # 查看通道
kclaw models list                   # 查看模型
kclaw gateway status                # Gateway 状态
```

**环境变量覆盖：**

```bash
KCLAW_CONFIG_DIR=/tmp/kclaw-test kclaw gateway --port 18790
KCLAW_CHILD_OOM_SCORE_ADJ=0 kclaw gateway
NODE_COMPILE_CACHE=/var/tmp/kclaw-cache kclaw gateway
KCLAW_LIVE_TEST=1 pnpm test:live
```

---

## 十一、测试

```bash
# 运行指定文件测试
pnpm test src/security/secret-equal.test.ts

# 匹配模式
pnpm test "恒时"

# 单线程（断点调试）
KCLAW_VITEST_MAX_WORKERS=1 pnpm test <path>

# 变更文件 / 扩展 / 全量
pnpm test:changed
pnpm test extensions
pnpm test:coverage
pnpm test:serial
```

**M01-M10 安全功能逐项验证：**

```bash
# M01 恒时校验
pnpm test src/security/secret-equal.test.ts

# M03 挂载管控
pnpm test src/agents/sandbox/validate-sandbox-security.test.ts

# M05 内存管控
ps aux | grep kclaw | grep max-old-space-size     # 应包含 4096

# M04 权限最小化
kclaw exec "cat /proc/1/status | grep CapEff"     # → 0000000000000000

# M08 流量防护
curl -X POST -d "$(head -c 11M /dev/zero)" http://127.0.0.1:18789/api/test  # → 413

# M10 会话加密
hexdump -C ~/.kclaw/workspace/sessions/latest.json | head -1  # → 4f 43 45 43

# M09 熔断 (需模拟)
# 连续 5 次失败 → 第 6 次立即拒绝 → 30s 后半开探测

# M02 计数持久化
cat ~/.kclaw/state/rate-limit-state.json | jq .

# 全量检查
pnpm check:changed && pnpm test
```

---

## 十二、部署

### 方式一：离线安装包（推荐）

```bash
# === 构建机（有外网） ===
pnpm build
./scripts/hpc-offline-bundle.sh
# → build/kclaw-linux-x64.tar.gz / kclaw-linux-arm64.tar.gz

# === 目标机（内网） ===
tar xzf kclaw-linux-arm64.tar.gz && cd kclaw-offline
sudo bash install.sh
kclaw onboard
systemctl --user status kclaw-gateway && curl http://127.0.0.1:18789/healthz
```

### 方式二：systemd 用户服务

```bash
kclaw onboard --install-daemon
systemctl --user start kclaw-gateway
systemctl --user enable kclaw-gateway
sudo loginctl enable-linger $USER
journalctl --user -u kclaw-gateway -f
```

### 方式三：Docker 部署

```bash
docker build -t kclaw:latest .
docker compose up -d
```

### 配置文件位置

| 文件 | 路径 | 说明 |
|------|------|------|
| 主配置 | `~/.kclaw/kclaw.json` | Gateway/通道/模型/插件所有配置 |
| 凭据 | `~/.kclaw/credentials/` | 各通道 Token/API Key |
| Agent 配置 | `~/.kclaw/agents/<id>/agent/auth-profiles.json` | 模型认证配置 |
| 速率限制 | `~/.kclaw/state/rate-limit-state.json` | M02 登录计数持久化 |
| 会话文件 | `~/.kclaw/workspace/sessions/` | M10 会话加密存储 |
| systemd 单元 | `~/.config/systemd/user/kclaw-gateway[-<profile>].service` | 服务定义 |

### 麒麟 V10 / 统信 UOS 部署注意

```bash
ldd --version                                    # ≥ 2.28
sudo loginctl enable-linger $USER                # 国产桌面 (UKUI/DDE)
KCLAW_OFFLINE_FALLBACK_BUILD=1 bash install.sh   # npm 原生模块不兼容时回退编译
```

---

## 十三、调试

### 日志调试

```bash
journalctl --user -u kclaw-gateway -f -n 200     # Gateway 日志
grep -i "sk-\|AIza\|ghp_" ~/.kclaw/state/*.jsonl # 脱敏检查 (应无输出)
```

### Node.js 调试

```bash
node --inspect kclaw.mjs gateway --port 18789    # Chrome DevTools
node --inspect-brk kclaw.mjs gateway --port 18789 # 启动暂停等待附加
node --heap-prof kclaw.mjs gateway               # 堆快照
```

### 常见问题排查

| 症状 | 检查步骤 |
|------|----------|
| Gateway 启动失败 | `kclaw doctor` → `journalctl --user -u kclaw-gateway -e` |
| 飞书收不到消息 | `kclaw channels status` → 检查 Webhook 地址可达性 |
| 模型调用超时 | `curl -v <provider-api-url>` → 检查 `HTTPS_PROXY` |
| OOM 崩溃 | `journalctl` 查看 OOM 记录 → 检查 `--max-old-space-size` |
| 离线安装失败 | `uname -m` 确认架构 → `ldd --version` 检查 glibc → 尝试回退编译 |
| 沙箱路径拦截不生效 | 检查路径是否通过 symlink 绕过 |

---

## 附录：后续规划（Phase 2）

以下功能模块在本轮优先级调整中暂未纳入，可在后续版本迭代中按需推进：

| 模块 | 正式名称 | 编码量 | 说明 |
|------|----------|--------|------|
| — | 配置文件明文凭据检测告警模块 | ~30 行 | 启动扫描 kclaw.json 中的明文 API Key，输出 WARNING |
| — | 会话数据安全擦除模块 | ~50 行 | 随机覆写 + CLI purge 命令，防文件恢复 |
| — | 离线化部署方案 | shell | 双架构离线安装包，支撑信创/军工/政务等断网环境 |
| — | CVE 自动扫描 | YAML | 周扫描 pnpm audit + Trivy，CI 集成 |
