---
name: n8n-goal-loop
description: Generate copy-ready /goal commands that drive an agent through the full build→validate→deploy→layered-test→iterate loop for n8n workflows. Use when the user wants to build, modify, debug, or test an n8n workflow; needs a goal instruction or spec for n8n development; designs n8n node pipelines (trigger→process→output); wants an automated dev-test closed loop; or asks for n8n workflow requirements, milestones, or acceptance criteria. Outputs a nine-element goal (outcome with node pipeline, data-flow contract, error handling, verification, constraints, boundaries, iteration, completion, pause) via a short alignment interview first. Chinese triggers: 搭建、修改、调试、测试 n8n 工作流；生成 n8n goal 指令、目标指令；设计 n8n 节点链路与数据流；n8n 开发与测试闭环；n8n 工作流需求、里程碑、验收标准。
license: MIT
---

# n8n Goal Loop

Turn an n8n workflow need into a copy-ready `/goal` that drives an agent through a full closed loop: build → validate → deploy → layered test → iterate.

This skill only generates the goal. It does not build, deploy, or run the workflow itself.

## Operating Mode

n8n workflows are complex. **Align with the user first via a short interview, then generate** — do not emit a goal from a single vague line (unless the user already states goal, node pipeline, and deliverable clearly).

Defaults:
- The user wants a paste-ready `/goal`, not a generic prompt.
- Keep the command prefix as `/goal` (not `/目标`).
- The first goal block must be the best recommended executable version, never a half-filled template — users copy the first block.
- For Chinese users: output the Chinese recommended version first, then an English-compatible mirror.
- Interview with numbered choices plus open questions. Never make the user fill out a long form.
- Ask only when the answer changes cost, risk, or direction. Auto-fill everything that has a standard n8n best practice (data-flow format, error handling, n8n red-lines) — do not ask about those.
- Every n8n goal must separate script-level testing from end-to-end testing. `execution success` is not proof the workflow works for the business.
- Generic categories only — never hardcode a specific vendor, brand, or the user's private project details.

## Workflow

1. Classify the task: build new workflow / modify existing / debug a failure / test & verify.
2. Run the alignment interview (see `references/interview-checklist.md`). Four layers:
   - Goal & deliverable — what problem, what final artifact.
   - Node pipeline — data source → processing → output sink.
   - Selection (generic categories) — trigger / deployment / data source / output channel.
   - Red lines — what must not be touched.
3. Generate the nine-element goal (see `references/n8n-9-elements.md` and `references/goal-command-playbook.md`).
4. Let the user confirm or adjust by replying with numbered choices (e.g. `按默认` or `1A 2C`).
5. Check the goal against `references/n8n-pitfalls.md` so known traps are pre-empted.
6. For file deliverables, run `python3 scripts/lint_goal_command.py <file>` before calling the goal done.

## Output Contract

Nine elements (qiaomu seven + data-flow + error handling). See `references/n8n-9-elements.md` for full detail.

```text
推荐执行版（中文，可直接复制）

/goal [Outcome: node pipeline 输入→处理→输出 + deliverable].
数据流：[node-to-node field mapping + key input/output fields + data format].
错误处理：[failure strategy for key nodes: continueOnFail / fallback store / degrade].
验证：[static check → deploy visible → script test (structure/start/execution progress) → end-to-end test (real data, execution success does not count)].
约束：[n8n red lines: no DB damage / no hardcoded secrets / Code node no HTTP / jsonBody UTF-8 / AI must not fabricate data].
边界：[only the target workflow, no others].
迭代策略：[small sample first; refresh cache after PUT; fix against pitfall library; max 3 rounds].
完成条件：[end-to-end test passes + business fields have real values].
暂停条件：[real credentials / publish-activate / real money / instance anomaly (e.g. SQLITE_CORRUPT)].

默认选择理由：[one sentence].

可选调整
1. [dimension]: A (default) / B / C

你可以直接回复：按默认，或类似 1A 2C.

Goal Draft (English-compatible)
[faithful English mirror with the same nine elements]
```

## Quality Bar

A strong n8n goal:
- Outcome states the node pipeline (input → process → output) and the deliverable.
- Data-flow contract covers node-to-node field mapping and key fields.
- Error handling covers failure fallback for fragile nodes.
- Verification separates script test from end-to-end test, with real business evidence.
- Constraints include n8n red lines (no DB damage, no hardcoded secrets, Code node does not issue HTTP, jsonBody uses UTF-8).
- Pause conditions cover real credentials, publish/activate, real money, and instance anomalies.

Reject or revise a goal that:
- Outcome says only "build a workflow" with no pipeline.
- Verification says only "test passes" with no layering.
- Missing data-flow contract or error handling.
- Uses `execution success` as completion proof.
- Hardcodes a specific vendor or brand instead of a generic category.

## Reference Files

- `references/goal-command-playbook.md` — nine-element template, n8n examples, anti-patterns.
- `references/interview-checklist.md` — the four-layer n8n alignment interview.
- `references/n8n-9-elements.md` — full detail on each of the nine elements (incl. data-flow and error handling).
- `references/default-goal-strategy.md` — n8n defaults, risk classification, vague-word translation.
- `references/n8n-pitfalls.md` — high-frequency n8n traps to pre-empt (load when building/debugging).
- `references/n8n-design-patterns.md` — node naming, data-flow patterns, accumulation, AI data passing.
- `references/test-strategy.md` — layered testing methodology (script vs end-to-end).
- `references/env-context.md` — local vs cloud deployment context.

## Scripts

- `scripts/lint_goal_command.py` — checks the `/goal` has all nine elements, no placeholders, and flags n8n-dangerous phrasing (e.g. "直接改线上", "自动下单", "硬编码密钥").

---

Forked from qiaomu-goal-meta-skill by 向阳乔木 (https://github.com/joeseesun). Adapted for n8n workflow development with an alignment-first flow and nine-element structure. MIT license.
