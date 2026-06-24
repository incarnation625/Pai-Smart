# PaiSmart-Go 完整启动部署文档

> 本文档基于代码静态阅读整理，不包含实际编译、运行和测试结果。

## 1. 项目简介

PaiSmart-Go（派聪明）是一个面向企业知识库场景的 RAG 系统，提供文档上传、解析、切块、向量化、检索和 AI 问答能力，并通过组织标签实现多租户访问控制。

核心能力：

- 用户注册、登录、JWT 鉴权、刷新令牌
- 组织标签树管理与主组织切换
- 大文件分片上传、断点续传、秒传判断
- Kafka 异步文档处理流水线
- Tika 文本抽取
- Elasticsearch 向量检索 + 关键词检索混合搜索
- WebSocket 流式问答
- 管理员用户与会话审计能力

## 2. 技术栈

### 2.1 后端

- Go
- Gin
- GORM
- Viper
- JWT
- Gorilla WebSocket

### 2.2 前端

- Vue 3
- TypeScript
- Vite
- Pinia
- Naive UI
- UnoCSS

### 2.3 中间件与基础设施

- MySQL 8
- Redis 7
- MinIO
- Kafka
- Zookeeper
- Elasticsearch 8.10.4
- Apache Tika

### 2.4 AI 相关

- Embedding：兼容 OpenAI 协议，默认接入 DashScope
- LLM：默认 DeepSeek Chat，也支持按 OpenAI 兼容协议切换本地 Ollama

## 3. 项目目录

```text
cmd/server            应用入口
configs               配置文件
deployments           Dockerfile / docker-compose
docs                  数据库初始化脚本
internal/config       配置加载
internal/handler      HTTP / WebSocket 接口层
internal/middleware   鉴权、日志、中间件
internal/model        领域模型
internal/pipeline     文件处理流水线
internal/repository   数据访问层
internal/service      业务服务层
pkg/database          MySQL / Redis 初始化
pkg/storage           MinIO 客户端
pkg/kafka             Kafka 生产与消费
pkg/es                Elasticsearch 初始化与索引
pkg/tika              Tika 客户端
pkg/embedding         Embedding 客户端
pkg/llm               LLM 客户端
frontend              Vue 前端
homepage              官网/首页静态站点
initfile              启动时自动导入的初始化文件
```

## 4. 环境准备

### 4.1 本地基础环境

- Go 1.23 或更高版本
- Node.js 18+
- pnpm
- Docker / Docker Compose

### 4.2 需要准备的外部能力

- DashScope Embedding API Key
- DeepSeek API Key

如需使用本地大模型：

- Ollama
- 本地可用的兼容模型，例如 `deepseek-r1:7b`

## 5. 基础设施部署

项目提供的 `deployments/docker-compose.yaml` 主要负责基础设施，不直接启动 Go 后端和 Vue 前端。

### 5.1 Docker Compose 包含的服务

| 服务 | 端口 | 用途 |
| --- | --- | --- |
| MySQL | `3307 -> 3306` | 业务主库 |
| Redis | `6380 -> 6379` | 缓存、上传状态、会话历史 |
| MinIO | `9000/9001` | 对象存储与控制台 |
| Tika | `9998` | 文档解析 |
| Zookeeper | `2181` | Kafka 依赖 |
| Kafka | `9092` | 文件处理任务队列 |
| Elasticsearch | `9200` | 向量和全文检索 |

### 5.2 一键启动基础设施

```bash
docker compose -f deployments/docker-compose.yaml up -d
```

### 5.3 可选分服务拉取镜像

```bash
docker compose -f deployments/docker-compose.yaml pull mysql
docker compose -f deployments/docker-compose.yaml pull redis
docker compose -f deployments/docker-compose.yaml pull minio
docker compose -f deployments/docker-compose.yaml pull tika
docker compose -f deployments/docker-compose.yaml pull zookeeper
docker compose -f deployments/docker-compose.yaml pull kafka
docker compose -f deployments/docker-compose.yaml pull es
```

### 5.4 停止基础设施

```bash
docker compose -f deployments/docker-compose.yaml down
```

## 6. 服务启动顺序

建议按下面顺序启动：

1. 启动 Docker 基础设施
2. 确认 MySQL、Redis、MinIO、Kafka、Elasticsearch、Tika 已可访问
3. 修改后端配置 `configs/config.yaml`
4. 启动 Go 后端
5. 安装前端依赖并启动前端

原因：

- 后端启动时会立即初始化 MySQL、Redis、MinIO、ES、Kafka Producer
- 后端随后会启动 Kafka Consumer
- 文档上传后的异步处理链路依赖 Kafka、MinIO、Tika、ES 全部可用

## 7. 配置说明

配置文件：`configs/config.yaml`

### 7.1 服务配置

```yaml
server:
  port: "8081"
  mode: "debug"
```

说明：

- 默认后端监听 `8081`
- `mode` 支持 `debug/release/test`

### 7.2 数据库与缓存

```yaml
database:
  mysql:
    dsn: "root:密码@tcp(127.0.0.1:3307)/PaiSmart?charset=utf8mb4&parseTime=True&loc=Local"
  redis:
    addr: "127.0.0.1:6380"
    password: "密码"
    db: 0
```

### 7.3 JWT

```yaml
jwt:
  secret: "your-secret"
  access_token_expire_hours: 24
  refresh_token_expire_days: 7
```

### 7.4 Kafka

```yaml
kafka:
  brokers: "127.0.0.1:9092"
  topic: "file-processing"
```

### 7.5 MinIO

```yaml
minio:
  endpoint: "127.0.0.1:9000"
  access_key_id: "minioadmin"
  secret_access_key: "minioadmin"
  use_ssl: false
  bucket_name: "uploads"
```

### 7.6 Tika

```yaml
tika:
  server_url: "http://127.0.0.1:9998"
```

### 7.7 Elasticsearch

```yaml
elasticsearch:
  addresses: "http://127.0.0.1:9200"
  username: ""
  password: ""
  index_name: "knowledge_base"
```

说明：

- 服务启动时会自动检查并创建索引
- 映射中使用 IK 分词插件，因此 Compose 已在 ES 启动阶段自动安装 `analysis-ik`

### 7.8 Embedding

```yaml
embedding:
  model: "text-embedding-v4"
  api_key: ""
  base_url: "https://dashscope.aliyuncs.com/compatible-mode/v1"
  dimensions: 2048
```

### 7.9 LLM

```yaml
llm:
  base_url: "https://api.deepseek.com/v1"
  model: "deepseek-chat"
  api_key: ""
```

如果切换本地 Ollama，可按代码注释调整：

```yaml
llm:
  base_url: "http://localhost:11434/v1"
  model: "deepseek-r1:7b"
  api_key: ""
```

### 7.10 Prompt 与生成参数

配置文件中还包含：

- `llm.prompt`
- `llm.generation`
- `ai.prompt`
- `ai.generation`

其中聊天服务优先读取 `ai.prompt`，不存在时回退到 `llm.prompt`。

## 8. 后端启动

### 8.1 直接运行

```bash
go run cmd/server/main.go
```

### 8.2 可执行文件构建命令

```bash
go build -o bin/server cmd/server/main.go
```

说明：

- 本次文档整理未执行构建，仅记录代码中已有命令

### 8.3 后端启动时会做什么

后端入口 `cmd/server/main.go` 启动时会依次完成：

1. 加载配置
2. 初始化日志
3. 初始化 MySQL、Redis、MinIO、Elasticsearch、Kafka Producer
4. 初始化 Repository、Service、Processor
5. 启动 Kafka Consumer
6. 扫描 `initfile/` 目录并自动导入初始化文件
7. 注册 Gin 路由
8. 启动 HTTP 服务

### 8.4 `initfile` 自动导入机制

项目启动后会异步扫描 `initfile/` 目录中的文件，自动按“标准上传流程”导入系统：

- 归属用户优先选择 `admin`
- 自动切片、合并
- 默认设置为公开文件
- 自动触发 Kafka 后处理链路

这个设计适合做演示环境初始化。

## 9. 前端启动

```bash
cd frontend
pnpm install
pnpm run dev
```

可选命令：

```bash
pnpm run dev:prod
pnpm run build
pnpm run build:test
pnpm run lint
pnpm run typecheck
```

说明：

- 本次仅阅读代码，没有执行安装、构建、类型检查或 lint
- 前端请求通过统一 `request` 实例发起
- 开发态是否走 Vite 代理由环境变量 `VITE_HTTP_PROXY` 控制

## 10. 功能设计

## 10.1 用户与权限体系

### 角色

- `USER`
- `ADMIN`

### 组织标签模型

- 每个用户注册后自动拥有一个私有标签：`PRIVATE_<username>`
- 用户可以被授予多个组织标签
- 用户可以选择其中一个作为 `primaryOrg`
- 管理员可以维护标签树和给用户分配标签

### 文档访问规则

当前文档访问依赖三类条件：

1. 本人上传
2. `isPublic = true`
3. 文档 `orgTag` 命中用户有效组织标签范围，且文档公开

搜索时还会沿组织标签树向上递归，把父标签一起纳入有效访问范围。

## 10.2 文档上传与处理流程

### 上传阶段

1. 前端选择文件，计算 MD5
2. 按 5MB 分片上传
3. 后端把分片存入 MinIO：`chunks/{fileMd5}/{chunkIndex}`
4. 上传位图写入 Redis
5. 分片信息写入 MySQL `chunk_info`
6. 主上传记录写入 `file_upload`

### 合并阶段

1. 所有分片上传完成后，调用 `/upload/merge`
2. 单分片用 `CopyObject`
3. 多分片用 `ComposeObject`
4. 合并结果写入 `merged/{fileName}`
5. 上传状态更新为已完成
6. 向 Kafka 发布 `file-processing` 任务

### 异步处理阶段

Kafka Consumer 收到任务后执行：

1. 从 MinIO 取回合并文件
2. 调用 Tika 提取文本
3. 按 `chunkSize=1000`、`chunkOverlap=100` 切块
4. 切块文本写入 MySQL `document_vectors`
5. 调用 Embedding 接口生成向量
6. 将向量和文本写入 Elasticsearch

## 10.3 知识库搜索设计

搜索接口：`GET /api/v1/search/hybrid`

核心策略：

1. 先对用户问题做轻量归一化
2. 调用 Embedding 获取查询向量
3. 在 Elasticsearch 中执行：
   - `knn` 语义召回
   - `match` 关键词检索
   - `match_phrase` 兜底
   - `rescore` 重排
4. 结合 `user_id / org_tag / is_public` 过滤权限
5. 批量回查文件名
6. 返回分块文本、得分、来源文件

## 10.4 AI 聊天设计

聊天接口采用 WebSocket：

- 连接地址：`/chat/:token`
- 消息为纯文本输入
- 回答按 `chunk` 增量返回
- 支持停止命令
- 回答完成后发送 `completion`

RAG 过程：

1. 先走混合搜索获取上下文
2. 组装系统 Prompt 与上下文片段
3. 读取 Redis 中的最近会话历史
4. 调用 LLM 流式生成
5. 将问答结果回写 Redis

## 10.5 管理后台设计

管理员能力包括：

- 用户列表分页
- 为用户分配组织标签
- 组织标签增删改查
- 组织标签树展示
- 查看所有用户会话记录

## 11. 项目亮点

### 11.1 上传链路完整

- 分片上传
- 秒传检查
- 断点续传
- 自动合并
- 异步解析

### 11.2 RAG 链路完整

- 文档解析
- 切块
- 向量化
- 向量索引
- 混合搜索
- 流式回答

### 11.3 多租户能力明确

- 私有组织空间
- 公共文档
- 组织级共享
- 管理员集中分配组织标签

### 11.4 技术解耦较好

- `handler / service / repository` 分层清晰
- Kafka 将上传和处理流程解耦
- Redis 承担高频状态，MySQL 承担元数据

### 11.5 演示体验友好

- `initfile/` 自动导入演示文档
- WebSocket 实时输出 AI 回复
- 前端支持文档预览、搜索、上传进度可视化

## 12. 部署方式说明

## 12.1 Dockerfile

仓库提供 `deployments/Dockerfile`：

- 第一阶段使用 `golang:1.22-alpine` 构建
- 第二阶段使用 `alpine` 运行
- 容器中默认复制 `configs/`

说明：

- Dockerfile 目前只覆盖 Go 后端
- 前端未在同一镜像中打包发布

## 12.2 推荐部署拆分

建议按三层拆分：

1. 基础设施层：MySQL / Redis / Kafka / ES / MinIO / Tika
2. 应用层：Go API 服务
3. 前端层：Vite 构建产物，由 Nginx 托管

## 13. 已知实现现状

这些内容不是部署阻塞项，但值得在上线前统一：

1. 部分接口返回结构尚未完全统一
2. `users/me` 返回的 `orgTags` 与前端类型不一致
3. 用户会话接口未处理前端日期筛选参数
4. 搜索接口未使用前端传入的 `userId`
5. 会话历史当前只存 Redis，不是持久化审计方案
6. `conversations` 模型已定义，但 DDL 中未创建对应表

## 14. 推荐上线前检查项

1. 补齐 `embedding.api_key`
2. 补齐 `llm.api_key` 或切换本地 Ollama
3. 确认 MinIO bucket `uploads` 已创建
4. 确认 Elasticsearch IK 插件安装成功
5. 确认 Kafka topic `file-processing` 已创建
6. 统一前后端接口返回格式
7. 校验多租户权限链路

## 15. 默认访问信息

按当前配置文件与 DDL：

- 后端地址：`http://127.0.0.1:8081`
- 前端开发地址：`http://127.0.0.1:9527`
- MinIO 控制台：`http://127.0.0.1:9001`
- Elasticsearch：`http://127.0.0.1:9200`
- Tika：`http://127.0.0.1:9998`

初始化账号：

- 管理员：`admin`
- 普通用户：`testuser`

说明：

- DDL 中已写入默认用户数据
- 密码为 BCrypt 哈希，本文档不反推明文密码
