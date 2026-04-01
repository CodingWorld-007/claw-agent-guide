# Level 5: Full System Integration - Building a Complete Agent 🚀

## What You'll Learn
- How all components work together
- Complete request-response workflow
- Building a production-ready agent
- Real-world implementation patterns
- Debugging and monitoring

## Part 1: System Architecture Overview

### Complete Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     USER INTERACTION                             │
│  ("Build a Docker container and deploy it")                     │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                  BOOTSTRAP PHASE                                 │
│  ┌──────────────────────────────────────────────────────┐       │
│  │ 1. Load Configuration                               │       │
│  │    - CLI Parser                                     │       │
│  │    - Trust Gate Verification                        │       │
│  └──────────────────────────────────────────────────────┘       │
│  ┌──────────────────────────────────────────────────────┐       │
│  │ 2. Initialize Systems (Parallel)                     │       │
│  │    - Load Tools Snapshot                            │       │
│  │    - Load Commands Snapshot                         │       │
│  │    - Load Permission Context                        │       │
│  │    - Initialize Cost Tracker                        │       │
│  └──────────────────────────────────────────────────────┘       │
│  ┌──────────────────────────────────────────────────────┐       │
│  │ 3. Setup Workspace                                   │       │
│  │    - Build Port Context                             │       │
│  │    - Assemble Tool Pool                             │       │
│  │    - Build Execution Registry                       │       │
│  │    - Create Query Engine                            │       │
│  └──────────────────────────────────────────────────────┘       │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                  REQUEST ROUTING                                 │
│  ┌──────────────────────────────────────────────────────┐       │
│  │ 1. Parse User Prompt                                │       │
│  │    - Tokenize request                              │       │
│  │    - Normalize tokens                              │       │
│  └──────────────────────────────────────────────────────┘       │
│  ┌──────────────────────────────────────────────────────┐       │
│  │ 2. Route to Tools/Commands                          │       │
│  │    - Match against available tools                 │       │
│  │    - Match against available commands              │       │
│  │    - Score by relevance                            │       │
│  └──────────────────────────────────────────────────────┘       │
│  ┌──────────────────────────────────────────────────────┐       │
│  │ 3. Apply Permissions                                │       │
│  │    - Check user role/permissions                   │       │
│  │    - Track denied accesses                         │       │
│  │    - Filter available tools                        │       │
│  └──────────────────────────────────────────────────────┘       │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                  EXECUTION PHASE                                 │
│  ┌──────────────────────────────────────────────────────┐       │
│  │ 1. Validate Budget                                  │       │
│  │    - Check token usage                             │       │
│  │    - Check turn count                              │       │
│  └──────────────────────────────────────────────────────┘       │
│  ┌──────────────────────────────────────────────────────┐       │
│  │ 2. Execute Tools/Commands                           │       │
│  │    - Get executor from registry                     │       │
│  │    - Apply tool pool configuration                 │       │
│  │    - Run with timeout                              │       │
│  │    - Capture output                                │       │
│  └──────────────────────────────────────────────────────┘       │
│  ┌──────────────────────────────────────────────────────┐       │
│  │ 3. Track Execution                                  │       │
│  │    - Record tool execution                         │       │
│  │    - Track costs                                   │       │
│  │    - Update usage summary                          │       │
│  └──────────────────────────────────────────────────────┘       │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                  RESPONSE & PERSISTENCE                          │
│  ┌──────────────────────────────────────────────────────┐       │
│  │ 1. Format Response                                  │       │
│  │    - Combine tool outputs                          │       │
│  │    - Return TurnResult object                       │       │
│  └──────────────────────────────────────────────────────┘       │
│  ┌──────────────────────────────────────────────────────┐       │
│  │ 2. Save State                                        │       │
│  │    - Update transcript store                        │       │
│  │    - Save session to disk                           │       │
│  │    - Record in cost tracker                         │       │
│  └──────────────────────────────────────────────────────┘       │
│  ┌──────────────────────────────────────────────────────┐       │
│  │ 3. Return to User                                    │       │
│  │    - Stream output                                 │       │
│  │    - Provide metadata                              │       │
│  └──────────────────────────────────────────────────────┘       │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   USER RECEIVES RESPONSE                         │
│  ("Container built. Deployed to production.")                   │
└─────────────────────────────────────────────────────────────────┘
```

## Part 2: Complete Integration Example

### Building a Production Agent

```python
# complete_agent.py
from pathlib import Path
from uuid import uuid4
from datetime import datetime

class ProductionAgent:
    """A complete, production-ready agent implementation"""
    
    def __init__(
        self,
        workspace_path: Path,
        user_id: str,
        role: str = 'user',
        max_turns: int = 8
    ):
        # Initialize bootstrap
        self.workspace_path = workspace_path
        self.user_id = user_id
        self.role = role
        self.session_id = uuid4().hex
        
        # Log bootstrap
        print(f"[BOOTSTRAP] Starting session {self.session_id}")
        
        # Stage 1: Load configuration
        print("[BOOTSTRAP] Stage 1: Loading configuration...")
        self.context = build_port_context(workspace_path)
        self.permission_context = ToolPermissionContext.for_user(user_id, role)
        
        # Stage 2: Load inventories
        print("[BOOTSTRAP] Stage 2: Loading tool/command inventories...")
        self.tools = get_tools()
        self.commands = get_commands()
        
        # Stage 3: Initialize systems
        print("[BOOTSTRAP] Stage 3: Initializing systems...")
        self.tool_pool = assemble_tool_pool()
        self.execution_registry = build_execution_registry()
        self.cost_tracker = CostTracker()
        
        # Stage 4: Create query engine
        print("[BOOTSTRAP] Stage 4: Creating query engine...")
        from query_engine import QueryEngineConfig
        config = QueryEngineConfig(max_turns=max_turns)
        self.query_engine = QueryEnginePort.from_workspace()
        self.query_engine.config = config
        self.query_engine.session_id = self.session_id
        
        # Stage 5: Setup runtime
        print("[BOOTSTRAP] Stage 5: Setting up runtime...")
        self.runtime = PortRuntime()
        
        # Bootstrap complete
        print(f"[BOOTSTRAP] Session {self.session_id} ready\n")
    
    def execute_request(self, user_prompt: str) -> dict:
        """
        Execute a complete request through the agent.
        
        Maps directly to the architecture:
        1. Routing Phase
        2. Permission Phase
        3. Execution Phase
        4. Response Phase
        """
        
        print(f"[REQUEST] User: {user_prompt}")
        print(f"[SESSION] {self.session_id} | Turn {len(self.query_engine.mutable_messages) + 1}")
        print()
        
        # === ROUTING PHASE ===
        print("[ROUTING] Analyzing request...")
        routed = self.runtime.route_prompt(user_prompt, limit=5)
        
        matched_commands = tuple(m.name for m in routed if m.kind == 'command')
        matched_tools = tuple(m.name for m in routed if m.kind == 'tool')
        
        print(f"[ROUTING] Matched commands: {matched_commands}")
        print(f"[ROUTING] Matched tools: {matched_tools}")
        print()
        
        # === PERMISSION PHASE ===
        print("[PERMISSION] Checking access...")
        denied = []
        allowed_tools = list(matched_tools)
        
        for tool_name in matched_tools:
            if self.permission_context.blocks(tool_name):
                denial = PermissionDenial(
                    tool_name=tool_name,
                    reason=f"User {self.user_id} (role: {self.role}) denied"
                )
                denied.append(denial)
                allowed_tools.remove(tool_name)
                print(f"[PERMISSION] ❌ {tool_name} - DENIED")
            else:
                print(f"[PERMISSION] ✅ {tool_name} - ALLOWED")
        
        print()
        
        # === EXECUTION PHASE ===
        print("[EXECUTION] Running tools/commands...")
        execution_messages = []
        
        for tool_name in allowed_tools:
            # Get executor
            executor = self.execution_registry.get_executor(tool_name)
            if not executor:
                print(f"[EXECUTION] ⚠️  No executor for {tool_name}")
                continue
            
            # Check tool pool
            tool_entry = self.tool_pool.entries.get(tool_name)
            if not tool_entry or not tool_entry.enabled:
                print(f"[EXECUTION] ⚠️  {tool_name} not enabled")
                continue
            
            # Execute
            try:
                print(f"[EXECUTION] Running {tool_name}...")
                output = executor.execute(tool_name, user_prompt)
                execution_messages.append(output)
                
                # Track cost
                tokens_used = len(output.split()) * 1.3
                self.cost_tracker.log_cost(tool_name, int(tokens_used), self.session_id)
                print(f"[EXECUTION] ✅ {tool_name} completed ({tokens_used:.0f} tokens)")
            except Exception as e:
                print(f"[EXECUTION] ❌ {tool_name} failed: {e}")
        
        print()
        
        # === RESPONSE PHASE ===
        print("[RESPONSE] Compiling results...")
        
        combined_output = '\n'.join(execution_messages) if execution_messages else "No tools executed"
        
        result = self.query_engine.submit_message(
            prompt=user_prompt,
            matched_commands=matched_commands,
            matched_tools=tuple(allowed_tools),
            denied_tools=tuple(denied)
        )
        
        # === PERSISTENCE PHASE ===
        print("[PERSISTENCE] Saving state...")
        session_path = save_session(
            self.session_id,
            self.query_engine.mutable_messages,
            self.query_engine.total_usage
        )
        print(f"[PERSISTENCE] Saved to {session_path}")
        
        print(f"\n[RESPONSE] {combined_output}\n")
        
        return {
            'session_id': self.session_id,
            'user_prompt': user_prompt,
            'matched_commands': matched_commands,
            'matched_tools': matched_tools,
            'denied_tools': [d.tool_name for d in denied],
            'output': combined_output,
            'cost': self.cost_tracker.total_cost(self.session_id),
            'usage': result.usage
        }
    
    def generate_session_report(self) -> str:
        """Generate complete session report"""
        lines = [
            f"# Session Report: {self.session_id}",
            "",
            f"**User:** {self.user_id}",
            f"**Role:** {self.role}",
            f"**Workspace:** {self.workspace_path}",
            "",
            "## Context",
            f"- Source files: {self.context.python_file_count}",
            f"- Test files: {self.context.test_file_count}",
            "",
            "## Execution Summary",
            f"- Turns executed: {len(self.query_engine.mutable_messages)}",
            f"- Total tokens: {self.query_engine.total_usage.total_tokens}",
            f"- Total cost: ${self.cost_tracker.total_cost(self.session_id):.2f}",
            "",
            "## Tools Available",
            f"- Total tools: {len(self.tools)}",
            f"- Enabled in pool: {len(self.tool_pool.get_enabled_tools())}",
            "",
            "## Permission Summary",
            f"- User role: {self.role}",
            f"- Tools denied: {len([d for d in self.query_engine.permission_denials])}",
        ]
        
        return '\n'.join(lines)
```

## Part 3: Multi-Turn Conversation

### Complete Conversation Example

```python
def run_multi_turn_conversation():
    """Demonstrate complete multi-turn conversation"""
    
    # Create agent
    agent = ProductionAgent(
        workspace_path=Path('./claw-code-main'),
        user_id='alice',
        role='user',
        max_turns=3
    )
    
    # Conversation turns
    prompts = [
        "List Python files in the src directory",
        "Count the total lines of code",
        "Generate a code summary"
    ]
    
    results = []
    for prompt in prompts:
        result = agent.execute_request(prompt)
        results.append(result)
        
        # Check if we should continue
        if result['execution'].stop_reason in ('max_turns_reached', 'budget_exceeded'):
            print("[SESSION] Stopping - limit reached")
            break
    
    # Final report
    print("\n" + "="*60)
    print("SESSION REPORT")
    print("="*60)
    print(agent.generate_session_report())
    
    return results
```

## Part 4: Error Handling & Recovery

### Robust Error Handler

```python
class RobustAgent(ProductionAgent):
    """Agent with advanced error handling"""
    
    def execute_request_safely(self, user_prompt: str) -> dict:
        """Execute with comprehensive error handling"""
        
        try:
            return self.execute_request(user_prompt)
        
        except PermissionError as e:
            print(f"[ERROR] Permission denied: {e}")
            return {
                'status': 'error',
                'error_type': 'permission',
                'message': str(e)
            }
        
        except TimeoutError as e:
            print(f"[ERROR] Execution timeout: {e}")
            return {
                'status': 'error',
                'error_type': 'timeout',
                'message': str(e)
            }
        
        except BudgetExceeded as e:
            print(f"[ERROR] Budget exceeded: {e}")
            # Graceful degradation: save and exit
            self.save_session()
            return {
                'status': 'error',
                'error_type': 'budget_exceeded',
                'message': str(e)
            }
        
        except Exception as e:
            print(f"[ERROR] Unexpected error: {e}")
            # Log error, save state, return error
            self.save_session()
            raise
```

## Part 5: Monitoring and Debugging

### Session Monitor

```python
class SessionMonitor:
    """Monitor and debug agent sessions"""
    
    def __init__(self, agent: ProductionAgent):
        self.agent = agent
        self.events = []
    
    def log_event(self, event_type: str, details: dict):
        """Log monitoring event"""
        event = {
            'timestamp': datetime.now().isoformat(),
            'type': event_type,
            'details': details
        }
        self.events.append(event)
    
    def generate_debug_report(self) -> str:
        """Generate debug report"""
        lines = ["# Debug Report", ""]
        
        for event in self.events:
            lines.append(f"## [{event['type']}] {event['timestamp']}")
            for key, value in event['details'].items():
                lines.append(f"- {key}: {value}")
            lines.append("")
        
        return '\n'.join(lines)

# Usage
monitor = SessionMonitor(agent)
monitor.log_event('bootstrap', {'session_id': agent.session_id})
monitor.log_event('execution', {'tool': 'BashTool', 'tokens': 150})

print(monitor.generate_debug_report())
```

## Part 6: Hands-On Implementation

### Exercise 1: Build a Simple Agent

```python
# TODO: Create and run a simple agent
agent = ProductionAgent(
    workspace_path=Path('.'),
    user_id='student',
    role='user'
)

result = agent.execute_request("List available tools")
print(result['output'])
```

### Exercise 2: Multi-Role Scenario

```python
# TODO: Test same request with different roles
prompts = ["Run command: ls -la", "Edit file: config.py", "Deploy to production"]
roles = ['user', 'admin', 'readonly']

for role in roles:
    agent = ProductionAgent(
        workspace_path=Path('.'),
        user_id='test_user',
        role=role
    )
    
    for prompt in prompts:
        result = agent.execute_request(prompt)
        print(f"{role}: {len(result['denied_tools'])} tools denied")
```

### Exercise 3: Budget Test

```python
# TODO: Test behavior when budget is exceeded
from query_engine import QueryEngineConfig

agent = ProductionAgent(
    workspace_path=Path('.'),
    user_id='budget_tester',
    role='user',
    max_turns=1  # Strict limit
)

# First request (succeeds)
result1 = agent.execute_request("First request")
print(f"Result 1: {result1['stop_reason']}")

# Second request (hits limit)
result2 = agent.execute_request("Second request")
print(f"Result 2: Should show max_turns_reached")
```

## Key Integration Patterns

| Pattern | Purpose | Example |
|---------|---------|---------|
| **Request → Route → Execute → Response** | Basic flow | User input → agent → output |
| **Permission Gate** | Security | Block tool before execution |
| **Budget Check** | Resource limits | Stop if tokens exceeded |
| **Error Recovery** | Reliability | Save state on failure |
| **State Persistence** | Continuity | Sessions survive restarts |
| **Cost Tracking** | Billing/Monitoring | Track resource usage |
| **Multi-mode Execution** | Flexibility | Local/remote/SSH options |

## Key Takeaways

✅ **Bootstrap Phase** prepares all systems
✅ **Routing Phase** matches requests to tools
✅ **Permission Phase** enforces access control
✅ **Execution Phase** runs matched tools
✅ **Response Phase** returns results
✅ **Persistence Phase** saves state
✅ All phases work together seamlessly

## From Beginner to Builder

### What You Now Understand

**Level 0** ✅ What is an agent conceptually  
**Level 1** ✅ How tools and commands work  
**Level 2** ✅ How query engine and routing work  
**Level 3** ✅ How runtime and sessions work  
**Level 4** ✅ How permissions and advanced features work  
**Level 5** ✅ How everything integrates together  

### You Can Now Build

✅ Custom tools and commands  
✅ Route requests intelligently  
✅ Manage sessions and state  
✅ Implement permission systems  
✅ Monitor costs and usage  
✅ Handle errors gracefully  
✅ Deploy production agents  

---

## Resources for Next Steps

### To Deepen Understanding
- Read `src/main.py` - Entry point
- Explore `src/models.py` - Data structures
- Review `reference_data/` - Available tools/commands
- Check `tests/` - Real usage patterns

### To Build More
- Create custom tools in tool_pool
- Implement custom commands
- Add new routing strategies
- Extend permission systems
- Build monitoring solutions

### To Deploy
- Set up persistent storage
- Configure cloud execution modes
- Implement API endpoints
- Add web interface
- Scale to multiple users

---

**Congratulations!** 🎉

You've completed the comprehensive CLAW-Code agent understanding journey. You now have the knowledge to build, customize, and operate intelligent agent systems.

The agent architecture you've learned is the foundation of modern AI-powered automation. Everything from code generation to autonomous workflows to multi-agent coordination builds on these core concepts.

**Happy Building!**
