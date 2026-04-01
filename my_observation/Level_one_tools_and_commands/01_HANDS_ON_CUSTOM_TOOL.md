# Level 1: Code-Along - Building Your First Tool

## Introduction
In this hands-on guide, you'll create your first custom tool and understand how it integrates with the CLAW-Code system.

## What You'll Build
A **FileCountTool** that counts the total lines of code in a project.

## Step 1: Understanding the Tool Structure

Every tool needs:
1. **Name** - Unique identifier
2. **Responsibility** - What it does
3. **Source Hint** - Where it comes from
4. **Execution Logic** - How it works

## Step 2: Create Tool Definition

Add to `reference_data/tools_snapshot.json`:
```json
{
    "name": "FileCountTool",
    "responsibility": "Count total lines in files matching pattern",
    "source_hint": "utils/analyzer"
}
```

## Step 3: Loading the New Tool

```python
# In your code
from tools import get_tool

tool = get_tool("FileCountTool")
print(tool.name)           # Output: FileCountTool
print(tool.responsibility) # Count total lines...
```

## Step 4: Creating the Execution Function

In a real implementation, you'd add this to your tools module:

```python
def execute_file_count_tool(pattern: str = "*.py") -> dict:
    """
    Count lines in files matching pattern.
    
    Args:
        pattern: File glob pattern (default: *.py)
    
    Returns:
        Dictionary with file count and total lines
    """
    import glob
    from pathlib import Path
    
    files = glob.glob(f"**/{pattern}", recursive=True)
    total_lines = 0
    
    for file_path in files:
        try:
            with open(file_path, 'r', encoding='utf-8') as f:
                total_lines += len(f.readlines())
        except Exception as e:
            print(f"Error reading {file_path}: {e}")
    
    return {
        "pattern": pattern,
        "file_count": len(files),
        "total_lines": total_lines,
        "average_per_file": total_lines // len(files) if files else 0
    }
```

## Step 5: Using Your Custom Tool

```python
from tools import execute_tool

# Execute your custom tool
result = execute_tool("FileCountTool", payload="*.py")

print(f"Tool: {result.name}")
print(f"Handled: {result.handled}")
print(f"Message: {result.message}")
```

## Step 6: Integration with Commands

Now create a command that uses your tool:

In `reference_data/commands_snapshot.json`:
```json
{
    "name": "AnalyzeCodebase",
    "responsibility": "Analyze codebase metrics including line count",
    "source_hint": "commands/analysis"
}
```

## Complete Working Example

```python
# analyze.py
from pathlib import Path
from tools import get_tool, execute_tool

class ProjectAnalyzer:
    def __init__(self, root_path: Path):
        self.root = Path(root_path)
    
    def analyze(self) -> dict:
        """Run full analysis using tools"""
        results = {
            "project_root": str(self.root),
            "file_count": self._count_files(),
            "line_count": self._count_lines(),
            "language_breakdown": self._analyze_languages()
        }
        return results
    
    def _count_files(self) -> int:
        """Count all files (using tool pattern)"""
        files = list(self.root.rglob("*"))
        return len([f for f in files if f.is_file()])
    
    def _count_lines(self) -> int:
        """Count all lines (using tool pattern)"""
        total = 0
        for file in self.root.rglob("*.py"):
            try:
                with open(file) as f:
                    total += len(f.readlines())
            except:
                pass
        return total
    
    def _analyze_languages(self) -> dict:
        """Analyze language distribution"""
        extensions = {}
        for file in self.root.rglob("*"):
            if file.is_file():
                ext = file.suffix or "no_ext"
                extensions[ext] = extensions.get(ext, 0) + 1
        return extensions

# Usage
analyzer = ProjectAnalyzer("./src")
results = analyzer.analyze()
print(results)
```

## Testing Your Tool

```python
# test_file_count_tool.py
import pytest
from tools import get_tool, execute_tool

def test_tool_exists():
    """Verify tool is registered"""
    tool = get_tool("FileCountTool")
    assert tool is not None
    assert tool.name == "FileCountTool"

def test_tool_execution():
    """Test tool execution"""
    result = execute_tool("FileCountTool", "*.py")
    assert result.handled == True
    assert len(result.message) > 0

def test_tool_not_found():
    """Test handling of non-existent tool"""
    result = execute_tool("NonExistentTool", "")
    assert result.handled == False
    assert "Unknown" in result.message
```

## Key Concepts Learned

1. ✅ Tool structure and registration
2. ✅ Tool snapshot JSON format
3. ✅ Tool execution and result handling
4. ✅ Integration with commands
5. ✅ Testing custom tools

## Common Mistakes to Avoid

❌ **Don't:** Create tool without adding to snapshot
✅ **Do:** Always register tools in tools_snapshot.json

❌ **Don't:** Forget error handling in tool execution
✅ **Do:** Wrap tool logic in try-except blocks

❌ **Don't:** Return arbitrary data types
✅ **Do:** Return standardized result objects

## Next Step

Move to Level 2 to learn how the **Query Engine** discovers and routes to your custom tools!

---

**Pro Tip:** Tools should be idempotent (safe to run multiple times) and should handle errors gracefully.
