# kclaw — OpenClaw 工业安全加固衍生版 项目规划

## 一、项目定义

| 维度         | 说明                                                                        |
| ------------ | --------------------------------------------------------------------------- |
| **项目名称** | kclaw                                                                       |
| **代码来源** | Fork OpenClaw (`https://github.com/openclaw/openclaw`) 当前最新 `main` 分支 |
| **维护模式** | 独立演进，不跟随上游合并                                                    |
| **定位**     | 面向工业/HPC 场景的出厂安全加固版本，内置 10 项安全功能模块                 |
| **发布形式** | 双架构离线安装包 (linux-x64 + linux-arm64)，预置飞书通道 + 国产模型         |
| **CLI**      | `kclaw`，配置目录 `~/.kclaw/`，环境变量前缀 `KCLAW_*`                       |

---

## 二、Fork 策略

```
openclaw/main ──Fork──> kclaw/main
                            │
                            ├── commit 1: 项目初始化 (package.json/README/FORK.md)
                            ├── commit 2: 全局重命名 (openclaw → kclaw)
                            ├── commit 3: 环境与网络安全加固 (M01 环境变量 + M02 网络白名单 + M03 Exec路径)
                            ├── commit 4: 合规与审计加固 (M04 凭据检测 + M05 完整性校验 + M06 会话隔离)
                            ├── commit 5: 部署与运行加固 (M07 安全响应头 + M08 限速持久化)
                            ├── commit 6: 数据安全加固 (M09 熔断管理 + M10 会话加密)
                            ├── commit 8: 国产化适配 (离线安装脚本)
                            ├── commit 9: CI 安全扫描 (CVE 扫描)
                            └── commit 10: 文档 + 配置模板
```

**不保留** `git remote add upstream` —— 独立演进，避免上游变更引入兼容性问题。

---

## 三、全局重命名清单

| 原值                             | 新值                    | 涉及位置                         | 方式     |
| -------------------------------- | ----------------------- | -------------------------------- | -------- |
| `openclaw` CLI 命令              | `kclaw`                 | `package.json` `bin` 字段        | 修改     |
| `openclaw.mjs` 入口              | `kclaw.mjs`             | 文件重命名 + `package.json` 引用 | 重命名   |
| `OPENCLAW_*` 环境变量            | `KCLAW_*`               | `src/` 全部 ~200 处引用          | 全局替换 |
| `~/.openclaw/` 配置目录          | `~/.kclaw/`             | `src/config/` 路径常量           | 修改     |
| `openclaw.json` 配置文件         | `kclaw.json`            | 默认配置文件名                   | 修改     |
| `openclaw-gateway.service`       | `kclaw-gateway.service` | systemd 单元                     | 修改     |
| `openclaw/plugin-sdk/*` 导入路径 | `kclaw/plugin-sdk/*`    | 全部插件 import 语句             | 全局替换 |
| `openclaw.plugin.json` 清单      | `kclaw.plugin.json`     | 清单文件名约定                   | 修改     |
| `"openclaw"` 配置块名称          | `"kclaw"`               | `kclaw.json` 顶层 key            | 修改     |
| 用户可见字符串 "OpenClaw"        | "kclaw"                 | CLI 帮助文本、日志、UI           | 全局替换 |

**不重命名的范围：**

- `extensions/` 目录内的 135+ 插件 — 保持与 OpenClaw 生态兼容
- `src/gateway/protocol/` — Gateway 协议版本号维持
- `CONTRIBUTORS`、`LICENSE` — 法律文件保留原样

---

## 四、功能模块清单（10 项）

### 模块总览

| 模块 | 正式名称                    | 优先级 | 编码量   | 对应缺陷                                                                                                                            | CWE      | 来源 |
| ---- | --------------------------- | ------ | -------- | ----------------------------------------------------------------------------------------------------------------------------------- | -------- | ---- |
| M01  | 环境变量强制白名单模块      | HIGH   | ~30 行改 | 默认黑名单模式，HPC 调度器凭据可泄露到 agent 子进程                                                                                 | CWE-526  | 审计 |
| M02  | 网络出站白名单控制模块      | HIGH   | ~40 行改 | 仅依赖 SSRF 黑名单，内网隔离可被绕过                                                                                                | CWE-923  | 审计 |
| M03  | Exec 命令路径越权防护模块   | HIGH   | ~35 行改 | 符号链接和路径遍历可绕过 exec 白名单路径限制                                                                                        | CWE-22   | 审计 |
| M04  | 明文凭据检测与告警模块      | MID    | ~30 行改 | 配置文件和启动参数中可能残留 API Key、Token 等明文凭据                                                                              | CWE-312  | 审计 |
| M05  | 配置文件完整性校验模块      | HIGH   | ~50 行改 | 配置文件可被篡改而无声告警，缺少密码学完整性验证                                                                                    | CWE-345  | 审计 |
| M06  | 会话上下文可见性硬隔离模块  | MID    | ~25 行改 | 多租户场景 agent 跨会话消息可泄露给无关方                                                                                           | CWE-200  | 审计 |
| M07  | HTTP 安全响应头强制加固模块 | HIGH   | ~20 行改 | 通用 HTTP 响应缺少 CSP/X-Frame-Options，HSTS 默认关闭，存在点击劫持和 MIME 嗅探风险                                                 | CWE-1021 | 审计 |
| M08  | 认证速率限制持久化模块      | HIGH   | ~60 行改 | 暴力破解计数器在内存中，重启清零可绕过锁定。攻击者可反复触发重启绕过限速，无限次尝试猜测认证令牌，一旦破解即获得 Gateway 完整控制权 | CWE-307  | 审计 |
| M09  | AI 服务熔断管理模块         | MID    | ~80 行新 | AI 服务故障时无熔断保护，连锁重试耗尽资源                                                                                           | CWE-754  | 审计 |
| M10  | 会话数据加密存储模块        | MID    | ~80 行新 | 对话历史明文存储，磁盘送修或报废时数据泄露                                                                                          | CWE-311  | 审计 |

> **来源说明**: 全部模块均来自 kclaw 独立安全审计，基于仓库中实际存在的代码进行加固或新增。原始计划中 3 条 GHSA 公告（ccwh/cwpp/gxg4）因对应代码路径在当前版本中不存在或已内置修复，不再作为独立模块。已移除的模块：M04 恒时校验（`secret-equal.ts` 已实现）、M05 挂载管控（用户确认移除）、M06 权限最小化（`config.ts` 已默认 `capDrop:["ALL"]`）、M06 资源配额（版本已内置）、M07 内存管控（功能单一）、M08 流量防护（版本已内置）、M08 部署前置校验（不适用离线安装包形式）。

**总计：~395 行 TS（新增 ~160 行 + 修改 ~235 行）**

### 缺陷→模块→工业场景落地链路

```
独立安全审计发现的 10 个缺陷
        │
        ▼
   M01-M10 10 个功能模块逐一修复
        │
    ┌────┼────┬────────────┬──────────┐
    ▼    ▼    ▼            ▼          ▼
   环境  网络  执行         合规       数据
   安全  安全  安全         校验       保护
  (M01  (M02  (M03        (M04      (M10)
   M07)  )     )       M05 M08)
                        M06)
        │
        ▼
  满足工业场景四大核心诉求:
  ① 多租户数据隔离 (M01/M06)
  ② 7×24 持续可用 (M08/M09)
  ③ 内网安全部署 (M02/M03/M08)
  ④ 数据保密合规 (M04/M05/M10)
```

### 行业推广适配性

| 行业场景            | 核心痛点                     | 对应模块        | 推广理由                                      |
| ------------------- | ---------------------------- | --------------- | --------------------------------------------- |
| 汽车制造 (广汽 HPC) | 多团队共用集群，CAE 数据敏感 | M01/M03/M06/M10 | 环境隔离 + 路径防护 + 会话隔离 + 数据加密     |
| 能源/电力           | 内网隔离，7×24 不可中断      | M08/M09         | 限速持久化 + 熔断保护，满足生产网安全要求     |
| 金融/政务           | 等保密评合规，防攻击         | M04/M05/M07/M08 | 凭据审计 + 完整性校验 + 安全响应头 + 认证加固 |
| 军工/航天           | 完全断网，数据绝密           | M02/M05/M10     | 网络管控 + 防篡改 + 存储加密，满足保密要求    |

---

### M01 — 环境变量强制白名单模块

**优先级**: HIGH
**编码量**: ~30 行（修改现有文件）
**来源**: 原始安全审计

#### 概述

当前 `src/infra/host-env-security.ts`（276 行）已实现环境变量黑名单过滤逻辑，但默认仍以黑名单模式运行——未被列入禁止清单的变量全部透传。改为白名单模式：仅允许明确声明的安全变量传入 agent 子进程，拒绝一切未声明变量。

#### 涉及文件

`src/infra/host-env-security.ts`（修改）、`src/infra/host-env-security-policy.d.ts`（修改）

#### 验收步骤

```
前置条件：
1. kclaw Gateway 已正常启动。
2. 环境变量白名单中已配置允许传递的安全变量（如 PATH、HOME、LANG）。
3. 宿主机环境中存在常见的系统环境变量。

操作步骤：
1. 启动 agent 会话，触发需要新建子进程的操作。
2. 检查子进程运行环境中实际可见的环境变量集合。
3. 尝试通过工具环境变量覆盖功能传入未在白名单中的变量。
4. 尝试传入白名单中已声明的合法变量作为对照。

预期结果：
1. 未在白名单中的变量不出现在子进程的运行环境中。
2. 白名单中明确允许的变量正常传递。
3. 工具环境变量覆盖功能只能传入白名单中已声明的变量，其他变量被静默丢弃。
```

---

### M02 — 网络出站白名单控制模块

**优先级**: HIGH
**编码量**: ~40 行（修改现有文件）
**来源**: 原始安全审计

#### 概述

当前 `src/infra/net/ssrf.ts`（733 行）已实现 DNS 锁定、CIDR 黑名单等 SSRF 防护机制。但在 HPC 内网或军工断网场景中，黑名单模式不够——需要可配置的出站白名单，仅允许 agent 连接指定的目标地址范围。新增配置项：`network.egress.allow`，支持 CIDR 和域名白名单，未在白名单中的目标一律拒绝。

#### 涉及文件

`src/infra/net/ssrf.ts`（修改）、`src/infra/net/fetch-guard.ts`（修改）、`src/config/zod-schema.ts`（修改）

#### 验收步骤

```
前置条件：
1. kclaw Gateway 已正常启动，已配置网络出站白名单（仅允许特定 CIDR 或域名）。
2. 能够模拟 agent 发起网络请求的场景。

操作步骤：
1. 通过 agent 向白名单中的地址发起网络请求。
2. 通过 agent 向白名单外的内网地址发起网络请求。
3. 通过 agent 向白名单外的外部公网地址发起网络请求。
4. 修改白名单配置并重新加载后重复上述测试。

预期结果：
1. 白名单中的目标地址请求正常完成。
2. 白名单外的地址请求被拒绝，返回明确的拒绝信息。
3. 配置变更重新加载后，白名单规则立即生效。
4. 不配置白名单时保持原有行为（仅执行 SSRF 黑名单检查），不改变向后兼容性。
```

---

### M03 — Exec 命令路径越权防护模块

**优先级**: HIGH
**编码量**: ~35 行（修改现有文件）
**来源**: 原始安全审计

#### 概述

当前 `src/infra/exec-allowlist-pattern.ts`（104 行）和 `src/infra/exec-safety.ts`（44 行）已实现基于文件系统真实路径的白名单匹配和输入安全过滤。但在 HPC 多租户场景中，agent 可能通过符号链接穿越到白名单路径之外，或通过路径遍历访问其他用户目录。需要增强：命令的真实路径必须落在可执行目录范围内，符号链接指向白名单外路径时拒绝执行，路径遍历（含 `..`）的命令被拒绝或规范化为有效路径后重新匹配。

#### 涉及文件

`src/infra/exec-allowlist-pattern.ts`（修改）、`src/infra/exec-safety.ts`（修改）、`src/agents/exec-defaults.ts`（修改）

#### 验收步骤

```
前置条件：
1. kclaw Gateway 已正常启动，已配置 exec 白名单，允许执行指定目录下的命令。
2. 系统中存在指向白名单路径外的符号链接。
3. 存在可通过路径遍历到达的受限目录。

操作步骤：
1. 通过 agent 以符号链接形式调用白名单中的命令（符号链接指向白名单外路径）。
2. 通过 agent 以包含路径遍历形式的命令调用白名单中的可执行文件。
3. 通过 agent 以绝对路径直接调用白名单中的命令作为对照。
4. 将工作目录设置为受限路径后发起 agent 命令执行。

预期结果：
1. 符号链接指向白名单外路径的命令调用被拒绝。
2. 路径遍历的命令调用被拒绝或修正后不解析到白名单外。
3. 不涉及路径越权的白名单命令调用正常执行。
4. 工作目录越界时命令执行被拒绝或沙箱拦截。
```

### M04 — 明文凭据检测与告警模块

**优先级**: MID
**编码量**: ~30 行（修改现有文件）
**来源**: 原始安全审计

#### 概述

当前 `src/secrets/audit.ts`（750 行）已实现凭据审计扫描功能，但扫描结果仅用于安全审计报告，不产生运行时告警。在工业部署场景中，配置文件或启动参数中可能残留 API Key、Access Token 等明文凭据——这些凭据可能在运维操作、配置同步、备份恢复过程中被无意写入。需要增强检测机制：在 Gateway 启动阶段强制扫描已知凭据模式，检测到明文凭据时输出告警并记录审计日志，必要时拒绝启动。

#### 涉及文件

`src/secrets/audit.ts`（修改）、`src/config/zod-schema.ts`（修改）

#### 验证标准

```
前置条件：
1. kclaw Gateway 已正常启动，凭据检测功能已开启。
2. 配置文件中手动插入若干已知格式的明文凭据作为测试数据。

操作步骤：
1. 启动 Gateway，观察启动日志中是否包含明文凭据检测告警。
2. 运行 kclaw doctor 安全审计，查看凭据检测结果。
3. 将检测策略从"告警"切换为"阻断"后重新启动。
4. 将配置文件中的明文凭据移除后重新启动。

预期结果：
1. Gateway 启动时检测到明文凭据，日志中包含明确告警信息（含位置和凭据类型）。
2. 安全审计报告中列出所有检测到的明文凭据风险项。
3. 阻断策略下 Gateway 拒绝启动，并提示需要移除明文凭据。
4. 凭据移除后不再产生告警，Gateway 正常启动。
```

---

### M05 — 配置文件完整性校验模块

**优先级**: HIGH
**编码量**: ~50 行（修改现有文件）
**来源**: 原始安全审计

#### 概述

当前配置管理已具备 SHA-256 哈希计算（`src/config/io.ts` 中 `hashConfigRaw()`）、原子写入、5 份滚动备份、异常检测（大小骤降、元数据丢失）和 `.last-good` 灾难恢复机制。但缺少密码学完整性验证——攻击者或错误程序可直接篡改 `kclaw.json` 或备份链而不会被检测。对于军工、金融等场景，需要"配置文件未被篡改"的可信证明。在启动时计算当前配置的 SHA-256 哈希并与已存储的可信基线比对；差异时根据策略拒绝启动或记录告警。备份链的 5 份 `.bak` 文件也纳入验证范围。

#### 涉及文件

`src/config/io.ts`（修改）、`src/config/backup-rotation.ts`（修改）、`src/config/zod-schema.ts`（修改）、`src/security/audit.ts`（修改）

#### 验证标准

```
前置条件：
1. 配置文件完整性校验功能已开启，基线哈希已存储。
2. kclaw Gateway 正在正常运行。

操作步骤：
1. Gateway 正常启动，验证启动日志中不包含完整性校验失败告警。
2. 停止 Gateway，在配置文件中做出非授权修改。
3. 重新启动 Gateway，观察启动结果。
4. 运行 kclaw doctor 安全审计，查看完整性检查结果。
5. 将配置文件恢复为正确内容后重新启动。

预期结果：
1. 未篡改时 Gateway 正常启动，无完整性告警。
2. 配置文件被篡改后，Gateway 根据策略行为：拒绝启动并给出明确错误信息，或在日志中输出 CRITICAL 级别告警。
3. 安全审计报告中明确指出配置文件与基线不一致的具体差异。
4. 配置恢复正确后 Gateway 正常启动，完整性状态恢复正常。
```

---

### M06 — 会话上下文可见性硬隔离模块

**优先级**: MID
**编码量**: ~25 行（修改现有文件）
**来源**: 原始安全审计

#### 概述

当前 `src/security/context-visibility.ts`（58 行）和 `src/sessions/send-policy.ts`（152 行）已实现会话上下文的可见性策略控制和跨会话消息发送策略。但在 HPC 多租户集群中，不同团队的 agent 共享同一 kclaw 实例——需要硬隔离模式：当开启后，不同 agent 之间的会话消息完全不可见，跨会话消息被强制拦截，任何会话上下文引用（历史、线程、引用消息）均被限制在当前 agent 范围内。新增配置项控制隔离级别（完全隔离 / 同 agent 共享 / 无限制）。

#### 涉及文件

`src/security/context-visibility.ts`（修改）、`src/sessions/send-policy.ts`（修改）、`src/config/zod-schema.ts`（修改）

#### 验证标准

```
前置条件：
1. kclaw Gateway 已正常启动，会话隔离功能已开启（设置为完全隔离模式）。
2. 已创建分别属于不同 agent 的两个活跃会话。

操作步骤：
1. 从 agent A 的会话中尝试引用 agent B 会话的消息。
2. 从 agent A 的会话中尝试向 agent B 的会话发送跨会话消息。
3. 从 agent A 的会话中查看 agent B 会话的历史记录。
4. 将隔离级别改为"同 agent 共享"后，验证属于同一 agent 的两个会话间消息可共享。
5. 将隔离级别改为"无限制"后，验证跨 agent 会话消息不再受限。

预期结果：
1. 完全隔离模式下，跨 agent 的消息引用被拒绝。
2. 完全隔离模式下，跨 agent 的会话消息发送被拦截。
3. 完全隔离模式下，agent A 无法查看 agent B 会话的历史记录。
4. 同 agent 共享模式下，同一 agent 的会话之间消息共享正常，跨 agent 仍被隔离。
5. 无限制模式下，所有限制解除。
```

### M07 — HTTP 安全响应头强制加固模块

**优先级**: HIGH
**编码量**: ~20 行（修改现有文件）
**来源**: 原始安全审计

#### 概述

当前 `src/gateway/http-common.ts` 已设置 `X-Content-Type-Options: nosniff`、`Referrer-Policy: no-referrer`、`Permissions-Policy` 三类安全头和可选的 HSTS。但 `Content-Security-Policy` 和 `X-Frame-Options` 仅覆盖 Control UI 页面，通用 HTTP 响应（API 路由、插件路由、session 端点等）均未设置；且 HSTS 默认不启用。改造目标：非 loopback 部署时默认开启 HSTS；对非 Control UI 和非 canvas 的通用 HTTP 响应默认追加 `X-Frame-Options: DENY` 和基线 `Content-Security-Policy`；新增配置开关 `gateway.http.securityHeaders.enforce` 供运维控制。

#### 涉及文件

`src/gateway/http-common.ts`（修改）、`src/config/zod-schema.ts`（修改）

#### 验收步骤

```
前置条件：
1. kclaw Gateway 已正常启动，监听地址为非 loopback。
2. 安全响应头强制加固功能已开启。

操作步骤：
1. 通过浏览器的开发工具或网络请求工具访问 Gateway 的 HTTP API 端点。
2. 检查响应头中是否包含 X-Frame-Options 和 Content-Security-Policy。
3. 检查响应头中是否包含 Strict-Transport-Security。
4. 访问 Control UI 页面，验证已有的安全响应头未被覆盖或重复设置。
5. 将 enforce 选项关闭后重新测试。

预期结果：
1. 非 Control UI 的通用 HTTP 响应包含 X-Frame-Options 和 Content-Security-Policy。
2. 非 loopback 部署的响应包含 Strict-Transport-Security 头。
3. Control UI 页面的安全响应头保持原有行为，不产生冲突或重复设置。
4. 关闭 enforce 后恢复原有行为，所有安全头回退到加固前的状态。
```

---

### M08 — 认证速率限制持久化模块

**优先级**: HIGH
**编码量**: ~60 行（修改现有文件）
**来源**: 原始安全审计

#### 概述

当前 `src/gateway/auth-rate-limit.ts`（236 行）已实现滑动窗口速率限制（`createAuthRateLimiter`），支持按 scope + clientIp 独立计数、可配置最大尝试次数/时间窗口/锁定时长、定期内存回收与 loopback 地址豁免。但计数器存储于纯内存 Map 中，Gateway 重启后全部归零——攻击者可反复触发重启绕过限速，无限次尝试猜测认证令牌，一旦破解即获得 Gateway 完整控制权。改为磁盘持久化：每次 `check()` 调用后将计数写入 `<stateDir>/rate-limit-state.json`，初始化时从文件恢复未过期的历史计数。

#### 涉及文件

`src/gateway/auth-rate-limit.ts`（修改）

#### 验收步骤

```
前置条件：
1. kclaw Gateway 已正常启动并配置了有效的认证令牌。
2. 当前限速配置为：最大失败次数 5 次、窗口 1 分钟、锁定 5 分钟。

操作步骤：
1. 使用错误令牌向 Gateway 连续发起若干次认证请求，直至触发拒绝。
2. 查看限速持久化文件是否记录有当前客户端的失败计数和首次尝试时间。
3. 重新启动 Gateway 服务。
4. 重启完成后立即使用正确令牌发起认证请求。
5. 等待超出锁定窗口后再次使用正确令牌发起认证。

预期结果：
1. 连续错误请求达到阈值后被 Gateway 拒绝。
2. 限速状态文件中记录有该客户端的失败计数和窗口信息。
3. Gateway 重启后计数未归零，锁定窗口内使用正确令牌也被限制。
4. 锁定窗口过期后正确令牌可正常通过认证。
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
      maxFailures: number; // 触发熔断的连续失败次数（默认 5）
      resetTimeoutMs: number; // 熔断后等待时间（默认 30s）
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

  private _onSuccess() {
    this.failureCount = 0;
    this.state = "closed";
  }
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
2. 该提供商的 API 端点可通过工具模拟连续返回失败。

操作步骤：
1. 向该 AI 提供商连续发起请求，使其持续返回失败。
2. 在连续失败达到预设阈值后再次发起请求。
3. 等待恢复时间窗口结束，然后发起一条探测请求。
4. 探测请求成功后继续发起正常请求。
5. 同时对另一个未发生故障的提供商发起请求。

预期结果：
1. 连续失败达到阈值后，后续请求被立即拒绝，不实际发送到下游 AI 服务。
2. 拒绝应答中包含熔断提示信息，不阻塞 agent 会话继续运行。
3. 恢复等待时间过后可发送探测请求，探测成功后恢复正常请求处理。
4. 多提供商独立熔断，一个提供商的故障不影响其他提供商的正常请求。
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

const ALGORITHM = "aes-256-gcm";
const KEY_LENGTH = 32;
const IV_LENGTH = 16;
const TAG_LENGTH = 16;
const SALT_LENGTH = 32;
const ENCRYPTED_MAGIC = Buffer.from([0x4f, 0x43, 0x45, 0x43]); // "OCEC"

export function deriveMasterKey(): Buffer {
  const machineId = readFileSync("/etc/machine-id", "utf8").trim();
  return scryptSync(`${machineId}:${require("node:os").hostname()}`, "kclaw-vault", KEY_LENGTH);
}

export function encrypt(plaintext: string, masterKey: Buffer): Buffer {
  const salt = randomBytes(SALT_LENGTH);
  const key = scryptSync(masterKey, salt, KEY_LENGTH);
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
1. kclaw 配置中已开启会话数据加密存储功能。
2. kclaw Gateway 已正常启动。
3. 至少已完成一次 agent 对话，产生了会话存储文件。

操作步骤：
1. 进入会话存储目录，以文件工具查看最新会话文件的内容。
2. 尝试通过文本工具直接读取该会话文件。
3. 将加密配置关闭后新建会话，查看新会话文件的内容。
4. 尝试在另一台不具备正确密钥的系统中打开加密会话文件。

预期结果：
1. 开启加密后生成的会话文件以加密格式存储，无法直接读取明文对话内容。
2. 关闭加密后新生成的会话文件恢复为可读的明文格式。
3. 加密会话文件在无正确密钥的系统中无法被解密恢复。
```

---

## 五、代码变更文件清单

```
kclaw/
├── kclaw.mjs                                # ★ 重命名
├── package.json                             # ★ 重命名 name/bin/scripts
├── pnpm-lock.yaml                           # 锁文件（不变）
│
├── src/
│   ├── config/
│   │   ├── paths.ts                         # ★ 路径常量 ~/.kclaw/
│   │   ├── io.ts                            # ★ M05 完整性校验
│   │   ├── backup-rotation.ts               # ★ M05 备份链验证
│   │   └── zod-schema.ts                    # ★ 新增 schema (M02/M04/M05/M06)
│   ├── infra/
│   │   ├── host-env-security.ts             # ★ M01 环境变量白名单
│   │   ├── net/
│   │   │   ├── ssrf.ts                      # ★ M02 网络出站白名单
│   │   │   └── fetch-guard.ts               # ★ M02 网络白名单接入
│   │   ├── exec-allowlist-pattern.ts        # ★ M03 路径越权检查
│   │   ├── exec-safety.ts                   # ★ M03 输入沙箱增强
│   │   └── circuit-breaker.ts               # ★ 新增 M09 熔断管理
│   ├── security/
│   │   ├── audit.ts                         # ★ M04 凭据检测 + M05 完整性审计
│   │   └── context-visibility.ts            # ★ M06 会话隔离
│   ├── sessions/
│   │   ├── send-policy.ts                   # ★ M06 会话隔离
│   │   └── encrypted-transcript.ts          # ★ 新增 M10 会话加密
│   ├── secrets/
│   │   └── audit.ts                         # ★ M04 凭据检测增强
│   ├── agents/
│   │   └── exec-defaults.ts                 # ★ M03 路径范围约束
│   ├── gateway/
│   │   ├── http-common.ts                   # ★ M07 安全响应头
│   │   └── auth-rate-limit.ts               # ★ M08 限速持久化
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

| 类型       | 数量     | 说明                                                                                                               |
| ---------- | -------- | ------------------------------------------------------------------------------------------------------------------ |
| 新增文件   | 3        | 2 个 TS (M09+M10) + 离线安装脚本                                                                                   |
| 修改文件   | 约 18    | 重命名 + 安全功能注入（覆盖 ~235 行改动）                                                                          |
| 全局替换   | ~200 处  | `KCLAW_` / `kclaw`                                                                                                 |
| 不涉及文件 | 其余全部 | 保持 OpenClaw 原始功能                                                                                             |
| 已移除     | 8        | 恒时校验 / 挂载管控 / 权限最小化 / 资源配额 / 流量防护 / 部署前置校验 / 内存管控 / 3 条 GHSA（代码不存在或已内置） |

---

## 六、架构介入点

```
kclaw 架构 (Fork 自 OpenClaw，★ 标记为变更注入点)

┌─────────────────────────────────────┐
│  原生应用  │  Web Control UI        │  ← 无变更
├─────────────────────────────────────┤
│  Gateway Server (HTTP/WS)           │
│  ├─ Auth ────── ★ M08 限速持久化     │
│  ├─ Sessions ── ★ M06 会话隔离      │
│  │              ★ M10 会话加密     │
│  └─ Config ──── ★ M05 完整性校验   │
├─────────────────────────────────────┤
│  Core Agent Runtime                 │
│  ├─ Server ──── ★ M07 安全响应头   │
│  ├─ Tool System ★ M03 Exec 路径防护 │
│  ├─ Transport ─ ★ M09 熔断管理     │
│  ├─ Network ─── ★ M02 网络白名单   │
│  └─ Env ─────── ★ M01 环境变量白名单│
├─────────────────────────────────────┤
│  Plugin SDK ─── 无变更              │
├─────────────────────────────────────┤
│  Extensions ─── 无变更 (兼容上游)    │
├─────────────────────────────────────┤
│  Infrastructure                     │
│  ├─ Config ──── ★ 全局重命名        │
│  ├─ Daemon ──── ★ systemd 适配     │
│  ├─ Secrets ─── ★ M04 凭据检测      │
│  └─ Security ── ★ M05 完整性校验   │
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
├── Day 3    M01 环境变量白名单 + M02 网络出站白名单
├── Day 4    M03 Exec 路径防护
├── Day 5    编译验证 + 基础冒烟测试 (麒麟 ARM64 + x86_64)

Week 2: 合规与数据安全
├── Day 1-2  M04 凭据检测 + M05 完整性校验
├── Day 3    M07 安全响应头 + M06 会话隔离 + M08 限速持久化
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

| 原始缺陷                              | CWE      | kclaw 整改模块                  | 来源 | 状态      |
| ------------------------------------- | -------- | ------------------------------- | ---- | --------- |
| 环境变量黑名单模式可泄露 HPC 凭据     | CWE-526  | M01 环境变量强制白名单模块      | 审计 | ⚠️ 待修复 |
| SSRF 黑名单可被绕过，内网隔离形同虚设 | CWE-923  | M02 网络出站白名单控制模块      | 审计 | ⚠️ 待修复 |
| 符号链接和路径遍历可绕过 exec 白名单  | CWE-22   | M03 Exec 命令路径越权防护模块   | 审计 | ⚠️ 待修复 |
| 配置文件可能残留明文 API Key/Token    | CWE-312  | M04 明文凭据检测与告警模块      | 审计 | ⚠️ 待修复 |
| 配置文件可被篡改而无声告警            | CWE-345  | M05 配置文件完整性校验模块      | 审计 | ⚠️ 待修复 |
| 多租户 agent 会话消息可跨 agent 泄露  | CWE-200  | M06 会话上下文可见性硬隔离模块  | 审计 | ⚠️ 待修复 |
| 通用 HTTP 响应缺少安全响应头          | CWE-1021 | M07 HTTP 安全响应头强制加固模块 | 审计 | ⚠️ 待修复 |
| 暴力破解限速重启清零可被反复绕过      | CWE-307  | M08 认证速率限制持久化模块      | 审计 | ⚠️ 待修复 |
| AI 服务故障无熔断保护                 | CWE-754  | M09 AI 服务熔断管理模块         | 审计 | ⚠️ 待修复 |
| 会话文件明文存储                      | CWE-311  | M10 会话数据加密存储模块        | 审计 | ⚠️ 待修复 |

---

## 九、风险与约束

| 风险                        | 影响                               | 应对                             |
| --------------------------- | ---------------------------------- | -------------------------------- |
| `KCLAW_*` env 全局替换遗漏  | 运行时隐式引用旧名                 | 用 `grep -r` 验证 + 自动化脚本   |
| 插件生态不兼容              | 外部插件引用 `openclaw/plugin-sdk` | 保留别名映射或导出兼容路径       |
| 上游关键安全补丁丢失        | 不跟随合并，漏洞需自行修复         | README 中记录已知差异 + 定期巡检 |
| 国内平台麒麟/UOS glibc 版本 | 预编译 ARM64 模块可能不兼容        | 离线包包含编译工具链回退脚本     |

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
pnpm build:strict-smoke           # SDK/出口边界变更时必须跑
                                  #   = build + plugin-sdk:dts + export checks
pnpm tsgo:prod                    # core + extensions 全量类型检查 (tsgo, 非 tsc)
pnpm tsgo:test                    # 测试文件类型检查
```

> **详细开发工作流**: 编译、测试、调试、门禁检查、Git 工作流参见 `docs/reference/kclaw-dev-guide.md`。

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
pnpm test <path-or-filter>

# 匹配模式
pnpm test "audit"

# 单线程（断点调试）
KCLAW_VITEST_MAX_WORKERS=1 pnpm test <path>

# 变更文件 / 扩展 / 全量
pnpm test:changed
pnpm test extensions
pnpm test:coverage
pnpm test:serial
```

**M01-M10 安全功能逐项验证：**

| 模块               | 验证方式 | 关键验证点                                                             |
| ------------------ | -------- | ---------------------------------------------------------------------- |
| M01 环境变量白名单 | 功能测试 | 未在白名单的宿主变量不传入子进程，白名单内变量正常传递                 |
| M02 网络出站白名单 | 功能测试 | 白名单外地址请求被拒绝，白名单内正常完成                               |
| M03 Exec 路径防护  | 功能测试 | 符号链接/路径遍历绕过白名单被拒绝，正常路径不受影响                    |
| M04 凭据检测       | 启动验证 | Gateway 启动时检测到明文凭据并告警，阻断模式下拒绝启动                 |
| M05 完整性校验     | 启动验证 | 配置未篡改时正常启动，篡改后告警或拒绝启动                             |
| M06 会话隔离       | 功能测试 | 完全隔离模式下跨 agent 消息不可见/不可发送，同 agent 共享正常          |
| M07 安全响应头     | 启动验证 | 通用 HTTP 响应含 X-Frame-Options/CSP/HSTS，Control UI 原有行为不受影响 |
| M08 限速持久化     | 启动验证 | 重启后锁定状态不丢失，窗口内正确令牌仍被限速，过期后恢复               |
| M09 熔断管理       | 模拟验证 | 连续失败后后续请求立即拒绝，半开探测成功恢复                           |
| M10 会话加密       | 集成验证 | 加密开启后磁盘文件不可读，关闭后恢复明文                               |

**全量门禁**: `pnpm check:changed && pnpm test`

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

| 文件         | 路径                                                       | 说明                           |
| ------------ | ---------------------------------------------------------- | ------------------------------ |
| 主配置       | `~/.kclaw/kclaw.json`                                      | Gateway/通道/模型/插件所有配置 |
| 完整性基线   | `~/.kclaw/config-integrity.json`                           | M05 配置文件完整性校验基线哈希 |
| 凭据         | `~/.kclaw/credentials/`                                    | 各通道 Token/API Key           |
| Agent 配置   | `~/.kclaw/agents/<id>/agent/auth-profiles.json`            | 模型认证配置                   |
| 会话文件     | `~/.kclaw/workspace/sessions/`                             | M10 会话加密存储               |
| systemd 单元 | `~/.config/systemd/user/kclaw-gateway[-<profile>].service` | 服务定义                       |

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

| 症状               | 检查步骤                                                        |
| ------------------ | --------------------------------------------------------------- |
| Gateway 启动失败   | `kclaw doctor` → `journalctl --user -u kclaw-gateway -e`        |
| 飞书收不到消息     | `kclaw channels status` → 检查 Webhook 地址可达性               |
| 模型调用超时       | `curl -v <provider-api-url>` → 检查 `HTTPS_PROXY`               |
| OOM 崩溃           | `journalctl` 查看 OOM 记录 → 检查 `--max-old-space-size`        |
| 离线安装失败       | `uname -m` 确认架构 → `ldd --version` 检查 glibc → 尝试回退编译 |
| 沙箱路径拦截不生效 | 检查路径是否通过 symlink 绕过                                   |

---

## 附录：后续规划（Phase 2）

以下功能模块在本轮优先级调整中暂未纳入，可在后续版本迭代中按需推进：

| 模块 | 正式名称                     | 编码量 | 说明                                               |
| ---- | ---------------------------- | ------ | -------------------------------------------------- |
| —    | 配置文件明文凭据检测告警模块 | ~30 行 | 启动扫描 kclaw.json 中的明文 API Key，输出 WARNING |
| —    | 会话数据安全擦除模块         | ~50 行 | 随机覆写 + CLI purge 命令，防文件恢复              |
| —    | 离线化部署方案               | shell  | 双架构离线安装包，支撑信创/军工/政务等断网环境     |
| —    | CVE 自动扫描                 | YAML   | 周扫描 pnpm audit + Trivy，CI 集成                 |
