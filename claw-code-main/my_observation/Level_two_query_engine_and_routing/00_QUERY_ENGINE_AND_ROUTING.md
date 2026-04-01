# Level 2: The Query Engine & Routing 🧠

## What You'll Learn
- How the query engine processes requests
- How routing matches requests to tools/commands
- How the turn-based system works
- How to implement custom routing logic
- Real examples from CLAW-Code

## Part 1: The Query Engine Overview

### What is the Query Engine?

The **Query Engine** is the "brain" of the agent. It:
1. Accepts user requests (prompts)
2. Routes them to appropriate tools/commands
3. Executes them within budget constraints
4. Returns results
5. Saves state for next turn

```python
# From query_engine.py
from dataclasses import dataclass

@dataclass(frozen=True)
class QueryEngineConfig:
    max_turns: int = 8              # Max interactions per session
    max_budget_tokens: int = 2000   # Token budget
    compact_after_turns: int = 12   # Compress transcript after N turns
    structured_output: bool = False # Return structured data
    structured_retry_limit: int = 2 # Retry attempts for structured output

@dataclass(frozen=True)
class TurnResult:
    prompt: str                          # Original request
    output: str                          # Result produced
    matched_commands: tuple[str, ...]    # Commands that matched
    matched_tools: tuple[str, ...]       # Tools that matched
    permission_denials: tuple[...]       # Denied tool accesses
    usage: UsageSummary                  # Token usage this turn
    stop_reason: str                     # Why execution stopped
```

### The QueryEnginePort Class

```python
# From query_engine.py
@dataclass
class QueryEnginePort:
    manifest: PortManifest                    # All available tools/commands
    config: QueryEngineConfig = ...           # Configuration
    session_id: str = ...                     # Unique session ID
    mutable_messages: list[str] = ...         # Conversation history
    permission_denials: list[PermissionDenial] = ... # Tracking denials
    total_usage: UsageSummary = ...           # Cumulative token usage
    transcript_store: TranscriptStore = ...   # Full conversation record

    @classmethod
    def from_workspace(cls) -> 'QueryEnginePort':
        """Create fresh query engine from current workspace"""
        return cls(manifest=build_port_manifest())

    @classmethod
    def from_saved_session(cls, session_id: str) -> 'QueryEnginePort':
        """Load query engine from previous session"""
        stored = load_session(session_id)
        return cls(
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
            ),
        )

    def submit_message(
        self,
        prompt: str,
        matched_commands: tuple[str, ...] = (),
        matched_tools: tuple[str, ...] = (),
        denied_tools: tuple[PermissionDenial, ...] = (),
    ) -> TurnResult:
        """Process a user message and return result"""
        # Check turn limit
        if len(self.mutable_messages) >= self.config.max_turns:
            return TurnResult(
                prompt=prompt,
                output=f'Max turns reached: {prompt}',
                matched_commands=matched_commands,
                matched_tools=matched_tools,
                permission_denials=denied_tools,
                usage=self.total_usage,
                stop_reason='max_turns_reached'
            )
        
        # Process the message (internally)
        # - Execute matched tools/commands
        # - Track token usage
        # - Update transcript
        # - Return results
```

## Part 2: How Routing Works

### Routing Mechanism in PortRuntime

```python
# From runtime.py
class PortRuntime:
    def route_prompt(self, prompt: str, limit: int = 5) -> list[RoutedMatch]:
        """
        Route a prompt to matching tools/commands.
        
        Process:
        1. Tokenize prompt (split by space, remove special chars)
        2. Match tokens against tool/command names (case-insensitive)
        3. Score matches by frequency
        4. Return top matches
        """
        # Step 1: Tokenize
        tokens = {
            token.lower() 
            for token in prompt.replace('/', ' ').replace('-', ' ').split() 
            if token
        }
        
        # Step 2: Collect matches by kind
        by_kind = {
            'command': self._collect_matches(tokens, PORTED_COMMANDS, 'command'),
            'tool': self._collect_matches(tokens, PORTED_TOOLS, 'tool'),
        }
        
        # Step 3: Select best matches
        selected: list[RoutedMatch] = []
        for kind in ('command', 'tool'):
            if by_kind[kind]:
                selected.append(by_kind[kind].pop(0))
        
        return selected[:limit]
    
    def _collect_matches(
        self,
        tokens: set[str],
        inventory: tuple[PortingModule, ...],
        kind: str
    ) -> list[RoutedMatch]:
        """Find matching items by token frequency"""
        matches = []
        for module in inventory:
            # Count how many tokens match this module's name
            name_tokens = set(module.name.lower().split('_'))
            score = len(tokens & name_tokens)
            
            if score > 0:
                matches.append(RoutedMatch(
                    kind=kind,
                    name=module.name,
                    source_hint=module.source_hint,
                    score=score
                ))
        
        # Sort by score (highest first)
        return sorted(matches, key=lambda m: m.score, reverse=True)
```

### Routing Example Walkthrough

**User Input:** "Please build and run the Docker container"

**Step 1: Tokenization**
```
Original: "Please build and run the Docker container"
Tokens: {"please", "build", "and", "run", "the", "docker", "container"}
```

**Step 2: Token Matching**
```
Against Commands:
- "BuildDocker" match: {"build", "docker"} → Score: 2
- "RunContainer" match: {"run", "container"} → Score: 2
- "DeployApp" match: {} → Score: 0

Against Tools:
- "BashTool" match: {} → Score: 0
- "DockerTool" match: {"docker"} → Score: 1
```

**Step 3: Selection (limit=2)**
```
Selected:
1. [Command] BuildDocker (score: 2)
2. [Tool] DockerTool (score: 1)
```

## Part 3: The Turn-Based System

### What is a "Turn"?

A **turn** is a complete cycle of:
```
User Request → Query Engine → Tool/Command Execution → Response → Save State
```

### Turn Flow

```python
# Typical turn flow
def run_turn(query_engine: QueryEnginePort, user_prompt: str):
    print(f"Turn {len(query_engine.mutable_messages) + 1}")
    
    # 1. Route the prompt
    runtime = PortRuntime()
    routed = runtime.route_prompt(user_prompt)
    
    # 2. Extract command/tool names
    commands = tuple(m.name for m in routed if m.kind == 'command')
    tools = tuple(m.name for m in routed if m.kind == 'tool')
    
    # 3. Check permissions
    denied = []
    # ... permission checking logic ...
    
    # 4. Submit and get result
    result = query_engine.submit_message(
        prompt=user_prompt,
        matched_commands=commands,
        matched_tools=tools,
        denied_tools=tuple(denied)
    )
    
    # 5. Return result
    return result
```

### Multi-Turn Conversation Example

```python
# Simulating a 3-turn conversation
query_engine = QueryEnginePort.from_workspace()

# Turn 1
result1 = query_engine.submit_message(
    "List Python files in src directory"
)
print(result1.output)

# Turn 2 (builds on Turn 1's context)
result2 = query_engine.submit_message(
    "Count lines in those files"
)
print(result2.output)

# Turn 3 (continues conversation)
result3 = query_engine.submit_message(
    "Generate a summary"
)
print(result3.output)

# Query engine remembers all messages and state
print(f"Total turns: {len(query_engine.mutable_messages)}")
print(f"Total usage: {query_engine.total_usage}")
```

## Part 4: Budget Management

### Token Budget

Each session has a token budget:

```python
config = QueryEngineConfig(
    max_budget_tokens=2000,  # Maximum 2000 tokens per session
    max_turns=8              # Or maximum 8 turns
)

# In submit_message, track usage:
def submit_message(self, prompt: str, ...):
    # Rough token estimate: word count * 1.3
    tokens_used = len(prompt.split()) * 1.3
    
    if self.total_usage.input_tokens + tokens_used > self.config.max_budget_tokens:
        return TurnResult(
            output="Token budget exceeded",
            stop_reason="budget_exceeded"
        )
```

### Turn Limit

Prevents infinite loops:

```python
if len(self.mutable_messages) >= self.config.max_turns:
    return TurnResult(
        output="Maximum turns reached",
        stop_reason="max_turns_reached"
    )
```

## Part 5: Implementing Custom Routing

### Custom Routing Strategy

```python
class SmartRouter:
    """Advanced routing with weighted matching"""
    
    def route_with_weights(
        self,
        prompt: str,
        tools: tuple[PortingModule, ...],
        weights: dict[str, float] = None
    ) -> list[RoutedMatch]:
        """
        Route using weighted relevance scoring.
        
        Example weights:
        {
            "BashTool": 1.5,      # Prefer bash tools
            "FileEditTool": 1.0,
            "DatabaseTool": 0.5   # Deprioritize database
        }
        """
        weights = weights or {}
        tokens = set(prompt.lower().split())
        
        matches = []
        for tool in tools:
            # Base score: token match count
            token_score = len(tokens & set(tool.name.lower().split('_')))
            
            # Apply weight
            weight = weights.get(tool.name, 1.0)
            final_score = token_score * weight
            
            if final_score > 0:
                matches.append(RoutedMatch(
                    kind='tool',
                    name=tool.name,
                    source_hint=tool.source_hint,
                    score=int(final_score)
                ))
        
        return sorted(matches, key=lambda m: m.score, reverse=True)
```

## Part 6: Practical Implementation

### Complete Turn Loop Example

```python
from query_engine import QueryEnginePort, QueryEngineConfig
from runtime import PortRuntime

def run_conversation(initial_prompt: str, max_turns: int = 3):
    """Run a simple turn-based conversation"""
    
    # Initialize
    config = QueryEngineConfig(max_turns=max_turns, max_budget_tokens=5000)
    query_engine = QueryEnginePort.from_workspace()
    query_engine.config = config
    runtime = PortRuntime()
    
    print(f"Starting session: {query_engine.session_id}")
    print(f"Configuration: max_turns={config.max_turns}, max_budget={config.max_budget_tokens}")
    print()
    
    # First turn
    current_prompt = initial_prompt
    
    while len(query_engine.mutable_messages) < config.max_turns:
        print(f"Turn {len(query_engine.mutable_messages) + 1}")
        print(f"Prompt: {current_prompt}")
        
        # Route
        routed = runtime.route_prompt(current_prompt, limit=3)
        print(f"Routing discovered: {len(routed)} matches")
        for match in routed:
            print(f"  - [{match.kind}] {match.name} (score: {match.score})")
        
        # Extract matched items
        commands = tuple(m.name for m in routed if m.kind == 'command')
        tools = tuple(m.name for m in routed if m.kind == 'tool')
        
        # Submit
        result = query_engine.submit_message(
            prompt=current_prompt,
            matched_commands=commands,
            matched_tools=tools
        )
        
        print(f"Result: {result.output}")
        print(f"Usage: {result.usage}")
        print(f"Stop reason: {result.stop_reason}")
        print()
        
        # For demo, break after first turn
        # In real scenario, you'd get next prompt from user
        break
    
    return query_engine

# Run it
engine = run_conversation("Build a Docker container for my app")
```

## Part 7: Hands-On Exercises

### Exercise 1: Simple Routing
```python
# TODO: Route different prompts and see results
from runtime import PortRuntime

runtime = PortRuntime()

prompts = [
    "Read the configuration file",
    "Execute bash commands",
    "Deploy to production",
    "Check database status"
]

for prompt in prompts:
    matches = runtime.route_prompt(prompt, limit=3)
    print(f"'{prompt}' → {len(matches)} matches")
```

### Exercise 2: Token Counting
```python
# TODO: Understand token vs word counting
def estimate_tokens(text: str) -> int:
    # Rough estimate: word count * 1.3
    return len(text.split()) * 1.3

texts = [
    "hello world",
    "This is a longer sentence with more content",
    "A short one"
]

for text in texts:
    tokens = estimate_tokens(text)
    print(f"'{text}': ~{tokens:.0f} tokens")
```

### Exercise 3: Multi-Turn Session
```python
# TODO: Simulate a multi-turn conversation
from query_engine import QueryEnginePort

engine = QueryEnginePort.from_workspace()

# Simulate 3 turns
for i in range(3):
    result = engine.submit_message(f"Turn {i+1} prompt")
    print(f"Turn {i+1}: {result.stop_reason}")
    print(f"Total messages: {len(engine.mutable_messages)}")
```

## Key Takeaways

✅ Query Engine is the brain that routes requests
✅ Routing uses token matching with scoring
✅ Turns exist within budget/limit constraints
✅ Every turn tracks state and usage
✅ Sessions persist across multiple turns
✅ Custom routing strategies can be implemented
✅ Budget prevents runaway execution

## Next Step → Level 3: Runtime & Session Management

Now that you understand routing and the query engine, let's learn how the **Runtime** manages sessions, persistence, and multi-turn state!

---

**Remember:** The Query Engine is decision-making, Routing is matching, Turns are cycles!
