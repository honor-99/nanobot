# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在本仓库中工作时提供指导。

## 项目概述

nanobot 是一个超轻量级个人 AI 助手框架（核心代码约 961 行）。支持 10+ 聊天平台（Telegram、Discord、WhatsApp、Slack、飞书、钉钉、Matrix、QQ、Email、Mochat），通过 LiteLLM 接入多种 LLM 提供商，支持 MCP 协议集成和技能/工具扩展系统。

## 常用开发命令

```bash
# 开发安装
pip install -e ".[dev]"

# 运行测试
pytest                              # 全部测试
pytest tests/test_foo.py            # 单个文件
pytest tests/test_foo.py::test_bar  # 单个测试

# 代码检查
ruff check nanobot/
ruff check --fix nanobot/           # 自动修复

# 运行 CLI
nanobot agent                       # 交互式聊天
nanobot gateway                     # 启动多平台网关
nanobot onboard                     # 初始化配置向导
```

## 架构

### 事件驱动消息流

```
Channel (Telegram/Discord/...) → InboundMessage → MessageBus → AgentLoop → LLM Provider
                                                                  ↕
                                                             ToolRegistry
                                                                  ↓
                                                           OutboundMessage → Channel
```

### 核心组件 (`nanobot/`)

- **`agent/loop.py`** — Agent 主循环：向 LLM 发送消息、处理工具调用、迭代直到完成
- **`agent/context.py`** — 系统提示词构建器：组装身份、引导文件、记忆、技能和运行时上下文
- **`agent/memory.py`** — 持久化记忆系统（MEMORY.md 存储事实，HISTORY.md 存储可搜索日志）
- **`agent/tools/`** — 内置工具（文件系统、Shell、Web、子 Agent、消息、定时任务、MCP 包装器）。均继承 `base.py` 中的 `Tool` 基类，通过 `ToolRegistry` 注册
- **`bus/`** — 事件驱动消息总线，包含 `InboundMessage`/`OutboundMessage` 事件类型和异步队列
- **`channels/`** — 聊天平台集成，均继承 `BaseChannel`。每个 Channel 负责将平台特定事件与总线消息格式互相转换
- **`providers/`** — LLM 提供商抽象层。`registry.py` 中的 `PROVIDERS` 元组是提供商元数据（环境变量、模型前缀、网关检测）的单一真实来源；`litellm_provider.py` 通过 LiteLLM 处理大多数提供商
- **`config/schema.py`** — 基于 Pydantic 的类型安全配置模型（`~/.nanobot/config.json`），同时支持 camelCase 和 snake_case 字段名
- **`skills/`** — Markdown 格式技能文件，带 YAML 前置元数据，由 `agent/skills.py` 加载
- **`session/`** — 每个会话的消息历史，JSONL 格式
- **`cron/`** — 基于 cron 表达式的定时任务执行
- **`cli/commands.py`** — 基于 Typer 的 CLI 入口

### 关键设计模式

- **插件注册表**：Channel、Provider、Tool 均使用注册表模式实现可扩展性
- **子 Agent 生成**：`SubagentManager` 运行独立的 Agent 实例执行后台任务
- **MCP 集成**：`MCPToolWrapper` 将 MCP 工具桥接为原生工具；支持 stdio 和 HTTP 传输
- **提示词缓存**：上下文构建器支持 Anthropic 风格的缓存控制块

## 代码风格

- Python ≥3.11，行长 100 字符
- Ruff 规则：E, F, I, N, W（忽略 E501）
- 全异步架构 — Channel 和 Agent 循环均使用 `asyncio`
- `pytest-asyncio` 配置 `asyncio_mode = "auto"`，异步测试函数无需额外装饰器
- 配置目录：`~/.nanobot/`；工作区文件：`~/.nanobot/workspace`

## 仓库结构

```
nanobot/        # Python 主包（源代码）
bridge/         # WhatsApp 桥接服务（Node.js/TypeScript，打包进 wheel）
tests/          # pytest 测试套件
case/           # 演示/示例配置
```
