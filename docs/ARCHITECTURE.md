# Trae Agent Architecture

This document provides a comprehensive overview of the Trae Agent architecture, including its design principles, core components, and how they interact.

## Table of Contents

- [Overview](#overview)
- [Design Principles](#design-principles)
- [System Architecture](#system-architecture)
- [Core Components](#core-components)
- [Data Flow](#data-flow)
- [Extension Points](#extension-points)

## Overview

Trae Agent is an LLM-based agent framework designed for general-purpose software engineering tasks. It features a modular, research-friendly architecture that allows researchers and developers to easily modify, extend, and analyze agent behavior.

### Key Characteristics

- **Modular Design**: Components are loosely coupled and can be independently modified or replaced
- **Multi-LLM Support**: Works with multiple LLM providers through a unified interface
- **Tool-Based Architecture**: Extensible tool system for various operations
- **Docker Support**: Can run in isolated Docker environments for safety
- **Trajectory Recording**: Complete execution logging for analysis and debugging

## Design Principles

1. **Transparency**: All agent actions and decisions are logged and traceable
2. **Modularity**: Components can be developed and tested independently
3. **Extensibility**: New tools, LLM providers, and agents can be added easily
4. **Research-Friendly**: Designed to facilitate experimentation and ablation studies
5. **Safety**: Docker isolation and controlled tool execution

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLI Interface                            │
│  (trae_agent/cli.py - Command line interface and entry point)   │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Agent Layer                                 │
│  ┌──────────────┐    ┌─────────────────┐                       │
│  │   Agent      │───▶│   TraeAgent     │                       │
│  │ (agent.py)   │    │ (trae_agent.py) │                       │
│  └──────────────┘    └────────┬────────┘                       │
│                               │                                  │
│                     ┌─────────▼──────────┐                      │
│                     │    BaseAgent       │                      │
│                     │  (base_agent.py)   │                      │
│                     └────────┬───────────┘                      │
└──────────────────────────────┼──────────────────────────────────┘
                               │
            ┌──────────────────┴──────────────────┐
            ▼                                      ▼
┌──────────────────────────┐         ┌─────────────────────────┐
│    LLM Client Layer      │         │     Tool Layer          │
│                          │         │                         │
│  ┌────────────────────┐ │         │  ┌──────────────────┐  │
│  │   LLMClient        │ │         │  │  Tool Registry   │  │
│  │ (llm_client.py)    │ │         │  │  (tools/__init__)│  │
│  └────────┬───────────┘ │         │  └────────┬─────────┘  │
│           │              │         │           │             │
│  ┌────────▼───────────┐ │         │  ┌────────▼─────────┐  │
│  │  Provider Clients  │ │         │  │  Individual      │  │
│  │  - OpenAI          │ │         │  │  Tools:          │  │
│  │  - Anthropic       │ │         │  │  - bash          │  │
│  │  - Google          │ │         │  │  - edit_tool     │  │
│  │  - Azure           │ │         │  │  - json_edit     │  │
│  │  - Doubao          │ │         │  │  - sequential    │  │
│  │  - Ollama          │ │         │  │  - task_done     │  │
│  │  - OpenRouter      │ │         │  │  - ckg_tool      │  │
│  └────────────────────┘ │         │  │  - mcp_tool      │  │
│                          │         │  └──────────────────┘  │
└──────────────────────────┘         └─────────────────────────┘
            │                                      │
            └──────────────────┬───────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Support Services                              │
│                                                                   │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────┐ │
│  │ Configuration    │  │  Trajectory      │  │  Console UI   │ │
│  │  - Config        │  │  Recorder        │  │  - Simple     │ │
│  │  - ModelConfig   │  │  (logging)       │  │  - Rich       │ │
│  │  - AgentConfig   │  │                  │  │  - Lakeview   │ │
│  └──────────────────┘  └──────────────────┘  └───────────────┘ │
│                                                                   │
│  ┌──────────────────┐  ┌──────────────────┐                     │
│  │ Docker Manager   │  │  MCP Client      │                     │
│  │  (isolation)     │  │  (external tools)│                     │
│  └──────────────────┘  └──────────────────┘                     │
└─────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. CLI Interface (`trae_agent/cli.py`)

The command-line interface is the primary entry point for users. It handles:

- **Command Parsing**: Processes user commands and options
- **Configuration Loading**: Resolves configuration from files, environment variables, and CLI arguments
- **Agent Initialization**: Creates and configures agent instances
- **Execution Modes**: Supports `run`, `interactive`, and `show-config` modes
- **Docker Configuration**: Sets up Docker environments when requested

**Key Functions:**
- `run()`: Execute a single task
- `interactive()`: Start an interactive session
- `show_config()`: Display current configuration
- `tools()`: List available tools

### 2. Agent Layer

#### Agent (`trae_agent/agent/agent.py`)

The `Agent` class is a factory and orchestrator that:
- Creates specific agent types (currently `TraeAgent`)
- Manages trajectory recording
- Coordinates CLI console and agent execution
- Handles MCP server initialization and cleanup

#### TraeAgent (`trae_agent/agent/trae_agent.py`)

The main agent implementation for software engineering tasks:
- Extends `BaseAgent`
- Implements task execution logic
- Manages project context (path, commit, patches)
- Discovers and integrates MCP tools
- Handles Docker-based execution

**Key Features:**
- System prompt: `TRAE_AGENT_SYSTEM_PROMPT` (defines agent behavior)
- Tool set: bash, edit tools, sequential thinking, task completion
- MCP integration: External tool discovery and integration

#### BaseAgent (`trae_agent/agent/base_agent.py`)

Abstract base class providing core agent functionality:

**Responsibilities:**
- Tool execution orchestration
- LLM interaction management
- State machine implementation
- Error handling and recovery
- Docker tool executor integration

**Agent State Machine:**
```
INITIAL → THINKING → CALLING_TOOL → THINKING → ... → COMPLETED
                 ↓                                    ↑
                 └──────────── ERROR ────────────────┘
```

**Key Methods:**
- `new_task()`: Initialize a new task
- `execute_task()`: Main execution loop
- `_process_step()`: Process a single agent step
- `_call_tools()`: Execute tool calls
- `_reflect()`: Generate reflections after tool execution

### 3. LLM Client Layer

#### LLMClient (`trae_agent/utils/llm_clients/llm_client.py`)

Unified interface for multiple LLM providers:

**Provider Support:**
- OpenAI (GPT models)
- Anthropic (Claude models)
- Google (Gemini models)
- Azure OpenAI
- Doubao (ByteDance)
- Ollama (local models)
- OpenRouter (multi-provider access)

**Key Features:**
- Automatic provider selection based on configuration
- Trajectory recording integration
- Chat history management
- Tool calling support

#### Provider-Specific Clients

Each provider has a dedicated client implementing `BaseLLMClient`:

**Common Interface:**
- `chat()`: Send messages and receive responses
- `set_trajectory_recorder()`: Enable execution logging
- `set_chat_history()`: Manage conversation context

**Provider-Specific Features:**
- **OpenAI**: Function calling, streaming support
- **Anthropic**: Tool use, prompt caching, thinking blocks
- **Google**: Multi-candidate generation, safety settings
- **Azure**: Deployment-specific configuration
- **Doubao**: Custom endpoint support
- **Ollama**: Local model execution
- **OpenRouter**: Access to multiple providers

### 4. Tool System

#### Tool Registry (`trae_agent/tools/__init__.py`)

Central registry for all available tools:

```python
tools_registry = {
    "bash": BashTool,
    "str_replace_based_edit_tool": EditTool,
    "json_edit_tool": JSONEditTool,
    "sequentialthinking": SequentialThinkingTool,
    "task_done": TaskDoneTool,
    "ckg_tool": CKGTool,
}
```

#### Base Tool (`trae_agent/tools/base.py`)

Abstract base class for all tools:

**Interface:**
- `name`: Tool identifier
- `description`: Tool purpose
- `parameters`: Input schema
- `execute()`: Tool execution logic
- `to_openai_schema()`: OpenAI-compatible schema
- `to_anthropic_schema()`: Anthropic-compatible schema
- `to_gemini_schema()`: Google Gemini-compatible schema

**Tool Result Structure:**
```python
@dataclass
class ToolResult:
    call_id: str
    name: str
    success: bool
    result: str | None
    error: str | None
```

#### Individual Tools

**1. BashTool (`bash_tool.py`)**
- Execute shell commands
- Persistent session state
- Timeout handling
- Background process support

**2. EditTool (`edit_tool.py`)**
- File viewing (with line numbers)
- File creation
- String replacement editing
- Text insertion
- Directory listing

**3. JSONEditTool (`json_edit_tool.py`)**
- JSON file viewing
- JSONPath-based editing
- Add/set/remove operations
- Validation and error handling

**4. SequentialThinkingTool (`sequential_thinking_tool.py`)**
- Structured reasoning
- Thought branching and revision
- Dynamic thought allocation
- Thinking history tracking

**5. TaskDoneTool (`task_done_tool.py`)**
- Task completion signaling
- Simple success marker

**6. CKGTool (`ckg_tool.py`)**
- Code Knowledge Graph operations
- Codebase understanding
- Not enabled by default

**7. MCPTool (`mcp_tool.py`)**
- Model Context Protocol integration
- Dynamic external tool discovery
- Server communication

### 5. Configuration System

#### Config (`trae_agent/utils/config.py`)

Hierarchical configuration management:

**Configuration Priority:**
1. Command-line arguments (highest)
2. Configuration file (YAML/JSON)
3. Environment variables
4. Default values (lowest)

**Configuration Classes:**

**ModelProvider:**
```python
@dataclass
class ModelProvider:
    api_key: str
    provider: str
    base_url: str | None
    api_version: str | None
```

**ModelConfig:**
```python
@dataclass
class ModelConfig:
    model: str
    model_provider: ModelProvider
    temperature: float
    top_p: float
    top_k: int
    max_tokens: int
    parallel_tool_calls: bool
    supports_tool_calling: bool
```

**TraeAgentConfig:**
```python
@dataclass
class TraeAgentConfig:
    model: ModelConfig
    max_steps: int
    tools: list[str]
    enable_lakeview: bool
    mcp_servers_config: dict[str, MCPServerConfig]
    allow_mcp_servers: list[str]
```

### 6. Support Services

#### Trajectory Recorder (`trae_agent/utils/trajectory_recorder.py`)

Comprehensive execution logging:

**Recorded Data:**
- LLM interactions (requests, responses, tokens)
- Agent steps (state, actions, results)
- Tool calls and results
- Timing information
- Error information

**Output Format:**
```json
{
  "task": "Task description",
  "start_time": "ISO timestamp",
  "end_time": "ISO timestamp",
  "provider": "llm provider",
  "model": "model name",
  "llm_interactions": [...],
  "agent_steps": [...],
  "success": true,
  "final_result": "Result summary",
  "execution_time": 123.45
}
```

#### Console System (`trae_agent/utils/cli/`)

Multiple console interfaces:

**1. SimpleConsole (`simple_console.py`)**
- Basic text output
- Minimal formatting
- Universal compatibility

**2. RichConsole (`rich_console.py`)**
- Enhanced formatting
- Progress bars
- Tables and panels
- Color coding

**3. Lakeview (`trae_agent/utils/lake_view.py`)**
- Concise step summaries
- Key action highlighting
- Reduced verbosity

**Console Factory (`console_factory.py`):**
- Automatic console selection
- Mode-specific configuration (run vs. interactive)

#### Docker Manager (`trae_agent/agent/docker_manager.py`)

Container-based execution:

**Capabilities:**
- Create containers from images
- Attach to existing containers
- Build from Dockerfiles
- Load from image files
- Workspace mounting
- Tool binary copying
- Container cleanup

**Docker Tool Executor (`docker_tool_executor.py`):**
- Proxies tool calls to container
- Path translation (host ↔ container)
- Selective tool execution (bash, edit tools)

#### MCP Client (`trae_agent/utils/mcp_client.py`)

Model Context Protocol integration:

**Features:**
- External server connection
- Tool discovery
- Dynamic tool registration
- Server lifecycle management

## Data Flow

### Task Execution Flow

```
1. User Input
   │
   ├─→ CLI parses command
   │
   ├─→ Config loaded and resolved
   │
   └─→ Agent created

2. Agent Initialization
   │
   ├─→ LLM client configured
   │
   ├─→ Tools registered
   │
   ├─→ Trajectory recorder initialized
   │
   ├─→ MCP servers connected (if configured)
   │
   └─→ Docker environment prepared (if requested)

3. Task Execution Loop
   │
   ├─→ Agent receives task
   │   │
   │   ├─→ System prompt + task → LLM
   │   │
   │   ├─→ LLM response recorded
   │   │
   │   ├─→ Tool calls extracted
   │   │
   │   ├─→ Tools executed
   │   │   │
   │   │   ├─→ In Docker (if configured)
   │   │   │   │
   │   │   │   └─→ Paths translated
   │   │   │
   │   │   └─→ Or locally
   │   │
   │   ├─→ Results collected
   │   │
   │   ├─→ Results → LLM
   │   │
   │   └─→ Step recorded
   │
   └─→ Repeat until:
       ├─→ Task completed (task_done called)
       ├─→ Max steps reached
       └─→ Error encountered

4. Finalization
   │
   ├─→ Trajectory saved
   │
   ├─→ Docker cleanup (if used)
   │
   ├─→ MCP servers disconnected
   │
   └─→ Results displayed
```

### Message Flow Between Components

```
CLI → Agent → BaseAgent → LLMClient → Provider API
                    ↓
                  Tools → ToolExecutor/DockerToolExecutor
                    ↓
              TrajectoryRecorder
                    ↓
              Console (display)
```

## Extension Points

### Adding a New LLM Provider

1. Create client in `trae_agent/utils/llm_clients/your_provider_client.py`
2. Extend `BaseLLMClient`
3. Implement `chat()` method
4. Add to `LLMProvider` enum
5. Add to `LLMClient` match statement
6. Add configuration support in `ModelProvider`

### Adding a New Tool

1. Create tool in `trae_agent/tools/your_tool.py`
2. Extend `Tool` base class
3. Implement required methods:
   - `name`, `description`, `parameters`
   - `execute()`, `_execute()`
   - Schema converters for each provider
4. Register in `tools_registry`
5. Add to agent configuration

### Adding a New Agent Type

1. Create agent in `trae_agent/agent/your_agent.py`
2. Extend `BaseAgent`
3. Implement specific behavior
4. Add to `AgentType` enum
5. Add to `Agent.__init__()` match statement
6. Create agent-specific configuration

### Customizing the Console

1. Create console in `trae_agent/utils/cli/your_console.py`
2. Extend `CLIConsole`
3. Implement display methods
4. Add to `ConsoleType` enum
5. Add to `ConsoleFactory.create_console()`

## Security Considerations

1. **API Key Protection**: Keys are never logged in trajectories
2. **Docker Isolation**: Tasks can run in isolated containers
3. **Tool Restrictions**: Tools have limited capabilities
4. **Path Validation**: File operations require absolute paths
5. **Timeout Protection**: Commands have execution limits

## Performance Considerations

1. **Prompt Caching**: Supported on Anthropic for repeated prompts
2. **Parallel Tool Calls**: Enabled for compatible providers
3. **Trajectory Streaming**: Continuous writing to avoid memory buildup
4. **Docker Overhead**: Container operations add latency
5. **MCP Latency**: External servers introduce network delays

## Future Architecture Directions

1. **Stateless HTTP Server**: FastAPI-based server for concurrent requests
2. **Multi-Agent Collaboration**: Multiple agents working together
3. **Plugin System**: Dynamic tool and agent loading
4. **Distributed Execution**: Multiple workers for parallel tasks
5. **Enhanced Caching**: Cross-session state persistence

## Related Documentation

- [Getting Started Guide](GETTING_STARTED.md)
- [Configuration Guide](CONFIGURATION.md)
- [Tools Documentation](tools.md)
- [Workflow Guide](WORKFLOW.md)
- [API Reference](API_REFERENCE.md)
- [Docker Usage](DOCKER.md)
- [Trajectory Recording](TRAJECTORY_RECORDING.md)
