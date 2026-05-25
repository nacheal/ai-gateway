# GuardAgent Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现 Sidecar 代理的端到端最小闭环——能拦截 OpenAI `/v1/chat/completions` 的 SSE 请求，对 messages 中的内网 IP 与 API Key 进行脱敏，转发到上游，并把响应流中的占位符还原后回写客户端。

**Architecture:** Node.js + Fastify HTTP 服务器，仅监听 `127.0.0.1`。脱敏引擎是一次性同步函数（输入完整 prompt 文本，输出脱敏文本 + ScopedContext 双向字典）。响应流还原通过三态状态机（OUTSIDE / MAYBE / INSIDE）实现，与协议解耦。每个请求一个独立的 ScopedContext，三处销毁保证（close / end / catch-finally）+ `FinalizationRegistry` 兜底。

**Tech Stack:** Node.js 22 LTS, TypeScript 5, Fastify 5, Vitest 2 (单元 + bench + 集成), undici 内置 fetch (Node 22), npm workspaces。

**Scope:** 仅 Phase 1。OpenAI 协议、内置规则两类（SECRET + IP）、scanner 状态机、集成测试、curl 验证脚本。**无 GUI、不打包 Tauri、不接 Anthropic、无 DB/PII 规则、无自定义规则**——这些都在后续 Phase。

**关联设计文档：** [`docs/superpowers/specs/2026-05-25-guardagent-design.md`](../specs/2026-05-25-guardagent-design.md)

---

## File Structure

Phase 1 完成时项目应当是这样：

```
ai-gateway/
├─ aiInfo/basePrd.md                       (existing)
├─ docs/superpowers/                       (existing)
├─ package.json                            ← Create: workspaces 根
├─ .gitignore                              ← Create
├─ sidecar/
│   ├─ package.json                        ← Create: sidecar workspace
│   ├─ tsconfig.json                       ← Create
│   ├─ vitest.config.ts                    ← Create
│   ├─ src/
│   │   ├─ log.ts                          ← Create: 结构化日志（~30 行）
│   │   ├─ context/scoped.ts               ← Create: ScopedContext 工厂 + 销毁兜底
│   │   ├─ engine/
│   │   │   ├─ rules.ts                    ← Create: 内置规则 (SECRET + IP)
│   │   │   ├─ anonymize.ts                ← Create: 文本→占位符（含重叠合并）
│   │   │   └─ scanner.ts                  ← Create: 流式状态机
│   │   ├─ sse/
│   │   │   └─ frames.ts                   ← Create: SSE 帧切分/拼装
│   │   ├─ providers/
│   │   │   └─ openai.ts                   ← Create: /v1/chat/completions 路由
│   │   └─ server.ts                       ← Create: Fastify 启动 + 端口选择
│   └─ test/
│       ├─ unit/
│       │   ├─ scoped.test.ts              ← Create
│       │   ├─ rules.test.ts               ← Create
│       │   ├─ anonymize.test.ts           ← Create
│       │   ├─ anonymize.bench.ts          ← Create (NFR-5.1 性能基准)
│       │   ├─ scanner.test.ts             ← Create
│       │   └─ frames.test.ts              ← Create
│       └─ integration/
│           ├─ mock-upstream.ts            ← Create: 假 OpenAI（Fastify 启）
│           ├─ openai-happy-path.test.ts   ← Create
│           ├─ openai-stream-cut.test.ts   ← Create
│           ├─ openai-abort.test.ts        ← Create
│           └─ openai-fail-closed.test.ts  ← Create
└─ scripts/
    └─ verify-curl.sh                      ← Create: 手工验证脚本
```

---

## Task 1: 项目骨架与工具链

**Files:**
- Create: `package.json` (workspaces 根)
- Create: `.gitignore`
- Create: `sidecar/package.json`
- Create: `sidecar/tsconfig.json`
- Create: `sidecar/vitest.config.ts`

- [ ] **Step 1: 写根 package.json**

文件路径：`package.json`

```json
{
  "name": "ai-gateway",
  "private": true,
  "version": "0.1.0",
  "workspaces": ["sidecar"],
  "scripts": {
    "test": "npm -w sidecar run test",
    "bench": "npm -w sidecar run bench",
    "dev": "npm -w sidecar run dev",
    "build": "npm -w sidecar run build"
  }
}
```

- [ ] **Step 2: 写 .gitignore**

文件路径：`.gitignore`

```
node_modules/
dist/
*.log
.DS_Store
.vscode/
.idea/
coverage/
```

- [ ] **Step 3: 写 sidecar/package.json**

文件路径：`sidecar/package.json`

```json
{
  "name": "@ai-gateway/sidecar",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/server.js",
  "scripts": {
    "dev": "tsx src/server.ts",
    "build": "tsc -p tsconfig.json",
    "test": "vitest run",
    "test:watch": "vitest",
    "bench": "vitest bench --run"
  },
  "dependencies": {
    "fastify": "^5.1.0"
  },
  "devDependencies": {
    "@types/node": "^22.10.0",
    "tsx": "^4.19.0",
    "typescript": "^5.6.0",
    "vitest": "^2.1.0"
  }
}
```

- [ ] **Step 4: 写 sidecar/tsconfig.json**

文件路径：`sidecar/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "lib": ["ES2022"],
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "declaration": false,
    "sourceMap": true
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist", "test"]
}
```

- [ ] **Step 5: 写 sidecar/vitest.config.ts**

文件路径：`sidecar/vitest.config.ts`

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['test/**/*.test.ts'],
    benchmark: {
      include: ['test/**/*.bench.ts'],
    },
    testTimeout: 10000,
    pool: 'forks',
  },
});
```

- [ ] **Step 6: 安装依赖**

运行命令（在仓库根）：

```bash
cd ai-gateway && npm install
```

预期：在 `node_modules/` 内拉下 fastify、vitest、tsx、typescript；无报错。

- [ ] **Step 7: 验证 TypeScript 与 Vitest 能跑**

运行命令：

```bash
cd ai-gateway && npx -w sidecar tsc --noEmit && npx -w sidecar vitest run
```

预期：`tsc` 无输出（暂无源码也无错误）；`vitest` 输出 `No test files found`。

- [ ] **Step 8: Commit**

```bash
cd ai-gateway && git add package.json .gitignore sidecar/package.json sidecar/tsconfig.json sidecar/vitest.config.ts package-lock.json
git commit -m "chore(sidecar): scaffold workspaces, ts, vitest"
```

---

## Task 2: 结构化日志 log.ts

**Files:**
- Create: `sidecar/src/log.ts`
- Test: `sidecar/test/unit/log.test.ts`

- [ ] **Step 1: 写失败测试**

文件路径：`sidecar/test/unit/log.test.ts`

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { log } from '../../src/log.js';

describe('log', () => {
  let stdoutSpy: ReturnType<typeof vi.spyOn>;

  beforeEach(() => {
    stdoutSpy = vi.spyOn(process.stdout, 'write').mockImplementation(() => true);
  });
  afterEach(() => {
    stdoutSpy.mockRestore();
  });

  it('writes one JSON line per call with required fields', () => {
    log.info('request.received', { requestId: 'req-1', path: '/v1/chat/completions' });
    expect(stdoutSpy).toHaveBeenCalledTimes(1);
    const line = stdoutSpy.mock.calls[0]![0] as string;
    expect(line.endsWith('\n')).toBe(true);
    const obj = JSON.parse(line);
    expect(obj.level).toBe('info');
    expect(obj.event).toBe('request.received');
    expect(obj.requestId).toBe('req-1');
    expect(obj.path).toBe('/v1/chat/completions');
    expect(typeof obj.ts).toBe('number');
  });

  it('supports warn and error levels', () => {
    log.warn('rule.slow', { rule: 'IP' });
    log.error('upstream.fail', { host: 'api.openai.com' });
    expect(stdoutSpy).toHaveBeenCalledTimes(2);
    const w = JSON.parse(stdoutSpy.mock.calls[0]![0] as string);
    const e = JSON.parse(stdoutSpy.mock.calls[1]![0] as string);
    expect(w.level).toBe('warn');
    expect(e.level).toBe('error');
  });
});
```

- [ ] **Step 2: 运行测试，确认失败**

```bash
cd ai-gateway/sidecar && npx vitest run test/unit/log.test.ts
```

预期：FAIL，错误信息 `Cannot find module '../../src/log.js'`。

- [ ] **Step 3: 写最小实现**

文件路径：`sidecar/src/log.ts`

```typescript
type Level = 'info' | 'warn' | 'error';
type Fields = Record<string, unknown>;

function write(level: Level, event: string, fields: Fields = {}): void {
  const line = JSON.stringify({ ts: Date.now(), level, event, ...fields }) + '\n';
  process.stdout.write(line);
}

export const log = {
  info: (event: string, fields?: Fields) => write('info', event, fields),
  warn: (event: string, fields?: Fields) => write('warn', event, fields),
  error: (event: string, fields?: Fields) => write('error', event, fields),
};
```

- [ ] **Step 4: 运行测试，确认通过**

```bash
cd ai-gateway/sidecar && npx vitest run test/unit/log.test.ts
```

预期：PASS，2 个测试用例通过。

- [ ] **Step 5: Commit**

```bash
cd ai-gateway && git add sidecar/src/log.ts sidecar/test/unit/log.test.ts
git commit -m "feat(sidecar): structured JSON line logger"
```

---

## Task 3: ScopedContext 与销毁兜底

**Files:**
- Create: `sidecar/src/context/scoped.ts`
- Test: `sidecar/test/unit/scoped.test.ts`

- [ ] **Step 1: 写失败测试**

文件路径：`sidecar/test/unit/scoped.test.ts`

```typescript
import { describe, it, expect } from 'vitest';
import { createScopedContext } from '../../src/context/scoped.js';

describe('ScopedContext', () => {
  it('creates context with requestId and empty maps', () => {
    const ctx = createScopedContext('req-1');
    expect(ctx.requestId).toBe('req-1');
    expect(ctx.anonymizeMap.size).toBe(0);
    expect(ctx.deanonymizeMap.size).toBe(0);
    expect(ctx.counters.size).toBe(0);
    expect(ctx.createdAt).toBeGreaterThan(0);
  });

  it('allocates monotonically increasing index per type', () => {
    const ctx = createScopedContext('req-1');
    expect(ctx.nextIndex('IP')).toBe(1);
    expect(ctx.nextIndex('IP')).toBe(2);
    expect(ctx.nextIndex('SECRET')).toBe(1);
    expect(ctx.nextIndex('IP')).toBe(3);
  });

  it('destroy clears all maps and counters', () => {
    const ctx = createScopedContext('req-1');
    ctx.anonymizeMap.set('a', 'b');
    ctx.deanonymizeMap.set('b', 'a');
    ctx.nextIndex('IP');
    ctx.destroy();
    expect(ctx.anonymizeMap.size).toBe(0);
    expect(ctx.deanonymizeMap.size).toBe(0);
    expect(ctx.counters.size).toBe(0);
  });

  it('destroy is idempotent', () => {
    const ctx = createScopedContext('req-1');
    ctx.destroy();
    expect(() => ctx.destroy()).not.toThrow();
  });
});
```

- [ ] **Step 2: 运行测试，确认失败**

```bash
cd ai-gateway/sidecar && npx vitest run test/unit/scoped.test.ts
```

预期：FAIL，`Cannot find module`。

- [ ] **Step 3: 写最小实现**

文件路径：`sidecar/src/context/scoped.ts`

```typescript
import { log } from '../log.js';

export interface ScopedContext {
  requestId: string;
  createdAt: number;
  anonymizeMap: Map<string, string>;
  deanonymizeMap: Map<string, string>;
  counters: Map<string, number>;
  nextIndex(type: string): number;
  destroy(): void;
}

const finalizer = new FinalizationRegistry<string>((requestId) => {
  log.warn('scoped.leaked', { requestId });
});

export function createScopedContext(requestId: string): ScopedContext {
  const anonymizeMap = new Map<string, string>();
  const deanonymizeMap = new Map<string, string>();
  const counters = new Map<string, number>();
  let destroyed = false;

  const ctx: ScopedContext = {
    requestId,
    createdAt: Date.now(),
    anonymizeMap,
    deanonymizeMap,
    counters,
    nextIndex(type) {
      const next = (counters.get(type) ?? 0) + 1;
      counters.set(type, next);
      return next;
    },
    destroy() {
      if (destroyed) return;
      destroyed = true;
      anonymizeMap.clear();
      deanonymizeMap.clear();
      counters.clear();
      finalizer.unregister(ctx);
    },
  };

  finalizer.register(ctx, requestId, ctx);
  return ctx;
}
```

- [ ] **Step 4: 运行测试，确认通过**

```bash
cd ai-gateway/sidecar && npx vitest run test/unit/scoped.test.ts
```

预期：PASS，4 个用例。

- [ ] **Step 5: Commit**

```bash
cd ai-gateway && git add sidecar/src/context/scoped.ts sidecar/test/unit/scoped.test.ts
git commit -m "feat(sidecar): ScopedContext with destroy guard and finalization registry"
```

---

## Task 4: 内置规则定义 rules.ts

**Files:**
- Create: `sidecar/src/engine/rules.ts`
- Test: `sidecar/test/unit/rules.test.ts`

- [ ] **Step 1: 写失败测试**

文件路径：`sidecar/test/unit/rules.test.ts`

```typescript
import { describe, it, expect } from 'vitest';
import { builtinRules, type Rule } from '../../src/engine/rules.js';

describe('builtinRules', () => {
  it('contains SECRET and IP rule categories', () => {
    const types = new Set(builtinRules.map((r) => r.type));
    expect(types.has('SECRET')).toBe(true);
    expect(types.has('IP')).toBe(true);
  });

  it('every rule has a globally-flagged regex', () => {
    for (const r of builtinRules) {
      expect(r.pattern.flags).toContain('g');
    }
  });

  it('IP rule matches 192.168.x.x and 10.x.x.x but skips public IPs', () => {
    const ipRule = builtinRules.find((r) => r.type === 'IP' && r.name === 'private-ipv4')!;
    const text = '192.168.1.100 server, 10.0.0.5 backup, 8.8.8.8 public';
    const matches = Array.from(text.matchAll(ipRule.pattern)).map((m) => m[0]);
    expect(matches).toContain('192.168.1.100');
    expect(matches).toContain('10.0.0.5');
    expect(matches).not.toContain('8.8.8.8');
  });

  it('SECRET rule matches openai-style sk- keys', () => {
    const keyRule = builtinRules.find((r) => r.type === 'SECRET' && r.name === 'openai-key')!;
    const text = 'API_KEY=sk-abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUV';
    const matches = Array.from(text.matchAll(keyRule.pattern)).map((m) => m[0]);
    expect(matches.length).toBe(1);
    expect(matches[0]!.startsWith('sk-')).toBe(true);
  });

  it('SECRET rule matches password assignment patterns', () => {
    const pwRule = builtinRules.find((r) => r.type === 'SECRET' && r.name === 'password-assignment')!;
    const text = "password: 'hunter2'\nPASSWORD=\"correctHorse\"";
    const matches = Array.from(text.matchAll(pwRule.pattern));
    expect(matches.length).toBe(2);
  });

  it('Rule type is exported and well-typed', () => {
    const r: Rule = builtinRules[0]!;
    expect(typeof r.name).toBe('string');
    expect(typeof r.type).toBe('string');
    expect(r.pattern).toBeInstanceOf(RegExp);
  });
});
```

- [ ] **Step 2: 运行测试，确认失败**

```bash
cd ai-gateway/sidecar && npx vitest run test/unit/rules.test.ts
```

预期：FAIL，`Cannot find module`。

- [ ] **Step 3: 写最小实现**

文件路径：`sidecar/src/engine/rules.ts`

```typescript
export type RuleType = 'SECRET' | 'IP' | 'DB' | 'NAME' | 'CUSTOM';

export interface Rule {
  name: string;
  type: RuleType;
  pattern: RegExp;
}

export const builtinRules: Rule[] = [
  {
    name: 'openai-key',
    type: 'SECRET',
    pattern: /sk-[a-zA-Z0-9]{32,}/g,
  },
  {
    name: 'password-assignment',
    type: 'SECRET',
    pattern: /(?:password|PASSWORD|passwd)\s*[:=]\s*['"][^'"\n]+['"]/g,
  },
  {
    name: 'aws-access-key',
    type: 'SECRET',
    pattern: /AKIA[0-9A-Z]{16}/g,
  },
  {
    name: 'private-ipv4',
    type: 'IP',
    pattern: /\b(?:10(?:\.\d{1,3}){3}|192\.168(?:\.\d{1,3}){2}|172\.(?:1[6-9]|2\d|3[01])(?:\.\d{1,3}){2})\b/g,
  },
];
```

- [ ] **Step 4: 运行测试，确认通过**

```bash
cd ai-gateway/sidecar && npx vitest run test/unit/rules.test.ts
```

预期：PASS，6 个用例。

- [ ] **Step 5: Commit**

```bash
cd ai-gateway && git add sidecar/src/engine/rules.ts sidecar/test/unit/rules.test.ts
git commit -m "feat(sidecar): built-in SECRET and IP rules"
```

---

## Task 5: 脱敏引擎 anonymize.ts（含重叠合并）

**Files:**
- Create: `sidecar/src/engine/anonymize.ts`
- Test: `sidecar/test/unit/anonymize.test.ts`

- [ ] **Step 1: 写失败测试**

文件路径：`sidecar/test/unit/anonymize.test.ts`

```typescript
import { describe, it, expect } from 'vitest';
import { anonymize } from '../../src/engine/anonymize.js';
import { createScopedContext } from '../../src/context/scoped.js';
import { builtinRules, type Rule } from '../../src/engine/rules.js';

describe('anonymize', () => {
  it('replaces a private IP with a [GUARD_IP_1] placeholder', () => {
    const ctx = createScopedContext('r1');
    const out = anonymize(['连接到 192.168.1.100 端口 8080'], builtinRules, ctx);
    expect(out[0]).toBe('连接到 [GUARD_IP_1] 端口 8080');
    expect(ctx.deanonymizeMap.get('[GUARD_IP_1]')).toBe('192.168.1.100');
  });

  it('reuses the same placeholder for repeated values', () => {
    const ctx = createScopedContext('r1');
    const out = anonymize(['ip1=192.168.1.100\nip2=192.168.1.100\nip3=10.0.0.1'], builtinRules, ctx);
    expect(out[0]).toBe('ip1=[GUARD_IP_1]\nip2=[GUARD_IP_1]\nip3=[GUARD_IP_2]');
  });

  it('shares ctx across multiple texts (multi-turn conversation)', () => {
    const ctx = createScopedContext('r1');
    const out = anonymize(['看 192.168.1.100', '再看 192.168.1.100'], builtinRules, ctx);
    expect(out[0]).toBe('看 [GUARD_IP_1]');
    expect(out[1]).toBe('再看 [GUARD_IP_1]');
  });

  it('handles both SECRET and IP categories with independent counters', () => {
    const ctx = createScopedContext('r1');
    const out = anonymize(['key=sk-abcdefghijklmnopqrstuvwxyzABCDEFGH ip=192.168.1.1'], builtinRules, ctx);
    expect(out[0]).toMatch(/key=\[GUARD_SECRET_1\] ip=\[GUARD_IP_1\]/);
  });

  it('keeps non-matching text unchanged', () => {
    const ctx = createScopedContext('r1');
    const out = anonymize(['普通文本 nothing sensitive 12345 abc'], builtinRules, ctx);
    expect(out[0]).toBe('普通文本 nothing sensitive 12345 abc');
  });

  it('returns empty string unchanged', () => {
    const ctx = createScopedContext('r1');
    const out = anonymize([''], builtinRules, ctx);
    expect(out[0]).toBe('');
  });

  it('overlapping matches: prefer longest interval', () => {
    const ctx = createScopedContext('r1');
    const customRules: Rule[] = [
      { name: 'short', type: 'IP', pattern: /192\.168/g },
      { name: 'long', type: 'IP', pattern: /192\.168\.1\.100/g },
    ];
    const out = anonymize(['ip=192.168.1.100'], customRules, ctx);
    expect(out[0]).toBe('ip=[GUARD_IP_1]');
    expect(ctx.deanonymizeMap.get('[GUARD_IP_1]')).toBe('192.168.1.100');
  });

  it('handles surrogate pairs (emoji) without corrupting indices', () => {
    const ctx = createScopedContext('r1');
    const out = anonymize(['😀 server 192.168.1.100 😀'], builtinRules, ctx);
    expect(out[0]).toBe('😀 server [GUARD_IP_1] 😀');
  });

  it('populates anonymizeMap and deanonymizeMap consistently', () => {
    const ctx = createScopedContext('r1');
    anonymize(['192.168.1.100'], builtinRules, ctx);
    expect(ctx.anonymizeMap.get('192.168.1.100')).toBe('[GUARD_IP_1]');
    expect(ctx.deanonymizeMap.get('[GUARD_IP_1]')).toBe('192.168.1.100');
  });
});
```

- [ ] **Step 2: 运行测试，确认失败**

```bash
cd ai-gateway/sidecar && npx vitest run test/unit/anonymize.test.ts
```

预期：FAIL，`Cannot find module`。

- [ ] **Step 3: 写最小实现**

文件路径：`sidecar/src/engine/anonymize.ts`

```typescript
import type { Rule } from './rules.js';
import type { ScopedContext } from '../context/scoped.js';

interface Match {
  start: number;
  end: number;
  value: string;
  type: string;
}

function collectMatches(text: string, rules: Rule[]): Match[] {
  const matches: Match[] = [];
  for (const rule of rules) {
    rule.pattern.lastIndex = 0;
    for (const m of text.matchAll(rule.pattern)) {
      if (m.index === undefined) continue;
      matches.push({
        start: m.index,
        end: m.index + m[0].length,
        value: m[0],
        type: rule.type,
      });
    }
  }
  return matches;
}

function pickLongestNonOverlapping(matches: Match[]): Match[] {
  matches.sort((a, b) => a.start - b.start || (b.end - b.start) - (a.end - a.start));
  const out: Match[] = [];
  let cursor = -1;
  for (const m of matches) {
    if (m.start >= cursor) {
      out.push(m);
      cursor = m.end;
    }
  }
  return out;
}

function placeholderFor(value: string, type: string, ctx: ScopedContext): string {
  const existing = ctx.anonymizeMap.get(value);
  if (existing) return existing;
  const idx = ctx.nextIndex(type);
  const ph = `[GUARD_${type}_${idx}]`;
  ctx.anonymizeMap.set(value, ph);
  ctx.deanonymizeMap.set(ph, value);
  return ph;
}

function rewrite(text: string, matches: Match[], ctx: ScopedContext): string {
  if (matches.length === 0) return text;
  const parts: string[] = [];
  let pos = 0;
  for (const m of matches) {
    if (m.start > pos) parts.push(text.slice(pos, m.start));
    parts.push(placeholderFor(m.value, m.type, ctx));
    pos = m.end;
  }
  if (pos < text.length) parts.push(text.slice(pos));
  return parts.join('');
}

export function anonymize(texts: string[], rules: Rule[], ctx: ScopedContext): string[] {
  return texts.map((text) => {
    const all = collectMatches(text, rules);
    const chosen = pickLongestNonOverlapping(all);
    return rewrite(text, chosen, ctx);
  });
}
```

- [ ] **Step 4: 运行测试，确认通过**

```bash
cd ai-gateway/sidecar && npx vitest run test/unit/anonymize.test.ts
```

预期：PASS，9 个用例。

- [ ] **Step 5: 写性能基准 (NFR-5.1)**

文件路径：`sidecar/test/unit/anonymize.bench.ts`

```typescript
import { bench, describe } from 'vitest';
import { anonymize } from '../../src/engine/anonymize.js';
import { createScopedContext } from '../../src/context/scoped.js';
import { builtinRules } from '../../src/engine/rules.js';

const blob = (() => {
  const lines: string[] = [];
  for (let i = 0; i < 1000; i++) {
    lines.push(`line ${i}: connect to 192.168.${i % 255}.${(i * 7) % 255} port 8080`);
    lines.push(`token = sk-${'a'.repeat(40)}${i}`);
    lines.push(`some text without secrets that is just here to pad bytes to ~50k.`);
  }
  return lines.join('\n');
})();

describe('anonymize 50k chars (NFR-5.1: <5ms)', () => {
  bench('full pass', () => {
    const ctx = createScopedContext('bench');
    anonymize([blob], builtinRules, ctx);
    ctx.destroy();
  });
});
```

- [ ] **Step 6: 运行基准并目测耗时**

```bash
cd ai-gateway/sidecar && npx vitest bench --run test/unit/anonymize.bench.ts
```

预期：单次耗时 < 5ms（NFR-5.1）。若超出 5ms，记录实测数字到设计文档"已识别风险"，但不阻塞 Phase 1。

- [ ] **Step 7: Commit**

```bash
cd ai-gateway && git add sidecar/src/engine/anonymize.ts sidecar/test/unit/anonymize.test.ts sidecar/test/unit/anonymize.bench.ts
git commit -m "feat(sidecar): anonymize engine with longest-interval overlap merge"
```

---

## Task 6: 流式状态机 scanner.ts

**Files:**
- Create: `sidecar/src/engine/scanner.ts`
- Test: `sidecar/test/unit/scanner.test.ts`

- [ ] **Step 1: 写失败测试 (状态机基础行为)**

文件路径：`sidecar/test/unit/scanner.test.ts`

```typescript
import { describe, it, expect } from 'vitest';
import { Scanner } from '../../src/engine/scanner.js';

function dict(entries: Array<[string, string]>): Map<string, string> {
  return new Map(entries);
}

describe('Scanner OUTSIDE/MAYBE/INSIDE basics', () => {
  it('passes through plain text immediately', () => {
    const s = new Scanner(new Map());
    expect(s.push('hello world')).toBe('hello world');
    expect(s.flush()).toBe('');
  });

  it('flushes a stray bracket that does not match the prefix', () => {
    const s = new Scanner(new Map());
    expect(s.push('see [1] here')).toBe('see [1] here');
  });

  it('detects and replaces a full placeholder in one chunk', () => {
    const s = new Scanner(dict([['[GUARD_IP_1]', '192.168.1.100']]));
    expect(s.push('server at [GUARD_IP_1] online')).toBe('server at 192.168.1.100 online');
  });

  it('passes unknown placeholder through unchanged (no throw)', () => {
    const s = new Scanner(dict([]));
    expect(s.push('mystery [GUARD_XX_99] here')).toBe('mystery [GUARD_XX_99] here');
  });

  it('on flush, dumps buffered text as-is (fail-safe)', () => {
    const s = new Scanner(new Map());
    expect(s.push('tail [GUARD_IP_')).toBe('tail ');
    expect(s.flush()).toBe('[GUARD_IP_');
  });
});

describe('Scanner placeholder split across chunks (FR-4.1 / FR-4.2)', () => {
  const placeholder = '[GUARD_IP_1]';
  const real = '192.168.1.100';
  const text = `prefix ${placeholder} suffix`;

  for (let i = 1; i < placeholder.length; i++) {
    it(`split at byte ${i} of "${placeholder}" -> recovers correctly`, () => {
      const cutPoint = 'prefix '.length + i;
      const a = text.slice(0, cutPoint);
      const b = text.slice(cutPoint);
      const s = new Scanner(dict([[placeholder, real]]));
      const out = s.push(a) + s.push(b) + s.flush();
      expect(out).toBe(`prefix ${real} suffix`);
    });
  }
});

describe('Scanner MAYBE false-trigger flush', () => {
  it('flushes incomplete maybe-prefix when next char diverges', () => {
    const s = new Scanner(new Map());
    expect(s.push('[1]')).toBe('[1]');
  });

  it('handles back-to-back brackets correctly', () => {
    const s = new Scanner(dict([['[GUARD_IP_1]', '10.0.0.5']]));
    expect(s.push('[a][b][GUARD_IP_1][c]')).toBe('[a][b]10.0.0.5[c]');
  });
});
```

- [ ] **Step 2: 运行测试，确认失败**

```bash
cd ai-gateway/sidecar && npx vitest run test/unit/scanner.test.ts
```

预期：FAIL，`Cannot find module`。

- [ ] **Step 3: 写最小实现**

文件路径：`sidecar/src/engine/scanner.ts`

```typescript
const PREFIX = '[GUARD_';

type State = 'OUTSIDE' | 'MAYBE' | 'INSIDE';

export class Scanner {
  private state: State = 'OUTSIDE';
  private buffer = '';

  constructor(private readonly deanonymizeMap: Map<string, string>) {}

  push(chunk: string): string {
    let out = '';
    for (const ch of chunk) {
      out += this.consume(ch);
    }
    return out;
  }

  flush(): string {
    const tail = this.buffer;
    this.buffer = '';
    this.state = 'OUTSIDE';
    return tail;
  }

  private consume(ch: string): string {
    if (this.state === 'OUTSIDE') {
      if (ch === '[') {
        this.buffer = ch;
        this.state = 'MAYBE';
        return '';
      }
      return ch;
    }

    if (this.state === 'MAYBE') {
      const next = this.buffer + ch;
      if (next === PREFIX) {
        this.buffer = next;
        this.state = 'INSIDE';
        return '';
      }
      if (PREFIX.startsWith(next)) {
        this.buffer = next;
        return '';
      }
      const dump = this.buffer;
      this.buffer = '';
      this.state = 'OUTSIDE';
      return dump + this.consume(ch);
    }

    if (ch === ']') {
      const full = this.buffer + ch;
      this.buffer = '';
      this.state = 'OUTSIDE';
      return this.deanonymizeMap.get(full) ?? full;
    }
    this.buffer += ch;
    return '';
  }
}
```

- [ ] **Step 4: 运行测试，确认通过**

```bash
cd ai-gateway/sidecar && npx vitest run test/unit/scanner.test.ts
```

预期：PASS，5（基础）+ 11（占位符 14 字节中 1..11 个有效切点）+ 2（误触发）= 18 个用例（实际切碎遍历从 1 到 placeholder.length-1 共 11 个迭代，所以 18 个）。

> Note: `[GUARD_IP_1]` 长度 12，切点 i=1..11，所以测试矩阵是 11 个；连同基础和误触发 = 18 个。

- [ ] **Step 5: Commit**

```bash
cd ai-gateway && git add sidecar/src/engine/scanner.ts sidecar/test/unit/scanner.test.ts
git commit -m "feat(sidecar): stream-safe placeholder scanner (state machine)"
```

---

## Task 7: SSE 帧切分与拼装 frames.ts

**Files:**
- Create: `sidecar/src/sse/frames.ts`
- Test: `sidecar/test/unit/frames.test.ts`

- [ ] **Step 1: 写失败测试**

文件路径：`sidecar/test/unit/frames.test.ts`

```typescript
import { describe, it, expect } from 'vitest';
import { FrameSplitter, formatDataFrame } from '../../src/sse/frames.js';

describe('FrameSplitter', () => {
  it('splits frames by double newline', () => {
    const s = new FrameSplitter();
    expect(s.push('data: 1\n\ndata: 2\n\n')).toEqual(['data: 1', 'data: 2']);
  });

  it('buffers incomplete trailing frame across pushes', () => {
    const s = new FrameSplitter();
    expect(s.push('data: he')).toEqual([]);
    expect(s.push('llo\n\ndata: ')).toEqual(['data: hello']);
    expect(s.push('world\n\n')).toEqual(['data: world']);
  });

  it('flush returns any remaining buffer as the last frame', () => {
    const s = new FrameSplitter();
    s.push('data: tail');
    expect(s.flush()).toEqual(['data: tail']);
    expect(s.flush()).toEqual([]);
  });

  it('decodes UTF-8 bytes streamed across chunks (surrogate-safe)', () => {
    const s = new FrameSplitter();
    const enc = new TextEncoder();
    const all = enc.encode('data: 你好\n\n');
    expect(s.pushBytes(all.slice(0, 9))).toEqual([]);
    expect(s.pushBytes(all.slice(9))).toEqual(['data: 你好']);
  });
});

describe('formatDataFrame', () => {
  it('serialises an object into an SSE data: line', () => {
    expect(formatDataFrame({ a: 1 })).toBe('data: {"a":1}\n\n');
  });
});
```

- [ ] **Step 2: 运行测试，确认失败**

```bash
cd ai-gateway/sidecar && npx vitest run test/unit/frames.test.ts
```

预期：FAIL，`Cannot find module`。

- [ ] **Step 3: 写最小实现**

文件路径：`sidecar/src/sse/frames.ts`

```typescript
export class FrameSplitter {
  private decoder = new TextDecoder('utf-8');
  private textBuffer = '';

  push(chunk: string): string[] {
    return this.consume(this.textBuffer + chunk);
  }

  pushBytes(bytes: Uint8Array): string[] {
    const text = this.decoder.decode(bytes, { stream: true });
    return this.consume(this.textBuffer + text);
  }

  flush(): string[] {
    const tail = this.textBuffer.trim();
    this.textBuffer = '';
    return tail ? [tail] : [];
  }

  private consume(combined: string): string[] {
    const parts = combined.split('\n\n');
    this.textBuffer = parts.pop() ?? '';
    return parts.filter((p) => p.length > 0);
  }
}

export function formatDataFrame(obj: unknown): string {
  return `data: ${JSON.stringify(obj)}\n\n`;
}
```

- [ ] **Step 4: 运行测试，确认通过**

```bash
cd ai-gateway/sidecar && npx vitest run test/unit/frames.test.ts
```

预期：PASS，5 个用例。

- [ ] **Step 5: Commit**

```bash
cd ai-gateway && git add sidecar/src/sse/frames.ts sidecar/test/unit/frames.test.ts
git commit -m "feat(sidecar): SSE frame splitter with UTF-8 streaming decoder"
```

---

## Task 8: OpenAI provider 路由

**Files:**
- Create: `sidecar/src/providers/openai.ts`

> 本任务先写实现（依赖 server 测试一并放到 Task 10 的集成测试覆盖）。

- [ ] **Step 1: 写实现**

文件路径：`sidecar/src/providers/openai.ts`

```typescript
import type { FastifyInstance, FastifyRequest, FastifyReply } from 'fastify';
import { randomUUID } from 'node:crypto';
import { createScopedContext, type ScopedContext } from '../context/scoped.js';
import { anonymize } from '../engine/anonymize.js';
import { Scanner } from '../engine/scanner.js';
import { FrameSplitter } from '../sse/frames.js';
import { builtinRules } from '../engine/rules.js';
import { log } from '../log.js';

interface ChatMessage {
  role: string;
  content: string | null;
}
interface ChatRequestBody {
  model: string;
  messages: ChatMessage[];
  stream?: boolean;
  [k: string]: unknown;
}

const UPSTREAM_URL = process.env.GUARD_OPENAI_UPSTREAM ?? 'https://api.openai.com/v1/chat/completions';

export function registerOpenAI(app: FastifyInstance): void {
  app.post('/v1/chat/completions', async (req, reply) => {
    const requestId = randomUUID();
    const ctx = createScopedContext(requestId);
    try {
      await handle(req, reply, ctx);
    } catch (err) {
      log.error('request.failed', { requestId, message: (err as Error).message });
      if (!reply.sent && !reply.raw.headersSent) {
        reply.code(502).send({ error: 'upstream_unreachable' });
      } else {
        reply.raw.end();
      }
      ctx.destroy();
    }
  });
}

async function handle(req: FastifyRequest, reply: FastifyReply, ctx: ScopedContext): Promise<void> {
  const body = req.body as ChatRequestBody | undefined;
  if (!body || !Array.isArray(body.messages)) {
    reply.code(400).send({ error: 'invalid_request' });
    ctx.destroy();
    return;
  }

  log.info('request.received', { requestId: ctx.requestId, path: '/v1/chat/completions', model: body.model });

  const contents = body.messages.map((m) => (typeof m.content === 'string' ? m.content : ''));
  const t0 = performance.now();
  const anonymised = anonymize(contents, builtinRules, ctx);
  const elapsed = performance.now() - t0;
  if (elapsed > 200) {
    log.error('anonymize.timeout', { requestId: ctx.requestId, ms: elapsed });
    reply.code(503).send({ error: 'anonymize_timeout' });
    ctx.destroy();
    return;
  }
  log.info('anonymize.done', { requestId: ctx.requestId, ms: elapsed, counters: Object.fromEntries(ctx.counters) });

  const cleanedBody: ChatRequestBody = {
    ...body,
    messages: body.messages.map((m, i) => ({ ...m, content: anonymised[i] ?? m.content })),
  };

  const ac = new AbortController();
  req.raw.on('close', () => ac.abort());

  let upstream: Response;
  try {
    upstream = await fetch(UPSTREAM_URL, {
      method: 'POST',
      signal: ac.signal,
      headers: {
        'Content-Type': 'application/json',
        ...(req.headers.authorization ? { Authorization: req.headers.authorization } : {}),
      },
      body: JSON.stringify(cleanedBody),
    });
  } catch (err) {
    log.error('upstream.fail', { requestId: ctx.requestId, message: (err as Error).message });
    reply.code(502).send({ error: 'upstream_unreachable' });
    ctx.destroy();
    return;
  }

  if (!body.stream || !upstream.body) {
    const text = await upstream.text();
    reply.code(upstream.status).header('Content-Type', upstream.headers.get('Content-Type') ?? 'application/json').send(text);
    ctx.destroy();
    return;
  }

  reply.hijack();
  reply.raw.writeHead(upstream.status, {
    'Content-Type': upstream.headers.get('Content-Type') ?? 'text/event-stream',
    'Cache-Control': 'no-cache, no-transform',
    Connection: 'keep-alive',
  });

  const splitter = new FrameSplitter();
  const scanner = new Scanner(ctx.deanonymizeMap);
  const reader = upstream.body.getReader();

  const finish = (): void => {
    const tail = splitter.flush();
    for (const frame of tail) writeFrame(reply.raw, frame, scanner);
    const buf = scanner.flush();
    if (buf) reply.raw.write(`data: ${JSON.stringify({ choices: [{ delta: { content: buf } }] })}\n\n`);
    if (!reply.raw.writableEnded) reply.raw.end();
    ctx.destroy();
    log.info('request.closed', { requestId: ctx.requestId });
  };

  try {
    while (true) {
      const { value, done } = await reader.read();
      if (done) break;
      const frames = splitter.pushBytes(value);
      for (const frame of frames) writeFrame(reply.raw, frame, scanner);
    }
    finish();
  } catch (err) {
    log.error('stream.fail', { requestId: ctx.requestId, message: (err as Error).message });
    if (!reply.raw.writableEnded) {
      reply.raw.write(`event: error\ndata: ${JSON.stringify({ error: 'stream_failed' })}\n\n`);
      reply.raw.end();
    }
    ctx.destroy();
  }
}

function writeFrame(raw: NodeJS.WritableStream, frame: string, scanner: Scanner): void {
  if (!frame.startsWith('data:')) {
    raw.write(frame + '\n\n');
    return;
  }
  const payload = frame.slice(5).trimStart();
  if (payload === '[DONE]') {
    raw.write('data: [DONE]\n\n');
    return;
  }
  let json: { choices?: Array<{ delta?: { content?: string } }> };
  try {
    json = JSON.parse(payload);
  } catch {
    raw.write(frame + '\n\n');
    return;
  }
  const choices = json.choices ?? [];
  for (const c of choices) {
    const content = c.delta?.content;
    if (typeof content === 'string') {
      c.delta!.content = scanner.push(content);
    }
  }
  raw.write(`data: ${JSON.stringify(json)}\n\n`);
}
```

- [ ] **Step 2: 静态类型检查**

```bash
cd ai-gateway/sidecar && npx tsc --noEmit
```

预期：无错误。

- [ ] **Step 3: Commit**

```bash
cd ai-gateway && git add sidecar/src/providers/openai.ts
git commit -m "feat(sidecar): OpenAI provider with anonymise + SSE restore pipeline"
```

---

## Task 9: server.ts 与端口选择

**Files:**
- Create: `sidecar/src/server.ts`

- [ ] **Step 1: 写实现**

文件路径：`sidecar/src/server.ts`

```typescript
import Fastify from 'fastify';
import { registerOpenAI } from './providers/openai.js';
import { log } from './log.js';

const HOST = '127.0.0.1';
const PORT_RANGE_START = 9999;
const PORT_RANGE_END = 10010;

async function tryListen(app: ReturnType<typeof Fastify>): Promise<number> {
  for (let port = PORT_RANGE_START; port <= PORT_RANGE_END; port++) {
    try {
      await app.listen({ host: HOST, port });
      return port;
    } catch (err) {
      const code = (err as NodeJS.ErrnoException).code;
      if (code !== 'EADDRINUSE') throw err;
    }
  }
  throw new Error(`No free port in range ${PORT_RANGE_START}-${PORT_RANGE_END}`);
}

export async function startServer(): Promise<{ app: ReturnType<typeof Fastify>; port: number }> {
  const app = Fastify({ logger: false, bodyLimit: 32 * 1024 * 1024 });

  app.get('/healthz', async () => ({ ok: true }));
  registerOpenAI(app);

  const port = await tryListen(app);
  log.info('server.listening', { host: HOST, port });
  return { app, port };
}

const isMain = import.meta.url === `file://${process.argv[1]}`;
if (isMain) {
  startServer().catch((err) => {
    log.error('server.start_failed', { message: (err as Error).message });
    process.exit(1);
  });
}
```

- [ ] **Step 2: 静态类型检查与冒烟测试**

```bash
cd ai-gateway/sidecar && npx tsc --noEmit
```

预期：无错误。

冒烟启动（后台运行，5 秒后 kill）：

```bash
cd ai-gateway/sidecar && (npx tsx src/server.ts & echo $! > /tmp/guard.pid; sleep 2; curl -s http://127.0.0.1:9999/healthz; kill $(cat /tmp/guard.pid))
```

预期：输出 `{"ok":true}`。

- [ ] **Step 3: Commit**

```bash
cd ai-gateway && git add sidecar/src/server.ts
git commit -m "feat(sidecar): Fastify server with port auto-selection"
```

---

## Task 10: Mock 上游 + 集成测试 happy path

**Files:**
- Create: `sidecar/test/integration/mock-upstream.ts`
- Create: `sidecar/test/integration/openai-happy-path.test.ts`

- [ ] **Step 1: 写 mock 上游**

文件路径：`sidecar/test/integration/mock-upstream.ts`

```typescript
import Fastify from 'fastify';

export interface MockUpstream {
  url: string;
  receivedBodies: unknown[];
  setReply(reply: { status: number; chunks: string[]; contentType?: string }): void;
  close(): Promise<void>;
}

export async function startMockUpstream(): Promise<MockUpstream> {
  const app = Fastify({ logger: false });
  const received: unknown[] = [];
  let nextReply: { status: number; chunks: string[]; contentType?: string } = {
    status: 200,
    chunks: ['data: {"choices":[{"delta":{"content":"hi"}}]}\n\n', 'data: [DONE]\n\n'],
  };

  app.post('/v1/chat/completions', async (req, reply) => {
    received.push(req.body);
    reply.hijack();
    reply.raw.writeHead(nextReply.status, {
      'Content-Type': nextReply.contentType ?? 'text/event-stream',
      'Cache-Control': 'no-cache',
    });
    for (const chunk of nextReply.chunks) {
      reply.raw.write(chunk);
      await new Promise((r) => setTimeout(r, 5));
    }
    reply.raw.end();
  });

  await app.listen({ host: '127.0.0.1', port: 0 });
  const addr = app.server.address();
  if (!addr || typeof addr === 'string') throw new Error('no address');
  const url = `http://127.0.0.1:${addr.port}/v1/chat/completions`;

  return {
    url,
    receivedBodies: received,
    setReply: (r) => {
      nextReply = r;
    },
    close: () => app.close(),
  };
}
```

- [ ] **Step 2: 写 happy-path 集成测试**

文件路径：`sidecar/test/integration/openai-happy-path.test.ts`

```typescript
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { startMockUpstream, type MockUpstream } from './mock-upstream.js';
import { startServer } from '../../src/server.js';
import type { FastifyInstance } from 'fastify';

let upstream: MockUpstream;
let server: { app: FastifyInstance; port: number };

beforeAll(async () => {
  upstream = await startMockUpstream();
  process.env.GUARD_OPENAI_UPSTREAM = upstream.url;
  server = await startServer();
});

afterAll(async () => {
  await server.app.close();
  await upstream.close();
});

async function collectSSE(res: Response): Promise<string> {
  const reader = res.body!.getReader();
  const dec = new TextDecoder();
  let full = '';
  while (true) {
    const { value, done } = await reader.read();
    if (done) break;
    full += dec.decode(value, { stream: true });
  }
  return full;
}

describe('OpenAI happy path', () => {
  it('redacts request and restores response across full round-trip', async () => {
    upstream.setReply({
      status: 200,
      chunks: [
        'data: {"choices":[{"delta":{"content":"server "}}]}\n\n',
        'data: {"choices":[{"delta":{"content":"is [GUARD_IP_1]"}}]}\n\n',
        'data: {"choices":[{"delta":{"content":" thanks"}}]}\n\n',
        'data: [DONE]\n\n',
      ],
    });

    const res = await fetch(`http://127.0.0.1:${server.port}/v1/chat/completions`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', Authorization: 'Bearer sk-test' },
      body: JSON.stringify({
        model: 'gpt-4',
        messages: [{ role: 'user', content: 'what is at 192.168.1.100?' }],
        stream: true,
      }),
    });

    expect(res.status).toBe(200);
    const text = await collectSSE(res);
    expect(text).toContain('192.168.1.100');
    expect(text).not.toContain('[GUARD_IP_1]');

    const sentBody = upstream.receivedBodies[0] as { messages: Array<{ content: string }> };
    expect(sentBody.messages[0]!.content).toBe('what is at [GUARD_IP_1]?');
  });

  it('returns 400 for non-JSON body shape', async () => {
    const res = await fetch(`http://127.0.0.1:${server.port}/v1/chat/completions`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ wrong: 'shape' }),
    });
    expect(res.status).toBe(400);
  });

  it('healthz reports ok', async () => {
    const res = await fetch(`http://127.0.0.1:${server.port}/healthz`);
    expect(await res.json()).toEqual({ ok: true });
  });
});
```

- [ ] **Step 3: 运行集成测试**

```bash
cd ai-gateway/sidecar && npx vitest run test/integration/openai-happy-path.test.ts
```

预期：PASS，3 个用例。

- [ ] **Step 4: Commit**

```bash
cd ai-gateway && git add sidecar/test/integration/mock-upstream.ts sidecar/test/integration/openai-happy-path.test.ts
git commit -m "test(sidecar): integration happy-path with mock upstream"
```

---

## Task 11: 集成测试 — 占位符跨 chunk 切碎

**Files:**
- Create: `sidecar/test/integration/openai-stream-cut.test.ts`

- [ ] **Step 1: 写测试**

文件路径：`sidecar/test/integration/openai-stream-cut.test.ts`

```typescript
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { startMockUpstream, type MockUpstream } from './mock-upstream.js';
import { startServer } from '../../src/server.js';
import type { FastifyInstance } from 'fastify';

let upstream: MockUpstream;
let server: { app: FastifyInstance; port: number };

beforeAll(async () => {
  upstream = await startMockUpstream();
  process.env.GUARD_OPENAI_UPSTREAM = upstream.url;
  server = await startServer();
});
afterAll(async () => {
  await server.app.close();
  await upstream.close();
});

async function readAllFrames(res: Response): Promise<string[]> {
  const reader = res.body!.getReader();
  const dec = new TextDecoder();
  const all: string[] = [];
  while (true) {
    const { value, done } = await reader.read();
    if (done) break;
    all.push(dec.decode(value, { stream: true }));
  }
  return all;
}

describe('OpenAI stream cut across chunks', () => {
  it('reassembles a placeholder broken into 4 SSE frames', async () => {
    upstream.setReply({
      status: 200,
      chunks: [
        'data: {"choices":[{"delta":{"content":"the host is "}}]}\n\n',
        'data: {"choices":[{"delta":{"content":"[GUA"}}]}\n\n',
        'data: {"choices":[{"delta":{"content":"RD_IP"}}]}\n\n',
        'data: {"choices":[{"delta":{"content":"_1]"}}]}\n\n',
        'data: {"choices":[{"delta":{"content":" done"}}]}\n\n',
        'data: [DONE]\n\n',
      ],
    });

    const res = await fetch(`http://127.0.0.1:${server.port}/v1/chat/completions`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', Authorization: 'Bearer sk-test' },
      body: JSON.stringify({
        model: 'gpt-4',
        messages: [{ role: 'user', content: 'host 192.168.1.100' }],
        stream: true,
      }),
    });

    const chunks = await readAllFrames(res);
    const concatenated = chunks.join('');

    expect(concatenated).toContain('192.168.1.100');
    for (const chunk of chunks) {
      expect(chunk.includes('[GUARD_') && !chunk.includes(']')).toBe(false);
      expect(chunk.includes('[GUA')).toBe(false);
    }
  });
});
```

- [ ] **Step 2: 运行测试**

```bash
cd ai-gateway/sidecar && npx vitest run test/integration/openai-stream-cut.test.ts
```

预期：PASS，1 个用例。

- [ ] **Step 3: Commit**

```bash
cd ai-gateway && git add sidecar/test/integration/openai-stream-cut.test.ts
git commit -m "test(sidecar): integration test for placeholder split across SSE chunks"
```

---

## Task 12: 集成测试 — abort 与 fail-closed

**Files:**
- Create: `sidecar/test/integration/openai-abort.test.ts`
- Create: `sidecar/test/integration/openai-fail-closed.test.ts`

- [ ] **Step 1: 写 abort 测试**

文件路径：`sidecar/test/integration/openai-abort.test.ts`

```typescript
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { startMockUpstream, type MockUpstream } from './mock-upstream.js';
import { startServer } from '../../src/server.js';
import type { FastifyInstance } from 'fastify';

let upstream: MockUpstream;
let server: { app: FastifyInstance; port: number };

beforeAll(async () => {
  upstream = await startMockUpstream();
  process.env.GUARD_OPENAI_UPSTREAM = upstream.url;
  server = await startServer();
});
afterAll(async () => {
  await server.app.close();
  await upstream.close();
});

describe('OpenAI abort', () => {
  it('client abort closes stream without throwing', async () => {
    const long: string[] = [];
    for (let i = 0; i < 50; i++) {
      long.push(`data: {"choices":[{"delta":{"content":"chunk ${i} "}}]}\n\n`);
    }
    long.push('data: [DONE]\n\n');
    upstream.setReply({ status: 200, chunks: long });

    const ac = new AbortController();
    const promise = fetch(`http://127.0.0.1:${server.port}/v1/chat/completions`, {
      method: 'POST',
      signal: ac.signal,
      headers: { 'Content-Type': 'application/json', Authorization: 'Bearer sk-test' },
      body: JSON.stringify({ model: 'gpt-4', messages: [{ role: 'user', content: 'hi' }], stream: true }),
    });

    setTimeout(() => ac.abort(), 30);
    await expect(promise.then((r) => r.text())).rejects.toThrow();

    await new Promise((r) => setTimeout(r, 50));
    const health = await fetch(`http://127.0.0.1:${server.port}/healthz`);
    expect(health.status).toBe(200);
  });
});
```

- [ ] **Step 2: 写 fail-closed 测试**

文件路径：`sidecar/test/integration/openai-fail-closed.test.ts`

```typescript
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { startMockUpstream, type MockUpstream } from './mock-upstream.js';
import { startServer } from '../../src/server.js';
import type { FastifyInstance } from 'fastify';

let upstream: MockUpstream;
let server: { app: FastifyInstance; port: number };

beforeAll(async () => {
  upstream = await startMockUpstream();
  process.env.GUARD_OPENAI_UPSTREAM = upstream.url;
  server = await startServer();
});
afterAll(async () => {
  await server.app.close();
  await upstream.close();
});

describe('OpenAI fail-closed', () => {
  it('502 from upstream is passed through as 502 body', async () => {
    upstream.setReply({
      status: 502,
      contentType: 'application/json',
      chunks: ['{"error":"upstream_bad"}'],
    });

    const res = await fetch(`http://127.0.0.1:${server.port}/v1/chat/completions`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', Authorization: 'Bearer sk-test' },
      body: JSON.stringify({ model: 'gpt-4', messages: [{ role: 'user', content: 'x' }], stream: false }),
    });

    expect(res.status).toBe(502);
  });

  it('unreachable upstream returns 502', async () => {
    process.env.GUARD_OPENAI_UPSTREAM = 'http://127.0.0.1:1/v1/chat/completions';
    await server.app.close();
    server = await startServer();

    const res = await fetch(`http://127.0.0.1:${server.port}/v1/chat/completions`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', Authorization: 'Bearer sk-test' },
      body: JSON.stringify({ model: 'gpt-4', messages: [{ role: 'user', content: 'x' }], stream: true }),
    });

    expect(res.status).toBe(502);

    process.env.GUARD_OPENAI_UPSTREAM = upstream.url;
    await server.app.close();
    server = await startServer();
  });
});
```

- [ ] **Step 3: 运行两个测试**

```bash
cd ai-gateway/sidecar && npx vitest run test/integration/openai-abort.test.ts test/integration/openai-fail-closed.test.ts
```

预期：PASS，3 个用例。

- [ ] **Step 4: Commit**

```bash
cd ai-gateway && git add sidecar/test/integration/openai-abort.test.ts sidecar/test/integration/openai-fail-closed.test.ts
git commit -m "test(sidecar): integration tests for abort and fail-closed"
```

---

## Task 13: curl 验证脚本 + Phase 1 验收门

**Files:**
- Create: `scripts/verify-curl.sh`
- Modify: `ai-gateway/README.md` (append Phase 1 verification section)

- [ ] **Step 1: 写 curl 验证脚本**

文件路径：`scripts/verify-curl.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

# Phase 1 manual verification: 起 Sidecar -> 发一个含敏感数据的非流请求到一个回显 echo 上游 -> 断言上游收到占位符。

GUARD_PORT="${GUARD_PORT:-9999}"

cleanup() {
  if [[ -n "${SIDECAR_PID:-}" ]] && kill -0 "$SIDECAR_PID" 2>/dev/null; then kill "$SIDECAR_PID"; fi
  if [[ -n "${ECHO_PID:-}" ]] && kill -0 "$ECHO_PID" 2>/dev/null; then kill "$ECHO_PID"; fi
}
trap cleanup EXIT

cat > /tmp/echo-upstream.mjs <<'EOF'
import http from 'node:http';
const srv = http.createServer((req, res) => {
  let body = '';
  req.on('data', (c) => (body += c));
  req.on('end', () => {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ received: JSON.parse(body) }));
  });
});
srv.listen(8765, '127.0.0.1');
EOF

node /tmp/echo-upstream.mjs &
ECHO_PID=$!
sleep 0.5

export GUARD_OPENAI_UPSTREAM="http://127.0.0.1:8765/v1/chat/completions"
( cd "$(dirname "$0")/../sidecar" && npx tsx src/server.ts ) &
SIDECAR_PID=$!
sleep 1

RESP=$(curl -s -X POST "http://127.0.0.1:${GUARD_PORT}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-test" \
  -d '{"model":"gpt-4","stream":false,"messages":[{"role":"user","content":"connect to 192.168.1.100 with sk-abcdefghijklmnopqrstuvwxyzABCDEFGHIJ"}]}')

echo "Upstream saw:"
echo "$RESP" | node -e "let s='';process.stdin.on('data',c=>s+=c);process.stdin.on('end',()=>{console.log(JSON.parse(s).received.messages[0].content)})"

if echo "$RESP" | grep -q "192.168.1.100"; then
  echo "FAIL: 上游收到了真实 IP！" >&2
  exit 1
fi
if echo "$RESP" | grep -q "\[GUARD_IP_"; then
  echo "OK: 上游只收到占位符。"
else
  echo "FAIL: 占位符缺失。" >&2
  exit 1
fi
```

- [ ] **Step 2: 让脚本可执行并运行**

```bash
cd ai-gateway && chmod +x scripts/verify-curl.sh && bash scripts/verify-curl.sh
```

预期：输出
```
Upstream saw:
connect to [GUARD_IP_1] with [GUARD_SECRET_1]
OK: 上游只收到占位符。
```

- [ ] **Step 3: 更新 README**

修改文件：`ai-gateway/README.md`（保留现有内容，追加以下章节）

```markdown

## Phase 1 验证

### 全套自动化测试

```bash
cd ai-gateway && npm test
```

### 手工 curl 验证

```bash
cd ai-gateway && bash scripts/verify-curl.sh
```

期望输出：上游收到的 `messages[0].content` 仅包含 `[GUARD_IP_1]` 与 `[GUARD_SECRET_1]` 占位符，**不包含**任何真实敏感数据。

### Phase 1 验收门

- 所有 unit 测试通过（log / scoped / rules / anonymize / scanner / frames）
- 所有 integration 测试通过（happy-path / stream-cut / abort / fail-closed）
- `anonymize` 50,000 字基准 < 5ms（NFR-5.1）
- `verify-curl.sh` 输出 `OK: 上游只收到占位符。`
```

- [ ] **Step 4: 跑全部测试做最后核对**

```bash
cd ai-gateway && npm test
```

预期：所有 unit + integration 测试全绿。

- [ ] **Step 5: Commit**

```bash
cd ai-gateway && git add scripts/verify-curl.sh ai-gateway/README.md
git commit -m "chore: add Phase 1 manual verification script and README section"
```

---

## Phase 1 验收清单

- [ ] `npm test` 全绿（unit + integration）
- [ ] `npm run bench` 中 `anonymize 50k chars` 单次 < 5ms
- [ ] `scripts/verify-curl.sh` 输出 OK
- [ ] 设计文档 § 6.1 列出的 5 项里程碑全部完成
- [ ] 无 GUI、未引入 Anthropic / DB / PII / 自定义规则（这些进入 Phase 2/3）
