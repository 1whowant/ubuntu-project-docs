# DeerFlow 技术分享：整体架构与 Gateway 层深度解析

> **目标**：深入浅出讲解 DeerFlow 整体架构和 Gateway 层  
> **风格**：技术深度 + 业务视角，产研团队友好  
> **章节**：Part 1 - 架构基础篇

---

## 一、执行摘要

### 1.1 一句话定义

**DeerFlow 是一个基于 LangGraph 的超级 Agent Harness，通过 Harness/App 分层架构，实现复杂任务的自动分解、多 Agent 并行执行和端到端交付。**

### 1.2 核心架构特点

| 特点 | 说明 | 业务价值 |
|------|------|---------|
| **分层架构** | Harness/App 严格分离 | 核心可复用，业务可定制 |
| **多服务设计** | Nginx + LangGraph + Gateway + Frontend | 职责清晰，可独立扩展 |
| **LangGraph 底座** | 状态机驱动的 Agent 编排 | 支持复杂工作流和断点续传 |
| **渐进式技能** | 按需加载 Skills | 节省 Token，提升效率 |

---

## 二、整体架构全景

### 2.1 架构概览图

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户访问层                               │
│              浏览器 → localhost:2026 (Nginx 统一入口)            │
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

### 2.2 组件职责详解

| 组件 | 技术栈 | 核心职责 | 设计理由 |
|------|--------|---------|---------|
| **Nginx** | Nginx | 统一入口，路由分发，SSL 终止 | 成熟稳定，支持高并发 |
| **Frontend** | Next.js 15 + React | 用户界面，聊天交互 | SSR/SSG，React 生态成熟 |
| **LangGraph Server** | Python + LangGraph | Agent 运行时，状态管理 | 状态机驱动，支持断点续传 |
| **Gateway API** | FastAPI | 文件上传、模型管理、MCP/技能配置 | 高性能异步，Python 生态 |

### 2.3 请求流转示例

**场景：用户发送消息"帮我研究 AI 趋势"**

```
用户输入消息
    ↓
Frontend (3000) 打包请求，发送到 /api/langgraph
    ↓
Nginx (2026) 识别路径，转发到 LangGraph Server (2024)
    ↓
LangGraph Server 创建/恢复 Thread（对话线程）
    ↓
调用 make_lead_agent() 创建总指挥
    ↓
Middleware 链处理（沙箱、技能、记忆等）
    ↓
Lead Agent 分解任务 → 调用工具/Sub-Agent
    ↓
通过 SSE 流式返回结果
    ↓
Frontend 实时展示回复
```

---

## 三、Harness/App 分层设计

### 3.1 分层架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        App Layer                                │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │    Gateway      │  │    Channels     │  │   Web Frontend  │  │
│  │   (FastAPI)     │  │   (IM 集成)      │  │   (Next.js)     │  │
│  │                 │  │                 │  │                 │  │
│  │ • 文件上传      │  │ • Slack         │  │ • 聊天界面      │  │
│  │ • 模型管理      │  │ • Discord       │  │ • 设置面板      │  │
│  │ • MCP 配置      │  │ • 其他 IM       │  │ • 文件展示      │  │
│  │ • 技能管理      │  │                 │  │                 │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ (单向依赖：App → Harness)
┌─────────────────────────────────────────────────────────────────┐
│                      Harness Layer                              │
│              (可发布包：packages/harness/deerflow)                │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐    │
│  │  agents/  │  │ sandbox/  │  │  models/  │  │  tools/   │    │
│  │           │  │           │  │           │  │           │    │
│  │• Lead     │  │• Docker   │  │• 模型工厂 │  │• 内置工具 │    │
│  │  Agent    │  │  Provider │  │• 多提供商 │  │• MCP 工具 │    │
│  │• Sub-     │  │• K8s      │  │  支持     │  │• Skills   │    │
│  │  Agent    │  │  Provider │  │           │  │  工具     │    │
│  │• 编排逻辑 │  │           │  │           │  │           │    │
│  └───────────┘  └───────────┘  └───────────┘  └───────────┘    │
│                                                                 │
│  核心设计原则：                                                 │
│  1. 业务无关：不依赖任何 App 层代码                             │
│  2. 可发布：独立的 Python 包，可 pip install                    │
│  3. 可扩展：Provider 模式支持多后端                              │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 为什么分层？

**问题背景**：
- 如果所有代码混在一起，业务逻辑和核心能力耦合
- 想复用核心能力时，必须带上所有业务代码
- 升级核心功能时，可能破坏业务代码

**分层解决方案**：

| 维度 | Harness 层 | App 层 |
|------|-----------|--------|
| **定位** | 可复用的核心能力 | 业务特定的实现 |
| **依赖** | 不依赖 App | 依赖 Harness |
| **发布** | 独立 Python 包 | 业务代码 |
| **测试** | 纯单元测试 | 集成测试 |
| **示例** | Agent 编排、沙箱、工具 | 文件上传接口、IM 集成 |

**类比理解**：
- **Harness** = 汽车发动机（标准化，可卖给任何车企）
- **App** = 车身设计（各品牌差异化竞争点）

### 3.3 依赖规则强制执行

**规则**：`App 可导入 deerflow，deerflow 严禁导入 app`

**实现机制**：

1. **运行时检查**：
```python
# deerflow/__init__.py
import inspect

def _enforce_boundary():
    for frame in inspect.stack():
        module = inspect.getmodule(frame[0])
        if module and module.__name__.startswith('app.'):
            raise ImportError(" deerflow 禁止导入 app 模块")
_enforce_boundary()
```

2. **CI/CD 检查**：
```yaml
# .github/workflows/ci.yml
- name: Check Import Boundaries
  run: lint-imports --config import-linter.cfg
  # 规则：deerflow.* 不能依赖 app.*
```

3. **代码审查清单**：
   - [ ] 新增文件是否遵循分层？
   - [ ] 是否存在跨层导入？
   - [ ] Harness 是否保持业务无关？

---

## 四、Gateway 层详解

### 4.1 Gateway 定位

**一句话**：Gateway 是"后勤部"，处理所有非 AI 的事务性工作。

**职责边界**：
| 类型 | Gateway 处理 | LangGraph 处理 |
|------|-------------|---------------|
| 文件上传/下载 | ✅ | ❌ |
| 模型列表查询 | ✅ | ❌ |
| MCP 配置管理 | ✅ | ❌ |
| 技能管理 | ✅ | ❌ |
| AI 推理/对话 | ❌ | ✅ |
| Agent 编排 | ❌ | ✅ |

### 4.2 核心接口详解

#### 4.2.1 文件上传

**接口**：`POST /api/threads/{thread_id}/uploads`

**流程**：
```
Frontend 上传文件
    ↓
Gateway 接收 multipart/form-data
    ↓
保存到 .deer-flow/threads/{id}/uploads/
    ↓
检测文件类型，调用 markitdown 转换
    PDF/Word/PPT → Markdown
    ↓
返回文件元数据（含虚拟路径）
    ↓
UploadsMiddleware 自动注入到 AI 上下文
```

**技术细节**：
- 支持格式：PDF、Word、PPT、Excel、图片、文本
- 存储结构：每个线程独立目录，隔离安全
- 虚拟路径：`/mnt/user-data/uploads/{filename}`

#### 4.2.2 模型管理

**接口**：`GET /api/models`

**流程**：
```
Frontend 请求模型列表
    ↓
Gateway 读取 config.yaml
    ↓
解析 models 配置
    ↓
返回：名称、显示名、是否支持思考/视觉
```

**配置示例**（config.yaml）：
```yaml
models:
  - name: gpt-4o
    display_name: GPT-4o
    use: langchain_openai:ChatOpenAI
    model: gpt-4o
    api_key: $OPENAI_API_KEY
    supports_thinking: true
    supports_vision: true
    
  - name: deepseek-v3
    display_name: DeepSeek V3
    use: deerflow.models.patched_deepseek:PatchedChatDeepSeek
    model: deepseek-reasoner
    api_key: $DEEPSEEK_API_KEY
    supports_thinking: true
```

#### 4.2.3 MCP 配置管理

**什么是 MCP**：Model Context Protocol，让 AI 调用外部工具的通用接口。

**接口**：
- `GET /api/mcp/config` - 获取配置
- `PUT /api/mcp/config` - 更新配置

**常用 MCP 工具**：
| 工具 | 功能 | 使用场景 |
|------|------|---------|
| GitHub | 操作仓库/Issue/PR | "帮我创建新仓库" |
| Slack | 发送消息 | "通知团队更新" |
| 文件系统 | 读写文件 | "读取配置文件" |
| PostgreSQL | 查询数据库 | "查询用户数据" |

#### 4.2.4 技能管理

**什么是 Skill**：Markdown 文件定义的操作手册，告诉 Agent 如何完成特定任务。

**接口**：
- `GET /api/skills` - 获取技能列表
- `PUT /api/skills/{name}` - 启用/禁用技能

**Skill 文件结构**：
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
```

### 4.3 Gateway 核心接口总结

| 接口 | 方法 | 功能 | 使用场景 |
|------|------|------|---------|
| `/api/models` | GET | 获取模型列表 | 切换 AI模型 |
| `/api/threads/{id}/uploads` | POST | 上传文件 | 上传 PDF/Word |
| `/api/mcp/config` | GET/PUT | MCP 配置 | 添加 GitHub 工具 |
| `/api/skills` | GET | 技能列表 | 查看可用技能 |
| `/api/skills/{name}` | PUT | 启用/禁用技能 | 开启新功能 |

---

## 五、技术亮点与业务价值

### 5.1 架构设计亮点

| 设计 | 技术实现 | 业务价值 |
|------|---------|---------|
| **分层架构** | Harness/App 分离 | 核心可复用，降低维护成本 |
| **多服务设计** | Nginx + 4 个服务 | 职责清晰，可独立扩展 |
| **LangGraph 底座** | 状态机驱动 | 支持复杂工作流，断点续传 |
| **渐进式技能** | 按需加载 Skills | 节省 Token，提升响应速度 |

### 5.2 Gateway 层价值

| 能力 | 解决的问题 | 量化收益 |
|------|-----------|---------|
| 文件处理 | AI 无法直接处理文件 | 支持 10+ 文件格式 |
| 模型管理 | 多模型切换复杂 | 一键切换，配置化 |
| MCP 集成 | 外部工具接入困难 | 标准化接口，即插即用 |
| 技能管理 | 能力扩展不灵活 | 热插拔，无需重启 |

---

## 六、总结

### 6.1 核心要点

1. **整体架构**：Nginx 统一入口，4 个服务各司其职
2. **分层设计**：Harness 核心可复用，App 业务可定制
3. **Gateway 定位**：后勤部，处理非 AI 的事务性工作
4. **技术选型**：LangGraph 状态机、FastAPI 高性能、Next.js 现代前端

### 6.2 记忆口诀

```
2026 - Nginx 门卫
3000  - Frontend 前台
2024  - LangGraph AI 大脑
8001  - Gateway 后勤部

Harness - 发动机（可复用）
App     - 车身（定制化）
```

### 6.3 下一步

- **Part 2**：Agent 运行时详解（Lead Agent、Sub-Agent、Middleware）
- **Part 3**：Sandbox 与 Memory 系统
- **Part 4**：应用场景与最佳实践

---

## 参考资料

- DeerFlow GitHub: https://github.com/bytedance/deer-flow
- 架构文档: backend/docs/ARCHITECTURE.md
- Gateway 文档: backend/docs/API.md
- LangGraph: https://langchain-ai.github.io/langgraph/

---

*Part 1 - 架构基础篇 | 深入浅出 DeerFlow 技术分享*
