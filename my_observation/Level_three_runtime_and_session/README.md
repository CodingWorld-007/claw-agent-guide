# Level 3: Summary & Navigation

## 📚 Documents in This Level

### 00_RUNTIME_AND_SESSION.md
**Duration:** 60-90 minutes  
**Difficulty:** ⭐⭐⭐ Intermediate-Advanced

Learn how sessions are created, managed, and persisted. Includes complete session lifecycle examples.

**Prerequisite:** Level 2 (Query Engine & Routing)

**Topics covered:**
- Runtime architecture and responsibilities
- Session structure and lifecycle
- PortContext (environment information)
- Session persistence and restoration
- Transcript storage
- Workspace setup and initialization
- Multi-mode execution (local, remote, SSH, etc.)
- Complete session workflow
- Session monitoring

**Outcome:** Understand how agent sessions work and how to persist state.

---

## 🎯 Learning Objectives

After completing Level 3, you should be able to:

- [ ] Understand runtime's role in agent systems
- [ ] Create and manage sessions
- [ ] Save and restore sessions from disk
- [ ] Track conversation transcripts
- [ ] Build context from workspace
- [ ] Understand multi-mode execution
- [ ] Implement session persistence
- [ ] Monitor session state

---

## 💡 Key Concepts

### Runtime = Execution Manager
The runtime:
- Creates sessions
- Routes prompts to query engine
- Manages conversation history
- Persists state to disk
- Handles different execution modes

### Session = Conversation Container
```
Session {
  ID: unique identifier
  Context: environment info
  History: messages so far
  Transcript: full record
  State: current variables
  Config: max turns, budget
}
```

### Session Persistence
```
Turn 1: Execute → Save to disk
Turn 2: Execute → Update on disk
Turn 3: Execute → Update on disk

Later:
Load Session → Restore Turn 1, 2, 3 context → Continue Turn 4
```

### Context = Environment Info
```
PortContext {
  source_root: "./src"
  tests_root: "./tests"
  python_file_count: 150
  archive_available: true
}
```

---

## 📊 Architecture

### Session Lifecycle

```
1. CREATE
   └─ New session ID
   └─ Initialize context
   └─ Setup workspace

2. INTERACT
   └─ Turn 1: execute, save
   └─ Turn 2: execute, save
   └─ Turn N: execute, save

3. PERSIST
   └─ Save to disk
   └─ Save transcript
   └─ Save metadata

4. RESTORE
   └─ Load from disk
   └─ Restore state
   └─ Continue from Turn N
```

### Multi-Mode Execution

```
Local Mode:
  User → QueryEngine → Tools → Local Machine

Remote Mode:
  User → QueryEngine → Tools → Remote Server (SSH)

Teleport Mode:
  User → QueryEngine → Tools → Multi-hop (A→B→C)

Direct-Connect Mode:
  User → QueryEngine → Tools → P2P Direct
```

---

## 💻 Code Patterns

### Create and Use Session
```python
from query_engine import QueryEnginePort
from session_store import save_session

# Create
engine = QueryEnginePort.from_workspace()
session_id = engine.session_id

# Interact
engine.submit_message("First prompt")
engine.submit_message("Second prompt")

# Persist
save_session(
    session_id,
    engine.mutable_messages,
    engine.total_usage
)
```

### Restore Session
```python
from query_engine import QueryEnginePort

# Load previous session
engine = QueryEnginePort.from_saved_session("session_id_here")

# Continue conversation
result = engine.submit_message("Third prompt (continues Turn 1-2)")

# Restore has context from previous turns
print(f"Previous messages: {len(engine.mutable_messages)}")
```

### Build and Use Context
```python
from context import build_port_context, render_context

context = build_port_context()
print(render_context(context))
# Output:
# Source root: ./src
# Python files: 150
# Test files: 45
# Archive available: true
```

---

## 🧪 Hands-On Exercises

### Exercise 1: Create and Save Session
```python
from uuid import uuid4
from query_engine import QueryEnginePort
from session_store import save_session

# Create fresh session
engine = QueryEnginePort.from_workspace()
print(f"Session: {engine.session_id}")

# Execute a few turns
engine.submit_message("Turn 1")
engine.submit_message("Turn 2")

# Save it
save_session(
    engine.session_id,
    engine.mutable_messages,
    engine.total_usage
)
print(f"Saved {len(engine.mutable_messages)} messages")
```

### Exercise 2: Restore and Continue
```python
from query_engine import QueryEnginePort

# Restore previous
restored = QueryEnginePort.from_saved_session("your_session_id")
print(f"Restored {len(restored.mutable_messages)} previous messages")

# Continue
new_result = restored.submit_message("Continue from before")
print(f"Now have {len(restored.mutable_messages)} messages")
```

### Exercise 3: Analyze Context
```python
from context import build_port_context

ctx = build_port_context()
print(f"Source files: {ctx.python_file_count}")
print(f"Archive: {ctx.archive_available}")
print(f"Tests: {ctx.test_file_count}")
```

---

## 🔄 Navigation

**← Previous Level:** [Level 2: Query Engine](../Level_two_query_engine_and_routing/)

**→ Next Level:** [Level 4: Permissions & Security](../Level_four_advanced_features/)

**Start with:** [Runtime & Sessions](./00_RUNTIME_AND_SESSION.md) →

---

## ❓ Common Questions

**Q: What's the difference between session and transcript?**  
A: Session is current state; transcript is complete history record.

**Q: Can sessions live forever?**  
A: Usually limited by storage and token budget per session.

**Q: How do sessions persist across machine restarts?**  
A: They're saved to disk (JSON) and reloaded by session ID.

**Q: What if I lose the session ID?**  
A: You can list saved sessions or search the session directory.

**Q: Can multiple sessions run simultaneously?**  
A: Yes - each session has unique ID and independent state.

**Q: What is teleport mode used for?**  
A: Executing through multiple intermediary servers.

---

**Ready?** Start with [Runtime & Sessions](./00_RUNTIME_AND_SESSION.md)
