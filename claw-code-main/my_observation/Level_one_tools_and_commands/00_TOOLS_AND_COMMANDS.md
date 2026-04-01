# Level 1: Tools and Commands - The Building Blocks 🔧

## What You'll Learn
- What tools are and how they work
- What commands are and how they differ from tools
- How to load, list, and execute tools
- How to create and register new tools
- Real examples from CLAW-Code

## Part 1: Understanding Tools

### What is a Tool?

A **tool** is the smallest unit of action an agent can perform. It's atomic - meaning it does ONE thing and does it well.

```python
# Tool Example from tools.py
@dataclass(frozen=True)
class ToolExecution:
    name: str              # e.g., "BashTool"
    source_hint: str       # e.g., "terminal/bash"
    payload: str           # e.g., "ls -la /home"
    handled: bool          # Success or failure
    message: str           # Result or error message
```

### Available Tools in CLAW-Code

```
File Operations:
  - FileReadTool          → Read file contents
  - FileEditTool          → Modify file contents
  - FileListTool          → List directory contents

System Operations:
  - BashTool              → Run bash commands
  - ShellTool             → Shell command execution
  - ProcessTool           → Manage processes

Data Operations:
  - JsonTool              → JSON parsing/validation
  - YamlTool              → YAML parsing
  - CsvTool               → CSV operations

Network Operations:
  - HttpTool              → HTTP requests
  - SocketTool            → Network connections

Database Operations:
  - DatabaseTool          → Query databases
  - SqlTool               → Execute SQL

MCP Tools (Model Context Protocol):
  - MCPFilesTool          → Advanced file ops
  - MCPDatabaseTool       → Advanced DB ops
  - MCPServerTool         → Server management
```

### Loading Tools

In CLAW-Code, tools are loaded from a JSON snapshot:

```python
# From tools.py
SNAPSHOT_PATH = Path('reference_data/tools_snapshot.json')

def load_tool_snapshot() -> tuple[PortingModule, ...]:
    """Load all available tools from JSON"""
    raw_entries = json.loads(SNAPSHOT_PATH.read_text())
    return tuple(
        PortingModule(
            name=entry['name'],
            responsibility=entry['responsibility'],
            source_hint=entry['source_hint'],
            status='mirrored',
        )
        for entry in raw_entries
    )

PORTED_TOOLS = load_tool_snapshot()
```

### Listing Tools

```python
# From tools.py - Get all tools
def get_tools(
    simple_mode: bool = False,           # Only basic tools
    include_mcp: bool = True,            # Include MCP tools
    permission_context: ToolPermissionContext | None = None
) -> tuple[PortingModule, ...]:
    """Return available tools based on filters"""
    tools = list(PORTED_TOOLS)
    
    # Optional: Filter to simple mode
    if simple_mode:
        tools = [m for m in tools if m.name in {
            'BashTool', 'FileReadTool', 'FileEditTool'
        }]
    
    # Optional: Exclude MCP tools  
    if not include_mcp:
        tools = [m for m in tools if 'mcp' not in m.name.lower()]
    
    # Apply permission filtering
    return filter_tools_by_permission_context(tools, permission_context)
```

### Executing a Tool

```python
# From tools.py
def execute_tool(name: str, payload: str = '') -> ToolExecution:
    """Find and execute a tool"""
    module = get_tool(name)  # Find tool by name
    
    if module is None:
        return ToolExecution(
            name=name,
            source_hint='',
            payload=payload,
            handled=False,
            message=f'Unknown tool: {name}'
        )
    
    # Execute the tool with payload
    action = f"Tool '{module.name}' from {module.source_hint} handles: {payload}"
    return ToolExecution(
        name=module.name,
        source_hint=module.source_hint,
        payload=payload,
        handled=True,
        message=action
    )
```

### Finding Tools by Query

```python
# From tools.py
def find_tools(query: str, limit: int = 20) -> list[PortingModule]:
    """Search for tools by name or responsibility"""
    needle = query.lower()
    matches = [
        module for module in PORTED_TOOLS
        if needle in module.name.lower() 
        or needle in module.source_hint.lower()
    ]
    return matches[:limit]

# Example usage
bash_tools = find_tools("bash")      # Returns BashTool, etc.
file_tools = find_tools("file")      # Returns all file tools
```

## Part 2: Understanding Commands

### What is a Command?

A **command** is a higher-level workflow that orchestrates multiple tools together. Commands provide user-friendly operations that hide the complexity of tool sequencing.

```
Tool Level (Low):
  FileReadTool("config.json") → Returns file content

Command Level (High):
  LoadConfig Command:
    ├─ FileReadTool("config.json")
    ├─ ValidateJsonTool(json_content)
    └─ ReturnConfigObject
```

### Available Commands in CLAW-Code

Commands are loaded similarly to tools:

```python
# From commands.py
SNAPSHOT_PATH = Path('reference_data/commands_snapshot.json')

def load_command_snapshot() -> tuple[PortingModule, ...]:
    """Load all available commands from JSON"""
    raw_entries = json.loads(SNAPSHOT_PATH.read_text())
    return tuple(
        PortingModule(
            name=entry['name'],
            responsibility=entry['responsibility'],
            source_hint=entry['source_hint'],
            status='mirrored',
        )
        for entry in raw_entries
    )

PORTED_COMMANDS = load_command_snapshot()
```

### Listing Commands

```python
# From commands.py
def get_commands(
    cwd: str | None = None,
    include_plugin_commands: bool = True,      # from plugins/
    include_skill_commands: bool = True         # from skills/
) -> tuple[PortingModule, ...]:
    """Get available commands with filters"""
    commands = list(PORTED_COMMANDS)
    
    if not include_plugin_commands:
        commands = [c for c in commands if 'plugin' not in c.source_hint.lower()]
    
    if not include_skill_commands:
        commands = [c for c in commands if 'skills' not in c.source_hint.lower()]
    
    return tuple(commands)
```

### Executing a Command

```python
# From commands.py
def execute_command(name: str, prompt: str = '') -> CommandExecution:
    """Execute a command by name"""
    module = get_command(name)
    
    if module is None:
        return CommandExecution(
            name=name,
            source_hint='',
            prompt=prompt,
            handled=False,
            message=f'Unknown command: {name}'
        )
    
    action = f"Command '{module.name}' handles: {prompt}"
    return CommandExecution(
        name=module.name,
        source_hint=module.source_hint,
        prompt=prompt,
        handled=True,
        message=action
    )
```

### Finding Commands

```python
# From commands.py
def find_commands(query: str, limit: int = 20) -> list[PortingModule]:
    """Search for commands"""
    needle = query.lower()
    matches = [
        module for module in PORTED_COMMANDS
        if needle in module.name.lower()
        or needle in module.source_hint.lower()
    ]
    return matches[:limit]

# Examples
build_commands = find_commands("build")        # Build-related commands
deploy_commands = find_commands("deploy")      # Deployment commands
```

## Part 3: Tools vs Commands

| Aspect | Tool | Command |
|--------|------|---------|
| **Scope** | Atomic, single action | Complex workflow |
| **Complexity** | Simple | Can be very complex |
| **Composability** | Cannot call other tools | Calls multiple tools/commands |
| **User Interface** | Low-level API | High-level operation |
| **Example** | ReadFile("path") | GenerateReport (combines read, process, format) |
| **Error Handling** | Single failure | Multi-step failure recovery |
| **Permissions** | Individual permission | May require multiple permissions |

## Part 4: Practical Implementation Example

### Walkthrough: "List Python files"

**Step 1: Tool-Based Approach**
```python
# Using tools directly
bash_tool = get_tool("BashTool")
execution = execute_tool(
    "BashTool", 
    payload="find . -name '*.py' -type f"
)
print(execution.message)
```

**Step 2: Command-Based Approach**
```python
# Using high-level command
command = get_command("ListPythonFiles")
execution = execute_command(
    "ListPythonFiles",
    prompt="Show me all Python files"
)
print(execution.message)
```

### Walkthrough: Building a Custom Tool

```python
# 1. Create tool definition in tools_snapshot.json
{
    "name": "CountTool",
    "responsibility": "Count lines in files",
    "source_hint": "utils/count"
}

# 2. Load it
tool = get_tool("CountTool")

# 3. Execute it
result = execute_tool("CountTool", payload="src/main.py")
```

## Part 5: Key Code Patterns

### Pattern 1: Tool Discovery
```python
def discover_tools_for_task(task_description: str) -> list[PortingModule]:
    """Find relevant tools for a task"""
    keywords = task_description.lower().split()
    relevant_tools = [
        tool for tool in PORTED_TOOLS
        if any(kw in tool.name.lower() for kw in keywords)
    ]
    return relevant_tools
```

### Pattern 2: Tool Chaining
```python
def chain_tools(tool_sequence: list[str], data: dict) -> dict:
    """Execute tools in sequence, passing results"""
    result = data
    for tool_name in tool_sequence:
        tool = get_tool(tool_name)
        if tool:
            execution = execute_tool(tool_name, str(result))
            result = execution.message
    return {"final_result": result}
```

### Pattern 3: Permission-Aware Tool Access
```python
def safe_execute_tool(
    tool_name: str,
    payload: str,
    permission_context: ToolPermissionContext
) -> ToolExecution:
    """Execute tool only if permission granted"""
    filtered_tools = filter_tools_by_permission_context(
        (get_tool(tool_name),),
        permission_context
    )
    
    if not filtered_tools:
        return ToolExecution(
            name=tool_name,
            source_hint='',
            payload=payload,
            handled=False,
            message=f'Permission denied for {tool_name}'
        )
    
    return execute_tool(tool_name, payload)
```

## Hands-On Exercise

### Exercise 1: List all file-related tools
```python
# TODO: Write code using find_tools()
file_tools = find_tools("file")
for tool in file_tools[:5]:
    print(f"{tool.name}: {tool.responsibility}")
```

### Exercise 2: Find tools by pattern
```python
# TODO: Search for tools matching "bash" or "shell"
bash_tools = find_tools("bash")
shell_tools = find_tools("shell")
combined = bash_tools + shell_tools
print(f"Found {len(combined)} tools")
```

### Exercise 3: Execute a tool safely
```python
# TODO: Try finding and executing a non-existent tool
result = execute_tool("NonExistentTool", "payload")
print(f"Handled: {result.handled}")
print(f"Message: {result.message}")
```

## Key Takeaways

✅ **Tools** are atomic actions (ReadFile, BashTool)
✅ **Commands** orchestrate tools into workflows
✅ **Tool execution** returns ToolExecution objects
✅ **Commands** are discoverable and filterable
✅ **Permissions** control which tools can be used
✅ **Routing** matches user intent to tools/commands

## Next Step → Level 2: Query Engine & Routing

Now that you understand the building blocks (tools & commands), let's learn how the agent **decides which ones to use** through the Query Engine!

---

**Remember:** Tools are like ingredients, commands are like recipes!
