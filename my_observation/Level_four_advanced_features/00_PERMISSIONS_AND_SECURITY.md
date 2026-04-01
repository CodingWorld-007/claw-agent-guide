# Level 4: Advanced Features - Permissions & Security 🔐

## What You'll Learn
- Permission system architecture
- Tool access control and denial tracking
- Execution registry and tool pool
- Bootstrap graphs and initialization control
- Cost tracking and usage monitoring
- Advanced filtering and access patterns

## Part 1: Permission System

### Understanding Permissions

Permissions control **which tools** users and agents can access.

```python
# From permissions.py (typical implementation)
from dataclasses import dataclass

@dataclass(frozen=True)
class ToolPermissionContext:
    """Controls which tools are allowed"""
    user_id: str
    allowed_tools: set[str]
    denied_tools: set[str]
    role: str  # 'admin', 'user', 'readonly'
    
    def blocks(self, tool_name: str) -> bool:
        """Check if tool access is blocked"""
        # Explicit deny takes precedence
        if tool_name in self.denied_tools:
            return True
        
        # Admin can access everything
        if self.role == 'admin':
            return False
        
        # ReadOnly users can't use edit tools
        if self.role == 'readonly':
            return 'Edit' in tool_name or 'Delete' in tool_name
        
        # Default: allow if not explicitly denied
        return False

class ToolPermissionContext:
    @staticmethod
    def for_user(user_id: str, role: str = 'user') -> 'ToolPermissionContext':
        """Create permission context for user"""
        if role == 'admin':
            return ToolPermissionContext(
                user_id=user_id,
                allowed_tools=set(),  # Empty = allow all
                denied_tools=set(),
                role='admin'
            )
        elif role == 'readonly':
            return ToolPermissionContext(
                user_id=user_id,
                allowed_tools={'ReadTool', 'ListTool', 'GrepTool'},
                denied_tools=set(),
                role='readonly'
            )
        else:  # Default user
            return ToolPermissionContext(
                user_id=user_id,
                allowed_tools=set(),
                denied_tools={'DatabaseDeleteTool', 'ProductionDeployTool'},
                role='user'
            )
```

### Permission Denial Tracking

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class PermissionDenial:
    """Record of denied tool access"""
    tool_name: str
    reason: str

def track_permission_denial(
    tool_name: str,
    permission_context: ToolPermissionContext
):
    """Record when tool access is denied"""
    if permission_context.blocks(tool_name):
        denial = PermissionDenial(
            tool_name=tool_name,
            reason=f"User {permission_context.user_id} (role: {permission_context.role}) denied access"
        )
        return denial
    return None
```

### Filtering Tools by Permission

```python
# From tools.py
def filter_tools_by_permission_context(
    tools: tuple[PortingModule, ...],
    permission_context: ToolPermissionContext | None = None
) -> tuple[PortingModule, ...]:
    """Remove tools user doesn't have permission for"""
    if permission_context is None:
        return tools  # No restrictions
    
    filtered = tuple(
        module for module in tools
        if not permission_context.blocks(module.name)
    )
    return filtered

# Usage
admin_context = ToolPermissionContext.for_user('alice', role='admin')
user_context = ToolPermissionContext.for_user('bob', role='user')

all_tools = get_tools()
admin_tools = filter_tools_by_permission_context(all_tools, admin_context)      # More tools
user_tools = filter_tools_by_permission_context(all_tools, user_context)        # Fewer tools
readonly_tools = filter_tools_by_permission_context(all_tools, ReadOnlyContext) # Minimal tools
```

### Permission Context in Query Engine

```python
# In query_engine.py
@dataclass
class QueryEnginePort:
    permission_context: ToolPermissionContext | None = None
    permission_denials: list[PermissionDenial] = field(default_factory=list)
    
    def submit_message_with_permissions(
        self,
        prompt: str,
        permission_context: ToolPermissionContext
    ) -> TurnResult:
        """Execute message respecting permissions"""
        # Route normally
        routed = runtime.route_prompt(prompt)
        
        # Check each matched tool
        denied = []
        allowed = []
        
        for match in routed:
            if match.kind == 'tool':
                if permission_context.blocks(match.name):
                    denied.append(PermissionDenial(
                        tool_name=match.name,
                        reason=f"Access denied for user {permission_context.user_id}"
                    ))
                else:
                    allowed.append(match.name)
        
        # Track denials
        self.permission_denials.extend(denied)
        
        # Execute only allowed tools
        return self.submit_message(
            prompt=prompt,
            matched_tools=tuple(allowed),
            denied_tools=tuple(denied)
        )
```

## Part 2: Execution Registry

### What is an Execution Registry?

The **execution registry** maps tool names to their execution functions.

```python
# From execution_registry.py (typical implementation)
from typing import Callable, Dict

@dataclass
class ToolExecutor:
    name: str
    execute_fn: Callable[[str], str]
    description: str

class ExecutionRegistry:
    """Maps tool names to execution implementations"""
    
    def __init__(self):
        self._registry: Dict[str, ToolExecutor] = {}
    
    def register(
        self,
        name: str,
        execute_fn: Callable,
        description: str = ""
    ):
        """Register a tool's execution function"""
        self._registry[name] = ToolExecutor(
            name=name,
            execute_fn=execute_fn,
            description=description
        )
    
    def get_executor(self, tool_name: str) -> ToolExecutor | None:
        """Get executor for tool"""
        return self._registry.get(tool_name)
    
    def execute(self, tool_name: str, payload: str) -> str:
        """Execute registered tool"""
        executor = self.get_executor(tool_name)
        if not executor:
            return f"Unknown tool: {tool_name}"
        
        try:
            return executor.execute_fn(payload)
        except Exception as e:
            return f"Error executing {tool_name}: {e}"

# Global registry
_REGISTRY = ExecutionRegistry()

def build_execution_registry() -> ExecutionRegistry:
    """Build registry with all tool implementations"""
    registry = ExecutionRegistry()
    
    # Register tools
    registry.register(
        "BashTool",
        lambda cmd: subprocess.run(cmd, shell=True, capture_output=True).stdout.decode(),
        "Execute bash commands"
    )
    
    registry.register(
        "FileReadTool",
        lambda path: open(path).read(),
        "Read file contents"
    )
    
    # ... register more tools ...
    
    return registry

_GLOBAL_REGISTRY = build_execution_registry()
```

## Part 3: Tool Pool and Assembly

### ToolPool

The **tool pool** is the collection of available tools with their configurations.

```python
# From tool_pool.py (typical implementation)
from dataclasses import dataclass, field

@dataclass
class ToolPoolEntry:
    name: str
    enabled: bool
    default_timeout: int
    environment: dict = field(default_factory=dict)

class ToolPool:
    """Manages pool of available tools with settings"""
    
    def __init__(self):
        self.entries: Dict[str, ToolPoolEntry] = {}
    
    def add_tool(
        self,
        name: str,
        enabled: bool = True,
        timeout: int = 30,
        environment: dict = None
    ):
        """Add tool to pool"""
        self.entries[name] = ToolPoolEntry(
            name=name,
            enabled=enabled,
            default_timeout=timeout,
            environment=environment or {}
        )
    
    def get_enabled_tools(self) -> list[str]:
        """Get only enabled tools"""
        return [
            name for name, entry in self.entries.items()
            if entry.enabled
        ]
    
    def disable_tool(self, name: str):
        """Disable tool (prevent execution)"""
        if name in self.entries:
            self.entries[name].enabled = False
    
    def configure_tool(self, name: str, timeout: int = None, env: dict = None):
        """Modify tool configuration"""
        if name in self.entries:
            if timeout:
                self.entries[name].default_timeout = timeout
            if env:
                self.entries[name].environment.update(env)

def assemble_tool_pool() -> ToolPool:
    """Build tool pool from configuration"""
    pool = ToolPool()
    
    # Core tools
    pool.add_tool("BashTool", enabled=True, timeout=60)
    pool.add_tool("FileReadTool", enabled=True, timeout=10)
    pool.add_tool("FileEditTool", enabled=True, timeout=10)
    
    # MCP tools
    pool.add_tool("MCPFilesTool", enabled=True, timeout=30)
    pool.add_tool("MCPDatabaseTool", enabled=True, timeout=120)
    
    # Optional tools
    pool.add_tool("DatabaseTool", enabled=False, timeout=60)  # Disabled by default
    
    return pool
```

## Part 4: Cost Tracking and Usage Monitoring

### Usage Summary

```python
# From models.py
@dataclass(frozen=True)
class UsageSummary:
    """Tracks token/resource usage"""
    input_tokens: int = 0
    output_tokens: int = 0
    
    @property
    def total_tokens(self) -> int:
        return self.input_tokens + self.output_tokens
    
    def add_turn(self, prompt: str, output: str) -> 'UsageSummary':
        """Calculate tokens for a turn"""
        return UsageSummary(
            input_tokens=self.input_tokens + len(prompt.split()),
            output_tokens=self.output_tokens + len(output.split())
        )
    
    def exceeds_budget(self, budget: int) -> bool:
        """Check if usage exceeds budget"""
        return self.total_tokens > budget
```

### Cost Tracker

```python
# From cost_tracker.py (typical implementation)
@dataclass
class CostEntry:
    timestamp: str
    tool_name: str
    tokens_used: int
    cost_dollars: float  # May vary by tool
    session_id: str

class CostTracker:
    """Tracks and reports costs"""
    
    def __init__(self):
        self.entries: list[CostEntry] = []
    
    def log_cost(
        self,
        tool_name: str,
        tokens: int,
        session_id: str
    ):
        """Log tool cost"""
        # Typical pricing: $0.01 per 1000 tokens
        cost = (tokens / 1000) * 0.01
        
        entry = CostEntry(
            timestamp=datetime.now().isoformat(),
            tool_name=tool_name,
            tokens_used=tokens,
            cost_dollars=cost,
            session_id=session_id
        )
        
        self.entries.append(entry)
    
    def total_cost(self, session_id: str = None) -> float:
        """Get total cost"""
        entries = self.entries if not session_id else [
            e for e in self.entries if e.session_id == session_id
        ]
        return sum(e.cost_dollars for e in entries)
    
    def cost_report(self) -> str:
        """Generate cost report"""
        by_tool = {}
        for entry in self.entries:
            if entry.tool_name not in by_tool:
                by_tool[entry.tool_name] = 0
            by_tool[entry.tool_name] += entry.cost_dollars
        
        lines = ['Cost Report', '']
        for tool, cost in sorted(by_tool.items(), key=lambda x: x[1], reverse=True):
            lines.append(f'- {tool}: ${cost:.4f}')
        lines.append('')
        lines.append(f'**Total: ${sum(by_tool.values()):.2f}**')
        
        return '\n'.join(lines)

_COST_TRACKER = CostTracker()
```

## Part 5: Bootstrap Graph and Initialization Control

### Bootstrap Stages

```python
# From bootstrap_graph.py
@dataclass(frozen=True)
class BootstrapGraph:
    """Tracks initialization stages"""
    stages: tuple[str, ...]
    
    def validate(self) -> bool:
        """Check all stages completed"""
        return len(self.stages) > 0
    
    def as_mermaid(self) -> str:
        """Render as Mermaid diagram"""
        lines = ['graph TD']
        for i, stage in enumerate(self.stages):
            current = f'A{i}["Stage {i}: {stage}"]'
            lines.append(f'  {current}')
            if i > 0:
                lines.append(f'  A{i-1} --> A{i}')
        return '\n'.join(lines)

def build_bootstrap_graph() -> BootstrapGraph:
    """Build typical bootstrap sequence"""
    return BootstrapGraph(stages=(
        'top-level prefetch side effects',
        'warning handler and environment guards',
        'CLI parser and pre-action trust gate',
        'setup() + commands/agents parallel load',
        'deferred init after trust',
        'mode routing: local / remote / ssh / teleport / direct-connect / deep-link',
        'query engine submit loop',
    ))

# Visualize the bootstrap process
graph = build_bootstrap_graph()
print(graph.as_mermaid())
```

## Part 6: Advanced Filtering Patterns

### Pattern 1: Complex Permission Rules

```python
class AdvancedPermissionContext:
    """More sophisticated permission system"""
    
    def __init__(self, user_id: str, role: str):
        self.user_id = user_id
        self.role = role
        self.time_based_restrictions = {}
        self.quota_limits = {}
    
    def can_access_tool(self, tool_name: str) -> bool:
        """Check if user can access tool"""
        # Base role check
        if self.role == 'admin':
            return True
        
        # Time-based restrictions
        current_hour = datetime.now().hour
        if tool_name in self.time_based_restrictions:
            allowed_hours = self.time_based_restrictions[tool_name]
            if current_hour not in allowed_hours:
                return False
        
        # Quota check
        if tool_name in self.quota_limits:
            usage = self.get_tool_usage(tool_name)
            limit = self.quota_limits[tool_name]
            if usage >= limit:
                return False
        
        return True
```

### Pattern 2: Tool Filtering by Capability

```python
def get_tools_by_capability(capability: str) -> list[PortingModule]:
    """Filter tools by required capability"""
    capability_map = {
        'file_operation': {'FileReadTool', 'FileEditTool', 'FileListTool'},
        'bash_execution': {'BashTool', 'ShellTool'},
        'network': {'HttpTool', 'SocketTool'},
        'database': {'DatabaseTool', 'SqlTool'},
    }
    
    required_tools = capability_map. get(capability, set())
    all_tools = get_tools()
    
    return [
        tool for tool in all_tools
        if tool.name in required_tools
    ]
```

## Part 7: Practical Examples

### Example 1: Secure Tool Execution

```python
def execute_tool_securely(
    tool_name: str,
    payload: str,
    user_id: str,
    role: str
) -> tuple[bool, str]:
    """Execute tool with permission checks"""
    
    # Create permission context
    perm = ToolPermissionContext.for_user(user_id, role)
    
    # Check permission
    if perm.blocks(tool_name):
        return False, f"Access denied: {tool_name}"
    
    # Get tool from pool
    pool = assemble_tool_pool()
    tool_entry = pool.entries.get(tool_name)
    
    # Check enabled
    if not tool_entry or not tool_entry.enabled:
        return False, f"Tool not available: {tool_name}"
    
    # Execute with timeout
    try:
        executor = _GLOBAL_REGISTRY.get_executor(tool_name)
        if not executor:
            return False, f"No executor for: {tool_name}"
        
        result = executor.execute(tool_name, payload)
        return True, result
    except Exception as e:
        return False, f"Execution error: {e}"
```

### Example 2: Multi-User Cost Tracking

```python
def track_multi_user_costs():
    """Track costs across multiple users"""
    tracker = CostTracker()
    
    # Log various tool uses
    tracker.log_cost("BashTool", 500, session_id="user1_session1")
    tracker.log_cost("DatabaseTool", 2000, session_id="user1_session2")
    tracker.log_cost("FileReadTool", 100, session_id="user2_session1")
    
    # Generate reports
    print(tracker.cost_report())
    print(f"User 1 cost: ${tracker.total_cost('user1_session1'):.2f}")
```

## Key Takeaways

✅ **Permissions** control tool access by role
✅ **PermissionDenial** tracks rejected attempts
✅ **ExecutionRegistry** maps tools to functions
✅ **ToolPool** manages tool configuration
✅ **CostTracker** monitors resource usage
✅ **BootstrapGraph** ensures proper initialization
✅ **UsageSummary** tracks token consumption

## Next Step → Level 5: Full System Integration

Now that you understand all the pieces, let's put it all together and see the **complete agent system in action**!

---

**Remember:** Permissions secure the system, Execution Registry implements tools, Cost Tracking monitors resources!
