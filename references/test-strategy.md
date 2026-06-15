# n8n 工作流分层测试方法论

> **用途**: n8n goal「验证」字段必读。强制使用分层测试，禁止用 execution success 作为完成证据。
> **铁律**: API/脚本测试 ≠ 端测，结论不能混用；execution 显示 success 不等于业务可用。

---

## 1. 测试分层铁律

| 测试类型 | 目标 | 是否依赖业务数据 | 能否证明"业务可用" |
|------|------|------|------|
| API/脚本测试 | 验证工作流结构、配置、执行状态、节点报错 | 低，可用最小数据 | 不能证明 |
| 端测 | 验证真实输入到真实输出的完整链路 | 高，必须真实数据 | 可以证明 |

三条原则：

1. API/脚本测试通过，只说明"工作流结构和执行框架基本可用"。
2. 端测通过，才说明"这套工作流对业务真的可用"。
3. 实例异常时（如 SQLITE_CORRUPT），先修实例，再谈工作流结论。

---

## 2. 实例异常优先排查

执行任何测试前，先确认实例健康。出现以下任一日志，**停止业务测试，先修实例**：

| 错误 | 含义 |
|------|------|
| `SQLITE_CORRUPT` / `SQLITE_IOERR` / `SQLITE_NOTADB` | 数据库损坏或 IO 异常 |
| `Last session crashed` | 上一轮运行崩溃 |
| execution 长期卡在 `Webhook Trigger` | 调度或持久化异常，非业务节点问题 |

检查命令：

```bash
docker logs --tail 120 ${CONTAINER_NAME} 2>&1
```

---

## 3. API/脚本测试：5 条路径

### 路径 A：Public API 结构检查

```bash
curl -s -H "X-N8N-API-KEY: ${N8N_API_KEY}" \
  "${N8N_BASE_URL}/api/v1/workflows/${WORKFLOW_ID}" | python3 - <<'PY'
import json, sys
data = json.load(sys.stdin)
print('工作流:', data['name'])
print('active:', data['active'])
print('nodes:', len(data.get('nodes', [])))
print('connections:', len(data.get('connections', {})))
PY
```

校验点：HTTP 200、active 状态、节点数量、关键节点（如 Webhook Trigger）存在。

### 路径 B：JSON 静态校验

```bash
python3 -m json.tool "${WORKFLOW_JSON_PATH}" >/dev/null && echo OK
```

### 路径 C：Webhook 启动测试

```bash
curl -sS -X POST "${N8N_BASE_URL}/webhook/${WEBHOOK_PATH}" \
  -H 'Content-Type: application/json' \
  -d '{}'
```

返回值判定：

| 返回值 | 含义 |
|------|------|
| `{"message":"Workflow was started"}` | 启动成功 |
| `{"code":0,"message":"There was a problem executing the workflow"}` | Webhook 收到请求，但结构或实例有问题 |

### 路径 D：执行记录检查

```bash
sqlite3 "${N8N_DB_PATH}" \
  "select id, workflowId, finished, mode, status, startedAt, stoppedAt from execution_entity order by cast(id as integer) desc limit 10;"
```

重点看：有无新 execution id、状态是否进入有效状态（running/success/error/crashed）、是否长期卡在 running 无进展。

### 路径 E：执行详情检查

```bash
curl -s -H "X-N8N-API-KEY: ${N8N_API_KEY}" \
  "${N8N_BASE_URL}/api/v1/executions/${EXEC_ID}?includeData=true"
```

判定规则：

- `runData` 非空：已进入业务节点执行。
- `runData` 为空且 `nodeExecutionStack` 只剩 `Webhook Trigger`：卡在启动后调度阶段。
- `status=crashed`：优先排查实例。

### API/脚本测试评估标准

**通过**：API 返回 200、JSON 可解析、Webhook 返回 started、产生新 execution、执行推进到至少一个业务节点。

**部分通过**：结构检查通过、工作流可启动，但业务节点失败。说明实例和结构 OK，需查外部 API/字段映射/网络。

**不通过**：API 读不出来、Webhook 报错、无新 execution、execution 长期卡在 Webhook Trigger、出现 SQLITE 错误。结论写："不能继续业务测试，需先修实例。"

---

## 4. 端测：真实输入→真实输出

### 启动路径

1. 在任务表准备一条任务，状态设为 `pending`，确保必填业务字段有值。
2. 触发工作流：

```bash
curl -sS -X POST "${N8N_BASE_URL}/webhook/${WEBHOOK_PATH}" \
  -H 'Content-Type: application/json' \
  -d '{}'
```

### 前置检查

确认任务表记录：状态为 `pending`、业务输入字段（图片/文本/参数）至少有一项有值。

### 观察点 A：任务表状态变化

```
pending → processing → completed / partial_completed / failed
```

- 始终停留 `pending`：主流程没抓到任务。
- 变成 `processing` 后长期不回写：流程中断或实例异常。

### 观察点 B：结果表新增记录

确认：是否新增记录、关联字段（任务ID/编号）是否对上、业务结果字段（URL/附件/内容）是否有值、错误信息是否为空。

### 端测评估标准

**通过**：任务进入 processing、执行完成、结果表有新增、结果字段有真实内容、状态正确回写。

**部分通过**：任务被抓取、跑到中后段、有结果写回，但部分字段缺失或部分失败。结论写："主链路可跑通，业务结果不完整。"

**不通过**：任务没被抓到、卡在 processing、结果表无新增、结果全是错误、execution 变成 crashed。

---

## 5. 测试结论模板

按问题类型分开记录，不混淆：

```md
### 测试结论
- 测试类型：API/脚本测试 或 端测
- 工作流 ID：${WORKFLOW_ID}
- 触发方式：webhook / manual
- 执行 ID：${EXEC_ID}
- 结果：通过 / 部分通过 / 不通过

### 关键事实
- Webhook 返回：...
- execution 状态：...
- 任务表状态：...
- 结果表：...

### 判断
- [结构问题] / [业务问题] / [实例问题]
- 一句话结论
```

**禁止**：用 "execution success" 单独作为通过结论。必须结合任务表状态推进、结果表写入、业务字段有值三项证据。
