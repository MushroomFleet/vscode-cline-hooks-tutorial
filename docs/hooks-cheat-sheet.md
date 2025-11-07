# Cline Hooks Cheat Sheet

Quick reference for all hook types, patterns, and common use cases.

---

## Hook Types Overview

| Hook Name | Timing | Can Block | Primary Use |
|-----------|--------|-----------|-------------|
| **TaskStart** | When task begins | No | Initialize, detect project, set context |
| **TaskResume** | When task resumes | No | Restore state, refresh context |
| **TaskCancel** | When task cancelled | No | Cleanup, logging |
| **PreToolUse** | Before tool executes | ✅ Yes | Validation, blocking, enforcement |
| **PostToolUse** | After tool completes | No | Learning, metrics, context injection |
| **UserPromptSubmit** | User sends message | Partial | Input validation, context injection |

---

## Base JSON Structure

All hooks receive these fields:

```json
{
  "clineVersion": "string",
  "hookName": "string",
  "timestamp": "ISO8601 string",
  "taskId": "string",
  "workspaceRoots": ["string"],
  "userId": "string"
}
```

---

## Hook-Specific Data

### TaskStart

```json
{
  "taskStart": {
    "taskMetadata": {
      "taskId": "string",
      "ulid": "string",
      "initialTask": "string"  // User's original request
    }
  }
}
```

**Use Cases:**
- Detect project type
- Initialize tracking
- Set project conventions
- Log task start

### TaskResume

```json
{
  "taskResume": {
    "taskMetadata": {
      "taskId": "string",
      "ulid": "string"
    },
    "previousState": {
      "lastMessageTs": "string",
      "messageCount": "string",
      "conversationHistoryDeleted": "string"
    }
  }
}
```

**Use Cases:**
- Restore context
- Refresh project state
- Log resumption
- Update external systems

### TaskCancel

```json
{
  "taskCancel": {
    "taskMetadata": {
      "taskId": "string",
      "ulid": "string",
      "completionStatus": "string"
    }
  }
}
```

**Use Cases:**
- Cleanup resources
- Log cancellation
- Update external trackers
- Save state

### PreToolUse

```json
{
  "preToolUse": {
    "toolName": "string",  // e.g., "write_to_file"
    "parameters": {
      // Tool-specific parameters
    }
  }
}
```

**Common Tool Names:**
- `write_to_file`
- `read_file`
- `execute_command`
- `list_files`
- `search_files`
- `list_code_definition_names`

**Use Cases:**
- Validate operations
- Block dangerous actions
- Enforce policies
- Check permissions

### PostToolUse

```json
{
  "postToolUse": {
    "toolName": "string",
    "parameters": {},
    "result": "string",
    "success": boolean,
    "executionTimeMs": number
  }
}
```

**Use Cases:**
- Learn from operations
- Track performance
- Build project knowledge
- Log results
- Inject context for next actions

### UserPromptSubmit

```json
{
  "userPromptSubmit": {
    "prompt": "string",
    "attachments": ["string"]
  }
}
```

**Use Cases:**
- Validate input
- Inject contextual hints
- Track user patterns
- Redirect common requests

---

## Tool-Specific Parameters

### write_to_file

```json
{
  "path": "string",      // File path
  "content": "string"    // File content
}
```

### read_file

```json
{
  "path": "string"       // File path
}
```

### execute_command

```json
{
  "command": "string"    // Shell command
}
```

### list_files

```json
{
  "path": "string",       // Directory path
  "recursive": boolean    // Recursive listing
}
```

### search_files

```json
{
  "path": "string",       // Search root
  "regex": "string",      // Search pattern
  "file_pattern": "string" // File glob
}
```

---

## Response Format

### Standard Response

```json
{
  "cancel": false,                 // true to block operation
  "contextModification": "",       // Text to inject
  "errorMessage": ""               // Shown when cancel=true
}
```

### Block Operation

```json
{
  "cancel": true,
  "contextModification": "",
  "errorMessage": "Explain why blocked"
}
```

### Allow with Warning

```json
{
  "cancel": false,
  "contextModification": "⚠️ WARNING: Explain concern"
}
```

### Allow with Context

```json
{
  "cancel": false,
  "contextModification": "WORKSPACE_RULES: Add guidance here"
}
```

---

## Essential Bash Patterns

### Read JSON Input

```bash
#!/usr/bin/env bash
input=$(cat)
```

### Extract Field with jq

```bash
# Simple field
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

# Nested field
file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')

# With default value
value=$(echo "$input" | jq -r '.field // "default"')

# Array element
workspace=$(echo "$input" | jq -r '.workspaceRoots[0]')
```

### Check Tool Type

```bash
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

if [[ "$tool_name" == "write_to_file" ]]; then
    # Handle file write
fi

if [[ "$tool_name" == "execute_command" ]]; then
    # Handle command execution
fi
```

### Pattern Matching

```bash
# Extension matching
if [[ "$file_path" == *.js ]]; then
    # JavaScript file
fi

# Exclude pattern
if [[ "$file_path" == *.js ]] && [[ "$file_path" != *.json ]]; then
    # .js but not .json
fi

# Multiple patterns
if [[ "$file_path" == *.ts ]] || [[ "$file_path" == *.tsx ]]; then
    # TypeScript file
fi

# Contains pattern
if [[ "$command" == *"rm -rf"* ]]; then
    # Contains rm -rf
fi

# Regex matching
if [[ "$command" =~ rm[[:space:]]+-[rfRF] ]]; then
    # Matches rm with -r or -f flags
fi
```

### File System Checks

```bash
# File exists
if [ -f "$file_path" ]; then
    # File exists
fi

# Directory exists
if [ -d "$directory" ]; then
    # Directory exists
fi

# Readable
if [ -r "$file_path" ]; then
    # File is readable
fi

# Executable
if [ -x "$file_path" ]; then
    # File is executable
fi
```

### String Manipulation

```bash
# Remove extension
filename="${file_path%.*}"

# Get extension
extension="${file_path##*.}"

# Remove .js and add .ts
new_path="${file_path%.js}.ts"

# Get directory
dir_name=$(dirname "$file_path")

# Get filename
file_name=$(basename "$file_path")
```

### Array Operations

```bash
# Define array
patterns=("pattern1" "pattern2" "pattern3")

# Loop through array
for pattern in "${patterns[@]}"; do
    echo "$pattern"
done

# Check if in array
if [[ " ${patterns[@]} " =~ " $value " ]]; then
    # Value is in array
fi
```

---

## Common Use Cases

### 1. Block File Type

```bash
#!/usr/bin/env bash
input=$(cat)
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

if [[ "$tool_name" == "write_to_file" ]]; then
    file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
    
    if [[ "$file_path" == *.js ]]; then
        echo '{"cancel": true, "errorMessage": "No .js files allowed"}' | jq -c '.'
        exit 0
    fi
fi

echo '{"cancel": false}'
```

### 2. Block Dangerous Command

```bash
#!/usr/bin/env bash
input=$(cat)
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

if [[ "$tool_name" == "execute_command" ]]; then
    command=$(echo "$input" | jq -r '.preToolUse.parameters.command')
    
    if [[ "$command" == *"rm -rf /"* ]]; then
        echo '{"cancel": true, "errorMessage": "Blocked dangerous command"}' | jq -c '.'
        exit 0
    fi
fi

echo '{"cancel": false}'
```

### 3. Detect Project Type

```bash
#!/usr/bin/env bash
input=$(cat)
workspace=$(echo "$input" | jq -r '.workspaceRoots[0]')

context=""

if [ -f "$workspace/package.json" ]; then
    context="WORKSPACE_RULES: Node.js project"
fi

echo "{\"cancel\": false, \"contextModification\": \"$context\"}"
```

### 4. Track Performance

```bash
#!/usr/bin/env bash
input=$(cat)
tool_name=$(echo "$input" | jq -r '.postToolUse.toolName')
exec_time=$(echo "$input" | jq -r '.postToolUse.executionTimeMs')

if [ "$exec_time" -gt 5000 ]; then
    context="PERFORMANCE: Slow operation detected (${exec_time}ms)"
    echo "{\"cancel\": false, \"contextModification\": \"$context\"}"
    exit 0
fi

echo '{"cancel": false}'
```

### 5. Log Operations

```bash
#!/usr/bin/env bash
input=$(cat)
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')
timestamp=$(date -Iseconds)

echo "[$timestamp] $tool_name" >> ~/.cline-operations.log

echo '{"cancel": false}'
```

### 6. Validate File Permissions

```bash
#!/usr/bin/env bash
input=$(cat)
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

if [[ "$tool_name" == "write_to_file" ]]; then
    file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
    
    # Check if trying to write to protected directory
    if [[ "$file_path" == /etc/* ]] || [[ "$file_path" == /usr/* ]]; then
        echo '{"cancel": true, "errorMessage": "Cannot write to system directory"}' | jq -c '.'
        exit 0
    fi
fi

echo '{"cancel": false}'
```

### 7. Inject Project Rules

```bash
#!/usr/bin/env bash
input=$(cat)
workspace=$(echo "$input" | jq -r '.workspaceRoots[0]')

context=""

if grep -q '"typescript"' "$workspace/package.json" 2>/dev/null; then
    context="WORKSPACE_RULES: Use TypeScript for all new files"
fi

echo "{\"cancel\": false, \"contextModification\": \"$context\"}"
```

---

## Essential jq Commands

```bash
# Extract value
jq -r '.field'

# Extract nested value
jq -r '.parent.child'

# Extract array element
jq -r '.array[0]'

# Extract with default
jq -r '.field // "default"'

# Check if field exists
jq 'has("field")'

# Get all keys
jq 'keys'

# Get array length
jq 'length'

# Pretty print
jq '.'

# Compact output (no whitespace)
jq -c '.'

# Filter array
jq '.array[] | select(.status == "active")'

# Map over array
jq '.array[] | .field'

# Count matches
jq '[.array[] | select(.status == "active")] | length'

# Create JSON
jq -n --arg var "$value" '{field: $var}'
```

---

## Python Hook Template

```python
#!/usr/bin/env python3
import json
import sys

# Read input
input_data = json.loads(sys.stdin.read())

# Extract data
tool_name = input_data.get('preToolUse', {}).get('toolName', '')

# Your logic here
cancel = False
context = ""
error_msg = ""

# Return response
response = {
    "cancel": cancel,
    "contextModification": context,
    "errorMessage": error_msg
}

print(json.dumps(response))
```

---

## Debugging Tips

### View Hook Execution

```bash
# Enable VS Code Output panel
# View > Output > Select "Cline"
```

### Test Hook Manually

```bash
# Create test input
echo '{"preToolUse":{"toolName":"write_to_file","parameters":{"path":"test.js"}}}' | .clinerules/hooks/PreToolUse
```

### Check Syntax

```bash
# Bash syntax check
bash -n .clinerules/hooks/PreToolUse

# Python syntax check
python3 -m py_compile .clinerules/hooks/PreToolUse
```

### Add Logging

```bash
# Log to stderr (visible in Output panel)
echo "Debug: tool=$tool_name" >&2

# Log to file
echo "[$timestamp] $message" >> ~/.cline-debug.log
```

### Validate JSON Output

```bash
# Your hook should output valid JSON
output=$(.clinerules/hooks/PreToolUse < test-input.json)
echo "$output" | jq empty  # Fails if invalid JSON
```

---

## Quick Setup

```bash
# Create hooks directory
mkdir -p .clinerules/hooks

# Create hook file
cat > .clinerules/hooks/PreToolUse << 'EOF'
#!/usr/bin/env bash
input=$(cat)
echo '{"cancel": false}'
EOF

# Make executable
chmod +x .clinerules/hooks/PreToolUse

# Test it
echo '{}' | .clinerules/hooks/PreToolUse
```

---

## Common Issues

### Hook Not Running
- [ ] "Enable Hooks" checked in Cline settings?
- [ ] Hook file executable? (`chmod +x`)
- [ ] Hook file in correct location?
- [ ] Correct file name (no extension)?
- [ ] Starts with shebang (`#!/usr/bin/env bash`)?

### Invalid JSON
- [ ] Output valid JSON? Test with `| jq .`
- [ ] Using `jq -c` for compact output?
- [ ] Escaping quotes properly?
- [ ] Last thing written to stdout is JSON?

### Hook Times Out
- [ ] Avoid long-running operations
- [ ] Use background processes for slow tasks
- [ ] Add timeout to external commands
- [ ] Keep hook logic simple

### Context Not Working
- [ ] Remember: affects FUTURE decisions
- [ ] Check next "API Request" block
- [ ] Context not too long? (50KB limit)
- [ ] Using clear, actionable language?

---

## File Locations

```
Project Root
├── .clinerules/
│   ├── hooks/
│   │   ├── TaskStart          # No extension!
│   │   ├── TaskResume
│   │   ├── TaskCancel
│   │   ├── PreToolUse
│   │   ├── PostToolUse
│   │   └── UserPromptSubmit
│   └── project-knowledge.json
│
└── ~/Documents/Cline/Rules/Hooks/  # Global hooks
    ├── TaskStart
    └── ...
```

---

## Best Practices

1. **Start Simple**: Begin with logging, then add logic
2. **Test Manually**: Test hooks outside of Cline first
3. **Handle Errors**: Use try-catch or `set -e`
4. **Keep It Fast**: Hooks should complete in < 1 second
5. **Log Everything**: Debug by logging to stderr or files
6. **Be Specific**: Narrow conditions prevent false positives
7. **Clear Messages**: Error messages should teach, not just block
8. **Version Control**: Commit hooks to share with team
9. **Document**: Comment your hook logic
10. **Iterate**: Start with warnings before blocking

---

## Security Considerations

- Hooks run with VS Code's permissions
- Can access all workspace files
- Can execute any command
- Can make network requests
- **Review hooks from untrusted sources**
- **Never run unverified hooks**
- **Test in safe environment first**

---

## Performance Tips

```bash
# Cache expensive operations
if [ ! -f ".cache/project-info" ]; then
    # Generate cache
    analyze_project > .cache/project-info
fi
result=$(cat .cache/project-info)

# Use background processes
long_operation &

# Set timeouts
timeout 5s command

# Avoid unnecessary operations
if [[ "$tool_name" != "write_to_file" ]]; then
    echo '{"cancel": false}'
    exit 0  # Skip file checks
fi
```

---

## Resources

- [Official Cline Docs](https://github.com/cline/cline/docs)
- [jq Manual](https://jqlang.github.io/jq/manual/)
- [Bash Guide](https://www.gnu.org/software/bash/manual/)
- [JSON Spec](https://www.json.org/)

---

## Version

**Document Version:** 1.0  
**Cline Version:** 3.1+  
**Last Updated:** 2025-11-07

---

## Quick Command Reference

```bash
# Enable hooks
# Cline > Settings > Feature > Enable Hooks

# Create hook
mkdir -p .clinerules/hooks
cat > .clinerules/hooks/PreToolUse << 'EOF'
#!/usr/bin/env bash
input=$(cat)
echo '{"cancel": false}'
EOF
chmod +x .clinerules/hooks/PreToolUse

# Test hook
echo '{"preToolUse":{"toolName":"test"}}' | .clinerules/hooks/PreToolUse

# View logs
# VS Code > View > Output > Select "Cline"

# Debug
echo "Debug message" >&2  # Visible in Output panel
echo "Log message" >> ~/.cline-debug.log  # Permanent log
```
