# Level 0: Quick Reference - Agent Concepts

## Agent Glossary

| Term | Definition | Example |
|------|-----------|---------|
| **Tool** | Atomic action/capability | ReadFileTool, SendEmailTool |
| **Command** | Orchestrated workflow using tools | schedule-meeting, deploy-app |
| **Query Engine** | Brain that processes requests | Matches intent → executes tools |
| **Runtime** | Execution environment | Manages sessions, state, persistence |
| **Permission** | Access control rule | allowed/denied for specific tools |
| **Session** | Single conversation context | User ID + conversation history |
| **Transcript** | Conversation record | All messages + tool executions |
| **Bootstrap** | Startup initialization | Load config, register tools, prepare system |
| **Routing** | Request → Right tool mapping | Token matching, relevance scoring |
| **Turn** | Single interaction cycle | Request → Execute → Response |

## The Agent Execution Cycle

```
1. USER SENDS REQUEST
   └─ "Build a Docker container for my app"

2. BOOTSTRAP CHECK
   └─ Verify session is valid, permissions loaded

3. REQUEST PARSING
   └─ Parse: "Docker", "container", "build", "app"

4. ROUTING & MATCHING
   └─ Find matching tools/commands by token similarity

5. PERMISSION CHECK
   └─ Verify user can access these tools

6. EXECUTION
   └─ Execute matched tools in sequence

7. RESPONSE
   └─ Return results, save to transcript

8. PERSISTENCE
   └─ Store session state for next turn
```

## Visual: Tool vs Command

```
TOOL (Low-level):
  ReadFile("path") → Returns: "file contents"
  
COMMAND (High-level):
  GenerateReport() → Uses ReadFile() + ProcessData() + FormatOutput()
```

## Permission System Example

```yaml
User: alice
Allowed Tools:
  - BashTool ✅
  - FileReadTool ✅
  - FileEditTool ❌ (Denied - read-only user)

User: bob
Allowed Tools:
  - BashTool ❌ (Denied - no direct terminal)
  - FileReadTool ✅
  - FileEditTool ✅
  - DatabaseTool ✅
```

## Bootstrap Stages (Initialization Order)

```
Stage 1: Top-level prefetch side effects
        └─ Load critical dependencies first

Stage 2: Warning handler & environment guards
        └─ Check system health, raise warnings

Stage 3: CLI parser & pre-action trust gate
        └─ Parse arguments, verify trust level

Stage 4: Setup() + parallel load
        └─ Load tools, commands, agents in parallel

Stage 5: Deferred initialization
        └─ Initialize after trust verification

Stage 6: Mode routing
        └─ Choose: local/remote/ssh/teleport/direct

Stage 7: Query engine loop
        └─ Start turn-based interaction
```

## CLAW-Code File Structure

```
src/
├── tools.py              ← Tool definitions & execution
├── commands.py           ← Command definitions & execution
├── query_engine.py       ← Main brain (routing, turn logic)
├── runtime.py            ← Session management & execution
├── context.py            ← Environment/path context
├── bootstrap_graph.py    ← Initialization stages
├── session_store.py      ← Persistence layer
├── models.py             ← Data structures
└── permissions.py        ← Permission system

reference_data/
├── tools_snapshot.json   ← All available tools
├── commands_snapshot.json ← All available commands
└── archive_surface_snapshot.json
```

## Key Insights

1. **Agent ≠ AI Model** - Agent is the framework; it uses AI externally
2. **Tools are composable** - Commands combine multiple tools
3. **Routing is probabilistic** - Token matching gives relevance scores
4. **Permissions are enforced** - Tools can be denied before execution
5. **Sessions are isolated** - Each conversation is independent
6. **Bootstrap is critical** - Proper initialization prevents errors
7. **Turn-based** - Each interaction is one "turn" with state tracking

## Common Beginner Questions

**Q: Can a tool call another tool?**
A: No directly. Tools are atomic. Commands can sequence tools together.

**Q: What if permission is denied?**
A: PermissionDenial object is created, returned to user, logged to transcript.

**Q: How long can a session last?**
A: Limited by max_turns (default 8) and token budget (default 2000 tokens).

**Q: Can I create custom tools?**
A: Yes - add to tool_snapshot.json and implement execution logic.

**Q: How does routing work exactly?**
A: Token-based matching - splits request into tokens, matches against tool/command names & descriptions, scores by frequency of matches.

---

[Continue to Level 1 →](../Level_one_tools_and_commands/README.md)
