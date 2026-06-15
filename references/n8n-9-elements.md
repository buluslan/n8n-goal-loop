# The Nine Elements of an n8n Goal

An n8n `/goal` has nine elements. The first seven are inherited from the qiaomu goal contract; **data-flow** and **error-handling** are added because they are where n8n workflows fail most often.

Each element is marked **[ask]** (must be confirmed with the user) or **[auto]** (filled by this skill from standard n8n best practice, the user only adjusts if wrong).

---

## 1. Outcome (目标结果)  — [ask]

What the workflow must become, stated as a result, not an activity. For n8n this **must include the node pipeline and the deliverable**.

Shape: `build a workflow that [input] → [processing] → [output], delivering [artifact]`.

Ask the user:
- What problem does this workflow solve?
- What is the final artifact the user receives? (a report / a batch of content / a notification / data written to a table)

Example:
> 搭建一个客户反馈自动分类工作流：读取反馈来源 → AI 按好评/差评/咨询分类 → 结果写回表格，差评触发通知。

---

## 2. Data Flow (数据流契约)  — [auto]

The contract for how data moves between nodes: field mapping, key input/output fields, data format. This is the blood of an n8n workflow and the single largest source of bugs (lost fields, wrong mapping, silent drops).

Derive it from the node pipeline the user gave. Cover:
- Each node's key input fields and output fields.
- How fields map from one node to the next (which upstream field feeds which downstream field).
- Data format conventions (external data must be cleaned before entering jsonBody: double-quotes → single-quotes, newlines → spaces; jsonBody uses UTF-8 characters, not `\u{}` escapes).
- Critical rule: the output fields of each node must cover every field its downstream nodes need.

Example:
> 数据流：[输入] 反馈原文+来源+ID → [分类节点] 输出 category(好评/差评/咨询)+置信度 → [写回] 表格字段映射(原文/分类/时间) → [通知] 差评触发，传反馈ID+原文摘要。关键：分类节点输出字段必须覆盖下游写入需求；外部数据先清洗双引号/换行。

---

## 3. Error Handling (错误处理)  — [auto]

How fragile nodes behave on failure, so one bad item does not kill the whole batch. n8n production workflows must define this.

Default strategy:
- AI / external-API nodes: `continueOnFail` + mark the item as failed/unclassified, do not block the batch.
- Write-back nodes: on failure, record a `pending` flag for retry, do not drag down the main flow.
- Insufficient data (e.g. too few samples for AI analysis): degrade to a descriptive note, never fabricate.
- Set an upper bound on retries per item (e.g. max 3).

Example:
> 错误处理：AI 分类节点失败 → continueOnFail + 标记 unclassified，不阻断整批；写回失败 → 记录待补传，不拖垮主流程；样本不足 → 降级为描述性说明，不编造分类。

---

## 4. Verification (验证)  — [auto]

Layered. Never accept `execution success` alone as proof.

Two layers (see `test-strategy.md`):
- **Script-level test**: static check (structure/nodes/connections), webhook start returns `Workflow was started`, new execution progresses past the trigger node into business nodes.
- **End-to-end test**: real input → real output. Task status moves `pending → processing → completed`, result sink has new records with real values, notifications actually arrive.

End-to-end pass is the only proof the workflow works for the business.

---

## 5. Constraints (约束)  — [auto]

What must not change, plus n8n red lines:
- Do not damage the n8n database (no direct writes to `database.sqlite` / execution tables).
- Do not touch unrelated workflows.
- Do not hardcode secrets — use n8n credentials.
- Code node must not issue HTTP requests (use the HTTP Request node) — the Task Runner sandbox blocks `fetch`/`crypto`.
- jsonBody uses UTF-8 characters, not `\u{XXXXX}` escapes.
- AI analysis must be grounded in real data; if samples are insufficient, state so explicitly instead of fabricating.
- Web scraping uses public pages only and respects rate limits / platform ToS.

---

## 6. Boundaries (边界)  — [auto]

Where the agent may write. Default: only the target workflow JSON and directly related files. Do not modify other workflows, credentials, or instance settings.

---

## 7. Iteration Policy (迭代策略)  — [auto]

Bounded autonomy:
- Get one minimal path working first (small sample), then scale.
- After a PUT update, `deactivate` + `activate` to refresh the execution cache (a known n8n pitfall); if still stale, `DELETE` + `POST` to rebuild.
- On failure, check the pitfall library before retrying; switch evidence source after 2 consecutive same-error failures.
- Max 3 focused improvement rounds before reporting remaining risk.

Do not write `keep trying` or `until it works`.

---

## 8. Completion (完成条件 / Stop when)  — [auto]

Evidence-based, not feeling-based:
- End-to-end test passes (task status advanced + result sink written + notification delivered).
- Business fields have real values (not empty / not error).
- Static checks pass, or missing config is explicitly reported.

---

## 9. Pause (暂停条件 / Pause if)  — [auto]

Stop and ask the user when any of these is required:
- Real store/vendor account login.
- Activating and publishing to production.
- Real money (ad budget, payment, refund).
- Instance anomaly (`SQLITE_CORRUPT` / `SQLITE_IOERR` / execution stuck at the trigger node) — fix the instance first, then resume workflow conclusions.
- Platform compliance judgment, copyrighted or branded assets, unclear ownership.

---

## Ask vs Auto — summary

| Element | Who | Why |
|---------|-----|-----|
| Outcome | ask | unique to the user |
| Data Flow | auto | derived from the pipeline + standard format rules |
| Error Handling | auto | standard best practice |
| Verification | auto | standard layered methodology |
| Constraints | auto | standard n8n red lines |
| Boundaries | auto | standard |
| Iteration | auto | standard |
| Completion | auto | standard |
| Pause | auto | standard |

The interview stays short because only the Outcome (and the four interview layers that feed it) needs the user. Everything else is auto-filled from n8n best practice; the user adjusts only if a filled value is wrong.
