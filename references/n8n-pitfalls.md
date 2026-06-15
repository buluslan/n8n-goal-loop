# n8n 工作流开发避坑清单

> 生成 n8n goal 时对照避坑，把高频坑预置进 constraints/iteration。每张卡片按「症状 → 检查项 → 标准做法」组织。

---

## 类别 1：节点参数版本不匹配

### 1.1 Switch v3 参数格式用了 V1/V2 旧格式

- **症状**：通过 API 导入的工作流，API 能读出节点，但 Web UI 画布完全空白；容器日志报 `Could not find property option`。
- **检查项**：Switch 节点的 `typeVersion` 是否为 `3`；参数是否用了 V1/V2 的 `dataType`/`value1`/`rules.rules[]`。
- **标准做法**：V3 用 `mode: "rules"` + `rules.values[]`，每条规则是一个 `conditions` 对象（含 `leftValue`/`rightValue`/`operator`/`combinator`），回退输出放 `options.fallbackOutput`。API 导入后立即看容器日志确认无 `Could not find property`。

### 1.2 Set 节点 manual 模式默认丢弃其他字段

- **症状**：Set 节点后下游字段（如 `run_mode`、`product_name`）全部丢失，Switch 无法路由，流程中断。
- **检查项**：Set 节点 `mode: "manual"` 时 `includeOtherFields` 是否为 `false`（默认）。
- **标准做法**：「追加字段」语义（如加 `status`）必须设 `includeOtherFields: true`；只有「替换数据」语义才保持默认 `false`。

### 1.3 Split In Batches v3 双输出行为与 v2 不同

- **症状**：期望依次处理 N 条，第一轮就结束，后续全部丢失；output 0 和 output 1 同时触发后立即终止。
- **检查项**：Split In Batches 的 `typeVersion` 是否为 `3`；是否误用了 v2 的串行假设。
- **标准做法**：简单场景优先用直接连接而非 Split In Batches；连接顺序口诀「0 号输出接聚合（done），1 号输出接处理（loop）」。若用 v3，明确其并行 output 行为。

### 1.4 Code 节点返回数组后又加 Split Out

- **症状**：Code 节点已返回 7 个 item，Split Out 后输出 0 项，后续节点全停。
- **检查项**：Code 节点是否用 `return array` 返回了多个 item；是否又接了 Split Out。
- **标准做法**：Code 节点返回数组时 n8n 自动拆为独立 item，不要再 Split Out。只有「单个 item 中嵌套数组字段」需要展开时才用 Split Out。

---

## 类别 2：数据流与引用

### 2.1 AI / LangChain 节点丢原始数据

- **症状**：AI 节点只返回输出，后续节点拿不到原始输入字段；`pairedItem` 元数据为 undefined。
- **检查项**：AI 输入节点是否保留了 `_original`。
- **标准做法**：准备 AI 输入时 `json: { prompt: ..., _original: item.json }`；解析 AI 结果时 `const original = $input.first().json._original; return [{ json: { ...original, ...aiResult } }]`。不要依赖 `pairedItem` 或从上游节点反查。

### 2.2 `$('Node').item` 多 item 场景只取第一条

- **症状**：4 条数据全部写入相同/错误值，或字段显示「未知」。
- **检查项**：Set/Code 节点是否用 `$('Node').item` 引用多 item 节点。
- **标准做法**：引用直接上游用 `$json`；批量处理时用 `$input.all()` + `map`。`$('Node').item` 只适用于「取任意一条」的场景。

### 2.3 HTTP Request 返回数组导致 item 膨胀

- **症状**：4 个请求 × 20 条评论 = 80 个 item，破坏管道 1:1 关系；无评论的请求丢失。
- **检查项**：HTTP Request 节点是否返回顶层 JSON 数组（自动拆分为多 item）。
- **标准做法**：HTTP 后加 Code 节点，按主键分组恢复 1:1；引用更上游节点获取原始 item 列表，确保空结果的项也输出空数组。

### 2.4 列表 API 响应未拆分（嵌套数组不自动拆）

- **症状**：API 返回 `{code, data: {items: [...]}}`，HTTP Request 输出 1 个 item，下游 Split In Batches 收到 1 个含嵌套数组的对象。
- **检查项**：HTTP Request 只自动拆「顶层 JSON 数组」，不拆嵌套数组。
- **标准做法**：HTTP 和下游之间插 Code 节点：`const items = response.data.items || []; return items.map(i => ({json: i}))`。

### 2.5 字段名与目标表不匹配

- **症状**：写入报 `field_name not found`；结果字段全为空。
- **检查项**：工作流字段名（英文 `status`）是否与目标表字段名（中文 `状态`）一致。
- **标准做法**：设计阶段锁定字段名（推荐与目标表一致），建立字段别名兼容层；新增/改名字段后同步解析节点和下游。每次字段变更做最小回归。

### 2.6 删除/重命名节点后表达式引用未同步

- **症状**：运行报 `The node 'X' doesn't exist, but it's used in an expression`。
- **检查项**：API 改节点名后是否全局搜索了所有 `$('X')` 和 `$items('X')` 引用。
- **标准做法**：改名前 `grep` 旧名，检查 Code 节点 `jsCode`、表达式 `={{ }}`、HTTP `jsonBody`、connections 对象，全部替换后再推送。节点命名避免括号/特殊字符。

### 2.7 并行分支存在表达式依赖竞态

- **症状**：节点 B 引用节点 A 输出，但两节点并行执行，B 偶发取不到 A 的值。
- **检查项**：是否存在表达式依赖的节点被连接到同一 output（并行）。
- **标准做法**：B 的表达式引用 A 的输出时，A 必须是 B 的上游（串行连接）。只有无依赖节点才能并行。

---

## 类别 3：Code 节点沙箱限制

### 3.1 沙箱不支持 fetch / this.helpers

- **症状**：Code 节点 `fetch()` 报 `fetch is not defined`；`this.helpers.httpRequest` 报 `this.helpers is not defined`。
- **检查项**：Code 节点是否在发 HTTP 请求（沙箱禁用）。
- **标准做法**：HTTP 调用用 `n8n-nodes-base.httpRequest` 节点，Code 节点只做数据转换/格式化。架构拆分：HTTP Request 节点 → 外部 API，Code 节点 → 纯数据格式化。

### 3.2 沙箱不支持 crypto（HMAC/签名无法算）

- **症状**：`require('crypto')` 报 `Module 'crypto' is disallowed`；`crypto.subtle` 报 `crypto is not defined`。
- **检查项**：Code 节点是否在做密码学操作（HMAC-SHA256 签名等）。
- **标准做法**：签名计算放工作流外部（外部脚本 + 环境变量注入），或关闭服务端签名校验。Webhook HMAC、AWS SigV4 等场景都必须在 Code 节点外处理。

### 3.3 云部署禁用 fs / Python

- **症状**：`Error: Module 'fs' is disallowed`；`Import of standard library module is disallowed`。
- **检查项**：是否在云实例上用了 `require('fs')`、Python 标准库、临时文件写入。
- **标准做法**：累积数据用 `$getWorkflowStaticData('global')` 或 Aggregate 节点；Python 逻辑转 JavaScript；文件输出改为 Code 节点返回 JSON 或写入云存储节点（S3/Sheets）。

---

## 类别 4：jsonBody 与表达式

### 4.1 特殊字符破坏 JSON（双引号/换行符）

- **症状**：HTTP Request jsonBody 报 `The value in the "JSON Body" field is not valid JSON`。
- **检查项**：外部数据（产品名、用户输入）是否未经清洗直接嵌入 jsonBody 模板。
- **标准做法**：上游 Code 节点新增清洗字段 `display_name`：`.replace(/"/g, "'").replace(/\n/g, ' ').trim()`；下游统一引用清洗后的字段。所有文本类 jsonBody 字段用 `JSON.stringify()` 包裹。

### 4.2 ES6 Unicode 转义 `\u{XXXXX}` 非法

- **症状**：jsonBody 报 `Bad Unicode escape in JSON`。
- **检查项**：JSON 字段是否用了 `\u{1F4CB}` 这种 ES6 花括号转义。
- **标准做法**：JSON 字段中 emoji 直接用 UTF-8 原字符；如需 ASCII-safe，用 4 位 `\uXXXX` 代理对。代码生成工作流 JSON 时用 `ensure_ascii=False`。

### 4.3 n8n 表达式不支持对象字面量取值

- **症状**：表达式 `{a:"x",b:"y"}[$json.key]` 报 `invalid syntax`。
- **检查项**：表达式 `{{ }}` 内是否用了对象字面量 + 方括号取值。
- **标准做法**：改用三元运算符链：`($json.key === "a" ? "x" : $json.key === "b" ? "y" : "默认")`。n8n 表达式是受限 JS 子集，对象字面量取值、解构等不支持。

### 4.4 Python f-string 三重花括号转义错误

- **症状**：Python 生成的 jsonBody 中 `{{ expression }}` 变成 `{{{ expression }}}`。
- **检查项**：是否用 Python f-string 生成含 n8n 表达式的 JSON 模板。
- **标准做法**：避免 f-string，改用字符串拼接或 `.format()`。生成 Code 节点源码时写真实换行 `\n`，不要写字面量 `\\n`（会导致 `Invalid or unexpected token`）。

---

## 类别 5：凭证与 API

### 5.1 API Key vs Cookie 认证

- **症状**：Cookie 认证一段时间后失效（JWT 约 7 天过期）；DB 重置后旧 API Key 失效返回 `unauthorized`。
- **检查项**：自动化脚本是否用 Cookie；DB 重置/迁移后是否重新生成了 API Key。
- **标准做法**：自动化脚本统一用 API Key（`X-N8N-API-KEY`），Cookie 仅临时调试。注意 `/rest/` 端点用 Cookie，`/api/v1/` 端点用 API Key。DB 重置后 API Key 的 JWT 签名密钥会变，必须重新生成。

### 5.2 OAuth2 redirect_uri 不匹配

- **症状**：`redirect_uri_mismatch` 或 `access_denied (403)`；点 Sign in 后立即报错。
- **检查项**：OAuth 回调 URI 是否完全匹配（协议/域名/路径/末尾斜杠）；客户端类型是否为 Web 应用；测试应用是否添加了测试用户。
- **标准做法**：回调 URI 用 `${N8N_BASE_URL}/rest/oauth2-credential/callback`（无末尾斜杠、必须 https 生产/http 本地）；客户端选「Web 应用」；测试模式加测试用户。配置后等 1-2 小时生效。

### 5.3 PUT settings 额外属性返回 400

- **症状**：`PUT /api/v1/workflows/{id}` 返回 400 `settings must NOT have additional properties`。
- **检查项**：PUT body 是否原样传回了 GET 返回的 settings（含 `executionOrder`/`binaryMode` 等运行时属性）。
- **标准做法**：PUT 只发送 `name`、`nodes`、`connections`、`settings`，且 settings 传空对象 `{}`。不要 `json.dumps(wf)` 发送完整对象。

### 5.4 PUT 后缓存问题需 deactivate + activate

- **症状**：API PUT 推送成功，但执行仍用旧代码，调试循环无效。
- **检查项**：执行引擎是否缓存了旧版本。
- **标准做法**：开发阶段频繁修改时直接 DELETE + POST 重建工作流；或每次 PUT 后 deactivate → activate 强制刷新。REST API（Cookie）不支持 PUT 更新，只有 Public API（API Key）支持。

### 5.5 activate 需要 versionId

- **症状**：`POST /api/v1/workflows/{id}/activate` 返回 400 `versionId required`；Webhook 触发报 `Workflow could not be started!`。
- **检查项**：activate 请求 body 是否传了 `versionId`；`activeVersionId` 是否已同步。
- **标准做法**：先 GET 获取 `versionId`，activate body 传 `{"versionId": "..."}`。导入工作流后需设 `activeVersionId` 再 activate。

### 5.6 HTTP Request + OAuth2 凭证不兼容

- **症状**：HTTP Request 节点用 `googleSheetsOAuth2Api` 凭证调外部 API 报 `Authorization failed`。
- **检查项**：是否用 HTTP Request 节点 + 内置 OAuth2 凭证调用第三方 API。
- **标准做法**：用 n8n 对应的原生节点（如 `googleSheets` 节点）操作该服务，原生节点内部处理 token 获取和刷新。

---

## 类别 6：实例与部署

### 6.1 SQLite WAL 模式并发损坏

- **症状**：`SQLITE_NOTADB: file is not a database`；`SQLITE_CORRUPT`；用户账户丢失无法登录。
- **检查项**：是否用默认 SQLite 且存在并发执行 / 快速连续触发。
- **标准做法**：生产用 PostgreSQL（n8n 推荐方案）；SQLite 场景减少并发（分批 4+3）。账户丢失时直接操作 DB 重置 bcrypt 密码。

### 6.2 卡在 Webhook Trigger / 无法 API 触发

- **症状**：`POST /rest/executions` 404；`/run` 端点报 `Cannot read properties of undefined`；Playwright 找不到 Execute Workflow 按钮。
- **检查项**：是否试图用 REST API 触发 Manual Trigger 工作流。
- **标准做法**：Manual Trigger 走 WebSocket，无法脚本化。调试期加 Webhook Trigger 节点，用 curl 触发 `POST ${N8N_BASE_URL}/webhook/{path}`；或用 API Key `POST /api/v1/executions`（需工作流已激活）。

### 6.3 Docker DNS 竞态 / 故障

- **症状**：并行分支偶发 `getaddrinfo EAI_AGAIN`；部分 HTTP 请求 DNS 超时。
- **检查项**：是否多分支同时发 HTTP 请求（Docker 内置 DNS 127.0.0.11 并发能力有限）。
- **标准做法**：所有外部 HTTP 节点配重试 `retryOnFail: true, maxTries: 5, waitBetweenTries: 10000`；docker-compose 显式配 `dns: [1.1.1.1, 8.8.8.8]` + `dns_opt: [timeout:5, attempts:3]`。DNS 抖动靠重试自动恢复，分批执行是兜底方案。

### 6.4 云部署环境限制（fs/Python/临时文件）

- **症状**：云实例 `Module 'fs' is disallowed`；Python 标准库禁用；临时文件不可写。
- **检查项**：是否在云环境用了本地文件系统或 Python。
- **标准做法**：数据累积用静态数据或 Aggregate 节点；Python 逻辑转 JavaScript（`json.loads→JSON.parse`、`datetime.now()→new Date()` 等）；文件输出改 Code 节点返回 JSON 或云存储节点。

### 6.5 运行态观测：executions API 非真值

- **症状**：webhook 运行中的执行不及时出现在 `GET /api/v1/executions`；`includeData=true` 执行结束后数据仍为空。
- **检查项**：是否依赖 API 获取运行态和历史执行输出。
- **标准做法**：运行态以实例事件日志为主观测源，API 作补充查询；调试通过目标表 API 交叉验证或 Web UI 实时监控。关键节点增加可读状态回写字段形成外部可见真值。
