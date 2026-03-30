# DeerFlow 整体架构

## 小明第一次使用 DeerFlow

### 场景1：打开网页

**小明的操作**：在浏览器输入 `localhost:2026`

**发生了什么？**

```
小明输入 localhost:2026
    ↓
Nginx（门卫）收到请求
    ↓
Nginx 判断：这是访问网页，不是 API 请求
    ↓
转发给 Frontend（端口3000）
    ↓
Frontend 返回一个漂亮的聊天界面
```

**技术解释**：

想象 Nginx 是一个大楼的门卫。它站在 2026 号门口，所有访客都要先经过它。门卫有一张"路由表"：

| 访客请求 | 送到哪里 |
|---------|---------|
| 访问网页（如 `/chat`） | Frontend（3000号房间） |
| 调用 AI 服务（`/api/langgraph/*`） | LangGraph Server（2024号房间） |
| 其他 API 请求（`/api/*`） | Gateway API（8001号房间） |

所以小明打开网页时，Nginx 一看"哦，是来看网页的"，就把请求转给了 Frontend。

---

### 场景2：发送消息

**小明的操作**：在聊天框输入"帮我研究AI趋势"，点击发送

**发生了什么？**

```
小明输入"帮我研究AI趋势"
    ↓
Frontend 把消息打包成请求
    ↓
请求发送到 /api/langgraph（Nginx 2026）
    ↓
Nginx 识别出这是 AI 请求，转发给 LangGraph Server（2024）
    ↓
LangGraph Server 启动 Lead Agent（总指挥）
    ↓
Lead Agent 开始思考：这个任务需要做什么？
    ↓
分解任务 → 调用工具 → 可能委托子代理 → 收集结果
    ↓
通过 SSE 流式返回结果给 Frontend
    ↓
Frontend 实时显示 AI 的回复
```

**技术解释**：

1. **Frontend 发送请求**：Next.js 应用通过 HTTP 请求把用户消息发出去

2. **Nginx 路由**：这次请求路径是 `/api/langgraph/*`，Nginx 知道这是要找 AI 大脑，就转发给 LangGraph Server

3. **LangGraph Server 处理**：
   - 创建/恢复一个 Thread（对话线程），保持上下文记忆
   - 调用 `make_lead_agent()` 创建一个 Lead Agent 实例
   - Lead Agent 就像一位"项目总指挥"，负责协调整个任务

4. **Middleware 链处理**：请求会经过一系列"中间件"处理：
   - 创建沙箱环境（隔离的工作空间）
   - 处理用户上传的文件
   - 加载相关技能（Skills）
   - 检查是否需要总结历史消息

5. **AI 推理与工具调用**：
   - Lead Agent 分析任务："研究 AI 趋势"需要搜索网络、整理信息、生成报告
   - 调用搜索工具（如 Tavily）获取最新信息
   - 可能创建子代理并行处理多个子任务
   - 在沙箱中执行代码、生成文件

6. **流式返回**：结果不是等全部完成才返回，而是通过 SSE（Server-Sent Events）实时推送给前端，小明能看到 AI 正在一步步思考

---

## 更多调用路径例子

### 例子A：文件上传流程

**小明的操作**：在聊天界面点击"上传PDF"按钮，选择一份研究报告

**发生了什么？**

```
小明点击"上传PDF"
    ↓
Frontend 调用 POST /api/threads/{id}/uploads
    ↓
Nginx 转发到 Gateway（8001）
    ↓
Gateway 做什么？
    - 接收文件（multipart/form-data）
    - 保存到 .deer-flow/threads/{id}/uploads/
    - 如果是PDF/Word/PPT，调用 markitdown 转文本
    - 返回文件元数据（文件名、大小、转换后的文本路径）
    ↓
下次对话时，UploadsMiddleware 自动注入文件信息到提示
    ↓
AI 能看到文件内容并基于它回答
```

**技术细节**：
- 上传端点：`POST /api/threads/{thread_id}/uploads`
- 支持格式：PDF、Word（.doc/.docx）、PPT（.ppt/.pptx）、Excel、图片、文本文件
- 大文件处理：通过 markitdown 转换为 Markdown 文本，方便 AI 处理
- 文件存储：每个线程有独立的 uploads 目录，隔离安全

---

### 例子B：获取模型列表

**小明的操作**：打开设置面板，想切换使用的 AI 模型

**发生了什么？**

```
小明打开设置面板，看到可用模型列表
    ↓
Frontend 调用 GET /api/models
    ↓
Nginx 转发到 Gateway（8001）
    ↓
Gateway 读取 config.yaml 中的 models 配置
    ↓
返回模型列表（名称、提供商、是否支持思考模式等）
    ↓
Frontend 展示模型选项，小明选择"DeepSeek-V3"
```

**返回示例**：
```json
{
  "models": [
    {
      "name": "gpt-4o",
      "display_name": "GPT-4o",
      "supports_thinking": false,
      "supports_vision": true
    },
    {
      "name": "deepseek-v3",
      "display_name": "DeepSeek V3",
      "supports_thinking": true,
      "supports_vision": false
    }
  ]
}
```

---

### 例子C：长对话的上下文摘要

**场景**：小明和 AI 已经对话了50轮，讨论一个很复杂的项目

**发生了什么？**

```
对话进行了50轮，token数超过8000
    ↓
SummarizationMiddleware 检测到 token_threshold 触发
    ↓
调用轻量级模型（如 gpt-4o-mini）总结前30轮
    ↓
将总结插入历史消息，保留最近20轮完整对话
    ↓
继续对话，模型看到的是"总结+最近20轮"
    ↓
AI 仍然理解完整上下文，但 token 数大幅降低
```

**配置控制**：
```yaml
summarization:
  enabled: true          # 是否启用自动摘要
  token_threshold: 8000  # 触发阈值
  summary_model: "gpt-4o-mini"  # 用于摘要的轻量级模型
  keep_messages: 20      # 保留最近多少条完整消息
```

**生活例子**：
> 就像一位秘书帮你整理会议记录。前30轮讨论变成了"会议纪要"，后面20轮是详细记录。AI 既能把握全局，又知道最近聊了什么。

---

## 核心组件详解

### 1. Nginx（门卫）

**是什么**：Nginx 是一个高性能的反向代理服务器，在 DeerFlow 中充当"统一入口"

**作用**：
- 对外暴露唯一端口（2026），简化访问
- 根据请求路径，把流量分发到不同的后端服务
- 提供负载均衡、静态资源缓存等功能

**生活例子**：
> 想象你去一个大型商场，Nginx 就是商场的总服务台。顾客（浏览器请求）都先到服务台，服务员根据你的需求（看网页/调用API），指引你去不同的楼层（Frontend/LangGraph/Gateway）。

**配置示例**（简化版）：
```nginx
# 访问网页 → Frontend
location / {
    proxy_pass http://localhost:3000;
}

# AI 请求 → LangGraph Server
location /api/langgraph/ {
    proxy_pass http://localhost:2024;
}

# 其他 API → Gateway
location /api/ {
    proxy_pass http://localhost:8001;
}
```

**Nginx 配置细节详解**：

```nginx
server {
    listen 2026;
    server_name localhost;

    # 1. 访问网页 → Frontend (Next.js)
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # 2. AI 请求 → LangGraph Server
    # 匹配规则：以 /api/langgraph/ 开头的请求
    location /api/langgraph/ {
        proxy_pass http://localhost:2024;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        # SSE 支持：保持长连接
        proxy_buffering off;
        proxy_read_timeout 86400;
    }

    # 3. 其他 API → Gateway API
    # 匹配规则：以 /api/ 开头，但不匹配上面的 /api/langgraph/
    location /api/ {
        proxy_pass http://localhost:8001;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
    }
}
```

**location 匹配规则说明**：

1. **精确匹配**：`location = /` 只匹配根路径
2. **前缀匹配**：`location /api/` 匹配所有以 `/api/` 开头的请求
3. **正则匹配**：`location ~ \.php$` 使用正则表达式
4. **优先级**：精确匹配 > 前缀匹配（长路径优先）> 正则匹配

**proxy_pass 转发逻辑**：

```
请求: /api/langgraph/threads/123/runs
      ↓
匹配 location /api/langgraph/
      ↓
转发到: http://localhost:2024/threads/123/runs
        （去掉匹配的 /api/langgraph/ 前缀）

请求: /api/models
      ↓
匹配 location /api/
      ↓
转发到: http://localhost:8001/models
```

---

### 2. Frontend（前台）

**是什么**：基于 Next.js + React 的现代化 Web 界面

**作用**：
- 提供用户交互界面（聊天窗口、文件上传、设置面板）
- 实时显示 AI 的流式回复（打字机效果）
- 管理用户会话、显示历史对话

**生活例子**：
> Frontend 就像是餐厅的前台服务员。它负责接待顾客（显示界面）、记录点单（发送消息）、把菜品端上桌（展示 AI 回复）。顾客不需要知道厨房（后端）怎么运作，只需要和服务员打交道。

**技术栈**：
- **Next.js**：React 框架，支持服务端渲染
- **React**：构建用户界面的 JavaScript 库
- **SSE（Server-Sent Events）**：接收服务器实时推送的消息

---

### 3. LangGraph Server（大脑）

**是什么**：DeerFlow 的核心 AI 引擎，基于 LangGraph 框架构建

**作用**：
- 运行 Lead Agent（总指挥），协调任务执行
- 管理对话状态（Thread State），保持上下文记忆
- 执行工具调用（搜索、代码执行、文件操作）
- 支持子代理委托，实现复杂任务分解

**生活例子**：
> LangGraph Server 就像是一位"项目总监"。当收到"研究 AI 趋势"的任务时，总监不会自己干所有活，而是：
> 1. 分析任务需要做什么
> 2. 分配给不同的专员（工具/子代理）：A 去搜索资料、B 整理数据、C 写报告
> 3. 汇总各专员的结果，形成最终交付物

**核心概念**：

| 概念 | 解释 | 例子 |
|-----|------|------|
| **Lead Agent** | 总指挥，负责任务分解和协调 | 项目经理 |
| **Thread** | 对话线程，保存一次完整对话的上下文 | 一个项目档案袋 |
| **ThreadState** | 线程状态，包含消息历史、沙箱信息、产物等 | 档案袋里的所有资料 |
| **Middleware** | 中间件，在请求处理前后执行特定逻辑 | 流水线质检员 |
| **Tool** | 工具，AI 可以调用的功能 | 专员的技能 |
| **Sub-agent** | 子代理，专门处理特定任务的 AI | 专业专员 |

**ThreadState 详细字段**：

| 字段 | 类型 | 作用 | 示例值 |
|------|------|------|--------|
| messages | list | 消息历史（来自 AgentState） | [{"role": "user", "content": "..."}] |
| sandbox | dict | 沙箱信息 | {"container_id": "abc123", "sandbox_id": "sandbox-001"} |
| thread_data | dict | 路径信息 | {"workspace": "/path/to/ws", "uploads": "/path/to/uploads"} |
| artifacts | list | 产物文件路径 | ["/outputs/report.md", "/outputs/chart.png"] |
| title | str | 自动生成的对话标题 | "AI趋势研究报告" |
| todos | list | 计划模式下的待办事项 | [{"id": 1, "content": "搜索资料", "done": false}] |
| uploaded_files | list | 用户上传文件的元数据 | [{"name": "doc.pdf", "path": "/uploads/doc.pdf"}] |
| viewed_images | dict | 视觉模型的图像缓存 | {"img1": {"base64": "...", "path": "/uploads/img.jpg"}} |

**Middleware 链（处理流程）**：

当消息到达 LangGraph Server 后，会依次经过这些处理：

1. **ThreadDataMiddleware**：创建线程目录（workspace、uploads、outputs）
2. **UploadsMiddleware**：处理用户上传的文件
3. **SandboxMiddleware**：获取或创建沙箱环境
4. **SummarizationMiddleware**：如果消息太长，自动总结历史
5. **TodoListMiddleware**（计划模式）：跟踪任务进度
6. **TitleMiddleware**：自动生成对话标题
7. **ViewImageMiddleware**：处理图片输入（视觉模型）
8. **ClarificationMiddleware**：处理需要用户澄清的情况

**Middleware 执行顺序图**：

```
请求进入
  ↓
ThreadDataMiddleware (before_agent)
  ↓
UploadsMiddleware (before_agent)
  ↓
SandboxMiddleware (before_agent)
  ↓
DanglingToolCallMiddleware (wrap_model_call)
  ↓
[可能触发 SummarizationMiddleware]
  ↓
模型调用
  ↓
[可能触发 TodoMiddleware, TokenUsageMiddleware, TitleMiddleware]
  ↓
工具调用
  ↓
ToolErrorHandlingMiddleware (wrap_tool_call)
  ↓
ClarificationMiddleware (wrap_tool_call)
  ↓
MemoryMiddleware (after_agent)
```

**关键 Middleware 说明**：

| Middleware | 执行时机 | 作用 |
|------------|----------|------|
| ThreadDataMiddleware | before_agent | 延迟创建 per-thread 目录结构 |
| UploadsMiddleware | before_agent | 读取文件元数据并注入到提示 |
| SandboxMiddleware | before_agent | 确保沙箱容器运行并注入 sandbox_id |
| DanglingToolCallMiddleware | wrap_model_call | 扫描消息历史查找缺少响应的工具调用 |
| SummarizationMiddleware | before_agent | token 超过阈值时总结历史消息 |
| TodoMiddleware | before_model | 计划模式下跟踪任务进度 |
| TokenUsageMiddleware | after_model | 记录 LLM token 消耗 |
| TitleMiddleware | after_model | 首次交换后自动生成标题 |
| MemoryMiddleware | after_agent | 排队对话进行异步记忆更新 |
| ToolErrorHandlingMiddleware | wrap_tool_call | 捕获异常并转换为错误消息 |
| ClarificationMiddleware | wrap_tool_call | 拦截 ask_clarification 工具调用 |

---

### 4. Gateway API（后勤部）

**是什么**：基于 FastAPI 的 REST API 服务，处理非 AI 核心功能

**作用**：
- 模型管理（列出可用模型、获取模型详情）
- MCP 服务器配置（管理外部工具源）
- 技能管理（启用/禁用技能）
- 文件上传/下载
- 线程数据清理
- 产物（Artifacts）管理
- 生成后续建议

**生活例子**：
> Gateway API 就像是公司的"后勤部门"。它不直接参与核心业务（AI 推理），但提供各种支持服务：管理员工名单（模型列表）、维护工具供应商关系（MCP 配置）、管理档案室（文件/线程清理）、整理项目交付物（Artifacts）。

**API 端点示例**：

| 端点 | 功能 |
|-----|------|
| `GET /api/models` | 获取所有可用模型列表 |
| `GET /api/mcp` | 获取 MCP 服务器配置 |
| `POST /api/skills/{name}/enable` | 启用某个技能 |
| `POST /api/threads/{id}/uploads` | 上传文件到指定线程 |
| `GET /api/threads/{id}/artifacts` | 获取线程的产物列表 |

---

## 配置文件

### config.yaml

**作用**：DeerFlow 的主配置文件，定义模型、工具、沙箱等核心设置

**内容示例**：
```yaml
# ==================== 模型配置 ====================
# 可以配置多个模型，支持不同提供商
models:
  # OpenAI 官方模型
  - name: "gpt-4o"
    provider: "openai"
    api_key: "$OPENAI_API_KEY"  # 从环境变量读取
    base_url: "https://api.openai.com/v1"
    supports_vision: true       # 支持图像识别
    supports_thinking: false
  
  # OpenAI 轻量级模型（用于摘要等场景）
  - name: "gpt-4o-mini"
    provider: "openai"
    api_key: "$OPENAI_API_KEY"
    base_url: "https://api.openai.com/v1"
    supports_vision: true
    supports_thinking: false
  
  # DeepSeek 模型（支持思考模式）
  - name: "deepseek-chat"
    provider: "openai_compatible"
    api_key: "$DEEPSEEK_API_KEY"
    base_url: "https://api.deepseek.com/v1"
    supports_vision: false
    supports_thinking: true     # 支持深度思考
  
  # 本地 Ollama 模型
  - name: "llama3.1"
    provider: "ollama"
    base_url: "http://localhost:11434"
    supports_vision: false
    supports_thinking: false

# ==================== 工具配置 ====================
# 控制哪些工具可用

tools:
  # 网络搜索工具
  - name: "tavily_search"
    enabled: true
    config:
      api_key: "$TAVILY_API_KEY"
      max_results: 5
  
  # 命令行执行工具
  - name: "bash"
    enabled: true
    config:
      allowed_commands: ["ls", "cat", "grep", "python", "pip"]
  
  # 代码执行工具（根据现有资料无法确定详细配置）
  - name: "python"
    enabled: true

# ==================== 沙箱配置 ====================
# 三种模式：local / docker / provisioner

sandbox:
  # 模式1：本地沙箱（开发环境推荐）
  type: "local"
  workspace_dir: "./.deer-flow/threads"
  
  # 模式2：Docker 沙箱（更好的隔离性）
  # type: "docker"
  # image: "deerflow-sandbox:latest"
  # memory_limit: "512m"
  # cpu_limit: "1.0"
  
  # 模式3：Provisioner 模式（生产/K8s环境）
  # type: "provisioner"
  # provisioner_url: "http://localhost:8002"

# ==================== 摘要配置 ====================
# 控制长对话的自动摘要行为

summarization:
  enabled: true              # 是否启用
  token_threshold: 8000      # 触发阈值（token数）
  summary_model: "gpt-4o-mini"  # 用于摘要的轻量级模型
  keep_messages: 20          # 保留最近多少条完整消息
  min_messages_to_summarize: 10  # 至少多少条消息才触发摘要

# ==================== 其他配置 ====================

# 日志配置
logging:
  level: "INFO"              # DEBUG / INFO / WARNING / ERROR
  format: "json"             # json / text

# 内存配置（根据现有资料无法确定详细配置）
memory:
  enabled: true
  max_memories: 100
```

**通俗解释**：
> `config.yaml` 就像是 DeerFlow 的"员工手册和供应商名录"。它告诉系统：
> - 可以用哪些 AI 模型（员工名单）
> - 每个模型怎么联系（API 密钥和地址）
> - 有哪些工具可用（供应商名录）
> - 沙箱怎么配置（工作环境设置）

**沙箱三种模式对比**：

| 模式 | 适用场景 | 隔离性 | 启动速度 | 资源占用 |
|------|----------|--------|----------|----------|
| **local** | 本地开发 | 低（进程级）| 快 | 低 |
| **docker** | 测试/小规模部署 | 高（容器级）| 中等 | 中等 |
| **provisioner** | 生产/K8s 集群 | 高（Pod级）| 按需 | 弹性 |

**环境变量替换规则**：
- 以 `$` 开头的值会从环境变量读取
- 例如：`$OPENAI_API_KEY` 会读取系统环境变量 `OPENAI_API_KEY`
- 安全性：避免在配置文件中硬编码敏感信息

---

### extensions_config.json

**作用**：管理扩展功能，包括 MCP 服务器和技能状态

**内容示例**：
```json
{
  "mcpServers": {
    "fetch": {
      "command": "uvx",
      "args": ["mcp-server-fetch"],
      "enabled": true
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"],
      "enabled": false
    }
  },
  "skills": {
    "deep_research": {
      "enabled": true,
      "auto_load": true
    },
    "slide_deck": {
      "enabled": false,
      "auto_load": false
    }
  }
}
```

**通俗解释**：
> `extensions_config.json` 就像是"插件和技能开关面板"。它控制：
> - **MCP 服务器**：外部工具源，比如网页抓取、文件系统访问等
> - **技能（Skills）**：AI 的专业能力包，比如深度研究、制作 PPT、生成网页等

**MCP 是什么**：
> MCP（Model Context Protocol）是一种标准协议，让 AI 可以连接各种外部工具。就像 USB 接口让电脑可以连接各种外设一样，MCP 让 AI 可以"即插即用"各种工具服务。

**Skill 是什么**：
> Skill 是一个 Markdown 文件，定义了完成某类任务的最佳实践。比如"深度研究"技能会告诉 AI：如何搜索、如何整理信息、如何引用来源、报告格式是什么。

---

## 总结

**一句话总结**：

> DeerFlow 是一个基于 LangGraph 的 AI Super Agent 系统，通过 Nginx 统一入口接收用户请求，Frontend 提供交互界面，LangGraph Server 作为 AI 大脑协调任务执行，Gateway API 提供辅助服务，配合配置文件实现模型、工具、技能的灵活管理。

**架构核心特点**：

1. **分层架构**：Nginx → Frontend/Gateway/LangGraph，职责清晰
2. **Agent 编排**：Lead Agent + 子代理，实现复杂任务分解
3. **沙箱执行**：隔离环境安全运行代码
4. **流式响应**：SSE 实时推送，用户体验流畅
5. **配置驱动**：YAML/JSON 配置，灵活可扩展
