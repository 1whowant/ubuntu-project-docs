# DeerFlow Sandbox 与 Tools/MCP 机制解析

> 目标读者：懂一点技术但不是专业人员
> 写作视角：小明用户故事

---

## 第一部分：Sandbox 底层实现和应用

### 1. Sandbox 是什么

**Sandbox（沙箱）** 是 DeerFlow 给 AI 提供的一个"真实的电脑"环境。它是一个隔离的执行环境（通常是 Docker 容器），让 AI 能够真正地执行代码、操作文件、运行命令。

想象一下，如果 AI 只是一个聊天机器人，它只能"说"不能"做"。但有了 Sandbox，AI 就能：
- 读写文件
- 执行 Bash 命令
- 运行 Python、Node.js 代码
- 生成报告、图表、PPT 等产物

**【小明的例子】**

```
小明："用 Python 分析一下这个 CSV 文件"
    ↓
Agent 生成 Python 代码
    ↓
Sandbox 启动 Docker 容器
    ↓
在容器里执行 Python 代码
    ↓
返回分析结果和生成的图表
```

小明只需要说一句话，AI 就能在 Sandbox 里完成从代码编写到执行的全过程，最后把结果和图表呈现给他。

> 参考：docs/external/01_architecture_overview.md

---

### 2. Sandbox 的三种模式

DeerFlow 支持三种 Sandbox 运行模式，适应不同的使用场景：

| 模式 | 适用场景 | 隔离性 | 性能 |
|------|---------|--------|------|
| **Local** | 开发测试 | 低（直接执行） | 高 |
| **Docker** | 生产环境 | 高（容器隔离） | 中 |
| **Provisioner/K8s** | 大规模部署 | 高（Pod 隔离） | 中 |

**【小明的对比体验】**

| 模式 | 小明的使用场景 |
|------|---------------|
| **Local** | 小明在自己电脑上测试新功能，代码直接在本地运行，速度快但隔离性弱 |
| **Docker** | 公司正式使用，每个任务运行在独立容器里，安全但启动稍慢 |
| **K8s** | 大型企业部署，自动扩缩容，适合成百上千的并发任务 |

小明刚开始学习时用 Local 模式，自己开发测试很方便；公司正式环境用 Docker 模式，每个用户的任务都隔离运行；如果是大厂大规模使用，就用 K8s 模式自动管理资源。

> 参考：docs/external/02_claude_developer_guide.md

---

### 3. 虚拟路径映射

Sandbox 使用虚拟路径系统，让 Agent 看到的文件路径和实际宿主机路径对应起来：

| 容器内路径（Agent 看到） | 宿主机实际路径 |
|------------------------|---------------|
| `/mnt/user-data/workspace/` | `.deer-flow/threads/{id}/workspace/` |
| `/mnt/user-data/uploads/` | `.deer-flow/threads/{id}/uploads/` |
| `/mnt/user-data/outputs/` | `.deer-flow/threads/{id}/outputs/` |
| `/mnt/skills/` | `skills/` |

**【小明的代码示例】**

```python
# Agent 在 Sandbox 里执行这行代码
with open("/mnt/user-data/workspace/data.csv") as f:
    data = f.read()

# 实际上读取的是宿主机的：
# .deer-flow/threads/123/workspace/data.csv
```

小明上传了一个 CSV 文件，Agent 通过虚拟路径读取它，完全不用关心文件实际存在哪里。这种映射让 Agent 的代码可以在任何环境下运行。

> 参考：docs/external/01_architecture_overview.md

---

### 4. Sandbox 生命周期（重点回答 5 个问题）

#### 问题 1：什么情况下会用 Sandbox？

Sandbox 在以下场景会被使用：

| 场景 | 示例 |
|------|------|
| 执行 bash 命令 | `ls`, `cat`, `python3 script.py` |
| 读写文件 | `read_file`, `write_file`, `str_replace` |
| 运行代码 | Python、Node.js、Shell 脚本 |
| 生成产物 | PDF 报告、PPT、图片、数据文件 |

**【小明的任务流程】**

```
小明："搜索 AI 趋势并生成报告"
    ↓
Agent 调用 web_search 工具（❌ 不需要 Sandbox，纯网络请求）
    ↓
Agent 生成 Python 代码分析数据（✅ 需要 Sandbox，写入文件）
    ↓
Agent 执行 Python 生成图表（✅ 需要 Sandbox，运行代码）
    ↓
Agent 保存报告到 outputs/（✅ 需要 Sandbox，写入产物）
```

小明发现：不是所有操作都需要 Sandbox，只有涉及文件和代码执行时才需要。

> 参考：docs/external/01_architecture_overview.md

---

#### 问题 2：Sandbox 的管理机制是什么？

DeerFlow 使用 **Provider 模式** 管理 Sandbox：

```
Thread 开始时
    ↓
SandboxMiddleware 调用 SandboxProvider.acquire()
    ↓
如果已有 Sandbox 复用，否则创建新的
    ↓
Thread 结束时
    ↓
可选择释放或保留 Sandbox
```

**Provider 类型：**

| Provider | 说明 |
|----------|------|
| `LocalSandboxProvider` | 单例模式，直接在本机执行 |
| `AioSandboxProvider` | Docker 容器管理，隔离执行 |

**【小明的理解】**

Provider 就像一个"沙箱管理员"，负责：
1. 检查有没有可用的沙箱
2. 没有就新建一个
3. 对话结束决定是回收还是保留

> 参考：docs/external/01_architecture_overview.md, docs/external/zread_09_lead_agent_middleware.md

---

#### 问题 3：是一个对话一个 Sandbox 吗？

**是的！** DeerFlow 采用 **per-thread 隔离** 设计：

- 每个 thread（对话）有自己的 workspace/uploads/outputs
- 不同 thread 之间文件完全隔离，互不干扰
- 就像每个用户有自己的独立工作空间

**【小明的两个对话】**

```
Thread A（小明研究 AI）
    ├── workspace/ai_research/
    ├── uploads/参考论文.pdf
    └── outputs/ai_report.md

Thread B（小明写代码）
    ├── workspace/code_project/
    ├── uploads/需求文档.docx
    └── outputs/app.py

两个 Thread 完全隔离，文件互不干扰！
```

小明可以同时和 AI 聊两个完全不同的主题，不用担心文件混在一起。

> 参考：docs/external/01_architecture_overview.md

---

#### 问题 4：Sandbox 内部执行什么操作？

Sandbox 内部主要执行以下操作：

| 操作类型 | 具体工具/命令 |
|----------|--------------|
| 文件读写 | `read_file`, `write_file`, `str_replace` |
| Bash 命令 | `bash`, `ls`, `cat`, `mkdir` |
| 代码运行 | `python3 script.py`, `node app.js` |
| 产物生成 | 生成 PDF、PPT、图片等 |

**【小明的操作示例】**

```bash
# Agent 在 Sandbox 里执行的典型操作

# 1. 查看工作区文件
ls /mnt/user-data/workspace/

# 2. 读取上传的文件
cat /mnt/user-data/uploads/data.csv

# 3. 运行 Python 脚本
python3 /mnt/user-data/workspace/analysis.py

# 4. 写入产物
echo "# 报告" > /mnt/user-data/outputs/result.md
```

小明上传文件后，Agent 在 Sandbox 里像操作普通 Linux 系统一样处理这些文件。

> 参考：docs/external/01_architecture_overview.md

---

#### 问题 5：Sandbox 是否可复用？还是对话结束就销毁？

**默认复用，对话结束后可选择保留或清理：**

| 阶段 | 行为 |
|------|------|
| 对话进行中 | 同一个 thread 内，Sandbox 一直复用 |
| 对话结束 | thread 删除时，Sandbox 释放 |
| 文件处理 | 可选择保留或删除 thread 目录 |
| 配置策略 | 支持设置自动清理策略 |

**【小明的 10 轮对话】**

```
小明和 Agent 对话了 10 轮
    ↓
每轮都复用同一个 Sandbox（不用重复创建）
    ↓
文件在 workspace/ 中逐渐累积
    ↓
对话结束（删除 thread）
    ↓
Sandbox 容器停止运行
    ↓
文件保留在宿主机 .deer-flow/threads/{id}/ 目录
```

小明发现：复用 Sandbox 让对话更流畅，不用每次都等容器启动；对话结束后文件还在，可以回头查看或下载。

> 参考：docs/external/01_architecture_overview.md, docs/external/zread_09_lead_agent_middleware.md

---

## 第二部分：Tools 和 MCP

### 1. Tools 是什么

**Tools（工具）** 是 AI 可以调用的功能，让 AI 从"只会说话"变成"能动手做事"。

DeerFlow 的工具分为几类：
- **内置工具**：`bash`, `read_file`, `write_file`, `web_search` 等
- **配置工具**：在 `config.yaml` 中定义的工具
- **MCP 工具**：外部服务集成的工具
- **子代理工具**：`task` 工具，用于委托子代理

**【小明的工具清单】**

```
小明问："你能做什么？"
    ↓
Agent 回答："我可以：
    - 🔍 搜索网络（web_search）
    - 📖 读取文件（read_file）
    - ✏️ 写入文件（write_file）
    - 💻 执行命令（bash）
    - 🖼️ 查看图片（view_image）
    - 👥 调用子代理（task）
    - ..."
```

小明发现 AI 的能力取决于配置了哪些工具，工具越多，AI 能做的事越多。

> 参考：docs/external/02_claude_developer_guide.md

---

### 2. Tool 的组装流程

DeerFlow 启动时会组装所有可用工具：

```
系统启动
    ↓
get_available_tools() 组装工具列表
    ↓
1. 内置工具（bash, read_file, write_file, str_replace, ls）
2. 配置定义的工具（web_search, web_fetch 等）
3. MCP 工具（动态加载）
4. 子代理工具（task，如果启用）
    ↓
工具列表注入到 Agent 系统提示
```

**【小明的 web_search 工具】**

```
config.yaml 配置了 web_search 工具
    ↓
启动时读取配置
    ↓
创建 web_search_tool 实例
    ↓
添加到可用工具列表
    ↓
Agent 知道可以调用 web_search
```

小明在配置文件中添加了一个搜索工具，重启后 Agent 就能使用它了。

> 参考：docs/external/02_claude_developer_guide.md

---

### 3. MCP（Model Context Protocol）

**MCP 是什么？**

MCP 是 **Model Context Protocol** 的缩写，是一个开放标准协议，让 AI 可以连接各种外部服务（GitHub、数据库、搜索引擎等）。

**MCP 的作用：**
- 标准化外部工具的接口
- 让 AI 能调用各种第三方服务
- 工具可以动态加载，无需修改核心代码

**【小明添加 GitHub 工具】**

```
小明在设置页面添加 GitHub MCP
    ↓
extensions_config.json 更新：
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "enabled": true
    }
  }
}
    ↓
Gateway 重启 MCP 客户端
    ↓
Agent 可用 GitHub 工具（get_repo_info, search_issues 等）
    ↓
小明："分析这个 GitHub 项目"
    ↓
Agent 调用 github.get_repo_info("bytedance/deer-flow")
```

小明通过简单的配置，就让 AI 获得了操作 GitHub 的能力，可以查询仓库信息、搜索 issue 等。

> 参考：docs/external/07_mcp_server.md

---

### 4. MCP vs 内置工具对比

| 特性 | 内置工具 | MCP 工具 |
|------|---------|---------|
| **安装** | 随系统自带 | 需要配置 |
| **类型** | 文件操作、代码执行 | 外部服务（GitHub、数据库等） |
| **配置** | `config.yaml` | `extensions_config.json` |
| **加载** | 启动时加载 | 延迟加载（首次使用时） |
| **示例** | bash, read_file | github, postgres, brave-search |

**【小明的对比体验】**

| 场景 | 工具类型 | 小明的操作 |
|------|---------|-----------|
| "读取本地文件" | 内置工具 | 直接调用 `read_file`，开箱即用 |
| "查 GitHub 仓库" | MCP 工具 | 先配置 GitHub MCP，然后才能使用 |

小明发现：内置工具像手机自带的功能，MCP 工具像需要下载安装的 App。

> 参考：docs/external/02_claude_developer_guide.md, docs/external/07_mcp_server.md

---

### 5. Tool 调用流程

```
Agent 思考："我需要搜索信息"
    ↓
生成 tool_call：
{"name": "web_search", "args": {"query": "AI 趋势"}}
    ↓
系统执行工具
    ↓
Sandbox 中执行（如果需要）
    ↓
返回 tool_result
    ↓
Agent 基于结果继续思考
```

**【小明的搜索请求】**

```
小明："2026年 AI 趋势如何？"
    ↓
Agent 决定调用 web_search
    ↓
调用 Tavily API 搜索
    ↓
返回搜索结果
    ↓
Agent 分析结果
    ↓
生成回答
```

小明问了一个问题，AI 自动决定需要搜索，调用工具获取信息，然后基于搜索结果生成回答。

> 参考：docs/external/01_architecture_overview.md

---

## 第三部分：Sandbox + Tools 协同工作

### 完整例子：小明让 AI 研究并生成报告

```
小明："研究 2026 AI 趋势，生成 PDF 报告"
    ↓
【Step 1】Agent 调用 web_search 工具
    - 工具类型：MCP/配置工具
    - 是否需要 Sandbox：❌ 否（网络请求）
    - 返回：搜索结果
    ↓
【Step 2】Agent 分析数据，生成 Python 脚本
    - 写入文件：/mnt/user-data/workspace/analysis.py
    - 是否需要 Sandbox：✅ 是（文件写入）
    ↓
【Step 3】Agent 执行 Python 脚本
    - 调用 bash 工具：python3 workspace/analysis.py
    - 是否需要 Sandbox：✅ 是（代码执行）
    - 生成图表到 outputs/
    ↓
【Step 4】Agent 生成 PDF 报告
    - 调用 write_file：/mnt/user-data/outputs/report.pdf
    - 是否需要 Sandbox：✅ 是（文件写入）
    ↓
【Step 5】小明下载报告
    - 通过 Gateway /api/threads/{id}/artifacts 获取
```

**【流程解析】**

| 步骤 | 操作 | 使用工具 | 使用 Sandbox |
|------|------|---------|-------------|
| 1 | 搜索 AI 趋势 | web_search (MCP) | ❌ |
| 2 | 生成分析脚本 | write_file (内置) | ✅ |
| 3 | 执行脚本生成图表 | bash (内置) | ✅ |
| 4 | 生成 PDF 报告 | write_file (内置) | ✅ |
| 5 | 下载报告 | Gateway API | - |

小明只说了句话，AI 就完成了一整套工作：搜索信息 → 分析数据 → 生成图表 → 输出报告。这就是 Sandbox 和 Tools 协同工作的威力。

> 参考：docs/external/01_architecture_overview.md, docs/external/08_api_reference.md

---

## 总结

### 核心要点

1. **Sandbox 是 AI 的"真实电脑"**：提供隔离的执行环境，让 AI 能真正执行代码和操作文件

2. **per-thread 隔离**：每个对话有独立的 Sandbox 和文件空间，互不干扰

3. **Tools 是 AI 的能力扩展**：内置工具处理文件和代码，MCP 工具连接外部服务

4. **MCP 让扩展更简单**：通过标准协议接入 GitHub、数据库等第三方服务

5. **协同工作**：网络工具不需要 Sandbox，文件/代码操作需要 Sandbox

### 小明的一句话总结

> "Sandbox 给了 AI 一双手，Tools 给了 AI 各种工具，MCP 让 AI 能使用外部服务，三者配合让 AI 从'会说话'变成'能做事'。"

---

## 参考资料

- [架构概览](docs/external/01_architecture_overview.md)
- [开发指南](docs/external/02_claude_developer_guide.md)
- [MCP 服务器配置](docs/external/07_mcp_server.md)
- [API 参考](docs/external/08_api_reference.md)
- [Lead Agent 中间件](docs/external/zread_09_lead_agent_middleware.md)
- [开发者社区分析](docs/external/12_dev_community_analysis.md)
