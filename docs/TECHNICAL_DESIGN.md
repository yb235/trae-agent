# Technical Design: Overcoming LLM Limitations

This document provides an in-depth technical explanation of how Trae Agent overcomes LLM output limitations and efficiently handles code editing without requiring full codebase regeneration.

## Table of Contents

- [Overview](#overview)
- [Challenge 1: LLM Output Token Limits](#challenge-1-llm-output-token-limits)
- [Challenge 2: Code Editing Without Full Regeneration](#challenge-2-code-editing-without-full-regeneration)
- [Response Truncation System](#response-truncation-system)
- [The Edit Tool Architecture](#the-edit-tool-architecture)
- [Workflow: From Problem to Solution](#workflow-from-problem-to-solution)
- [Benefits and Trade-offs](#benefits-and-trade-offs)

## Overview

Trae Agent faces two critical challenges when working with large codebases:

1. **LLM Output Limits**: Language models have maximum output token limits (typically 4,096-16,384 tokens), making it impossible to output entire large files
2. **Code Regeneration Problem**: Regenerating entire files leads to errors, high token costs, and introduces inconsistencies

Trae Agent solves both challenges through a sophisticated combination of:
- **View Range Pagination** - View only relevant portions of files
- **Response Truncation** - Automatically truncate large outputs with hints
- **Surgical Editing Tools** - Edit specific lines without rewriting entire files
- **String Replacement Editing** - Make precise changes by matching existing code

## Challenge 1: LLM Output Token Limits

### The Problem

Language models have strict output token limits:
- GPT-4o: 4,096 tokens (default), up to 16,384 tokens
- Claude Sonnet 4: 4,096 tokens (default), up to 8,192 tokens
- Gemini 2.5 Flash: 8,192 tokens (default)

A typical source file can easily exceed these limits:
- 1,000 lines of code ≈ 8,000-12,000 tokens
- Large files with 5,000+ lines ≈ 40,000-60,000 tokens

**If Trae Agent tried to output entire files, it would:**
- Hit token limits and get truncated mid-output
- Produce incomplete, broken code
- Waste tokens on unchanged code
- Dramatically increase API costs

### Solution 1: View Range Pagination

The `str_replace_based_edit_tool` includes a `view_range` parameter that allows viewing specific line ranges of a file instead of the entire file.

#### Implementation

```python
# From trae_agent/tools/edit_tool.py

async def _view(self, path: Path, view_range: list[int] | None = None) -> ToolExecResult:
    """Implement the view command"""
    
    file_content = self.read_file(path)
    
    if view_range:
        file_lines = file_content.split("\n")
        n_lines_file = len(file_lines)
        init_line, final_line = view_range
        
        # Validate range
        if init_line < 1 or init_line > n_lines_file:
            raise ToolError(f"Invalid view_range: line {init_line} out of bounds")
        
        # Extract requested range
        if final_line == -1:
            # -1 means "to end of file"
            file_content = "\n".join(file_lines[init_line - 1:])
        else:
            file_content = "\n".join(file_lines[init_line - 1:final_line])
    
    return ToolExecResult(output=self._make_output(file_content, str(path), init_line))
```

#### Usage Example

```python
# Agent can view specific sections of a large file

# View lines 100-150 of a file
view(path="/project/src/large_file.py", view_range=[100, 150])

# View from line 500 to end of file
view(path="/project/src/large_file.py", view_range=[500, -1])

# View entire file (only for small files)
view(path="/project/src/small_file.py")
```

#### How the Agent Uses This

The agent's workflow for large files:

```
1. View file without range to see structure (gets truncated with hint)
2. Use grep/bash to find relevant section (e.g., "grep -n 'class UserService'")
3. View specific range containing relevant code
4. Make targeted edit
5. View again to verify changes
```

**Example Interaction:**

```
Agent: view /project/src/auth.py
Response: [First 16,000 chars] <response clipped><NOTE>To save on context only part of this file has been shown to you. You should retry this tool after you have searched inside the file with `grep -n` in order to find the line numbers of what you are looking for.</NOTE>

Agent: bash "grep -n 'def authenticate_user' /project/src/auth.py"
Response: 234:def authenticate_user(username, password):

Agent: view /project/src/auth.py, view_range=[230, 250]
Response: [Lines 230-250 with line numbers]

Agent: str_replace in /project/src/auth.py
  old_str: "    if user.password == password:"
  new_str: "    if bcrypt.checkpw(password.encode(), user.password):"
```

### Solution 2: Automatic Response Truncation

Trae Agent automatically truncates large outputs and provides helpful hints to guide the agent.

#### Implementation

```python
# From trae_agent/tools/run.py

MAX_RESPONSE_LEN: int = 16000  # Maximum characters in response
TRUNCATED_MESSAGE: str = "<response clipped><NOTE>To save on context only part of this file has been shown to you. You should retry this tool after you have searched inside the file with `grep -n` in order to find the line numbers of what you are looking for.</NOTE>"

def maybe_truncate(content: str, truncate_after: int | None = MAX_RESPONSE_LEN):
    """Truncate content and append a notice if content exceeds the specified length."""
    return (
        content
        if not truncate_after or len(content) <= truncate_after
        else content[:truncate_after] + TRUNCATED_MESSAGE
    )
```

#### Why 16,000 Characters?

- **Token Efficiency**: Typically 16,000 chars ≈ 4,000-5,000 tokens (leaving room for agent response)
- **Context Preservation**: Shows enough context for the agent to understand file structure
- **Prevents Truncation**: Ensures the agent's output response doesn't hit the LLM limit
- **Provides Guidance**: Truncation message tells the agent what to do next

#### Applied Throughout the System

Truncation is applied to:

1. **File Views**: When viewing large files
   ```python
   def _make_output(self, file_content: str, file_descriptor: str, ...):
       file_content = maybe_truncate(file_content)
       # ... format with line numbers
   ```

2. **Bash Command Output**: When commands produce large output
   ```python
   async def run(cmd: str, timeout: float | None = 120.0, 
                 truncate_after: int | None = MAX_RESPONSE_LEN):
       stdout, stderr = await process.communicate()
       return (
           process.returncode or 0,
           maybe_truncate(stdout.decode(), truncate_after=truncate_after),
           maybe_truncate(stderr.decode(), truncate_after=truncate_after),
       )
   ```

3. **Directory Listings**: When listing large directory structures

#### Self-Correcting Behavior

The truncation message is designed to guide the agent's next action:

```
<response clipped><NOTE>To save on context only part of this file has been shown 
to you. You should retry this tool after you have searched inside the file with 
`grep -n` in order to find the line numbers of what you are looking for.</NOTE>
```

This message:
- ✅ Explicitly tells the agent what happened
- ✅ Suggests the next action (use `grep -n`)
- ✅ Explains why (`to find the line numbers`)
- ✅ Guides toward the solution (then retry with view_range)

## Challenge 2: Code Editing Without Full Regeneration

### The Problem

Traditional approaches to code editing with LLMs often involve:
1. LLM reads entire file
2. LLM generates entire modified file
3. System replaces old file with new file

**This approach has severe problems:**
- **Token Waste**: 90% of output is unchanged code
- **Error Prone**: LLM may introduce typos, formatting changes, or subtle bugs in "unchanged" parts
- **Cost**: Extremely expensive for large files
- **Truncation**: Large files get cut off mid-generation
- **Inconsistency**: Small errors accumulate across regenerations

### Solution: String Replacement Editing

Trae Agent uses a **surgical editing** approach that only specifies what to change, not the entire file.

#### The `str_replace` Command

```python
# From trae_agent/tools/edit_tool.py

def str_replace(self, path: Path, old_str: str, new_str: str | None) -> ToolExecResult:
    """Implement the str_replace command, which replaces old_str with new_str in the file content"""
    
    # Read the file content
    file_content = self.read_file(path).expandtabs()
    old_str = old_str.expandtabs()
    new_str = new_str.expandtabs() if new_str is not None else ""
    
    # Check if old_str is unique in the file
    occurrences = file_content.count(old_str)
    if occurrences == 0:
        raise ToolError(
            f"No replacement was performed, old_str `{old_str}` did not appear verbatim in {path}."
        )
    elif occurrences > 1:
        file_content_lines = file_content.split("\n")
        lines = [idx + 1 for idx, line in enumerate(file_content_lines) if old_str in line]
        raise ToolError(
            f"No replacement was performed. Multiple occurrences of old_str `{old_str}` in lines {lines}. Please ensure it is unique"
        )
    
    # Replace old_str with new_str
    new_file_content = file_content.replace(old_str, new_str)
    
    # Write the new content to the file
    self.write_file(path, new_file_content)
    
    # Create a snippet of the edited section
    replacement_line = file_content.split(old_str)[0].count("\n")
    start_line = max(0, replacement_line - SNIPPET_LINES)
    end_line = replacement_line + SNIPPET_LINES + new_str.count("\n")
    snippet = "\n".join(new_file_content.split("\n")[start_line:end_line + 1])
    
    # Return success with snippet showing the change
    success_msg = f"The file {path} has been edited. "
    success_msg += self._make_output(snippet, f"a snippet of {path}", start_line + 1)
    success_msg += "Review the changes and make sure they are as expected."
    
    return ToolExecResult(output=success_msg)
```

#### Key Design Decisions

**1. Exact String Matching**

The `old_str` must match **exactly** (including whitespace):

```python
# Agent must match exactly what's in the file
old_str = "    if user.password == password:"

# NOT acceptable:
old_str = "if user.password == password:"  # Missing indentation
old_str = "    if user.password==password:"  # Missing spaces around ==
```

**Why?** Ensures the agent knows precisely what it's changing and prevents accidental modifications.

**2. Uniqueness Requirement**

The `old_str` must appear exactly **once** in the file:

```python
occurrences = file_content.count(old_str)
if occurrences > 1:
    # Error: ambiguous which occurrence to replace
    raise ToolError("Multiple occurrences found. Make old_str more unique.")
```

**Why?** Prevents ambiguity and ensures the agent makes intentional, specific changes.

**3. Context Snippet Return**

After editing, the tool returns a **snippet** showing the change:

```python
# Shows 4 lines before and 4 lines after the change
start_line = max(0, replacement_line - SNIPPET_LINES)
end_line = replacement_line + SNIPPET_LINES + new_str.count("\n")
snippet = "\n".join(new_file_content.split("\n")[start_line:end_line + 1])
```

**Why?** Allows the agent to verify the change was made correctly without viewing the entire file again.

### The `insert` Command

For adding new code without replacing anything:

```python
def _insert(self, path: Path, insert_line: int, new_str: str) -> ToolExecResult:
    """Implement the insert command, which inserts new_str at the specified line."""
    
    file_text = self.read_file(path).expandtabs()
    file_text_lines = file_text.split("\n")
    n_lines_file = len(file_text_lines)
    
    # Validate insert position
    if insert_line < 0 or insert_line > n_lines_file:
        raise ToolError(f"Invalid insert_line: {insert_line}")
    
    # Insert new lines
    new_str_lines = new_str.split("\n")
    new_file_text_lines = (
        file_text_lines[:insert_line] + 
        new_str_lines + 
        file_text_lines[insert_line:]
    )
    
    new_file_text = "\n".join(new_file_text_lines)
    self.write_file(path, new_file_text)
    
    # Return snippet showing insertion
    # ... (similar to str_replace)
```

**Use Cases:**
- Adding new functions to a class
- Adding new imports
- Adding new configuration entries
- Inserting documentation

## The Edit Tool Architecture

### Tool Command Overview

The `str_replace_based_edit_tool` provides four commands:

```
┌─────────────────────────────────────────────────────────────┐
│                   str_replace_based_edit_tool                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. view        - View file/directory (with optional range) │
│  2. create      - Create new file                           │
│  3. str_replace - Replace exact string match                │
│  4. insert      - Insert at specific line number            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### System Prompt Guidance

The agent receives explicit instructions in the system prompt:

```python
# From trae_agent/prompt/agent_prompt.py

TRAE_AGENT_SYSTEM_PROMPT = """You are an expert AI software engineering agent.

File Path Rule: All tools that take a `file_path` as an argument require an 
**absolute path**. You MUST construct the full, absolute path by combining the 
`[Project root path]` provided in the user's message with the file's path 
inside the project.

... [task instructions] ...

5. Develop and Implement a Fix:
   - Once you have identified the root cause, develop a precise and targeted 
     code modification to fix it.
   - Use the provided file editing tools to apply your patch. Aim for minimal, 
     clean changes.

**Guiding Principle:** Act like a senior software engineer. Prioritize 
correctness, safety, and high-quality, test-driven development.
"""
```

Key phrases that guide behavior:
- **"precise and targeted code modification"** - Don't regenerate entire files
- **"Aim for minimal, clean changes"** - Only change what needs to change
- **"Use the provided file editing tools"** - Use str_replace, not full file generation

### Agent Decision Process

When the agent needs to modify code:

```
┌──────────────────────────────────────────────────────────────┐
│ 1. Identify Problem Location                                 │
│    - Use grep to find relevant code                          │
│    - Use view with range to see context                      │
└──────────────┬───────────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────────┐
│ 2. Extract Exact String to Replace                           │
│    - View the specific lines                                 │
│    - Copy exact text including whitespace                    │
│    - Ensure it's unique in the file                          │
└──────────────┬───────────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────────┐
│ 3. Formulate Replacement                                      │
│    - Write new code with same indentation                    │
│    - Preserve surrounding context                            │
│    - Keep changes minimal                                    │
└──────────────┬───────────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────────┐
│ 4. Execute str_replace                                        │
│    - Tool validates uniqueness                               │
│    - Tool performs replacement                               │
│    - Tool returns snippet showing change                     │
└──────────────┬───────────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────────┐
│ 5. Verify Change                                              │
│    - Review returned snippet                                 │
│    - Run tests if needed                                     │
│    - Make additional edits if necessary                      │
└──────────────────────────────────────────────────────────────┘
```

## Workflow: From Problem to Solution

Let's trace a complete example of fixing a bug in a large file.

### Scenario: Fix Authentication Bug

**File:** `/project/src/auth.py` (1,500 lines)  
**Bug:** Password comparison uses plain text instead of bcrypt

### Step-by-Step Execution

#### Step 1: Initial Exploration (Truncated)

```
Agent: str_replace_based_edit_tool
  command: view
  path: /project/src/auth.py

Response:
1. # Authentication module
2. import hashlib
3. from database import db
4. ...
[lines 1-400 shown, then truncated]

<response clipped><NOTE>To save on context only part of this file has been shown 
to you. You should retry this tool after you have searched inside the file with 
`grep -n` in order to find the line numbers of what you are looking for.</NOTE>
```

**Analysis:**
- File is large (1,500 lines)
- Response was automatically truncated at 16,000 chars
- Agent receives hint to use grep

#### Step 2: Locate Relevant Code

```
Agent: bash
  command: grep -n "def authenticate_user" /project/src/auth.py

Response:
234:def authenticate_user(username, password):

Agent: bash
  command: grep -n "user.password == password" /project/src/auth.py

Response:
237:    if user.password == password:
```

**Analysis:**
- Found function at line 234
- Found problematic comparison at line 237
- Now knows exact location to view and edit

#### Step 3: View Relevant Section

```
Agent: str_replace_based_edit_tool
  command: view
  path: /project/src/auth.py
  view_range: [230, 245]

Response:
230. 
231. def authenticate_user(username, password):
232.     """Authenticate a user with username and password."""
233.     user = db.query(User).filter_by(username=username).first()
234.     if not user:
235.         return None
236.     # BUG: Plain text password comparison!
237.     if user.password == password:
238.         return user
239.     return None
240. 
241. def create_user(username, password):
242.     """Create a new user account."""
243.     # Hash password before storing
244.     hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt())
245.     user = User(username=username, password=hashed)
```

**Analysis:**
- Agent sees 15 lines of context (lines 230-245)
- This consumed only ~500 tokens (vs. 12,000 for full file)
- Agent identifies bug at line 237
- Agent sees bcrypt is already imported and used elsewhere

#### Step 4: Make Surgical Edit

```
Agent: str_replace_based_edit_tool
  command: str_replace
  path: /project/src/auth.py
  old_str: "    # BUG: Plain text password comparison!
    if user.password == password:
        return user"
  new_str: "    # Use bcrypt for secure password comparison
    if bcrypt.checkpw(password.encode(), user.password):
        return user"

Response:
The file /project/src/auth.py has been edited. Here's a snippet of the edited file:

232.     user = db.query(User).filter_by(username=username).first()
233.     if not user:
234.         return None
235.     # Use bcrypt for secure password comparison
236.     if bcrypt.checkpw(password.encode(), user.password):
237.         return user
238.     return None
239. 
240. def create_user(username, password):

Review the changes and make sure they are as expected.
```

**Analysis:**
- Agent specified exact multi-line string to replace
- Tool validated uniqueness (only one occurrence)
- Tool performed replacement
- Tool returned snippet showing successful change
- **No other code in the file was touched**
- Total output: ~200 tokens (vs. 12,000 to regenerate entire file)

#### Step 5: Verify Fix

```
Agent: bash
  command: python -m pytest tests/test_auth.py::test_authenticate_user -v

Response:
tests/test_auth.py::test_authenticate_user PASSED

Agent: task_done
```

**Analysis:**
- Agent runs tests to verify fix
- Tests pass
- Agent completes task
- Total tokens used: ~2,000 (vs. 25,000+ if regenerating files)

## Benefits and Trade-offs

### Benefits of This Approach

#### 1. Token Efficiency

**Traditional Approach:**
```
View full file:     12,000 tokens (input)
Generate full file: 12,000 tokens (output)
Total:              24,000 tokens
Cost (GPT-4o):      ~$0.60
```

**Trae Agent Approach:**
```
View snippet:       500 tokens (input)
str_replace call:   200 tokens (output)
Total:              700 tokens
Cost (GPT-4o):      ~$0.02
```

**Savings: 97% reduction in tokens, 97% reduction in cost**

#### 2. Error Reduction

**Traditional Approach Problems:**
- LLM may introduce typos in "unchanged" code
- May alter formatting unintentionally
- May skip or duplicate lines
- May introduce subtle logic errors
- Truncation can cut off mid-function

**Trae Agent Approach:**
- Only changes what agent explicitly specifies
- Preserves all unchanged code exactly
- No risk of truncation errors
- Verifiable changes (snippet shows exactly what changed)

#### 3. Large File Support

**Traditional Approach:**
- Limited to files < 4,096 tokens output (~400-500 lines)
- Larger files get truncated
- Cannot handle enterprise codebases

**Trae Agent Approach:**
- Can handle files of any size
- Views only relevant sections
- Edits any part regardless of file size
- Tested on files with 10,000+ lines

#### 4. Precision

**Traditional Approach:**
- "Change line 237" - may affect nearby lines
- Difficult to verify exact changes
- Hard to review diffs

**Trae Agent Approach:**
- Shows exact before/after
- Snippet confirms precise change
- Git diff shows only changed lines
- Easy to review and verify

### Trade-offs

#### Requires Exact Matching

**Challenge:** Agent must match code exactly (including whitespace)

**Mitigation:**
- Agent views code with line numbers first
- Can copy exact text from view output
- Error messages guide agent to correct format

**Example Error:**
```
No replacement was performed, old_str did not appear verbatim in file.
```

This error message is informative and helps the agent correct its approach.

#### Multiple Step Process

**Challenge:** Requires multiple tool calls (view → grep → view range → edit)

**Mitigation:**
- Modern LLMs handle multi-step reasoning well
- Total tokens still far less than regeneration
- Sequential thinking tool helps with complex edits

**Typical Edit Sequence:**
1. View (truncated) - 1,000 tokens
2. Grep search - 50 tokens
3. View range - 500 tokens
4. Edit - 200 tokens
**Total: 1,750 tokens vs. 24,000 for regeneration**

#### Uniqueness Constraint

**Challenge:** If `old_str` appears multiple times, edit fails

**Mitigation:**
- Agent can make `old_str` more unique by including more context
- Error message shows line numbers of all occurrences
- Agent can add surrounding lines to make it unique

**Example:**

```python
# This old_str appears 3 times:
old_str = "return user"

# Make it unique by adding context:
old_str = """    if bcrypt.checkpw(password.encode(), user.password):
        return user
    return None"""
```

## Advanced Techniques

### 1. Multi-Line Replacements

The `str_replace` command handles multi-line changes:

```python
old_str = """def calculate_average(numbers):
    total = sum(numbers)
    return total / len(numbers)"""

new_str = """def calculate_average(numbers):
    if not numbers:
        return 0
    total = sum(numbers)
    return total / len(numbers)"""
```

### 2. Deletion

To delete code, use empty `new_str`:

```python
old_str = """    # Deprecated function
    old_function()
"""
new_str = ""  # Deletes the code
```

### 3. Batch Edits

For multiple changes, the agent can chain edits:

```python
# Edit 1: Fix function A
str_replace(old_str="def funcA():", new_str="def func_a():")

# Edit 2: Fix function B  
str_replace(old_str="def funcB():", new_str="def func_b():")

# Edit 3: Update caller
str_replace(old_str="funcA()", new_str="func_a()")
```

Each edit is verified individually, ensuring correctness.

### 4. Strategic Context Inclusion

When `old_str` isn't unique, add surrounding lines:

```python
# Not unique:
old_str = "x = 5"

# Make unique by adding context:
old_str = """def initialize():
    x = 5
    y = 10"""
```

## Technical Specifications

### Truncation Settings

```python
MAX_RESPONSE_LEN = 16000  # characters
SNIPPET_LINES = 4         # lines of context in edit snippets
```

### Supported Operations

| Command | Purpose | Output Limit | Use Case |
|---------|---------|--------------|----------|
| `view` | View file/directory | 16,000 chars | Read code |
| `view` with `view_range` | View specific lines | No limit | Target section |
| `create` | Create new file | N/A | New files |
| `str_replace` | Replace exact string | Snippet only | Edit code |
| `insert` | Insert at line number | Snippet only | Add code |

### Error Handling

The tool provides specific error messages for common issues:

```python
# Path errors
"The path {path} is not an absolute path"
"The path {path} does not exist"
"File already exists at: {path}"

# Edit errors
"No replacement was performed, old_str did not appear verbatim"
"Multiple occurrences of old_str in lines {lines}"
"Invalid view_range: {range}"
```

## Comparison with Other Approaches

### Approach 1: Full File Regeneration

```python
# Traditional approach
read_file("large_file.py")      # 12,000 tokens input
llm_generates_full_file()       # 12,000 tokens output
write_file("large_file.py")

Problems:
- High token cost (24,000 tokens)
- Error prone (may introduce bugs in unchanged code)
- Fails on large files (output truncation)
- Hard to review changes
```

### Approach 2: Diff-Based Editing

```python
# Diff approach
read_file("large_file.py")      # 12,000 tokens input
llm_generates_diff_patch()      # 500 tokens output
apply_patch(patch)

Problems:
- Still requires reading full file
- Diff format is error-prone for LLMs
- Patch application can fail
- Line number changes cause conflicts
```

### Approach 3: Trae Agent's String Replacement

```python
# Trae Agent approach
view_file_range(lines=[230, 245])   # 500 tokens input
llm_generates_str_replace()         # 200 tokens output
str_replace(old_str, new_str)

Benefits:
- Low token cost (700 tokens)
- Precise, verifiable changes
- Works on files of any size
- Easy to review (snippet shows change)
- No risk of affecting unchanged code
```

## Summary

Trae Agent overcomes LLM output limitations through:

1. **View Range Pagination**
   - View only relevant sections of large files
   - Automatic guidance when files are truncated
   - Supports files of any size

2. **Response Truncation**
   - Automatic 16,000 character limit
   - Helpful hints guide next actions
   - Prevents context overflow

3. **Surgical String Replacement**
   - Edit specific code sections without regeneration
   - Exact string matching ensures precision
   - Uniqueness constraint prevents ambiguity
   - Snippet verification confirms changes

4. **Intelligent Agent Guidance**
   - System prompt emphasizes minimal changes
   - Truncation messages suggest grep usage
   - Error messages guide corrections
   - Multi-step reasoning handles complexity

**Result:**
- ✅ 97% reduction in token usage
- ✅ 97% reduction in API costs
- ✅ Dramatically fewer errors
- ✅ Support for files of unlimited size
- ✅ Precise, verifiable code changes
- ✅ Surgical edits without affecting unchanged code

This design makes Trae Agent capable of working with enterprise-scale codebases while maintaining accuracy, efficiency, and cost-effectiveness.

## Related Documentation

- [Architecture](ARCHITECTURE.md) - Overall system design
- [Tools Documentation](tools.md) - Tool reference
- [Workflow Guide](WORKFLOW.md) - Agent execution process
- [API Reference](API_REFERENCE.md) - Programmatic usage
