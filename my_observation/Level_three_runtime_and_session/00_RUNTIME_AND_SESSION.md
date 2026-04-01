# Level 3: Runtime and Session Management 🏃

## What You'll Learn
- How the runtime manages sessions
- Session persistence and transcript storage
- Context and environment setup
- Multi-mode execution (local, remote, SSH)
- Session restoration and state management

## Part 1: Understanding the Runtime

### What is the Runtime?

The **Runtime** is the execution layer that:
1. Creates and manages sessions
2. Routes prompts through the query engine
3. Maintains conversation history
4. Persists state to disk
5. Handles error recovery

```python
# From runtime.py
from dataclasses import dataclass

@dataclass
class RuntimeSession:
    prompt: str                        # Original user request
    context: PortContext              # Environment context
    setup: WorkspaceSetup             # Environment setup
    setup_report: SetupReport          # Setup diagnostics
    system_init_message: str           # System initialization
    history: HistoryLog                # Conversation history
    routed_matches: list[RoutedMatch]  # Matched tools/commands
    turn_result: TurnResult            # Turn execution result
    command_execution_messages: tuple[str, ...]  # Command outputs
    tool_execution_messages: tuple[str, ...]     # Tool outputs
    stream_events: tuple[dict, ...]               # Event log
    persisted_session_path: str                   # Where it was saved

class PortRuntime:
    def route_prompt(self, prompt: str, limit: int = 5) -> list[RoutedMatch]:
        """Route a prompt to tools/commands"""
        ...
```

### Session Structure

```
Session
├── Session ID (unique identifier)
├── Context
│   ├── Source root
│   ├── Test root
│   ├── Assets root
│   └── Python/Test file counts
├── Setup
│   ├── Python version
│   ├── Platform
│   └── Environment variables
├── History
│   ├── Message 1
│   ├── Message 2
│   └── Message 3...
├── Permission Context
│   └── Allowed/denied tools
└── Transcript
    └── Full conversation record
```

## Part 2: Session Context

### PortContext - Environment Information

```python
# From context.py
from dataclasses import dataclass
from pathlib import Path

@dataclass(frozen=True)
class PortContext:
    source_root: Path           # src/ directory
    tests_root: Path            # tests/ directory
    assets_root: Path           # assets/ directory
    archive_root: Path          # archived snapshot
    python_file_count: int      # Number of .py files
    test_file_count: int        # Number of test files
    asset_file_count: int       # Number of assets
    archive_available: bool     # Is archive present

def build_port_context(base: Path | None = None) -> PortContext:
    """Build context from workspace structure"""
    root = base or Path(__file__).resolve().parent.parent
    source_root = root / 'src'
    tests_root = root / 'tests'
    assets_root = root / 'assets'
    archive_root = root / 'archive' / 'claude_code_ts_snapshot' / 'src'
    
    return PortContext(
        source_root=source_root,
        tests_root=tests_root,
        assets_root=assets_root,
        archive_root=archive_root,
        python_file_count=sum(1 for p in source_root.rglob('*.py') if p.is_file()),
        test_file_count=sum(1 for p in tests_root.rglob('*.py') if p.is_file()),
        asset_file_count=sum(1 for p in assets_root.rglob('*') if p.is_file()),
        archive_available=archive_root.exists(),
    )

def render_context(context: PortContext) -> str:
    """Format context for display"""
    return '\n'.join([
        f'Source root: {context.source_root}',
        f'Test root: {context.tests_root}',
        f'Assets root: {context.assets_root}',
        f'Python files: {context.python_file_count}',
        f'Test files: {context.test_file_count}',
    ])
```

## Part 3: Session Storage and Persistence

### Session Store

```python
# From session_store.py (typical implementation)
from dataclasses import dataclass
from pathlib import Path
import json

@dataclass
class StoredSession:
    session_id: str
    messages: list[str]
    input_tokens: int
    output_tokens: int
    created_at: str
    updated_at: str

def save_session(
    session_id: str,
    messages: list[str],
    usage: UsageSummary,
    storage_path: Path = None
) -> Path:
    """Persist session to disk"""
    storage_path = storage_path or Path.home() / '.claw' / 'sessions'
    storage_path.mkdir(parents=True, exist_ok=True)
    
    session_file = storage_path / f'{session_id}.json'
    
    data = {
        'session_id': session_id,
        'messages': messages,
        'input_tokens': usage.input_tokens,
        'output_tokens': usage.output_tokens,
        'created_at': datetime.now().isoformat(),
        'updated_at': datetime.now().isoformat(),
    }
    
    session_file.write_text(json.dumps(data, indent=2))
    return session_file

def load_session(session_id: str) -> StoredSession:
    """Load session from disk"""
    storage_path = Path.home() / '.claw' / 'sessions'
    session_file = storage_path / f'{session_id}.json'
    
    if not session_file.exists():
        raise FileNotFoundError(f'Session not found: {session_id}')
    
    data = json.loads(session_file.read_text())
    return StoredSession(
        session_id=data['session_id'],
        messages=data['messages'],
        input_tokens=data['input_tokens'],
        output_tokens=data['output_tokens'],
        created_at=data['created_at'],
        updated_at=data['updated_at'],
    )
```

### Transcript Store

```python
# From transcript.py (typical implementation)
@dataclass
class TranscriptEntry:
    timestamp: str
    type: str  # 'prompt', 'tool_execution', 'command_execution', 'result'
    content: str
    metadata: dict = None

@dataclass
class TranscriptStore:
    entries: list[TranscriptEntry] = field(default_factory=list)
    flushed: bool = False  # Persisted to disk

    def add_entry(
        self,
        entry_type: str,
        content: str,
        metadata: dict = None
    ):
        """Add entry to transcript"""
        entry = TranscriptEntry(
            timestamp=datetime.now().isoformat(),
            type=entry_type,
            content=content,
            metadata=metadata or {}
        )
        self.entries.append(entry)
    
    def flush_to_file(self, path: Path):
        """Save transcript to disk"""
        data = [
            {
                'timestamp': e.timestamp,
                'type': e.type,
                'content': e.content,
                'metadata': e.metadata
            }
            for e in self.entries
        ]
        path.write_text(json.dumps(data, indent=2))
        self.flushed = True
    
    def as_markdown(self) -> str:
        """Format transcript as markdown"""
        lines = ['# Transcript', '']
        for entry in self.entries:
            lines.append(f'## [{entry.type}] {entry.timestamp}')
            lines.append(entry.content)
            if entry.metadata:
                lines.append(f'_Metadata: {entry.metadata}_')
            lines.append('')
        return '\n'.join(lines)
```

## Part 4: Workspace Setup

### Setup and Initialization

```python
# From setup.py (typical implementation)
from dataclasses import dataclass
import subprocess
import sys

@dataclass
class WorkspaceSetup:
    python_version: str
    implementation: str  # CPython, PyPy, etc.
    platform_name: str
    
    def startup_steps(self) -> list[str]:
        """Steps taken during startup"""
        return [
            f'Python {self.python_version} ({self.implementation})',
            f'Platform: {self.platform_name}',
            'Tools loaded from snapshot',
            'Commands loaded from snapshot',
            'Bootstrap graph compiled',
            'Permission context initialized',
        ]

@dataclass
class SetupReport:
    workspace_ready: bool
    dependencies_available: bool
    permissions_loaded: bool
    test_command: str

def run_setup() -> tuple[WorkspaceSetup, SetupReport]:
    """Initialize workspace"""
    # Detect Python
    setup = WorkspaceSetup(
        python_version='.'.join(map(str, sys.version_info[:3])),
        implementation=sys.implementation.name,
        platform_name=sys.platform
    )
    
    # Check readiness
    report = SetupReport(
        workspace_ready=True,
        dependencies_available=True,
        permissions_loaded=True,
        test_command='pytest tests/'
    )
    
    return setup, report
```

## Part 5: Multi-Mode Execution

### Execution Modes

CLAW-Code supports multiple execution modes:

```python
# From main.py / remote_runtime.py

# 1. LOCAL MODE (default)
#    - Execute directly on local machine
#    - Direct access to file system

# 2. REMOTE MODE
#    - Execute on remote server
#    - Via SSH connection

def run_remote_mode(target: str):
    """Execute code on remote host"""
    # Connect to target
    # Setup remote environment
    # Execute tools remotely
    # Stream results back
    pass

# 3. SSH MODE
#    - Secure shell connection
#    - KeyFile-based authentication

def run_ssh_mode(target: str):
    """Execute via SSH"""
    pass

# 4. TELEPORT MODE
#    - Multi-hop execution
#    - Through intermediate servers

def run_teleport_mode(target: str):
    """Execute via teleport"""
    pass

# 5. DIRECT-CONNECT MODE
#    - P2P connection
#    - No intermediaries

def run_direct_connect_mode(target: str):
    """Direct peer-to-peer execution"""
    pass

# 6. DEEP-LINK MODE
#    - Protocol handler
#    - Custom routing

def run_deep_link_mode(target: str):
    """Execute via deep link protocol"""
    pass
```

## Part 6: Complete Session Lifecycle

### Creating a New Session

```python
from datetime import datetime
from uuid import uuid4

def create_new_session() -> RuntimeSession:
    """Create brand new session"""
    session_id = uuid4().hex
    context = build_port_context()
    setup, setup_report = run_setup()
    
    session = RuntimeSession(
        prompt="",
        context=context,
        setup=setup,
        setup_report=setup_report,
        system_init_message=build_system_init_message(),
        history=HistoryLog(),
        routed_matches=[],
        turn_result=None,
        command_execution_messages=(),
        tool_execution_messages=(),
        stream_events=(),
        persisted_session_path=""
    )
    
    return session
```

### Restoring a Session

```python
def restore_session(session_id: str) -> QueryEnginePort:
    """Restore previous session"""
    # Load from disk
    stored = load_session(session_id)
    
    # Rebuild state
    query_engine = QueryEnginePort(
        manifest=build_port_manifest(),
        session_id=stored.session_id,
        mutable_messages=list(stored.messages),
        total_usage=UsageSummary(
            stored.input_tokens,
            stored.output_tokens
        ),
        transcript_store=TranscriptStore(
            entries=list(stored.messages),
            flushed=True
        )
    )
    
    return query_engine
```

### Saving a Session

```python
def persist_session(
    session_id: str,
    messages: list[str],
    usage: UsageSummary,
    transcript: TranscriptStore
):
    """Save session state"""
    # Save to database/disk
    session_path = save_session(session_id, messages, usage)
    
    # Save transcript
    transcript_path = session_path.parent / f'{session_id}.transcript.md'
    transcript.flush_to_file(transcript_path)
    
    return session_path
```

## Part 7: Practical Examples

### Example 1: Complete Session Flow

```python
from runtime import PortRuntime
from query_engine import QueryEnginePort

# 1. Create fresh session
engine = QueryEnginePort.from_workspace()
runtime = PortRuntime()

print(f"Session ID: {engine.session_id}")
print(f"Config: max_turns={engine.config.max_turns}")

# 2. First turn
prompt1 = "List Python files"
routed1 = runtime.route_prompt(prompt1)
result1 = engine.submit_message(prompt1)
print(result1.output)

# 3. Second turn (with context)
prompt2 = "Count their lines"
routed2 = runtime.route_prompt(prompt2)
result2 = engine.submit_message(prompt2)
print(result2.output)

# 4. Save session
session_path = save_session(
    engine.session_id,
    engine.mutable_messages,
    engine.total_usage
)
print(f"Saved to: {session_path}")

# 5. Later, restore it
engine_restored = QueryEnginePort.from_saved_session(engine.session_id)
print(f"Restored {len(engine_restored.mutable_messages)} messages")
```

### Example 2: Context-Aware Execution

```python
def analyze_workspace():
    """Analyze current workspace using runtime"""
    context = build_port_context()
    
    print(f"Source files: {context.python_file_count}")
    print(f"Test files: {context.test_file_count}")
    print(f"Archive available: {context.archive_available}")
    
    # Create session with this context
    engine = QueryEnginePort.from_workspace()
    
    # Execute commands aware of workspace
    result = engine.submit_message(
        "Analyze the codebase metrics"
    )
    
    return result

analyze_workspace()
```

## Part 8: Hands-On Exercises

### Exercise 1: Create and Restore Session
```python
# TODO: Create a session, save it, and restore it
from uuid import uuid4

session_id = uuid4().hex
engine = QueryEnginePort.from_workspace()

# Execute some turns
engine.submit_message("First prompt")
engine.submit_message("Second prompt")

# Save
save_session(
    engine.session_id,
    engine.mutable_messages,
    engine.total_usage
)

# Restore
restored = QueryEnginePort.from_saved_session(engine.session_id)
print(f"Original: {len(engine.mutable_messages)} messages")
print(f"Restored: {len(restored.mutable_messages)} messages")
```

### Exercise 2: Context Analysis
```python
# TODO: Analyze workspace context
context = build_port_context()
print(render_context(context))
```

### Exercise 3: Multi-Mode Routing
```python
# TODO: See how different modes would work
runtime = PortRuntime()

# Local
prompt = "run tests"
local_matches = runtime.route_prompt(prompt)

# Would be same routing in all modes
# Just different execution target
```

## Key Takeaways

✅ Runtime manages sessions and state
✅ Context captures workspace information
✅ Sessions persist and can be restored
✅ Transcripts record full conversation history
✅ Multiple execution modes supported
✅ Setup and initialization is critical
✅ Session ID uniquely identifies each conversation

## Next Step → Level 4: Advanced Features

Now that you understand runtime and sessions, let's explore **permissions, security, and advanced system features**!

---

**Remember:** Runtime is execution context, SessionStore is persistence, Transcript is history!
