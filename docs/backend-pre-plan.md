# Backend Pre-Plan

版本: v0.1 草案  
日期: 2026-07-07  
适用项目: SSR Web Chat Bot  
目标规模: 小规模自用或半公开使用，用户数量预计不超过三位数

## 1. 结论

可以将后端数据库和后端服务部署在一台阿里云 ECS 云服务器上。对当前项目规模来说，这是可行且成本可控的方案。

推荐架构:

```txt
Frontend: Vercel Nuxt/Vue SSR
Backend API: 阿里云 ECS 上的 Node.js 服务
Database: PostgreSQL
File Storage: 第一版可本机磁盘，稳妥方案用阿里云 OSS
Reverse Proxy: Nginx
HTTPS: Certbot 或阿里云 SSL 证书
Process Runtime: Docker Compose 或 PM2
```

如果你希望降低服务器运维复杂度，推荐用 Docker Compose 管理 `api + postgres`。如果你更想理解后端运行细节，也可以不用 Docker，直接在 ECS 上安装 Node.js、PostgreSQL、Nginx 和 PM2。

## 2. 为什么仍然需要后端服务

当前应用第一版已经确认必须登录，并且存在这些服务端能力:

1. 多用户账号体系。
2. 会话、消息、文档、模型配置按用户隔离。
3. 用户自带模型 endpoint 和 API key 的服务端加密存储。
4. Markdown 文档上传和读取。
5. 流式 AI 对话请求代理。
6. 基础限流、审计和日志。

这些能力不适合完全放在浏览器端实现。后端服务需要负责认证、权限、密钥保护、数据库读写和模型服务调用。

## 3. 数据库选择

### 3.1 推荐: PostgreSQL

推荐使用 PostgreSQL。原因:

1. 当前业务数据是典型关系型数据: 用户、会话、消息、文档、模型配置、审计记录。
2. PostgreSQL 对事务、索引、外键和权限边界支持成熟。
3. PostgreSQL 支持 `jsonb`，适合存模型参数、token usage、错误信息、provider metadata。
4. 第三版如果要做 RAG，可以通过 `pgvector` 扩展支持向量检索，不需要立刻换数据库。
5. 小规模部署下，单机 PostgreSQL 足够支撑三位数用户。

### 3.2 不优先选择 MongoDB

MongoDB 不是不能用，但当前项目需要清晰的用户边界、会话关系、文档绑定、审计记录和模型配置关联。关系型数据库会更直接，查询和约束也更容易写正确。

### 3.3 不优先选择 SQLite

SQLite 适合本地原型，但不建议作为这个多用户 Web 应用的一版生产数据库。原因:

1. 多用户并发写入和备份恢复不如 PostgreSQL 稳。
2. 后续迁移到 PostgreSQL 仍需要重新设计部署。
3. 如果后端服务和数据库分开部署，SQLite 不适合作为共享服务数据库。

## 4. 后端框架选择

Express 和 Koa 都可行。

推荐优先级:

1. 如果你后端经验较少，优先 Express。
2. 如果你希望框架更轻、愿意自己组合中间件，可以用 Koa。

当前项目不需要复杂后端框架。关键不是 Express 还是 Koa，而是要把这些边界做好:

1. 认证中间件。
2. `user_id` 从服务端 session 派生。
3. 请求体验证。
4. API key 加密和解密。
5. 数据库查询必须按 owner 过滤。
6. 文件上传大小和类型校验。
7. 流式响应和超时控制。
8. 日志脱敏。

## 5. 阿里云 ECS 需要安装什么

### 5.1 推荐安装方式: Docker Compose

服务器系统建议选择 Ubuntu LTS 或 Debian 稳定版。

需要安装:

```txt
Docker
Docker Compose
Nginx
Certbot
Git
Node.js 20+ 或 22+
pnpm
PostgreSQL client tools, 例如 psql
```

说明:

1. Docker Compose 负责运行后端 API 和 PostgreSQL。
2. Nginx 负责反向代理和 HTTPS。
3. Certbot 负责自动申请和续期 Let’s Encrypt 证书。
4. Node.js/pnpm 主要用于本机调试、构建或非 Docker 部署。
5. `psql` 用于连接数据库、排查问题和执行迁移。

### 5.2 不使用 Docker 时需要安装

如果不用 Docker，则服务器上至少需要:

```txt
Node.js 20+ 或 22+
pnpm
PostgreSQL 16+
PM2
Nginx
Certbot
Git
```

这种方式也能跑，但 PostgreSQL、Node 服务、日志、备份和进程守护都需要你分别管理。

## 6. Node.js 后端建议依赖

无论使用 Express 还是 Koa，都建议使用 TypeScript。

### 6.1 Express 路线

```txt
express
cors
helmet
cookie-parser
zod
pg
drizzle-orm
jsonwebtoken 或 jose
bcrypt 或 argon2
multer 或 busboy
pino
dotenv
```

如果做 OAuth 登录，可以再引入:

```txt
passport
passport-github2 或其他 provider 包
```

### 6.2 Koa 路线

```txt
koa
@koa/router
koa-bodyparser
@koa/cors
koa-helmet
zod
pg
drizzle-orm
jsonwebtoken 或 jose
bcrypt 或 argon2
busboy
pino
dotenv
```

### 6.3 AI 调用相关

```txt
ai
@ai-sdk/openai
```

OpenAI-compatible endpoint 可以用 AI SDK 的 OpenAI provider 处理。服务端读取用户配置，解密 API key 后创建 provider，并通过后端代理流式返回给前端。

## 7. 数据库表建议

第一版建议至少包含:

```txt
users
auth_accounts
auth_sessions
chat_sessions
messages
documents
session_documents
model_endpoints
audit_events
```

核心原则:

1. 每张业务表都要有 `user_id` 或能通过父表关联到 `user_id`。
2. 客户端不能提交 `user_id` 来决定 owner。
3. 服务端必须从登录 session 取得 `user_id`。
4. 查询会话、文档、模型配置时必须校验 owner。
5. 用户 API key 只能保存加密后的密文。

## 8. 文件存储选择

### 8.1 第一版简化方案: ECS 本机磁盘

如果只是低流量、小规模使用，可以先把 Markdown 原文存到 ECS 本机磁盘，并在数据库中记录路径。

需要注意:

1. 路径必须按用户和文档 id 隔离。
2. 上传文件大小必须限制。
3. 只能允许 `.md`。
4. 需要定期备份上传目录。
5. 不能让 Nginx 直接暴露上传目录。

### 8.2 更稳妥方案: 阿里云 OSS

如果希望后续部署更稳，建议第一版就用 OSS。

优点:

1. 文件不依赖 ECS 本机磁盘。
2. 备份、迁移和扩容更清晰。
3. 后端只保存 object key，不保存公网可直接访问的永久 URL。

缺点:

1. 需要你创建 OSS bucket。
2. 需要你管理 AccessKey 或 RAM 子账号权限。
3. 开发时多一层配置。

## 9. 推荐部署拓扑

### 9.1 前端继续 Vercel，后端放 ECS

推荐方案:

```txt
browser
  -> Vercel Nuxt frontend
  -> https://api.example.com
  -> ECS Nginx
  -> Node.js API
  -> PostgreSQL
```

优点:

1. 前端仍然享受 Vercel 的部署体验。
2. 后端和数据库由你自己控制。
3. 用户自带 API key 可以安全存到自己的数据库。
4. 成本适合小规模使用。

需要处理:

1. CORS。
2. Cookie domain 和 SameSite。
3. HTTPS。
4. API 域名。
5. 前后端环境变量分别配置。

### 9.2 前后端都放 ECS

也可以把 Nuxt 和后端 API 都放 ECS。

优点:

1. 架构更集中。
2. Cookie/session 配置更简单。
3. 不依赖 Vercel。

缺点:

1. 你需要自己管理前端构建、SSR 进程、Nginx 和部署。
2. 运维负担更高。

当前建议: 前端继续 Vercel，后端和数据库放 ECS。

## 10. 你需要手动完成的任务

这些任务涉及账号、服务器权限、域名、密钥或云资源，通常需要你亲自完成:

1. 购买或准备阿里云 ECS。
2. 选择服务器系统，建议 Ubuntu LTS 或 Debian。
3. 配置阿里云安全组，开放 `22`、`80`、`443`，不要公网开放数据库端口。
4. 准备域名，并把 API 子域名解析到 ECS，例如 `api.example.com`。
5. 登录服务器，确认使用 root 还是创建单独部署用户。
6. 安装 Docker、Docker Compose、Nginx、Certbot、Git。
7. 配置 GitHub deploy key 或服务器可用的代码拉取方式。
8. 准备后端环境变量。
9. 准备数据库密码。
10. 准备 JWT/Auth secret。
11. 准备用于加密用户 API key 的 master secret。
12. 如果使用 OSS，创建 bucket 和 RAM 子账号，配置最小权限 AccessKey。
13. 申请 HTTPS 证书，或允许 Certbot 自动申请。
14. 决定登录方式: OAuth、邮箱验证码、用户名密码。
15. 决定前端部署位置: 继续 Vercel，还是迁到 ECS。

## 11. 后端必须实现的任务

这些可以由开发实现:

1. 搭建 Node.js + Express/Koa 服务。
2. 配置 TypeScript、lint、环境变量读取。
3. 设计 PostgreSQL schema 和 migration。
4. 实现登录和 session 中间件。
5. 实现 user scope 数据访问。
6. 实现会话 CRUD。
7. 实现消息存储和流式聊天接口。
8. 实现 Markdown 上传、读取、绑定当前会话。
9. 实现 OpenAI-compatible endpoint 配置。
10. 实现用户 API key 加密存储。
11. 实现 endpoint 连接测试。
12. 实现 SSRF 防护和 header 白名单。
13. 实现基础 rate limit。
14. 实现 audit events。
15. 实现日志脱敏。
16. 写 Dockerfile 和 docker-compose。
17. 写 Nginx 反向代理配置。
18. 写数据库备份脚本。

## 12. 安全底线

必须遵守:

1. 数据库端口不对公网开放。
2. 用户 API key 不返回给浏览器。
3. API key 加密密钥不存数据库，只放服务器环境变量。
4. 后端日志不能打印 Authorization header、API key、完整用户私密文档。
5. 自定义 endpoint 必须禁止访问 localhost、私有网段、metadata IP。
6. 上传文件必须限制大小和类型。
7. 所有业务 API 必须要求登录。
8. 所有业务查询必须按当前登录用户过滤。
9. 生产环境必须使用 HTTPS。
10. 数据库和上传文件必须有备份策略。

## 13. 建议的第一版实施顺序

1. 准备 ECS、域名、安全组、HTTPS。
2. 使用 Docker Compose 跑通 PostgreSQL。
3. 搭建 Node.js API skeleton。
4. 实现登录和 `user_id` 派生。
5. 实现 session/message 数据模型。
6. 实现流式聊天 API。
7. 实现 Markdown 上传。
8. 实现模型 endpoint 配置。
9. 实现用户 API key 加密存储。
10. 接入前端。
11. 加入日志、限流和备份。

## 14. 当前推荐决策

1. 数据库: PostgreSQL。
2. 后端框架: Express，除非你明确更喜欢 Koa。
3. 部署方式: Docker Compose。
4. 文件存储: 第一版可以本机磁盘，推荐尽早切到阿里云 OSS。
5. 前端部署: 继续 Vercel。
6. 后端部署: 阿里云 ECS。
7. API key 存储: 数据库加密存储。
8. 登录: 第一版必须做。
