# Domain 2 综合练习：MCP 工具设计与集成

## 练习目标
构建一个符合 `Domain 2: Tool Design & MCP Integration（工具设计与 MCP 集成）` 考试思路的小型训练案例，覆盖以下能力：

- 设计 3 个 `MCP tools（MCP 工具）`
- 其中故意制造 1 对 `ambiguous tool pair（有歧义的工具对）`
- 为工具写出正确版 `tool descriptions（工具描述）`
- 设计四类 `structured error responses（结构化错误响应）`
- 用 `.mcp.json` 配置 `project-level（项目级）` MCP 服务器
- 使用 `environment variable expansion（环境变量展开）`
- 设计 `tool_choice（工具调用策略）` 的强制首步

---

## 一、场景

场景选择：`Customer Support Resolution Agent（客服问题处理代理）`

用户常见请求：

> check the status of order #12345  
> find customer jane@example.com  
> can this customer get a refund for order #88291?

这个场景特别适合练 `Domain 2`，因为它天然包含：

- 容易误路由的相似工具
- 不同类型的错误
- 需要受限工具，而不是泛权限工具
- 需要明确的首步强制调用

---

## 二、先故意设计一个坏版本

我们先做一对会误导模型的工具：

### 工具 1
- 名称：`get_customer`
- 坏描述：`Retrieves customer information`

### 工具 2
- 名称：`lookup_order`
- 坏描述：`Retrieves order information`

### 工具 3
- 名称：`check_refund_eligibility`
- 坏描述：`Checks whether a refund is possible`

### 这个坏版本的问题
- `get_customer` 和 `lookup_order` 的描述都太泛
- 模型无法可靠区分：
  - 输入是什么
  - 哪种请求该调哪个
  - 哪些情况不该用该工具
- 这是典型的 `ambiguous tool pair（有歧义的工具对）`

---

## 三、把它修成考试喜欢的版本

### 工具 1：`get_customer`

**正确描述**

Use this tool to retrieve a customer profile by `customer_id（客户 ID）` or `email（邮箱）`. Best for requests such as "find customer jane@example.com", "look up customer C10291", or "show the customer account owner". Input must be a single customer identifier. Do not use this tool for order status, shipment tracking, or order-specific questions. Use `lookup_order` for order queries.

**输入契约**
- `customer_id` 或 `email`
- 单个客户标识

**边界**
- 适合：客户资料、账户所有者、客户标识查询
- 不适合：订单状态、物流、订单退款资格

---

### 工具 2：`lookup_order`

**正确描述**

Use this tool to retrieve order details and current order status by `order_id（订单 ID）`. Best for requests such as "check the status of order #12345", "has order 88291 shipped?", or "when was this order delivered?". Input must be a single order ID. Do not use this tool for customer profile lookups or account ownership checks. Use `get_customer` for customer records.

**输入契约**
- `order_id`
- 单个订单号

**边界**
- 适合：订单状态、发货、交付时间
- 不适合：客户资料、账户归属

---

### 工具 3：`check_refund_eligibility`

**正确描述**

Use this tool to check whether a specific order is eligible for refund under current refund policy. Best for requests such as "can order 88291 be refunded?" or "is this order within the refund window?". Input requires a single `order_id`. This tool returns eligibility and business-policy reasons only. It does not process refunds and does not verify customer identity.

**输入契约**
- `order_id`

**边界**
- 适合：退款资格判断
- 不适合：执行退款、客户身份验证、订单状态查询

---

## 四、四类错误响应

每个工具都应返回结构化错误，而不是模糊文本。

### 1. transient error（瞬时错误）

```json
{
  "isError": true,
  "errorCategory": "transient",
  "isRetryable": true,
  "message": "Order service timed out while retrieving order #12345."
}
```

### 2. validation error（校验错误）

```json
{
  "isError": true,
  "errorCategory": "validation",
  "isRetryable": false,
  "message": "Invalid order_id format. Expected a single numeric order ID."
}
```

### 3. business error（业务错误）

```json
{
  "isError": true,
  "errorCategory": "business",
  "isRetryable": false,
  "message": "Order #88291 is outside the refund window and is not eligible for refund."
}
```

### 4. permission error（权限错误）

```json
{
  "isError": true,
  "errorCategory": "permission",
  "isRetryable": false,
  "message": "Access denied. Current credentials cannot read customer records."
}
```

### 正常空结果示例

注意，这不是错误：

```json
{
  "isError": false,
  "results": []
}
```

这表示 `valid empty result（有效空结果）`，不是 `access failure（访问失败）`。

---

## 五、.mcp.json 配置

这份配置放在项目根目录，属于 `project-level（项目级）` 配置。

```json
{
  "servers": {
    "support-tools": {
      "command": "python",
      "args": ["./mcp/support_server.py"],
      "env": {
        "SUPPORT_API_TOKEN": "${SUPPORT_API_TOKEN}",
        "CUSTOMER_DB_TOKEN": "${CUSTOMER_DB_TOKEN}"
      }
    }
  }
}
```

### 为什么这样配
- `.mcp.json`：团队共享、受版本控制
- `${SUPPORT_API_TOKEN}`：避免把凭证写死进仓库
- 每位开发者本地设置自己的环境变量

---

## 六、tool_choice 强制首步

### 需求
对于任何退款请求，系统都必须先做 `check_refund_eligibility（检查退款资格）`，不能直接自由发挥。

### 错误做法
- 用 `auto（自动）`，让模型自己判断第一步

### 正确做法
对首步使用强制指定工具：

```json
{
  "tool_choice": {
    "type": "tool",
    "name": "check_refund_eligibility"
  }
}
```

### 为什么这符合考试偏好
- 这是明确的 `mandatory first step（强制前置步骤）`
- 适合用指定工具，而不是留给模型自由选择

---

## 七、标准测试集

### 测试 1：误路由修复验证

请求：

> check the status of order #12345

预期：
- 调用 `lookup_order`
- 不应调用 `get_customer`

### 测试 2：客户查询

请求：

> find customer jane@example.com

预期：
- 调用 `get_customer`

### 测试 3：退款资格首步强制

请求：

> can this customer get a refund for order #88291?

预期：
- 首步必须调用 `check_refund_eligibility`
- 不应直接输出文本

### 测试 4：校验错误

请求：

> check order status for ABC-???

预期：
- 返回 `validation error（校验错误）`
- 不应原样重试

### 测试 5：有效空结果

请求：

> find customer nobody@example.com

预期：
- 工具返回 `results: []`
- 代理应表明无匹配结果
- 不应重试三次

---

## 八、这道综合练习在考什么

### 2.1 Tool interface design（工具接口设计）
- 先修 `tool descriptions（工具描述）`
- 不先上分类器

### 2.2 Structured error responses（结构化错误响应）
- 四类错误必须分清
- `valid empty result（有效空结果）` 不等于错误

### 2.3 Tool distribution and tool_choice（工具分发与 tool_choice）
- 工具按角色分配
- 强制前置步骤用指定 `tool_choice（工具调用策略）`

### 2.4 MCP server integration（MCP 服务器集成）
- 用 `.mcp.json`
- 用 `environment variable expansion（环境变量展开）`

### 2.5 Built-in tools（内建工具）
- 这一题不直接考内建工具实现，但会间接考你是否理解“低成本、高杠杆”修复顺序

---

## 九、一句话标准答案

先设计一对故意含糊的工具描述，再把它们改写成带用途、输入、示例和边界的清晰描述；为工具定义四类结构化错误响应；将 MCP 服务器写入 `.mcp.json` 并通过环境变量注入凭证；最后对退款场景使用强制指定的 `tool_choice（工具调用策略）`，保证第一步一定调用 `check_refund_eligibility（检查退款资格）`。
