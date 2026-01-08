# Configuration Guide

This comprehensive guide covers all configuration options for Trae Agent, from basic setup to advanced customization.

## Table of Contents

- [Configuration Overview](#configuration-overview)
- [Configuration Priority](#configuration-priority)
- [YAML Configuration](#yaml-configuration)
- [Environment Variables](#environment-variables)
- [Command-Line Arguments](#command-line-arguments)
- [Provider-Specific Configuration](#provider-specific-configuration)
- [MCP Server Configuration](#mcp-server-configuration)
- [Advanced Settings](#advanced-settings)
- [Configuration Examples](#configuration-examples)

## Configuration Overview

Trae Agent can be configured through three methods:
1. **YAML Configuration File** (recommended)
2. **Environment Variables**
3. **Command-Line Arguments**

The configuration file `trae_config.yaml` is the recommended approach for persistent settings, while environment variables and CLI arguments are useful for overrides and one-off changes.

## Configuration Priority

Settings are resolved in this order (highest to lowest priority):

```
1. Command-Line Arguments  (highest priority)
2. Configuration File
3. Environment Variables
4. Default Values          (lowest priority)
```

**Example:**
```bash
# Model in config file: claude-sonnet-4-20250514
# Model in environment: GPT-4o
# Model in CLI argument: gemini-2.5-flash

trae-cli run "task" --model gemini-2.5-flash
# Uses: gemini-2.5-flash (CLI wins)
```

## YAML Configuration

### Basic Structure

```yaml
# trae_config.yaml

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
    api_key: sk-ant-...
    provider: anthropic
  
  openai:
    api_key: sk-...
    provider: openai

models:
  trae_agent_model:
    model_provider: anthropic
    model: claude-sonnet-4-20250514
    max_tokens: 4096
    temperature: 0.5
    top_p: 0.95
    top_k: 40
    parallel_tool_calls: true
    max_retries: 3
```

### Configuration Sections

#### 1. Agents Section

Defines agent-specific settings.

```yaml
agents:
  trae_agent:
    # Enable concise output mode
    enable_lakeview: true
    
    # Reference to model configuration
    model: trae_agent_model
    
    # Maximum execution steps
    max_steps: 200
    
    # Available tools
    tools:
      - bash
      - str_replace_based_edit_tool
      - json_edit_tool
      - sequentialthinking
      - task_done
      # - ckg_tool  # Optional: Code Knowledge Graph
    
    # Optional: MCP servers to enable
    allow_mcp_servers:
      - playwright
      - filesystem
```

**Field Descriptions:**

| Field | Type | Description | Default |
|-------|------|-------------|---------|
| `enable_lakeview` | boolean | Show concise step summaries | `false` |
| `model` | string | Reference to model config name | required |
| `max_steps` | integer | Maximum agent steps (1-500) | `200` |
| `tools` | list | Enabled tools | required |
| `allow_mcp_servers` | list | Enabled MCP servers | `[]` |

#### 2. Model Providers Section

Defines LLM provider credentials and endpoints.

```yaml
model_providers:
  # Anthropic (Claude)
  anthropic:
    api_key: sk-ant-api03-...
    provider: anthropic
    # base_url: https://api.anthropic.com  # Optional override
  
  # OpenAI (GPT)
  openai:
    api_key: sk-...
    provider: openai
    # base_url: https://api.openai.com/v1  # Optional override
  
  # Google (Gemini)
  google:
    api_key: AI...
    provider: google
    # base_url: https://generativelanguage.googleapis.com  # Optional
  
  # Azure OpenAI
  azure:
    api_key: ...
    provider: azure
    base_url: https://your-resource.openai.azure.com
    api_version: 2024-02-15-preview
  
  # Doubao (ByteDance)
  doubao:
    api_key: ...
    provider: doubao
    base_url: https://ark.cn-beijing.volces.com/api/v3/
  
  # Ollama (Local)
  ollama:
    api_key: not-required
    provider: ollama
    base_url: http://localhost:11434
  
  # OpenRouter (Multi-provider)
  openrouter:
    api_key: sk-or-...
    provider: openai  # Uses OpenAI-compatible API
    base_url: https://openrouter.ai/api/v1
```

**Field Descriptions:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `api_key` | string | Yes | API authentication key |
| `provider` | string | Yes | Provider type (anthropic, openai, google, azure, doubao, ollama) |
| `base_url` | string | No | Custom API endpoint |
| `api_version` | string | Azure only | Azure API version |

#### 3. Models Section

Defines specific model configurations.

```yaml
models:
  # Claude Sonnet 4
  trae_agent_model:
    model_provider: anthropic
    model: claude-sonnet-4-20250514
    max_tokens: 4096
    temperature: 0.5
    top_p: 0.95
    top_k: 40
    parallel_tool_calls: true
    max_retries: 3
  
  # GPT-4o
  gpt4o_model:
    model_provider: openai
    model: gpt-4o
    max_tokens: 4096
    temperature: 0.5
    top_p: 0.95
    parallel_tool_calls: true
    max_retries: 3
  
  # Gemini 2.5 Flash
  gemini_model:
    model_provider: google
    model: gemini-2.5-flash
    max_tokens: 8192
    temperature: 0.5
    top_p: 0.95
    candidate_count: 1
    max_retries: 3
```

**Field Descriptions:**

| Field | Type | Description | Default |
|-------|------|-------------|---------|
| `model_provider` | string | Reference to provider config | required |
| `model` | string | Model name/identifier | required |
| `max_tokens` | integer | Maximum output tokens | `4096` |
| `temperature` | float | Randomness (0.0-2.0) | `0.5` |
| `top_p` | float | Nucleus sampling (0.0-1.0) | `0.95` |
| `top_k` | integer | Top-k sampling (Anthropic) | `40` |
| `parallel_tool_calls` | boolean | Enable parallel tool execution | `true` |
| `max_retries` | integer | Retry failed API calls | `3` |
| `candidate_count` | integer | Response candidates (Gemini) | `1` |
| `stop_sequences` | list | Custom stop sequences | `null` |

#### 4. MCP Servers Section

Configure Model Context Protocol servers for external tools.

```yaml
mcp_servers:
  playwright:
    command: npx
    args:
      - "@playwright/mcp@0.0.27"
    env:
      PLAYWRIGHT_BROWSERS_PATH: "~/.cache/ms-playwright"
  
  filesystem:
    command: npx
    args:
      - "@modelcontextprotocol/server-filesystem"
      - "/path/to/allowed/directory"
  
  github:
    command: npx
    args:
      - "@modelcontextprotocol/server-github"
    env:
      GITHUB_TOKEN: ghp_...
```

**Field Descriptions:**

| Field | Type | Description |
|-------|------|-------------|
| `command` | string | Command to execute |
| `args` | list | Command arguments |
| `env` | dict | Environment variables |

#### 5. Lakeview Section (Optional)

Configure concise output mode.

```yaml
lakeview:
  enabled: true
  max_length: 100
  show_tool_names: true
```

## Environment Variables

Environment variables provide an alternative to storing sensitive data in config files.

### Standard Variables

```bash
# Provider API Keys
export ANTHROPIC_API_KEY="sk-ant-..."
export OPENAI_API_KEY="sk-..."
export GOOGLE_API_KEY="AI..."
export AZURE_API_KEY="..."
export DOUBAO_API_KEY="..."
export OLLAMA_API_KEY="not-required"
export OPENROUTER_API_KEY="sk-or-..."

# Provider Base URLs (optional)
export ANTHROPIC_BASE_URL="https://api.anthropic.com"
export OPENAI_BASE_URL="https://api.openai.com/v1"
export GOOGLE_BASE_URL="https://generativelanguage.googleapis.com"
export AZURE_BASE_URL="https://your-resource.openai.azure.com"
export DOUBAO_BASE_URL="https://ark.cn-beijing.volces.com/api/v3/"
export OLLAMA_BASE_URL="http://localhost:11434"

# Azure Specific
export AZURE_API_VERSION="2024-02-15-preview"

# Configuration File Location
export TRAE_CONFIG_FILE="~/my-custom-config.yaml"
```

### Using .env File

Create a `.env` file in your project root:

```bash
# .env
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
GOOGLE_API_KEY=AI...
```

The `.env` file is automatically loaded when running `trae-cli`.

**Security Note:** Never commit `.env` files to version control. Add to `.gitignore`:
```bash
echo ".env" >> .gitignore
```

## Command-Line Arguments

CLI arguments provide the highest priority for configuration.

### Common Arguments

```bash
# Provider and Model
--provider, -p          # LLM provider (anthropic, openai, google, etc.)
--model, -m             # Specific model name
--model-base-url        # Custom API endpoint
--api-key, -k           # API key

# Execution Settings
--max-steps             # Maximum execution steps
--working-dir, -w       # Working directory
--config-file           # Path to config file

# Output Settings
--trajectory-file, -t   # Trajectory output path
--console-type, -ct     # Console type (simple, rich)

# Task Flags
--must-patch, -mp       # Require patch generation
--patch-path, -pp       # Patch output path

# Docker Settings
--docker-image          # Docker image to use
--docker-container-id   # Attach to existing container
--dockerfile-path       # Build from Dockerfile
--docker-image-file     # Load from tar file
--docker-keep           # Keep container after execution
```

### Usage Examples

```bash
# Override model
trae-cli run "Add tests" --model gpt-4o

# Use different provider
trae-cli run "Refactor code" --provider google --model gemini-2.5-flash

# Custom working directory
trae-cli run "Fix bug" --working-dir /path/to/project

# Increase max steps
trae-cli run "Complex task" --max-steps 300

# Custom configuration file
trae-cli run "Task" --config-file custom-config.yaml

# Save trajectory to specific file
trae-cli run "Debug" --trajectory-file debug-session.json
```

## Provider-Specific Configuration

### Anthropic (Claude)

```yaml
model_providers:
  anthropic:
    api_key: sk-ant-api03-...
    provider: anthropic

models:
  claude_model:
    model_provider: anthropic
    model: claude-sonnet-4-20250514  # or claude-opus-4-20250514
    max_tokens: 4096
    temperature: 0.5
    top_p: 0.95
    top_k: 40
```

**Available Models:**
- `claude-sonnet-4-20250514` - Balanced performance/cost
- `claude-opus-4-20250514` - Highest capability
- `claude-3-5-sonnet-20241022` - Previous generation

**Features:**
- Prompt caching (automatic)
- Thinking blocks (automatic)
- Tool use (built-in)

### OpenAI (GPT)

```yaml
model_providers:
  openai:
    api_key: sk-...
    provider: openai

models:
  gpt_model:
    model_provider: openai
    model: gpt-4o  # or gpt-4o-mini, o1-preview
    max_tokens: 4096
    temperature: 0.5
    top_p: 0.95
    parallel_tool_calls: true
```

**Available Models:**
- `gpt-4o` - Latest GPT-4 optimized
- `gpt-4o-mini` - Faster, cheaper
- `o1-preview` - Reasoning model
- `o1-mini` - Smaller reasoning model

### Google (Gemini)

```yaml
model_providers:
  google:
    api_key: AI...
    provider: google

models:
  gemini_model:
    model_provider: google
    model: gemini-2.5-flash  # or gemini-2.5-pro
    max_tokens: 8192
    temperature: 0.5
    top_p: 0.95
    candidate_count: 1
```

**Available Models:**
- `gemini-2.5-flash` - Fast and efficient
- `gemini-2.5-pro` - Most capable
- `gemini-1.5-flash` - Previous generation
- `gemini-1.5-pro` - Previous generation pro

### Azure OpenAI

```yaml
model_providers:
  azure:
    api_key: your-key
    provider: azure
    base_url: https://your-resource.openai.azure.com
    api_version: 2024-02-15-preview

models:
  azure_model:
    model_provider: azure
    model: gpt-4o  # Your deployment name
    max_tokens: 4096
```

**Required:**
- `base_url` - Your Azure resource URL
- `api_version` - Azure API version
- `model` - Your deployment name (not the model name)

### Ollama (Local)

```yaml
model_providers:
  ollama:
    api_key: not-required
    provider: ollama
    base_url: http://localhost:11434

models:
  local_model:
    model_provider: ollama
    model: qwen3  # or llama3.1, codellama, etc.
    max_tokens: 4096
```

**Setup:**
```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull a model
ollama pull qwen3

# Run Trae Agent
trae-cli run "Task" --provider ollama --model qwen3
```

### OpenRouter

```yaml
model_providers:
  openrouter:
    api_key: sk-or-...
    provider: openai  # OpenRouter uses OpenAI-compatible API
    base_url: https://openrouter.ai/api/v1

models:
  openrouter_model:
    model_provider: openrouter
    model: "anthropic/claude-3-5-sonnet"  # Full model path
```

**Available Models:**
- `anthropic/claude-3-5-sonnet`
- `openai/gpt-4o`
- `google/gemini-2.5-flash`
- Many more at [openrouter.ai](https://openrouter.ai/models)

## MCP Server Configuration

### What is MCP?

Model Context Protocol (MCP) allows Trae Agent to use external tools and services beyond the built-in tools.

### Common MCP Servers

#### Playwright (Browser Automation)

```yaml
mcp_servers:
  playwright:
    command: npx
    args:
      - "@playwright/mcp@0.0.27"

agents:
  trae_agent:
    allow_mcp_servers:
      - playwright
```

**Usage:**
```bash
trae-cli run "Take a screenshot of example.com"
```

#### Filesystem

```yaml
mcp_servers:
  filesystem:
    command: npx
    args:
      - "@modelcontextprotocol/server-filesystem"
      - "/allowed/path"
      - "/another/allowed/path"
```

#### GitHub

```yaml
mcp_servers:
  github:
    command: npx
    args:
      - "@modelcontextprotocol/server-github"
    env:
      GITHUB_TOKEN: ghp_...
```

### Installing MCP Servers

Most MCP servers are npm packages:

```bash
# Install globally
npm install -g @playwright/mcp

# Or use npx (no installation needed)
# Trae Agent will use npx if the server isn't installed
```

## Advanced Settings

### Custom Tool Selection

```yaml
agents:
  trae_agent:
    tools:
      - bash
      - str_replace_based_edit_tool
      # Disable sequential thinking for faster execution
      # - sequentialthinking
      - task_done
```

### Multiple Model Configurations

```yaml
models:
  fast_model:
    model_provider: anthropic
    model: claude-sonnet-4-20250514
    temperature: 0.7
    max_tokens: 2048
  
  accurate_model:
    model_provider: anthropic
    model: claude-opus-4-20250514
    temperature: 0.3
    max_tokens: 8192
  
  local_model:
    model_provider: ollama
    model: qwen3
    temperature: 0.5

# Use different models for different tasks
agents:
  trae_agent:
    model: fast_model  # Default
```

Then override per-task:
```bash
trae-cli run "Simple task"  # Uses fast_model
trae-cli run "Complex task" --model accurate_model
```

### Retry Configuration

```yaml
models:
  robust_model:
    model_provider: anthropic
    model: claude-sonnet-4-20250514
    max_retries: 5  # Retry API calls up to 5 times
```

### Temperature and Sampling

```yaml
models:
  # Creative model (higher temperature)
  creative_model:
    temperature: 0.9
    top_p: 0.95
  
  # Deterministic model (lower temperature)
  precise_model:
    temperature: 0.1
    top_p: 0.9
  
  # Balanced model
  balanced_model:
    temperature: 0.5
    top_p: 0.95
```

## Configuration Examples

### Example 1: Multi-Provider Setup

```yaml
agents:
  trae_agent:
    model: default_model
    max_steps: 200
    tools: [bash, str_replace_based_edit_tool, sequentialthinking, task_done]

model_providers:
  anthropic:
    api_key: sk-ant-...
    provider: anthropic
  openai:
    api_key: sk-...
    provider: openai
  google:
    api_key: AI...
    provider: google

models:
  default_model:
    model_provider: anthropic
    model: claude-sonnet-4-20250514
    max_tokens: 4096
  
  fast_model:
    model_provider: google
    model: gemini-2.5-flash
    max_tokens: 8192
  
  powerful_model:
    model_provider: openai
    model: gpt-4o
    max_tokens: 4096
```

Usage:
```bash
trae-cli run "Task"  # Uses Claude
trae-cli run "Task" --model fast_model  # Uses Gemini
trae-cli run "Task" --model powerful_model  # Uses GPT-4o
```

### Example 2: Docker Development Setup

```yaml
agents:
  trae_agent:
    model: dev_model
    max_steps: 300
    tools: [bash, str_replace_based_edit_tool, json_edit_tool, task_done]

models:
  dev_model:
    model_provider: openai
    model: gpt-4o-mini
    temperature: 0.5
```

Usage:
```bash
trae-cli run "Run tests" --docker-image python:3.12
trae-cli run "Build app" --dockerfile-path ./Dockerfile
```

### Example 3: Research Configuration

```yaml
agents:
  trae_agent:
    enable_lakeview: false  # Full output for analysis
    model: research_model
    max_steps: 500
    tools:
      - bash
      - str_replace_based_edit_tool
      - sequentialthinking  # Important for reasoning
      - task_done

models:
  research_model:
    model_provider: anthropic
    model: claude-opus-4-20250514
    temperature: 0.3  # Lower temperature for consistency
    max_tokens: 8192
```

### Example 4: Production Configuration

```yaml
agents:
  trae_agent:
    enable_lakeview: true
    model: prod_model
    max_steps: 150
    tools: [bash, str_replace_based_edit_tool, task_done]

models:
  prod_model:
    model_provider: anthropic
    model: claude-sonnet-4-20250514
    temperature: 0.5
    max_tokens: 4096
    max_retries: 3
```

## Troubleshooting Configuration

### Verify Configuration

```bash
trae-cli show-config
```

### Common Issues

**Issue: "API key not found"**
```bash
# Check if set in config
cat trae_config.yaml | grep api_key

# Check environment variables
echo $ANTHROPIC_API_KEY

# Set explicitly
export ANTHROPIC_API_KEY="your-key"
```

**Issue: "Configuration file not found"**
```bash
# Check file exists
ls -la trae_config.yaml

# Use absolute path
trae-cli run "task" --config-file /full/path/to/config.yaml

# Or set environment variable
export TRAE_CONFIG_FILE=/full/path/to/config.yaml
```

**Issue: "Model not found"**
```yaml
# Ensure model is defined in models section
models:
  my_model:  # This name must match
    model_provider: anthropic
    model: claude-sonnet-4-20250514

agents:
  trae_agent:
    model: my_model  # Must reference the name above
```

## Related Documentation

- [Getting Started](GETTING_STARTED.md)
- [Architecture](ARCHITECTURE.md)
- [API Reference](API_REFERENCE.md)
- [Legacy Configuration](legacy_config.md)

## Summary

Trae Agent's configuration system is flexible and powerful:

- **Use YAML** for persistent, version-controlled configuration
- **Use environment variables** for sensitive data
- **Use CLI arguments** for one-off overrides
- **MCP servers** extend capabilities beyond built-in tools
- **Multiple models** can be defined and switched between

For most users, the recommended setup is:
1. Copy `trae_config.yaml.example` to `trae_config.yaml`
2. Add your API key to the appropriate provider section
3. Adjust `max_steps` and `tools` as needed
4. Start with `claude-sonnet-4-20250514` or `gpt-4o`
