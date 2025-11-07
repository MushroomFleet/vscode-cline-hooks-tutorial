# Cline Hooks Quick Reference Cheatsheet

**Fast lookup for common patterns, commands, and hook types**

## 📋 Hook Types Summary

| Hook | Timing | Can Block? | Primary Use | Context Affects |
|------|--------|------------|-------------|-----------------|
| **TaskStart** | Task begins | No | Initialize, detect project | Entire task |
| **TaskResume** | Task resumes | No | Restore state | Resumed task |
| **TaskCancel** | Task cancelled | No | Cleanup, logging | N/A |
| **PreToolUse** | Before tool runs | ✅ Yes | Validation, blocking | Next tool use |
| **PostToolUse** | After tool runs | No | Learning, metrics | Future decisions |
| **UserPromptSubmit** | User sends message | Partial | Input validation | Next AI response |

## 🔧 Quick Setup

```bash
# Project-specific hooks
mkdir -p .clinerules/hooks
cd .clinerules/hooks

# Create hook (no extension!)
touch TaskStart
chmod +x TaskStart

# Edit
code TaskStart
```

## 📝 Hook File Template

### Bash Template

```bash
#!/usr/bin/env bash

# Read input
input=$(cat)

# Extract fields
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')
task_id=$(echo "$input" | jq -r '.taskId')

# Your logic here

# Return response
echo '{
  "cancel": false,
  "contextModification": ""
}'
```

### Python Template

```python
#!/usr/bin/env python3

import json
import sys

# Read input
input_data = json.loads(sys.stdin.read())

# Extract fields
tool_name = input_data.get('preToolUse', {}).get('toolName')
task_id = input_data.get('taskId')

# Your logic here

# Return response
response = {
    "cancel": False,
    "contextModification": ""
}

print(json.dumps(response))
```

## 🎯 Common JSON Paths

### All Hooks (Base Fields)

```bash
.clineVersion          # "1.0.0"
.hookName              # "PreToolUse"
.timestamp             # "2025-11-06T10:30:00Z"
.taskId                # "unique-task-id"
.workspaceRoots[0]     # "/path/to/workspace"
.userId                # "user-identifier"
```

### PreToolUse

```bash
.preToolUse.toolName                  # "write_to_file"
.preToolUse.parameters.path           # File path
.preToolUse.parameters.content        # File content
.preToolUse.parameters.command        # Command string
```

### PostToolUse

```bash
.postToolUse.toolName                 # "execute_command"
.postToolUse.parameters               # Same as PreToolUse
.postToolUse.result                   # Tool output
.postToolUse.success                  # true/false
.postToolUse.executionTimeMs          # 1234
```

### TaskStart

```bash
.taskStart.taskMetadata.taskId        # Task ID
.taskStart.taskMetadata.ulid          # ULID
.taskStart.taskMetadata.initialTask   # User's request
```

## 🔍 jq Essentials

### Basic Extraction

```bash
# Get value
echo "$input" | jq -r '.field'

# Get with default
echo "$input" | jq -r '.field // "default"'

# Get nested
echo "$input" | jq -r '.level1.level2.field'

# Get array element
echo "$input" | jq -r '.array[0]'
```

### Checking Values

```bash
# Check if field exists
echo "$input" | jq 'has("field")'

# Check type
echo "$input" | jq -r '.field | type'

# Check if null
echo "$input" | jq -r '.field // "is null"'
```

### Validation

```bash
# Validate JSON
echo "$input" | jq empty 2>/dev/null && echo "valid" || echo "invalid"

# Pretty print
echo "$input" | jq '.'

# Compact
echo "$input" | jq -c '.'
```

## 🛠️ Common Patterns

### Safe Field Extraction

```bash
# With default
field=$(echo "$input" | jq -r '.path.to.field // "default"')

# Check before use
if [ -n "$field" ] && [ "$field" != "null" ]; then
    # Use field
fi
```

### Tool Type Checking

```bash
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

case "$tool_name" in
    "write_to_file")
        # Handle file writes
        ;;
    "execute_command")
        # Handle commands
        ;;
    "read_file")
        # Handle file reads
        ;;
    *)
        # Default case
        ;;
esac
```

### File Extension Matching

```bash
file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')

# Check extension
if [[ "$file_path" == *.ts ]]; then
    echo "TypeScript file"
fi

# Multiple extensions
if [[ "$file_path" =~ \.(ts|tsx|js|jsx)$ ]]; then
    echo "JavaScript/TypeScript file"
fi
```

### Pattern Matching

```bash
command=$(echo "$input" | jq -r '.preToolUse.parameters.command')

# Contains pattern
if [[ "$command" == *"npm install"* ]]; then
    echo "Installing packages"
fi

# Starts with
if [[ "$command" == sudo* ]]; then
    echo "Sudo command"
fi

# Regex
if [[ "$command" =~ ^(npm|yarn|pnpm)[[:space:]]install ]]; then
    echo "Package installation"
fi
```

## 📤 Response Formats

### Allow Operation

```json
{
  "cancel": false,
  "contextModification": ""
}
```

### Block Operation

```json
{
  "cancel": true,
  "errorMessage": "Reason for blocking",
  "contextModification": ""
}
```

### Allow with Context

```json
{
  "cancel": false,
  "contextModification": "WORKSPACE_RULES: Additional context here"
}
```

### Context Prefixes

```bash
"WORKSPACE_RULES: ..."        # Project conventions
"PERFORMANCE: ..."             # Performance guidance
"SECURITY: ..."                # Security concerns
"CODE_QUALITY: ..."            # Quality feedback
"DOCUMENTATION: ..."           # Doc reminders
"ERROR_PATTERN: ..."           # Common mistakes
"BEST_PRACTICE: ..."           # Recommendations
```

## 🚫 Blocking Examples

### Block by File Extension

```bash
if [[ "$file_path" == *.js ]]; then
    echo '{
      "cancel": true,
      "errorMessage": "JavaScript files not allowed. Use .ts instead."
    }'
    exit 0
fi
```

### Block by Command Pattern

```bash
if [[ "$command" =~ rm[[:space:]]+-rf ]]; then
    echo '{
      "cancel": true,
      "errorMessage": "rm -rf is blocked for safety"
    }'
    exit 0
fi
```

### Conditional Blocking

```bash
if [ "$file_writes" -gt 50 ]; then
    echo '{
      "cancel": true,
      "errorMessage": "Too many file operations. Break into smaller tasks."
    }'
    exit 0
fi
```

## 📊 Logging Patterns

### Error Logging

```bash
ERROR_LOG="$HOME/.cline-hooks-errors.log"

log_error() {
    echo "[ERROR $(date -Iseconds)] $*" >> "$ERROR_LOG"
}

log_error "Something went wrong"
```

### Debug Output (stderr)

```bash
# Visible in VSCode Output panel
echo "Debug message" >&2

# Conditional debug
if [ "$DEBUG" = "true" ]; then
    echo "Debug: $variable" >&2
fi
```

### JSONL Logging

```bash
LOG_FILE="/tmp/cline-events.jsonl"

# Append JSON line
echo "$input" | jq -c '.' >> "$LOG_FILE"

# Read back
cat "$LOG_FILE" | jq '.'
```

## 🔄 State Management

### Save State

```bash
STATE_FILE="/tmp/cline-state-${task_id}.json"

state=$(jq -n \
  --arg task_id "$task_id" \
  --argjson count "$count" \
  '{taskId: $task_id, count: $count}')

echo "$state" > "$STATE_FILE"
```

### Load State

```bash
if [ -f "$STATE_FILE" ]; then
    state=$(cat "$STATE_FILE")
    count=$(echo "$state" | jq -r '.count')
fi
```

### Update State

```bash
# Read
state=$(cat "$STATE_FILE")

# Modify
state=$(echo "$state" | jq --argjson new_count "$new_count" '.count = $new_count')

# Write
echo "$state" > "$STATE_FILE"
```

## ⚡ Performance Tips

### Timeouts

```bash
# Wrap with timeout
timeout 5s your-command

# In hook
if ! output=$(timeout 3s external_tool 2>&1); then
    # Handle timeout
fi
```

### Caching

```bash
CACHE_FILE="/tmp/cline-cache-$key.json"
CACHE_TTL=300  # 5 minutes

# Check cache
if [ -f "$CACHE_FILE" ]; then
    age=$(($(date +%s) - $(stat -f %m "$CACHE_FILE" 2>/dev/null || stat -c %Y "$CACHE_FILE")))
    
    if [ "$age" -lt "$CACHE_TTL" ]; then
        # Use cache
        result=$(cat "$CACHE_FILE")
    fi
fi

# Update cache
echo "$result" > "$CACHE_FILE"
```

### Early Exit

```bash
# Exit early when possible
if [[ "$tool_name" != "write_to_file" ]]; then
    echo '{"cancel": false, "contextModification": ""}'; exit 0
fi

# Don't do unnecessary work
```

## 🧪 Testing

### Manual Test

```bash
# Create test input
cat << 'EOF' | ./PreToolUse
{
  "clineVersion": "1.0.0",
  "hookName": "PreToolUse",
  "timestamp": "2025-11-06T10:30:00Z",
  "taskId": "test-123",
  "workspaceRoots": ["/workspace"],
  "userId": "user-001",
  "preToolUse": {
    "toolName": "write_to_file",
    "parameters": {
      "path": "/workspace/test.ts",
      "content": "test"
    }
  }
}
EOF
```

### Check Exit Code

```bash
./PreToolUse < test-input.json
echo "Exit code: $?"  # Should be 0
```

### Validate JSON Output

```bash
output=$(./PreToolUse < test-input.json)
echo "$output" | jq empty && echo "Valid JSON" || echo "Invalid JSON"
```

## 🐛 Debugging

### Enable Debug Mode

```bash
export CLINE_HOOK_DEBUG=true
```

### Check Logs

```bash
# Error log
tail -f ~/.cline-hooks-errors.log

# VSCode Output panel
# View → Output → Select "Cline"
```

### Syntax Check

```bash
# Bash
bash -n PreToolUse

# Python
python3 -m py_compile PreToolUse
```

### Permissions

```bash
# Check
ls -l PreToolUse

# Should show: -rwxr-xr-x

# Fix
chmod +x PreToolUse
```

## 🔐 Security

### Environment Variables

```bash
# Read securely
SECRET=$(echo "$input" | jq -r '.secret // env.SECRET_KEY // "default"')

# Use env vars
export TEAM_SLACK_WEBHOOK="https://hooks.slack.com/..."
webhook="${TEAM_SLACK_WEBHOOK}"
```

### Input Sanitization

```bash
# Escape for shell
safe_input=$(printf '%q' "$input")

# Validate before use
if [[ ! "$file_path" =~ ^[a-zA-Z0-9/_.-]+$ ]]; then
    echo "Invalid characters in path" >&2
    exit 0
fi
```

### Temp Files

```bash
# Create securely
temp_file=$(mktemp)

# Clean up
trap "rm -f $temp_file" EXIT

# Your logic
echo "data" > "$temp_file"

# File is auto-deleted on exit
```

## 📚 Common Tool Parameters

### write_to_file

```json
{
  "path": "/path/to/file.ts",
  "content": "file contents here..."
}
```

### read_file

```json
{
  "path": "/path/to/file.ts"
}
```

### execute_command

```json
{
  "command": "npm install lodash"
}
```

### list_files

```json
{
  "path": "/path/to/directory",
  "recursive": true
}
```

## 🔗 Useful Commands

### File Operations

```bash
# Check if file exists
[ -f "$file_path" ] && echo "exists"

# Check if directory exists
[ -d "$dir_path" ] && echo "exists"

# Get file size
size=$(wc -c < "$file_path")

# Count lines
lines=$(wc -l < "$file_path")

# Get extension
ext="${file_path##*.}"

# Get basename
name=$(basename "$file_path")

# Get directory
dir=$(dirname "$file_path")
```

### String Operations

```bash
# Length
length=${#string}

# Substring
sub="${string:0:10}"  # First 10 chars

# Replace
new="${string/old/new}"  # First occurrence
new="${string//old/new}"  # All occurrences

# Remove prefix/suffix
no_prefix="${string#prefix}"
no_suffix="${string%suffix}"

# Uppercase/lowercase
upper="${string^^}"
lower="${string,,}"
```

### JSON with jq

```bash
# Create JSON
jq -n --arg key "value" '{$key}'

# Merge
jq -s '.[0] * .[1]' file1.json file2.json

# Filter array
jq '[.[] | select(.status == "active")]'

# Transform
jq 'map({name: .name, id: .id})'

# Count
jq 'length'
```

## 💡 Best Practices

1. **Always return valid JSON**
2. **Exit with code 0**
3. **Validate inputs**
4. **Use timeouts**
5. **Log errors**
6. **Clean up temp files**
7. **Keep hooks fast (<5s)**
8. **Test thoroughly**
9. **Document behavior**
10. **Version your hooks**

## 📍 File Locations

```bash
# Project hooks
.clinerules/hooks/

# User hooks (all projects)
~/Documents/Cline/Rules/Hooks/

# User config
~/.cline-personal-config.json

# Logs
~/.cline-hooks-errors.log

# Temp state
/tmp/cline-state-*.json
```

## 🆘 Quick Troubleshooting

| Problem | Solution |
|---------|----------|
| Hook not running | Check "Enable Hooks" in settings |
| Permission denied | Run `chmod +x hookname` |
| Invalid JSON | Use `jq empty` to validate |
| Timeout | Reduce hook complexity |
| Context not working | Remember it affects NEXT decision |
| Can't find jq | Install: `brew install jq` (Mac) or `apt-get install jq` (Linux) |

## 🔄 Return to Course

- [Course Overview](README.md)
- [Module 1: Foundation](module-01-foundation.md)
- [Debugging Guide](debugging-guide.md)
- [Patterns Library](patterns-library.md)

---

**Quick Reference v1.0** | Last Updated: 2025-11-06
