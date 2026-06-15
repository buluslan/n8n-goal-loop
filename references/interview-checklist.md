# n8n Alignment Interview

n8n workflows are complex, so align with the user before generating. Run a short, four-layer interview. Use numbered choices with defaults; the user should be able to reply `按默认` or `1B 2C`.

Skip a layer only if the user already stated it clearly. Never make the user fill out a long form.

## Layer 1 — Goal & Deliverable

- What problem does this workflow solve?
- What is the final artifact the user receives? (a report / a batch of content / a notification / data written to a table / a file)
- Is a minimal first version acceptable, or must it be production-complete?

If vague, offer:
1. 交付物：A 数据写回表格（默认） / B 生成文件 / C 发通知 / D 多个

## Layer 2 — Node Pipeline (the core)

- **Input**: where does data come from? (user-provided file / third-party API / web scrape / database / table trigger)
- **Processing**: what happens in the middle? (AI classify / AI generate / data compute / conditional branch / format convert)
- **Output**: where does the result go? (write back to table / IM notification / generate file / push to another system)

Ask these three explicitly — the node pipeline is the soul of the goal. If vague, offer generic-category choices:
1. 数据来源：A 用户提供 / B 第三方 API / C 网页抓取 / D 数据库
2. 处理方式：A AI 分类 / B AI 生成 / C 数据计算 / D 条件判断
3. 输出去向：A 写回表格 / B IM 通知 / C 生成文件 / D 推送系统

## Layer 2b — 具体服务确认（通用类别确定后必问）

通用类别只是方向；**具体服务影响节点配置**（用哪个 n8n 节点、怎么接凭证），必须确认，否则工作流搭不出来。

- 数据源/输出是「表格」→ 哪个服务？Google Sheets / 飞书多维表格 / 腾讯文档 / Notion / 本地 Excel·CSV
- 处理是「AI」→ 三个子项都要问：
  - 模型服务：OpenAI (GPT) / Anthropic (Claude) / Google (Gemini) / 国产（通义·文心·智谱 GLM）
  - 接入方式：n8n 内置 LangChain AI 节点 / HTTP Request 调官方 API / 第三方中转（OpenRouter 等）
  - 具体模型名：哪个型号
- 数据源是「第三方 API」→ 哪个 API（用户已有 key 的）
- 数据源是「网页抓取」→ 哪个抓取服务 / 自建
- 输出是「IM 通知」→ 飞书 / 钉钉 / 企业微信 / Slack / Telegram

注意：这一层问的是「用户实际要用什么具体服务」，**不是 skill 预设品牌**。skill 内容（references/示例）保持通用，但访谈必须确认用户的具体选择——否则节点和凭证配不出来。

## Layer 3 — Selection (generic categories, never a brand)

- **Trigger**: scheduled / webhook / manual / event-driven
- **Deployment**: local (Docker or direct) / cloud
- **Scale hint**: small batch first, or large volume?

If deployment is cloud, note that file-system operations and some libraries may be restricted — the goal should prefer static data and HTTP Request nodes.

## Layer 4 — Red Lines

- What must not be touched? (production data / live store settings / real money / compliance boundaries)
- Are real credentials required, or can the first version use sample/mock data?

This layer decides the pause conditions.

## Interview Output Shape

After the interview, output the nine-element goal (see `goal-command-playbook.md`), then:

```text
默认选择理由：[one sentence on why these defaults are lowest-cost / safest / best-validate-core].

可选调整
1. [dimension]: A (default) / B / C
2. [dimension]: A (default) / B / C

你可以直接回复：按默认，或类似 1A 2C。
```

## Rules

- Keep the interview to 2-3 rounds. The goal is to reduce ambiguity, not exhaust the user.
- Auto-fill data-flow, error handling, constraints, verification, iteration, completion, pause from standard n8n best practice — do not ask about those.
- Ask only when the answer changes cost, risk, or direction.
- If the domain (e.g. a specific platform's API) is unfamiliar, generate a discovery-first outcome: "first read the platform's docs/sample data, then implement."
