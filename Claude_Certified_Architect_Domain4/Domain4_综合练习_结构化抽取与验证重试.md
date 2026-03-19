# Domain 4 综合练习：结构化抽取与验证重试

## 练习目标
构建一个符合 `Domain 4: Prompt Engineering & Structured Output（提示工程与结构化输出）` 考试思路的小型抽取方案，覆盖以下能力：

- `explicit criteria（显式标准）`
- `few-shot prompting（少样本提示）`
- `tool_use + JSON schema（工具调用 + JSON 模式）`
- `validation-retry loops（验证-重试循环）`
- `batch vs synchronous（批处理 vs 同步）` 选择
- 抽取前后质量对比

---

## 一、目标场景

任务：从 10 份业务文档中抽取结构化发票数据。

文档格式混杂：
- 标准发票 PDF
- 内嵌表格的采购单
- narrative invoice email（叙述式发票邮件）
- 带 bibliography（参考文献）风格注释的供应商对账文件

目标字段：
- `invoice_number（发票号）`
- `invoice_date（开票日期）`
- `due_date（到期日）`
- `vendor_name（供应商名）`
- `customer_name（客户名）`
- `stated_total（声明总额）`
- `currency（货币）`
- `purchase_order_number（采购单号）`
- `document_type（文档类型）`
- `document_type_other_detail（文档类型其他说明）`
- `confidence（置信度）`
- `conflict_detected（冲突标记）`
- `calculated_total（计算总额）`

---

## 二、第一版错误设计（考试会故意给你的错法）

### 错法 1：prompt-only JSON（纯提示 JSON）

```text
Please return valid JSON with the following fields...
```

问题：
- 可能产生 `syntax errors（语法错误）`
- 结构不稳定

### 错法 2：所有字段都 required（必填）

问题：
- 文档没有 `purchase_order_number（采购单号）` 时，模型容易编造

### 错法 3：只写长说明，不给 few-shot（少样本）

问题：
- varied document formats（多样文档格式） 下判断不一致
- 明明有值却常常填 `null`

---

## 三、正确设计：tool_use + JSON schema（工具调用 + JSON 模式）

### 抽取工具

工具名：`extract_invoice_data`

### 设计目标
- 保证结构合法
- 对缺失信息提供合法出口
- 对模糊分类提供扩展出口

### 示例 schema 设计

```json
{
  "type": "object",
  "properties": {
    "invoice_number": { "type": ["string", "null"] },
    "invoice_date": { "type": ["string", "null"], "description": "Use YYYY-MM-DD when known." },
    "due_date": { "type": ["string", "null"], "description": "Use YYYY-MM-DD when known." },
    "vendor_name": { "type": ["string", "null"] },
    "customer_name": { "type": ["string", "null"] },
    "stated_total": { "type": ["number", "null"] },
    "calculated_total": { "type": ["number", "null"] },
    "currency": { "type": ["string", "null"] },
    "purchase_order_number": { "type": ["string", "null"] },
    "document_type": {
      "type": "string",
      "enum": ["invoice", "purchase_order", "statement", "receipt", "other", "unclear"]
    },
    "document_type_other_detail": { "type": ["string", "null"] },
    "confidence": { "type": "number" },
    "conflict_detected": { "type": "boolean" }
  },
  "required": [
    "document_type",
    "confidence",
    "conflict_detected"
  ]
}
```

### 为什么这样设计
- `required（必填）` 只保留真正必须存在的控制字段
- 对业务信息字段使用 `nullable（可空）`
- `document_type` 包含：
  - `other（其他）`
  - `unclear（不明确）`
- 避免为了满足 schema（模式） 而编造数据

---

## 四、Prompt（提示）中的 explicit criteria（显式标准）

### 错误提示

```text
Be careful and extract conservatively.
```

### 正确提示

```text
Extract only values that are explicitly supported by the document.
If a field is absent, return null.
If document type cannot be determined confidently, return "unclear".
If the document type does not fit the listed enums, return "other" and explain in document_type_other_detail.
Return invoice_date and due_date in YYYY-MM-DD format when present.
Set conflict_detected to true when totals or identities conflict within the source.
```

### 核心点
- 不说“保守一点”
- 直接定义：
  - 什么时候填值
  - 什么时候填 `null`
  - 什么时候填 `unclear`
  - 什么时候填 `other`

---

## 五、Few-shot prompting（少样本提示）

### 为什么需要
10 份文档格式不一致，仅靠字段说明不够稳定。

### few-shot 设计原则
- 使用 `2-4 examples（2 到 4 个示例）`
- 优先覆盖最易混淆的格式

### 示例覆盖
- 示例 1：标准发票 PDF
- 示例 2：表格型采购单
- 示例 3：叙述式邮件发票
- 示例 4：来源分散、需判 `unclear（不明确）` 的文件

### 每个示例应包含
- 输入片段
- 正确输出
- 为什么这么填
- 为什么不选其他看似合理字段

### 目的
- 减少 varied-format extraction failures（多格式抽取失败）
- 降低 `null` 误填
- 降低字段错位

---

## 六、Validation-retry loops（验证-重试循环）

### 验证规则示例
- `calculated_total` 应与 `stated_total` 一致
- 日期格式必须为 `YYYY-MM-DD`
- 当 `document_type = other` 时，`document_type_other_detail` 不应为空

### 重试时返回给模型的内容
1. `original document（原始文档）`
2. `failed extraction（失败抽取结果）`
3. `specific validation error（具体验证错误）`

### 示例

```json
{
  "validation_error": "calculated_total (135.00) does not match stated_total (120.00)"
}
```

### 有效场景
- 格式错误
- 字段放错
- 总额不一致

### 无效场景
- 文档根本没有 `purchase_order_number`

### 一句话
- 重试修“抽错”，不修“文档没写”

---

## 七、前后质量对比

### Before（无 few-shot、无验证重试）
- 10 份文档中：
  - 3 份出现字段错位
  - 2 份把存在的信息抽成 `null`
  - 1 份编造了 `purchase_order_number`
  - 2 份日期格式不一致

### After（加 few-shot + schema + validation-retry）
- 10 份文档中：
  - 字段错位显著下降
  - `null` 误填减少
  - 编造减少，因为 schema 允许 `null` / `unclear`
  - 日期格式趋于统一

### 考试表达重点
- few-shot（少样本） 改善 varied document formats（多样文档格式） 抽取质量
- schema（模式） 提高结构可靠性
- validation-retry（验证-重试） 修复结构合法但内容错误的结果

---

## 八、Batch vs synchronous（批处理 vs 同步）

### 如果这 10 份文档是夜间离线处理
- 可考虑 `Message Batches API（消息批处理 API）`

### 如果这是用户在等待结果的实时抽取
- 应使用 `Synchronous API（同步 API）`

### 为什么
- batch（批处理） 更便宜
- 但有最长 24 小时窗口
- 不适合 blocking workflow（阻塞型工作流）

---

## 九、标准答案总结

### 正确技术组合
- 用 `tool_use + JSON schema（工具调用 + JSON 模式）` 保证结构化输出合法
- 用 `explicit criteria（显式标准）` 明确填值边界
- 用 `few-shot prompting（少样本提示）` 解决 varied-format extraction（一致性与多格式抽取）
- 用 `validation-retry loops（验证-重试循环）` 修复格式错、字段错位与总额冲突
- 用 `batch vs sync（批处理 vs 同步）` 按延迟容忍度选接口

### 一句话标准答案
- schema 管结构，few-shot 管一致性，validation-retry 管自我修正，batch 只给不等结果的人用。
