# Cline Hooks Debugging Guide

Complete troubleshooting reference for diagnosing and fixing hook issues.

---

## Table of Contents

1. [Quick Diagnostic Checklist](#quick-diagnostic-checklist)
2. [Common Problems & Solutions](#common-problems--solutions)
3. [Debugging Techniques](#debugging-techniques)
4. [Error Messages Explained](#error-messages-explained)
5. [Testing Strategies](#testing-strategies)
6. [Performance Issues](#performance-issues)
7. [Advanced Debugging](#advanced-debugging)

---

## Quick Diagnostic Checklist

When a hook isn't working, check these in order:

- [ ] **Hooks Enabled**: Settings > Feature > "Enable Hooks" checkbox ✓
- [ ] **File Location**: Hook in `.clinerules/hooks/` or `~/Documents/Cline/Rules/Hooks/`
- [ ] **File Name**: Exact match (e.g., `PreToolUse`, not `pre-tool-use` or `PreToolUse.sh`)
- [ ] **Executable**: `ls -l` shows `-rwxr-xr-x` (run `chmod +x` if needed)
- [ ] **Shebang**: First line is `#!/usr/bin/env bash` or appropriate interpreter
- [ ] **Syntax**: No bash/python syntax errors (`bash -n hookfile`)
- [ ] **JSON Output**: Hook outputs valid JSON (`| jq .` for validation)
- [ ] **Stdout Clean**: Only JSON on stdout, logging on stderr

---

## Common Problems & Solutions

### Problem 1: Hook Not Executing

**Symptoms:**
- No output in VS Code Output panel
- Hook behavior not happening
- File operations not being intercepted

**Diagnosis Steps:**

```bash
# 1. Check if hooks are enabled
# Open Cline Settings > Feature > Verify "Enable Hooks" is checked

# 2. Verify file location and name
ls -la .clinerules/hooks/
# Should show: PreToolUse, TaskStart, etc. (no extensions!)

# 3. Check executable permission
ls -l .clinerules/hooks/PreToolUse
# Should show: -rwxr-xr-x (x = executable)

# 4. Test manual execution
echo '{"preToolUse":{"toolName":"test"}}' | .clinerules/hooks/PreToolUse
# Should output JSON
```

**Solutions:**

```bash
# Enable hooks
# Cline > Settings > Feature > Check "Enable Hooks"

# Fix file location
mkdir -p .clinerules/hooks
mv PreToolUse.sh .clinerules/hooks/PreToolUse  # Remove extension!

# Make executable
chmod +x .clinerules/hooks/PreToolUse

# Restart VS Code (sometimes needed)
# Command Palette > "Reload Window"
```

---

### Problem 2: Invalid JSON Output

**Symptoms:**
- Error: "Hook returned invalid JSON"
- Hook executes but Cline ignores it
- Strange behavior or crashes

**Diagnosis:**

```bash
# Test hook output
output=$(.clinerules/hooks/PreToolUse < test-input.json)

# Validate JSON
echo "$output" | jq .
# If error appears, JSON is invalid

# Check what's actually being output
echo "$output" | cat -A  # Show hidden characters
```

**Common JSON Mistakes:**

```bash
# ❌ WRONG: Logging mixed with JSON
echo "Processing..."
echo '{"cancel": false}'

# ✅ CORRECT: Logging to stderr
echo "Processing..." >&2
echo '{"cancel": false}'

# ❌ WRONG: Unescaped quotes
echo "{"message": "File "test.js" blocked"}"

# ✅ CORRECT: Escaped quotes or using jq
echo '{"message": "File test.js blocked"}' | jq -c '.'

# ❌ WRONG: Multiple JSON objects
echo '{"cancel": false}'
echo '{"cancel": false}'

# ✅ CORRECT: Single JSON object
echo '{"cancel": false}'

# ❌ WRONG: Extra text after JSON
echo '{"cancel": false}'
echo "Done!"

# ✅ CORRECT: JSON is last output, logging to stderr
echo "Done!" >&2
echo '{"cancel": false}'
```

**Solutions:**

```bash
# Use jq to ensure valid JSON
echo '{"cancel": true, "errorMessage": "test"}' | jq -c '.'

# Redirect all logging to stderr
echo "Debug info" >&2

# Validate JSON in your hook
cat > PreToolUse << 'EOF'
#!/usr/bin/env bash
input=$(cat)

# Your logic here...

# Build JSON response
response='{"cancel": false}'

# Validate before outputting
if echo "$response" | jq empty 2>/dev/null; then
    echo "$response"
else
    echo '{"cancel": false}' >&2
    echo '{"cancel": false}'
fi
EOF
```

---

### Problem 3: Hook Times Out

**Symptoms:**
- Hook hangs for several seconds
- Eventually fails with timeout error
- Cline becomes unresponsive

**Diagnosis:**

```bash
# Time your hook
time echo '{"preToolUse":{"toolName":"test"}}' | .clinerules/hooks/PreToolUse

# Should be < 1 second
```

**Common Causes:**

1. **External network calls without timeout**
```bash
# ❌ WRONG: No timeout
curl https://api.example.com/check

# ✅ CORRECT: With timeout
curl --max-time 2 https://api.example.com/check 2>/dev/null || echo "timeout"
```

2. **Reading large files**
```bash
# ❌ WRONG: Reading entire file
content=$(cat large-file.txt)

# ✅ CORRECT: Read first N lines
content=$(head -100 large-file.txt)
```

3. **Complex operations**
```bash
# ❌ WRONG: Complex analysis in hook
analyze_entire_codebase

# ✅ CORRECT: Do it asynchronously
{
    analyze_entire_codebase > /tmp/analysis.txt
} &
```

4. **Infinite loops**
```bash
# ❌ WRONG: Might loop forever
while [ condition ]; do
    # something
done

# ✅ CORRECT: Add iteration limit
count=0
while [ condition ] && [ $count -lt 100 ]; do
    # something
    ((count++))
done
```

**Solutions:**

```bash
# Add timeouts to commands
timeout 5s your-command

# Use background processes for slow operations
{
    slow_operation
} &

# Cache expensive operations
if [ -f ".cache/result" ]; then
    result=$(cat .cache/result)
else
    result=$(expensive_operation)
    echo "$result" > .cache/result
fi

# Exit early when possible
if [[ "$tool_name" != "write_to_file" ]]; then
    echo '{"cancel": false}'
    exit 0  # Don't do file checks
fi
```

---

### Problem 4: Context Not Appearing

**Symptoms:**
- `contextModification` set but not visible
- Cline doesn't seem to "know" the context
- Expected behavior not happening

**Understanding Context Timing:**

```
Step 1: AI decides to write file.js
Step 2: PreToolUse hook runs
        - Can block (cancel: true)
        - Can inject context
Step 3: If not blocked, operation executes
Step 4: Context gets added to conversation
Step 5: NEXT AI request includes that context ← KEY!
```

**Diagnosis:**

```bash
# Check VS Code Output panel
# View > Output > Select "Cline"
# Look for your contextModification text

# Verify JSON is correct
echo '{"cancel": false, "contextModification": "TEST"}' | jq .

# Check context length
context="your context here"
echo "Context length: ${#context} characters"
# Should be < 50,000 characters
```

**Common Mistakes:**

```bash
# ❌ WRONG: Expecting immediate effect
# PreToolUse blocks file.js creation
# Sets context: "Use .ts files"
# Expects AI to immediately switch to .ts
# Result: Doesn't work - operation already decided

# ✅ CORRECT: Context affects next decision
# PreToolUse blocks file.js creation
# Sets context: "Use .ts files"
# AI sees error + context
# NEXT operation: AI tries .ts file
```

**Solutions:**

```bash
# For immediate effect, use blocking
if [[ "$file_path" == *.js ]]; then
    echo '{
      "cancel": true,
      "errorMessage": "Use .ts instead"
    }' | jq -c '.'
    exit 0
fi

# For learning, use context in PostToolUse
# This teaches for FUTURE operations
if [[ "$file_path" == *test* ]]; then
    context="PROJECT_PATTERN: Tests colocated with source"
    echo "{\"cancel\": false, \"contextModification\": \"$context\"}"
fi

# Make context clear and actionable
context="WORKSPACE_RULES:\n- Use TypeScript (.ts/.tsx)\n- Place components in src/components/\n- Use PascalCase for component files"

# Prefix context for categorization
"WORKSPACE_RULES: ..."
"PERFORMANCE: ..."
"SECURITY: ..."
"PROJECT_PATTERN: ..."
```

---

### Problem 5: Hook Blocks Everything

**Symptoms:**
- All operations being blocked
- Can't make any changes
- Constantly seeing error messages

**Diagnosis:**

```bash
# Test with different operations
echo '{"preToolUse":{"toolName":"read_file"}}' | .clinerules/hooks/PreToolUse
echo '{"preToolUse":{"toolName":"write_to_file","parameters":{"path":"test.txt"}}}' | .clinerules/hooks/PreToolUse
echo '{"preToolUse":{"toolName":"execute_command","parameters":{"command":"ls"}}}' | .clinerules/hooks/PreToolUse

# All should return {"cancel": false} for safe operations
```

**Common Causes:**

```bash
# ❌ WRONG: Missing specific tool check
file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
if [[ "$file_path" == *.js ]]; then
    echo '{"cancel": true}'
    exit 0
fi

# This blocks even when tool_name is "execute_command"
# because .parameters.path doesn't exist!

# ✅ CORRECT: Check tool type first
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

if [[ "$tool_name" == "write_to_file" ]]; then
    file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
    if [[ "$file_path" == *.js ]]; then
        echo '{"cancel": true}'
        exit 0
    fi
fi
```

**Solutions:**

```bash
# Always check tool type first
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

if [[ "$tool_name" == "write_to_file" ]]; then
    # File-specific checks
elif [[ "$tool_name" == "execute_command" ]]; then
    # Command-specific checks
fi

# Allow by default
echo '{"cancel": false}'

# Be specific with conditions
if [[ "$file_path" == *.js ]] && [[ "$file_path" != *.json ]]; then
    # Only block .js, not .json
fi

# Add logging to debug
echo "Tool: $tool_name, Path: $file_path" >&2
```

---

### Problem 6: Syntax Errors

**Symptoms:**
- Hook won't execute at all
- Bash/Python errors in Output panel
- "command not found" errors

**Diagnosis:**

```bash
# Check bash syntax
bash -n .clinerules/hooks/PreToolUse

# Check python syntax
python3 -m py_compile .clinerules/hooks/PreToolUse

# Run with error output
bash -x .clinerules/hooks/PreToolUse < test-input.json 2>&1
```

**Common Bash Errors:**

```bash
# ❌ WRONG: Missing quotes
if [[ $file_path == *.js ]]; then

# ✅ CORRECT: Variables quoted
if [[ "$file_path" == *.js ]]; then

# ❌ WRONG: Incorrect operator
if [ "$a" == "$b" ]; then  # == only works in [[

# ✅ CORRECT: Use correct operator
if [[ "$a" == "$b" ]]; then  # [[ supports ==
if [ "$a" = "$b" ]; then     # [ needs =

# ❌ WRONG: Missing semicolon
for file in $(ls); do echo $file; done

# ✅ CORRECT: Proper syntax
for file in $(ls); do 
    echo "$file"
done

# ❌ WRONG: Using grep exit code incorrectly
if grep "pattern" file; then
    # This fails if pattern not found
fi

# ✅ CORRECT: Capture grep result
if grep -q "pattern" file 2>/dev/null; then
    # -q is quiet, 2>/dev/null suppresses errors
fi
```

---

## Debugging Techniques

### Technique 1: Add Comprehensive Logging

```bash
#!/usr/bin/env bash

# Create debug log file
DEBUG_LOG="/tmp/cline-hook-debug.log"

log_debug() {
    echo "[$(date -Iseconds)] $*" >> "$DEBUG_LOG"
}

input=$(cat)

log_debug "=== Hook Started ==="
log_debug "Input: $input"

tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')
log_debug "Tool name: $tool_name"

if [[ "$tool_name" == "write_to_file" ]]; then
    file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
    log_debug "File path: $file_path"
    
    if [[ "$file_path" == *.js ]]; then
        log_debug "Blocking .js file"
        echo '{"cancel": true, "errorMessage": "No .js files"}' | jq -c '.'
        exit 0
    fi
fi

log_debug "Allowing operation"
echo '{"cancel": false}'
```

**View logs:**
```bash
tail -f /tmp/cline-hook-debug.log
```

### Technique 2: Step-by-Step Execution

```bash
#!/usr/bin/env bash

set -x  # Print each command before executing

input=$(cat)
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

if [[ "$tool_name" == "write_to_file" ]]; then
    file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
    echo "Checking file: $file_path" >&2
fi

set +x  # Stop printing commands

echo '{"cancel": false}'
```

### Technique 3: JSON Structure Inspection

```bash
#!/usr/bin/env bash

input=$(cat)

# Save complete input for inspection
echo "$input" | jq '.' > "/tmp/hook-input-$(date +%s).json"

# Print structure to stderr
echo "=== JSON Structure ===" >&2
echo "$input" | jq 'keys' >&2
echo "$input" | jq '.preToolUse | keys' >&2

# Show field types
echo "=== Field Types ===" >&2
echo "$input" | jq 'to_entries | .[] | "\(.key): \(.value | type)"' >&2

echo '{"cancel": false}'
```

### Technique 4: Conditional Debugging

```bash
#!/usr/bin/env bash

# Only debug specific cases
DEBUG_MODE=false

if [ -f ".clinerules/debug-hooks" ]; then
    DEBUG_MODE=true
fi

debug_log() {
    if [ "$DEBUG_MODE" = true ]; then
        echo "[DEBUG] $*" >&2
    fi
}

input=$(cat)

debug_log "Hook started"
debug_log "Input: $input"

# ... rest of hook logic ...

echo '{"cancel": false}'
```

**Enable debugging:**
```bash
touch .clinerules/debug-hooks  # Enable
rm .clinerules/debug-hooks     # Disable
```

---

## Error Messages Explained

### "Hook returned invalid JSON"

**Cause:** Hook's stdout contains non-JSON text or malformed JSON.

**Fix:**
```bash
# Validate your JSON output
echo '{"cancel": false}' | jq .

# Use jq to generate JSON
jq -n --arg msg "$message" '{"cancel": false, "errorMessage": $msg}'

# Ensure only JSON on stdout
echo "Debug info" >&2  # stderr for logging
echo '{"cancel": false}'  # stdout for JSON
```

### "Hook execution timeout"

**Cause:** Hook took longer than allowed time (usually 10 seconds).

**Fix:**
```bash
# Add timeouts to external commands
timeout 5s command

# Use background processes
{
    slow_task > /tmp/result.txt
} &

# Exit early
if [[ "$tool_name" != "write_to_file" ]]; then
    echo '{"cancel": false}'
    exit 0
fi
```

### "Hook not found"

**Cause:** Hook file doesn't exist or wrong location.

**Fix:**
```bash
# Verify location
ls -la .clinerules/hooks/

# Check file name (must match exactly)
# Correct: PreToolUse
# Wrong: pre-tool-use, PreToolUse.sh, pretooluse

# Create if missing
mkdir -p .clinerules/hooks
cat > .clinerules/hooks/PreToolUse << 'EOF'
#!/usr/bin/env bash
input=$(cat)
echo '{"cancel": false}'
EOF
chmod +x .clinerules/hooks/PreToolUse
```

### "Permission denied"

**Cause:** Hook file not executable.

**Fix:**
```bash
chmod +x .clinerules/hooks/PreToolUse
chmod +x .clinerules/hooks/TaskStart
chmod +x .clinerules/hooks/PostToolUse

# Verify
ls -l .clinerules/hooks/
# Should show: -rwxr-xr-x
```

---

## Testing Strategies

### Strategy 1: Unit Testing

Create test inputs and verify outputs:

```bash
# test-pretooluse.sh
#!/usr/bin/env bash

test_hook() {
    local description="$1"
    local input="$2"
    local expected_cancel="$3"
    
    result=$(echo "$input" | .clinerules/hooks/PreToolUse)
    actual_cancel=$(echo "$result" | jq -r '.cancel')
    
    if [[ "$actual_cancel" == "$expected_cancel" ]]; then
        echo "✅ PASS: $description"
    else
        echo "❌ FAIL: $description (expected: $expected_cancel, got: $actual_cancel)"
    fi
}

# Test cases
test_hook "Allow .ts files" \
    '{"preToolUse":{"toolName":"write_to_file","parameters":{"path":"test.ts"}}}' \
    "false"

test_hook "Block .js files" \
    '{"preToolUse":{"toolName":"write_to_file","parameters":{"path":"test.js"}}}' \
    "true"

test_hook "Allow .json files" \
    '{"preToolUse":{"toolName":"write_to_file","parameters":{"path":"data.json"}}}' \
    "false"
```

### Strategy 2: Integration Testing

Test with real Cline:

```bash
# 1. Create test project
mkdir -p /tmp/test-hooks
cd /tmp/test-hooks

# 2. Add your hooks
mkdir -p .clinerules/hooks
cp ~/my-hooks/PreToolUse .clinerules/hooks/
chmod +x .clinerules/hooks/PreToolUse

# 3. Test scenarios in Cline
# - Try creating blocked file type
# - Try allowed operations
# - Check Output panel for logs
```

### Strategy 3: Regression Testing

Keep a test suite:

```bash
# tests/hook-tests.sh
#!/usr/bin/env bash

echo "Running hook regression tests..."

# Test 1: Basic functionality
result=$(echo '{"preToolUse":{"toolName":"write_to_file","parameters":{"path":"test.js"}}}' | .clinerules/hooks/PreToolUse)
if echo "$result" | jq -e '.cancel == true' > /dev/null; then
    echo "✅ Test 1: Block .js files"
else
    echo "❌ Test 1: Failed"
    exit 1
fi

# Test 2: JSON validity
if echo "$result" | jq empty 2>/dev/null; then
    echo "✅ Test 2: Valid JSON"
else
    echo "❌ Test 2: Invalid JSON"
    exit 1
fi

# Add more tests...

echo "All tests passed!"
```

---

## Performance Issues

### Issue: Slow Hook Execution

**Profile your hook:**

```bash
# Add timing to your hook
start_time=$(date +%s%N)

# ... your hook logic ...

end_time=$(date +%s%N)
elapsed=$(( (end_time - start_time) / 1000000 ))
echo "Hook took ${elapsed}ms" >&2
```

**Optimize:**

```bash
# ❌ SLOW: Running jq multiple times
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')
file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
content=$(echo "$input" | jq -r '.preToolUse.parameters.content')

# ✅ FASTER: Parse once
eval "$(echo "$input" | jq -r '@sh "tool_name=\(.preToolUse.toolName) file_path=\(.preToolUse.parameters.path)"')"

# ❌ SLOW: Multiple file checks
[ -f file1 ]
[ -f file2 ]
[ -f file3 ]

# ✅ FASTER: Batch check
files=("file1" "file2" "file3")
for f in "${files[@]}"; do
    [ -f "$f" ] && echo "$f exists"
done

# ❌ SLOW: External command for every check
if grep -q "pattern" file; then ...

# ✅ FASTER: Read file once
content=$(cat file)
if echo "$content" | grep -q "pattern"; then ...
```

---

## Advanced Debugging

### Remote Debugging with VSCode

1. Add debugging breakpoints to your hook
2. Use Python's `debugpy` or bash debugging tools
3. Attach VSCode debugger

### Logging to System Log

```bash
# Log to syslog
logger -t cline-hook "Hook executed: $tool_name"

# View logs
tail -f /var/log/syslog | grep cline-hook
```

### Creating Debug Builds

```bash
# Create debug version of hook
cat > .clinerules/hooks/PreToolUse.debug << 'EOF'
#!/usr/bin/env bash
set -x  # Full debugging
exec 2>>/tmp/hook-debug.log  # Log all stderr

source .clinerules/hooks/PreToolUse
EOF

chmod +x .clinerules/hooks/PreToolUse.debug

# Swap for debugging
mv .clinerules/hooks/PreToolUse .clinerules/hooks/PreToolUse.prod
mv .clinerules/hooks/PreToolUse.debug .clinerules/hooks/PreToolUse
```

---

## Quick Reference

### Debugging Checklist

1. Check hook file exists and is executable
2. Validate JSON output with `jq`
3. Add logging to stderr
4. Test manually with sample input
5. Check VS Code Output panel
6. Verify hook timing (should be < 1s)
7. Review error messages in context
8. Test with different input scenarios

### Common Commands

```bash
# Test hook
echo '{"preToolUse":{"toolName":"test"}}' | ./hookfile

# Validate JSON
echo "$output" | jq .

# Check syntax
bash -n hookfile

# Make executable
chmod +x hookfile

# View logs
tail -f /tmp/cline-debug.log

# Check execution time
time ./hookfile < input.json
```

---

**Remember:** Good debugging is methodical. Start with the checklist, add logging, test in isolation, then integrate with Cline.
