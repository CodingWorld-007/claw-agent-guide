# Level 2: Summary & Navigation

## 📚 Documents in This Level

### 00_QUERY_ENGINE_AND_ROUTING.md
**Duration:** 60-90 minutes  
**Difficulty:** ⭐⭐⭐ Intermediate-Advanced

Learn how the agent decides which tools to use. Covers query engine, routing, and turn-based systems.

**Prerequisite:** Level 1 (Tools & Commands)

**Topics covered:**
- Query engine architecture
- QueryEngineConfig and budgets
- TurnResult structure
- Routing mechanisms
- Token-based matching
- Scoring and relevance
- Turn-based conversation loops
- Budget management (tokens, turns)
- Custom routing strategies
- Multi-turn examples
- Complete turn loop implementation

**Outcome:** Understand how agents make intelligent routing decisions.

---

## 🎯 Learning Objectives

After completing Level 2, you should be able to:

- [ ] Explain what the query engine does
- [ ] Understand how routing works via token matching
- [ ] Know what a "turn" is
- [ ] Implement custom routing logic
- [ ] Manage token budgets
- [ ] Build multi-turn conversations
- [ ] Score routing matches
- [ ] Handle turn limits

---

## 💡 Key Concepts

### Query Engine = The Brain
The query engine:
- Listens to user requests
- Routes them to tools/commands
- Tracks state and usage
- Enforces budgets
- Returns formatted results

### Routing = Smart Matching
```
User: "Build a Docker container and deploy"

Tokenize: {"build", "docker", "container", "deploy"}

Match:
- BuildDockerCommand → score: 2
- DeployCommand → score: 2  
- DockerTool → score: 1

Result: [BuildDockerCommand, DeployCommand, DockerTool]
```

### Turn = One Interaction
```
Turn 1: User asks → Agent responds → Save state
Turn 2: User asks → Agent responds (uses Turn 1 context) → Save state
Turn 3: User asks → Agent responds (uses Turns 1-2) → Save state
```

### Budget = Limits
```
max_turns: 8           # Stop after 8 turns
max_budget_tokens: 2000 # Stop after 2000 tokens
```

---

## 📊 Architecture

```
┌──────────────────┐
│   User Request   │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────┐
│   Tokenize & Parse           │
│   "build docker deploy"      │
│   → {build, docker, deploy}  │
└────────┬─────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│   Token Matching             │
│   Find matching tools/cmds   │
│   Score by frequency         │
└────────┬─────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│   Routing Result             │
│   1. BuildDockerCommand (2)  │
│   2. DeployCommand (2)       │
│   3. DockerTool (1)          │
└────────┬─────────────────────┘
         │
         ▼
┌──────────────────┐
│   Return Matches │
└──────────────────┘
```

---

## 💻 Code Patterns

### Basic Routing
```python
from runtime import PortRuntime

runtime = PortRuntime()
matches = runtime.route_prompt("Build Docker container", limit=5)

for match in matches:
    print(f"{match.kind}: {match.name} (score: {match.score})")
```

### Multi-Turn Session
```python
from query_engine import QueryEnginePort

engine = QueryEnginePort.from_workspace()

# Turn 1
result1 = engine.submit_message("First request")

# Turn 2 (has Turn 1 context)
result2 = engine.submit_message("Second request")

# Turn 3 (has Turns 1-2 context)
result3 = engine.submit_message("Third request")
```

### Custom Routing
```python
class SmartRouter:
    def route_with_weights(self, prompt, tools, weights):
        # Token matching with custom weights
        tokens = set(prompt.lower().split())
        matches = []
        
        for tool in tools:
            score = len(tokens & set(tool.name.lower().split()))
            weighted = score * weights.get(tool.name, 1.0)
            matches.append((tool.name, weighted))
        
        return sorted(matches, key=lambda x: x[1], reverse=True)
```

---

## 🧪 Hands-On Exercises

### Exercise 1: Simple Routing
Route different prompts and observe matches:
```python
runtime = PortRuntime()
prompts = [
    "Read configuration file",
    "Execute bash commands",
    "Deploy to production"
]

for prompt in prompts:
    matches = runtime.route_prompt(prompt, limit=3)
    print(f"'{prompt}' → {len(matches)} matches")
```

### Exercise 2: Token Counting
Understand token estimation:
```python
def estimate_tokens(text):
    return len(text.split()) * 1.3

texts = ["hello", "This is longer", "Very extensive piece"]
for text in texts:
    print(f"{text}: ~{estimate_tokens(text):.0f} tokens")
```

### Exercise 3: Multi-Turn Conversation
Simulate a conversation:
```python
engine = QueryEnginePort.from_workspace()

for i in range(3):
    result = engine.submit_message(f"Turn {i+1} prompt")
    print(f"Messages: {len(engine.mutable_messages)}")
    print(f"Tokens: {engine.total_usage.total_tokens}")
```

---

## 🔄 Navigation

**← Previous Level:** [Level 1: Tools & Commands](../Level_one_tools_and_commands/)

**→ Next Level:** [Level 3: Runtime & Sessions](../Level_three_runtime_and_session/)

**Start with:** [Query Engine & Routing](./00_QUERY_ENGINE_AND_ROUTING.md) →

---

## ❓ Common Questions

**Q: How does routing know which tool to pick?**  
A: Token matching - counts how many words from the request match each tool name.

**Q: What if multiple tools have same score?**  
A: Order is determined by position in the matched list; can add secondary sorting.

**Q: What happens when the budget is exceeded?**  
A: Execution stops and returns `budget_exceeded` stop reason.

**Q: Can I change the routing strategy?**  
A: Yes! You can implement custom routers as shown in examples.

**Q: How long can a session last?**  
A: Limited by max_turns (default 8) or max_budget_tokens (default 2000).

---

**Ready?** Start with [Query Engine & Routing](./00_QUERY_ENGINE_AND_ROUTING.md)
