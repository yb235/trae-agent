# Trae Agent Documentation

Welcome to the Trae Agent documentation! This guide will help you understand and effectively use Trae Agent for software engineering tasks.

## 📚 Documentation Index

### Getting Started

- **[Getting Started Guide](GETTING_STARTED.md)** - Installation, configuration, and your first task
  - Prerequisites and installation
  - Quick configuration
  - Running your first task
  - Common use cases
  - Troubleshooting basics

### Core Concepts

- **[Architecture](ARCHITECTURE.md)** - System design and component overview
  - Design principles
  - System architecture diagram
  - Core components (Agent, LLM Client, Tools)
  - Data flow and execution model
  - Extension points

- **[Technical Design](TECHNICAL_DESIGN.md)** - How Trae Agent overcomes LLM limitations
  - Handling LLM output token limits
  - Code editing without full regeneration
  - View range pagination system
  - String replacement architecture
  - Response truncation mechanism

- **[Workflow Guide](WORKFLOW.md)** - How Trae Agent processes tasks
  - Agent lifecycle
  - Execution phases
  - Decision-making process
  - Tool selection and usage
  - Error handling and recovery

### Configuration & Setup

- **[Configuration Guide](CONFIGURATION.md)** - Comprehensive configuration reference
  - YAML configuration
  - Environment variables
  - Command-line arguments
  - Provider-specific configuration
  - MCP server configuration
  - Advanced settings

### Tools & Features

- **[Tools Documentation](tools.md)** - Available tools and their usage
  - bash - Shell command execution
  - str_replace_based_edit_tool - File editing
  - json_edit_tool - JSON manipulation
  - sequential_thinking - Structured reasoning
  - task_done - Completion signaling

- **[Docker Usage](DOCKER.md)** - Running tasks in Docker containers
  - Why use Docker
  - Docker modes (image, Dockerfile, container)
  - Basic and advanced usage
  - How it works
  - Best practices and troubleshooting

- **[Trajectory Recording](TRAJECTORY_RECORDING.md)** - Execution logging and analysis
  - What is recorded
  - Usage and file format
  - Analysis and debugging

### Development

- **[API Reference](API_REFERENCE.md)** - Programmatic usage
  - Agent API
  - LLM Client API
  - Tools API
  - Configuration API
  - Trajectory Recording API
  - Usage examples

- **[Examples and Use Cases](EXAMPLES.md)** - Real-world examples
  - Code generation
  - Bug fixing
  - Testing
  - Refactoring
  - Documentation
  - Code review
  - Migration
  - DevOps

### Additional Resources

- **[Roadmap](roadmap.md)** - Future plans and features
- **[Legacy Configuration](legacy_config.md)** - JSON configuration (deprecated)
- **[Contributing Guide](../CONTRIBUTING.md)** - How to contribute

## 🚀 Quick Links

### For First-Time Users

1. Start with [Getting Started Guide](GETTING_STARTED.md)
2. Learn about [Tools](tools.md) available
3. Try [Examples](EXAMPLES.md) to see what's possible

### For Developers

1. Read [Architecture](ARCHITECTURE.md) to understand the system
2. Check [API Reference](API_REFERENCE.md) for programmatic usage
3. Review [Contributing Guide](../CONTRIBUTING.md) to contribute

### For Advanced Users

1. Deep dive into [Workflow Guide](WORKFLOW.md)
2. Explore [Configuration Guide](CONFIGURATION.md) for customization
3. Use [Docker](DOCKER.md) for isolated execution

## 📖 Documentation by Use Case

### "I want to get started quickly"
→ [Getting Started Guide](GETTING_STARTED.md)

### "I want to understand how it works"
→ [Architecture](ARCHITECTURE.md), [Technical Design](TECHNICAL_DESIGN.md), and [Workflow Guide](WORKFLOW.md)

### "I want to configure for my needs"
→ [Configuration Guide](CONFIGURATION.md)

### "I want to use specific features"
→ [Tools Documentation](tools.md), [Docker Usage](DOCKER.md)

### "I want to use it programmatically"
→ [API Reference](API_REFERENCE.md)

### "I want to see examples"
→ [Examples and Use Cases](EXAMPLES.md)

### "I want to debug or analyze"
→ [Trajectory Recording](TRAJECTORY_RECORDING.md)

## 🎯 Key Features

### Multi-LLM Support
Configure once, use with any provider:
- Anthropic (Claude)
- OpenAI (GPT)
- Google (Gemini)
- Azure OpenAI
- Ollama (local models)
- OpenRouter (multi-provider access)

**Learn more:** [Configuration Guide](CONFIGURATION.md)

### Rich Tool Ecosystem
Powerful tools for software engineering:
- File editing and creation
- Bash command execution
- JSON manipulation
- Structured reasoning
- And more...

**Learn more:** [Tools Documentation](tools.md)

### Docker Integration
Safe, isolated execution:
- Run in containers
- Multiple environments
- Reproducible builds

**Learn more:** [Docker Usage](DOCKER.md)

### Trajectory Recording
Complete execution logging:
- LLM interactions
- Tool calls and results
- Timing and token usage

**Learn more:** [Trajectory Recording](TRAJECTORY_RECORDING.md)

### Lakeview Mode
Concise, readable output:
- Summary of each step
- Key actions highlighted
- Reduced verbosity

**Learn more:** [Getting Started Guide](GETTING_STARTED.md)

## 💡 Common Tasks

### Generate Code
```bash
trae-cli run "Create a REST API endpoint for user management"
```
→ See [Examples: Code Generation](EXAMPLES.md#code-generation)

### Fix Bugs
```bash
trae-cli run "Fix the authentication bug in auth.py"
```
→ See [Examples: Bug Fixing](EXAMPLES.md#bug-fixing)

### Write Tests
```bash
trae-cli run "Create pytest tests for the UserService class"
```
→ See [Examples: Testing](EXAMPLES.md#testing)

### Refactor Code
```bash
trae-cli run "Refactor the payment processing logic"
```
→ See [Examples: Refactoring](EXAMPLES.md#refactoring)

### Generate Documentation
```bash
trae-cli run "Create comprehensive API documentation"
```
→ See [Examples: Documentation](EXAMPLES.md#documentation)

## 🔍 Understanding Trae Agent

### How It Works

1. **Understand**: Agent receives your task and understands requirements
2. **Explore**: Navigates codebase to gather context
3. **Plan**: Develops a strategy using structured reasoning
4. **Execute**: Makes changes using available tools
5. **Verify**: Tests and validates the solution
6. **Complete**: Signals successful completion

**Learn more:** [Workflow Guide](WORKFLOW.md)

### Architecture Overview

```
CLI → Agent → LLM Client → Provider (Anthropic/OpenAI/etc.)
        ↓
      Tools → bash, edit_tool, json_edit, etc.
        ↓
   Trajectory Recorder → Logs all actions
```

**Learn more:** [Architecture](ARCHITECTURE.md)

## 📊 What Makes Trae Agent Different?

### Research-Friendly Design
- **Transparent**: All actions logged in trajectories
- **Modular**: Components can be modified independently
- **Extensible**: Easy to add new tools and providers
- **Analyzable**: Detailed execution data for research

**Learn more:** [Architecture](ARCHITECTURE.md)

### Production-Ready Features
- **Multi-provider support**: Not locked to one LLM
- **Docker isolation**: Safe execution
- **Error handling**: Robust recovery mechanisms
- **Configurable**: Adapt to your needs

**Learn more:** [Configuration Guide](CONFIGURATION.md)

## 🛠️ Development Resources

### For Contributors
- [Contributing Guide](../CONTRIBUTING.md)
- [Architecture](ARCHITECTURE.md)
- [API Reference](API_REFERENCE.md)

### For Researchers
- [Architecture](ARCHITECTURE.md) - System design
- [Technical Design](TECHNICAL_DESIGN.md) - LLM limitation solutions
- [Workflow Guide](WORKFLOW.md) - Execution model
- [Trajectory Recording](TRAJECTORY_RECORDING.md) - Data collection

### For Integration
- [API Reference](API_REFERENCE.md) - Programmatic usage
- [Configuration Guide](CONFIGURATION.md) - Setup options

## 🆘 Getting Help

### Documentation
Start here! Most questions are answered in:
- [Getting Started](GETTING_STARTED.md)
- [Configuration](CONFIGURATION.md)
- [Examples](EXAMPLES.md)

### Community
- **Discord**: [Join our Discord](https://discord.gg/VwaQ4ZBHvC)
- **GitHub Issues**: [Report bugs or request features](https://github.com/bytedance/trae-agent/issues)
- **Discussions**: [Ask questions](https://github.com/bytedance/trae-agent/discussions)

### Technical Report
For academic details, see our [technical report](https://arxiv.org/abs/2507.23370)

## 📝 Documentation Status

| Document | Status | Last Updated |
|----------|--------|--------------|
| Getting Started | ✅ Complete | 2025-01-08 |
| Architecture | ✅ Complete | 2025-01-08 |
| Technical Design | ✅ Complete | 2025-01-08 |
| Workflow Guide | ✅ Complete | 2025-01-08 |
| Configuration | ✅ Complete | 2025-01-08 |
| Tools | ✅ Complete | Existing |
| Docker Usage | ✅ Complete | 2025-01-08 |
| API Reference | ✅ Complete | 2025-01-08 |
| Examples | ✅ Complete | 2025-01-08 |
| Trajectory Recording | ✅ Complete | Existing |
| Roadmap | ✅ Complete | Existing |

## 🔄 Keeping Up to Date

The documentation is continuously updated as Trae Agent evolves. Check the [GitHub repository](https://github.com/bytedance/trae-agent) for the latest version.

## 📄 License

Trae Agent is licensed under the MIT License. See [LICENSE](../LICENSE) for details.

---

**Ready to get started?** Begin with the [Getting Started Guide](GETTING_STARTED.md)!

**Have questions?** Check [Examples](EXAMPLES.md) or join our [Discord](https://discord.gg/VwaQ4ZBHvC)!

**Want to contribute?** Read the [Contributing Guide](../CONTRIBUTING.md)!
