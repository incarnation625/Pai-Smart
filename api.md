# API 接口说明文档

> 基于当前仓库代码整理，范围覆盖 `cmd/server/main.go`、`internal/handler/*`、`internal/service/*`、前端 `frontend/src` 的真实调用方式。本文档不基于运行结果，仅基于静态阅读。

## 1. 文档范围与统一规范

### 1.1 服务入口

- HTTP 基础前缀：`/api/v1`
- WebSocket 连接地址：`/chat/:token`
- 前端开发态通常通过 Vite 代理访问后端，实际代理前缀由环境变量决定。

### 1.2 鉴权规范

- 登录后返回 `token` 与 `refreshToken`
- 受保护接口通过请求头传递：

```http
Authorization: Bearer <access_token>
```

- 管理员接口除登录态外，还要求用户 `role = ADMIN`
- 令牌刷新接口：`POST /api/v1/auth/refreshToken`

### 1.3 推荐统一响应结构

当前项目里响应格式存在两种风格：

- 风格 A：`{ code, message, data }`
- 风格 B：直接返回业务字段或 `{ error: "..." }`

建议统一为：

```json
{
  "code": 200,
  "message": "success",
  "data": {}
}
```

失败时统一为：

```json
{
  "code": 400,
  "message": "参数错误",
  "data": null
}
```

### 1.4 推荐统一分页结构

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "content": [],
    "totalElements": 0,
    "totalPages": 0,
    "size": 10,
    "number": 1
  }
}
```

### 1.5 推荐统一字段规范

- JSON 字段统一使用 `camelCase`
- Query 参数建议也统一用 `camelCase`
- 主键类字段统一命名：
  - 用户：`userId`
  - 文件：`fileMd5`
  - 组织标签：`tagId`
- 布尔值统一使用 `isXxx`

## 2. 当前代码中的前后端对接注意事项

以下问题已经在代码中出现，文档编写时一并标记，便于后续统一：

1. `GET /api/v1/users/me` 直接返回 `model.User`，其中 `orgTags` 实际是逗号分隔字符串，但前端类型期望 `string[]`。
2. `POST /api/v1/upload/check`、`POST /api/v1/upload/fast-upload` 当前返回裸对象，没有 `code/message/data` 包装。
3. `GET /api/v1/search/hybrid` 只使用 `query` 和 `topK`，前端传的 `userId` 参数会被忽略。
4. `GET /api/v1/users/conversation` 当前仅按登录用户读取 Redis 会话历史，不处理前端传入的日期筛选参数。
5. 聊天 WebSocket 当前实际发送的是纯文本消息；仓库说明里提到的 `{conversationId, query}` 协议，在代码中尚未实现。
6. 文档下载与预览接口按 `fileName` 查询文件，适合文件名唯一的前提；若同一用户或多用户存在同名文件，存在歧义风险。

## 3. 接口清单

### 3.1 认证模块

#### 3.1.1 刷新 Token

- 方法：`POST`
- 路径：`/api/v1/auth/refreshToken`
- 鉴权：否

请求体：

```json
{
  "refreshToken": "string"
}
```

成功响应：

```json
{
  "code": 200,
  "message": "Token refreshed successfully",
  "data": {
    "token": "string",
    "refreshToken": "string"
  }
}
```

### 3.2 用户模块

#### 3.2.1 用户注册

- 方法：`POST`
- 路径：`/api/v1/users/register`
- 鉴权：否

请求体：

```json
{
  "username": "string",
  "password": "string"
}
```

说明：

- 注册成功后自动创建私有组织标签：`PRIVATE_<username>`
- 新用户默认角色为 `USER`

#### 3.2.2 用户登录

- 方法：`POST`
- 路径：`/api/v1/users/login`
- 鉴权：否

请求体：

```json
{
  "username": "string",
  "password": "string"
}
```

成功响应：

```json
{
  "code": 200,
  "message": "Login successful",
  "data": {
    "token": "string",
    "refreshToken": "string"
  }
}
```

#### 3.2.3 获取当前用户信息

- 方法：`GET`
- 路径：`/api/v1/users/me`
- 鉴权：是

当前实现返回：

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "id": 1,
    "username": "admin",
    "role": "ADMIN",
    "orgTags": "PRIVATE_admin",
    "primaryOrg": "PRIVATE_admin",
    "createdAt": "2026-06-23T00:00:00Z",
    "updatedAt": "2026-06-23T00:00:00Z"
  }
}
```

推荐统一后返回：

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "id": 1,
    "username": "admin",
    "role": "ADMIN",
    "orgTags": ["PRIVATE_admin"],
    "primaryOrg": "PRIVATE_admin"
  }
}
```

#### 3.2.4 退出登录

- 方法：`POST`
- 路径：`/api/v1/users/logout`
- 鉴权：是

说明：

- 服务端会把当前 access token 加入 Redis 黑名单

#### 3.2.5 设置主组织

- 方法：`PUT`
- 路径：`/api/v1/users/primary-org`
- 鉴权：是

请求体：

```json
{
  "primaryOrg": "tag_id"
}
```

说明：

- 后端按当前登录用户处理，不读取 `userId`
- 只能设置为当前用户已有的组织标签之一

#### 3.2.6 获取当前用户组织标签

- 方法：`GET`
- 路径：`/api/v1/users/org-tags`
- 鉴权：是

成功响应：

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "orgTags": ["PRIVATE_admin", "TEAM_RD"],
    "primaryOrg": "TEAM_RD",
    "orgTagDetails": [
      {
        "tagId": "TEAM_RD",
        "name": "研发部",
        "description": "..."
      }
    ]
  }
}
```

### 3.3 文件上传模块

#### 3.3.1 秒传检查

- 方法：`POST`
- 路径：`/api/v1/upload/check`
- 鉴权：是

请求体：

```json
{
  "md5": "string"
}
```

当前实现响应：

```json
{
  "completed": true,
  "uploadedChunks": [0, 1]
}
```

推荐统一响应：

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "completed": true,
    "uploadedChunks": [0, 1]
  }
}
```

#### 3.3.2 上传分片

- 方法：`POST`
- 路径：`/api/v1/upload/chunk`
- 鉴权：是
- Content-Type：`multipart/form-data`

表单字段：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `file` | file | 是 | 当前分片 |
| `fileMd5` | string | 是 | 文件 MD5 |
| `fileName` | string | 是 | 原始文件名 |
| `totalSize` | number | 是 | 文件总大小 |
| `chunkIndex` | number | 是 | 当前分片序号，从 0 开始 |
| `orgTag` | string | 否 | 组织标签，空时自动回填当前用户主组织 |
| `isPublic` | boolean | 否 | 是否公开 |

成功响应：

```json
{
  "code": 200,
  "message": "分片上传成功",
  "data": {
    "uploaded": [0, 1, 2],
    "progress": 37.5
  }
}
```

说明：

- 后端分片大小固定为 `5MB`
- 支持的后缀：`.pdf .doc .docx .xls .xlsx .ppt .pptx .txt .md`

#### 3.3.3 合并分片

- 方法：`POST`
- 路径：`/api/v1/upload/merge`
- 鉴权：是

请求体：

```json
{
  "fileMd5": "string",
  "fileName": "string"
}
```

成功响应：

```json
{
  "code": 200,
  "message": "文件合并成功，任务已发送到 Kafka",
  "data": {
    "object_url": "string"
  }
}
```

说明：

- 合并成功后会发送 Kafka 消息，异步触发 Tika 解析、切块、向量化、ES 索引

#### 3.3.4 获取上传状态

- 方法：`GET`
- 路径：`/api/v1/upload/status`
- 鉴权：是

Query 参数：

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| `file_md5` | 是 | 文件 MD5 |

当前实现响应：

```json
{
  "code": 200,
  "message": "获取上传状态成功",
  "data": {
    "fileName": "demo.pdf",
    "fileType": "PDF文档",
    "uploaded": [0, 1],
    "progress": 50,
    "totalChunks": 4
  }
}
```

建议统一 Query 命名为 `fileMd5`

#### 3.3.5 获取支持的文件类型

- 方法：`GET`
- 路径：`/api/v1/upload/supported-types`
- 鉴权：是

成功响应：

```json
{
  "code": 200,
  "message": "获取支持的文件类型成功",
  "data": {
    "supportedExtensions": [".pdf", ".docx", ".txt"],
    "supportedTypes": ["PDF文档", "Word文档", "文本文件"],
    "description": "..."
  }
}
```

#### 3.3.6 快速秒传检查

- 方法：`POST`
- 路径：`/api/v1/upload/fast-upload`
- 鉴权：是

请求体：

```json
{
  "md5": "string"
}
```

当前实现响应：

```json
{
  "uploaded": true
}
```

### 3.4 文档管理模块

#### 3.4.1 获取当前用户可访问文档

- 方法：`GET`
- 路径：`/api/v1/documents/accessible`
- 鉴权：是

访问规则：

- 自己上传的文档
- `isPublic = true` 的文档
- `orgTag` 在当前用户组织范围内且 `isPublic = true` 的文档

响应示例：

```json
{
  "code": 200,
  "message": "success",
  "data": [
    {
      "id": 1,
      "fileMd5": "xxx",
      "fileName": "demo.pdf",
      "totalSize": 1024,
      "status": 1,
      "userId": 1,
      "orgTag": "TEAM_RD",
      "isPublic": true,
      "createdAt": "2026-06-23T00:00:00Z",
      "mergedAt": "2026-06-23T00:00:00Z"
    }
  ]
}
```

#### 3.4.2 获取当前用户上传的文档

- 方法：`GET`
- 路径：`/api/v1/documents/uploads`
- 鉴权：是

说明：

- 在 `file_upload` 基础上附加 `orgTagName`

#### 3.4.3 删除文档

- 方法：`DELETE`
- 路径：`/api/v1/documents/:fileMd5`
- 鉴权：是

说明：

- 普通用户只能删除自己上传的文件
- 管理员具备扩展删除能力，但当前实现删除数据库记录时仍按文件所属用户删除

#### 3.4.4 获取下载链接

- 方法：`GET`
- 路径：`/api/v1/documents/download`
- 鉴权：是

Query 参数：

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| `fileName` | 是 | 文件名 |

成功响应：

```json
{
  "code": 200,
  "message": "文件下载链接生成成功",
  "data": {
    "fileName": "demo.pdf",
    "downloadUrl": "https://...",
    "fileSize": 1024
  }
}
```

#### 3.4.5 预览文件内容

- 方法：`GET`
- 路径：`/api/v1/documents/preview`
- 鉴权：是

Query 参数：

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| `fileName` | 是 | 文件名 |

成功响应：

```json
{
  "code": 200,
  "message": "文件预览内容获取成功",
  "data": {
    "fileName": "demo.pdf",
    "content": "提取后的纯文本内容",
    "fileSize": 1024
  }
}
```

### 3.5 搜索模块

#### 3.5.1 混合搜索

- 方法：`GET`
- 路径：`/api/v1/search/hybrid`
- 鉴权：是

Query 参数：

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| `query` | 是 | 搜索文本 |
| `topK` | 否 | 返回条数，默认 `10` |
| `userId` | 否 | 前端当前会传，但后端未使用 |

成功响应：

```json
{
  "code": 200,
  "message": "success",
  "data": [
    {
      "fileMd5": "xxx",
      "fileName": "demo.pdf",
      "chunkId": 0,
      "textContent": "命中的分块内容",
      "score": 12.34,
      "userId": "1",
      "orgTag": "TEAM_RD",
      "isPublic": true
    }
  ]
}
```

说明：

- 检索逻辑为 `向量召回 + 文本 match + rescore`
- 权限过滤条件包括：
  - 本人文档
  - 公开文档
  - 当前用户有效组织标签范围内的公开文档

### 3.6 会话历史模块

#### 3.6.1 获取当前用户会话历史

- 方法：`GET`
- 路径：`/api/v1/users/conversation`
- 鉴权：是

成功响应：

```json
{
  "code": 200,
  "message": "success",
  "data": [
    {
      "role": "user",
      "content": "你好",
      "timestamp": "2026-06-23T10:00:00Z"
    },
    {
      "role": "assistant",
      "content": "你好，我是派聪明助手",
      "timestamp": "2026-06-23T10:00:05Z"
    }
  ]
}
```

说明：

- 当前实现仅返回 Redis 中当前会话的最近 20 条消息
- 前端传入的 `start_date`、`end_date` 在该接口中未生效

### 3.7 聊天模块

#### 3.7.1 获取 WebSocket 停止令牌

- 方法：`GET`
- 路径：`/api/v1/chat/websocket-token`
- 鉴权：否
- 说明：当前代码未加鉴权中间件，但前端通常在登录态下调用

成功响应：

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "cmdToken": "WSS_STOP_CMD_xxx"
  }
}
```

#### 3.7.2 建立聊天 WebSocket

- 方法：`GET`
- 路径：`/chat/:token`
- 鉴权：通过路径中的 JWT token 验证

当前前端发送协议：

- 普通聊天：直接发送纯文本
- 停止生成：

```json
{
  "type": "stop",
  "_internal_cmd_token": "WSS_STOP_CMD_xxx"
}
```

服务端返回消息类型：

1. 文本增量：

```json
{
  "chunk": "回答片段"
}
```

2. 停止确认：

```json
{
  "type": "stop",
  "message": "响应已停止",
  "timestamp": 1710000000000,
  "date": "2026-06-23T10:00:00"
}
```

3. 完成通知：

```json
{
  "type": "completion",
  "status": "finished",
  "message": "响应已完成",
  "timestamp": 1710000000000,
  "date": "2026-06-23T10:00:00"
}
```

4. 错误：

```json
{
  "error": "AI服务暂时不可用，请稍后重试"
}
```

推荐统一后的聊天消息协议：

```json
{
  "conversationId": "string",
  "query": "用户问题"
}
```

### 3.8 管理员模块

#### 3.8.1 用户列表

- 方法：`GET`
- 路径：`/api/v1/admin/users/list`
- 鉴权：管理员

Query 参数：

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| `page` | 否 | 页码，默认 1 |
| `size` | 否 | 每页条数，默认 10 |

成功响应：

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "content": [
      {
        "userId": 1,
        "username": "admin",
        "role": "ADMIN",
        "orgTags": [
          {
            "tagId": "PRIVATE_admin",
            "name": "admin的私人空间"
          }
        ],
        "primaryOrg": "PRIVATE_admin",
        "status": 0,
        "createdAt": "2026-06-23 10:00:00"
      }
    ],
    "totalElements": 1,
    "totalPages": 1,
    "size": 10,
    "number": 1
  }
}
```

说明：

- 当前后端不支持前端页面里提交的 `keyword`、`orgTag`、`status` 筛选

#### 3.8.2 为用户分配组织标签

- 方法：`PUT`
- 路径：`/api/v1/admin/users/:userId/org-tags`
- 鉴权：管理员

请求体：

```json
{
  "orgTags": ["PRIVATE_testuser", "TEAM_RD"]
}
```

#### 3.8.3 查询全部会话记录

- 方法：`GET`
- 路径：`/api/v1/admin/conversation`
- 鉴权：管理员

Query 参数：

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| `userid` | 否 | 指定用户 ID |
| `start_date` | 否 | 开始日期，格式 `YYYY-MM-DD` |
| `end_date` | 否 | 结束日期，格式 `YYYY-MM-DD` |

成功响应：

```json
{
  "code": 200,
  "message": "success",
  "data": [
    {
      "username": "testuser",
      "role": "assistant",
      "content": "回答内容",
      "timestamp": "2026-06-23T10:00:00"
    }
  ]
}
```

#### 3.8.4 创建组织标签

- 方法：`POST`
- 路径：`/api/v1/admin/org-tags`
- 鉴权：管理员

请求体：

```json
{
  "tagId": "TEAM_RD",
  "name": "研发部",
  "description": "研发团队标签",
  "parentTag": "COMPANY"
}
```

#### 3.8.5 查询组织标签列表

- 方法：`GET`
- 路径：`/api/v1/admin/org-tags`
- 鉴权：管理员

#### 3.8.6 查询组织标签树

- 方法：`GET`
- 路径：`/api/v1/admin/org-tags/tree`
- 鉴权：管理员

响应结构：

```json
[
  {
    "tagId": "COMPANY",
    "name": "公司",
    "description": "",
    "parentTag": null,
    "children": [
      {
        "tagId": "TEAM_RD",
        "name": "研发部",
        "description": "",
        "parentTag": "COMPANY",
        "children": []
      }
    ]
  }
]
```

#### 3.8.7 更新组织标签

- 方法：`PUT`
- 路径：`/api/v1/admin/org-tags/:id`
- 鉴权：管理员

请求体：

```json
{
  "name": "研发一部",
  "description": "更新后的描述",
  "parentTag": "COMPANY"
}
```

#### 3.8.8 删除组织标签

- 方法：`DELETE`
- 路径：`/api/v1/admin/org-tags/:id`
- 鉴权：管理员

## 4. 建议落地的统一接口改造清单

1. 所有接口统一返回 `{ code, message, data }`
2. `orgTags` 统一改成数组，避免字符串/数组双形态
3. `file_md5` 统一改为 `fileMd5`
4. 搜索接口删除无效 `userId` 参数，或补齐后端实现
5. 用户会话接口要么支持日期筛选，要么前端停止传参
6. WebSocket 请求体统一改成 JSON 协议，便于扩展 `conversationId`、多轮上下文、停止命令
7. 下载/预览接口建议改为按 `fileMd5` 查询，避免同名文件歧义
