# DeerFlow 第二章：Gateway 层详解（后勤部的工作）

> **目标**：理解 Gateway 处理哪些事务，以及如何处理  
> **风格**：用"小明"的用户故事，解释"用户做了什么 → 系统发生了什么"  
> **章节**：第 2 章，共 5 章

---

## 2.1 Gateway 是什么

### 通俗解释

想象 DeerFlow 是一个大型公司：
- **LangGraph Server** = CEO 办公室（做决策、思考）
- **Gateway** = 后勤部（处理杂活、不直接参与决策）

**Gateway 处理的事务**：
- 文件收发（上传/下载）
- 设备清单（模型列表查询）
- 工具配置（MCP 配置管理）
- 档案管理（技能管理、记忆读取）

**技术细节**：
- Gateway 是一个 **FastAPI** 应用
- 运行在端口 **8001**
- 通过 Nginx 反向代理对外暴露（统一入口 2026）

---

## 2.2 小明的文件上传之旅

### 场景：拖一份 PDF 到聊天窗口

**小明看到了什么？**

文件显示在聊天界面，进度条显示上传中，完成后 AI 说"我已阅读这份报告"。

**背后发生了什么？**

```
小明拖动 report.pdf 到聊天窗口
    ↓
Frontend 调用 POST /api/threads/abc123/uploads
    请求内容：文件数据 + 文件名 + 线程ID
    ↓
Nginx 转发给 Gateway（8001）
    ↓
Gateway 接收文件，保存到：
    .deer-flow/threads/abc123/uploads/report.pdf
    ↓
Gateway 检测文件类型："是 PDF，需要转换"
    ↓
调用 markitdown 工具，转换为 Markdown：
    .deer-flow/threads/abc123/uploads/report.md
    ↓
Gateway 返回给 Frontend：
    {
        "filename": "report.pdf",
        "virtual_path": "/mnt/user-data/uploads/report.pdf",
        "markdown_file": "report.md"
    }
    ↓
Frontend 显示"上传成功"
    ↓
下次对话时，UploadsMiddleware 自动把文件内容
    注入到 AI 的系统提示中
    ↓
AI 能"看到"报告内容，基于它回答问题
```

**文件处理流程详解**：

| 步骤 | 做什么 | 技术细节 |
|------|--------|---------|
| 1. 接收 | Gateway 接收 HTTP 请求 | multipart/form-data 格式 |
| 2. 保存 | 存储到线程目录 | `.deer-flow/threads/{id}/uploads/` |
| 3. 转换 | PDF → Markdown | 使用 markitdown 工具 |
| 4. 返回 | 返回文件元数据 | 包含虚拟路径供沙箱使用 |
| 5. 注入 | UploadsMiddleware 工作 | 自动添加到 AI 上下文 |

**支持的文件类型**：
- PDF（自动转 Markdown）
- Word（.doc/.docx，自动转 Markdown）
- PPT（.ppt/.pptx，自动转 Markdown）
- Excel（.xls/.xlsx）
- 图片（PNG、JPG 等）
- 文本文件（.txt、.md、.py 等）

---

## 2.3 小明查看模型列表

### 场景：打开设置面板，想切换 AI 模型

**小明看到了什么？**

下拉框显示：GPT-4o、Claude 3 Opus、DeepSeek V3、Gemini 2.5...

**背后发生了什么？**

```
小明打开设置面板
    ↓
Frontend 调用 GET /api/models
    ↓
Nginx 转发给 Gateway（8001）
    ↓
Gateway 读取 config.yaml 的 models 部分
    ↓
返回模型列表：
[
    {
        "name": "gpt-4o",
        "display_name": "GPT-4o",
        "supports_thinking": true,
        "supports_vision": true
    },
    {
        "name": "claude-3-opus", 
        "display_name": "Claude 3 Opus",
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
    ↓
Frontend 展示在下拉框
    ↓
小明选择"DeepSeek V3"
    ↓
下次对话时，LangGraph 使用 DeepSeek V3 模型
```

**关键点**：
- Gateway 只是"读配置"，不实际运行 AI
- 真正的 AI 推理在 LangGraph Server 里进行
- 模型配置在 `config.yaml` 中定义

**模型配置示例**（config.yaml）：
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

---

## 2.4 小明配置 MCP 工具

### 场景：想在 DeerFlow 里使用 GitHub 工具

**什么是 MCP？**

MCP = Model Context Protocol（模型上下文协议）

通俗解释：让 AI 能调用外部工具的"通用接口"

**小明做了什么？**

在设置页面点击"添加 MCP"，填写：
- 名称：github
- 命令：`npx -y @modelcontextprotocol/server-github`
- 环境变量：`GITHUB_TOKEN=ghp_xxx`

**背后发生了什么？**

```
小明填写 MCP 配置，点击保存
    ↓
Frontend 调用 PUT /api/mcp/config
    请求体：{
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
    ↓
Gateway 更新 extensions_config.json 文件
    ↓
返回成功状态
    ↓
下次 Agent 启动时，自动加载 GitHub 工具
    ↓
小明可以说："帮我创建一个新的 GitHub 仓库"
    Agent 就能调用 GitHub MCP 工具完成操作
```

**常用 MCP 工具**：
| 工具 | 功能 |
|------|------|
| GitHub | 操作 GitHub 仓库、Issue、PR |
| Slack | 发送消息、查询频道 |
| 文件系统 | 读写本地文件 |
| PostgreSQL | 查询数据库 |
| Brave Search | 网络搜索 |

---

## 2.5 小明管理技能（Skills）

### 场景：查看和启用/禁用技能

**什么是 Skill？**

Skill = 操作手册（Markdown 文件）

告诉 Agent 如何完成特定任务，如：
- 深度研究（deep-research）
- 生成报告（report-generation）
- 创建幻灯片（slide-creation）

**小明看到了什么？**

技能列表页面，显示：
- ✅ 深度研究（已启用）
- ✅ 生成报告（已启用）
- ❌ 网页生成（已禁用）

**背后发生了什么？**

```
小明打开技能管理页面
    ↓
Frontend 调用 GET /api/skills
    ↓
Gateway 扫描 skills/ 目录
    ↓
返回技能列表：
[
    {
        "name": "deep-research",
        "display_name": "深度研究",
        "enabled": true,
        "description": "进行深度网络研究，生成结构化报告"
    },
    {
        "name": "web-page",
        "display_name": "网页生成", 
        "enabled": false,
        "description": "生成响应式网页"
    }
]
    ↓
小明点击"启用网页生成"
    ↓
Frontend 调用 PUT /api/skills/web-page
    请求体：{"enabled": true}
    ↓
Gateway 更新技能状态
    ↓
下次对话时，Agent 可以使用网页生成技能
```

**Skill 文件结构**：
```
skills/public/deep-research/
├── SKILL.md          # 技能定义（必须）
├── README.md         # 使用说明
└── examples/         # 示例
    └── example1.md
```

**SKILL.md 内容示例**：
```markdown
# 深度研究技能

## 描述
进行深度网络研究，生成结构化报告

## 使用场景
- 市场调研
- 竞品分析
- 技术趋势研究

## 执行步骤
1. 搜索最新信息（使用 web_search 工具）
2. 分析来源可靠性
3. 综合整理报告
4. 添加引用来源

## 输出格式
- 执行摘要
- 详细发现
- 数据来源
- 建议行动
```

---

## 2.6 Gateway 核心接口总结

| 接口 | 功能 | 小明场景 |
|------|------|---------|
| `GET /api/models` | 获取模型列表 | 查看可用 AI 模型 |
| `POST /api/threads/{id}/uploads` | 上传文件 | 上传 PDF 报告 |
| `GET /api/mcp/config` | 获取 MCP 配置 | 查看已配置的工具 |
| `PUT /api/mcp/config` | 更新 MCP 配置 | 添加 GitHub 工具 |
| `GET /api/skills` | 获取技能列表 | 查看可用技能 |
| `PUT /api/skills/{name}` | 启用/禁用技能 | 开启网页生成功能 |

---

## 本章小结

**Gateway = 后勤部**

不直接参与 AI 思考，但处理所有"杂活"：
- 📁 文件上传/管理
- 📋 模型列表查询
- 🔧 MCP 工具配置
- 📖 技能管理

**记住**：
- Gateway 运行在 **8001** 端口
- 通过 Nginx（2026）对外暴露
- 只是"读配置/存文件"，真正的 AI 在 **2024**（LangGraph）

---

*下一章：小明的任务被分解了——Lead Agent 是如何工作的？*
