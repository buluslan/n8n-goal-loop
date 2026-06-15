# n8n Goal Command Playbook

## What This Skill Produces

A nine-element `/goal` command for an n8n workflow. The command prefix stays `/goal` (not `/目标`). For Chinese users the body can be fully Chinese.

A normal prompt tells the agent what to do now. A goal defines a durable operating contract: the node pipeline and deliverable, how data flows, how failures are handled, what must not change, where work may happen, how to iterate, when to stop, and when to pause.

## The Nine-Element Template

```text
/goal [Outcome: node pipeline 输入→处理→输出 + deliverable].
数据流：[node-to-node field mapping + key input/output fields + data format].
错误处理：[failure strategy for fragile nodes: continueOnFail / fallback / degrade].
验证：[static check → deploy → script test → end-to-end test].
约束：[n8n red lines].
边界：[only the target workflow].
迭代策略：[small sample first; refresh cache after PUT; max 3 rounds].
完成条件：[end-to-end test passes + real business values].
暂停条件：[real credentials / publish / money / instance anomaly].
```

Field-by-field (see `n8n-9-elements.md` for full detail):

| Element | Answers | Good content |
|---|---|---|
| Outcome | What workflow, what deliverable? | Node pipeline (input→process→output) + final artifact |
| Data Flow | How do fields move? | Node-to-node mapping, key fields, format rules |
| Error Handling | What if a node fails? | continueOnFail, fallback store, degrade on low data |
| Verification | How to prove it? | Static check + script test + end-to-end test (real data) |
| Constraints | What must not change? | DB integrity, no hardcoded secrets, Code node no HTTP, UTF-8 jsonBody |
| Boundaries | Where may it write? | Only the target workflow JSON |
| Iteration | How to recover? | Small sample first, refresh cache, fix against pitfalls, max 3 rounds |
| Completion | When done? | End-to-end passes + business fields have real values |
| Pause | When to stop and ask? | Real credentials, publish, money, instance anomaly |

## Strong Example — Build a classification workflow

```text
推荐执行版（中文，可直接复制）

/goal 搭建 n8n 客户反馈自动分类工作流：读取反馈来源 → AI 按好评/差评/咨询分类 → 结果写回表格，差评触发通知。
数据流：反馈原文+来源+ID → 分类节点输出 category(好评/差评/咨询)+置信度 → 写回表格字段(原文/分类/时间) → 差评触发通知，传反馈ID+原文摘要。关键：分类节点输出字段必须覆盖下游写入需求；外部数据先清洗双引号/换行。
错误处理：AI 分类节点失败 → continueOnFail + 标记 unclassified，不阻断整批；写回失败 → 记录待补传，不拖垮主流程；样本不足 → 降级为描述性说明，不编造分类。
验证：校验脚本通过 → 部署可见 nodes 数正确 → 脚本测试(API 读 workflow 200 + Webhook 返回 started + execution 推进到业务节点) → 端测(任务 pending→completed + 表格写入 + 通知送达，execution success 不算)。
约束：不破坏数据库/不碰无关工作流/不硬编码密钥(用 credentials)/Code 节点不发 HTTP/jsonBody 用 UTF-8/AI 须基于真实数据不编造。
边界：只新建该工作流，不改其他。
迭代策略：先小批量跑通再扩量；PUT 后 deactivate+activate 刷新缓存；失败对照踩坑库修复，最多 3 轮。
完成条件：端测全过(分类正确+写回+通知)。
暂停条件：需真实凭证/发布激活/涉及资金/实例异常时暂停。

默认选择理由：先小样本验证分类准确性，再扩量。
```

## Strong Example — Debug a failing node

```text
/goal 定位分类节点执行失败的根因：区分是结构问题、业务数据问题、还是实例问题，给出修复方案并重测。
验证：复现失败 → 读节点输入输出与日志 → 区分问题层级 → 修复后端测通过该节点。
约束：不破坏数据库；如出现 SQLITE_CORRUPT/卡在 trigger，先修实例再谈工作流。
迭代策略：先脚本测试定位层级，再针对性修复；同一错误连续 2 次后换证据来源。
完成条件：该节点在端测中输出正确字段。
暂停条件：需真实凭证/涉及资金/实例需重建时暂停。
```

## Anti-Patterns — reject or revise

Weak:
```text
/goal 做个 n8n 工作流处理评论。
```

Why weak: no node pipeline, no deliverable, no data flow, no test layering.

Better: state the pipeline (input→process→output), the deliverable, the data-flow contract, and layered verification (see the strong example above).

Avoid:
- `execution success` as the only completion proof.
- `keep trying` or `until it looks good` as iteration.
- Missing data-flow contract or error handling.
- Hardcoding a specific vendor/brand instead of a generic category.
- Missing pause conditions for credentials, publish, money, or instance anomalies.
- **Square brackets `[...]` to label nodes or stages** — lint treats `[...]` as an unresolved placeholder and rejects the goal. Describe nodes in plain text (e.g. `via the Google Sheets read node`), never `[Google Sheets read]`. The brackets in the template above are placeholders to fill and remove, not labels to keep.
