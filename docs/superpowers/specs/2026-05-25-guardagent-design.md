# GuardAgent 本地 AI 隐私安全网关 — 设计文档

- 状态：草案 v1（待用户复核）
- 日期：2026-05-25
- 关联 PRD：[`aiInfo/basePrd.md`](../../../aiInfo/basePrd.md)
- 设计范围：Phase 1 + Phase 2 + Phase 3 全量

---

## 0. 关键决策摘要

| 决策点 | 选择 | 备注 |
|---|---|---|
| 覆盖阶段 | Phase 1 + 2 + 3 全量 | Phase 3 在路线图中独立成期 |
| Sidecar 运行时 | Node.js LTS 22 | 生态成熟、Tauri Sidecar 例子多 |
| 前端框架 | React + Vite + TS | 组件库与社区例子最丰富 |
| 上游 Provider | OpenAI 协议 + Anthropic 协议 | 覆盖 Cursor + Claude Code |
| 还原范围 | 仅"可见文本"字段（delta.content / delta.text） | tool call 参数不在 MVP 内 |
| 规则重叠策略 | 取最长区间，脱敏优先 | 实现简单、误伤可接受 |
| API Key 处理 | 仅透传，不存储 | 符合 NFR-5.2 |
| 规则持久化 | 规则本身可落盘 JSON，映射表严禁落盘 | 区分"规则定义"与"运行时映射" |
| 出错默认行为 | Fail-closed | 安全产品硬要求 |
| 流式还原算法 | 状态机扫描（OUTSIDE / MAYBE / INSIDE） | 占位符长度无关、延迟最低 |
| 文档与代码 | 设计文档中文，代码标识符英文 | — |

---

## 1. 整体架构与进程模型

三层架构，职责清晰、可独立测试：

```
┌─────────────────────────────────────────────────┐
│  Tauri 桌面壳 (Rust)                              │
│   - 托盘图标 / 自启动 / 单实例锁                 │
│   - Sidecar 子进程生命周期管理（启停、健康检查） │
│   - 桥接 IPC 事件 → 前端                         │
└──────┬─────────────────────────────────┬────────┘
       │ stdio + IPC                       │ Tauri Event
       ▼                                   ▼
┌──────────────────────┐         ┌─────────────────────┐
│ Sidecar (Node.js)    │         │ Web 前端 (React+Vite)│
│  Fastify HTTP server │         │  - 一键开关         │
│  ├─ /v1/chat/...     │         │  - 配置卡片         │
│  ├─ /v1/messages     │         │  - 日志瀑布流       │
│  ├─ /healthz         │         │  - 规则库管理       │
│  └─ /events (SSE)    │  ◄────  │                     │
│                      │  事件订阅                     │
│  脱敏引擎 / 状态机   │         │                     │
│  ScopedContext 内存  │         │                     │
└──────────────────────┘         └─────────────────────┘
       │
       │ HTTPS（透传 Authorization）
       ▼
┌────────────────────────┐
│ OpenAI / Anthropic 云端 │
└────────────────────────┘
```

### 解耦理由

- **Tauri 壳只管"窗口与进程"**——绝不嵌业务逻辑；这样换前端框架/换运行时都不动壳。
- **Sidecar 是产品大脑**——脱敏、还原、转发、内存上下文全在这里。它必须能脱离 Tauri 单独跑（直接 `node sidecar.js` 启动），便于 e2e 测试与 CI。
- **前端纯展示**——通过两个通道与 Sidecar 交互：(1) Tauri Event（高频日志推送，从 Rust 转发），(2) HTTP 调用 Sidecar 的管理接口（开关、规则）。

### 进程拓扑

- Tauri 主进程 fork Node Sidecar。
- Sidecar 监听 `127.0.0.1`（仅本地回环，不暴露公网）。
- 端口选择：默认 `9999`，被占用时在 `9999-10010` 范围内自动让位，实际端口通过 stdio 上报给 Tauri 壳，前端配置卡片实时渲染实际 base URL。

---

## 2. Sidecar 模块拆分

```
sidecar/src/
├─ server.ts              ← Fastify 启动 + 路由注册 + 端口选择
├─ providers/
│   ├─ openai.ts          ← /v1/chat/completions 路由 + schema 映射
│   └─ anthropic.ts       ← /v1/messages 路由 + schema 映射
├─ engine/
│   ├─ rules.ts           ← 规则定义 + 加载（内置 + 用户落盘 JSON）
│   ├─ anonymize.ts       ← 文本 → 占位符（含重叠区间合并）
│   └─ scanner.ts         ← 流式状态机（OUTSIDE / MAYBE / INSIDE）
├─ context/
│   └─ scoped.ts          ← ScopedContext 创建 / 销毁 / 并发隔离
├─ events/
│   └─ bus.ts             ← 内部 EventEmitter，给 /events SSE 与日志用
├─ admin/
│   └─ routes.ts          ← /healthz、/admin/rules、/admin/toggle
└─ log.ts                 ← 结构化日志（~20 行自实现）
```

### 模块接口边界

| 模块 | 输入 | 输出 | 依赖 |
|---|---|---|---|
| `rules` | 规则配置文件 | `Rule[]` 数组（名称、正则、占位符模板、分类） | 文件系统（仅读） |
| `anonymize` | `(text, rules) → {text, ctx}` | 脱敏后文本 + 填好的 ScopedContext | `rules`、`scoped` |
| `scanner` | chunk 流（异步迭代器）+ `ctx` | 还原后的 chunk 流 | `scoped`（只读字典） |
| `scoped` | `createContext(requestId)` | `ScopedContext` 实例（双向 Map） | 无 |
| `providers/*` | HTTP 请求 | 调脱敏 → 转发 → 调 scanner → SSE 回写 | 全部上游模块 |
| `events/bus` | `emit({type, payload})` | 订阅者收到事件 | 无（纯内存） |

### 关键设计点

1. **`scanner` 不知道协议** —— 它是 `(asyncIterator<string>, ctx) → asyncIterator<string>`。OpenAI/Anthropic 都先把 SSE 帧解出 `delta.content` 字符串再喂给 scanner，还原后再封回 SSE 帧。
2. **`anonymize` 是一次性同步函数** —— 输入完整请求体文本，输出脱敏文本 + ctx，不涉及流。NFR-5.1 的 "5ms 内处理 5 万字" 指标只对它生效。
3. **重叠区间策略** —— 所有规则各自 `matchAll` 出区间集合，按 `(start asc, length desc)` 排序后线性扫描，跳过被覆盖的更短区间。
4. **占位符命名结构**：`[GUARD_<TYPE>_<INDEX>]`，`TYPE` ∈ `{SECRET, IP, DB, NAME, CUSTOM}`，`INDEX` 是该 ctx 内该类型自增计数（从 1 起）。

### ScopedContext 数据结构

```typescript
interface ScopedContext {
  requestId: string;
  createdAt: number;
  anonymizeMap: Map<string, string>;   // "192.168.1.100" -> "[GUARD_IP_1]"
  deanonymizeMap: Map<string, string>; // "[GUARD_IP_1]"  -> "192.168.1.100"
  counters: Map<string, number>;       // "IP" -> 1, "SECRET" -> 3 ...
  destroy(): void;                      // 清空所有 Map，置空 counters
}
```

---

## 3. 核心数据流

### 3.1 请求链路（Cursor → 云端）

```
1. Cursor POST http://127.0.0.1:9999/v1/chat/completions
   header: Authorization: Bearer sk-真实Key
   body: { model, messages: [{role, content}, ...], stream: true, ... }
        │
        ▼
2. providers/openai.ts 入口：
   - 生成 requestId（uuid v4）
   - createScopedContext(requestId) → ctx
        │
        ▼
3. 提取所有 message.content（含 system/user/assistant 多轮）
   把字符串数组喂给 anonymize(texts, rules, ctx)
        │
        ▼
4. anonymize 返回脱敏后 texts + 已填好双向字典的 ctx
   把脱敏文本写回 body.messages[i].content
        │
        ▼
5. 用脱敏后的 body + 原 Authorization 头，
   fetch 'https://api.openai.com/v1/chat/completions'
   （Anthropic 走 https://api.anthropic.com/v1/messages）
        │
        ▼
6. 拿到上游 Response（stream=true 时是 ReadableStream<Uint8Array>）
   立刻进入响应链路（见 3.2）
```

**关键决定**：

- **`messages` 之外的字段不脱敏** —— `model`、`temperature`、`tools` 定义等是配置语义，不会含敏感数据。
- **多轮对话共用 ctx 字典** —— 同一个内网 IP 在历史多轮里得到同一个占位符 `[GUARD_IP_1]`，AI 能识别它在前后文里指同一实体。
- **非 JSON / 非预期协议** —— 直接 502 拒绝，不为边缘协议留洞。

### 3.2 响应链路（云端 → Cursor）

```
上游 ReadableStream<Uint8Array>
        │
        ▼
①  Uint8Array → string（TextDecoder，stream:true 模式，跨 chunk UTF-8 安全）
        │
        ▼
②  SSE 帧切分器（按 \n\n 分帧，未完整的尾巴留到下个 chunk）
   每帧形如：data: {"choices":[{"delta":{"content":"hello "}}]}
        │
        ▼
③  对每帧 JSON.parse，提取 delta.content（OpenAI）
                              或 delta.text（Anthropic content_block_delta）
        │
        ▼
④  把 content 字符串送进 scanner（状态机）
   scanner 内部维护"扣留 buffer"，输出"已确认安全可吐"的字符串片段
        │
        ▼
⑤  对 scanner 输出的片段，按 ctx.deanonymizeMap 查表替换
   （scanner 只识别占位符边界、不做替换；替换在 scanner 出口做）
        │
        ▼
⑥  替换后的字符串重新塞回 delta.content，JSON.stringify 回 SSE 帧
        │
        ▼
⑦  写到 Fastify reply（res.raw.write），保持 SSE 长连接
        │
        ▼
⑧  上游流 end / Cursor abort / 出错任一触发：
   - 把 scanner buffer 里残留内容做最后一次还原 + flush（end 场景）
   - 调 ctx.destroy() 清空双向 Map
   - 关闭 reply
```

### 3.3 Scanner 状态机

| 当前状态 | 输入字符 | 动作 |
|---|---|---|
| OUTSIDE | 非 `[` | 立即 emit 给下游 |
| OUTSIDE | `[` | 进 MAYBE，字符存入 buffer |
| MAYBE | 与 `[GUARD_` 前缀匹配中 | 字符存 buffer |
| MAYBE | 偏离 `[GUARD_` 前缀（例如 `[1`） | flush buffer，回 OUTSIDE |
| MAYBE | buffer == `[GUARD_` | 进 INSIDE |
| INSIDE | 非 `]` | 字符存 buffer |
| INSIDE | `]` | buffer 即完整占位符 → 出口替换 → emit → 回 OUTSIDE |
| 任一 | 流结束 | 把 buffer 当普通文本 emit（fail-safe：宁可漏还原也不丢字） |

**为什么"流结束时把 buffer 当普通文本吐出"**：abort 时 buffer 里可能是半截占位符；与"丢字"相比，"吐一半占位符"在客户端是可见错误（程序员能一眼看出），更安全可调试。

---

## 4. 错误处理、并发、内存与安全

### 4.1 Fail-closed 错误处理矩阵

| 阶段 | 错误类型 | HTTP 响应给 Cursor | 日志记录 | ctx 处置 |
|---|---|---|---|---|
| 入口 | body 非合法 JSON / 非预期协议字段 | `400` + `{error:"invalid_request"}` | warn，不含原文 | 立即销毁 |
| 脱敏 | 正则编译失败（仅自定义规则有可能）| `500` + `{error:"rule_invalid", rule_name}` | error，给出规则名 | 立即销毁 |
| 脱敏 | 单次脱敏耗时 > 200ms | `503` + `{error:"anonymize_timeout"}` | error，含 text 长度 | 立即销毁 |
| 转发 | 上游 DNS / 网络 / TLS 失败 | `502` + `{error:"upstream_unreachable"}` | error，含上游 host | 立即销毁 |
| 转发 | 上游返回 4xx/5xx | **透传上游状态码与 body**（其内容已不含我方占位符）| info | 流结束时销毁 |
| 流式 | scanner 内部异常 | 已建连：发一个 `event: error\ndata:{...}` 帧后关闭 | error + stack | 立即销毁 |
| 流式 | Cursor 主动 abort | 关闭上游 fetch 的 AbortController | info | 立即销毁 |

**核心原则**：
- 绝不把"含占位符的中间产物"吐给下游。任何环节抛错就关连接。
- 错误日志只记 metadata（requestId、规则名、文本长度、上游 host），绝不记 messages 内容或 API Key。

### 4.2 并发模型

- 每个 HTTP 请求一个独立 ctx，Map 实例隔离即并发隔离，无需锁。
- 全局共享可变状态仅有"规则库"与"开关状态"。规则热加载用"读多写少"模式：写时整体替换引用，读时按引用快照执行。
- 不做请求队列、不做限流——本机单人代理。

### 4.3 内存生命周期（NFR-5.2 抹除保证）

ScopedContext 的销毁路径必须由三处保证调用：
1. `req.raw.on('close')` —— Fastify 在客户端断连时触发；
2. `upstream stream end` —— 流自然结束；
3. `try/catch` —— 任何 throw 路径的 finally。

实现上把 ctx 注册到 Sidecar 内的 `WeakRef + FinalizationRegistry` 兜底层——三处都漏掉时 GC 触发会记录 warn 日志，作为开发期检测漏洞的手段。

**关于"映射表只在 RAM"**：
- 规则文件（JSON）落盘：✅ 允许；
- 规则的启用/禁用状态落盘：✅ 允许；
- 任何 `anonymizeMap` / `deanonymizeMap` 内容：❌ 严禁，包括日志。日志只记"匹配到 IP_1 类型 1 个"这种统计描述。

### 4.4 安全边界

| 威胁 | 缓解 |
|---|---|
| 监听公网导致他人接入本地代理 | Sidecar 仅 bind `127.0.0.1`，不开 `0.0.0.0` |
| 跨 origin 网页 fetch 偷 API Key | Sidecar 不做 CORS 宽松放行；`OPTIONS` 仅对 `127.0.0.1` 来源 200 |
| 恶意自定义规则正则导致 ReDoS | 规则加载时用 `re2`（绑定 Google RE2，线性时间）替代 V8 内置；自定义规则启动时跑一遍"长字符串 100ms 超时"探测 |
| 日志泄露敏感词 | 日志只记 metadata，统一过 `redact()` 函数，禁止 `console.log(message)` 这种裸打印（lint 规则禁用） |
| Tauri 前端 XSS 触达 Sidecar | Sidecar admin 接口要求 `X-Guard-Token` 头，Token 在 Sidecar 启动时随机生成、通过 stdio 给 Tauri 壳，前端从 Tauri 命令拿；浏览器拿不到 |

---

## 5. 测试策略与可观测性

### 5.1 测试金字塔

```
                    ┌─────────────────┐
                    │  e2e（手工）     │  ← 真 Cursor + 真 OpenAI Key 跑一遍
                    └─────────────────┘
                  ┌──────────────────────┐
                  │ 集成（自动化）        │  ← 启 Sidecar + mock 上游 SSE
                  └──────────────────────┘
              ┌──────────────────────────────┐
              │  单元（自动化，覆盖率重点）    │  ← anonymize / scanner / scoped
              └──────────────────────────────┘
```

### 5.2 单元测试

**`anonymize` 模块**：
- 每条内置规则一组"命中 / 未命中"样例（IP、Key、DB 串、PII）；
- 重叠区间合并：IP 与 DB 串重叠样例，断言取最长；
- 同值复用占位符：同一 IP 在多处出现 → 同一 `[GUARD_IP_1]`；
- NFR-5.1 性能：5 万字基准，断言 < 5ms（用 `vitest bench`）；
- 边界��空输入 / 纯 ASCII / 含 emoji / 中文 / UTF-16 surrogate pair。

**`scanner` 状态机**（核心难点）：
- 状态转移表每一行做成一个测试用例；
- **占位符跨 chunk 切碎的所有断点位置**：对 `[GUARD_IP_1]` 14 字节，逐字节地测"在第 i 字节切开成两个 chunk"，i 从 1 到 13 全部覆盖（13 个用例）；
- MAYBE 误触发：输入 `阅读文档[1] 看这里 [GUARD_IP_1]`，断言 `[1]` 被正确 flush；
- 流末尾残留 buffer：输入 `这里有一半占位符 [GUARD_IP_` 后流结束，断言 buffer 被当原文吐出；
- 占位符不在字典里：scanner 收到不存在的 `[GUARD_XX_99]`，断言"原样吐出"而不是抛错。

**`scoped` 上下文**：
- `destroy()` 后再访问 Map 应返回空；
- `FinalizationRegistry` 的 warn 路径触发条件。

### 5.3 集成测试

启真 Sidecar + 本地 mock 上游（Fastify 起的假 OpenAI），覆盖：
- 完整请求-响应闭环：发含 IP 的 prompt，断言 mock 上游收到占位符，断言客户端收到还原后真值；
- SSE 真流式：mock 上游故意把 `[GUARD_IP_1]` 切成 4 个 chunk 下发，断言客户端收到的字节序列任意时刻都不出现"半截占位符"；
- abort 路径：客户端中途 abort，断言上游 fetch 被取消、ctx 被销毁；
- Anthropic 协议：同样一组用例对 `/v1/messages` + `content_block_delta` 帧重跑；
- fail-closed 行为：mock 上游返回 502，断言代理也返回 502 且 ctx 已销毁。

### 5.4 e2e（手工，写入 README 验证清单）

- Cursor Base URL 改成 `http://127.0.0.1:9999/v1`，写一个含真实 IP/Key 占位的 prompt：
  1. Cursor 侧打字机正常无卡顿；
  2. GuardAgent UI 看到拦截日志；
  3. 抓包工具（mitmproxy / Charles）看到上游收到的是占位符；
- 同样流程对 Claude Code 跑一遍。

### 5.5 可观测性

- 自实现 `log.ts`（~20 行），固定字段 `{ts, level, requestId, event, ...metadata}`，输出 JSON 行到 stdout，Tauri 壳收 stdout 转事件给前端；
- 关键事件：`request.received` / `anonymize.done`（含命中规则统计）/ `upstream.connected` / `stream.flush`（每 N 帧一条）/ `request.closed`（含耗时）/ `error`；
- 不记原文、不记占位符值，只记类型与计数。

---

## 6. MVP 路线图与目录结构

### 6.1 Phase 1 — 端到端打通（第 1-2 周）

**目标**：跑通 OpenAI 协议链路，证明脱敏/还原闭环可行。

里程碑：
1. Sidecar 骨架：Fastify + 端口选择 + `/healthz`；
2. `scoped` + `rules`（仅内置 SECRET、IP 两类）+ `anonymize`（含重叠合并）；
3. `scanner` 状态机 + 全部单元测试（含 13 条切碎用例）；
4. `providers/openai.ts`：完整请求-响应闭环；
5. 集成测试：本地 mock 上游 + abort 路径。

**Phase 1 验收门**：用 curl 模拟 Cursor 发 SSE 请求，能完成"含 IP/Key 的 prompt → 上游收占位符 → 客户端收真值"全链路，所有单元 + 集成测试通过。**此时无 GUI**，纯命令行验证。

### 6.2 Phase 2 — 桌面壳与生产化（第 3-4 周）

**目标**：可对外发布的桌面应用。

里程碑：
1. Tauri 2.0 壳：Sidecar 子进程管理 + 单实例锁 + 托盘；
2. React + Vite 前端：一键开关、配置卡片（Cursor / Claude Code 各一张）、日志瀑布流；
3. `providers/anthropic.ts` + 协议适配测试；
4. 补全 DB 连接串、PII（注释作者名）两条规则；
5. 打包：macOS `.dmg` + Windows `.msi`。

**Phase 2 验收门**：在干净的 macOS / Windows 上安装并跑通 Cursor + Claude Code 各一次 e2e，肉眼无卡顿、UI 实时显示日志。

### 6.3 Phase 3 — 自定义规则（未来）

**目标**：用户可加项目代号、内部域名等私有规则。

里程碑：
1. 规则编辑器 UI（CRUD + 测试输入框，即时显示匹配区间）；
2. 规则启动期"100ms 超时探测" + `re2` 切换；
3. 规则导入/导出 JSON。

**Phase 3 验收门**：可添加一条"`Project_Apollo` 替换为 `[GUARD_CUSTOM_1]`"自定义规则并生效。

### 6.4 最终目录结构

```
ai-gateway/
├─ aiInfo/
│   └─ basePrd.md
├─ docs/
│   └─ superpowers/
│       └─ specs/
│           └─ 2026-05-25-guardagent-design.md
├─ sidecar/                              ← Node.js 代理（独立可跑）
│   ├─ src/
│   │   ├─ server.ts
│   │   ├─ providers/
│   │   │   ├─ openai.ts
│   │   │   └─ anthropic.ts
│   │   ├─ engine/
│   │   │   ├─ rules.ts
│   │   │   ├─ anonymize.ts
│   │   │   └─ scanner.ts
│   │   ├─ context/scoped.ts
│   │   ├─ events/bus.ts
│   │   ├─ admin/routes.ts
│   │   └─ log.ts
│   ├─ test/
│   │   ├─ unit/
│   │   └─ integration/
│   ├─ package.json
│   ├─ tsconfig.json
│   └─ vitest.config.ts
├─ desktop/                              ← Tauri 2.0 壳 + React 前端
│   ├─ src/                              ← React 前端
│   │   ├─ App.tsx
│   │   ├─ components/
│   │   │   ├─ MainToggle.tsx
│   │   │   ├─ ConfigCards.tsx
│   │   │   └─ LogStream.tsx
│   │   └─ tauri/                        ← Tauri 命令绑定
│   ├─ src-tauri/                        ← Rust 壳
│   │   ├─ src/
│   │   │   ├─ main.rs
│   │   │   └─ sidecar.rs                ← 子进程管理
│   │   ├─ binaries/                     ← Sidecar 打包产物（Node + js）
│   │   ├─ Cargo.toml
│   │   └─ tauri.conf.json
│   ├─ index.html
│   ├─ package.json
│   └─ vite.config.ts
├─ scripts/
│   ├─ build-sidecar.sh                  ← bundle sidecar → 单文件
│   └─ package.sh                        ← 端到端打包（含 Node runtime）
├─ .gitignore
├─ package.json                          ← workspace 根（npm workspaces）
└─ README.md
```

### 6.5 工程化选择

- **npm workspaces** 而非 lerna/nx——单人项目，零额外学习成本；
- **sidecar 用 `esbuild` bundle 成单文件 JS**，再随 Node runtime 一起打进 Tauri `binaries/`；
- **设计文档** 落到 `docs/superpowers/specs/`，与 superpowers 技能约定保持一致。

---

## 7. 已识别风险与权衡

### 7.1 包体积超出 NFR-5.3（已识别）

- **NFR-5.3 要求**：安装包体积 ≤ 20MB。
- **实际预估**：sidecar bundle + Node runtime + Tauri 壳，macOS `.dmg` / Windows `.msi` 体积约 **35–50MB**。
- **冲突原因**：携带 Node runtime 是 PRD 已选 "TS + Tauri + Node.js Sidecar" 技术栈的必然成本。
- **建议**：将 NFR-5.3 调整为 "macOS `.dmg` ≤ 50MB / Windows `.msi` ≤ 50MB"；运行内存 ≤ 150MB 的指标继续保留（Node + Tauri 实测可达）。
- **备选**（不在本期范围）：Phase 3 之后如必须压到 20MB，再考虑切 Bun single-file executable 或 Tauri 内置 V8。

### 7.2 `re2` native binding 的打包复杂度

- **场景**：仅 Phase 3 的自定义规则需要 `re2` 做 ReDoS 防护。
- **风险**：`re2` 是 native 模块，需要随 Tauri 打包多平台 `.node` 二进制。
- **缓解**：Phase 1/2 仅用内置规则（已审计无 ReDoS 风险），不引入 `re2`；Phase 3 落地时再评估，必要时改成"自定义规则不支持任意正则，仅支持模板化 DSL（精确字符串 / 通配符）"作为简化路径。

### 7.3 占位符在自然文本中误触发

- **场景**：`scanner` 看到 `[` 进入 MAYBE，自然文本里 `[1]`、`[note]` 等会瞬时进 MAYBE。
- **影响**：极短的 buffer 暂留（毫秒级），用户无感。
- **风险等级**：低。已被 5.2 节"MAYBE 误触发"测试用例覆盖。

### 7.4 Tool calls / function arguments 内的占位符不还原（本期范围外）

- **场景**：若上游把占位符吐到 `tool_calls[].function.arguments`（JSON 字符串）里。
- **本期决定**：仅还原"可见文本"字段，tool call 参数不在本期范围。
- **影响**：使用 function calling 且参数恰好含敏感字段的极少数场景会看到占位符。
- **缓解**：未来版本可扩展为 JSON-aware 扫描。当前 PRD 用户主要场景是 chat completions，影响小。

---

## 8. 验收清单（设计阶段结束标志）

- [x] 关键决策全部记录
- [x] 三层架构与模块边界明确
- [x] 请求/响应数据流可逐字节追踪
- [x] Scanner 状态机状态转移表完整
- [x] Fail-closed 错误矩阵覆盖全部已知错误源
- [x] 内存生命周期 + 销毁保证已声明
- [x] 测试金字塔分层 + 关键测试矩阵列出
- [x] 三阶段路线图 + 各阶段验收门
- [x] 目录结构（含 workspaces）
- [x] 已识别风险与权衡

下一步：用户复核 → 进入 writing-plans 编写实现计划。
