# DeerFlow 技术深度解析：从 Harness 到 Agent 架构

> **目标**：深入浅出讲解 DeerFlow 核心技术架构  
> **风格**：技术深度 + 业务视角，产研团队友好  
> **字数**：5000+ 字  
> **版本**：2.0（最终版）

---

## 第1章：基础概念——Prompt、Context 与 Harness

### 1.1 三个核心概念

#### Prompt（提示词）

**是什么**：给 AI 的指令，告诉它要做什么。

**示例**：
```
"帮我研究 AI 编程助手市场，输出一份报告"
```

**局限**：单次指令，无法处理复杂多步骤任务。

#### Context（上下文）

**是什么**：AI 能看到的背景信息，包括对话历史、用户偏好、相关文件等。

**示例**：
```
- 用户之前问过："什么是 Copilot"
- 用户上传了：一份市场报告.pdf
- 系统提示："你是一个专业研究员"
```

**关键**：Context 决定 AI 回答的质量。好的 Context 工程能显著提升效果。

#### Harness（驾驭系统）

**是什么**：不是直接替代人类，而是放大人类能力的系统。

**核心能力**：
1. **任务拆解**：把复杂任务分解为可执行的子任务
2. **资源调度**：自动选择最合适的模型和工具
3. **并行加速**：多个子任务同时执行
4. **质量保证**：自动验证中间结果
5. **成果固化**：产出可交付的完整成果

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

---

## 第2章：DeerFlow 架构概览

### 2.1 整体架构图

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

### 2.2 组件职责

| 组件 | 核心职责 | 技术选型 | 设计理由 |
|------|---------|---------|---------|
| **Nginx** | 统一入口，路由分发 | Nginx | 成熟稳定，支持高并发 |
| **Frontend** | 用户界面 | Next.js 15 | SSR/SSG，React 生态 |
| **LangGraph Server** | Agent 运行时 | Python + LangGraph | 状态机驱动，支持复杂工作流 |
| **Gateway API** | 业务 API | FastAPI | 高性能异步，Python 生态 |

### 2.3 数据流转

**整体流程**：
```
用户请求 → Nginx → 路由分发 → 各服务处理 → 返回响应
```

**路由规则**：
| 路径前缀 | 目标服务 | 处理内容 |
|---------|---------|---------|
| `/api/langgraph/*` | LangGraph Server | AI 推理、Agent 编排 |
| `/api/*` | Gateway API | 文件、模型、MCP、技能 |
| `/*` | Frontend | 静态资源、页面渲染 |

### 2.4 Harness/App 分层

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

## 第3章：Gateway 层详解

### 3.1 Gateway 定位

**一句话**：处理所有非 AI 的事务性工作。

**职责边界**：
| 类型 | Gateway 处理 | LangGraph 处理 |
|------|-------------|---------------|
| 文件上传/下载 | ✅ | ❌ |
| 模型列表查询 | ✅ | ❌ |
| MCP/Skills 配置 | ✅ | ❌ |
| AI 推理/对话 | ❌ | ✅ |

### 3.2 用例子串起接口

#### 例子1：文件上传

**场景**：用户上传一份 PDF 报告

**流程**：
```
用户上传 report.pdf
    ↓
POST /api/threads/{id}/uploads
    ↓
Gateway 接收文件
    ↓
保存到线程目录
    ↓
PDF → Markdown 转换
    ↓
返回文件元数据
    ↓
下次对话自动注入到 AI 上下文
```

**关键接口**：
- `POST /api/threads/{id}/uploads` - 上传文件
- 支持：PDF、Word、PPT、Excel、图片、文本
- 自动转换：PDF/Word/PPT → Markdown

#### 例子2：模型切换

**场景**：用户想从 GPT-4o 切换到 DeepSeek V3

**流程**：
```
用户打开设置面板
    ↓
GET /api/models
    ↓
Gateway 读取 config.yaml
    ↓
返回模型列表（名称、能力、配置）
    ↓
用户选择 DeepSeek V3
    ↓
下次对话使用新模型
```

**关键接口**：
- `GET /api/models` - 获取模型列表
- `GET /api/models/{name}` - 获取模型详情

#### 例子3：配置 MCP 工具

**场景**：用户想添加 GitHub 工具

**流程**：
```
用户填写 MCP 配置
    - 名称：github
    - 命令：npx -y @modelcontextprotocol/server-github
    - Token：ghp_xxx
    ↓
PUT /api/mcp/config
    ↓
Gateway 更新 extensions_config.json
    ↓
下次 Agent 启动时加载 GitHub 工具
    ↓
用户可以说："帮我创建 GitHub 仓库"
```

**关键接口**：
- `GET /api/mcp/config` - 获取配置
- `PUT /api/mcp/config` - 更新配置

#### 例子4：管理 Skills

**场景**：用户想启用"深度研究"技能

**流程**：
```
用户打开技能管理页面
    ↓
GET /api/skills
    ↓
返回技能列表（名称、描述、状态）
    ↓
用户启用"深度研究"
    ↓
PUT /api/skills/deep-research
    ↓
下次对话可使用该技能
```

**关键接口**：
- `GET /api/skills` - 获取技能列表
- `PUT /api/skills/{name}` - 启用/禁用技能

### 3.3 Gateway 接口总结

| 接口 | 方法 | 功能 | 所属例子 |
|------|------|------|---------|
| `/api/threads/{id}/uploads` | POST | 上传文件 | 例子1 |
| `/api/models` | GET | 模型列表 | 例子2 |
| `/api/mcp/config` | GET/PUT | MCP 配置 | 例子3 |
| `/api/skills` | GET | 技能列表 | 例子4 |
| `/api/skills/{name}` | PUT | 启用/禁用技能 | 例子4 |

---

## 第4章：Agent 架构详解

### 4.1 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     Agent 运行时架构                             │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    LangGraph Server                      │   │
│  │                      (Port 2024)                         │   │
│  │                                                         │   │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │   │
│  │  │   Thread    │    │  Middleware │    │   Sandbox   │  │   │
│  │  │   State     │───→│    Chain    │───→│  Provider   │  │   │
│  │  │             │    │  (16个)     │    │             │  │   │
│  │  └─────────────┘    └─────────────┘    └─────────────┘  │   │
│  │         │                   │                   │        │   │
│  │         ▼                   ▼                   ▼        │   │
│  │  ┌─────────────────────────────────────────────────────┐  │   │
│  │  │                  Lead Agent                         │  │   │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐         │  │   │
│  │  │  │  Planning│→ │ Execution│→ │ Synthesis│         │  │   │
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
│  └─────────────────────────────────────────────────────┘  │   │
│                                                           │   │
└───────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Loop 实现（伪代码）

**核心机制**：状态机驱动的循环

```python
# LangGraph 状态定义
class ThreadState:
    messages: list          # 对话历史
    step_count: int         # 当前步骤
    next_action: str        # 下一步动作
    is_complete: bool       # 是否完成

# Agent Loop 伪代码
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
- **断点续传**：任意步骤可暂停、恢复
- **循环支持**：支持反思、迭代、回溯
- **流式输出**：通过 SSE 实时推送中间结果

### 4.3 组件接入

#### Tools（工具系统）

**三层工具架构**：
```
┌─────────────────────────────────────────────────┐
│                 Tool Layer                      │
├─────────────────────────────────────────────────┤
│  1. Built-in Tools（内置工具）                   │
│     - web_search, web_fetch                     │
│     - bash, read_file, write_file               │
├─────────────────────────────────────────────────┤
│  2. MCP Tools（外部工具）                        │
│     - GitHub, Slack, PostgreSQL                 │
│     - 通过 MCP 协议接入                          │
├─────────────────────────────────────────────────┤
│  3. Skills Tools（技能工具）                     │
│     - deep-research, report-generation          │
│     - 通过 SKILL.md 定义                         │
└─────────────────────────────────────────────────┘
```

#### Sandbox（沙箱系统）

**作用**：提供隔离的执行环境

**伪代码**：
```python
class SandboxProvider:
    def create(self, config) -> Session:
        # 创建隔离容器
        pass
    
    def execute(self, session, command):
        # 在沙箱中执行命令
        pass
    
    def read_file(self, session, path):
        # 读取沙箱内文件
        pass
    
    def write_file(self, session, path, content):
        # 写入文件到沙箱
        pass
```

**隔离机制**：
- 容器隔离（Docker/K8s）
- 网络隔离（默认无网络）
- 资源限制（CPU/内存配额）

#### Memory（记忆系统）

**作用**：跨会话保持用户偏好和知识

**伪代码**：
```python
class MemoryManager:
    def load_context(self, user_id) -> dict:
        # 加载用户记忆
        pass
    
    def update_memory(self, user_id, new_facts):
        # 更新记忆（异步+防抖）
        pass
    
    def format_for_prompt(self, context) -> str:
        # 格式化为 Prompt 可用的上下文
        pass
```

### 4.4 Sub-Agent 原理

**是什么**：Lead Agent 创建的子代理，负责执行子任务

**触发条件**：
1. 任务复杂，需要并行处理
2. 需要隔离上下文，避免干扰
3. 需要长时间运行，不阻塞主流程

**调度机制**（伪代码）：
```python
class SubAgentManager:
    def __init__(self):
        self.scheduler_pool = ThreadPool(max_workers=3)  # 调度池
        self.execution_pool = ThreadPool(max_workers=3)  # 执行池
    
    def spawn(self, task, parent_state) -> SubAgent:
        # 创建子代理
        sub_agent = SubAgent(
            task=task,
            context=inherit_context(parent_state),  # 继承部分上下文
            sandbox=parent_state.sandbox  # 共享沙箱
        )
        
        # 提交到调度池
        future = self.scheduler_pool.submit(self._run, sub_agent)
        return future
    
    def _run(self, sub_agent):
        # 在执行池中运行
        result = sub_agent.execute()
        return result
```

**关键参数**：
- 最大并发：3 个（硬编码）
- 超时控制：15 分钟
- 上下文继承：选择性继承父代理状态

### 4.5 MCP 与 Skills 实现

#### MCP（Model Context Protocol）

**是什么**：标准化协议，让 AI 调用外部工具

**实现方式**：
```python
# MCP 配置示例（extensions_config.json）
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

# Agent 使用 MCP
async def use_mcp_tool(server_name, tool_name, params):
    client = MCPClient(server_name)
    result = await client.call_tool(tool_name, params)
    return result
```

#### Skills

**是什么**：Markdown 文件定义的操作手册

**实现方式**：
```python
# Skill 加载
class SkillManager:
    def load_skill(self, skill_name) -> Skill:
        skill_path = f"skills/public/{skill_name}/SKILL.md"
        content = read_file(skill_path)
        
        # 解析 frontmatter 和正文
        metadata, body = parse_markdown(content)
        
        return Skill(
            name=metadata.name,
            description=metadata.description,
            instructions=body
        )
    
    def inject_to_prompt(self, skill: Skill, prompt: str) -> str:
        # 将技能指令注入到系统提示
        return f"{skill.instructions}\n\n{prompt}"
```

**渐进式加载**：
- 只在需要时加载，节省 Token
- 支持热插拔，无需重启

### 4.6 完整案例：从会话到完成

**场景**：用户请求"研究 AI 编程助手市场，生成报告"

**完整流程**：

```
Step 1: 发起会话
─────────────────
用户输入："研究 AI 编程助手市场，生成报告"
    ↓
Frontend 发送请求到 /api/langgraph
    ↓
LangGraph Server 创建 Thread（对话线程）
    ↓
初始化 ThreadState（消息历史、上下文）

Step 2: 规划（Planning）
────────────────────────
Lead Agent 分析任务：
    "这是一个研究任务，需要：
     1. 搜索市场概况
     2. 分析主要竞品
     3. 整理技术趋势
     4. 生成报告"
    ↓
创建执行计划（Plan）
    ↓
决定：需要创建 3 个 Sub-Agent 并行处理

Step 3: 执行（Execution）
────────────────────────
Lead Agent 创建 Sub-Agent Pool：
    
    Sub-Agent 1: 搜索市场概况
        - 调用 web_search 工具
        - 收集市场规模、增长率数据
        - 返回：市场概况摘要
    
    Sub-Agent 2: 分析竞品
        - 调用 web_search 工具
        - 分析 GitHub Copilot、Cursor、Codeium
        - 返回：竞品对比表
    
    Sub-Agent 3: 技术趋势
        - 调用 web_fetch 工具
        - 阅读技术博客、论文
        - 返回：技术趋势总结
    
    （3 个 Sub-Agent 并行执行，最大并发 3）

Step 4: 汇总（Synthesis）
────────────────────────
Lead Agent 收集 3 个 Sub-Agent 的结果
    ↓
整合信息，去重，验证
    ↓
生成结构化报告：
    - 执行摘要
    - 市场概况
    - 竞品分析
    - 技术趋势
    - 建议行动
    ↓
调用 Sandbox 写入报告文件

Step 5: 完成
──────────
Lead Agent 标记任务完成
    ↓
通过 SSE 流返回报告内容
    ↓
Frontend 展示完整报告
    ↓
MemoryManager 异步更新记忆：
    "用户关注 AI 编程助手市场"
```

**关键技术点**：
- **ThreadState**：保持完整对话上下文
- **Middleware 链**：16 个中间件处理（沙箱、技能、记忆等）
- **Sub-Agent Pool**：双线程池调度，最大并发 3
- **Sandbox**：隔离执行环境，文件操作安全
- **Memory**：异步更新，跨会话记忆

---

## 总结

### 核心要点

1. **Harness**：不是替代人类，而是放大人类能力的驾驭系统
2. **分层架构**：Nginx + 4 服务，Harness/App 分离
3. **Gateway**：后勤部，处理文件/模型/MCP/技能
4. **Agent**：状态机驱动，支持 Loop、Sub-Agent、工具接入

### 关键技术

| 技术 | 作用 | 实现方式 |
|------|------|---------|
| LangGraph | Agent 运行时 | 状态机 + Checkpoint |
| Middleware | 请求处理链 | 16 个中间件 |
| Sub-Agent | 并行任务 | 双线程池调度 |
| Sandbox | 隔离执行 | Docker/K8s Provider |
| MCP | 外部工具 | 标准化协议 |
| Skills | 能力扩展 | Markdown 定义 |

### 记忆口诀

```
Harness - 放大器（不是替代，是放大）
分层   - 发动机 + 车身（可复用 + 可定制）
Gateway - 后勤部（杂活我来做）
Agent  - 总指挥（规划 → 执行 → 汇总）
Sub    - 小分队（并行加速）
```

---

## 参考资料

- DeerFlow GitHub: https://github.com/bytedance/deer-flow
- LangGraph: https://langchain-ai.github.io/langgraph/
- MCP: https://modelcontextprotocol.io
- FastAPI: https://fastapi.tiangolo.com

---

*深入浅出 DeerFlow 技术解析 | 5000+ 字完整版*
