# CLAUDE.md

Claude Code 反向工程复现项目。运行时为 **Bun**（非 Node.js），TypeScript strict 模式，monorepo。

---

## 硬性规则（违反则 CI 失败）

1. **每次修改后必须运行 `bun run precheck`**（typecheck + lint fix + test），零错误才算完成。
2. **`feature()` 只能用在 `if` 条件或三元表达式中**，不能赋值给变量，不能在 `&&` 链里，不能在箭头函数体里。
   ```ts
   if (feature('X')) { ... }        // ✅
   feature('X') ? a : b             // ✅
   const x = feature('X')           // ❌ Bun 构建器无法静态消除
   arr.filter(() => feature('X'))   // ❌
   ```
3. **生产代码禁止 `as any`**。用 `as unknown as T` 双重断言，或补 interface。测试文件的 mock 数据可以用。
4. **Commit 使用 Conventional Commits**：`feat:` / `fix:` / `docs:` / `chore:` / `refactor:`。

---

## 常用命令

```bash
bun install              # 安装依赖
bun run dev              # 开发模式（注入所有 feature flags）
bun run precheck         # ✅ 任务完成前必跑：typecheck + lint:fix + test
bun test <file>          # 运行单个测试文件
bun run build            # 构建（Bun，代码分割）
bun run build:vite       # 构建（Vite 备用管道）
bun run lint:fix         # 自动修复 lint 问题
```

---

## 容易踩的坑（非显而易见）

| 坑 | 正确做法 |
|----|---------|
| Ink 框架位置 | `packages/@ant/ink/`，**不是** `src/ink/`（该目录不存在） |
| React Compiler 代码 | 组件里大量 `const $ = _c(N)` 是反编译产物，**正常，不要删** |
| `src/*` 路径别名 | tsconfig 已映射，`import from 'src/utils/...'` 是合法的 |
| MACRO 常量 | 版本号等只改 `scripts/defines.ts`，dev 和 build 都从这里注入 |
| `@ts-expect-error` | 只在下方代码确实有类型错误时保留。类型修复后若 directive 变为 unused（TS2578）就删掉 |
| `tsc` vs Biome 冲突 | 属性声明需要但 biome 报 `noUnusedPrivateClassMembers`，用 `// biome-ignore` 抑制，不要删声明 |
| tool_use_id 配对 | API 要求每个 tool_use 都有 tool_result，用 `ensureToolResultPairing()` |
| prompt cache 一致性 | 不要修改已发出的消息对象字段（破坏 cache），只追加新字段时才 clone |
| 写工具并发 | FileEdit/Bash/FileWrite 必须串行，否则 patch 竞态。`partitionToolCalls` 已处理 |
| mock 污染 | Bun `mock.module` 是进程全局的，见下方测试规范 |

---

## 架构速查（指针，不是描述）

```
核心链路：
  src/entrypoints/cli.tsx      → 入口，快速路径分流
  src/main.tsx                 → Commander.js CLI 定义（~5674 行）
  src/screens/REPL.tsx         → 交互界面（onSubmit → onQuery → onQueryImpl）
  src/query.ts                 → Agentic Loop（queryLoop while(true)）
  src/QueryEngine.ts           → 会话生命周期管理
  src/services/api/claude.ts   → 流式 API 客户端

工具系统：
  src/Tool.ts                  → Tool 接口定义
  src/tools.ts                 → 工具注册表（条件装载）
  packages/builtin-tools/src/tools/  → 60 个工具实现

类型：
  src/types/message.ts         → 消息类型（UserMessage / AssistantMessage / ...)
  src/types/permissions.ts     → PermissionMode / PermissionResult
  src/Tool.ts                  → ToolUseContext（最重要的上下文类型）

状态：
  src/state/AppState.tsx       → UI 状态类型
  src/state/store.ts           → createStore（类 Zustand）
  src/bootstrap/state.ts       → 模块级 singleton（sessionId / CWD / token counts）

System Prompt 装配：
  src/constants/prompts.ts     → getSystemPrompt()，所有片段拼接
  src/constants/system.ts      → "You are Claude Code..." 身份前缀
  src/context.ts               → getUserContext() / getSystemContext()
  src/utils/claudemd.ts        → CLAUDE.md 文件层级发现

多 Provider：
  src/services/api/openai/     → CLAUDE_CODE_USE_OPENAI=1
  src/services/api/gemini/     → CLAUDE_CODE_USE_GEMINI=1
  src/services/api/grok/       → CLAUDE_CODE_USE_GROK=1
  src/utils/model/providers.ts → Provider 选择（modelType > 环境变量 > firstParty）
```

---

## Feature Flag 系统

- **声明**：`import { feature } from 'bun:bundle'`（Bun 内置，不要替换）
- **运行时启用**：`FEATURE_<NAME>=1 bun run dev`
- **Dev 模式**：全部 feature 默认启用（`scripts/dev.ts`）
- **Build 默认启用列表**：见 `build.ts` 的 `DEFAULT_BUILD_FEATURES`
- **新增功能**：用标准 `feature('NEW_FLAG')` 模式，不要绕过直接 import

---

## 测试规范

### Mock 使用原则

**只 mock 有副作用的依赖链（网络、文件系统、随机数），不 mock 纯函数。**

必须 mock 的模块（因为 `bootstrap/state.ts` 有模块级副作用）：
`log.ts`、`debug.ts`、`bun:bundle`、`settings/settings.js`、`config.ts`、`auth.ts`

**使用共享 mock，不要内联定义：**
```ts
import { logMock } from "../../../tests/mocks/log";
mock.module("src/utils/log.ts", logMock);
```

### ⚠️ Mock 污染（最高优先级规则）

**Bun 的 `mock.module` 是进程全局的（last-write-wins）**，同一进程中所有测试文件共享同一模块注册表。

**核心规则：不要 mock 被测模块的上层业务模块。** 应 mock 底层 HTTP 层（axios）而非业务模块。

如果目录下同时有 `launch*.test.ts`（集成）和 `api.test.ts`（单元），集成测试必须 mock axios，否则会污染 `api.test.ts` 的真实模块。

排查步骤：
1. `bun test path/to/suspect.test.ts` 单独跑是否通过
2. `bun test path/to/__tests__/` 和其他文件一起跑是否失败
3. 检查 `mock.module` 的 specifier 是否与其他测试的 import 路径解析到同一模块

### 类型规范

```ts
as any              // ❌ 生产代码禁止
as unknown as T     // ✅ 双重断言
Record<string, unknown>  // ✅ 未知结构对象
```

---

## 构建说明

- **为什么必须代码分割**：Bun/JSC 全量解析单文件 17MB 产物会导致 RSS 暴涨至 1GB。分割为 600+ chunk 后 `--version` 仅需 35MB RSS。
- **两套构建管道**：`bun run build`（Bun 原生）和 `bun run build:vite`（Vite），产物结构略有差异（chunks 目录位置不同）。
- **Vendor 路径**：`src/utils/distRoot.ts` 提供统一的 `distRoot()` 函数，所有工具通过它定位 `dist/vendor/`，不要内联 `import.meta.url` 路径推算。
- **Node.js 兼容**：`build.ts` 自动后处理 `import.meta.require`，产物可以直接用 `node dist/cli.js` 运行。

---

## 深入参考

| 文档 | 内容 |
|------|------|
| `docs/learning-guide.md` | 14 阶段学习路线图（以复现为目标）|
| `docs/message-flow-deep-dive.md` | 完整消息链路深度解析（源码级，1200+ 行）|
| `docs/conversation/the-loop.mdx` | Agentic Loop 源码级解析 |
| `docs/conversation/streaming.mdx` | 流式响应机制 |
| `.impeccable.md` | Web UI 设计上下文（RCS 控制面板 / 文档站设计时必读）|
