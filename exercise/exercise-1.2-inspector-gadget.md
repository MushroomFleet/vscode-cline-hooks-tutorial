# Exercise 1.2: "Inspector Gadget" - Understanding Hook Data

**Module:** Foundation - Understanding Hook Mechanics  
**Duration:** 20 minutes  
**Hook Type:** PreToolUse  
**Difficulty:** ⭐ Beginner

## Learning Objectives

By completing this exercise, you will:
- Understand the complete data structure for tool operations
- Learn the difference between stdout (JSON) and stderr (logging)
- Master inspection techniques for debugging hooks
- Identify common vs. tool-specific fields
- Gain confidence working with different tool types

## Scenario

Before building complex validation logic, you need to understand exactly what data Cline sends to your hooks. This exercise creates a diagnostic hook that captures and displays all the information available during tool operations. Think of it as X-ray vision for your hooks!

## Prerequisites

- Completed Exercise 1.1
- `jq` installed and working
- Basic understanding of JSON structure

## Why This Exercise Matters

When building real hooks, you'll often wonder:
- "What data is available for this tool?"
- "What's the structure of the parameters?"
- "How can I detect file types or command patterns?"

This inspector hook answers all these questions by showing you the raw data.

## Step-by-Step Instructions

### Step 1: Create the PreToolUse Inspector Hook

```bash
cd .clinerules/hooks

cat > PreToolUse << 'EOF'
#!/usr/bin/env bash

# Read the input
input=$(cat)

# Create a timestamped log file for inspection
timestamp=$(date +%Y%m%d-%H%M%S)
log_file="/tmp/cline-tool-inspector-${timestamp}.json"

# Save the complete JSON payload with pretty printing
echo "$input" | jq '.' > "$log_file"

# Extract key information for quick reference
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')
task_id=$(echo "$input" | jq -r '.taskId')
hook_timestamp=$(echo "$input" | jq -r '.timestamp')

# Print summary to stderr (appears in Cline output panel)
echo "╔════════════════════════════════════════╗" >&2
echo "║     TOOL OPERATION INSPECTION          ║" >&2
echo "╚════════════════════════════════════════╝" >&2
echo "Tool Name: $tool_name" >&2
echo "Task ID: $task_id" >&2
echo "Timestamp: $hook_timestamp" >&2
echo "Full data: $log_file" >&2
echo "----------------------------------------" >&2

# Show parameter count
param_count=$(echo "$input" | jq '.preToolUse.parameters | length')
echo "Parameters: $param_count field(s)" >&2

# List parameter keys
if [ "$param_count" -gt 0 ]; then
    echo "Available parameters:" >&2
    echo "$input" | jq -r '.preToolUse.parameters | keys[]' | while read key; do
        echo "  - $key" >&2
    done
fi

echo "========================================" >&2

# Don't block anything - just observe
echo '{
  "cancel": false,
  "contextModification": ""
}'
EOF

chmod +x PreToolUse
```

### Step 2: Understand the Code Structure

**Key Techniques Explained:**

```bash
log_file="/tmp/cline-tool-inspector-${timestamp}.json"
```
Creates a unique filename with timestamp so each inspection is saved separately.

```bash
echo "$input" | jq '.' > "$log_file"
```
The `.` in jq means "output everything" - this saves the complete JSON with formatting.

```bash
echo "Tool Name: $tool_name" >&2
```
The `>&2` redirects output to stderr instead of stdout. This is important because:
- **stdout** = JSON response to Cline (must be valid JSON!)
- **stderr** = logging/debugging messages (shown in VS Code Output panel)

```bash
echo "$input" | jq '.preToolUse.parameters | keys[]'
```
This extracts just the parameter names, showing what data is available.

### Step 3: Test the Inspector

Now trigger different tool operations and watch what gets captured:

#### Test 1: File Write Operation

In Cline, ask:
```
Create a new file called test.txt with the content "Hello, World!"
```

Check the inspection:
```bash
# Find the most recent inspection file
ls -lt /tmp/cline-tool-inspector-* | head -1

# View it
cat /tmp/cline-tool-inspector-20251107-103045.json | jq '.'
```

You should see something like:
```json
{
  "clineVersion": "3.1.0",
  "hookName": "PreToolUse",
  "timestamp": "2025-11-07T10:30:45Z",
  "taskId": "abc123",
  "workspaceRoots": ["/path/to/project"],
  "userId": "user-xyz",
  "preToolUse": {
    "toolName": "write_to_file",
    "parameters": {
      "path": "test.txt",
      "content": "Hello, World!"
    }
  }
}
```

#### Test 2: Command Execution

In Cline, ask:
```
Run 'ls -la' to list files
```

View the new inspection file - you'll see:
```json
{
  "preToolUse": {
    "toolName": "execute_command",
    "parameters": {
      "command": "ls -la"
    }
  }
}
```

#### Test 3: File Reading

In Cline, ask:
```
Read the contents of package.json
```

The inspection will show:
```json
{
  "preToolUse": {
    "toolName": "read_file",
    "parameters": {
      "path": "package.json"
    }
  }
}
```

### Step 4: Create a Comparison Chart

After running several operations, analyze the differences:

```bash
# View all inspection files
cat /tmp/cline-tool-inspector-*.json | jq '.preToolUse.toolName' | sort | uniq

# Count operations by type
cat /tmp/cline-tool-inspector-*.json | jq -r '.preToolUse.toolName' | sort | uniq -c
```

Create a reference document:
```bash
cat > tool-reference.md << 'EOF'
# Tool Operation Reference

## Common Fields (All Tools)
- `clineVersion`: Version of Cline
- `hookName`: Always "PreToolUse" for this hook
- `timestamp`: ISO 8601 timestamp
- `taskId`: Unique task identifier
- `workspaceRoots`: Array of workspace paths
- `userId`: User identifier

## Tool-Specific Parameters

### write_to_file
- `path`: File path (string)
- `content`: File content (string)

### execute_command
- `command`: Shell command (string)

### read_file
- `path`: File path (string)

### list_files
- `path`: Directory path (string)
- `recursive`: Boolean flag

### search_files
- `path`: Search root path
- `regex`: Search pattern
- `file_pattern`: File glob pattern
EOF
```

### Step 5: Enhanced Inspector (Optional)

Create an even more detailed version:

```bash
cat > PreToolUse << 'EOF'
#!/usr/bin/env bash

input=$(cat)
timestamp=$(date +%Y%m%d-%H%M%S)
log_file="/tmp/cline-tool-inspector-${timestamp}.json"

# Save full payload
echo "$input" | jq '.' > "$log_file"

# Extract data
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')
task_id=$(echo "$input" | jq -r '.taskId')

# Detailed console output
echo "╔════════════════════════════════════════╗" >&2
echo "║     TOOL INSPECTION REPORT             ║" >&2
echo "╚════════════════════════════════════════╝" >&2
echo "" >&2
echo "🔧 Tool: $tool_name" >&2
echo "📋 Task: $task_id" >&2
echo "📁 Log: $log_file" >&2
echo "" >&2

# Tool-specific insights
case "$tool_name" in
    "write_to_file")
        path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
        content_length=$(echo "$input" | jq -r '.preToolUse.parameters.content | length')
        echo "📝 Writing to: $path" >&2
        echo "📊 Content size: $content_length bytes" >&2
        echo "📂 Extension: ${path##*.}" >&2
        ;;
    "execute_command")
        command=$(echo "$input" | jq -r '.preToolUse.parameters.command')
        echo "⚡ Command: $command" >&2
        echo "🔐 Contains sudo: $(echo $command | grep -q sudo && echo 'YES ⚠️' || echo 'no')" >&2
        ;;
    "read_file")
        path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
        echo "📖 Reading: $path" >&2
        if [ -f "$path" ]; then
            size=$(wc -c < "$path" 2>/dev/null || echo "unknown")
            echo "📊 File size: $size bytes" >&2
        fi
        ;;
esac

echo "" >&2
echo "========================================" >&2

# Show field types for learning
echo "" >&2
echo "Field Types:" >&2
echo "$input" | jq -r 'to_entries | .[] | "\(.key): \(.value | type)"' | head -6 >&2

echo '{
  "cancel": false,
  "contextModification": ""
}'
EOF

chmod +x PreToolUse
```

## Understanding the Output

### What You See in VS Code Output Panel

When you have the enhanced inspector running, you'll see output like:

```
╔════════════════════════════════════════╗
║     TOOL INSPECTION REPORT             ║
╚════════════════════════════════════════╝

🔧 Tool: write_to_file
📋 Task: 1e2d3c4b5a
📁 Log: /tmp/cline-tool-inspector-20251107-103045.json

📝 Writing to: src/components/Header.tsx
📊 Content size: 1247 bytes
📂 Extension: tsx

========================================

Field Types:
clineVersion: string
hookName: string
timestamp: string
taskId: string
workspaceRoots: array
userId: string
```

### What You See in Log Files

The JSON files in `/tmp/` contain the complete data structure:

```bash
# Pretty-print any inspection file
jq '.' /tmp/cline-tool-inspector-20251107-103045.json

# Extract just the parameters
jq '.preToolUse.parameters' /tmp/cline-tool-inspector-*.json

# Find all file write operations
jq 'select(.preToolUse.toolName == "write_to_file") | .preToolUse.parameters.path' /tmp/cline-tool-inspector-*.json
```

## Data Structure Deep Dive

### Base Fields (Present in ALL Hooks)

```json
{
  "clineVersion": "3.1.0",        // Cline's version
  "hookName": "PreToolUse",       // Which hook this is
  "timestamp": "2025-11-07T...",  // When it happened
  "taskId": "abc123",             // Task identifier
  "workspaceRoots": ["/path"],    // Project paths
  "userId": "user-xyz"            // User identifier
}
```

### PreToolUse Specific Fields

```json
{
  "preToolUse": {
    "toolName": "string",      // Name of the tool about to execute
    "parameters": {            // Tool-specific parameters
      // Varies by tool
    }
  }
}
```

### Common Tool Parameters

| Tool Name | Parameters | Purpose |
|-----------|-----------|---------|
| `write_to_file` | `path`, `content` | Write content to a file |
| `read_file` | `path` | Read file contents |
| `execute_command` | `command` | Run shell command |
| `list_files` | `path`, `recursive` | List directory |
| `list_code_definition_names` | `path` | Extract code symbols |
| `search_files` | `path`, `regex`, `file_pattern` | Search for text |

## Practical Applications

### Use Case 1: Building Validators

Once you know the structure, you can build validators:

```bash
# Validate file extensions
if [[ "$tool_name" == "write_to_file" ]]; then
    path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
    if [[ "$path" == *.js ]]; then
        # Block it!
    fi
fi
```

### Use Case 2: Command Safety Checks

```bash
# Check for dangerous commands
if [[ "$tool_name" == "execute_command" ]]; then
    command=$(echo "$input" | jq -r '.preToolUse.parameters.command')
    if [[ "$command" == *"rm -rf"* ]]; then
        # Block it!
    fi
fi
```

### Use Case 3: Path Pattern Detection

```bash
# Detect test files
if [[ "$path" == *"test"* ]] || [[ "$path" == *"spec"* ]]; then
    # It's a test file
fi
```

## Common Issues and Solutions

### Issue: Too Many Log Files

**Solution:** Clean up old inspections:
```bash
# Keep only last 10 inspections
ls -t /tmp/cline-tool-inspector-*.json | tail -n +11 | xargs rm -f

# Or delete all
rm /tmp/cline-tool-inspector-*.json
```

### Issue: Can't Read Log Files

**Solution:** Make sure permissions are correct:
```bash
ls -l /tmp/cline-tool-inspector-*.json
# Should be readable
```

### Issue: Output Not Showing in VS Code

**Solution:**
1. Open VS Code Output panel (View > Output)
2. Select "Cline" from the dropdown
3. Make sure you're using `>&2` for logging

### Issue: jq Errors

**Solution:** Test jq queries separately:
```bash
# Test your jq query
echo '{"test": "value"}' | jq '.test'

# Check if JSON is valid
echo "$input" | jq empty
```

## Validation Checklist

- [ ] PreToolUse hook created and executable
- [ ] Inspection files appear in `/tmp/`
- [ ] Console output shows in VS Code Output panel
- [ ] Can identify different tool types
- [ ] Can extract parameters for each tool
- [ ] Understand stdout vs stderr usage

## What You Learned

✅ **Data Inspection**: How to view complete hook payloads  
✅ **Stdout vs Stderr**: Where to send JSON vs logging output  
✅ **Tool Types**: Different tools have different parameters  
✅ **Field Extraction**: Using jq to navigate JSON structures  
✅ **Debugging**: Techniques for understanding hook data  
✅ **Pattern Recognition**: Common patterns across tools

## Challenge Extensions

### Challenge 1: Parameter Type Inspector

Add type checking to your inspector:

```bash
echo "$input" | jq '.preToolUse.parameters | to_entries | .[] | "\(.key): \(.value | type)"' >&2
```

### Challenge 2: Statistics Collector

Count operations by type:

```bash
# Add to your hook
echo "$tool_name" >> /tmp/cline-tool-stats.txt

# View stats
sort /tmp/cline-tool-stats.txt | uniq -c | sort -rn
```

### Challenge 3: Content Preview

For file operations, show a preview:

```bash
if [[ "$tool_name" == "write_to_file" ]]; then
    content=$(echo "$input" | jq -r '.preToolUse.parameters.content')
    preview=$(echo "$content" | head -c 100)
    echo "Preview: $preview..." >&2
fi
```

### Challenge 4: Workspace Detection

Detect which workspace is active:

```bash
workspace=$(echo "$input" | jq -r '.workspaceRoots[0]')
workspace_name=$(basename "$workspace")
echo "Workspace: $workspace_name" >&2
```

## Next Steps

You now have the ability to inspect any hook data! This skill is invaluable for building more complex hooks.

**Next Exercise:** [Exercise 2.1: TypeScript Guardian](exercise-2.1-typescript-guardian.md)

You'll use your inspection knowledge to build a real validator that blocks unwanted file types.

## Reference

**Hook Type:** PreToolUse  
**Can Block:** Yes  
**Timing:** Before tool executes  
**Common Uses:** Inspection, validation, debugging  
**Response Required:** `{"cancel": false, "contextModification": ""}`

**Files Created:**
- `.clinerules/hooks/PreToolUse` - The inspector hook
- `/tmp/cline-tool-inspector-*.json` - Inspection logs
- `tool-reference.md` - Your reference guide (optional)

**Key Commands:**
```bash
# View latest inspection
ls -t /tmp/cline-tool-inspector-*.json | head -1 | xargs cat | jq '.'

# Count tool types
cat /tmp/cline-tool-inspector-*.json | jq -r '.preToolUse.toolName' | sort | uniq -c

# Find specific tool operations
jq 'select(.preToolUse.toolName == "write_to_file")' /tmp/cline-tool-inspector-*.json
```
