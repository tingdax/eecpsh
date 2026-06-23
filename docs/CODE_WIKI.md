# Trae Agent — Code Wiki

> 项目仓库：<https://github.com/bytedance/trae-agent>
> 技术报告：[arXiv:2507.23370](https://arxiv.org/abs/2507.23370)
> 版本：0.1.0 · Python ≥ 3.12 · MIT License
> 维护方：ByteDance Ltd.

---

## 目录

1. [项目概览](#1-项目概览)
2. [整体架构](#2-整体架构)
3. [顶层目录结构](#3-顶层目录结构)
4. [核心模块职责](#4-核心模块职责)
   - 4.1 [trae_agent.agent — 智能体核心](#41-trae_agentagent--智能体核心)
   - 4.2 [trae_agent.tools — 工具生态](#42-trae_agenttools--工具生态)
   - 4.3 [trae_agent.utils.llm_clients — 多 LLM 适配层](#43-trae_agentutilsllm_clients--多-llm-适配层)
   - 4.4 [trae_agent.utils.cli — 控制台与交互界面](#44-trae_agentutilscli--控制台与交互界面)
   - 4.5 [trae_agent.utils — 配置、MCP、轨迹记录等基础设施](#45-trae_agentutils--配置mcp轨迹记录等基础设施)
   - 4.6 [trae_agent.prompt — 系统提示词](#46-trae_agentprompt--系统提示词)
5. [关键类与函数说明](#5-关键类与函数说明)
6. [依赖关系与第三方生态](#6-依赖关系与第三方生态)
7. [配置文件体系](#7-配置文件体系)
8. [执行主流程（ReAct Loop）](#8-执行主流程react-loop)
9. [Lakeview 子系统](#9-lakeview-子系统)
10. [MCP（Model Context Protocol）集成](#10-mcpmodel-context-protocol-集成)
11. [Docker 执行模式](#11-docker-执行模式)
12. [轨迹记录（Trajectory Recording）](#12-轨迹记录trajectory-recording)
13. [运行方式与常用命令](#13-运行方式与常用命令)
14. [测试体系](#14-测试体系)
15. [评估与基准（evaluation/）](#15-评估与基准evaluation)
16. [扩展路线图（roadmap）](#16-扩展路线图roadmap)
17. [常见问题排查](#17-常见问题排查)
18. [附录：关键文件清单](#18-附录关键文件清单)

---

## 1. 项目概览

**Trae Agent** 是一款基于 LLM 的通用软件工程任务智能体。它通过 CLI 向 LLM（大语言模型）下达自然语言任务，由 LLM 自主选择工具（文件编辑、Shell、序列化思考等）并循环执行，直至完成任务。

**项目定位**：以"研究友好"为核心，强调**模块化、透明、可扩展**，便于研究者进行消融实验、行为分析、能力扩展。

**核心特性**：
- 🌊 **Lakeview**：对每一步 agent 操作生成简短摘要
- 🤖 **多 LLM 支持**：OpenAI、Anthropic、Doubao、Azure、OpenRouter、Ollama、Google Gemini
- 🛠️ **丰富工具生态**：文件编辑、bash 执行、序列化思考、任务完成、MCP 工具
- 🎯 **交互模式**：交互式多轮对话
- 📊 **轨迹记录**：完整记录每一步 agent 行为，便于调试与分析
- ⚙️ **灵活配置**：YAML 为主，支持环境变量与命令行覆盖
- 🐳 **Docker 沙箱**：可在 Docker 容器内隔离执行（需 PyInstaller 工具）
- 🔌 **MCP 协议**：可接入任意兼容 MCP 的服务（如 Playwright）

---

## 2. 整体架构

Trae Agent 采用经典的 **Agent Loop（ReAct 风格）** 架构。整体可划分为五层：

```
┌────────────────────────────────────────────────────────────────────┐
│                          CLI 入口层 (cli.py)                        │
│  trae-cli run / interactive / show-config / tools                   │
└────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│                        Agent 调度层 (agent/agent.py)                │
│  Agent / AgentType · 装配 LLMClient、CLIConsole、TrajectoryRecorder │
└────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│                  智能体执行层 (agent/base_agent.py)                  │
│  BaseAgent / TraeAgent · ReAct 主循环、步进、反思、并行工具调用      │
└────────────────────────────────────────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  LLM 适配层      │  │   工具执行层      │  │  轨迹/反思层     │
│  llm_clients/    │  │   tools/         │  │  trajectory_     │
│  Anthropic       │  │  BashTool        │  │  recorder.py     │
│  OpenAI          │  │  TextEditorTool  │  │  lake_view.py    │
│  Google          │  │  JSONEditTool    │  │                  │
│  Azure · etc.    │  │  MCPTool · etc.  │  │                  │
└──────────────────┘  └──────────────────┘  └──────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│   基础设施层 (utils/) — 配置、CLI 控制台、MCP 客户端、Lakeview      │
└────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
                ┌───────────────────────────────┐
                │   可选：DockerToolExecutor    │
                │   隔离环境 + 工具二进制        │
                └───────────────────────────────┘
```

**数据流（单次 task）**：
1. CLI 解析参数 → `Config.create()` 加载 YAML → `Config.resolve_config_values()` 注入 CLI/env 覆盖项
2. `Agent` 装配 `LLMClient`、CLI 控制台、TrajectoryRecorder，按 agent_type 实例化 `TraeAgent`
3. `agent.new_task(task, extra_args)` 构造 system prompt + 初始 user message
4. `agent.execute_task()` 启动主循环（最多 max_steps 步）：
   - `_run_llm_step` → LLM 返回（含 tool_calls）
   - 若 LLM 调用 `task_done` → 完成；否则调用 `_tool_call_handler` 串行/并行执行工具
   - 反思（reflect_on_result）→ 把结果写回 messages
5. `execute_task` 结束 → 关闭工具 → 清理 Docker/MCP → 落盘轨迹

---

## 3. 顶层目录结构

```
trae-agent/
├── trae_agent/                  # 核心 Python 包
│   ├── __init__.py
│   ├── cli.py                   # Click 入口：run / interactive / show-config / tools
│   ├── agent/                   # 智能体核心
│   │   ├── agent.py             #   Agent 工厂（AgentType 枚举）
│   │   ├── agent_basics.py      #   AgentStep / AgentExecution / AgentError
│   │   ├── base_agent.py        #   BaseAgent（ReAct 循环）
│   │   ├── trae_agent.py        #   TraeAgent（软件工程特化）
│   │   └── docker_manager.py    #   Docker 容器生命周期
│   ├── tools/                   # 工具生态
│   │   ├── base.py              #   Tool / ToolExecutor / ToolResult
│   │   ├── bash_tool.py         #   持久化 bash 会话
│   │   ├── edit_tool.py         #   str_replace 编辑工具
│   │   ├── edit_tool_cli.py     #   独立 CLI（用于 Docker 容器内）
│   │   ├── json_edit_tool.py    #   JSONPath 编辑工具
│   │   ├── json_edit_tool_cli.py
│   │   ├── sequential_thinking_tool.py
│   │   ├── task_done_tool.py
│   │   ├── mcp_tool.py
│   │   ├── ckg_tool.py          #   代码知识图谱工具
│   │   ├── ckg/                 #   CKG 后端
│   │   ├── docker_tool_executor.py
│   │   ├── run.py               #   公共 run/maybe_truncate
│   │   └── __init__.py          #   tools_registry
│   ├── prompt/
│   │   └── agent_prompt.py      #   TRAE_AGENT_SYSTEM_PROMPT
│   └── utils/                   # 基础设施
│       ├── config.py            #   Config / AgentConfig / ModelConfig
│       ├── legacy_config.py     #   JSON 配置兼容层
│       ├── constants.py
│       ├── trajectory_recorder.py
│       ├── lake_view.py
│       ├── mcp_client.py
│       ├── cli/                 #   CLI 控制台
│       │   ├── cli_console.py
│       │   ├── console_factory.py
│       │   ├── simple_console.py
│       │   ├── rich_console.py
│       │   ├── rich_console.tcss
│       │   └── cli_console.py
│       └── llm_clients/         #   多 LLM 客户端
│           ├── base_client.py
│           ├── llm_client.py
│           ├── llm_basics.py
│           ├── retry_utils.py
│           ├── anthropic_client.py
│           ├── openai_client.py
│           ├── google_client.py
│           ├── azure_client.py
│           ├── openrouter_client.py
│           ├── ollama_client.py
│           ├── doubao_client.py
│           └── openai_compatible_base.py
├── tests/                       # 单元测试
├── evaluation/                  # 评测脚本与 SWE-bench 集成
├── docs/                        # 项目文档
│   ├── roadmap.md
│   ├── tools.md
│   ├── TRAJECTORY_RECORDING.md
│   └── legacy_config.md
├── server/Readme.md             # 预留服务化文档
├── trae_config.yaml.example     # 配置示例
├── trae_config.json.example
├── pyproject.toml               # 项目元数据与依赖
├── Makefile                     # uv + pytest + pre-commit 快捷目标
└── README.md
```

---

## 4. 核心模块职责

### 4.1 `trae_agent.agent` — 智能体核心

| 文件 | 关键类/函数 | 职责 |
|---|---|---|
| [agent.py](file:///workspace/trae-agent-repo/trae_agent/agent/agent.py) | `Agent`、`AgentType` | 工厂入口；根据 `agent_type` 字符串分发到 `TraeAgent`；装配 `LLMClient`、`TrajectoryRecorder`、`CLIConsole`；对外暴露 `run()` |
| [base_agent.py](file:///workspace/trae-agent-repo/trae_agent/agent/base_agent.py) | `BaseAgent(ABC)` | 通用 ReAct 主循环：`execute_task` → `_run_llm_step` → `_tool_call_handler`；维护 message history、token 累计、MCP 清理 |
| [trae_agent.py](file:///workspace/trae-agent-repo/trae_agent/agent/trae_agent.py) | `TraeAgent` | 软件工程特化：定义默认工具集、初始 system prompt、检查 `task_done` 工具调用、`must_patch` 校验（`remove_patches_to_tests`）、MCP 工具发现 |
| [agent_basics.py](file:///workspace/trae-agent-repo/trae_agent/agent/agent_basics.py) | `AgentStep`、`AgentExecution`、`AgentStepState`、`AgentState`、`AgentError` | 数据类与状态机；`AgentStepState` ∈ {THINKING, CALLING_TOOL, REFLECTING, COMPLETED, ERROR} |
| [docker_manager.py](file:///workspace/trae-agent-repo/trae_agent/agent/docker_manager.py) | `DockerManager` | 容器生命周期：从 image / dockerfile / tar / container_id 启动、复制工具、附加 shell、停止 |

**关键类签名**：

```python
# agent.py
class Agent:
    def __init__(
        self,
        agent_type: AgentType | str,
        config: Config,
        trajectory_file: str | None = None,
        cli_console: CLIConsole | None = None,
        docker_config: dict | None = None,
        docker_keep: bool = True,
    ): ...

    async def run(self, task: str, extra_args: dict[str, str] | None = None,
                  tool_names: list[str] | None = None) -> AgentExecution: ...

# base_agent.py
class BaseAgent(ABC):
    async def execute_task(self) -> AgentExecution: ...
    def llm_indicates_task_completed(self, llm_response: LLMResponse) -> bool: ...
    def _is_task_completed(self, llm_response: LLMResponse) -> bool: ...
    def task_incomplete_message(self) -> str: ...
    def reflect_on_result(self, tool_results: list[ToolResult]) -> str | None: ...
    @abstractmethod
    async def cleanup_mcp_clients(self) -> None: ...

# trae_agent.py
class TraeAgent(BaseAgent):
    TraeAgentToolNames = [
        "str_replace_based_edit_tool", "sequentialthinking",
        "json_edit_tool", "task_done", "bash",
    ]
    async def initialise_mcp(self): ...
    async def discover_mcp_tools(self): ...
    def get_git_diff(self) -> str: ...
    def remove_patches_to_tests(self, model_patch: str) -> str: ...
```

### 4.2 `trae_agent.tools` — 工具生态

| 工具名（注册键） | 类 | 功能 |
|---|---|---|
| `bash` | `BashTool` | 持久化 bash 会话；120s 超时；`restart=true` 重置；`Sentinel` 模式解析 `$?` |
| `str_replace_based_edit_tool` | `TextEditorTool` | view / create / str_replace / insert 四种子命令；`old_str` 必须唯一 |
| `json_edit_tool` | `JSONEditTool` | JSONPath 语法：view / set / add / remove；`jsonpath-ng` 实现 |
| `sequentialthinking` | `SequentialThinkingTool` | 序列化思考：thought_number / total_thoughts / revises_thought / branch_from_thought |
| `task_done` | `TaskDoneTool` | 无参；信号任务完成 |
| `ckg` | `CKGTool` | 代码知识图谱查询 |
| MCP 工具（动态） | `MCPTool` | 透传 MCP 服务发现的 tool schemas |

**Tool 基类** ([base.py](file:///workspace/trae-agent-repo/trae_agent/tools/base.py))：

```python
class Tool(ABC):
    @abstractmethod
    def get_name(self) -> str: ...
    @abstractmethod
    def get_description(self) -> str: ...
    @abstractmethod
    def get_parameters(self) -> list[ToolParameter]: ...
    @abstractmethod
    async def execute(self, arguments: ToolCallArguments) -> ToolExecResult: ...
    def get_input_schema(self) -> dict: ...  # OpenAI strict mode 兼容

class ToolExecutor:
    async def parallel_tool_call(self, calls: list[ToolCall]) -> list[ToolResult]: ...
    async def sequential_tool_call(self, calls: list[ToolCall]) -> list[ToolResult]: ...
    async def execute_tool_call(self, call: ToolCall) -> ToolResult: ...
```

**注册表**（[\_\_init\_\_.py](file:///workspace/trae-agent-repo/trae_agent/tools/__init__.py)）：

```python
tools_registry: dict[str, type[Tool]] = {
    "bash": BashTool,
    "str_replace_based_edit_tool": TextEditorTool,
    "json_edit_tool": JSONEditTool,
    "sequentialthinking": SequentialThinkingTool,
    "task_done": TaskDoneTool,
    "ckg": CKGTool,
}
```

### 4.3 `trae_agent.utils.llm_clients` — 多 LLM 适配层

**统一抽象**：

```python
# llm_basics.py
@dataclass
class LLMMessage:      role, content, tool_call, tool_result
@dataclass
class LLMResponse:     content, usage, model, finish_reason, tool_calls
@dataclass
class LLMUsage:        input_tokens, output_tokens, cache_*, reasoning_tokens

# base_client.py
class BaseLLMClient(ABC):
    def set_chat_history(self, messages): ...
    def chat(self, messages, model_config, tools=None, reuse_history=True) -> LLMResponse: ...
    def set_trajectory_recorder(self, recorder): ...
```

**`LLMClient` 工厂**（[llm_client.py](file:///workspace/trae-agent-repo/trae_agent/utils/llm_clients/llm_client.py)）根据 `LLMProvider` 枚举分发：

| 枚举 | 适配器 | 备注 |
|---|---|---|
| `OPENAI` | `OpenAIClient` | 使用 `openai.OpenAI` / `responses.create`；支持 `o3` / `o4-mini` / `gpt-5` 特殊 temperature 行为 |
| `ANTHROPIC` | `AnthropicClient` | 对 `str_replace_based_edit_tool` 与 `bash` 走原生 `text_editor_20250429` / `bash_20250124` 工具类型 |
| `GOOGLE` | `GoogleClient` | `google-genai` SDK；`function_declarations` |
| `AZURE` | `AzureClient` | 兼容 `max_completion_tokens`（gpt-5 / o3 / o4-mini） |
| `OPENROUTER` | `OpenRouterClient` | OpenAI 兼容协议，自定义 base_url |
| `DOUBAO` | `DoubaoClient` | 火山方舟 OpenAI 兼容 |
| `OLLAMA` | `OllamaClient` | 本地模型；同样 OpenAI 兼容协议 |

**重试机制**（[retry_utils.py](file:///workspace/trae-agent-repo/trae_agent/utils/llm_clients/retry_utils.py)）：

```python
def retry_with(func, provider_name="OpenAI", max_retries=3):
    """对 _create_*_response 类方法做 3-30s 随机退避重试"""
```

**OpenAI strict 模式**：当 `model_provider == "openai"` 时，`Tool.get_input_schema()` 会把所有参数标为 required（nullable 化）、object 追加 `additionalProperties: false`、顶层同样加 `additionalProperties: false`。

### 4.4 `trae_agent.utils.cli` — 控制台与交互界面

| 类 | 模式 | 用途 |
|---|---|---|
| `SimpleCLIConsole` | RUN | 基于 rich 的 print 表格输出 |
| `RichCLIConsole` + `RichConsoleApp` | INTERACTIVE | 基于 `textual` 的 TUI（实时 token、滚动日志、输入建议） |
| `ConsoleFactory` | — | 根据 mode/type 创建实例；INTERACTIVE 默认 RICH，RUN 默认 SIMPLE |

抽象基类 `CLIConsole` 在 [cli_console.py](file:///workspace/trae-agent-repo/trae_agent/utils/cli/cli_console.py) 中定义：
- `start()` / `stop()`
- `update_status(step, execution)`
- `print_task_details(details)` / `print(msg)`
- `get_task_input()` / `get_working_dir_input()`（仅交互模式）
- `set_lakeview(config)`

`AGENT_STATE_INFO` 字典把 `AgentStepState` 映射到 `(rich 颜色, emoji)`。

### 4.5 `trae_agent.utils` — 配置、MCP、轨迹记录等基础设施

#### 4.5.1 配置系统（[config.py](file:///workspace/trae-agent-repo/trae_agent/utils/config.py)）

核心数据类：

```python
@dataclass ModelProvider:   api_key, provider, base_url, api_version
@dataclass ModelConfig:     model, model_provider, temperature, top_p, top_k,
                            parallel_tool_calls, max_retries, max_tokens,
                            max_completion_tokens, supports_tool_calling,
                            candidate_count, stop_sequences
@dataclass MCPServerConfig: command/args/env/cwd, url, http_url/headers, tcp,
                            timeout, trust, description
@dataclass AgentConfig:     allow_mcp_servers, mcp_servers_config,
                            max_steps, model, tools
@dataclass TraeAgentConfig: enable_lakeview, tools
@dataclass LakeviewConfig:  model
@dataclass Config:          lakeview, model_providers, models, trae_agent
```

**优先级链**：`CLI 参数 > 配置文件 > 环境变量 > 默认值`。
**`Config.create(config_file=...)`**：YAML 解析（`.json` 后缀自动走 LegacyConfig）。

**`resolve_config_value` 辅助函数**：按 `cli > config > env` 解析单值。

#### 4.5.2 MCP 客户端（[mcp_client.py](file:///workspace/trae-agent-repo/trae_agent/utils/mcp_client.py)）

`MCPClient` 封装 `mcp.client.stdio.stdio_client` 与 `ClientSession`，通过 `AsyncExitStack` 管理异步资源；当前仅 stdio 传输已实现，HTTP/WebSocket 显式抛 `NotImplementedError`。

#### 4.5.3 轨迹记录（[trajectory_recorder.py](file:///workspace/trae-agent-repo/trae_agent/utils/trajectory_recorder.py)）

`TrajectoryRecorder` 负责把任务执行中所有 LLM 交互与 agent steps 写入 JSON 文件：
- 自动命名：`trajectories/trajectory_YYYYMMDD_HHMMSS.json`
- 公开方法：`start_recording` / `record_llm_interaction` / `record_agent_step` / `finalize_recording` / `save_trajectory`

### 4.6 `trae_agent.prompt` — 系统提示词

`TRAE_AGENT_SYSTEM_PROMPT`（[agent_prompt.py](file:///workspace/trae-agent-repo/trae_agent/prompt/agent_prompt.py)）是 7 步的 SWE 排错流程：
1. 理解问题
2. 探索代码
3. **复现 bug**（关键步骤）
4. 调试与定位根因
5. 实施修复
6. 严格验证（复现脚本 + 既有测试 + 新增测试 + 边界）
7. 总结

末尾给出 `sequential_thinking` 使用指南，并强调用 `task_done` 结束。

---

## 5. 关键类与函数说明

### 5.1 `Agent`（工厂类）

```python
class AgentType(Enum):
    TraeAgent = "trae_agent"

class Agent:
    async def run(self, task: str, extra_args: dict[str, str] | None = None,
                  tool_names: list[str] | None = None) -> AgentExecution
```
- `extra_args` 必须包含 `project_path` 与 `issue`；`must_patch` / `patch_path` / `base_commit` 可选。
- `tool_names` 为 `None` 时按 agent_type 默认工具集加载。
- 内部按 `agent_type` 分发到具体 agent 类（`match` 模式匹配）。
- 启动时根据 `config.trae_agent.enable_lakeview` 决定是否初始化 Lakeview。

### 5.2 `BaseAgent.execute_task`

主循环伪代码：
```python
for step_number in range(1, max_steps + 1):
    step = AgentStep(step_number, state=THINKING)
    try:
        messages = await self._run_llm_step(step, messages, execution)
        await self._finalize_step(step, messages, execution)
        if execution.agent_state == COMPLETED: break
    except Exception as e:
        step.state = ERROR; step.error = str(e)
        break
finally:
    if docker_manager and not docker_keep: docker_manager.stop()
    await self._close_tools()
```

`_run_llm_step` 内部判断：
- LLM 声明完成 → 检查 `_is_task_completed`（TraeAgent 还会校验 `must_patch` 非空且去除 test 目录 diff）
- 否则 → `_tool_call_handler` 串行/并行执行（受 `model_config.parallel_tool_calls` 控制）

### 5.3 `Tool.get_input_schema` 与 OpenAI strict 模式

为了 OpenAI 严格函数调用：所有参数 `required=True`，`type` 变 `[<type>, "null"]`，object 加 `additionalProperties: false`。

### 5.4 `TraeAgent.remove_patches_to_tests`

源自 Aider-AI 的 SWE-bench 辅助函数：从 model_patch 中剔除所有 `diff --git a/...` 目标位于 `test/ tests/ testing/ test_* tox.ini` 的 hunk，确保只评估"非测试"代码改动。

### 5.5 `BashTool` 内部 `_BashSession`

通过 `asyncio.create_subprocess_shell` 启动常驻 `/bin/bash`（Windows 走 `cmd.exe /v:on`），用 sentinel banner `,,,,bash-command-exit-__ERROR_CODE__-banner,,,,` 解析退出码；流式 stdout/stderr 通过 `_buffer` 直接读取（避免阻塞 EOF）。

### 5.6 `Config.create` 与 `Config.resolve_config_values`

`create` 是 classmethod，仅解析 YAML；`resolve_config_values` 才是把 CLI/env 实际覆写到 dataclass 上：
```python
config = Config.create(config_file="trae_config.yaml") \
            .resolve_config_values(provider="anthropic", model="...", api_key="...")
```

### 5.7 `ConsoleFactory.create_console`

```python
@staticmethod
def create_console(console_type: ConsoleType, mode: ConsoleMode, lakeview_config) -> CLIConsole
@staticmethod
def get_recommended_console_type(mode: ConsoleMode) -> ConsoleType
    # INTERACTIVE -> RICH, RUN -> SIMPLE
```

### 5.8 `LakeView.extract_task_in_step` / `extract_tag_in_step`

两个独立的 LLM 调用：
1. 用 `EXTRACTOR_PROMPT` 抽取 `<task>...</task><details>...</details>` 摘要
2. 用 `TAGGER_PROMPT` 标 `<tags>WRITE_TEST,EXAMINE_CODE,...</tags>`

最多重试 10 次（解析失败时）。

---

## 6. 依赖关系与第三方生态

来自 [pyproject.toml](file:///workspace/trae-agent-repo/pyproject.toml)：

| 依赖 | 用途 |
|---|---|
| `openai>=1.86.0` | OpenAI 官方 SDK（含 Responses API） |
| `anthropic>=0.54.0,<=0.60.0` | Claude API |
| `google-genai>=1.24.0` | Google Gemini |
| `ollama>=0.5.1` | 本地 Ollama |
| `mcp==1.12.2` | Model Context Protocol |
| `click>=8.0.0` + `asyncclick>=8.0.0` | CLI 框架 |
| `rich>=13.0.0` + `textual>=0.50.0` | TUI 输出 |
| `pydantic>=2.0.0` | 数据建模（间接） |
| `pyyaml>=6.0.2` | 配置解析 |
| `python-dotenv>=1.0.0` | `.env` 加载 |
| `jsonpath-ng>=1.7.0` | JSON 编辑工具 |
| `tree-sitter==0.21.3` + `tree-sitter-languages==1.10.2` | CKG 代码图谱 |
| `socksio>=1.0.0` | SOCKS 代理 |
| `pyinstaller==6.15.0` | 把 `edit_tool_cli.py` / `json_edit_tool_cli.py` 打包进 Docker 容器 |
| `typing-extensions>=4.0.0` | 类型注解 |
| `ruff>=0.12.4` | 静态检查与格式化 |

**optional-dependencies**：
- `test`：`pytest` / `pytest-asyncio` / `pytest-mock` / `pytest-cov` / `pre-commit`
- `evaluation`：`datasets` / `docker` / `pexpect` / `unidiff`

**dev-dependency**：`types-pyyaml` 仅供类型检查。

---

## 7. 配置文件体系

### 7.1 YAML 主配置（推荐）

参考 [trae_config.yaml.example](file:///workspace/trae-agent-repo/trae_config.yaml.example)：

```yaml
agents:
  trae_agent:
    enable_lakeview: true
    model: trae_agent_model
    max_steps: 200
    tools:
      - bash
      - str_replace_based_edit_tool
      - sequentialthinking
      - task_done

model_providers:
  anthropic:
    api_key: your_anthropic_api_key
    provider: anthropic
  openai:
    api_key: your_openai_api_key
    provider: openai
    base_url: https://api.openai.com/v1  # 可选

models:
  trae_agent_model:
    model_provider: anthropic
    model: claude-sonnet-4-20250514
    max_tokens: 4096
    temperature: 0.5

mcp_servers:
  playwright:
    command: npx
    args: ["@playwright/mcp@0.0.27"]
```

### 7.2 环境变量

```
OPENAI_API_KEY / OPENAI_BASE_URL
ANTHROPIC_API_KEY / ANTHROPIC_BASE_URL
GOOGLE_API_KEY / GOOGLE_BASE_URL
OPENROUTER_API_KEY / OPENROUTER_BASE_URL
DOUBAO_API_KEY / DOUBAO_BASE_URL
TRAE_CONFIG_FILE           # 自定义配置文件路径
```

### 7.3 旧版 JSON 配置

详见 [docs/legacy_config.md](file:///workspace/trae-agent-repo/docs/legacy_config.md)。CLI 入口 `Config.create()` 检测 `.json` 后缀自动调用 `create_from_legacy_config`。

---

## 8. 执行主流程（ReAct Loop）

```
        ┌──────────────────────────────────────────────┐
        │  CLI: trae-cli run "<task>"                  │
        │  → Config.create(...).resolve_config_values  │
        └──────────────────┬───────────────────────────┘
                           ▼
        ┌──────────────────────────────────────────────┐
        │  Agent.run(task, extra_args)                 │
        │   - new_task: 构造 sys + user 消息           │
        │   - initialise_mcp （如开启）                │
        │   - agent.execute_task()                     │
        └──────────────────┬───────────────────────────┘
                           ▼
   ┌───────────────────────────────────────────────────────┐
   │  BaseAgent.execute_task 主循环（max_steps 步）        │
   │   while step <= max_steps:                            │
   │     step.state = THINKING                             │
   │     llm_resp = LLMClient.chat(messages, tools)        │
   │     if llm_indicates_task_completed:                  │
   │        if _is_task_completed: COMPLETED, return        │
   │        else: push task_incomplete_message              │
   │     else:                                             │
   │        step.state = CALLING_TOOL                      │
   │        tool_results = ToolExecutor.parallel/sequential│
   │        if any failed: reflect_on_result → REFLECTING   │
   │        push tool results into messages                │
   │   finally: close_tools, stop docker, cleanup_mcp      │
   └───────────────────────────────────────────────────────┘
                           ▼
        ┌──────────────────────────────────────────────┐
        │  TraeAgent.execute_task（覆写）               │
        │   finalize_recording (TrajectoryRecorder)    │
        │   if patch_path: write git diff to file      │
        └──────────────────────────────────────────────┘
```

---

## 9. Lakeview 子系统

源码：[trae_agent/utils/lake_view.py](file:///workspace/trae-agent-repo/trae_agent/utils/lake_view.py)

**目的**：在控制台为每步 agent 操作生成 ≤10 词的任务摘要 + ≤30 词的细节，并把当前步骤打上 8 类标签（WRITE_TEST / VERIFY_TEST / EXAMINE_CODE / WRITE_FIX / VERIFY_FIX / REPORT / THINK / OUTLIER）。

**主流程**：
1. `_agent_step_str` 提取当前 step 的文本（content + tool_calls 描述）
2. `extract_task_in_step(prev, this)` 调 LLM 解析 `<task>` 与 `<details>`
3. `extract_tag_in_step(this)` 调 LLM 打标签，10 次重试
4. 返回 `LakeViewStep(desc_task, desc_details, tags_emoji)`

**启用方式**：`agents.trae_agent.enable_lakeview: true` 且 `lakeview.model` 引用一个 `models.*`。

**注意**：当 `extract_tag_in_step` 中 steps_fmt 长度 > 300 000 时跳过打标签，避免长上下文。

---

## 10. MCP（Model Context Protocol）集成

源码：[trae_agent/utils/mcp_client.py](file:///workspace/trae-agent-repo/trae_agent/utils/mcp_client.py)、[trae_agent/agent/trae_agent.py](file:///workspace/trae-agent-repo/trae_agent/agent/trae_agent.py) 的 `discover_mcp_tools`。

**配置**：
```yaml
mcp_servers:
  playwright:
    command: npx
    args: ["@playwright/mcp@0.0.27"]
allow_mcp_servers:
  - playwright
```

**执行流**：
1. `TraeAgent.initialise_mcp` → `discover_mcp_tools`
2. 对每个 `allow_mcp_servers` 中的 server：`MCPClient.connect_and_discover` → 启动 stdio → `session.list_tools` → 包装为 `MCPTool` 加入 `_tools`
3. `cleanup_mcp_clients` 在 task 结束时统一关闭

**已实现**：stdio 传输（`mcp.client.stdio.stdio_client`）。
**未实现**（显式抛 NotImplementedError）：HTTP / WebSocket 传输。

---

## 11. Docker 执行模式

源码：[trae_agent/agent/docker_manager.py](file:///workspace/trae-agent-repo/trae_agent/agent/docker_manager.py)、[trae_agent/tools/docker_tool_executor.py](file:///workspace/trae-agent-repo/trae_agent/tools/docker_tool_executor.py)、[trae_agent/agent/base_agent.py](file:///workspace/trae-agent-repo/trae_agent/agent/base_agent.py) 的 `__init__`。

**触发条件**（CLI）：以下参数互斥
- `--docker-image <name>`：直接 run 镜像
- `--docker-container-id <id>`：附加到既有容器
- `--dockerfile-path <abs path>`：构建镜像
- `--docker-image-file <tar>`：载入 tar 镜像
- `--working-dir`：宿主机挂载到容器 `/workspace`
- `--docker-keep=false`：任务结束移除容器

**前置条件**：
- `docker` CLI 可用 + daemon 在跑（`check_docker()` 在 CLI 入口校验）
- `trae_agent/dist/` 含 `edit_tool` 与 `json_edit_tool` 可执行（PyInstaller 产物）。首次使用 CLI 会自动调用 `build_with_pyinstaller` 编译

**工具隔离**：`DockerToolExecutor` 包装原生 `ToolExecutor`；当工具名 ∈ {`bash`, `str_replace_based_edit_tool`, `json_edit_tool`} 时改为在容器内执行对应 CLI 二进制。

**挂载语义**：宿主机 `--working-dir` ↔ 容器 `/workspace`，调用工具时自动做路径翻译。

---

## 12. 轨迹记录（Trajectory Recording）

详见 [docs/TRAJECTORY_RECORDING.md](file:///workspace/trae-agent-repo/docs/TRAJECTORY_RECORDING.md)。

**关键路径**：
- 自动路径：`trajectories/trajectory_YYYYMMDD_HHMMSS.json`
- 自定义：`--trajectory-file path.json`

**JSON 结构顶层字段**：
```
task, start_time, end_time, provider, model, max_steps,
llm_interactions[], agent_steps[], success, final_result, execution_time
```

**LLM 交互元素字段**：`timestamp, provider, model, input_messages, response{content,model,finish_reason,usage,tool_calls}, tools_available`

**Agent 步骤元素字段**：`step_number, timestamp, state, llm_messages, llm_response, tool_calls, tool_results, reflection, error`

**写入时机**：每一步后立即 `save_trajectory`（非任务结束才落盘）。

**安全注意**：文件**不记录 API Key**，但仍可能含项目源码；建议加入 `.gitignore`。

---

## 13. 运行方式与常用命令

### 13.1 安装

```bash
git clone https://github.com/bytedance/trae-agent.git
cd trae-agent
uv sync --all-extras
source .venv/bin/activate
cp trae_config.yaml.example trae_config.yaml
# 编辑 trae_config.yaml 填入 API Key
```

### 13.2 基本命令

```bash
trae-cli run "Create a hello world Python script"
trae-cli interactive
trae-cli show-config
trae-cli tools
```

### 13.3 模型指定

```bash
trae-cli run "..." --provider openai --model gpt-4o
trae-cli run "..." --provider anthropic --model claude-sonnet-4-20250514
trae-cli run "..." --provider google --model gemini-2.5-flash
trae-cli run "..." --provider openrouter --model "anthropic/claude-3-5-sonnet"
trae-cli run "..." --provider doubao --model doubao-seed-1.6
trae-cli run "..." --provider ollama --model qwen3
```

### 13.4 高级选项

| 选项 | 说明 |
|---|---|
| `--working-dir` / `-w` | 切换工作目录 |
| `--trajectory-file` / `-t` | 指定轨迹文件 |
| `--must-patch` / `-mp` | 强制要求产出 patch，否则不结束 |
| `--config-file` | 自定义配置（也可 `TRAE_CONFIG_FILE` env） |
| `--api-key` / `-k` | 临时 API Key |
| `--max-steps` | 覆盖 max_steps |
| `--model-base-url` | 自定义 base URL |
| `--console-type` / `-ct` | `simple` / `rich` |
| `--agent-type` / `-at` | 当前仅 `trae_agent` |

### 13.5 Docker 模式

```bash
trae-cli run "Add tests" --docker-image python:3.11
trae-cli run "..." --docker-image python:3.12 --working-dir test_workdir/
trae-cli run "..." --docker-container-id 91998a56056c
trae-cli run "..." --dockerfile-path /abs/Dockerfile
trae-cli run "..." --docker-image-file image.tar
trae-cli run "..." --docker-image python:3.11 --docker-keep false
```

### 13.6 交互模式内置命令

```
<task description>   执行任务
status              显示 agent 信息
help                显示可用命令
clear               清屏
exit / quit         退出
```

### 13.7 Makefile 快捷

```bash
make install-dev     # uv venv + sync --all-extras
make test            # pytest（跳过外部服务）
make pre-commit      # 安装并运行 pre-commit
make fix-format      # ruff format + check --fix
make clean
```

---

## 14. 测试体系

测试位于 [tests/](file:///workspace/trae-agent-repo/tests/)，按模块分类：

| 路径 | 内容 |
|---|---|
| `tests/agent/test_trae_agent.py` | agent 行为（new_task、execute_task、must_patch 流程） |
| `tests/tools/test_bash_tool.py` | BashTool 持久化、超时、重启 |
| `tests/tools/test_edit_tool.py` | str_replace / create / insert / view |
| `tests/tools/test_json_edit_tool.py` | JSONPath 各操作 |
| `tests/tools/test_mcp_tool.py` | MCP 工具封装 |
| `tests/utils/test_config.py` | YAML / LegacyConfig 解析、CLI/env 覆盖 |
| `tests/utils/test_google_client.py` | Gemini 客户端 |
| `tests/utils/test_mcp_client.py` | MCP 连接与清理 |
| `tests/utils/test_ollama_client_utils.py` | Ollama 客户端 |
| `tests/utils/test_openrouter_client_utils.py` | OpenRouter 客户端 |
| `tests/test_cli.py` | CLI 子命令的 smoke test |

**运行**：
```bash
make test                          # 跳过 Ollama/OpenRouter/Google
SKIP_OLLAMA_TEST=false make test   # 跑全部
```

**pytest 标记**：`@pytest.mark.slow / integration / unit`；`asyncio_mode = "auto"`。

---

## 15. 评估与基准（evaluation/）

[evaluation/](file:///workspace/trae-agent-repo/evaluation/) 包含 SWE-bench 风格的代码补丁选择与生成评测：

```
evaluation/
├── README.md
├── run_evaluation.py         # 入口
├── setup.sh                  # 评测环境准备
├── utils.py
├── patch_selection/
│   ├── README.md
│   ├── analysis.py
│   ├── selector.py
│   ├── selector_evaluation.py
│   ├── example/example.jsonl
│   └── trae_selector/
│       ├── __init__.py
│       ├── selector_agent.py
│       ├── sandbox.py
│       ├── utils.py
│       └── tools/tools/      # 复用核心 tools/
│           ├── base.py
│           ├── bash.py
│           ├── edit.py
│           ├── execute_bash.py
│           ├── execute_str_replace_editor.py
│           └── run.py
```

**selector_agent**：在多个候选 patch 中按"是否最小化、是否修改测试、是否能复现修复"等标准挑选最佳结果；通过 `sandbox.py` 在受控环境运行候选测试。

---

## 16. 扩展路线图（roadmap）

源自 [docs/roadmap.md](file:///workspace/trae-agent-repo/docs/roadmap.md)：

1. **SDK 化** — 提供无头 API 与流式轨迹，便于嵌入 CI/CD 与应用
2. **沙箱环境** — 容器化/虚拟化的安全执行；并行任务
3. **轨迹分析** — 集成 W&B Weave、MLFlow；做 token / 决策模式分析
4. **更结构化文件 & MCP 扩展** — Jupyter Notebooks、配置文件的原生支持
5. **多智能体协作 & 高级 agentic flow** — 多 agent 协同、专业化分工
6. **社区共建** — 欢迎提交 issue / PR / 学术成果

---

## 17. 常见问题排查

| 症状 | 解决方案 |
|---|---|
| `ImportError: ...` | `PYTHONPATH=. trae-cli run "..."` 或 `uv run trae-cli ...` |
| `command not found: trae-cli` | `uv run trae-cli ...` 或 `pip install -e .` |
| API Key 未生效 | `echo $OPENAI_API_KEY`；`trae-cli show-config` |
| Docker 模式报"未配置" | 启动 docker daemon；检查 `docker version` |
| 轨迹文件找不到 | 任务结束后默认输出在终端；可用 `--trajectory-file` 指定 |
| 工具未注册 | `trae-cli tools` 列出当前可用工具；检查 `agents.<name>.tools` 列表 |
| 提示词版本过旧 | 升级后清空 `.venv` 重装：`uv sync --reinstall` |

---

## 18. 附录：关键文件清单

| 模块 | 关键文件 | 作用 |
|---|---|---|
| CLI | [cli.py](file:///workspace/trae-agent-repo/trae_agent/cli.py) | Click 入口；`run` / `interactive` / `show-config` / `tools` |
| Agent 工厂 | [agent/agent.py](file:///workspace/trae-agent-repo/trae_agent/agent/agent.py) | Agent 类 + AgentType 枚举 |
| 主循环 | [agent/base_agent.py](file:///workspace/trae-agent-repo/trae_agent/agent/base_agent.py) | ReAct 主循环 |
| 特化 | [agent/trae_agent.py](file:///workspace/trae-agent-repo/trae_agent/agent/trae_agent.py) | TraeAgent：默认工具集、system prompt、patch 校验 |
| 数据结构 | [agent/agent_basics.py](file:///workspace/trae-agent-repo/trae_agent/agent/agent_basics.py) | AgentStep / AgentExecution / 状态机 |
| Docker | [agent/docker_manager.py](file:///workspace/trae-agent-repo/trae_agent/agent/docker_manager.py) | 容器生命周期 |
| Tool 基类 | [tools/base.py](file:///workspace/trae-agent-repo/trae_agent/tools/base.py) | Tool / ToolExecutor / ToolResult |
| Bash 工具 | [tools/bash_tool.py](file:///workspace/trae-agent-repo/trae_agent/tools/bash_tool.py) | 持久化 shell 会话 |
| 文件编辑 | [tools/edit_tool.py](file:///workspace/trae-agent-repo/trae_agent/tools/edit_tool.py) | str_replace 系列 |
| JSON 编辑 | [tools/json_edit_tool.py](file:///workspace/trae-agent-repo/trae_agent/tools/json_edit_tool.py) | JSONPath |
| 序列化思考 | [tools/sequential_thinking_tool.py](file:///workspace/trae-agent-repo/trae_agent/tools/sequential_thinking_tool.py) | 链式思考 |
| 任务结束 | [tools/task_done_tool.py](file:///workspace/trae-agent-repo/trae_agent/tools/task_done_tool.py) | 终止信号 |
| MCP 工具 | [tools/mcp_tool.py](file:///workspace/trae-agent-repo/trae_agent/tools/mcp_tool.py) | MCP tool 适配 |
| CKG | [tools/ckg_tool.py](file:///workspace/trae-agent-repo/trae_agent/tools/ckg_tool.py) | 代码知识图谱 |
| Docker Tool Exec | [tools/docker_tool_executor.py](file:///workspace/trae-agent-repo/trae_agent/tools/docker_tool_executor.py) | 容器内执行工具 |
| LLM 工厂 | [utils/llm_clients/llm_client.py](file:///workspace/trae-agent-repo/trae_agent/utils/llm_clients/llm_client.py) | LLMProvider 路由 |
| LLM 基类 | [utils/llm_clients/base_client.py](file:///workspace/trae-agent-repo/trae_agent/utils/llm_clients/base_client.py) | BaseLLMClient |
| 数据格式 | [utils/llm_clients/llm_basics.py](file:///workspace/trae-agent-repo/trae_agent/utils/llm_clients/llm_basics.py) | LLMMessage / LLMResponse / LLMUsage |
| 重试 | [utils/llm_clients/retry_utils.py](file:///workspace/trae-agent-repo/trae_agent/utils/llm_clients/retry_utils.py) | 3-30s 退避 |
| 配置 | [utils/config.py](file:///workspace/trae-agent-repo/trae_agent/utils/config.py) | Config / AgentConfig / ModelConfig |
| Legacy Config | [utils/legacy_config.py](file:///workspace/trae-agent-repo/trae_agent/utils/legacy_config.py) | JSON 兼容 |
| 轨迹 | [utils/trajectory_recorder.py](file:///workspace/trae-agent-repo/trae_agent/utils/trajectory_recorder.py) | 落盘 JSON |
| Lakeview | [utils/lake_view.py](file:///workspace/trae-agent-repo/trae_agent/utils/lake_view.py) | 步骤摘要 + 标签 |
| MCP 客户端 | [utils/mcp_client.py](file:///workspace/trae-agent-repo/trae_agent/utils/mcp_client.py) | stdio MCP |
| CLI 抽象 | [utils/cli/cli_console.py](file:///workspace/trae-agent-repo/trae_agent/utils/cli/cli_console.py) | CLIConsole ABC |
| 简单控制台 | [utils/cli/simple_console.py](file:///workspace/trae-agent-repo/trae_agent/utils/cli/simple_console.py) | rich print |
| 富控制台 | [utils/cli/rich_console.py](file:///workspace/trae-agent-repo/trae_agent/utils/cli/rich_console.py) | textual TUI |
| 工厂 | [utils/cli/console_factory.py](file:///workspace/trae-agent-repo/trae_agent/utils/cli/console_factory.py) | ConsoleFactory |
| 系统 Prompt | [prompt/agent_prompt.py](file:///workspace/trae-agent-repo/trae_agent/prompt/agent_prompt.py) | TRAE_AGENT_SYSTEM_PROMPT |

---

*本文档基于 [bytedance/trae-agent](https://github.com/bytedance/trae-agent) main 分支（v0.1.0）源码分析生成；如版本升级请以最新源码为准。*
