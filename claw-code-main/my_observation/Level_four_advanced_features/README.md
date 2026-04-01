# Level 4: Summary & Navigation

## 📚 Documents in This Level

### 00_PERMISSIONS_AND_SECURITY.md
**Duration:** 70-90 minutes  
**Difficulty:** ⭐⭐⭐ Advanced

Learn advanced features: permissions, execution registry, tool pools, and cost tracking.

**Prerequisite:** Level 3 (Runtime & Sessions)

**Topics covered:**
- Permission system architecture
- Tool access control
- Permission denial tracking
- Role-based access control
- Execution registry patterns
- Tool pool configuration
- Cost tracking and monitoring
- Usage monitoring
- Bootstrap graphs
- Advanced filtering patterns
- Complete security examples

**Outcome:** Understand how to secure and monitor agent systems.

---

## 🎯 Learning Objectives

After completing Level 4, you should be able to:

- [ ] Implement permission systems
- [ ] Design role-based access control
- [ ] Track permission denials
- [ ] Build execution registries
- [ ] Configure tool pools
- [ ] Track costs and usage
- [ ] Monitor agent execution
- [ ] Implement advanced security

---

## 💡 Key Concepts

### Permission System = Access Control
```
User: alice, Role: admin
  → Can access: ALL tools

User: bob, Role: user
  → Can access: FileReadTool, BashTool
  → Cannot: DatabaseDeleteTool, ProductionDeployTool

User: charlie, Role: readonly
  → Can access: FileReadTool, ListTool
  → Cannot: FileEditTool, DeleteTool
```

### Tool Pool = Configuration Manager
```
ToolPool {
  BashTool:
    enabled: true
    timeout: 60s
    environment: {TIMEOUT: 60}
  
  DatabaseTool:
    enabled: false (disabled)
    timeout: 30s
}
```

### Cost Tracking = Resource Monitoring
```
Turn 1:
  BashTool: 500 tokens → $0.005
  
Turn 2:
  DatabaseTool: 2000 tokens → $0.020

Total Cost: $0.025
```

### Execution Registry = Implementation Mapping
```
Tool Name → Execute Function

BashTool → execute_bash(cmd)
FileReadTool → execute_read_file(path)
HttpTool → execute_http_request(url)
```

---

## 📊 Security Architecture

### Permission Check Flow

```
Tool Request
    ↓
Is user known? → No → Deny
    ↓ Yes
Check user role
    ↓
Check allowed_tools set
    ↓
Check denied_tools set
    ↓
Check time-based restrictions
    ↓
Check quota limits
    ↓
Allow/Deny → Record PermissionDenial
```

### Cost Tracking Flow

```
Tool Execution
    ↓
Measure tokens used
    ↓
Calculate cost ($0.01 per 1000 tokens typical)
    ↓
Record in CostTracker
    ↓
Check against budget
    ↓
Report in session
```

---

## 💻 Code Patterns

### Permission System
```python
from permissions import ToolPermissionContext

# Create context for user
context = ToolPermissionContext.for_user('alice', role='admin')

# Check access
if context.blocks('SensitiveTool'):
    print("Access denied")
else:
    print("Access granted")

# Filter tools by permission
tools = filter_tools_by_permission_context(all_tools, context)
```

### Execution Registry
```python
from execution_registry import build_execution_registry

registry = build_execution_registry()

# Get executor for tool
executor = registry.get_executor('BashTool')

# Execute
if executor:
    result = executor.execute('BashTool', 'ls -la')
```

### Tool Pool Configuration
```python
from tool_pool import assemble_tool_pool

pool = assemble_tool_pool()

# Disable tool
pool.disable_tool('DatabaseTool')

# Configure timeout
pool.configure_tool('BashTool', timeout=120)

# Get enabled tools
enabled = pool.get_enabled_tools()
```

### Cost Tracking
```python
from cost_tracker import CostTracker

tracker = CostTracker()

# Log tool use
tracker.log_cost('BashTool', 500, session_id='s1')
tracker.log_cost('HttpTool', 1000, session_id='s1')

# Get total cost
cost = tracker.total_cost(session_id='s1')
print(f"Session cost: ${cost:.2f}")

# Generate report
print(tracker.cost_report())
```

---

## 🧪 Hands-On Exercises

### Exercise 1: Implement Permission System
```python
from permissions import ToolPermissionContext

# Create admin context
admin = ToolPermissionContext.for_user('admin', role='admin')

# Create user context
user = ToolPermissionContext.for_user('bob', role='user')

# Create readonly context
readonly = ToolPermissionContext.for_user('charlie', role='readonly')

# Test access
for tool in ['BashTool', 'FileEditTool', 'DatabaseDeleteTool']:
    print(f"Admin: {not admin.blocks(tool)}")
    print(f"User: {not user.blocks(tool)}")
    print(f"Readonly: {not readonly.blocks(tool)}")
    print()
```

### Exercise 2: Track Costs
```python
from cost_tracker import CostTracker

tracker = CostTracker()

# Simulate tool executions
tools_used = [
    ('BashTool', 500),
    ('FileReadTool', 100),
    ('HttpTool', 750),
    ('BashTool', 1200),
]

for tool, tokens in tools_used:
    tracker.log_cost(tool, tokens, session_id='session1')

# Analyze
print(tracker.cost_report())
print(f"Total: ${tracker.total_cost('session1'):.2f}")
```

### Exercise 3: Configure Tool Pool
```python
from tool_pool import assemble_tool_pool

pool = assemble_tool_pool()

# View enabled tools
print(f"Enabled tools: {len(pool.get_enabled_tools())}")

# Disable a tool
pool.disable_tool('DatabaseTool')
print(f"After disable: {len(pool.get_enabled_tools())}")

# Configure a tool
pool.configure_tool('BashTool', timeout=90)
tool_config = pool.entries['BashTool']
print(f"BashTool timeout: {tool_config.default_timeout}s")
```

---

## 🔐 Security Best Practices

### Role Design
```
admin:
  - Access all tools
  - No time restrictions
  - High quotas

user:
  - Access core tools
  - Read/write restricted
  - Moderate quotas

readonly:
  - Access read tools only
  - No write tools
  - Monitoring tools only
```

### Cost Monitoring
```
Track per:
  - Session
  - User
  - Tool type
  - Time period

Alert on:
  - Unusual spikes
  - Quota exceeded
  - Budget threshold
```

### Tool Restrictions
```
Always disabled by default:
  - Production changes
  - Database deletes
  - System configuration changes
  - Security-sensitive operations

Only for admin:
  - System administration
  - Batch operations
  - Advanced configuration
```

---

## 🔄 Navigation

**← Previous Level:** [Level 3: Runtime & Sessions](../Level_three_runtime_and_session/)

**→ Next Level:** [Level 5: Complete Integration](../Level_five_full_integration/)

**Start with:** [Permissions & Security](./00_PERMISSIONS_AND_SECURITY.md) →

---

## ❓ Common Questions

**Q: How do permissions prevent unwanted tool use?**  
A: Tools are filtered before execution; denied tools never run.

**Q: Can permissions change during a session?**  
A: Generally no - set at session start. Can reload if needed.

**Q: How is cost calculated?**  
A: Typically tokens * rate (e.g., $0.01 per 1000 tokens).

**Q: What happens if a user exceeds budget?**  
A: Tools are denied and execution stops.

**Q: Can I implement custom permission rules?**  
A: Yes - extend ToolPermissionContext with your logic.

**Q: How much does each tool cost?**  
A: Varies by implementation; typically proportional to tokens.

---

**Ready?** Start with [Permissions & Security](./00_PERMISSIONS_AND_SECURITY.md)
