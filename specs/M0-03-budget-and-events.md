# M0-03 预算、超时、取消与事件流

- 状态:草稿(待你冻结 G3)
- 风险等级:R1
- 对应设计章节:docs/design/lumi.md 第十章(Agent loop:终止条件写死、事件流)、第十四章(M0)
- 依赖的前置 spec:M0-01

## 1. 背景与目标

给 Agent loop 加上「写死在引擎里」的终止条件(轮数、工具调用数、token、时长)、取消能力,以及结构化事件流。事件流是后续「现场出报告」等待态(正在搜索…正在读…)与审计日志的数据来源。

## 2. 范围

做:

- `Limits` 与校验、各类终止条件及对应 `RunStatus`
- 通过 `AbortSignal` 的超时与外部取消,并级联到模型调用与工具调用
- `AgentEvent` 类型、事件接收器、JSONL 输出器
- 敏感信息脱敏

不做:

- 真实模型客户端、命令行入口、SSE 传输
- 美元成本换算
- `searching` / `reading` 这类面向用户的活动映射(由前端或后续 spec 基于事件做)
- `waiting_confirm` 事件(仅在类型中预留,本 spec 不产生)

## 3. 接口与数据结构

```ts
interface Limits {
  maxTurns: number;
  maxToolCalls: number;
  maxTotalTokens: number;   // 累计 input + output
  maxDurationMs: number;
}

type RunStatus =
  | 'completed' | 'max_turns' | 'max_tool_calls'
  | 'budget_exhausted' | 'timeout' | 'aborted' | 'error';

type EventBase = { runId: string; seq: number; ts: string };
type AgentEvent = EventBase & (
  | { type: 'run_started'; limits: Limits }
  | { type: 'turn_started'; turn: number }
  | { type: 'assistant_message'; textLength: number; toolCallCount: number; usage: { inputTokens: number; outputTokens: number } }
  | { type: 'tool_call_started'; callId: string; name: string; sideEffect: SideEffect; inputSummary: string }
  | { type: 'tool_call_finished'; callId: string; name: string; ok: boolean; durationMs: number; trust: Trust; outputBytes: number }
  | { type: 'tool_call_denied'; callId: string; name: string; reason: string }
  | { type: 'run_finished'; status: RunStatus; turns: number; toolCalls: number; usage: { inputTokens: number; outputTokens: number }; durationMs: number }
);

interface EventSink { emit(e: AgentEvent): void }
function createJsonlSink(write: (line: string) => void): EventSink;
```

`runAgent` 新增参数:`limits: Limits`、`signal?: AbortSignal`、`sink?: EventSink`、`clock?: { now(): Date }`、`newRunId?: () => string`(后两者用于测试的确定性)。

设计决定:

- 模型调用与工具调用都会收到由「超时」与「外部取消」合并而成的 `AbortSignal`
- token 预算在**下一次模型调用之前**检查;单次调用自身可能超出预算,这是已知限制,写入代码注释
- 工具调用数在**执行之前**检查,超限的调用不会被执行
- `inputSummary` 截断到 200 字符;键名匹配 `key|token|secret|password|authorization`(不区分大小写)的值替换为 `[redacted]`
- 事件中不包含工具返回的正文,只记录字节数

## 4. 验收标准

- AC-1:`Limits` 用 zod 校验:全部为正整数,有默认值与合理上限;非法配置在启动时抛出明确的配置错误
- AC-2:超过 `maxTurns`:以 `max_turns` 结束,并返回已有 messages
- AC-3:超过 `maxToolCalls`:超限的调用不执行,以 `max_tool_calls` 结束
- AC-4:累计 token 超过 `maxTotalTokens`:在下一次模型调用前以 `budget_exhausted` 结束
- AC-5:超过 `maxDurationMs`:以 `timeout` 结束,进行中的模型调用与工具调用收到 abort 信号
- AC-6:外部 `AbortSignal` 触发:以 `aborted` 结束,级联取消,没有悬挂的 Promise
- AC-7:每次运行都产生事件序列:`run_started` 为第一个,`run_finished` 为最后一个且恰好一个(包括 error 在内的任何终止方式);同一 `runId`;`seq` 从 1 开始严格递增
- AC-8:每个 `tool_call_started` 之后都有对应的 `tool_call_finished` 或 `tool_call_denied`;并行工具的事件允许交错,但各自成对
- AC-9:`inputSummary` 被截断并脱敏;事件中不含工具返回的正文
- AC-10:`createJsonlSink` 每个事件输出一行合法 JSON;`write` 抛异常不得中断运行
- AC-11:`run_finished` 中的 `usage`、`toolCalls`、`durationMs` 与实际一致
- AC-12:`AgentEvent` 以 zod discriminated union 导出,可被前端复用校验

## 5. 测试要求

- 使用 `FakeModelClient`、假工具、假定时器与注入的 `clock`,禁止真实等待与真实网络
- 每条 AC 至少一个测试,测试名以 `AC-n:` 开头
- AC-5、AC-6 必须证明「进行中的调用确实收到了 abort 信号」,而不仅是循环退出
- AC-7 对全部 `RunStatus` 分别验证事件序列完整

## 6. 涉及文件与依赖

允许新增或修改:

- `src/agent/limits.ts`、`src/agent/events.ts`、`src/agent/sinks.ts`、`src/agent/loop.ts`、`src/agent/index.ts`
- `tests/agent/limits.test.ts`、`tests/agent/events.test.ts`、`tests/agent/loop-budget.test.ts`

允许新增的依赖:无(`zod` 已在 M0-01 引入)。

明确禁止修改:`.github/`、`.cursor/`、`AGENTS.md`、`src/governance/`、`src/tools/`。对 M0-01 已有测试的修改,仅限适配 `runAgent` 新增的参数,不得削弱断言。

## 7. 非功能要求

- 事件发送为同步、轻量,不得因事件处理拖慢循环
- 不使用全局可变状态;时间与随机 ID 均可注入

## 8. 发布、回滚与线上核实判据

不适用(无线上环境)。

## 9. 完成定义

- 所有 AC 有对应测试且通过;`pnpm lint`、`pnpm typecheck`、`pnpm test` 通过
- 变更约 400 行以内;若超出,拆为 M0-03a(预算与终止:AC-1 到 AC-6)与 M0-03b(事件流:AC-7 到 AC-12)两个 PR
- 回执完整

## 10. 开放问题

- 默认限制值(轮数、工具调用数、token、时长)的具体数字:先取保守默认值并写在常量里,M0-05 接入真实模型后用实际 trace 调整
