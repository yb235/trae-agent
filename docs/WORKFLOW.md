# Trae Agent Workflow Guide

This guide explains how Trae Agent processes tasks, makes decisions, and completes software engineering work. Understanding this workflow will help you write better task descriptions and debug issues when they occur.

## Table of Contents

- [Overview](#overview)
- [Agent Lifecycle](#agent-lifecycle)
- [Execution Phases](#execution-phases)
- [Decision-Making Process](#decision-making-process)
- [Tool Selection and Usage](#tool-selection-and-usage)
- [Error Handling and Recovery](#error-handling-and-recovery)
- [Best Practices](#best-practices)
- [Common Patterns](#common-patterns)

## Overview

Trae Agent follows a structured workflow when processing tasks:

```
Task Input → Understanding → Exploration → Planning → 
Execution → Verification → Completion
```

Each phase involves interaction between the LLM and various tools, with all actions logged in the trajectory for transparency.

## Agent Lifecycle

### 1. Initialization

When you run a task, Trae Agent:

```python
# Configuration is loaded
config = Config.create(config_file="trae_config.yaml")

# Agent is created
agent = Agent(agent_type="trae_agent", config=config)

# Tools are registered
tools = [bash, edit_tool, sequential_thinking, task_done]

# LLM client is initialized
llm_client = LLMClient(model_config)

# Trajectory recording begins
trajectory_recorder.start_recording(task)
```

### 2. Task Setup

The agent receives:
- **Task description**: Your natural language instruction
- **Project path**: Working directory (current dir or specified with `--working-dir`)
- **System prompt**: Instructions on how to behave as a software engineer
- **Available tools**: The toolkit for accomplishing the task

### 3. Execution Loop

The agent enters a step-by-step execution loop:

```python
while not task_complete and step_count < max_steps:
    # 1. Think about next action
    response = llm_client.chat(messages, tools)
    
    # 2. Execute tool calls if any
    if response.tool_calls:
        results = execute_tools(response.tool_calls)
        messages.append(tool_results)
    
    # 3. Record the step
    trajectory_recorder.record_agent_step(step)
    
    # 4. Check for completion
    if "task_done" in tool_calls:
        task_complete = True
```

### 4. Finalization

When the task completes:
- Final trajectory is written to disk
- Docker containers are cleaned up (if used)
- MCP servers are disconnected (if used)
- Summary is displayed to user

## Execution Phases

### Phase 1: Understanding

**Goal**: Comprehend the task requirements

**Actions**:
- Parse the task description
- Identify key requirements
- Determine success criteria
- Plan initial approach

**Example**:
```
Task: "Fix the authentication bug in login.py"

Agent thinks:
- Need to find login.py
- Need to understand what the bug is
- Need to reproduce the bug
- Need to implement a fix
- Need to verify the fix works
```

**Tools Used**:
- `sequential_thinking` - Break down the problem
- `bash` - Check if files exist

### Phase 2: Exploration

**Goal**: Understand the codebase context

**Actions**:
- Locate relevant files
- Read code structure
- Understand dependencies
- Identify related components

**Example**:
```
Agent actions:
1. bash: ls -la
2. str_replace_based_edit_tool: view /project/src/login.py
3. str_replace_based_edit_tool: view /project/tests/test_login.py
4. bash: grep -r "authentication" --include="*.py"
```

**Tools Used**:
- `bash` - List files, search code
- `str_replace_based_edit_tool` (view) - Read file contents

### Phase 3: Planning

**Goal**: Develop a strategy to complete the task

**Actions**:
- Analyze gathered information
- Consider multiple approaches
- Select the best approach
- Break down into steps

**Example**:
```
Agent thinks:
"I've identified the bug. The login function doesn't handle
None values from the database query. I'll:
1. Add a None check
2. Return an appropriate error
3. Add tests for this case
4. Run tests to verify"
```

**Tools Used**:
- `sequential_thinking` - Structured reasoning

### Phase 4: Execution

**Goal**: Implement the solution

**Actions**:
- Make code changes
- Create new files if needed
- Run commands
- Iteratively refine

**Example**:
```
Agent actions:
1. str_replace_based_edit_tool: str_replace in /project/src/login.py
   Old: return user
   New: return user if user else None

2. bash: python -m pytest tests/test_login.py -v

3. (if tests fail) Make additional fixes

4. bash: python -m pytest tests/test_login.py -v
```

**Tools Used**:
- `str_replace_based_edit_tool` (str_replace, insert, create) - Modify files
- `bash` - Run tests, build, lint
- `json_edit_tool` - Modify JSON/config files

### Phase 5: Verification

**Goal**: Ensure the solution works correctly

**Actions**:
- Run tests
- Execute verification scripts
- Check for regressions
- Validate edge cases

**Example**:
```
Agent actions:
1. bash: python -m pytest tests/ -v
2. bash: python -m pytest tests/test_login.py::test_empty_credentials
3. bash: python src/login.py  # Manual verification if needed
```

**Tools Used**:
- `bash` - Run test suites, linters, type checkers
- `str_replace_based_edit_tool` - Create test scripts

### Phase 6: Completion

**Goal**: Signal successful task completion

**Actions**:
- Verify all requirements met
- Generate summary (optional)
- Signal completion

**Example**:
```
Agent thinks:
"I've fixed the authentication bug by adding proper None handling,
added test cases, and verified all tests pass. Task complete."

Agent action:
- task_done
```

**Tools Used**:
- `task_done` - Signal completion

## Decision-Making Process

### How the Agent Chooses Actions

The LLM receives:

1. **System Prompt**: Instructions on being a software engineer
2. **Task Description**: What to accomplish
3. **Previous Actions**: What has been done so far
4. **Tool Results**: Output from previous tool calls
5. **Available Tools**: What actions are possible

Based on this context, the LLM decides:
- What to do next
- Which tool(s) to use
- What parameters to provide

### Sequential Thinking

For complex decisions, the agent uses the `sequential_thinking` tool:

```
Thought 1: "I need to understand where the bug is"
Thought 2: "Let me check the login function first"
Thought 3: "The issue is in the null check. I can fix this by..."
Thought 4: "Wait, I should also check if this affects other functions"
Thought 5: "After analysis, the fix should be isolated to login()"
```

This structured thinking helps with:
- Complex debugging
- Architecture decisions
- Multiple solution paths
- Uncertain situations

### Tool Call Patterns

**Sequential Tools**:
```python
# Agent calls tools one at a time
step_1: bash("ls -la")
step_2: view("/path/file.py")
step_3: str_replace(...)
```

**Parallel Tools** (when supported):
```python
# Agent calls multiple tools simultaneously
parallel:
  - view("/path/file1.py")
  - view("/path/file2.py")
  - bash("grep pattern *.py")
```

## Tool Selection and Usage

### When to Use Each Tool

#### bash
**Use for:**
- Listing files and directories
- Searching code (grep, find)
- Running tests
- Building projects
- Installing dependencies
- Checking command output

**Example tasks:**
- "Run the test suite"
- "Find all files containing 'TODO'"
- "Install missing dependencies"

#### str_replace_based_edit_tool
**Use for:**
- Reading file contents
- Creating new files
- Editing existing code
- Inserting new code
- Viewing directory structure

**Example tasks:**
- "Add a new function to utils.py"
- "Fix the typo in README.md"
- "Create a new module"

#### json_edit_tool
**Use for:**
- Modifying JSON configuration files
- Updating package.json
- Changing settings in JSON
- Adding/removing JSON properties

**Example tasks:**
- "Update the version in package.json"
- "Add a new dependency"
- "Change the database host in config.json"

#### sequential_thinking
**Use for:**
- Breaking down complex problems
- Considering multiple approaches
- Debugging difficult issues
- Architecture decisions

**Example tasks:**
- "Design a caching strategy"
- "Debug why tests are flaky"
- "Choose between approaches A and B"

#### task_done
**Use for:**
- Signaling successful completion
- After verification is complete

**Only when:**
- All requirements are met
- Tests pass
- Changes are verified

### Tool Usage Patterns

#### Pattern 1: Explore → Edit → Verify

```
1. bash: ls src/
2. view: /project/src/auth.py
3. str_replace: Fix the bug in auth.py
4. bash: pytest tests/test_auth.py
5. task_done
```

#### Pattern 2: Search → Analyze → Modify

```
1. bash: grep -r "deprecated_function" .
2. sequential_thinking: Analyze all usages
3. str_replace: Update call in file1.py
4. str_replace: Update call in file2.py
5. bash: Run tests
6. task_done
```

#### Pattern 3: Create → Test → Iterate

```
1. create: /project/new_module.py
2. bash: python -m pytest tests/
3. (if failed) str_replace: Fix issue
4. bash: python -m pytest tests/
5. task_done
```

## Error Handling and Recovery

### Types of Errors

#### 1. Tool Execution Errors

**Example**: File not found
```
Tool: str_replace_based_edit_tool
Error: File /project/missing.py does not exist
```

**Recovery**:
- Agent recognizes the error
- Adjusts approach (e.g., create file first)
- Retries with corrected action

#### 2. Command Failures

**Example**: Test failure
```
Tool: bash
Command: pytest tests/
Error: 2 tests failed
```

**Recovery**:
- Agent reads test output
- Understands what went wrong
- Makes fixes
- Reruns tests

#### 3. Syntax Errors

**Example**: Invalid code
```
Tool: str_replace_based_edit_tool
Result: File updated

Tool: bash
Command: python -m pytest
Error: SyntaxError in modified file
```

**Recovery**:
- Agent detects syntax error
- Reviews the code change
- Fixes the syntax issue
- Verifies fix

#### 4. Logic Errors

**Example**: Wrong behavior
```
Tool: bash
Command: python test_script.py
Output: Expected 42, got 24
```

**Recovery**:
- Agent analyzes logic
- Uses sequential_thinking to debug
- Identifies root cause
- Implements correct logic

### Recovery Strategies

#### Strategy 1: Incremental Fixes

Make small changes and verify each:
```
1. Make minimal change
2. Test
3. If failed, revert and try differently
4. If succeeded, continue
```

#### Strategy 2: Gather More Context

When stuck:
```
1. Read more related files
2. Check documentation
3. Run diagnostic commands
4. Use sequential_thinking to analyze
```

#### Strategy 3: Alternative Approaches

If one approach fails repeatedly:
```
1. sequential_thinking: Consider alternatives
2. Try a different approach
3. If successful, continue
4. If failed, try next alternative
```

### Max Steps Protection

If the agent reaches `max_steps` without completion:
- Execution stops
- Trajectory is saved
- Partial work is preserved
- User can review and continue

**Solutions:**
- Increase `max_steps` in config
- Break task into smaller pieces
- Provide more specific instructions

## Best Practices

### 1. Write Clear Task Descriptions

✓ **Good**:
```
"Add input validation to the signup form in signup.js to check:
- Email format is valid
- Password is at least 8 characters
- Username contains no special characters
Then add tests for these validations"
```

✗ **Bad**:
```
"Fix the signup thing"
```

### 2. Provide Context

✓ **Good**:
```
"The authentication tests in tests/test_auth.py are failing
because the mock database returns None. Fix the login function
in src/auth.py to handle this case"
```

✗ **Bad**:
```
"Tests are broken, fix them"
```

### 3. Set Appropriate Scope

✓ **Good**: "Refactor the payment processing logic in payment.py into separate functions"

✗ **Bad**: "Refactor the entire codebase"

### 4. Enable Verification

Include verification steps in your task:
```
"Add the logging feature to database.py and write tests to
verify logs are created correctly"
```

### 5. Use Working Directories

```bash
# Good: Work in the specific project
trae-cli run "Add feature X" --working-dir ~/projects/myapp

# Avoid: Running in wrong directory
cd /tmp
trae-cli run "Add feature X"  # Oops, wrong place!
```

### 6. Review Trajectories

After execution:
```bash
# Review what the agent did
cat trajectories/trajectory_*.json | jq .

# Check for issues:
# - Did it take too many steps?
# - Did it make errors?
# - How much did it cost (token usage)?
```

### 7. Use Version Control

```bash
git status  # Before running agent
trae-cli run "Your task"
git diff    # Review changes
git add .   # If good
git commit  # Save work
```

## Common Patterns

### Pattern: Test-Driven Fix

```
Task: "Fix bug #123 in the authentication system"

Workflow:
1. Create reproduction script
2. Run script to confirm bug
3. Analyze root cause
4. Implement fix
5. Run script to verify fix
6. Run full test suite
7. Complete
```

### Pattern: Feature Addition

```
Task: "Add password strength meter to signup page"

Workflow:
1. Locate signup page file
2. Understand current structure
3. Create strength meter function
4. Integrate into signup form
5. Add styling/UI elements
6. Write tests
7. Run tests
8. Complete
```

### Pattern: Refactoring

```
Task: "Refactor the database module to use async/await"

Workflow:
1. Review current implementation
2. Plan migration strategy
3. Update one function at a time
4. Run tests after each change
5. Update all call sites
6. Run full test suite
7. Update documentation
8. Complete
```

### Pattern: Documentation

```
Task: "Generate API documentation for the user service"

Workflow:
1. Read through all API functions
2. Understand parameters and returns
3. Generate docstrings for each function
4. Create overview documentation
5. Add usage examples
6. Verify formatting
7. Complete
```

## Debugging Workflows

### When the Agent Gets Stuck

**Symptoms:**
- Repeating the same action
- Not making progress
- Hitting max_steps

**Solutions:**

1. **Check the trajectory** to see where it's looping:
   ```bash
   cat trajectories/latest.json | jq '.agent_steps[] | {step: .step_number, state: .state}'
   ```

2. **Simplify the task**:
   ```bash
   # Instead of: "Fix all bugs and add all features"
   # Try: "Fix the login bug in auth.py"
   ```

3. **Provide more context**:
   ```bash
   # Instead of: "Fix the test"
   # Try: "Fix test_login in tests/test_auth.py which fails because mock_db returns None"
   ```

4. **Increase max_steps** if task is legitimately complex:
   ```yaml
   # in trae_config.yaml
   agents:
     trae_agent:
       max_steps: 300  # Increase from 200
   ```

### When Changes Are Incorrect

**Review before applying:**
```bash
# Run in a git repository
git init  # If not already a repo
git add .
git commit -m "Before Trae Agent"

trae-cli run "Your task"

# Review changes
git diff

# If bad, revert
git reset --hard HEAD

# If good, commit
git add .
git commit -m "Changes by Trae Agent"
```

## Related Documentation

- [Architecture](ARCHITECTURE.md) - System design and components
- [Tools Documentation](tools.md) - Detailed tool descriptions
- [Configuration Guide](CONFIGURATION.md) - Advanced configuration
- [API Reference](API_REFERENCE.md) - Programmatic usage
- [Getting Started](GETTING_STARTED.md) - Installation and first steps

## Summary

Trae Agent follows a structured workflow:
1. **Understand** the task
2. **Explore** the codebase
3. **Plan** the solution
4. **Execute** the changes
5. **Verify** correctness
6. **Complete** the task

Understanding this workflow helps you:
- Write better task descriptions
- Anticipate agent behavior
- Debug issues effectively
- Get better results

For complex tasks, the agent uses `sequential_thinking` to reason through multiple steps and alternatives, ensuring high-quality solutions.
