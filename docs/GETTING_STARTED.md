# Getting Started with Trae Agent

Welcome to Trae Agent! This guide will help you get up and running quickly, from installation to executing your first task.

## Table of Contents

- [What is Trae Agent?](#what-is-trae-agent)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Your First Task](#your-first-task)
- [Understanding the Output](#understanding-the-output)
- [Common Use Cases](#common-use-cases)
- [Next Steps](#next-steps)

## What is Trae Agent?

Trae Agent is an LLM-powered assistant for software engineering tasks. It can:

- **Understand natural language instructions** and convert them into concrete actions
- **Navigate and analyze codebases** to understand structure and logic
- **Write, edit, and debug code** across multiple programming languages
- **Execute commands** to run tests, build projects, and verify changes
- **Generate documentation** and explain complex code
- **Work in isolated Docker environments** for safety and reproducibility

### How It Works

1. You provide a task in natural language (e.g., "Fix the authentication bug")
2. Trae Agent uses an LLM (like Claude or GPT-4) to understand the task
3. It explores your codebase using various tools (file editing, bash commands, etc.)
4. It makes changes, tests them, and iterates until the task is complete
5. All actions are logged in a trajectory file for review

## Prerequisites

Before installing Trae Agent, ensure you have:

### Required

1. **Python 3.12 or higher**
   ```bash
   python --version  # Should show 3.12+
   ```

2. **UV Package Manager** (recommended Python package installer)
   ```bash
   # Install UV (macOS/Linux)
   curl -LsSf https://astral.sh/uv/install.sh | sh
   
   # Verify installation
   uv --version
   ```

3. **API Key for an LLM Provider**
   - [OpenAI API Key](https://platform.openai.com/api-keys) for GPT models
   - [Anthropic API Key](https://console.anthropic.com/) for Claude models
   - [Google AI Studio](https://aistudio.google.com/app/apikey) for Gemini models
   - Or other supported providers (Azure, Doubao, Ollama, OpenRouter)

### Optional

- **Docker** (if you want to run tasks in isolated containers)
  ```bash
  docker --version  # Verify Docker is installed
  ```

- **Git** (for working with repositories)
  ```bash
  git --version
  ```

## Installation

### Step 1: Clone the Repository

```bash
git clone https://github.com/bytedance/trae-agent.git
cd trae-agent
```

### Step 2: Install Dependencies

Using UV (recommended):
```bash
# Install all dependencies including optional test tools
uv sync --all-extras

# Activate the virtual environment
source .venv/bin/activate
```

Alternative using pip:
```bash
# Create a virtual environment
python -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install the package
pip install -e .
```

### Step 3: Verify Installation

```bash
# Check if trae-cli is available
trae-cli --version

# Should output: Trae Agent, version 0.1.0
```

If you see "command not found", make sure your virtual environment is activated.

## Configuration

### Quick Configuration (YAML)

1. **Copy the example configuration:**
   ```bash
   cp trae_config.yaml.example trae_config.yaml
   ```

2. **Edit `trae_config.yaml` with your API key:**
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
       api_key: your_anthropic_api_key_here  # Replace with your key
       provider: anthropic

   models:
     trae_agent_model:
       model_provider: anthropic
       model: claude-sonnet-4-20250514
       max_tokens: 4096
       temperature: 0.5
   ```

3. **Important**: Replace `your_anthropic_api_key_here` with your actual API key

### Alternative: Environment Variables

You can also set API keys via environment variables:

```bash
# Add to your ~/.bashrc, ~/.zshrc, or create a .env file
export ANTHROPIC_API_KEY="your-api-key-here"
export OPENAI_API_KEY="your-openai-key-here"
export GOOGLE_API_KEY="your-google-key-here"
```

### Verify Configuration

```bash
trae-cli show-config
```

This will display your current configuration, showing which provider, model, and tools are configured.

## Your First Task

Let's start with a simple task to verify everything works.

### Example 1: Create a Hello World Script

```bash
# Create a test directory
mkdir ~/trae-test
cd ~/trae-test

# Run your first task
trae-cli run "Create a Python script that prints 'Hello, World!' and saves it as hello.py"
```

**What happens:**
1. Trae Agent receives your task
2. It uses the LLM to understand what needs to be done
3. It creates a file called `hello.py` with the appropriate code
4. It verifies the file was created successfully
5. It reports completion

**Check the result:**
```bash
cat hello.py
# Should show: print('Hello, World!')

python hello.py
# Should output: Hello, World!
```

### Example 2: Add a Feature to Existing Code

Create a simple file first:
```bash
cat > calculator.py << 'EOF'
def add(a, b):
    return a + b

def subtract(a, b):
    return a - b
EOF
```

Now ask Trae Agent to extend it:
```bash
trae-cli run "Add multiply and divide functions to calculator.py"
```

**Check the result:**
```bash
cat calculator.py
# Should now include multiply() and divide() functions
```

### Example 3: Debug and Fix Code

Create a file with a bug:
```bash
cat > buggy.py << 'EOF'
def calculate_average(numbers):
    total = sum(numbers)
    return total / len(numbers)

# This will crash with empty list
result = calculate_average([])
print(result)
EOF
```

Ask Trae Agent to fix it:
```bash
trae-cli run "Fix the bug in buggy.py where it crashes with an empty list"
```

## Understanding the Output

### Console Output

When you run a task, you'll see output like:

```
╭─────────────────── Task Details ────────────────────╮
│ Task: Create a Python script...                     │
│ Model Provider: anthropic                           │
│ Model: claude-sonnet-4-20250514                     │
│ Max Steps: 200                                      │
│ Tools: bash, str_replace_based_edit_tool, ...       │
╰─────────────────────────────────────────────────────╯

Step 1: [Thinking]
• Creating a new Python file hello.py

Step 2: [Calling Tool: str_replace_based_edit_tool]
• Creating file with Hello World code

Step 3: [Thinking]
• File created successfully

Step 4: [Calling Tool: task_done]
• Task completed

✓ Trajectory saved to: trajectories/trajectory_20250108_101530.json
```

### Lakeview Mode

If `enable_lakeview: true` in your config, you'll see concise summaries:

```
📝 Creating hello.py with print statement
✓ File created successfully
```

### Trajectory Files

Every execution creates a trajectory file in the `trajectories/` directory. This JSON file contains:

- Complete conversation with the LLM
- All tool calls and results
- Timing information
- Token usage
- Final outcome

**View a trajectory:**
```bash
cat trajectories/trajectory_20250108_101530.json | jq .
```

This is useful for:
- Debugging failed tasks
- Understanding agent decision-making
- Analyzing token usage
- Compliance and auditing

## Common Use Cases

### Use Case 1: Code Generation

```bash
# Create a complete module
trae-cli run "Create a user authentication module with login, logout, and password hashing"

# Generate tests
trae-cli run "Create pytest tests for the authentication module"

# Add documentation
trae-cli run "Add docstrings to all functions in auth.py following Google style"
```

### Use Case 2: Debugging

```bash
# Find and fix a bug
trae-cli run "Debug why the login function returns None instead of a user object"

# Reproduce an issue
trae-cli run "Create a script that reproduces the issue described in issue #42"

# Add error handling
trae-cli run "Add proper error handling to all database operations in models.py"
```

### Use Case 3: Refactoring

```bash
# Improve code structure
trae-cli run "Refactor the payment processing logic into separate functions"

# Update to new patterns
trae-cli run "Update all class-based views to use async/await syntax"

# Remove dead code
trae-cli run "Remove all unused imports and variables from the project"
```

### Use Case 4: Documentation

```bash
# Generate README
trae-cli run "Create a comprehensive README.md for this project"

# Add code comments
trae-cli run "Add comments explaining the algorithm in sorting.py"

# Create API docs
trae-cli run "Generate API documentation for all public functions"
```

### Use Case 5: Testing

```bash
# Write tests
trae-cli run "Create unit tests for the UserService class with 100% coverage"

# Run and fix failing tests
trae-cli run "Run the test suite and fix any failing tests"

# Add edge cases
trae-cli run "Add test cases for edge cases in the validation logic"
```

## Advanced Features

### Using Different Models

```bash
# Use GPT-4o
trae-cli run "Analyze this codebase" --provider openai --model gpt-4o

# Use Gemini
trae-cli run "Optimize this algorithm" --provider google --model gemini-2.5-flash

# Use local Ollama
trae-cli run "Add comments to this code" --provider ollama --model qwen3
```

### Working in Docker

```bash
# Run in a Python container
trae-cli run "Run the tests" --docker-image python:3.12

# Build from Dockerfile
trae-cli run "Test the application" --dockerfile-path ./Dockerfile

# Attach to existing container
trae-cli run "Debug the server" --docker-container-id abc123
```

### Interactive Mode

For multiple tasks in sequence:

```bash
trae-cli interactive

# Now you can enter tasks one at a time:
> Add a new feature to utils.py
> Run the tests
> Fix any errors
> exit
```

### Custom Working Directory

```bash
# Work in a specific directory
trae-cli run "Update the database schema" --working-dir /path/to/project

# With custom trajectory file
trae-cli run "Refactor the API" --working-dir ~/myproject --trajectory-file refactor.json
```

## Troubleshooting

### "Command not found: trae-cli"

**Solution:**
```bash
# Make sure virtual environment is activated
source .venv/bin/activate

# Or use uv run
uv run trae-cli run "your task"
```

### "API key not found"

**Solution:**
```bash
# Check your configuration
trae-cli show-config

# Verify API key is set
echo $ANTHROPIC_API_KEY

# Or set it in trae_config.yaml
```

### "Permission denied"

**Solution:**
```bash
# Ensure working directory has proper permissions
chmod +x /path/to/project
```

### Agent seems stuck

**Solution:**
- Press `Ctrl+C` to interrupt
- Check the trajectory file to see where it stopped
- Try simplifying your task description
- Increase `max_steps` in configuration if needed

### Docker issues

**Solution:**
```bash
# Check Docker is running
docker ps

# Verify Docker access
docker run hello-world
```

## Best Practices

### 1. Clear Task Descriptions

✓ Good: "Add error handling to the login function in auth.py to catch ValueError and return a meaningful error message"

✗ Bad: "Fix the login thing"

### 2. One Task at a Time

✓ Good: Run separate tasks for "add feature" and "write tests"

✗ Bad: "Add feature X, Y, Z and write all tests and update docs and refactor everything"

### 3. Verify Changes

Always review the changes Trae Agent made:
```bash
git diff  # See what changed
```

### 4. Use Version Control

Work in a git repository so you can easily revert changes:
```bash
git init
git add .
git commit -m "Before Trae Agent changes"
# Run Trae Agent
git diff  # Review changes
```

### 5. Start Small

Begin with simple tasks to understand how Trae Agent works, then gradually tackle more complex tasks.

## Next Steps

Now that you're up and running, explore these topics:

1. **[Architecture](ARCHITECTURE.md)** - Understand how Trae Agent works internally
2. **[Configuration Guide](CONFIGURATION.md)** - Advanced configuration options
3. **[Tools Documentation](tools.md)** - Learn about available tools
4. **[Workflow Guide](WORKFLOW.md)** - Best practices for complex tasks
5. **[Docker Usage](DOCKER.md)** - Run tasks in isolated environments
6. **[API Reference](API_REFERENCE.md)** - Programmatic usage
7. **[Contributing](../CONTRIBUTING.md)** - Help improve Trae Agent

## Getting Help

- **Documentation**: Check the [docs folder](.) for detailed guides
- **Issues**: Report bugs or request features on [GitHub Issues](https://github.com/bytedance/trae-agent/issues)
- **Discord**: Join the [Trae Agent Discord](https://discord.gg/VwaQ4ZBHvC) for community support
- **Examples**: See the `examples/` directory for more use cases

Happy coding with Trae Agent! 🚀
