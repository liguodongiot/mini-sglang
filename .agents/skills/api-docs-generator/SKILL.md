---
name: api-docs-generator
description: "从代码库生成全面、对开发者友好的 API 文档，包括端点、参数、示例和最佳实践。"
date_added: "2026-03-04"
---

# API 文档生成器

## 概述

从代码库自动生成清晰、全面的 API 文档。这项功能可帮助您创建专业的文档，其中包括端点描述、请求/响应示例、身份验证详情、错误处理和使用指南。

非常适合 REST API、GraphQL API 和 WebSocket API。

## 何时使用此技能

- 当你需要记录新的 API 时使用
- 当你用于更新现有 API 文档时使用
- 当你的 API 缺乏清晰的文档时使用
- 在引导新开发者熟悉你的 API 时使用
- 在为外部用户准备 API 文档时使用
- 在创建 OpenAPI/Swagger 规范时使用

## 工作原理

### 步骤一：分析 API 结构

首先，我会检查你的 API 代码库以了解：

- 可用端点和路由
- HTTP 方法（GET、POST、PUT、DELETE 等）
- 请求参数和请求体结构
- 响应格式和状态码
- 身份验证和授权要求
- 错误处理模式

### 步骤二：生成端点文档

针对每个端点，我将创建包含以下内容的文档：

**端点详情：**

- HTTP 方法和 URL 路径
- 简要描述其功能
- 身份验证要求
- 限速信息（如适用）

**请求规格：**

- 路径参数
- 查询参数
- 请求头
- 请求体模式（包含类型和验证规则）


**响应规范：**

- 成功响应（状态码 + 主体结构）
- 错误响应（所有可能的错误代码）
- 响应头


**代码示例：**

- cURL 命令
- JavaScript/TypeScript（fetch/axios）
- Python（requests）
- 根据需要使用其他语言


### 步骤三：添加使用指南

我将包括：

- 入门指南
- 身份验证设置
- 常见用例
- 最佳实践
- 限速详情


### 步骤四：文档错误处理

清晰的错误记录，包括：

- 所有可能的错误代码
- 错误消息格式
- 故障排除指南
- 常见错误场景及解决方案

### 步骤五：创建交互式示例

我会尽可能提供：

- OpenAPI/Swagger 规范
- 交互式代码示例
- 示例回答


## 示例

### 示例一：REST API 端点文档

```markdown
## 创建用户

创建一个新的用户账号。

**端点：** `POST /api/v1/users`

**认证：** Required (Bearer token)

**请求体：**
\`\`\`json
{
  "email": "user@example.com",      // Required: Valid email address
  "password": "SecurePass123!",     // Required: Min 8 chars, 1 uppercase, 1 number
  "name": "John Doe",               // Required: 2-50 characters
  "role": "user"                    // Optional: "user" or "admin" (default: "user")
}
\`\`\`

**成功的响应体 (201 Created)：**
\`\`\`json
{
  "id": "usr_1234567890",
  "email": "user@example.com",
  "name": "John Doe",
  "role": "user",
  "createdAt": "2026-01-20T10:30:00Z",
  "emailVerified": false
}
\`\`\`

**错误的响应体：**

- `400 Bad Request` - Invalid input data
  \`\`\`json
  {
    "error": "VALIDATION_ERROR",
    "message": "Invalid email format",
    "field": "email"
  }
  \`\`\`

- `409 Conflict` - Email already exists
  \`\`\`json
  {
    "error": "EMAIL_EXISTS",
    "message": "An account with this email already exists"
  }
  \`\`\`

- `401 Unauthorized` - Missing or invalid authentication token

**请求示例 (cURL):**
\`\`\`bash
curl -X POST https://api.example.com/api/v1/users \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "SecurePass123!",
    "name": "John Doe"
  }'
\`\`\`

**请求示例 (JavaScript):**
\`\`\`javascript
const response = await fetch('https://api.example.com/api/v1/users', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    email: 'user@example.com',
    password: 'SecurePass123!',
    name: 'John Doe'
  })
});

const user = await response.json();
console.log(user);
\`\`\`

**请求示例 (Python):**
\`\`\`python
import requests

response = requests.post(
    'https://api.example.com/api/v1/users',
    headers={
        'Authorization': f'Bearer {token}',
        'Content-Type': 'application/json'
    },
    json={
        'email': 'user@example.com',
        'password': 'SecurePass123!',
        'name': 'John Doe'
    }
)

user = response.json()
print(user)
\`\`\`
```

### 示例二：GraphQL API 文档

```markdown

## 用户查询

通过ID拉取用户信息。

**Query:**
\`\`\`graphql
query GetUser($id: ID!) {
  user(id: $id) {
    id
    email
    name
    role
    createdAt
    posts {
      id
      title
      publishedAt
    }
  }
}
\`\`\`

**变量：**
\`\`\`json
{
  "id": "usr_1234567890"
}
\`\`\`

**响应体：**
\`\`\`json
{
  "data": {
    "user": {
      "id": "usr_1234567890",
      "email": "user@example.com",
      "name": "John Doe",
      "role": "user",
      "createdAt": "2026-01-20T10:30:00Z",
      "posts": [
        {
          "id": "post_123",
          "title": "My First Post",
          "publishedAt": "2026-01-21T14:00:00Z"
        }
      ]
    }
  }
}
\`\`\`

**错误：**
\`\`\`json
{
  "errors": [
    {
      "message": "User not found",
      "extensions": {
        "code": "USER_NOT_FOUND",
        "userId": "usr_1234567890"
      }
    }
  ]
}
\`\`\`
```


## 最佳实践

### ✅ 建议这样做

- 保持一致性 - 所有端点使用相同的格式。
- 提供示例 - 提供多种语言的可运行代码示例
- 文档错误 - 列出所有可能的错误代码及其含义
- 展示真实数据 - 使用真实的示例数据，而不是“foo”和“bar”。
- 解释参数 - 描述每个参数的作用及其限制
- API 版本控制 - 在 URL 中包含版本号 (/api/v1/)
- 添加时间戳 - 显示文档上次更新时间
- 链接相关端点 - 帮助用户发现相关功能
- 包含速率限制 - 记录任何速率限制策略

### ❌ 请勿这样做

- 不要忽略错误案例 - 用户需要知道可能出现什么问题。
- 不要使用模糊的描述 - 例如，“获取数据”这样的描述毫无帮助。
- 不要忽略特殊情况 - 文档分页、筛选、排序
- 不要留下有问题的示例 - 测试所有代码示例
- 不要使用过时的信息 - 保持文档与代码同步
- 不要把事情复杂化 - 保持简洁易读。
- 别忘了响应头 — 记录重要的响应头

## 文档结构

### 推荐章节

1. 介绍
    - API 的功能
    - 基本 URL
    - API 版本

2. 快速入门
    - 简单示例入门
    - 常见示例演示

3. 端点
    - 按资源分类
    - 每个端点的完整详细信息

4. 数据模型
    - 规格定义
    - 字段描述
    - 验证规则

5. 错误处理
    - 错误代码参考
    - 错误响应格式
    - 故障排除指南

6. 速率限制
    - 限制和配额
    - 处理速率限制错误

7. 更新日志
    - API 版本历史记录
    - 重大变更
    - 弃用通知

9. SDK 和工具
    - 官方客户端库
    - OpenAPI 规范

