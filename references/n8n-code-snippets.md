# n8n Code 节点标准代码片段

> **用途**：Code 节点标准代码片段库，Claude 在 n8n Code 节点写代码时直接复用，避免重复踩坑。
> 字段名均已通用化（content / category / value），不绑定具体平台或项目。适用 JavaScript Mode（云部署禁用 Python 与 fs 模块）。

---

## 1. AI 数据传递标准模式

**用途**：AI 节点只返回输出本身，不携带输入数据、不保留 `pairedItem`。用 `_original` 显式保存原始数据，解析时恢复。

```javascript
// 准备 AI 输入节点
return $input.all().map(item => ({
  json: {
    prompt: `分析以下内容：\n${item.json.content}`,
    _original: item.json        // 保存原始数据
  }
}));

// 解析 AI 结果节点
const original = $input.first().json._original;
const aiResult = JSON.parse($input.first().json.output);
return [{ json: { ...original, category: aiResult.category, score: aiResult.score } }];
```

**注意**：不要依赖 `pairedItem.json`（AI 节点不保证存在）；不要只返回 AI 结果（原始字段全丢）；不要用 `$('上游').item.json` 反查（多分支下不稳定）。

---

## 2. 批量处理标准模式

**用途**：多条数据全部处理，避免只处理第一条导致结果坍缩。

```javascript
const items = $input.all();
const results = items.map((item, index) => {
  try {
    const processed = processData(item.json);   // 你的处理逻辑
    return { json: { ...item.json, ...processed, _index: index } };
  } catch (error) {
    return { json: { ...item.json, error: error.message } };   // 单条失败不影响整批
  }
});
return results;   // 输入输出数量一致
```

**注意**：必须 `$input.all()` + `map`；`$input.first()` 会把 N 条坍缩成 1 条。配合 Split In Batches 循环时，0 号输出接聚合节点（循环结束）、1 号输出接处理节点（每次循环）——口诀 **0 聚合，1 处理**。

---

## 3. 数据提取标准模式

**用途**：从 AI 纯文本输出中提取结构化字段，单正则不稳，用多模式降级。

```javascript
function extractField(text, patterns) {
  for (const pattern of patterns) {
    const match = text.match(pattern);
    if (match && match[1]) return match[1].trim();
  }
  return '未知';   // 合理默认值
}

const namePatterns = [
  /名称[：:]\s*([^\n]+)/i,
  /^1[.、]\s*([^\n]+)/m,
  /Name[:]\s*([^\n]+)/i
];
const valuePatterns = [
  /数值[：:]\s*[$¥￥€£]?\s*([\d,]+\.?\d*)/i,
  /(\d[\d,]*\.?\d*)/
];
const name  = extractField(aiResponse, namePatterns);
const value = extractField(aiResponse, valuePatterns);

return [{ json: { name, value, raw: aiResponse } }];   // 保留原始输出便于回溯
```

**注意**：每字段 3-5 个模式，精确到宽松排序；默认值用 `未知`/`N/A`（不要空串，否则下游难区分"提取失败"和"真为空"）；中英文各备一模式。

---

## 4. 静态数据标准模式

**用途**：跨节点/跨执行累积状态，替代被云部署禁用的 fs 模块。

```javascript
// 初始化（循环前一次）
const staticData = $getWorkflowStaticData('global');
staticData.results = [];

// 累积（循环中）
const staticData = $getWorkflowStaticData('global');
staticData.results = staticData.results || [];   // 防御性初始化
staticData.results.push(parsedItem);

// 读取（循环后）
const allData = $getWorkflowStaticData('global').results;
```

**注意**：简单循环优先用 Aggregate 节点（`aggregate: aggregateAllItemData`），更可靠无状态。**重要边界**：staticData 不可靠地用于"跨 `Wait` 的多分支最终聚合"——分支各自成功但聚合节点可能读空。正确做法：每分支落到独立 Finalize 节点，由最后分支显式读取：
```javascript
const branchA = $items('Finalize Branch A', 0, 0);
const branchB = $items('Finalize Branch B', 0, 0);
```

---

## 5. 数据验证标准模式

**用途**：送入 AI 前过滤无效数据（空值、表头行、类型错误、过短内容），省 API 调用、防崩溃。

```javascript
function validateItem(item) {
  let value = item.json.content || item.json.value;
  if (typeof value !== 'string') value = String(value);        // 层1：类型（数字等强转）
  if (!value || !value.trim()) return false;                    // 层2：空值（空格也要 trim）
  const specialValues = ['content', 'value', 'N/A', 'NULL'];    // 层3：表头/占位符
  if (specialValues.includes(value.trim())) return false;
  if (value.length < 5) return false;                           // 层4：长度
  return true;
}

const validItems = $input.all().filter(validateItem);
return validItems;
```

**注意**：Sheets/CSV 纯数字会被解析为 Number，`Number.trim` 不存在会崩，必须先 `String(value)`；空格字符串是 truthy，必须 `trim()` 后判空；CSV 表头字段名要加入黑名单。

---

## 6. 日期格式化标准模式

**用途**：生成文件名、时间戳。n8n 表达式 `{{ $now.format('YYYY-MM-DD') }}` 在部分节点原样输出不生效，用 JS 原生 API。

```javascript
const now = new Date();
const formats = {
  'YYYY-MM-DD':      now.toISOString().split('T')[0],                        // 2026-06-15
  'YYYYMMDD':        now.toISOString().split('T')[0].replace(/-/g, ''),      // 20260615
  'YYYY-MM-DD HH:mm': now.toISOString().slice(0, 16).replace('T', ' '),      // 2026-06-15 14:30
  'timestamp':       now.getTime(),
  'CN': `${now.getFullYear()}年${now.getMonth() + 1}月${now.getDate()}日`
};
const fileName = `output_${formats['YYYY-MM-DD']}.csv`;
```

**注意**：文件名/条件判断用 JS 原生，简单拼接可用 n8n 表达式；`getMonth()` 返回 0-11 要 +1；`toISOString()` 是 UTC，需本地时区时用 `toLocaleString`。

---

## 7. AI 输出解析标准模式

**用途**：AI 常用 ```` ```json ... ``` ```` 包裹 JSON 或前后带解释文字，直接 `JSON.parse` 会失败，需先清洗。

```javascript
function parseAIJson(raw) {
  if (!raw || typeof raw !== 'string') return null;
  let cleaned = raw.trim();
  cleaned = cleaned.replace(/^```(?:json)?\s*/i, '').replace(/\s*```$/, '');   // 去代码块
  const match = cleaned.match(/(\{[\s\S]*\}|\[[\s\S]*\])/);                    // 提取 JSON 主体
  if (match) cleaned = match[1];
  try {
    return JSON.parse(cleaned);
  } catch (e) {
    try { return JSON.parse(cleaned.replace(/,\s*([}\]])/g, '$1')); }          // 修复尾逗号
    catch (e2) { console.log('解析失败:', e2.message); return null; }
  }
}

const aiOutput = $input.first().json.output || $input.first().json.text;        // 字段名兼容
const parsed = parseAIJson(aiOutput);
if (!parsed) return [{ json: { error: 'AI 输出解析失败', raw: aiOutput } }];    // 失败带 raw 便于调试
return [{ json: { ...parsed, _parsedAt: new Date().toISOString() } }];
```

**注意**：AI 输出字段名可能是 `output`/`text`/`response`/`message.content`，先兼容取值；三层降级（去代码块→正则提取→修尾逗号）；失败必须返回带 `raw` 的错误项，不要静默吞错；纯文本非 JSON 时配合第 3 节正则提取。
