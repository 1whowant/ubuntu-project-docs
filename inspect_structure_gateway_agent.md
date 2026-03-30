# DeerFlow Gateway 层与 Agent 核心解析

> 目标读者：懂一点技术但不是专业人员  
> 写作风格：用"小明"用户故事视角，解释"用户做了什么 → 系统发生了什么"

---

## 第一部分：Gateway 层（后勤部）

### 1. Gateway 是什么

想象 DeerFlow 是一个大型公司，Gateway 就是公司的**后勤部门**——它不直接参与"思考"和"决策"（那是 AI 的事），但负责处理所有"杂活"：文件收发、设备清单、工具配置、档案管理。

**技术细节**：
- Gateway 是一个 **FastAPI** 应用，运行在端口 **8001**
- 它处理所有"非 AI"的事务：文件上传、模型列表查询、MCP 配置、技能管理、记忆读取
- 通过 Nginx 反向代理对外暴露，统一入口在端口 2026

**小明的例子**：
```
小明打开 DeerFlow 网页
    ↓
浏览器连接到 https://localhost:2026
    ↓
Nginx 根据路径分流：
    - /api/langgraph/* → 给 LangGraph（AI 大脑）
    - /api/* → 给 Gateway（后勤部）
    - /* → 给前端页面
```

---

### 2. 核心接口详解

#### 2.1 模型管理

**接口**：
- `GET /api/models` - 获取可用模型列表
- `GET /api/models/{name}` - 获取模型详情

**小明的例子**：
```
小明打开设置面板，看到"模型选择"下拉框
    ↓
Frontend 调用 GET /api/models
    ↓
Gateway 读取 config.yaml 的 models 部分
    ↓
返回：[
    {"name": "gpt-4o", "display_name": "GPT-4o", "supports_thinking": true},
    {"name": "claude-3-opus", "display_name": "Claude 3 Opus", "supports_thinking": false},
    {"name": "deepseek-v3", "display_name": "DeepSeek V3", "supports_thinking": true}
]
    ↓
下拉框显示：GPT-4o、Claude 3 Opus、DeepSeek V3
```

**关键点**：Gateway 只是"读配置"，真正的 AI 推理在 LangGraph 里进行。

---

#### 2.2 文件上传

**接口**：
- `POST /api/threads/{id}/uploads` - 上传文件到指定线程
- `GET /api/threads/{id}/uploads/list` - 列出已上传文件
- `DELETE /api/threads/{id}/uploads/{filename}` - 删除文件

**小明的例子**：
```
小明把一份 PDF 报告拖到聊天窗口
    ↓
Frontend 调用 POST /api/threads/abc123/uploads
    上传文件：report.pdf
    ↓
Gateway 接收文件，保存到：
    .deer-flow/threads/abc123/uploads/report.pdf
    ↓
检测到是 PDF，自动调用 markitdown 转成文本：
    .deer-flow/threads/abc123/uploads/report.md
    ↓
返回给 Frontend：
    {
        "filename": "report.pdf",
        "virtual_path": "/mnt/user-data/uploads/report.pdf",
        "markdown_file": "report.md",
        "markdown_virtual_path": "/mnt/user-data/uploads/report.md"
    }
    ↓
下次对话时，UploadsMiddleware 自动把文件信息注入到系统提示
    Agent 就能"看到"这份报告的内容了
```

**支持的文件类型**：PDF、PPT、Excel、Word（自动转换为 Markdown）

---

#### 2.3 MCP 配置管理

**什么是 MCP**：Model Context Protocol，让 AI 能调用外部工具的协议（比如 GitHub、Slack、文件系统）

**接口**：
- `GET /api/mcp/config` - 获取当前 MCP 配置
- `PUT /api/mcp/config` - 更新 MCP 配置

**小明的例子**：
```
小明想在 DeerFlow 里用 GitHub 工具
    ↓
小明在设置页面点击"添加 MCP"，填写：
    - 名称：github
    - 命令：npx -y @modelcontextprotocol/server-github
    - 环境变量：GITHUB_TOKEN=ghp_xxx
    ↓
Frontend 调用 PUT /api/mcp/config
    ↓
Gateway 更新 extensions_config.json 文件
    ↓
下次 Agent 启动时，自动加载 GitHub 工具
    小明可以说："帮我创建一个新的 GitHub 仓库"
    Agent 就能调用 GitHub MCP 工具完成操作
```

---

#### 2.4 技能管理

**什么是 Skill**：Markdown 文件定义的操作手册，告诉 Agent 如何完成特定任务（如深度研究、生成报告）

**接口**：
- `GET /api/skills` - 获取所有技能列表
- `GET /api/skills/{name}` - 获取技能详情
- `PUT /api/skills/{name}` - 启用/禁用技能
- `POST /api/skills/install` - 安装新技能

**小明的例子**：
```
小明想进行深度研究
    ↓
DeerFlow 检测到需要使用 deep-research 技能
    ↓
Agent 读取 skills/public/deep-research/SKILL.md
    ↓
SKILL.md 内容告诉 Agent：
    "深度研究需要以下步骤：
     1. 搜索最新信息
     2. 分析来源可靠性
     3. 综合整理报告
     4. 添加引用来源"
    ↓
Agent 按照手册执行深度研究
```

**渐进加载**：只有需要用到的技能才会加载，节省 token。

---

#### 2.5 记忆管理

**什么是 Memory**：跨会话记住用户偏好（如"小明喜欢详细分析"）

**接口**：
- `GET /api/memory` - 获取记忆数据
- `POST /api/memory/reload` - 重新加载记忆
- `GET /api/memory/config` - 获取记忆配置

**小明的例子**：
```
小明和 DeerFlow 聊了 10 次
    ↓
每次对话结束，MemoryMiddleware 把对话内容发给轻量级 LLM
    ↓
LLM 提取关键事实：
    - "小明是软件工程师"
    - "小明喜欢详细的代码示例"
    - "小明偏好 Python 语言"
    ↓
保存到 .deer-flow/memory.json
    ↓
下次小明打开 DeerFlow
    Gateway 读取记忆，注入到系统提示
    Agent 就知道："哦，是小明，他喜欢详细分析"
```

**记忆结构**：
```json
{
  "user": {
    "workContext": {"summary": "软件工程师", "updatedAt": "2024-01-15"},
    "personalContext": {"summary": "喜欢详细分析", "updatedAt": "2024-01-15"}
  },
  "facts": [
    {"content": "小明偏好 Python", "confidence": 0.9, "createdAt": "2024-01-10"}
  ]
}
```

---

#### 2.6 线程管理

**接口**：
- `DELETE /api/threads/{id}` - 删除线程本地数据

**小明的例子**：
```
小明想删除一个旧的对话
    ↓
Frontend 调用 DELETE /api/threads/abc123
    ↓
Gateway 删除 .deer-flow/threads/abc123/ 目录
    （包括 uploads、workspace、outputs 等子目录）
    ↓
小明的硬盘空间释放了
```

**注意**：这只是删除 DeerFlow 管理的本地文件，LangGraph 的线程状态需要另外清理。

---

#### 2.7 产物访问

**接口**：
- `GET /api/threads/{id}/artifacts/{path}` - 获取 Agent 生成的文件

**小明的例子**：
```
小明让 DeerFlow 生成了一份报告
    Agent 把报告保存到 /mnt/user-data/outputs/report.html
    ↓
小明在聊天界面点击"下载报告"
    ↓
Frontend 调用 GET /api/threads/abc123/artifacts/mnt/user-data/outputs/report.html
    ↓
Gateway 读取文件并返回
    ↓
浏览器下载 report.html
```

**安全机制**：HTML/SVG 等"活跃内容"强制作为附件下载，防止 XSS 攻击。

---

### 3. Gateway vs LangGraph 分工对比

| 任务 | Gateway (后勤部) | LangGraph (大脑) |
|------|------------------|------------------|
| 文件上传 | ✅ 接收并保存文件 | ❌ |
| AI 推理 | ❌ | ✅ 调用 LLM 模型 |
| 模型列表 | ✅ 读取配置返回 | ❌ |
| 工具调用 | ❌ | ✅ 执行工具逻辑 |
| MCP 配置 | ✅ 读写配置文件 | ❌ |
| 技能管理 | ✅ 扫描技能目录 | ❌ |
| 记忆读取 | ✅ 提供记忆数据 | ✅ 注入到提示 |
| 线程清理 | ✅ 删除本地文件 | ✅ 删除状态 |
| 产物下载 | ✅ 提供文件访问 | ❌ |

**简单理解**：
- **Gateway** = 后勤部，管杂活
- **LangGraph** = 大脑，管思考

---

## 第二部分：Agent 核心（大脑）

### 1. Lead Agent 是什么

**Lead Agent** 是 DeerFlow 的"总指挥"，负责协调整个 AI 系统的工作。

**技术细节**：
- 由 `make_lead_agent(config)` 函数创建
- 每个对话线程有一个独立的 Lead Agent 实例
- 支持动态选择模型（GPT-4、Claude、DeepSeek 等）

**小明的例子**：
```
小明在聊天框输入问题
    ↓
Frontend 发送请求到 LangGraph Server
    ↓
LangGraph 调用 make_lead_agent(config) 创建 Agent
    ↓
Agent 开始处理小明的请求
```

---

### 2. 简单任务 vs 复杂任务

#### 简单任务：直接回答

```
小明："什么是 AI Agent？"
    ↓
Lead Agent：这个问题简单，直接调用 LLM 回答
    ↓
返回："AI Agent 是一种能够感知环境、做出决策并执行行动的自主系统..."
    ↓
完成
```

#### 复杂任务：触发 Sub-Agent

```
小明："研究 2026 年 AI 趋势，写一份详细报告"
    ↓
Lead Agent 分析：这个任务复杂，需要拆解
    ↓
拆解为 3 个子任务：
    - 子任务 A：搜索 2026 AI 新闻（搜索代理）
    - 子任务 B：分析技术趋势（分析代理）
    - 子任务 C：生成报告文档（写作代理）
    ↓
并行启动 3 个 Sub-Agent（最多 3 个并发）
    ↓
通过 SSE 实时推送进度给小明：
    "搜索代理进行中..."
    "分析代理完成 50%..."
    "写作代理开始生成..."
    ↓
3 个子任务都完成后
    ↓
Lead Agent 汇总结果 → 生成最终报告
    ↓
返回给小明：完整的 2026 AI 趋势报告
```

---

### 3. Sub-Agent 详解

#### 并发控制

```python
# 双线程池设计
_scheduler_pool = ThreadPoolExecutor(max_workers=3)    # 调度器线程池
_execution_pool = ThreadPoolExecutor(max_workers=3)    # 执行器线程池

MAX_CONCURRENT_SUBAGENTS = 3    # 最大并发子代理数
TIMEOUT = 15分钟                # 子代理超时时间
```

**小明的例子**：
```
小明让 DeerFlow 同时研究 5 个话题
    ↓
Lead Agent 发现超过最大并发数 3
    ↓
SubagentLimitMiddleware 介入：
    - 只允许同时运行 3 个子代理
    - 第 4、5 个任务排队等待
    ↓
前 3 个完成后，再启动后 2 个
```

#### 生命周期

```
task() 工具被调用
    ↓
SubagentExecutor 创建执行器
    ↓
提交到 _scheduler_pool（后台线程）
    ↓
_scheduler_pool 提交到 _execution_pool 执行
    ↓
每 5 秒轮询进度
    ↓
通过 SSE 事件推送给 Frontend
    ↓
任务完成，返回结果给 Lead Agent
```

**状态流转**：
```
PENDING（等待中）→ RUNNING（运行中）→ COMPLETED（完成）
                                    → FAILED（失败）
                                    → TIMED_OUT（超时）
```

---

### 4. Context Engineering（上下文工程）

#### 问题：对话太长怎么办？

LLM 有上下文长度限制（比如 8K、32K、128K tokens），对话太长会：
1. 超出模型处理能力
2. 费用暴涨
3. 响应变慢

#### 解决方案：SummarizationMiddleware

```
对话已经进行了 50 轮，token 数超过 8000
    ↓
SummarizationMiddleware 触发
    ↓
调用轻量级模型（如 GPT-3.5）总结前 30 轮对话
    ↓
保留：
    - 摘要："之前讨论了 AI Agent 架构、Sub-Agent 设计..."
    - 最近 20 轮完整对话
    ↓
继续对话，token 数控制在限制内
```

**配置示例**：
```yaml
summarization:
  enabled: true
  trigger:
    - type: token_count
      value: 8000    # token 超过 8000 触发
  keep:
    type: count
    value: 20        # 保留最近 20 轮
```

---

### 5. Skill 渐进加载

#### 什么是 Skill

Skill 是一个 Markdown 文件（SKILL.md），定义了如何完成特定任务的操作手册。

**小明的例子**：
```
小明："深度研究量子计算最新进展"
    ↓
Lead Agent 识别到需要使用 deep-research 技能
    ↓
加载 skills/public/deep-research/SKILL.md
    ↓
SKILL.md 内容告诉 Agent：
    "深度研究的工作流程：
     1. 使用搜索工具收集最新信息
     2. 评估来源可靠性
     3. 交叉验证关键数据
     4. 生成带引用的报告"
    ↓
Agent 按手册执行深度研究任务
```

#### 为什么渐进加载

```
场景：DeerFlow 有 20 个技能，每个技能 500 tokens
    ↓
如果全部加载：20 × 500 = 10000 tokens
    ↓
渐进加载：只用 deep-research 技能 → 500 tokens
    ↓
节省 95% 的 token，响应更快、成本更低
```

---

### 6. Memory 长期记忆

#### 什么是 Memory

跨会话记住用户偏好，下次对话时自动注入到系统提示。

#### 工作流程

```
对话结束
    ↓
MemoryMiddleware 把对话加入队列
    ↓
30 秒防抖（避免频繁更新）
    ↓
轻量级 LLM 提取关键事实
    ↓
写入 .deer-flow/memory.json
    ↓
下次对话，自动注入到系统提示
```

**小明的例子**：
```
第一次对话：
    小明："我是 Python 开发者，喜欢详细的技术解释"
    ↓
    对话结束 → 提取事实 → 保存记忆

一周后第二次对话：
    小明："帮我写个脚本"
    ↓
    Agent 看到系统提示包含："用户是 Python 开发者，喜欢详细解释"
    ↓
    Agent 自动用 Python 写，并加上详细注释
```

#### 存储结构

```json
{
  "version": "1.0",
  "lastUpdated": "2024-01-15T10:30:00Z",
  "user": {
    "workContext": {
      "summary": "Python 开发者，专注后端开发",
      "updatedAt": "2024-01-15"
    },
    "personalContext": {
      "summary": "喜欢详细的技术解释",
      "updatedAt": "2024-01-10"
    }
  },
  "facts": [
    {
      "id": "fact_001",
      "content": "小明偏好 Python 语言",
      "confidence": 0.95,
      "createdAt": "2024-01-10"
    },
    {
      "id": "fact_002",
      "content": "小明喜欢详细的代码示例",
      "confidence": 0.88,
      "createdAt": "2024-01-12"
    }
  ]
}
```

---

## 第三部分：登录验证（重点检查）

### 问题：DeerFlow 有用户登录吗？

**检查方法**：

1. **查看 Gateway 路由**：检查 `app/gateway/app.py` 和所有 router 文件
   - 没有发现 `/api/auth` 或 `/api/login` 路由
   - 没有发现认证中间件

2. **查看代码中的 auth 相关文件**：
   ```bash
   find /home/ubuntu-wego/wego/projects/deer-flow -type f -name "*.py" | xargs grep -l "auth\|login" 2>/dev/null
   ```
   结果：只有 MCP OAuth、社区工具 API Key 等，没有用户认证系统

3. **查看 Frontend**：根据现有资料，Frontend 是 Next.js 应用
   - 没有发现登录页面相关代码

### 结论

**根据现有资料可以确定**：

| 项目 | 状态 |
|------|------|
| 用户登录系统 | ❌ 不存在 |
| 用户表/数据库 | ❌ 不存在 |
| 认证中间件 | ❌ 不存在 |
| 多用户支持 | ❌ 不支持 |

**DeerFlow 是单用户系统**：
- 没有内置的用户管理和认证机制
- 启动后直接可用，无需登录
- 适合个人本地使用或单用户部署
- **多用户需要外部扩展**（如通过 Nginx 添加 Basic Auth、OAuth 代理等）

**小明的例子**：
```
小明第一次使用 DeerFlow
    ↓
打开浏览器访问 http://localhost:2026
    ↓
直接看到聊天界面，没有登录页面
    ↓
立即开始对话
```

**安全提示**：
- 如果部署到公网，建议通过反向代理添加认证（如 Nginx Basic Auth、Authelia 等）
- 默认情况下任何人都能访问，不适合多用户场景

---

## 总结

### Gateway 层 = 后勤部
- **职责**：文件、模型、MCP、技能、记忆的管理
- **特点**：不处理 AI 推理，只处理"杂活"
- **端口**：8001（内部），通过 Nginx 2026 暴露

### Agent 核心 = 大脑
- **Lead Agent**：总指挥，协调所有任务
- **Sub-Agent**：处理复杂任务的"小工"
- **Middleware**：各种"插件"增强功能

### 关键设计
- **渐进加载**：需要时才加载，节省资源
- **双线程池**：调度 + 执行，防止阻塞
- **上下文摘要**：解决长对话问题
- **长期记忆**：跨会话记住用户偏好

### 登录验证
- **单用户系统**：没有登录机制
- **本地优先**：适合个人使用
- **多用户需扩展**：外部认证方案

---

*文档基于 DeerFlow 代码分析生成*
*参考资料：architecture_overview.md, claude_developer_guide.md, lead_agent_middleware.md, api_reference.md, dev_community_analysis.md*