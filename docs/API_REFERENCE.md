# API Reference

This document provides comprehensive API documentation for using Trae Agent programmatically in Python applications.

## Table of Contents

- [Overview](#overview)
- [Agent API](#agent-api)
- [LLM Client API](#llm-client-api)
- [Tools API](#tools-api)
- [Configuration API](#configuration-api)
- [Trajectory Recording API](#trajectory-recording-api)
- [Usage Examples](#usage-examples)

## Overview

Trae Agent can be used as a Python library in your applications. This allows you to:

- Embed agent capabilities in your applications
- Create custom workflows and automations
- Build tools that leverage LLM-powered software engineering
- Integrate with existing systems

### Basic Import

```python
from trae_agent.agent import Agent
from trae_agent.utils.config import Config
```

## Agent API

### Agent Class

The main interface for creating and running agents.

```python
from trae_agent.agent import Agent, AgentType

class Agent:
    def __init__(
        self,
        agent_type: AgentType | str,
        config: Config,
        trajectory_file: str | None = None,
        cli_console: CLIConsole | None = None,
        docker_config: dict | None = None,
        docker_keep: bool = True,
    )
```

**Parameters:**
- `agent_type`: Type of agent ("trae_agent")
- `config`: Configuration object
- `trajectory_file`: Optional path for trajectory recording
- `cli_console`: Optional console for output
- `docker_config`: Optional Docker configuration
- `docker_keep`: Keep Docker container after execution

**Methods:**

#### `async run(task: str, extra_args: dict | None, tool_names: list[str] | None)`

Execute a task asynchronously.

**Parameters:**
- `task`: Natural language task description
- `extra_args`: Additional arguments (project_path, must_patch, etc.)
- `tool_names`: Override default tools for this task

**Returns:**
- `AgentExecution`: Execution result object

**Example:**
```python
import asyncio
from trae_agent.agent import Agent
from trae_agent.utils.config import Config

# Load configuration
config = Config.create(config_file="trae_config.yaml")

# Create agent
agent = Agent("trae_agent", config)

# Run task
async def main():
    result = await agent.run(
        task="Create a hello world Python script",
        extra_args={
            "project_path": "/path/to/project",
            "must_patch": "false"
        }
    )
    print(f"Success: {result.success}")
    print(f"Result: {result.final_result}")

asyncio.run(main())
```

### TraeAgent Class

The concrete implementation of the agent for software engineering.

```python
from trae_agent.agent.trae_agent import TraeAgent
from trae_agent.utils.config import TraeAgentConfig

class TraeAgent(BaseAgent):
    def __init__(
        self,
        trae_agent_config: TraeAgentConfig,
        docker_config: dict | None = None,
        docker_keep: bool = True,
    )
```

**Methods:**

#### `async execute_task() -> AgentExecution`

Execute the configured task.

**Returns:**
- `AgentExecution`: Result of execution

#### `new_task(task: str, extra_args: dict | None, tool_names: list[str] | None)`

Configure a new task.

**Parameters:**
- `task`: Task description
- `extra_args`: Additional arguments
- `tool_names`: Tools to use

#### `async initialise_mcp()`

Initialize MCP servers and discover external tools.

### AgentExecution Class

Result of agent execution.

```python
@dataclass
class AgentExecution:
    success: bool
    final_result: str
    error: str | None
    steps: list[AgentStep]
```

**Attributes:**
- `success`: Whether task completed successfully
- `final_result`: Final output or result message
- `error`: Error message if failed
- `steps`: List of execution steps

### AgentStep Class

Represents a single step in agent execution.

```python
@dataclass
class AgentStep:
    step_number: int
    state: AgentStepState
    llm_response: LLMResponse | None
    tool_calls: list[ToolCall]
    tool_results: list[ToolResult]
    reflection: str | None
    error: str | None
```

## LLM Client API

### LLMClient Class

Unified interface for multiple LLM providers.

```python
from trae_agent.utils.llm_clients.llm_client import LLMClient
from trae_agent.utils.config import ModelConfig

class LLMClient:
    def __init__(self, model_config: ModelConfig)
```

**Methods:**

#### `chat(messages: list[LLMMessage], model_config: ModelConfig, tools: list[Tool] | None, reuse_history: bool) -> LLMResponse`

Send messages to the LLM.

**Parameters:**
- `messages`: List of conversation messages
- `model_config`: Model configuration
- `tools`: Available tools
- `reuse_history`: Whether to maintain conversation history

**Returns:**
- `LLMResponse`: Response from the LLM

**Example:**
```python
from trae_agent.utils.llm_clients.llm_client import LLMClient
from trae_agent.utils.llm_clients.llm_basics import LLMMessage
from trae_agent.utils.config import ModelConfig, ModelProvider

# Configure model
model_config = ModelConfig(
    model="claude-sonnet-4-20250514",
    model_provider=ModelProvider(
        api_key="sk-ant-...",
        provider="anthropic"
    ),
    temperature=0.5,
    top_p=0.95,
    top_k=40,
    parallel_tool_calls=True,
    max_retries=3,
    max_tokens=4096
)

# Create client
client = LLMClient(model_config)

# Send message
messages = [
    LLMMessage(role="user", content="Write a hello world program")
]

response = client.chat(messages, model_config, tools=None)
print(response.content)
```

#### `set_trajectory_recorder(recorder: TrajectoryRecorder | None)`

Enable trajectory recording.

#### `set_chat_history(messages: list[LLMMessage])`

Set conversation history.

### LLMMessage Class

Represents a message in the conversation.

```python
@dataclass
class LLMMessage:
    role: str  # "system", "user", "assistant"
    content: str | list[dict]
    tool_calls: list[ToolCall] | None = None
    tool_results: list[ToolResult] | None = None
```

### LLMResponse Class

Response from the LLM.

```python
@dataclass
class LLMResponse:
    content: str
    model: str
    finish_reason: str
    usage: dict
    tool_calls: list[ToolCall] | None = None
```

## Tools API

### Tool Base Class

Create custom tools by extending the `Tool` class.

```python
from trae_agent.tools.base import Tool, ToolExecResult, ToolParameter

class CustomTool(Tool):
    @property
    def name(self) -> str:
        return "custom_tool"
    
    @property
    def description(self) -> str:
        return "Description of what this tool does"
    
    @property
    def parameters(self) -> list[ToolParameter]:
        return [
            ToolParameter(
                name="param1",
                type="string",
                description="First parameter",
                required=True
            )
        ]
    
    def _execute(self, **kwargs) -> ToolExecResult:
        # Implementation
        result = do_something(kwargs["param1"])
        return ToolExecResult(output=result)
```

### Built-in Tools

#### BashTool

Execute shell commands.

```python
from trae_agent.tools.bash_tool import BashTool

tool = BashTool()
result = tool.execute(command="ls -la", restart=False)
print(result.result)
```

#### EditTool

File editing operations.

```python
from trae_agent.tools.edit_tool import EditTool

tool = EditTool()

# View file
result = tool.execute(
    command="view",
    path="/path/to/file.py"
)

# Create file
result = tool.execute(
    command="create",
    path="/path/to/new_file.py",
    file_text="print('Hello')"
)

# Edit file
result = tool.execute(
    command="str_replace",
    path="/path/to/file.py",
    old_str="old text",
    new_str="new text"
)
```

#### JSONEditTool

JSON file operations.

```python
from trae_agent.tools.json_edit_tool import JSONEditTool

tool = JSONEditTool()

# View JSON
result = tool.execute(
    command="view",
    file_path="/path/to/config.json"
)

# Set value
result = tool.execute(
    command="set",
    file_path="/path/to/config.json",
    json_path="$.database.host",
    value="localhost"
)
```

### ToolExecutor

Execute tool calls.

```python
from trae_agent.tools.base import ToolExecutor, ToolCall

executor = ToolExecutor(tools)

tool_calls = [
    ToolCall(
        name="bash",
        call_id="call_1",
        arguments={"command": "ls"}
    )
]

results = executor.execute(tool_calls)
```

## Configuration API

### Config Class

Load and manage configuration.

```python
from trae_agent.utils.config import Config

class Config:
    @staticmethod
    def create(config_file: str = "trae_config.yaml") -> Config
```

**Methods:**

#### `resolve_config_values(**kwargs) -> Config`

Override configuration values.

**Example:**
```python
config = Config.create("trae_config.yaml")
config = config.resolve_config_values(
    provider="openai",
    model="gpt-4o",
    max_steps=300
)
```

### ModelConfig Class

Model configuration.

```python
from trae_agent.utils.config import ModelConfig, ModelProvider

model_config = ModelConfig(
    model="claude-sonnet-4-20250514",
    model_provider=ModelProvider(
        api_key="sk-ant-...",
        provider="anthropic",
        base_url=None,
        api_version=None
    ),
    temperature=0.5,
    top_p=0.95,
    top_k=40,
    parallel_tool_calls=True,
    max_retries=3,
    max_tokens=4096,
    supports_tool_calling=True
)
```

### TraeAgentConfig Class

Agent-specific configuration.

```python
from trae_agent.utils.config import TraeAgentConfig

agent_config = TraeAgentConfig(
    model=model_config,
    max_steps=200,
    tools=["bash", "str_replace_based_edit_tool", "task_done"],
    enable_lakeview=True,
    mcp_servers_config=None,
    allow_mcp_servers=[]
)
```

## Trajectory Recording API

### TrajectoryRecorder Class

Record agent execution for analysis.

```python
from trae_agent.utils.trajectory_recorder import TrajectoryRecorder

recorder = TrajectoryRecorder(filename="my_trajectory.json")
```

**Methods:**

#### `start_recording(task: str, provider: str, model: str, max_steps: int)`

Begin recording.

#### `record_llm_interaction(messages, response, provider, model, tools)`

Record LLM interaction.

#### `record_agent_step(step_number, state, llm_messages, llm_response, tool_calls, tool_results, reflection, error)`

Record agent step.

#### `finalize_recording(success: bool, final_result: str, error: str | None)`

Complete recording.

**Example:**
```python
from trae_agent.utils.trajectory_recorder import TrajectoryRecorder

recorder = TrajectoryRecorder("execution.json")
recorder.start_recording(
    task="Create hello world",
    provider="anthropic",
    model="claude-sonnet-4-20250514",
    max_steps=200
)

# ... execution ...

recorder.finalize_recording(
    success=True,
    final_result="Task completed",
    error=None
)
```

## Usage Examples

### Example 1: Simple Task Execution

```python
import asyncio
from trae_agent.agent import Agent
from trae_agent.utils.config import Config

async def run_task():
    # Load configuration
    config = Config.create("trae_config.yaml")
    
    # Create agent
    agent = Agent("trae_agent", config)
    
    # Execute task
    result = await agent.run(
        task="Create a function to calculate fibonacci numbers",
        extra_args={"project_path": "/tmp/test"}
    )
    
    print(f"Success: {result.success}")
    print(f"Result: {result.final_result}")
    
    return result

# Run
result = asyncio.run(run_task())
```

### Example 2: Custom Configuration

```python
import asyncio
from trae_agent.agent import Agent
from trae_agent.utils.config import (
    Config, TraeAgentConfig, ModelConfig, ModelProvider
)

async def run_with_custom_config():
    # Create configuration programmatically
    model_provider = ModelProvider(
        api_key="sk-ant-...",
        provider="anthropic"
    )
    
    model_config = ModelConfig(
        model="claude-sonnet-4-20250514",
        model_provider=model_provider,
        temperature=0.3,
        top_p=0.9,
        top_k=40,
        parallel_tool_calls=True,
        max_retries=3,
        max_tokens=4096
    )
    
    agent_config = TraeAgentConfig(
        model=model_config,
        max_steps=100,
        tools=["bash", "str_replace_based_edit_tool", "task_done"],
        enable_lakeview=False
    )
    
    config = Config(trae_agent=agent_config)
    
    # Create and run agent
    agent = Agent("trae_agent", config, trajectory_file="my_run.json")
    
    result = await agent.run(
        task="Write a unit test for the login function",
        extra_args={"project_path": "/path/to/project"}
    )
    
    return result

result = asyncio.run(run_with_custom_config())
```

### Example 3: Using Docker

```python
import asyncio
from trae_agent.agent import Agent
from trae_agent.utils.config import Config

async def run_in_docker():
    config = Config.create("trae_config.yaml")
    
    # Docker configuration
    docker_config = {
        "image": "python:3.12",
        "workspace_dir": "/home/user/project"
    }
    
    agent = Agent(
        "trae_agent",
        config,
        docker_config=docker_config,
        docker_keep=False  # Remove container after
    )
    
    result = await agent.run(
        task="Run the test suite",
        extra_args={"project_path": "/home/user/project"}
    )
    
    return result

result = asyncio.run(run_in_docker())
```

### Example 4: Multiple Tasks in Sequence

```python
import asyncio
from trae_agent.agent import Agent
from trae_agent.utils.config import Config

async def run_multiple_tasks():
    config = Config.create("trae_config.yaml")
    agent = Agent("trae_agent", config)
    
    tasks = [
        "Create a User class with name and email attributes",
        "Add a method to validate email format",
        "Write unit tests for the User class"
    ]
    
    results = []
    for task in tasks:
        result = await agent.run(
            task=task,
            extra_args={"project_path": "/tmp/myproject"}
        )
        results.append(result)
        
        if not result.success:
            print(f"Task failed: {task}")
            break
    
    return results

results = asyncio.run(run_multiple_tasks())
```

### Example 5: Custom Tool Usage

```python
import asyncio
from trae_agent.agent import Agent
from trae_agent.utils.config import Config

async def run_with_custom_tools():
    config = Config.create("trae_config.yaml")
    
    # Override tools for this specific task
    custom_tools = ["bash", "str_replace_based_edit_tool"]
    
    agent = Agent("trae_agent", config)
    
    result = await agent.run(
        task="Add logging to the application",
        extra_args={"project_path": "/path/to/app"},
        tool_names=custom_tools
    )
    
    return result

result = asyncio.run(run_with_custom_tools())
```

### Example 6: Error Handling

```python
import asyncio
from trae_agent.agent import Agent
from trae_agent.utils.config import Config

async def run_with_error_handling():
    config = Config.create("trae_config.yaml")
    agent = Agent("trae_agent", config)
    
    try:
        result = await agent.run(
            task="Complex task that might fail",
            extra_args={"project_path": "/path/to/project"}
        )
        
        if result.success:
            print("Task completed successfully!")
            print(f"Result: {result.final_result}")
        else:
            print("Task failed:")
            print(f"Error: {result.error}")
            
    except Exception as e:
        print(f"Exception occurred: {e}")
        # Handle error
    
    return result

result = asyncio.run(run_with_error_handling())
```

### Example 7: Accessing Trajectory Data

```python
import asyncio
import json
from trae_agent.agent import Agent
from trae_agent.utils.config import Config

async def analyze_execution():
    config = Config.create("trae_config.yaml")
    trajectory_file = "detailed_execution.json"
    
    agent = Agent("trae_agent", config, trajectory_file=trajectory_file)
    
    result = await agent.run(
        task="Refactor the database module",
        extra_args={"project_path": "/path/to/project"}
    )
    
    # Read trajectory data
    with open(trajectory_file, 'r') as f:
        trajectory = json.load(f)
    
    # Analyze
    print(f"Total steps: {len(trajectory['agent_steps'])}")
    print(f"Total tokens: {sum(
        step['llm_response']['usage']['input_tokens'] + 
        step['llm_response']['usage']['output_tokens']
        for step in trajectory['llm_interactions']
    )}")
    
    return trajectory

trajectory = asyncio.run(analyze_execution())
```

### Example 8: Integration with Web Framework

```python
from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel
import asyncio
from trae_agent.agent import Agent
from trae_agent.utils.config import Config

app = FastAPI()

class TaskRequest(BaseModel):
    task: str
    project_path: str

class TaskResponse(BaseModel):
    task_id: str
    status: str

# Store results
task_results = {}

async def execute_task(task_id: str, task: str, project_path: str):
    """Background task execution"""
    config = Config.create("trae_config.yaml")
    agent = Agent("trae_agent", config)
    
    result = await agent.run(
        task=task,
        extra_args={"project_path": project_path}
    )
    
    task_results[task_id] = {
        "success": result.success,
        "result": result.final_result,
        "error": result.error
    }

@app.post("/tasks", response_model=TaskResponse)
async def create_task(request: TaskRequest, background_tasks: BackgroundTasks):
    """Create a new task"""
    import uuid
    task_id = str(uuid.uuid4())
    
    background_tasks.add_task(
        execute_task,
        task_id,
        request.task,
        request.project_path
    )
    
    return TaskResponse(task_id=task_id, status="running")

@app.get("/tasks/{task_id}")
async def get_task_result(task_id: str):
    """Get task result"""
    if task_id not in task_results:
        return {"status": "running"}
    
    return {
        "status": "completed",
        **task_results[task_id]
    }
```

## Best Practices

### 1. Always Use Async

Trae Agent is designed for async operation:

```python
# Good
async def main():
    result = await agent.run(task)

asyncio.run(main())

# Bad - won't work
result = agent.run(task)  # Error: awaitable not awaited
```

### 2. Handle Errors Gracefully

```python
try:
    result = await agent.run(task)
    if not result.success:
        # Handle failure
        log_error(result.error)
except Exception as e:
    # Handle exception
    log_exception(e)
```

### 3. Use Configuration Files

Prefer configuration files over hardcoded values:

```python
# Good
config = Config.create("trae_config.yaml")

# Avoid
config = create_config_manually()  # Harder to maintain
```

### 4. Enable Trajectory Recording

For debugging and analysis:

```python
agent = Agent("trae_agent", config, trajectory_file="execution.json")
```

### 5. Cleanup Resources

When using Docker or MCP:

```python
try:
    result = await agent.run(task)
finally:
    # Cleanup happens automatically
    pass
```

## Related Documentation

- [Getting Started](GETTING_STARTED.md)
- [Configuration Guide](CONFIGURATION.md)
- [Architecture](ARCHITECTURE.md)
- [Workflow Guide](WORKFLOW.md)

## Summary

The Trae Agent API provides:

- **Agent**: High-level interface for task execution
- **LLMClient**: Multi-provider LLM access
- **Tools**: Extensible tool system
- **Config**: Flexible configuration management
- **TrajectoryRecorder**: Detailed execution logging

All operations are asynchronous and designed for embedding in larger applications.
