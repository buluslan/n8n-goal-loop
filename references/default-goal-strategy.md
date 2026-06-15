# n8n Default Goal Strategy

## Interview-First (n8n adaptation)

Unlike a generic coding task, an n8n workflow is complex enough that **the skill aligns with the user first**, then generates. Do not emit a goal from a single vague line unless the user already states goal, node pipeline, and deliverable.

Reason: the node pipeline and deliverable are unique to the user and cannot be safely assumed. Getting them wrong wastes a full build-test cycle.

## Output Priority

For Chinese users, output in this order:

1. `推荐执行版（中文，可直接复制）` — the best nine-element `/goal`
2. `默认选择理由` — one concise sentence
3. `可选调整` — numbered choices with defaults
4. `你可以直接回复`
5. `Goal Draft (English-compatible)` — faithful mirror

Do not put a half-filled template before the recommended executable goal.

## Auto-Fill Rule

Auto-fill every element that has a standard n8n best practice:

- **Data Flow**: derive from the node pipeline; apply standard format rules (clean external data before jsonBody; UTF-8 not `\u{}`; each node's output covers downstream needs).
- **Error Handling**: continueOnFail on fragile nodes + mark failed + fallback store; degrade on low data; max 3 retries.
- **Verification**: layered — static check → deploy → script test → end-to-end test.
- **Constraints**: n8n red lines (no DB damage, no hardcoded secrets, Code node no HTTP, UTF-8 jsonBody, AI grounded in real data).
- **Boundaries / Iteration / Completion / Pause**: standard templates.

Ask the user only about: goal, node pipeline, generic-category selection, and red lines.

## Risk Classification

- **Low risk**: local prototype, sample/mock data, isolated test workflow, non-destructive formatting.
- **Medium risk**: changes to an existing production workflow, public-facing output, shared credentials, external APIs with test keys.
- **High risk**: production data, real payments, real store/vendor accounts, destructive deletion, compliance decisions, copyrighted assets, publish/activate to production.

Behavior:
- Low risk: auto-fill and generate a copy-ready goal.
- Medium risk: auto-fill, add explicit boundaries and pause conditions.
- High risk: ask a numbered decision, or generate a goal that pauses before the risky action (e.g. discovery-only until the user confirms).

## Vague Words

Do not ban vague direction words. Translate them into iteration and verification.

Example:
```text
处理方式：AI 把反馈分类成好评/差评/咨询，输出结构化 category 字段。
验证：用 5 条真实样本跑通，检查 category 字段非空且分类合理；样本不足时降级说明。
迭代策略：基于样本结果调 prompt，最多 3 轮。
```

The vague words guide taste; verification proves whether the result is acceptable.

## Iteration Defaults

Bounded autonomy:
```text
迭代策略：先小样本跑通闭环；PUT 后 deactivate+activate 刷新缓存；失败对照踩坑库修复；同一错误连续 2 次后换证据来源；最多 3 轮。
```

Do not write `keep trying` or `until it works`.

## Completion Rule

End-to-end test pass is the only completion proof. `execution success` alone is not acceptable — a workflow can report success while writing nothing to the result sink.

## Finalization Rule

After the user answers choices, output `最终可复制 /goal` and keep the response mostly to one code block. Do not repeat the full explanation unless asked.
