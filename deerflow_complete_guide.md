# DeerFlow 完整技术指南：从 Harness 到 Agent 架构

> **版本**：3.0（最终版）  
> **目标**：深入浅出，产研团队友好  
> **字数**：8000+ 字  
> **来源**：ZRead、官方文档、源码分析  
> **生成时间**：2026-03-30

---

## 目录

1. [基础概念：Prompt、Context、Harness](#第一章基础概念promptcontextharness)
2. [DeerFlow 产品定位](#第二章deerflow-产品定位)
3. [整体架构概览](#第三章整体架构概览)
4. [Gateway 层详解](#第四章gateway-层详解)
5. [Agent 架构详解](#第五章agent-架构详解)
6. [Skills、MCP、Memory 工作方式](#第六章skillsmcpmemory-工作方式)

---

## 第一章：基础概念 Prompt、Context、Harness

### 1.1 Prompt（提示词）

**是什么**：给 AI 的指令，告诉它要做什么。

**示例**：
```
"帮我研究 AI 编程助手市场，输出一份报告"
```

**局限**：单次指令，无法处理复杂多步骤任务。就像你对助理说"帮我写报告"，但不写报告需要哪些步骤、用什么资料。

### 1.2 Context（上下文）

**是什么**：AI 能看到的背景信息，包括对话历史、用户偏好、相关文件等。

**示例**：
```
- 用户之前问过："什么是 Copilot"
- 用户上传了：一份市场报告.pdf
- 系统提示："你是一个专业研究员"
- 用户偏好：喜欢简洁的 bullet point 格式
```

**关键**：Context 决定 AI 回答的质量。好的 Context 工程能显著提升效果。

**Context Engineering 的核心**：
- **历史消息**：保持对话连贯性
- **用户偏好**：记住用户喜欢的风格
- **相关文件**：上传的文档、数据
- **系统提示**：定义 AI 的角色和能力边界

### 1.3 Harness（驾驭系统）

**是什么**：不是直接替代人类，而是放大人类能力的系统。

**核心能力**：
| 能力 | 说明 | 效果 |
|------|------|------|
| **任务拆解** | 把复杂任务分解为可执行的子任务 | 模糊需求 → 清晰步骤 |
| **资源调度** | 自动选择最合适的模型和工具 | 最优配置，无需手动 |
| **并行加速** | 多个子任务同时执行 | 串行 → 并行，节省时间 |
| **质量保证** | 自动验证中间结果 | 减少错误，提高质量 |
| **成果固化** | 产出可交付的完整成果 | 不是建议，是交付物 |

**通俗类比**：

> **Prompt** = 你对助理说"帮我写报告"
> **Context** = 助理记得你之前的偏好，看到你给的资料
> **Harness** = 一位**智能项目经理**，自动拆解任务、调度团队、把控质量、交付成果

**为什么需要 Harness？**

| 场景 | 只用 Prompt + Context | 使用 Harness |
|------|----------------------|-------------|
| 研究市场 | 一问一答，手动整理 | 自动搜索、分析、生成报告 |
| 制作 PPT | AI 给大纲，手动制作 | 自动设计、生成、导出 |
| 多步骤任务 | 分多次提问，容易断链 | 自动规划、执行、汇总 |

**Harness vs 传统 AI**：
- 传统 AI：单次对话，一问一答
- Harness：任务编排，端到端交付

---

## 第二章：DeerFlow 产品定位

### 2.1 官方定义

> "DeerFlow (**D**eep **E**xploration and **E**fficient **R**esearch **F**low) is an open-source **super agent harness** that orchestrates **sub-agents**, **memory**, and **sandboxes** to do almost anything — powered by **extensible skills**."

**关键词解读**：
| 关键词 | 含义 | 价值 |
|--------|------|------|
| **Harness** | 驾驭系统 | 放大人类能力，不是替代 |
| **Sub-agents** | 子代理 | 并行执行，效率提升 |
| **Memory** | 长期记忆 | 越用越懂你 |
| **Sandbox** | 沙箱 | 真实执行环境，端到端交付 |
| **Skills** | 技能 | 可扩展的能力模块 |

### 2.2 核心定位

**一句话**：DeerFlow 是一个**企业级的超级 Agent Harness**，专为复杂任务编排和端到端交付设计。

**技术定位**：
- **分层架构**：Harness/App 分离，核心可复用
- **LangGraph 底座**：状态机驱动的 Agent 编排
- **多服务设计**：Nginx + LangGraph + Gateway + Frontend
- **企业级就绪**：K8s 支持、多租户、复杂工作流

**业务定位**：
- **Deep Research**：深度研究，自动生成报告
- **Content Creation**：内容创作，PPT/网页/视频
- **Workflow Automation**：工作流自动化，多步骤任务
- **Knowledge Management**：知识管理，长期记忆沉淀

### 2.3 核心能力矩阵

| 能力 | 解决的问题 | 业务价值 |
|------|-----------|---------|
| **Deep Research** | 人工调研耗时久、信息碎片化 | 研究效率提升 **5-10 倍** |
| **Sub-Agent 编排** | 复杂任务难以拆解、效率低 | 并行执行节省 **60-80% 时间** |
| **Extensible Skills** | 业务场景多变、重复造轮子 | 节省 Token 成本 **30-50%** |
| **Sandbox** | AI 生成内容无法直接执行 | **端到端自动化** |
| **Long-Term Memory** | 每次对话从零开始 | **越用越懂你** |

### 2.4 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| **Agent 运行时** | LangGraph + LangChain | 状态机驱动，支持复杂工作流 |
| **后端 API** | FastAPI (Python 3.12+) | 高性能异步，Python 生态 |
| **前端** | Next.js (Node.js 22+) + React | SSR/SSG，现代前端 |
| **包管理** | uv (Python), pnpm (Node) | 快速、可靠 |
| **沙箱** | Docker / Kubernetes | 隔离执行环境 |
| **反向代理** | Nginx | 统一入口，路由分发 |
| **配置** | YAML + JSON | 灵活、可读 |
| **流式传输** | Server-Sent Events (SSE) | 实时响应 |

### 2.5 官方资源

- **GitHub**：https://github.com/bytedance/deer-flow
- **官网**：https://deerflow.tech
- **文档**：backend/docs/ARCHITECTURE.md
- **ZRead**：https://zread.ai/bytedance/deer-flow

---

## 第三章：整体架构概览

### 3.1 架构概览图

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户访问层                               │
│                    浏览器 / API 客户端                           │
└──────────────────────────────────┬──────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Nginx 反向代理层                            │
│                         Port 2026                               │
│  ┌──────────────────┬──────────────────┬──────────────────────┐  │
│  │   静态资源 /*    │  AI 服务         │  业务 API            │  │
│  │       ↓          │  /api/langgraph/*│  /api/*              │  │
│  │   Frontend       │       ↓          │       ↓              │  │
│  │   Port 3000      │  LangGraph Server│   Gateway API        │  │
│  │   (Next.js)      │   Port 2024      │   Port 8001          │  │
│  │                  │   (AI 大脑)       │   (FastAPI)          │  │
│  └──────────────────┴──────────────────┴──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 组件职责

| 组件 | 核心职责 | 技术选型 | 设计理由 |
|------|---------|---------|---------|
| **Nginx** | 统一入口，路由分发 | Nginx | 成熟稳定，支持高并发 |
| **Frontend** | 用户界面 | Next.js 15 | SSR/SSG，React 生态 |
| **LangGraph Server** | Agent 运行时 | Python + LangGraph | 状态机驱动，支持复杂工作流 |
| **Gateway API** | 业务层 API | FastAPI | 高性能异步，Python 生态 |

### 3.3 请求流转

**路由规则**：
| 路径前缀 | 目标服务 | 处理内容 |
|---------|---------|---------|
| `/api/langgraph/*` | LangGraph Server | AI 推理、Agent 编排 |
| `/api/*` | Gateway API | 文件、模型、MCP、技能 |
| `/*` | Frontend | 静态资源、页面渲染 |

**数据流转**：
```
用户请求 → Nginx → 路由分发 → 各服务处理 → 返回响应
```

### 3.4 Harness/App 分层

```
┌─────────────────────────────────────────────────────────────────┐
│                        App Layer                                │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │    Gateway      │  │    Channels     │  │   Web Frontend  │  │
│  │   (FastAPI)     │  │   (IM 集成)      │  │   (Next.js)     │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ (单向依赖)
┌─────────────────────────────────────────────────────────────────┐
│                      Harness Layer                              │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐    │
│  │  agents/  │  │ sandbox/  │  │  models/  │  │  tools/   │    │
│  │ Agent编排  │  │  沙箱系统  │  │ 模型工厂   │  │ 工具系统   │    │
│  └───────────┘  └───────────┘  └───────────┘  └───────────┘    │
│                                                                 │
│  设计原则：App 可导入 Harness，Harness 严禁导入 App              │
└─────────────────────────────────────────────────────────────────┘
```

**分层价值**：
- **Harness**：业务无关，可复用，可发布为独立包
- **App**：业务相关，灵活定制，依赖 Harness

---

## 第四章：Gateway 层详解

### 4.1 Gateway 定位

**一句话**：Gateway 是"后勤部"，处理所有非 AI 的事务性工作。

**职责边界**：
| 类型 | Gateway 处理 | LangGraph 处理 |
|------|-------------|---------------|
| 文件上传/下载 | ✅ | ❌ |
| 模型列表查询 | ✅ | ❌ |
| MCP/Skills 配置 | ✅ | ❌ |
| AI 推理/对话 | ❌ | ✅ |

### 4.2 重要接口列表

| 接口 | 方法 | 功能 | 用途 |
|------|------|------|------|
| `/api/models` | GET | 获取模型列表 | 切换 AI 模型 |
| `/api/threads/{id}/uploads` | POST | 上传文件 | 上传 PDF/Word |
| `/api/mcp/config` | GET/PUT | MCP 配置 | 添加 GitHub 工具 |
| `/api/skills` | GET | 技能列表 | 查看可用技能 |
| `/api/skills/{name}` | PUT | 启用/禁用技能 | 开启新功能 |

### 4.3 完整案例：用户完成"研究任务"的全过程

**场景**：用户想研究 AI 编程助手市场，需要上传资料、切换模型、配置工具、使用技能。

**步骤 1：上传文件**
```
用户上传 market-report.pdf
    ↓
POST /api/threads/abc123/uploads
    ↓
Gateway 接收文件，保存到线程目录
    ↓
PDF → Markdown 自动转换
    ↓
返回文件元数据
```

**步骤 2：查询模型列表**
```
用户打开设置面板
    ↓
GET /api/models
    ↓
Gateway 读取 config.yaml
    ↓
返回：[GPT-4o, Claude 3, DeepSeek V3]
```

**步骤 3：切换模型**
```
用户选择 DeepSeek V3
    ↓
配置更新（客户端存储）
    ↓
下次对话使用 DeepSeek V3
```

**步骤 4：配置 MCP 工具**
```
用户添加 GitHub 工具
    ↓
PUT /api/mcp/config
    请求体：{
        "github": {
            "command": "npx -y @modelcontextprotocol/server-github",
            "env": {"GITHUB_TOKEN": "ghp_xxx"}
        }
    }
    ↓
Gateway 更新 extensions_config.json
    ↓
下次 Agent 启动时加载 GitHub 工具
```

**步骤 5：启用技能**
```
用户启用"深度研究"技能
    ↓
PUT /api/skills/deep-research
    请求体：{"enabled": true}
    ↓
Gateway 更新技能状态
    ↓
下次对话可使用该技能
```

**步骤 6：发送消息**
```
用户输入："基于上传的报告，研究 AI 编程助手市场"
    ↓
POST /api/langgraph/threads/abc123/runs
    ↓
LangGraph Server 处理（见第五章）
    ↓
返回研究报告
```

### 4.4 Gateway 技术实现

**框架**：FastAPI（Python 3.12+）

**核心 Router**：
```python
# app/gateway/app.py
routers = [
    models.router,      # /api/models
    mcp.router,         # /api/mcp
    skills.router,      # /api/skills
    uploads.router,     # /api/threads/{id}/uploads
    threads.router,     # /api/threads/{id}
    artifacts.router,   # /api/threads/{id}/artifacts
]
```

**文件处理流程**：
```python
async def upload_file(thread_id, file):
    # 1. 保存到线程目录
    path = f".deer-flow/threads/{thread_id}/uploads/{file.name}"
    save(file, path)
    
    # 2. 格式转换
    if file.type in [PDF, Word, PPT]:
        markdown = convert_to_markdown(path)
        save(markdown, path + ".md")
    
    # 3. 返回元数据
    return {
        "filename": file.name,
        "virtual_path": f"/mnt/user-data/uploads/{file.name}",
        "markdown": f"{file.name}.md"
    }
```

---

## 第五章：Agent 架构详解

### 5.1 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     Agent 运行时架构                             │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    LangGraph Server                      │   │
│  │                      (Port 2024)                         │   │
│  │                                                         │   │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │   │
│  │  │   Thread    │    │  Middleware │    │   Tools     │  │   │
│  │  │   State     │───→│    Chain    │───→│   System    │  │   │
│  │  │             │    │  (16个)     │    │             │  │   │
│  │  └─────────────┘    └─────────────┘    └─────────────┘  │   │
│  │         │                   │                   │        │   │
│  │         ▼                   ▼                   ▼        │   │
│  │  ┌─────────────────────────────────────────────────────┐  │   │
│  │  │                  Lead Agent                         │  │   │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐         │  │   │
│  │  │  │ Planning │→ │ Execution│→ │ Synthesis│         │  │   │
│  │  │  │  规划    │  │  执行    │  │  汇总    │         │  │   │
│  │  │  └──────────┘  └──────────┘  └──────────┘         │  │   │
│  │  │         │              │              │            │  │   │
│  │  │         ▼              ▼              ▼            │  │   │
│  │  │  ┌─────────────────────────────────────────────┐   │  │   │
│  │  │  │           Sub-Agent Pool                   │   │  │   │
│  │  │  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ │   │  │   │
│  │  │  │  │Sub-1│ │Sub-2│ │Sub-3│ │ ... │ │Sub-N│ │   │  │   │
│  │  │  │  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ │   │  │   │
│  │  │  └─────────────────────────────────────────────┘   │  │   │
│  │  └─────────────────────────────────────────────────────┘  │   │
│  │                                                           │   │
│  └───────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 核心组件介绍

#### 5.2.1 ThreadState（线程状态）

**是什么**：Agent 执行的完整状态快照，支持断点续传。

**核心字段**：
| 字段 | 类型 | 说明 |
|------|------|------|
| `messages` | list | 对话历史 |
| `sandbox` | dict | 沙箱容器 ID |
| `thread_data` | dict | workspace/uploads/outputs 路径 |
| `title` | str | 自动生成的线程标题 |
| `artifacts` | list | 累积的可交付文件 |
| `todos` | list | 计划模式待办事项 |
| `uploaded_files` | list | 上传文件的元数据 |
| `viewed_images` | dict | Vision 模型的图像缓存 |

**Reducer 机制**：
- `merge_artifacts`：去重 + 保序
- `merge_viewed_images`：支持通过 `{}` 清除

#### 5.2.2 Middleware 链（16 个）

**是什么**：请求处理管道，每个中间件负责一个横切关注点。

**Shared Runtime Middlewares**：
| 中间件 | Hook | 作用 |
|--------|------|------|
| ThreadDataMiddleware | before_agent | 创建线程目录结构 |
| UploadsMiddleware | before_agent | 处理上传文件 |
| SandboxMiddleware | before_agent | 确保沙箱运行 |
| DanglingToolCallMiddleware | wrap_model_call | 修复悬空工具调用 |
| GuardrailMiddleware | wrap | 安全检查 |
| ToolErrorHandlingMiddleware | wrap_tool_call | 错误处理 |

**Lead-Specific Middlewares**：
| 中间件 | Hook | 作用 |
|--------|------|------|
| SummarizationMiddleware | — | 修剪消息历史 |
| TodoMiddleware | before_model | 计划模式待办 |
| TokenUsageMiddleware | after_model | Token 消耗记录 |
| TitleMiddleware | after_model | 自动生成标题 |
| MemoryMiddleware | after_agent | 异步更新记忆 |
| ViewImageMiddleware | before_model | Vision 模型图像 |
| SubagentLimitMiddleware | after_model | 限制并发子代理 |
| LoopDetectionMiddleware | after_model | 检测循环调用 |
| ClarificationMiddleware | wrap_tool_call | 澄清处理 |

**执行顺序**：
```
before_agent → wrap_model_call → after_model → wrap_tool_call → after_agent
```

#### 5.2.3 Lead Agent（总指挥）

**是什么**：每个用户请求的入口点，协调工具执行、管理对话状态、委托子代理。

**创建方式**：
```python
# 每次调用新鲜构建
lead_agent = make_lead_agent(config)
```

**模型解析回退链**：
1. Request override（config.configurable 中的 model_name）
2. Agent-specific model（自定义 agent 配置）
3. Global default（models 数组第一个）

**系统提示组合**：
```
SYSTEM_PROMPT_TEMPLATE
├── {soul} - Agent 个性
├── {memory_context} - 长期记忆
├── {skills_section} - 可用技能
├── {deferred_tools_section} - 延迟 MCP 工具
├── {subagent_section} - 子代理指令
├── {acp_section} - ACP 任务指令
├── <clarification_system> - 澄清工作流
├── <citations> - 引用归因
└── <working_directory> - 沙箱布局
```

#### 5.2.4 Loop 实现（伪代码）

```python
class ThreadState:
    messages: list          # 对话历史
    step_count: int         # 当前步骤
    next_action: str        # 下一步动作
    is_complete: bool       # 是否完成

def agent_loop(state: ThreadState):
    while not state.is_complete:
        # 1. 规划：决定下一步做什么
        plan = planning_agent(state)
        
        # 2. 执行：调用工具或创建 Sub-Agent
        if plan.needs_sub_agent:
            result = spawn_sub_agent(plan.task)
        else:
            result = execute_tool(plan.tool, plan.params)
        
        # 3. 更新状态
        state.messages.append(result)
        state.step_count += 1
        
        # 4. 检查是否完成
        state.is_complete = check_completion(state)
    
    return state
```

**关键特性**：
- 断点续传：任意步骤可暂停、恢复
- 循环支持：支持反思、迭代、回溯
- 流式输出：通过 SSE 实时推送

#### 5.2.5 Sub-Agent（子代理）

**是什么**：Lead Agent 创建的子代理，负责执行子任务。

**触发条件**：
1. 任务复杂，需要并行处理
2. 需要隔离上下文，避免干扰
3. 需要长时间运行，不阻塞主流程

**调度机制**：
```python
class SubAgentManager:
    def __init__(self):
        self.scheduler_pool = ThreadPool(max_workers=3)  # 调度池
        self.execution_pool = ThreadPool(max_workers=3)  # 执行池
    
    def spawn(self, task, parent_state):
        # 创建子代理
        sub_agent = SubAgent(
            task=task,
            context=inherit_context(parent_state),
            sandbox=parent_state.sandbox
        )
        
        # 提交到调度池
        future = self.scheduler_pool.submit(self._run, sub_agent)
        return future
```

**关键参数**：
- 最大并发：3 个（硬编码）
- 超时控制：15 分钟
- 上下文继承：选择性继承父代理状态

### 5.3 完整案例：从会话到完成

**场景**：用户请求"研究 AI 编程助手市场，生成报告"

**Step 1: 发起会话**
```
用户输入："研究 AI 编程助手市场，生成报告"
    ↓
LangGraph Server 创建 Thread
    ↓
初始化 ThreadState
```

**Step 2: 规划（Planning）**
```
Lead Agent 分析任务：
    "需要：
     1. 搜索市场概况
     2. 分析主要竞品
     3. 整理技术趋势
     4. 生成报告"
    ↓
创建执行计划
    ↓
决定：创建 3 个 Sub-Agent 并行处理
```

**Step 3: 执行（Execution）**
```
Lead Agent 创建 Sub-Agent Pool：
    
    Sub-Agent 1: 搜索市场概况
        - 调用 web_search
        - 返回：市场概况摘要
    
    Sub-Agent 2: 分析竞品
        - 调用 web_search
        - 返回：竞品对比表
    
    Sub-Agent 3: 技术趋势
        - 调用 web_fetch
        - 返回：技术趋势总结
    
    （3 个并行执行，最大并发 3）
```

**Step 4: 汇总（Synthesis）**
```
Lead Agent 收集 3 个 Sub-Agent 的结果
    ↓
整合信息，去重，验证
    ↓
生成结构化报告
    ↓
写入报告文件
```

**Step 5: 完成**
```
Lead Agent 标记任务完成
    ↓
通过 SSE 流返回报告
    ↓
MemoryManager 异步更新记忆
```

---

## 第六章：Skills、MCP、Memory 工作方式

### 6.1 Skills（技能系统）

**是什么**：Markdown 文件定义的操作手册，告诉 Agent 如何完成特定任务。

**文件结构**：
```
skills/public/deep-research/
├── SKILL.md          # 技能定义（必须）
├── README.md         # 使用说明
└── examples/         # 示例
```

**SKILL.md 示例**：
```markdown
# 深度研究技能

## 描述
进行深度网络研究，生成结构化报告

## 执行步骤
1. 搜索最新信息（web_search）
2. 分析来源可靠性
3. 综合整理报告
4. 添加引用来源

## 输出格式
- 执行摘要
- 详细发现
- 数据来源
- 建议行动
```

**工作方式**：
1. **渐进式加载**：只在需要时加载，节省 Token
2. **动态注入**：通过 `{skills_section}` 注入系统提示
3. **热插拔**：支持运行时启用/禁用

**伪代码**：
```python
class SkillManager:
    def load_skill(self, skill_name):
        skill_path = f"skills/public/{skill_name}/SKILL.md"
        content = read_file(skill_path)
        metadata, body = parse_markdown(content)
        return Skill(name=metadata.name, instructions=body)
    
    def inject_to_prompt(self, skill, prompt):
        return f"{skill.instructions}\n\n{prompt}"
```

### 6.2 MCP（Model Context Protocol）

**是什么**：标准化协议，让 AI 调用外部工具。

**配置示例**：
```json
{
    "mcpServers": {
        "github": {
            "enabled": true,
            "type": "stdio",
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-github"],
            "env": {"GITHUB_TOKEN": "ghp_xxx"}
        }
    }
}
```

**工作方式**：
1. **配置加载**：Gateway 读取 extensions_config.json
2. **工具发现**：Agent 启动时发现可用 MCP 工具
3. **调用执行**：通过 stdio/SSE 与 MCP Server 通信

**常用 MCP 工具**：
| 工具 | 功能 |
|------|------|
| GitHub | 操作仓库/Issue/PR |
| Slack | 发送消息 |
| PostgreSQL | 查询数据库 |
| Brave Search | 网络搜索 |

**伪代码**：
```python
async def use_mcp_tool(server_name, tool_name, params):
    client = MCPClient(server_name)
    result = await client.call_tool(tool_name, params)
    return result
```

### 6.3 Memory（长期记忆）

**是什么**：跨会话保持用户偏好和知识。

**存储结构**：
```json
{
    "user": {
        "workContext": {"summary": "...", "updatedAt": "..."},
        "personalContext": {"summary": "...", "updatedAt": "..."}
    },
    "facts": [
        {
            "id": "uuid",
            "content": "事实内容",
            "category": "preference",
            "confidence": 0.85
        }
    ]
}
```

**工作方式**：
1. **异步更新**：对话结束后异步提取事实
2. **防抖机制**：30 秒防抖，避免频繁写入
3. **动态注入**：通过 `{memory_context}` 注入系统提示

**伪代码**：
```python
class MemoryManager:
    def load_context(self, user_id):
        # 加载用户记忆
        return memory_store.get(user_id)
    
    def update_memory(self, user_id, new_facts):
        # 异步更新（防抖）
        queue.append(new_facts)
        if time_since_last_update > 30s:
            flush_to_disk()
    
    def format_for_prompt(self, context):
        # 格式化为 XML <memory> 块
        return f"<memory>{context}</memory>"
```

---

## 总结

### 核心要点

1. **Harness**：放大人类能力的驾驭系统
2. **分层架构**：Harness/App 分离，核心可复用
3. **Gateway**：后勤部，处理文件/模型/MCP/技能
4. **Agent**：状态机驱动，支持 Loop、Sub-Agent、工具接入
5. **扩展性**：Skills、MCP、Memory 按需加载

### 关键技术

| 技术 | 作用 | 实现 |
|------|------|------|
| LangGraph | Agent 运行时 | 状态机 + Checkpoint |
| Middleware | 请求处理链 | 16 个中间件 |
| Sub-Agent | 并行任务 | 双线程池调度 |
| Skills | 能力扩展 | Markdown 定义 |
| MCP | 外部工具 | 标准化协议 |
| Memory | 长期记忆 | 异步 + 防抖 |

### 记忆口诀

```
Harness - 放大器（不是替代，是放大）
分层   - 发动机 + 车身（可复用 + 可定制）
Gateway - 后勤部（杂活我来做）
Agent  - 总指挥（规划 → 执行 → 汇总）
Sub    - 小分队（并行加速）
Skills - 技能书（按需加载）
MCP    - 工具箱（标准化接口）
Memory - 备忘录（越用越懂你）
```

---

## 参考资料

- DeerFlow GitHub: https://github.com/bytedance/deer-flow
- ZRead: https://zread.ai/bytedance/deer-flow
- LangGraph: https://langchain-ai.github.io/langgraph/
- MCP: https://modelcontextprotocol.io

---

*深入浅出 DeerFlow 完整技术指南 | 8000+ 字*
