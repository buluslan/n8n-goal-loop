# n8n 设计模式规范

> 设计 n8n 工作流节点链路与数据流时参考此规范——命名、数据流模式、累积策略、AI 传参、错误处理、环境适配。

---

## 1. 节点命名规范

**格式**: `功能描述 - 具体操作`

- 使用中文命名，便于阅读和维护。
- 同类节点使用相同前缀（技术/服务名），保持全局一致。
- 命名需自解释——看到节点名即知其职责。

**示例**:
- `Google Sheets - 读取数据`
- `Code - 验证数据格式`
- `Gemini AI - 内容打标`
- `Set - 格式化输出`
- `HTTP - 调用外部 API`

---

## 2. 三种数据流模式

### 模式 A：单次处理

```
输入 → 处理 → 输出
```

- **适用**: 一次性任务、单条记录转换、无迭代的线性流程。

### 模式 B：循环处理（Split In Batches）

```
输入 → Split In Batches → 循环体 ┐
        ←────────────────────────┘
        ↓
     后续处理
```

- **适用**: 逐条或分批处理大量数据（AI 调用、限流 API、逐项聚合）。
- `batchSize` 推荐 1–10；限流严格的 API 取 1，宽松批量取 5–10。
- 循环体内用 `_original` 透传数据，循环出口用 Aggregate 合并。
- 循环末尾回连 Split In Batches 节点形成回路。

### 模式 C：并行处理

```
    ┌── 分支 A ──┐
输入 →            ├─→ 合并 → 输出
    └── 分支 B ──┘
```

- **适用**: 多个相互独立的任务可同时执行（多维分析、多源抓取）。
- 用 Merge 节点汇总各分支结果。

---

## 3. 数据累积方案选择

| 方案 | 推荐度 | 适用场景 | 云端 | 本地 |
|------|--------|---------|------|------|
| **Aggregate 节点** | ★★★★★ | 所有需合并多批次结果的场景 | 推荐 | 推荐 |
| **静态数据 `$getWorkflowStaticData`** | ★★★★ | 跨执行持久化、计数器、增量游标 | 推荐 | 可用 |
| **临时文件** | ★★ | 仅本地开发调试 | 不推荐 | 可用 |

### 决策树

1. 只需在一次执行内合并批次结果？→ **Aggregate 节点**（首选）。
2. 需跨多次执行保留状态（游标、累计计数、缓存）？→ **静态数据**。
3. 纯本地调试、想快速落盘看中间产物？→ **临时文件**（部署前必须移除）。

### 边界：staticData 与多分支 Wait

- `$getWorkflowStaticData` **不能**作为跨 Wait 节点多分支的最终聚合点。
- Wait 节点恢复后的执行上下文与原始执行隔离，staticData 写入可能丢失或读到过期值。
- 多分支 Wait 场景：每分支用 Aggregate 收集本分支产物，再用 Merge 汇总，最后一次性写 staticData 或落库。
- 若需可靠的跨执行持久化，改用外部存储（数据库表 / 电子表格），而非依赖 staticData。

---

## 4. AI 数据传递规范（_original 模式）

**问题**: AI 节点只返回模型输出，输入字段会丢失，导致原始数据断链。

**标准模式**: 在送入 AI 前把原始数据塞进 `_original` 字段，AI 输出后再展开恢复。

```javascript
// 准备 AI 输入：保留原始数据
return $input.all().map(item => ({
  json: {
    prompt: `请分析：${item.json.content}`,
    _original: item.json   // 保留整份原始数据
  }
}));
```

```javascript
// 处理 AI 输出：恢复原始数据并附加结果
const item = $input.first().json;
const original = item._original;
const aiResult = item.response;   // AI 节点输出字段

return {
  json: {
    ...original,      // 恢复原始字段
    analysis: aiResult // 附加 AI 结果
  }
};
```

- 不要用 `pairedItem` 在云环境下做数据关联——不稳定。
- 解析 AI 返回的 Markdown 包裹 JSON 时，先剥离 ` ```json ` 标记再 `JSON.parse`，失败时返回 `{ error, raw }` 兜底。

---

## 5. 错误处理规范

### Code 节点：try-catch 包裹

```javascript
try {
  const result = processInput($input.first().json);
  return { json: result };
} catch (error) {
  return {
    json: {
      error: true,
      message: error.message,
      input: $input.first().json   // 回带输入便于排查
    }
  };
}
```

### HTTP Request：Continue + If 校验

- 节点 `Settings → On Error` 设为 **Continue**（失败不中断流程）。
- 下游接 **If 节点**检查 `{{ $json.status }}` / 业务字段，区分成功与失败分支。
- 失败分支记录原因、跳过或重试，避免单次外部调用失败拖垮整条工作流。

---

## 6. 环境适配

### 云端（容器化 / 托管 n8n）

**禁用**:
- 文件系统操作（`require('fs')`、`writeFileSync`）。
- Python Code 节点（多数镜像不内置 Python）。
- 依赖临时文件落盘的方案（容器重启即丢）。

**改用**:
- 状态持久化 → `$getWorkflowStaticData` 或外部存储。
- 脚本逻辑 → JavaScript（Code 节点）。
- 文件型存储 → 云端表格 / 对象存储 / 数据库。

### 本地（自部署、Docker）

- 文件系统、Python、本地数据库均可用，适合开发调试。
- **部署前**必须把所有文件系统依赖替换为云端兼容方案，避免迁移时返工。

---

## 要点速查

| 关注点 | 选择 |
|--------|------|
| 批量数据合并 | Aggregate 节点 |
| 跨执行状态 | `$getWorkflowStaticData` 或外部存储 |
| AI 前后数据透传 | `_original` 字段 |
| AI 返回解析 | 剥离 Markdown → JSON.parse → 兜底 |
| 外部调用容错 | HTTP Continue + If 校验 |
| 节点命名 | `服务 - 操作`，中文，一致前缀 |
| 云端禁区 | fs / Python / 临时文件 |
