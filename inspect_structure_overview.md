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
# 模型配置
models:
  - name: "gpt-4o"
    provider: "openai"
    api_key: "$OPENAI_API_KEY"  # 从环境变量读取
    base_url: "https://api.openai.com/v1"
  
  - name: "deepseek-chat"
    provider: "openai_compatible"
    api_key: "$DEEPSEEK_API_KEY"
    base_url: "https://api.deepseek.com/v1"

# 工具配置
tools:
  - name: "tavily_search"
    enabled: true
    config:
      api_key: "$TAVILY_API_KEY"
  
  - name: "bash"
    enabled: true

# 沙箱配置
sandbox:
  type: "local"  # 可选: local, docker, provisioner
  workspace_dir: "./workspace"

# 摘要配置（控制何时自动总结历史消息）
summarization:
  enabled: true
  token_threshold: 8000
  summary_model: "gpt-4o-mini"
```

**通俗解释**：
> `config.yaml` 就像是 DeerFlow 的"员工手册和供应商名录"。它告诉系统：
> - 可以用哪些 AI 模型（员工名单）
> - 每个模型怎么联系（API 密钥和地址）
> - 有哪些工具可用（供应商名录）
> - 沙箱怎么配置（工作环境设置）

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
