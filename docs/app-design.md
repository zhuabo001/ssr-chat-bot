# SSR Web Chat Bot 应用设计文档

版本: v0.3 草案  
日期: 2026-07-07  
目标平台: Vercel Hobby  
技术基线: Vue 3 + Nuxt SSR + Nitro server API

## 1. 设计简报

构建一个可部署到 Vercel Hobby 的服务端渲染 Web 应用。应用的第一核心体验是登录后的 AI 流式对话工作台；第一版必须包含登录，未登录用户不能进入会话、文档、模型配置等业务界面。应用同时支持 Markdown 文档上传、自定义 OpenAI-compatible 模型服务端点配置、会话侧边栏切换，以及高度复刻 Codex 桌面端的工作台布局和视觉风格。

当前仓库基本为空，因此本文档按从零搭建方案设计。

## 1.1 已确认决策

1. 账号体系: 第一版必须做登录。多用户账号体系是 V1 前置能力，不提供匿名工作台或访客聊天模式。
2. API key 存储: 平台级默认模型 key 可以先放 Vercel env；如果允许每个用户配置自己的 URL 和 API key，第一版就需要服务端数据库加密存储，不能只依赖 Vercel env。
3. Markdown 文档: 第一版只绑定当前会话，第三版再做跨会话文档库。
4. RAG/向量检索: 第一版不做，第三版再引入 embeddings、chunking、semantic search。
5. UI 文案语言: 跟随浏览器语言，至少预留中文和英文 i18n。
6. UI 风格: 高度复刻 Codex 桌面端布局和视觉，而不是只借鉴信息密度。
7. 部署目标: Vercel Hobby。
8. 模型 provider 第一优先级: OpenAI-compatible。
9. 登录是会话、消息、文档、模型配置、限流、审计和 BYOK 的上游依赖。所有业务对象都必须从服务端 session 派生 `user_id`。

## 2. 产品目标

### 2.1 MVP 目标

1. 用户必须先注册/登录，登录后才能进入工作台。
2. 用户可以创建、切换、重命名、删除会话。
3. 用户可以在当前会话中与 AI 进行流式对话。
4. 用户可以上传 `.md` 文档，并将文档内容作为当前会话上下文使用。
5. 用户可以选择 OpenAI-compatible 模型服务端点。
6. 若第一版支持用户自带 key，用户可以配置自己的 endpoint、model、api key、额外 headers、默认参数，API key 由服务端加密后保存。
7. 若第一版暂不支持用户自带 key，则只提供由平台管理员通过 Vercel env 配置的默认 endpoint。
8. 应用可以部署在 Vercel Hobby，避免依赖本地持久化文件系统。
9. UI 第一屏就是可用聊天工作台，不做营销型 landing page。

### 2.2 非目标

1. 第一版不做复杂 RAG 检索、向量库、多文档语义搜索。
2. 第一版不做团队协作、共享会话、多人实时编辑。
3. 第一版不做插件系统、工具调用市场、复杂 agent 编排。
4. 第一版不承诺支持任意文件类型，仅支持 Markdown。
5. 第一版不把任何 API key 暴露给浏览器端 AI SDK 直接调用。
6. 第一版不做跨会话文档库。
7. 第一版不提供匿名聊天、访客会话、未登录本地草稿同步。

## 3. 关键产品决策

### 3.1 技术栈

推荐使用 Nuxt 作为 Vue 3 SSR 应用框架。原因:

1. Nuxt 支持 Vue 3、SSR、文件路由、服务端 API 和 Nitro 运行时。
2. Vercel 官方文档说明 Nuxt 的静态和 SSR 站点可以零配置部署到 Vercel。
3. Nuxt 在 Vercel 上会把 `server/api`、`server/routes`、`server/middleware` 打包为 server-rendered Function，适合承载聊天 API、上传 API 和配置 API。

建议初始依赖:

```txt
nuxt
vue
typescript
zod
ai
@ai-sdk/vue
@ai-sdk/openai
@vercel/blob
drizzle-orm
postgres 或 @neondatabase/serverless
tailwindcss
lucide-vue-next
```

说明: Vercel Postgres 已不再作为新项目产品提供，官方推荐通过 Marketplace 接入 Postgres provider，例如 Neon。因此数据库层不要写死为旧的 Vercel Postgres 产品。

### 3.2 运行时与部署

1. 构建命令: `nuxt build`，不要用 `nuxt generate` 作为主路径，因为聊天和配置 API 需要 SSR/server functions。
2. 部署平台: Vercel Hobby。
3. SSR 策略: 应用 shell、会话页、设置页启用 SSR；聊天消息流由客户端触发 API 流。
4. API 运行时: 默认 Node.js Function。暂不使用 Edge Runtime，因为自定义 provider SDK、数据库连接、对象存储和流式响应对 Node 兼容性更稳。
5. 函数时长: 聊天流需要受 Vercel Hobby Function duration 限制约束，前端必须支持中止、重试、错误恢复；服务端应设置低于平台上限的应用级超时。

### 3.3 持久化

Vercel 部署环境不应依赖本地文件写入。推荐:

1. 关系型数据: Marketplace Postgres，例如 Neon。存储用户、会话、消息、模型配置、附件元数据。
2. 文件对象: Vercel Blob。存储上传的 Markdown 原文，第一版建议使用 private blob。
3. 可选缓存/限流: Upstash Redis。V1 必须至少按 authenticated user 做基础请求预算；低流量演示可先用数据库计数，公开部署建议接入外部限流存储。
4. 平台默认模型密钥: Vercel env。
5. 用户自带模型密钥: 数据库加密字段，不能使用 Vercel env 作为动态用户 secret 存储。

### 3.4 API key 存储决策

Vercel env 适合存平台级、部署级配置，例如 `DEFAULT_OPENAI_COMPATIBLE_BASE_URL`、`DEFAULT_OPENAI_COMPATIBLE_API_KEY`、`DEFAULT_MODEL`。它不适合存每个用户在应用内提交的 URL/API key，原因:

1. Env 是项目或团队级配置，不是应用业务数据模型。
2. Env 变更不会作用到旧部署，更新后需要重新部署，不适合用户在设置页实时保存。
3. Env 的可见性和权限跟 Vercel 项目成员相关，不跟应用内 user scope 绑定。
4. Env 总大小有限，不适合承载多个用户的动态 secret 集合。
5. 用户删除、轮换、禁用自己的 key 需要业务级审计和权限控制，应该由数据库记录承载。

因此第一版需要二选一:

1. `v1-env-only`: 第一版只支持平台管理员配置一个或少量默认 OpenAI-compatible endpoint，用户不能保存自己的 API key。此路径可以完全使用 Vercel env，但仍然必须登录，因为会话、消息、文档和用量都按用户隔离。
2. `v1-byok`: 第一版支持用户自己配置 URL 和 API key。此路径必须在第一版引入数据库加密密钥存储，并把模型配置纳入 user scope。

当前产品目标如果坚持“用户自己配自己的 URL 和 API key”，推荐选择 `v1-byok`，不要把该能力推到第二版，否则设置页只能做 endpoint/model 选择，不能真正 BYOK。

## 4. 用户体验设计

### 4.1 信息架构

主应用采用三层工作台结构:

1. 登录边界: 未登录用户只看到登录页或 auth provider 跳转，不能加载工作台数据。
2. 左侧侧边栏: 会话列表、搜索、新建会话、设置入口。
3. 中间主区域: 当前会话消息流、文档引用提示、输入框。
4. 右侧检查器: 当前会话上下文、上传文档、模型参数、运行状态。右侧检查器在小屏幕上改为抽屉。

### 4.2 主要页面

#### `/`

受保护的应用工作台。未登录时跳转到登录流程；登录后默认打开最近会话；若没有会话，创建当前用户的临时空会话。

#### `/login`

登录入口。展示精简登录面板和 OAuth/email 登录按钮，视觉应贴近 Codex 桌面端的工具面板，不做营销 hero。

#### `/auth/callback`

认证回调页。由所选 auth provider 处理，成功后回到用户原本想访问的工作台路由。

#### `/chat/:sessionId`

受保护的指定会话工作台。SSR 阶段必须校验 session owner 后才能加载会话元数据和历史消息，客户端接管后继续流式对话。

#### `/settings/models`

受保护的模型端点配置页。支持新增、编辑、测试连接、设置默认模型；所有配置都绑定当前登录用户。

#### `/settings/profile`

受保护的用户偏好页。第一版可以只保留主题、默认模型、语言策略、数据保留策略。语言默认跟随浏览器，可手动覆盖为中文或英文。

### 4.3 Codex 桌面端视觉复刻原则

设计方向不是营销型 AI 产品，而是高度贴近 Codex 桌面端的开发者工作台:

1. 三栏工作台优先，左侧会话列表、中间对话、右侧上下文检查器。
2. 深色主题作为默认呈现，同时支持系统跟随和浅色主题。
3. 低饱和中性色、细边框、克制阴影，避免营销站式渐变、hero 和装饰图形。
4. 左侧线程列表突出标题、更新时间、模型标记、文档标记，交互密度接近桌面端。
5. 输入区是命令中心，支持上传、模型选择、发送、中止。
6. 圆角控制在 6px 到 8px，按钮和列表项保持紧凑。
7. 尽量使用图标按钮和 tooltip，不用冗长说明文本占据工作区。
8. 页面不嵌套卡片。重复项可以是列表行，设置项可以是分组面板。
9. 空状态、错误状态、设置表单也应保持工具感，不做大插画和营销式说明。

### 4.4 核心组件

```txt
AppShell
  Sidebar
    SessionSearch
    SessionList
    SidebarFooter
  ChatWorkspace
    ChatHeader
    MessageList
    EmptyConversation
    Composer
  ContextInspector
    DocumentPanel
    ModelRunPanel
    SessionSettingsPanel
```

消息组件:

1. `UserMessage`: 显示用户输入、附件 chip。
2. `AssistantMessage`: 支持 Markdown 渲染、代码块、复制按钮、重新生成。
3. `SystemNotice`: 显示模型切换、上传成功、上下文截断等状态。
4. `StreamingCursor`: 显示当前流式输出状态。
5. `ErrorMessage`: 支持重试和查看请求摘要。

Composer 控件:

1. 文本输入。
2. 上传 Markdown。
3. 模型选择器。
4. temperature/max tokens 等快速参数入口。
5. Send/Stop 按钮。

## 5. 功能设计

### 5.1 会话管理

会话生命周期:

1. 新建会话: 从服务端 session 取得 `user_id`，创建属于当前用户的 draft session。
2. 首条消息发送后，自动生成标题。
3. 切换会话时保存当前输入草稿。
4. 删除会话需要二次确认。
5. 会话列表支持按更新时间倒序和本地搜索。

会话标题生成:

1. MVP 可以用第一条用户消息前 40 字作为标题。
2. 后续可调用低成本模型异步生成标题。

### 5.2 聊天流

前端使用 `@ai-sdk/vue` 的 `useChat` 或封装后的 `useConversationStream`。服务端使用 AI SDK Core 调用 provider 并返回 UI message stream。

基本流程:

1. 前端在登录态下发送 `sessionId`、用户消息、选定 `modelEndpointId`、附件引用。
2. 服务端从 auth session 取得 `user_id`，拒绝匿名请求。
3. 服务端校验 session owner、模型配置 owner、附件绑定关系和附件可访问性。
4. 服务端读取最近消息和选定 Markdown 文档内容。
5. 服务端构造 prompt，按 token budget 截断上下文。
6. 服务端调用模型 provider 并流式返回。
7. 前端边收边渲染。
8. 流结束后服务端按当前 `user_id` 持久化 assistant message 和 usage metadata。

异常处理:

1. 用户中止: 保存 user message，assistant message 标记为 aborted。
2. Provider 超时: 显示可重试状态。
3. 内容超长: 服务端返回上下文截断说明。
4. API key 错误: 引导用户去模型设置页修复。

### 5.3 Markdown 上传

第一版只支持 `.md`:

1. 文件大小默认上限: 1 MB 到 5 MB，最终值待确认。
2. MIME/type 和扩展名双重校验。
3. 服务端读取文本，拒绝二进制和过大文件。
4. 原文存入 private Vercel Blob。
5. 提取纯文本摘要、标题、字符数、估算 token 数存入数据库。
6. 渲染 Markdown 时必须 sanitize HTML，默认不执行内联 HTML。

上下文策略:

1. 上传文档默认绑定到当前登录用户和当前会话。
2. 发送消息时，只有被勾选的文档进入 prompt。
3. 若文档超过上下文预算，第一版采用头部、标题层级和最近选区组合截断。
4. 第三版再引入跨会话文档库、embeddings、chunking、semantic search。

### 5.4 自定义模型端点

模型端点配置字段:

```txt
id
userId
displayName
providerType: openai-compatible
baseUrl
model
apiKeySource: platform-env | encrypted-db
apiKeyEnvName
apiKeyCiphertext
apiKeyIv
apiKeyTag
headers
defaultTemperature
defaultMaxOutputTokens
enabled
createdAt
updatedAt
```

第一版优先支持 OpenAI-compatible endpoint，因为 AI SDK 的 OpenAI provider 支持 `createOpenAI({ baseURL, apiKey, headers })`。这样可以兼容 OpenAI、OpenRouter、部分自建代理和兼容 OpenAI Chat Completions/Responses 的服务。

配置模式:

1. 平台默认 endpoint: 管理员通过 Vercel env 配置，所有用户可选择但不可查看 key。
2. 用户自带 endpoint: 用户在设置页提交 base URL、model、api key。服务端只在创建/更新时接收明文，立即用服务端密钥加密后存入数据库，之后 UI 只显示 masked 状态。
3. 第一版如果选择 `v1-env-only`，模型设置页只能允许已登录用户选择平台 endpoint 和调整非敏感参数。
4. 第一版如果选择 `v1-byok`，模型设置页必须完整支持用户自带 endpoint 和加密密钥存储。

安全限制:

1. API key 不下发给浏览器。
2. 服务端调用自定义 endpoint 前做 SSRF 防护: 只允许 `https`，阻止 localhost、私有网段、metadata IP、file 协议。
3. headers 使用白名单，禁止用户覆盖 `host`、`connection`、`content-length` 等敏感头。
4. endpoint 配置必须有连接测试。
5. 用户自带 API key 必须加密存储；平台默认 API key 可以使用 Vercel env。
6. 数据库加密需要一个部署级 master secret，例如 `APP_SECRET_KEY`，该 secret 存在 Vercel env 中，用于加解密用户 key。数据库泄露时不能直接暴露明文 key。
7. 服务端日志、错误响应和 telemetry 都不能记录明文 key、Authorization header 或完整上游请求头。

### 5.5 设置体验

模型设置页应支持:

1. OpenAI-compatible presets: OpenAI、OpenRouter、自定义兼容端点。
2. Endpoint 表单实时校验。
3. Test 按钮发送最小请求，不创建会话消息。
4. 默认模型标记。
5. 隐藏 API key，仅允许替换或删除。
6. 每个会话可覆盖默认模型。
7. `v1-env-only` 模式下，隐藏用户 API key 输入，只展示平台已配置 endpoint。
8. `v1-byok` 模式下，允许用户新增、测试、禁用自己的 endpoint。

## 5.6 国际化

第一版采用浏览器语言作为默认语言:

1. 中文浏览器默认中文 UI。
2. 非中文浏览器默认英文 UI。
3. 用户可以在 profile 中手动覆盖语言。
4. 所有空状态、错误、表单标签、toast、确认文案都从 i18n dictionary 读取。
5. 服务端错误码稳定，前端按语言映射展示文案。

## 6. 数据模型草案

```sql
users (
  id,
  email,
  display_name,
  avatar_url,
  auth_provider,
  auth_subject,
  email_verified_at,
  created_at,
  updated_at
)

auth_accounts (
  id,
  user_id,
  provider,
  provider_account_id,
  created_at,
  updated_at
)

auth_sessions (
  id,
  user_id,
  expires_at,
  created_at,
  updated_at
)

model_endpoints (
  id,
  user_id,
  display_name,
  provider_type,
  base_url,
  model,
  api_key_source,
  api_key_env_name,
  api_key_ciphertext,
  api_key_iv,
  api_key_tag,
  headers_json,
  defaults_json,
  enabled,
  last_tested_at,
  last_test_status,
  created_at,
  updated_at
)

chat_sessions (
  id,
  user_id,
  title,
  default_model_endpoint_id,
  archived_at,
  created_at,
  updated_at
)

messages (
  id,
  session_id,
  role,
  content,
  status,
  model_endpoint_id,
  token_usage_json,
  error_json,
  created_at,
  updated_at
)

documents (
  id,
  user_id,
  filename,
  blob_url,
  content_sha256,
  size_bytes,
  title,
  summary,
  token_estimate,
  created_at
)

session_documents (
  session_id,
  document_id,
  enabled,
  created_at
)

audit_events (
  id,
  user_id,
  type,
  metadata_json,
  created_at
)
```

第一版必须保留真实 `users` 和 `user_id` scope。任何查询都不能只靠前端传入的 id 过滤，必须从 authenticated user 派生 owner 条件。  
如果选择 Auth.js，`auth_accounts`、`auth_sessions`、verification token 等表可以按 Auth.js adapter schema 落库；如果选择 Clerk 或 Supabase Auth，本地数据库仍需要一张映射应用用户的 `users` 表。

如果选择 `v1-env-only`，`model_endpoints` 可以只存平台 endpoint 的非敏感配置，并通过 `api_key_env_name` 指向 Vercel env。  
如果选择 `v1-byok`，`api_key_ciphertext`、`api_key_iv`、`api_key_tag` 必须启用，并配套 key rotation 预案。

## 7. API 草案

Nuxt server API 路由:

```txt
GET    /api/auth/session
POST   /api/auth/logout
GET    /api/me
POST   /api/chat
GET    /api/sessions
POST   /api/sessions
GET    /api/sessions/:id
PATCH  /api/sessions/:id
DELETE /api/sessions/:id
GET    /api/sessions/:id/messages
POST   /api/documents
GET    /api/documents
PATCH  /api/sessions/:id/documents/:documentId
GET    /api/model-endpoints
POST   /api/model-endpoints
PATCH  /api/model-endpoints/:id
DELETE /api/model-endpoints/:id
POST   /api/model-endpoints/:id/test
```

认证相关路由由所选 auth provider 暴露，以上 auth 路由是应用侧最低抽象。业务 API 的默认策略是 authenticated-only，只有 `/login`、auth callback、session 查询和 logout 可以匿名访问。

`POST /api/chat` 请求:

```json
{
  "sessionId": "session_123",
  "message": {
    "role": "user",
    "content": "请总结我上传的文档"
  },
  "modelEndpointId": "model_123",
  "documentIds": ["doc_123"]
}
```

`POST /api/chat` 响应:

1. 成功: streaming response。
2. 配置错误: JSON error，带 `MODEL_CONFIG_INVALID`。
3. 上下文过长: JSON error 或流前 metadata，带 `CONTEXT_TOO_LARGE`。
4. Provider 错误: JSON error，保留可展示摘要，隐藏密钥和完整上游响应。

## 8. 安全与隐私

必须优先处理:

1. API key 只在服务端使用。
2. 自定义 endpoint 做 SSRF 防护。
3. 上传文件大小、类型、内容校验。
4. Markdown 渲染 sanitization，避免 XSS。
5. 请求体 schema 校验，建议用 Zod。
6. 所有会话、消息、文档、模型配置都按 user scope 查询。
7. 日志不能记录 API key、Authorization header、完整用户私密文档。
8. 加入基础 rate limit，防止模型账单失控。
9. 在 UI 中显示当前模型和可能产生费用的操作。
10. V1 必须实现认证中间件，所有 server API 默认拒绝匿名请求，登录页、session 查询、logout 和 OAuth callback 除外。
11. 用户自带 key 的创建、替换、删除、测试需要写入 `audit_events`，但 audit metadata 不能包含明文 key。

身份认证方案:

1. 第一版推荐 Auth.js + OAuth provider，适合 Nuxt/Nitro 服务端鉴权，也避免第一版自建密码体系。
2. 如果需要更快落地托管登录，可以选择 Clerk 或 Supabase Auth，但要接受外部服务依赖。
3. 无论选择哪种认证，数据库内的 `user_id` 必须来自服务端 session，不接受客户端传入。
4. Vercel Deployment Protection 只能保护部署入口，不能替代应用内多用户账号体系。

## 9. Prompt 与上下文策略

服务端构造 messages:

1. system/developer message: 产品默认行为、回答语言、引用文档规则。
2. conversation history: 最近 N 轮消息。
3. document context: 已启用 Markdown 文档的节选。
4. current user message。

上下文预算:

```txt
model_context_budget
  - reserved_for_output
  - reserved_for_system
  - recent_messages
  - document_context
```

第一版不要把所有文档无条件塞进 prompt。UI 需要明确显示“本次将使用哪些文档”。

## 10. UI 状态

必须设计的状态:

1. 未登录。
2. 登录中。
3. 认证回调处理中。
4. 登录失败。
5. 空会话。
6. 会话加载中。
7. 消息 streaming。
8. 用户中止生成。
9. Provider 配置缺失。
10. API key 错误。
11. Markdown 上传中、上传成功、上传失败。
12. 文档过大。
13. 当前会话没有可用模型。
14. 小屏幕侧边栏折叠和右侧检查器抽屉。

## 11. 目录结构建议

```txt
app.vue
nuxt.config.ts
pages/
  index.vue
  login.vue
  auth/callback.vue
  chat/[sessionId].vue
  settings/models.vue
  settings/profile.vue
layouts/
  default.vue
components/
  app/
  auth/
  chat/
  sidebar/
  settings/
  documents/
composables/
  useAuthUser.ts
  useChatSession.ts
  useConversationStream.ts
  useModelEndpoints.ts
  useDocuments.ts
i18n/
  zh-CN.ts
  en-US.ts
server/
  api/
    auth/
    me.get.ts
    chat.post.ts
    sessions/
    documents/
    model-endpoints/
  services/
    ai/
    auth/
    crypto/
    db/
    documents/
    security/
  utils/
shared/
  schemas/
  types/
```

## 12. 分阶段实现计划

### Phase 1: 工程骨架、登录基线与 UI Shell

1. 初始化 Nuxt + TypeScript。
2. 选择并接入 V1 auth provider，默认推荐 Auth.js + OAuth provider。
3. 实现 `/login`、auth callback、logout、session 查询和 protected route guard。
4. 接入 Tailwind 和基础 design tokens。
5. 建立 i18n dictionary，默认跟随浏览器语言。
6. 完成高度贴近 Codex 桌面端的 AppShell、Sidebar、ChatWorkspace、Composer 静态结构。
7. 登录后使用 mock data 完成会话切换；未登录只能进入登录流程。

验收:

1. Vercel 可部署。
2. 首页 SSR 正常。
3. 桌面和移动布局不重叠。
4. 深色工作台视觉接近 Codex 桌面端。
5. UI 文案能在中文/英文之间切换。
6. 未登录访问 `/`、`/chat/:sessionId`、`/settings/*` 会进入登录流程。
7. 登录后能进入 mock 工作台，退出后不能继续访问业务界面。

### Phase 2: 用户映射、会话和消息持久化

1. 接入 Postgres provider。
2. 建表和 migration，包括应用 `users` 表和 auth adapter/mapping 表。
3. 登录成功时创建或更新应用用户映射。
4. 实现 user scope 下的 session/message CRUD。
5. 侧边栏连接真实数据。
6. 所有业务 API 增加 authenticated-only guard，并从服务端 session 派生 `user_id`。

验收:

1. 不同登录用户只能看到自己的会话和消息。
2. 刷新页面后会话和消息保留。
3. 删除、重命名、切换会话稳定。
4. 匿名请求访问业务 API 会被拒绝。
5. 客户端伪造 `user_id` 不会改变服务端 owner 判断。

### Phase 3: 平台默认模型的流式聊天

1. 实现 `/api/chat`。
2. 接入 AI SDK 和 Vercel env 中的默认 OpenAI-compatible endpoint。
3. 前端支持 streaming、stop、retry。
4. 记录 token usage 和错误摘要。
5. 加入 Hobby 环境下的应用级超时、登录用户级用量计数和用户中止处理。

验收:

1. 能连续对话。
2. 流式输出可中止。
3. Provider 错误不会泄露密钥。
4. 默认模型 key 不会出现在浏览器请求、响应、日志中。
5. 未登录无法调用 `/api/chat`。

### Phase 4: 当前会话 Markdown 上传

1. 实现 Markdown 上传 API。
2. 接入 Vercel Blob private storage。
3. 文档绑定到会话。
4. 聊天请求可选择文档上下文。
5. 明确不做跨会话文档库和向量检索。

验收:

1. 上传 `.md` 后可在当前会话使用。
2. 超大文件、非 Markdown 文件被拒绝。
3. 文档内容进入 prompt 前可见、可关闭。
4. 用户不能访问其他用户或其他会话未绑定的文档。
5. 未登录无法上传或读取文档。

### Phase 5A: `v1-env-only` 模型配置路径

适用条件: 第一版不允许用户保存自己的 API key，只使用平台管理员在 Vercel env 中配置的默认 endpoint。

1. 模型设置页展示平台 endpoint、默认模型和可调非敏感参数。
2. 允许用户按会话选择平台 endpoint。
3. 支持连接测试，但测试使用平台 env key。
4. 隐藏 API key 输入能力。

验收:

1. 用户能选择平台提供的 OpenAI-compatible endpoint。
2. 会话能选择不同模型或参数。
3. 错误配置有明确提示。
4. 无用户 API key 存储风险。
5. 未登录无法读取或修改模型设置。

### Phase 5B: `v1-byok` 用户自带模型端点路径

适用条件: 第一版需要用户自己配置 URL 和 API key。

1. 完成模型设置页。
2. 支持用户新增 OpenAI-compatible endpoint。
3. 使用 Vercel env 中的 `APP_SECRET_KEY` 加密用户 API key 后写入数据库。
4. 实现连接测试。
5. 实现 endpoint SSRF guard、DNS/IP 校验和 header whitelist。
6. 支持替换、禁用、删除用户 key。
7. 写入 endpoint 创建、测试、删除 audit event。

验收:

1. 用户能添加自己的 endpoint 和 API key。
2. 数据库中不出现明文 API key。
3. 会话能选择用户自己的模型。
4. 其他用户无法读取、测试或使用该 endpoint。
5. 错误配置有明确提示，且不泄露上游 Authorization header。
6. 未登录无法创建、测试、更新或删除 endpoint。

### Phase 6: 产品硬化

1. Rate limit。
2. 日志和基础观测。
3. 移动端适配。
4. 可访问性和键盘操作。
5. Hobby 成本保护和请求预算。
6. 错误码、本地化文案和恢复动作完善。

## 12.1 版本路线图

### V1

1. 强制登录和多用户账号体系。
2. 当前会话聊天。
3. 当前会话 Markdown 上传。
4. OpenAI-compatible provider。
5. 高度复刻 Codex 桌面端布局和视觉。
6. `v1-env-only` 或 `v1-byok` 二选一。

### V2

1. 如果 V1 选择 `v1-env-only`，V2 引入用户自带 API key 的数据库加密存储。
2. 模型配置体验增强: key rotation、连接健康状态、默认参数模板。
3. 更完整的审计、限流和账单保护。

### V3

1. 跨会话文档库。
2. Markdown chunking。
3. Embeddings 和语义检索。
4. 多文档引用、来源定位和上下文管理。

## 13. 已确认结论与剩余决策

已确认:

1. 第一版必须支持登录；未登录用户不能进入业务工作台或调用业务 API。
2. 第一版部署目标是 Vercel Hobby。
3. UI 高度复刻 Codex 桌面端布局和视觉。
4. UI 文案默认跟随浏览器语言。
5. 第一版 Markdown 只绑定当前会话。
6. 第一版不做 RAG/向量检索。
7. 第三版做跨会话文档库和 RAG。
8. 模型 provider 第一优先级是 OpenAI-compatible。

剩余关键决策:

1. 第一版是否必须支持用户自带 URL 和 API key。
2. 如果必须支持，选择 `v1-byok`，第一版加入数据库加密密钥存储。
3. 如果可以暂缓，选择 `v1-env-only`，第一版只使用 Vercel env 中的平台默认 endpoint，第二版再做用户自带 key。

API key 存储结论:

1. Vercel env 可行: 仅适用于平台管理员配置的默认模型服务。
2. Vercel env 不可行: 不适用于每个用户在应用内动态保存自己的 URL/API key。
3. 用户自带 key 的正确路径: 服务端接收明文一次，用 Vercel env 中的 master secret 加密，密文存数据库，之后仅在服务端发起模型调用时临时解密。

## 14. 官方参考

1. Nuxt Vercel 部署: https://nuxt.com/deploy/vercel
2. Vercel Nuxt 框架文档: https://vercel.com/docs/frameworks/full-stack/nuxt
3. Vercel Functions limits: https://vercel.com/docs/functions/limitations
4. Vercel Storage overview: https://vercel.com/docs/storage
5. Vercel Blob: https://vercel.com/docs/vercel-blob
6. Postgres on Vercel: https://vercel.com/docs/postgres
7. AI SDK `useChat`: https://ai-sdk.dev/docs/reference/ai-sdk-ui/use-chat
8. AI SDK OpenAI provider: https://ai-sdk.dev/providers/ai-sdk-providers/openai
9. Vercel Environment Variables: https://vercel.com/docs/environment-variables
10. Vercel Sensitive Environment Variables: https://vercel.com/docs/environment-variables/sensitive-environment-variables
