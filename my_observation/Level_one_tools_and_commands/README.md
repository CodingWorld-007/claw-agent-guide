# Level 1: Summary & Navigation

## 📚 Documents in This Level

### 00_TOOLS_AND_COMMANDS.md
**Duration:** 45-60 minutes  
**Difficulty:** ⭐⭐ Intermediate

Deep dive into tools and commands - the building blocks of agents. Includes code examples.

**Prerequisite:** Level 0

**Topics covered:**
- What tools are and how they work
- All tool types in CLAW-Code
- Loading, listing, and executing tools
- Finding tools by query
- What commands are
- How commands vs tools differ
- Command discovery and execution
- Tool-command composition patterns

**Outcome:** Understand tools/commands deeply and see code patterns.

---

### 01_HANDS_ON_CUSTOM_TOOL.md
**Duration:** 30-45 minutes  
**Difficulty:** ⭐⭐ Intermediate

Create your first custom tool from scratch. Includes complete working examples.

**Prerequisite:** 00_TOOLS_AND_COMMANDS.md

**Topics covered:**
- Custom tool structure
- Tool registration via JSON
- Loading custom tools
- Execution implementation
- Integration with CLAW-Code
- Testing custom tools
- Complete working example (ProjectAnalyzer)

**Outcome:** You'll have built and tested your first custom tool.

---

## 🎯 Learning Objectives

After completing Level 1, you should be able to:

- [ ] Explain what a tool is and isn't
- [ ] Explain what a command is and isn't
- [ ] List the main tool categories
- [ ] Load and search for tools
- [ ] Create a custom tool
- [ ] Register a tool in the system
- [ ] Execute a tool with error handling
- [ ] Write tests for tools

---

## 💡 Key Concepts

### Tool = Atomic Action
```
Tool:    Read file("path") → Contents
Command: Generate Report → Uses ReadFile + ProcessData + Format
```

### Tool Lifecycle
1. Register in `tools_snapshot.json`
2. Load via `load_tool_snapshot()`
3. Find via `find_tools()` or `get_tool()`
4. Execute via `execute_tool()`
5. Receive `ToolExecution` result

### CLAW-Code Tool Types
- **File ops:** FileReadTool, FileEditTool, FileListTool
- **System:** BashTool, ProcessTool
- **Data:** JsonTool, YamlTool, CsvTool
- **Network:** HttpTool, SocketTool
- **Database:** DatabaseTool, SqlTool
- **MCP:** MCPFilesTool, MCPDatabaseTool

---

## 📁 Code Structure

```python
# Load tools
from tools import get_tools, find_tools, get_tool

# Find and execute
bash_tool = get_tool("BashTool")
result = execute_tool("BashTool", "ls -la")

# Filter and discover
file_tools = find_tools("file", limit=10)
```

---

## 🧪 Hands-On Practice

### Try This:
```python
# 1. List all tools
tools = get_tools()
print(f"Total tools: {len(tools)}")

# 2. Find file tools
file_tools = find_tools("file")
for tool in file_tools[:3]:
    print(f"- {tool.name}: {tool.responsibility}")

# 3. Execute a tool
result = execute_tool("BashTool", "pwd")
print(result.output)
```

### Create Your Own Tool:
- Follow the steps in `01_HANDS_ON_CUSTOM_TOOL.md`
- Build a LineCountTool
- Register and test it
- Use it in a command

---

## 🔄 Navigation

**← Previous Level:** [Level 0: What is an Agent?](../Level_zero_agent_understanding/)

**→ Next Level:** [Level 2: Query Engine & Routing](../Level_two_query_engine_and_routing/)

**Start with:** [Tools & Commands](./00_TOOLS_AND_COMMANDS.md) →

---

## ❓ Common Questions

**Q: What's the difference between tools and commands?**  
A: Tools are atomic (do one thing); commands orchestrate them (combine tools).

**Q: Can tools call other tools?**  
A: No directly. Commands can sequence tools together though.

**Q: Do I need to know Python?**  
A: Basic Python helps, but concepts are language-agnostic.

**Q: How many tools are there?**  
A: CLAW-Code has 500+ registered tools in the snapshot.

**Q: Can I add custom tools?**  
A: Yes! That's what `01_HANDS_ON_CUSTOM_TOOL.md` teaches.

---

**Ready?** Start with [Tools & Commands](./00_TOOLS_AND_COMMANDS.md)
