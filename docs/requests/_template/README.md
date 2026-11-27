# REQ-xxx 需求目录模板

复制整个 `_template` 目录为 `docs/requests/REQ-001`(编号递增),按阶段逐个填写:

| 阶段 | 文件 | 谁填 | 人工门 |
|---|---|---|---|
| ① 需求 | request.md | Claude 整理,你确认 | 小需求可跳过后续设计 |
| ② 调研 | findings.md | Claude(只读) | |
| ③ 方案 | proposal.md → decision.md | Claude 出方案,你拍板 | G1 |
| ④ 设计 | design/、copy.json(有前端时) | Claude 出,你审 | G2 |
| ⑤ 契约 | spec.md(冻结后不得随意改) | Claude 写,你冻结 | G3 |
| ⑥ 派活 | tasks.md | Claude | |
| ⑦ 实现 | receipt.md(同时填在 PR 描述里) | Cursor | |
| ⑧ 验收 | acceptance.md | Claude 与 Kimi | G4(仅 R2) |
| ⑨ 核实 | verification.md | 自动 + Claude 写结论 | |

spec 的模板在 `specs/_template.md`;REQ 目录里的 spec.md 是它的一份拷贝。
