# DeerFlow vs OpenClaw 详细技术对比分析（已验证版）

> 目标读者：技术人员（需要深入技术实现细节）  
> 分析时间：2026-03-30  
> **验证状态**: OpenClaw 内容已逐条验证（来源：docs.openclaw.ai）

---

## 第一部分：整体架构对比

### 1. 架构分层对比

| 维度 | DeerFlow | OpenClaw | 技术差异 |
|------|----------|----------|---------|
| 架构模式 | Harness/App 严格分层 | **Gateway + Agent + Skills + Nodes** | DeerFlow 分层更严格，OpenClaw 采用中心化 Gateway 架构 |
| 入口设计 | Nginx 统一入口(2026) | **Gateway 单一入口 (Port 18789)** | OpenClaw 统一 WebSocket + HTTP 端口 |
| 服务拆分 | 3服务(Nginx+LangGraph+Gateway+Frontend) | **单一 Gateway 进程** | OpenClaw 架构更集中 |
| 通信协议 | HTTP REST | **WebSocket + HTTP** | OpenClaw 使用 WebSocket 双向通信 |

### 1.2 OpenClaw Gateway 架构（✅ 已验证）

**官方文档来源**: docs.openclaw.ai/concepts/architecture

```
┌─────────────────────────────────────────┐
│              Gateway                     │
│  (WebSocket Control Plane + HTTP API)   │
│  Port: 18789 (default)                  │
└──────────────┬──────────────────────────┘
               │
    ┌──────────┼──────────┬──────────┐
    ↓          ↓          ↓          ↓
  Pi Agent   CLI      WebChat    macOS App
             Web UI   (浏览器)    iOS/Android Nodes
```

**核心设计原则**（官方引用）：
> "A single long‑lived **Gateway** owns all messaging surfaces... Control-plane clients connect to the Gateway over **WebSocket**"
> — Source: docs.openclaw.ai/concepts/architecture

**技术实现对比**:

| 特性 | DeerFlow | OpenClaw |
|------|----------|----------|
| 协议 | HTTP REST | **WebSocket (ws://127.0.0.1:18789)** |
| 首帧要求 | - | **必须发送 `connect`** |
| 消息格式 | JSON | **JSON over WebSocket** |
| 认证方式 | API Key | **Token 或 Password** |
| 默认端口 | 多端口 | **单一端口 18789** |

**OpenClaw HTTP API 端点**（已验证）：
- `GET /v1/models` - OpenAI 兼容
- `POST /v1/chat/completions` - OpenAI 兼容
- `POST /v1/responses` - OpenAI 兼容
- `POST /tools/invoke` - 工具调用
- `/__openclaw__/canvas/` - Canvas 主机
- `/__openclaw__/a2ui/` - A2UI 主机
- `/hooks` - Webhook 端点

> Source: docs.openclaw.ai/gateway

---

## 第二部分：Channel 渠道对比

### 2.1 支持的渠道（✅ 已验证）

**OpenClaw 官方渠道列表**（来源：docs.openclaw.ai/channels）：

**原生支持渠道**（20+ 种）：

| 渠道 | 技术实现 | 状态 |
|------|---------|------|
| WhatsApp | **Baileys** | ✅ 原生 |
| Telegram | **grammY Bot API** | ✅ 原生 |
| Discord | **discord.js** | ✅ 原生 |
| Slack | **Bolt SDK** | ✅ 原生 |
| Signal | **signal-cli** | ✅ 原生 |
| iMessage | **BlueBubbles / legacy** | ✅ 原生 |
| Feishu | **WebSocket** | ✅ 原生 |
| Google Chat | **HTTP webhook** | ✅ 原生 |
| IRC | **IRC protocol** | ✅ 原生 |
| LINE | **Messaging API** | ✅ Plugin |
| Matrix | **Matrix protocol** | ✅ Plugin |
| Mattermost | **Bot API + WebSocket** | ✅ Plugin |
| Microsoft Teams | **Bot Framework** | ✅ Plugin |
| Nextcloud Talk | **WebSocket** | ✅ Plugin |
| Nostr | **NIP-04** | ✅ Plugin |
| Synology Chat | **Webhooks** | ✅ Plugin |
| Tlon | **Urbit** | ✅ Plugin |
| Twitch | **IRC** | ✅ Plugin |
| WebChat | **WebSocket UI** | ✅ 内置 |
| WeChat | **Tencent iLink** | ✅ Plugin |
| Zalo | **Bot API** | ✅ Plugin |
| Zalo Personal | **QR login** | ✅ Plugin |
| Voice Call | **Plivo/Twilio** | ✅ Plugin |

**DeerFlow 渠道支持**（根据大纲）：
- 主打 Web 网页端
- 渠道支持有限
- 无独立客户端

**对比结论**:
| 维度 | DeerFlow | OpenClaw |
|------|----------|----------|
| 渠道数量 | 有限 | **20+ 种** |
| Web 端 | ✅ 主打 | ✅ Control UI + WebChat |
| 移动端 | ❌ | **✅ iOS/Android App** |
| 桌面端 | ❌ | **✅ macOS Menu Bar** |

---

## 第三部分：自动化能力对比

### 3.1 Cron 调度器（✅ 已验证）

**OpenClaw Cron 官方文档**（来源：docs.openclaw.ai/automation/cron-jobs）：

> "Cron is the Gateway's built-in scheduler. It persists jobs, wakes the agent at the right time, and can optionally deliver output back to a chat."

**任务类型**（已验证）：
| 类型 | 描述 | 配置 |
|------|------|------|
| `at` | 一次性定时任务 | `schedule.kind: "at"` |
| `every` | 固定间隔 | `schedule.kind: "every"` |
| `cron` | Cron 表达式 | `schedule.kind: "cron"` |

**执行模式**（已验证）：
| 模式 | Session Target | 用途 |
|------|---------------|------|
| Main session | `sessionTarget: "main"` | 下次心跳运行 |
| Isolated | `sessionTarget: "isolated"` | 独立会话运行 |
| Current session | `sessionTarget: "current"` | 绑定当前会话 |
| Custom session | `sessionTarget: "session:xxx"` | 持久命名会话 |

**存储位置**（已验证）：
> "Job store: `~/.openclaw/cron/jobs.json`"
> — Source: docs.openclaw.ai/automation/cron-jobs

**DeerFlow 自动化现状**（根据大纲）：
- ❌ 暂无定时任务
- ❌ 暂无 hook 触发器
- ❌ 被动式架构

### 3.2 Webhook 触发器（✅ 已验证）

**OpenClaw Webhook 官方文档**（来源：docs.openclaw.ai/automation/webhook）：

**端点**（已验证）：
| 端点 | 方法 | 功能 |
|------|------|------|
| `/hooks/wake` | POST | 触发系统事件 |
| `/hooks/agent` | POST | 运行独立 Agent |
| `/hooks/<name>` | POST | 自定义映射 |

**配置示例**（官方）：
```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
    mappings: [
      { match: { path: "gmail" }, action: "agent", deliver: true }
    ]
  }
}
```

### 3.3 Heartbeat 机制（✅ 已验证）

**官方文档**（来源：docs.openclaw.ai/gateway/heartbeat）：

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last"
      }
    }
  }
}
```

**对比结论**:
| 维度 | DeerFlow | OpenClaw |
|------|----------|----------|
| 定时任务 | ❌ 暂无 | **✅ 内置 Cron** |
| Webhook | ❌ 暂无 | **✅ 支持** |
| Heartbeat | ❌ 暂无 | **✅ 支持** |
| 触发器 | ❌ 被动式 | **✅ 主动式** |

---

## 第四部分：Agent / Sub-Agent 架构对比

### 4.1 Multi-Agent 支持（✅ 已验证）

**OpenClaw 官方文档**（来源：docs.openclaw.ai/concepts/multi-agent）：

> "An **agent** is a fully scoped brain with its own: **Workspace**, **State directory** (`agentDir`), **Session store**"

**隔离维度**（已验证）：
| 维度 | 路径 |
|------|------|
| Workspace | `~/.openclaw/workspace-<agentId>` |
| Agent Dir | `~/.openclaw/agents/<agentId>/agent` |
| Sessions | `~/.openclaw/agents/<agentId>/sessions` |
| Auth | `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` |

**配置示例**（官方）：
```json5
{
  agents: {
    list: [
      { id: "home", workspace: "~/.openclaw/workspace-home" },
      { id: "work", workspace: "~/.openclaw/workspace-work" }
    ]
  },
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } }
  ]
}
```

### 4.2 Sub-Agent 机制（✅ 已验证）

**OpenClaw 官方文档**（来源：docs.openclaw.ai/tools/subagents）：

> "Sub-agents are background agent runs spawned from an existing agent run. They run in their own session (`agent:<agentId>:subagent:<uuid>`)"

**嵌套深度**（已验证）：
| 深度 | Session Key | 权限 |
|------|-------------|------|
| 0 | `agent:<id>:main` | 可 spawn |
| 1 | `agent:<id>:subagent:<uuid>` | 可 spawn (maxSpawnDepth≥2) |
| 2 | `agent:<id>:subagent:<uuid>:subagent:<uuid>` | 不可 spawn |

**配置**（官方）：
```json5
{
  agents: {
    defaults: {
      subagents: {
        maxSpawnDepth: 2,
        maxChildrenPerAgent: 5,
        maxConcurrent: 8,
        runTimeoutSeconds: 900
      }
    }
  }
}
```

**工具**（已验证）：
- `sessions_spawn` - 启动子 Agent
- `subagents` - 管理子 Agent（list/kill/steer）
- `sessions_send` - 跨 Agent 消息

### 4.3 Session 管理（✅ 已验证）

**OpenClaw 官方文档**（来源：docs.openclaw.ai/concepts/session）：

**DM 隔离选项**（已验证）：
| 选项 | 描述 |
|------|------|
| `main` | 所有 DM 共享一个会话 |
| `per-peer` | 按发送者隔离 |
| `per-channel-peer` | 按渠道+发送者隔离（推荐） |
| `per-account-channel-peer` | 按账号+渠道+发送者隔离 |

**存储位置**（已验证）：
> "Store: `~/.openclaw/agents/<agentId>/sessions/sessions.json`"
> "Transcripts: `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`"
> — Source: docs.openclaw.ai/concepts/session

---

## 第五部分：Sandbox 详细技术对比

### 5.1 DeerFlow Sandbox 架构

```python
# DeerFlow 的 Sandbox 抽象层
class SandboxProvider(ABC):
    @abstractmethod
    def acquire(self, thread_id: str) -> Sandbox:
        """获取或创建沙箱"""
        pass
    
    @abstractmethod
    def release(self, thread_id: str) -> None:
        """释放沙箱资源"""
        pass
```

### 5.2 OpenClaw Sandbox（✅ 已验证）

**官方文档**（来源：docs.openclaw.ai/gateway/sandboxing）：

**沙箱模式**（已验证）：
| 模式 | 描述 |
|------|------|
| `off` | 无沙箱 |
| `non-main` | 仅非主会话使用沙箱（默认） |
| `all` | 所有会话使用沙箱 |

**作用域**（已验证）：
| 作用域 | 描述 |
|--------|------|
| `session` | 每个会话一个容器（默认） |
| `agent` | 每个 Agent 一个容器 |
| `shared` | 所有会话共享一个容器 |

**后端**（已验证）：
| 后端 | 描述 |
|------|------|
| `docker` | 本地 Docker（默认） |
| `ssh` | 远程 SSH |
| `openshell` | OpenShell 托管 |

**配置示例**（官方）：
```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        scope: "session",
        workspaceAccess: "none",
        backend: "docker"
      }
    }
  }
}
```

**工作区访问**（已验证）：
| 选项 | 描述 |
|------|------|
| `none` | 沙箱独立工作区 |
| `ro` | 只读挂载 Agent 工作区 |
| `rw` | 读写挂载 Agent 工作区 |

---

## 第六部分：Skills 系统对比

### 6.1 OpenClaw Skills（✅ 已验证）

**官方文档**（来源：docs.openclaw.ai/tools/skills）：

**加载优先级**（从高到低，已验证）：
```
1. <workspace>/skills/                    (最高)
2. <workspace>/.agents/skills/
3. ~/.agents/skills/
4. ~/.openclaw/skills/
5. Bundled skills (随安装包)
6. skills.load.extraDirs/Plugin skills     (最低)
```

**格式**（AgentSkills 兼容，已验证）：
```markdown
---
name: image-lab
description: Generate or edit images
metadata:
  {
    "openclaw": {
      "requires": { "bins": ["uv"], "env": ["GEMINI_API_KEY"] },
      "primaryEnv": "GEMINI_API_KEY"
    }
  }
---
```

### 6.2 ClawHub 技能市场（✅ 已验证）

**官方文档**（来源：docs.openclaw.ai/tools/skills）：

> "ClawHub is the public skills registry for OpenClaw. Browse at https://clawhub.com"

**CLI 命令**（已验证）：
```bash
openclaw skills install <skill-slug>
openclaw skills update --all
```

---

## 第七部分：Nodes 节点系统对比

### 7.1 OpenClaw Nodes（✅ 已验证）

**官方文档**（来源：docs.openclaw.ai/nodes）：

> "A **node** is a companion device (macOS/iOS/Android/headless) that connects to the Gateway **WebSocket** with `role: "node"`"

**支持的 Node 平台**（已验证）：
| 平台 | 功能 |
|------|------|
| iOS | Canvas、Camera、Location、Talk Mode、Voice Wake |
| Android | Canvas、Camera、Screen Recording、SMS、Notifications |
| macOS | Canvas、Camera、Screen Recording、System Commands |
| Headless | System Commands |

**Node 命令**（已验证）：
| 命令族 | 功能 |
|--------|------|
| `canvas.*` | Canvas 控制、导航、截图 |
| `camera.*` | 拍照、录像 |
| `screen.record` | 屏幕录制 |
| `location.get` | 获取位置 |
| `system.run` | 远程执行命令 |
| `notifications.*` | 通知管理 |

### 7.2 iOS App 功能（✅ 已验证）

**官方文档**（来源：docs.openclaw.ai/platforms/ios）：

| 功能 | 状态 |
|------|------|
| Canvas | ✅ |
| Screen Snapshot | ✅ |
| Camera Capture | ✅ |
| Location | ✅ |
| Talk Mode | ✅ |
| Voice Wake | ✅ |

---

## 第八部分：验证结论

### 8.1 OpenClaw 验证状态汇总

| 章节 | 验证状态 | 官方来源 |
|------|---------|---------|
| Gateway 架构 | ✅ 已验证 | docs.openclaw.ai/concepts/architecture |
| WebSocket 协议 | ✅ 已验证 | docs.openclaw.ai/concepts/architecture |
| HTTP API | ✅ 已验证 | docs.openclaw.ai/gateway |
| Channel 支持（20+） | ✅ 已验证 | docs.openclaw.ai/channels |
| Web 端（Control UI/WebChat） | ✅ 已验证 | docs.openclaw.ai/web |
| Cron 自动化 | ✅ 已验证 | docs.openclaw.ai/automation/cron-jobs |
| Webhook 触发器 | ✅ 已验证 | docs.openclaw.ai/automation/webhook |
| Heartbeat | ✅ 已验证 | docs.openclaw.ai/gateway/heartbeat |
| Multi-Agent | ✅ 已验证 | docs.openclaw.ai/concepts/multi-agent |
| Sub-Agent | ✅ 已验证 | docs.openclaw.ai/tools/subagents |
| Session 管理 | ✅ 已验证 | docs.openclaw.ai/concepts/session |
| Skills 系统 | ✅ 已验证 | docs.openclaw.ai/tools/skills |
| ClawHub | ✅ 已验证 | docs.openclaw.ai/tools/skills |
| 沙箱系统 | ✅ 已验证 | docs.openclaw.ai/gateway/sandboxing |
| Nodes 系统 | ✅ 已验证 | docs.openclaw.ai/nodes |
| iOS App | ✅ 已验证 | docs.openclaw.ai/platforms/ios |

### 8.2 需要修正的内容

| 原内容 | 修正 |
|--------|------|
| MCP 支持 | ⚠️ **未找到官方 MCP 文档**，仅发现 `openclaw mcp` CLI 命令 |
| Web 端描述 | OpenClaw 有 WebChat 和 Control UI，但需要 Gateway 运行 |

### 8.3 综合对比表

| 维度 | DeerFlow | OpenClaw |
|------|----------|----------|
| **架构** | Harness/App 分层 | **Gateway 中心化** |
| **协议** | HTTP REST | **WebSocket + HTTP** |
| **渠道** | 有限 | **20+ 种** |
| **Web 端** | ✅ 主打 | ✅ Control UI + WebChat |
| **移动端** | ❌ | **✅ iOS/Android** |
| **桌面端** | ❌ | **✅ macOS** |
| **自动化** | ❌ 被动式 | **✅ Cron + Webhook + Heartbeat** |
| **Multi-Agent** | 未知 | **✅ 原生支持** |
| **Sub-Agent** | 未知 | **✅ 支持嵌套** |
| **沙箱** | Provider 模式 | **Docker/SSH/OpenShell** |
| **Skills** | 未知 | **✅ ClawHub 生态** |
| **Nodes** | ❌ | **✅ iOS/Android/macOS** |

---

## 参考资料

### OpenClaw 官方文档（已验证）
1. docs.openclaw.ai/concepts/architecture - Gateway 架构
2. docs.openclaw.ai/gateway - Gateway 运行手册
3. docs.openclaw.ai/channels - 渠道支持
4. docs.openclaw.ai/automation/cron-jobs - Cron 自动化
5. docs.openclaw.ai/automation/webhook - Webhook
6. docs.openclaw.ai/concepts/multi-agent - Multi-Agent
7. docs.openclaw.ai/tools/subagents - Sub-Agent
8. docs.openclaw.ai/concepts/session - Session 管理
9. docs.openclaw.ai/tools/skills - Skills 系统
10. docs.openclaw.ai/gateway/sandboxing - 沙箱系统
11. docs.openclaw.ai/nodes - Nodes 系统
12. docs.openclaw.ai/platforms/ios - iOS App

### DeerFlow 资料
- 基于用户提供的大纲和内部文档

---

*文档生成时间：2026-03-30*  
*验证状态：OpenClaw 内容全部来自官方文档（docs.openclaw.ai）*  
*分析工具：Senior Tech Architect Engineer Agent*