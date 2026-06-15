# n8n 开发最佳实践参考

> 开发 n8n 节点、表达式、Code 节点、工作流时的通用最佳实践。摘自开源项目 **n8n-skills**（7 个 Claude skill 精华），覆盖表达式语法、Code gotcha、节点配置、工作流模式、验证规则。
>
> 致谢来源：n8n-skills by czlonkowski — https://github.com/czlonkowski/n8n-skills （MIT License）

---

## 1. 表达式语法

表达式是节点间传数据的方式，写错是最常见的错误来源。

**核心规则**
1. 动态内容必须用双花括号：`✅ {{$json.email}}`　`❌ $json.email`（当字面文本）　`❌ {$json.email}`
2. 不嵌套花括号：`❌ {{{$json.field}}}`
3. 带空格/特殊字符的字段名用方括号+引号：`✅ {{$json['field name']}}`
4. 节点引用必须加引号、大小写敏感：`✅ {{$node["HTTP Request"].json.data}}`　`❌ {{$node.HTTP Request}}`　`❌ {{$node["http request"]}}`

**核心变量**

| 变量 | 用途 |
|------|------|
| `$json` | 当前节点输出数据 |
| `$node["名称"]` | 引用任意上游节点输出 |
| `$now` / `$today` | 当前时间/今天零点（Luxon） |
| `$env` | 环境变量（受 `N8N_BLOCK_ENV_ACCESS_IN_NODE` 限制；被禁则改用 credentials） |
| `$workflow` / `$execution` | 工作流/执行上下文 |

**关键 Gotcha：Webhook 数据在 `$json.body`**（最高频错误）
```
Webhook 输出: { headers, params, query, body: { name, email, message } }
❌ {{$json.name}}      ✅ {{$json.body.name}}
```

**何时不用表达式**

| 场景 | 改用 |
|------|------|
| Code 节点内 | 直接 JS：`$json.email`，**不写 `{{}}`** |
| Webhook 路径 | 静态字符串 |
| Credential 字段 | n8n credential 系统 |
| 对象属性值 | 前缀 `=`：`"name": "={{$json.body.name}}"` |

---

## 2. Code 节点（JavaScript）

**数据访问（按使用率）**

| 模式 | 用途 |
|------|------|
| `$input.all()` | 全部 item，批处理/聚合（26%） |
| `$input.first()` | 首个 item，单条/API 响应（25%） |
| `$input.item` | 当前 item，仅 Each Item 模式（19%） |

78% 成功工作流用默认 "Run Once for All Items"；仅逐条独立操作时用 "Each Item"。避免裸用 `$json`，优先 `$input.first().json`。

**返回格式**（决定 39% 的失败）：必须返回「对象数组，每个含 `json` 键」
```javascript
✅ return [{ json: { field1: value1 } }];                 // 单条（39%成功节点用此式）
✅ return $input.all().map(i => ({ json: i.json }));      // 多条
✅ return [];                                              // 无数据
✅ return $input.all();                                    // 透传
❌ return "processed";       // 裸字符串
❌ return { json: {...} };   // 缺数组
❌ return [{ data: v }];     // 应为 json
```

**内置函数**（无需 import）：`$helpers.httpRequest(opts)`、`$now`(Luxon: `.toISO()`/`.plus({days:7})`)、`$jmespath(data, query)`、`$getWorkflowStaticData('global')`（跨执行累积）；标准对象 `Math/Date/JSON/Buffer/console/Object/Array`。

**Top 5 错误（占验证失败 62%+）**

| # | 错误 | 占比 | 修复 |
|---|------|------|------|
| 1 | 空代码 | 23% | 必须有实现，否则换 Set |
| 2 | 缺 return | 15% | 所有分支都 return，空也 `return []` |
| 3 | 误用 `{{}}` | 8% | Code 内用模板字符串 `` `${v}` `` |
| 4 | 括号不匹配 | 6% | JSONB 多行串正确转义引号 |
| 5 | 返回包装错 | 5% | 数组套 `{json:{}}` |

**语言选择**：95% 用 JavaScript（快、稳定、全 helper）。Python 仅需标准库特性（re/hashlib/statistics）时用，**不能装外部库**（无 requests/pandas/numpy），HTTP 用 `urllib`。

---

## 3. 节点配置规则

**属性依赖（Progressive Disclosure）**：字段可见性由其他字段值决定。HTTP Request 典型级联：
```
method=POST → 显现 sendBody → contentType="json" → specifyBody → jsonBody
```
策略：从最小配置开始，每次 `validate_node` 后按报错补字段。

**Operation 决定必填字段**：Slack message 的 `post` 要 channel+text，`update` 要 messageId+text（channel 非必填）。先定 resource+operation，再查必填项。

**get_node 档位**
- `detail:"standard"`（默认，~1-2K token，覆盖 95%）——先用
- `mode:"search_properties", propertyQuery:"auth"`——找特定字段
- `detail:"full"`（~3-8K token）——标准档不足时才用

**AI 连接类型**（AI Agent 用 `sourceOutput` 指定）：`ai_languageModel`、`ai_tool`、`ai_memory`、`ai_outputParser`、`ai_embedding`、`ai_vectorStore`、retriever、文档加载器。拓扑：`Trigger → AI Agent（接 languageModel+tool+memory） → Output`。

---

## 4. 工作流模式（6 种覆盖 90%+）

| 模式 | 链路 |
|------|------|
| **Webhook 处理**（最常见） | Webhook → 校验 → 转换 → 响应/通知 |
| **HTTP API 集成** | Trigger → HTTP Request → 转换 → 写入 → 错误处理 |
| **数据库操作** | Schedule → Query → 转换 → Write → Verify |
| **AI Agent** | Trigger → AI Agent（Model+Tools+Memory） → Output |
| **定时任务** | Schedule → Fetch → 处理 → 投递 → 日志 |
| **批处理** | Prepare → SplitInBatches → 逐批 → 累积 → 聚合 |

**SplitInBatches 约定**：`main[0]`=done（全完成后触发一次，**接聚合，加 Limit 1**）；`main[1]`=每批循环体。跨迭代累积：循环内 `$('Node').all()` 只返回最后一批，用 `$getWorkflowStaticData('global')` 在 Code 里累积。

**构建清单**：规划（选模式→列节点→理数据流→定错误策略）→ 实现（触发器→数据源→凭证→转换→输出→错误处理）→ 验证（逐节点→整体→测试→边界）→ 部署（检查执行顺序/超时→激活→监控）。

---

## 5. 验证规则

**Profile 选择**

| Profile | 何时用 | 特点 |
|---------|--------|------|
| `minimal` | 编辑中快查 | 只验必填，可能漏 |
| **`runtime`（推荐）** | 部署前 | 验必填+类型+枚举+依赖，平衡 |
| `ai-friendly` | AI 生成配置 | 同 runtime 但降噪 |
| `strict` | 生产关键 | 全验+最佳实践+安全，误报多 |

**验证是迭代过程**：预期 2-3 轮 `validate → 修 → validate`，正常现象。流程：配置 → `validate_node({profile:"runtime"})` → 读报错 → 补字段 → 再验。

**Auto-sanitization（保存时自动修，无需手动）**
- 二元操作符（equals/contains/greaterThan/startsWith 等）：自动移除错误的 `singleValue`
- 一元操作符（isEmpty/isNotEmpty/true/false）：自动补 `singleValue: true`
- IF v2.2+/Switch v3.2+：自动补 `conditions.options` 元数据
- **不能自动修**：断开连接（用 `cleanStaleConnections`）、分支数不匹配

**False Positive（误报）**：warning 技术上"不对"但场景可接受——
- Missing error handling：简单/测试/非关键可忽略，生产重要数据该修
- No retry logic：幂等操作/API 自带重试可忽略
- Missing rate limiting：内部 API/低频可忽略
- Unbounded query：小表/聚合/测试可忽略
- 降噪：AI 生成配置用 `profile:"ai-friendly"`

---

## 6. Code 节点上线前速查

- [ ] 代码非空有实现
- [ ] 所有分支都有 return
- [ ] 返回 `[{json:{...}}]`
- [ ] 用 `$input.all/first/item`，不裸用 `$json`
- [ ] Code 内无 `{{}}`（用模板字符串）
- [ ] guard clause 处理 null/undefined
- [ ] JSONB 引号正确转义
- [ ] 无死循环
- [ ] 优先 map/filter/reduce
- [ ] 各分支返回结构一致

---

## 来源与许可

提炼自 **n8n-skills**（MIT License）：https://github.com/czlonkowski/n8n-skills · 作者 czlonkowski。
数据基础：38,094 个 Code 节点实例、2,653+ 工作流模板。原始文档：`CODE_NODE_BEST_PRACTICES.md`、`USAGE.md`、7 份 `SKILL.md`。遵守 MIT：可自由复制/修改/分发，需保留版权与许可声明。
