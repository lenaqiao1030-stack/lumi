# M0-01 手写 Agent loop 骨架

- 状态:草稿(待你冻结 G3)
- 风险等级:R2(定义工具权限与 Hook 的骨架,属于治理代码;合并前由你逐项审阅)
- 对应设计章节:docs/design/lumi.md 第十章(Agent loop、Tool System、Governance)、第十四章(M0)
- 依赖的前置 spec:无

## 1. 背景与目标

实现一个不依赖任何 Agent SDK 的最小 Agent loop:调用模型 → 解析工具调用 → 校验输入 → PreToolUse Hook → 执行 → PostToolUse Hook → 结果回填 → 继续,直到模型不再调用工具。本 spec 只做引擎骨架,使用假模型测试,不接真实模型,不含具体工具。

目标是把「模型、工具、Hook、权限」之间的接口定对,后续所有能力(搜索、预算、事件、权限分级)都建立在它上面。

## 2. 范围

做:

- 核心类型:消息、工具定义、工具输出、模型客户端接口、Hook 接口、运行结果
- `runAgent` 循环引擎(含最基础的 `maxTurns` 上限)
- 工具注册与按名查找
- 假模型 `FakeModelClient`(仅测试用)

不做:

- 真实模型客户端、命令行入口(后续 spec)
- 预算(token、工具调用数、时长)、取消、事件流(M0-03)
- 权限分级判定与确认流程(后续 spec;本 spec 只提供 Hook 接口)
- 任何具体工具

## 3. 接口与数据结构

```ts
type SideEffect = 'read' | 'write_internal' | 'write_external' | 'destructive';
type Trust = 'trusted' | 'untrusted';

interface ToolOutput { content: string; trust?: Trust; isError?: boolean }

interface ToolContext { runId: string; signal: AbortSignal }

interface ToolDefinition<I = unknown> {
  name: string;
  description: string;
  inputSchema: z.ZodType<I>;
  sideEffect: SideEffect;
  idempotent: boolean;
  concurrencySafe: boolean;
  timeoutMs: number;
  execute(input: I, ctx: ToolContext): Promise<ToolOutput>;
}

type ToolCall = { id: string; name: string; input: unknown };

type Message =
  | { role: 'user'; content: string }
  | { role: 'assistant'; content: string; toolCalls: ToolCall[] }
  | { role: 'tool'; callId: string; name: string; content: string; trust: Trust; isError: boolean };

interface ModelResponse {
  text: string;
  toolCalls: ToolCall[];
  usage: { inputTokens: number; outputTokens: number };
}
interface ModelClient {
  complete(req: { system: string; messages: Message[]; tools: ToolDefinition[]; signal: AbortSignal }): Promise<ModelResponse>;
}

type PreToolDecision =
  | { decision: 'allow'; input?: unknown }
  | { decision: 'deny'; reason: string };
interface Hooks {
  preToolUse?(call: ToolCall, tool: ToolDefinition, ctx: ToolContext): Promise<PreToolDecision>;
  postToolUse?(call: ToolCall, output: ToolOutput, ctx: ToolContext): Promise<void>;
}

type RunStatus = 'completed' | 'max_turns' | 'error';
interface RunResult { status: RunStatus; messages: Message[]; finalText?: string; turns: number; error?: string }

function runAgent(opts: {
  model: ModelClient; system: string; tools: ToolDefinition[];
  hooks?: Hooks; input: string; maxTurns: number;
}): Promise<RunResult>;
```

设计决定:

- 同一轮的多个工具调用:`concurrencySafe` 为 true 的并行执行(并发上限固定常量 4),其余串行;回填的 tool 消息顺序与模型给出的调用顺序一致
- 工具未显式给出 `trust` 时,按 `untrusted` 处理
- `preToolUse` 抛异常按 deny 处理(fail-closed);`postToolUse` 抛异常只记录,不影响循环

## 4. 验收标准

- AC-1:模型返回不含工具调用的文本时,循环在该轮以 `completed` 结束,`finalText` 等于该文本
- AC-2:模型请求工具时,按 `inputSchema` 解析输入并执行工具,结果以 tool 消息回填,下一轮的模型请求中能看到该消息
- AC-3:同一轮多个调用,`concurrencySafe` 的并行、其余串行;回填顺序与调用顺序一致
- AC-4:未知工具名:回填 `isError: true` 的 tool 消息,循环继续,不抛异常
- AC-5:输入不满足 `inputSchema`:回填包含校验失败原因的错误消息,工具不被执行
- AC-6:工具抛异常或超过 `timeoutMs`:捕获并回填 `isError: true`,循环继续
- AC-7:`preToolUse` 返回 deny:工具不执行,回填包含 reason 的拒绝消息;返回 allow 并带 `input` 时,使用改写后的输入,且改写后的输入必须再次通过 `inputSchema` 校验
- AC-8:`preToolUse` 抛异常按 deny 处理;`postToolUse` 在每次工具执行(含失败)后被调用,其抛出的异常不影响循环
- AC-9:tool 消息保留工具输出的 `trust`;工具未给出时为 `untrusted`
- AC-10:达到 `maxTurns` 仍未结束:返回 `status: 'max_turns'` 与已有 messages,不抛异常
- AC-11:`ModelClient.complete` 抛异常:返回 `status: 'error'`、已有 messages 与错误信息,不向上抛出

## 5. 测试要求

- 全部使用 `FakeModelClient` 与假工具,禁止真实模型与真实网络
- 每条 AC 至少一个测试,测试名以 `AC-n:` 开头
- AC-3 用可控的延迟与完成顺序证明「并行」与「顺序回填」
- AC-6 的超时测试使用假定时器,不真实等待

## 6. 涉及文件与依赖

允许新增或修改:

- `src/agent/types.ts`、`src/agent/loop.ts`、`src/agent/hooks.ts`、`src/agent/index.ts`
- `tests/agent/loop.test.ts`、`tests/helpers/fake-model.ts`

允许新增的依赖:`zod`(其余一律不加)。

明确禁止修改:`.github/`、`.cursor/`、`AGENTS.md`、`src/governance/`。

## 7. 非功能要求

- 引擎本身不做任何 I/O、不读环境变量、不使用全局可变状态
- 无 `any`;对外导出类型完整

## 8. 发布、回滚与线上核实判据

不适用(无线上环境)。

## 9. 完成定义

- 所有 AC 有对应测试且通过;`pnpm lint`、`pnpm typecheck`、`pnpm test` 通过
- 变更约 400 行以内;若超出,拆为 M0-01a(类型与循环核心:AC-1、2、4、5、6、10、11)与 M0-01b(并行、Hook、trust:AC-3、7、8、9)两个 PR
- 回执完整

## 10. 开放问题

- 无
