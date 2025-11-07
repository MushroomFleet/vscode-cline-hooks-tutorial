# Cline Hooks - Frequently Asked Questions (FAQ)

Answers to the most common questions about Cline Hooks.

---

## General Questions

### What are Cline Hooks?

Hooks are custom scripts that run automatically at specific points in Cline's workflow. They let you:
- Validate operations before they execute (e.g., block `.js` files in TypeScript projects)
- Monitor tool usage and performance
- Inject context that shapes AI decisions
- Track operations for analytics or compliance
- Trigger external tools or services

Think of them as middleware for your AI assistant.

### Why would I want to use hooks?

Hooks solve real problems:

**Without Hooks:**
- Cline might create `.js` files in your TypeScript project
- No way to block dangerous commands
- Can't enforce team coding standards
- No performance tracking
- No integration with external tools

**With Hooks:**
- Automatically enforce project conventions
- Block operations before they cause issues
- Build up project knowledge over time
- Track everything for analytics
- Connect Cline to your existing tools

### Do hooks slow down Cline?

Properly written hooks add minimal overhead (< 100ms typically). However:

**Fast Hooks** (< 50ms):
- Simple file extension checks
- Pattern matching
- Logging operations
- Basic validation

**Slower Hooks** (50-500ms):
- Running linters
- File system searches
- External API calls

**Avoid:**
- Long network requests (use timeouts!)
- Reading entire large files
- Complex computations
- Unbounded loops

**Best Practice:** Keep hooks under 1 second total execution time.

### Are hooks secure?

**Security Considerations:**

✅ **Safe:**
- Hooks run with VS Code's permissions
- Can't escalate privileges
- Sandboxed to workspace

⚠️ **Important:**
- Hooks can access all workspace files
- Can execute shell commands
- Can make network requests
- **Never run hooks from untrusted sources!**

**Best Practices:**
- Review all hooks before enabling
- Don't download hooks from unknown sources
- Test hooks in isolated environments first
- Use version control for team hooks
- Audit hook behavior regularly

---

## Setup & Configuration

### Where should I put my hook files?

Two options:

**Project-Specific Hooks** (Recommended for most):
```
/path/to/your/project/
└── .clinerules/
    └── hooks/
        ├── TaskStart
        ├── PreToolUse
        └── PostToolUse
```
- Only apply to this project
- Commit to version control
- Shared with team

**Global/Personal Hooks:**
```
~/Documents/Cline/Rules/Hooks/
├── TaskStart
├── PreToolUse
└── PostToolUse
```
- Apply to all projects
- Personal preferences
- Organization-wide standards

### Do hook files need file extensions?

**No!** Hook files must have exact names with no extensions:

✅ **Correct:**
- `TaskStart`
- `PreToolUse`
- `PostToolUse`

❌ **Wrong:**
- `TaskStart.sh`
- `pre-tool-use`
- `PreToolUse.bash`
- `task_start`

### How do I enable hooks?

1. Open Cline in VS Code
2. Click "Settings" (top right)
3. Click "Feature" in left navigation
4. Check "Enable Hooks" checkbox

**Verify it worked:**
- Create a simple hook
- Start a Cline task
- Check VS Code Output panel (View > Output > Select "Cline")

### Can I disable hooks temporarily?

Yes, several ways:

**Method 1: Uncheck in settings**
- Settings > Feature > Uncheck "Enable Hooks"

**Method 2: Rename hooks directory**
```bash
mv .clinerules/hooks .clinerules/hooks.disabled
```

**Method 3: Make hooks non-executable**
```bash
chmod -x .clinerules/hooks/*
```

**Method 4: Early exit in hook**
```bash
#!/usr/bin/env bash
# Add this at the top of your hook
if [ -f ".clinerules/disable-hooks" ]; then
    echo '{"cancel": false}'
    exit 0
fi
```

### Can I use hooks on Windows?

**Currently: No**

Hooks are only supported on:
- ✅ macOS
- ✅ Linux

Windows support is not available yet.

**Workaround for Windows users:**
- Use WSL (Windows Subsystem for Linux)
- Develop in Linux VM
- Wait for official Windows support

---

## Writing Hooks

### What language should I use for hooks?

**Bash** (Most Common):
```bash
#!/usr/bin/env bash
input=$(cat)
echo '{"cancel": false}'
```

✅ Pros: Simple, fast, good for file/command checks
❌ Cons: Limited data structures, string manipulation quirks

**Python** (More Complex Logic):
```python
#!/usr/bin/env python3
import json, sys
input_data = json.loads(sys.stdin.read())
print(json.dumps({"cancel": False}))
```

✅ Pros: Rich libraries, better data handling, easier testing
❌ Cons: Slightly slower, requires Python installed

**Other Languages:**
- Node.js: `#!/usr/bin/env node`
- Ruby: `#!/usr/bin/env ruby`
- Any language that can read stdin and write JSON to stdout

### How do I read the input data?

**Bash:**
```bash
#!/usr/bin/env bash
input=$(cat)
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')
```

**Python:**
```python
#!/usr/bin/env python3
import json
import sys

input_data = json.loads(sys.stdin.read())
tool_name = input_data['preToolUse']['toolName']
```

**Node.js:**
```javascript
#!/usr/bin/env node
const input = JSON.parse(require('fs').readFileSync(0, 'utf-8'));
const toolName = input.preToolUse.toolName;
```

### What must my hook output?

Every hook **must** output valid JSON with at minimum:

```json
{
  "cancel": false
}
```

**Full response options:**
```json
{
  "cancel": false,              // true = block operation
  "contextModification": "",    // text to inject
  "errorMessage": ""            // shown when cancel=true
}
```

**Important:** Only output JSON to stdout! Use stderr for logging:
```bash
echo "Debug info" >&2          # stderr - logging
echo '{"cancel": false}'       # stdout - JSON response
```

### Can I block all operations with one hook?

Yes, but it's not recommended. Instead:

**❌ Don't do this:**
```bash
#!/usr/bin/env bash
echo '{"cancel": true, "errorMessage": "Everything blocked!"}'
```

**✅ Do this:**
```bash
#!/usr/bin/env bash
input=$(cat)
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

# Only block specific dangerous operations
if [[ "$tool_name" == "execute_command" ]]; then
    command=$(echo "$input" | jq -r '.preToolUse.parameters.command')
    if [[ "$command" == *"rm -rf /"* ]]; then
        echo '{"cancel": true, "errorMessage": "Dangerous command blocked"}'
        exit 0
    fi
fi

# Allow everything else
echo '{"cancel": false}'
```

### How do I test my hooks before using them with Cline?

**Step 1: Test syntax**
```bash
# Bash
bash -n .clinerules/hooks/PreToolUse

# Python
python3 -m py_compile .clinerules/hooks/PreToolUse
```

**Step 2: Test with sample input**
```bash
echo '{"preToolUse":{"toolName":"write_to_file","parameters":{"path":"test.js"}}}' | .clinerules/hooks/PreToolUse
```

**Step 3: Validate JSON output**
```bash
output=$(.clinerules/hooks/PreToolUse < test-input.json)
echo "$output" | jq .  # Should not error
```

**Step 4: Create test suite**
```bash
#!/usr/bin/env bash
# test-hooks.sh

test_case() {
    input="$1"
    expected_cancel="$2"
    
    result=$(echo "$input" | .clinerules/hooks/PreToolUse)
    actual=$(echo "$result" | jq -r '.cancel')
    
    if [[ "$actual" == "$expected_cancel" ]]; then
        echo "✅ PASS"
    else
        echo "❌ FAIL"
    fi
}

test_case '{"preToolUse":{"toolName":"write_to_file","parameters":{"path":"test.js"}}}' "true"
test_case '{"preToolUse":{"toolName":"write_to_file","parameters":{"path":"test.ts"}}}' "false"
```

---

## Hook Behavior

### When exactly do hooks run?

**TaskStart**: When user starts a new task
**TaskResume**: When task resumes after interruption  
**PreToolUse**: Before any tool executes (write_to_file, execute_command, etc.)
**PostToolUse**: After tool completes
**TaskCancel**: When user cancels task
**UserPromptSubmit**: When user sends a message

### Can PreToolUse hooks block operations?

Yes! Set `"cancel": true`:

```bash
#!/usr/bin/env bash
input=$(cat)
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

if [[ "$tool_name" == "write_to_file" ]]; then
    file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
    
    if [[ "$file_path" == *.js ]]; then
        echo '{
          "cancel": true,
          "errorMessage": "JavaScript files not allowed"
        }'
        exit 0
    fi
fi

echo '{"cancel": false}'
```

### Can other hooks block operations?

**TaskStart, TaskResume, TaskCancel**: No, these are informational only
**PostToolUse**: No, operation already completed
**UserPromptSubmit**: Partially (can inject context, but can't fully block)

Only **PreToolUse** can truly block operations before they happen.

### What's the difference between blocking and context injection?

**Blocking (PreToolUse):**
- Stops operation immediately
- Shows error message to user
- Operation never executes

```bash
echo '{"cancel": true, "errorMessage": "Blocked!"}'
```

**Context Injection (Any Hook):**
- Operation proceeds
- Adds information to conversation
- Affects FUTURE decisions

```bash
echo '{"cancel": false, "contextModification": "WORKSPACE_RULES: Use TypeScript"}'
```

### Why isn't my contextModification affecting behavior?

**Context timing is key:**

```
1. AI decides: "I'll create file.js"
2. PreToolUse runs: injects "Use TypeScript"
3. Operation executes (file.js created)
4. Context added to conversation
5. NEXT AI decision sees "Use TypeScript" ← Here!
```

Context affects **future** decisions, not current ones.

**If you need immediate effect:**
- Use `cancel: true` to block the operation
- The error message teaches the AI what to do instead

### How long can contextModification be?

**Limit: 50KB (approximately 50,000 characters)**

**Recommendations:**
- Keep it concise (< 500 characters typically)
- Focus on actionable guidance
- Use clear, structured format

```bash
# ❌ Too verbose
context="This is a TypeScript project which means that you should be using TypeScript for all of your files and that includes using the .ts extension for regular TypeScript files and the .tsx extension for TypeScript files that contain JSX or React components..."

# ✅ Concise
context="WORKSPACE_RULES: Use TypeScript (.ts/.tsx) for all new files"
```

---

## Common Patterns

### How do I block specific file types?

```bash
#!/usr/bin/env bash
input=$(cat)
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

if [[ "$tool_name" == "write_to_file" ]]; then
    file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
    
    # Block .js files (but not .json)
    if [[ "$file_path" == *.js ]] && [[ "$file_path" != *.json ]]; then
        echo '{
          "cancel": true,
          "errorMessage": "Use .ts instead of .js"
        }' | jq -c '.'
        exit 0
    fi
fi

echo '{"cancel": false}'
```

### How do I block dangerous commands?

```bash
#!/usr/bin/env bash
input=$(cat)
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

if [[ "$tool_name" == "execute_command" ]]; then
    command=$(echo "$input" | jq -r '.preToolUse.parameters.command')
    
    # Check for dangerous patterns
    dangerous=("rm -rf /" "mkfs" "dd if=/dev/zero")
    
    for pattern in "${dangerous[@]}"; do
        if [[ "$command" == *"$pattern"* ]]; then
            echo '{
              "cancel": true,
              "errorMessage": "Dangerous command blocked: '"$pattern"'"
            }' | jq -c '.'
            exit 0
        fi
    done
fi

echo '{"cancel": false}'
```

### How do I auto-detect project type?

```bash
#!/usr/bin/env bash
input=$(cat)
workspace=$(echo "$input" | jq -r '.workspaceRoots[0]')

context=""

# Node.js project
if [ -f "$workspace/package.json" ]; then
    context="WORKSPACE_RULES: Node.js project."
    
    if grep -q '"typescript"' "$workspace/package.json"; then
        context="$context Use TypeScript."
    fi
fi

# Python project
if [ -f "$workspace/requirements.txt" ]; then
    context="WORKSPACE_RULES: Python project. Follow PEP 8."
fi

echo "{\"cancel\": false, \"contextModification\": \"$context\"}"
```

### How do I track performance?

```bash
#!/usr/bin/env bash
input=$(cat)
exec_time=$(echo "$input" | jq -r '.postToolUse.executionTimeMs')

if [ "$exec_time" -gt 5000 ]; then
    context="PERFORMANCE: Operation took ${exec_time}ms. Consider optimizing."
    echo "{\"cancel\": false, \"contextModification\": \"$context\"}"
    exit 0
fi

echo '{"cancel": false}'
```

---

## Debugging

### My hook isn't running. What should I check?

1. **Is "Enable Hooks" checked?**
   - Settings > Feature > "Enable Hooks"

2. **Is the file executable?**
   ```bash
   ls -l .clinerules/hooks/PreToolUse
   # Should show: -rwxr-xr-x
   chmod +x .clinerules/hooks/PreToolUse
   ```

3. **Is the file name correct?**
   - Must be exact: `PreToolUse` not `PreToolUse.sh`

4. **Does it have a shebang?**
   ```bash
   head -1 .clinerules/hooks/PreToolUse
   # Should show: #!/usr/bin/env bash
   ```

5. **Are there syntax errors?**
   ```bash
   bash -n .clinerules/hooks/PreToolUse
   ```

### How do I see hook output and errors?

**VS Code Output Panel:**
1. View > Output (or Cmd/Ctrl+Shift+U)
2. Select "Cline" from dropdown
3. All stderr output appears here

**Log to file:**
```bash
echo "Debug info" >> /tmp/cline-debug.log
tail -f /tmp/cline-debug.log
```

### My hook is timing out. What should I do?

Hooks must complete quickly (< 10 seconds, ideally < 1 second).

**Solutions:**

1. **Add timeouts to external commands:**
```bash
timeout 5s curl https://api.example.com
```

2. **Use background processes:**
```bash
{
    slow_operation > /tmp/result.txt
} &
```

3. **Cache expensive operations:**
```bash
if [ -f ".cache/result" ]; then
    result=$(cat .cache/result)
else
    result=$(expensive_operation)
    echo "$result" > .cache/result
fi
```

4. **Exit early:**
```bash
if [[ "$tool_name" != "write_to_file" ]]; then
    echo '{"cancel": false}'
    exit 0  # Skip unnecessary checks
fi
```

### How do I debug JSON output issues?

```bash
# Test your hook
output=$(.clinerules/hooks/PreToolUse < test-input.json)

# Validate JSON
echo "$output" | jq .

# If error, check for:
# 1. Multiple JSON objects (only last one counts)
# 2. Non-JSON text on stdout (use stderr for logging!)
# 3. Unescaped quotes
# 4. Missing commas or braces

# Use jq to generate valid JSON
jq -n --arg msg "$message" '{"cancel": false, "errorMessage": $msg}'
```

---

## Advanced Topics

### Can hooks call external APIs?

Yes, but with caution:

```bash
#!/usr/bin/env bash
input=$(cat)

# Always use timeouts!
response=$(curl --max-time 2 -s https://api.example.com/check || echo "timeout")

if [[ "$response" == *"error"* ]]; then
    context="API Error: $response"
    echo "{\"cancel\": false, \"contextModification\": \"$context\"}"
    exit 0
fi

echo '{"cancel": false}'
```

**Best Practices:**
- Always set timeouts (2-5 seconds)
- Handle failures gracefully
- Don't block on API errors
- Cache results when possible
- Consider rate limits

### Can hooks modify files?

Yes, but be careful:

```bash
#!/usr/bin/env bash
input=$(cat)
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

if [[ "$tool_name" == "write_to_file" ]]; then
    file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
    
    # Create a parallel file
    echo "File $file_path is about to be created" > "$file_path.log"
fi

echo '{"cancel": false}'
```

**Caution:** Don't modify the file Cline is trying to write - it will overwrite your changes!

### Can hooks maintain state across calls?

Yes, using files or databases:

```bash
#!/usr/bin/env bash
input=$(cat)

STATE_FILE="/tmp/cline-hook-state.json"

# Load state
if [ -f "$STATE_FILE" ]; then
    state=$(cat "$STATE_FILE")
else
    state='{"counter": 0}'
fi

# Update state
counter=$(echo "$state" | jq -r '.counter')
((counter++))
state=$(echo "$state" | jq ".counter = $counter")

# Save state
echo "$state" > "$STATE_FILE"

# Use state
if [ "$counter" -gt 10 ]; then
    context="You've performed $counter operations"
    echo "{\"cancel\": false, \"contextModification\": \"$context\"}"
    exit 0
fi

echo '{"cancel": false}'
```

### Can I have different hooks for different projects?

Yes! Two approaches:

**Approach 1: Project-specific hooks (Recommended)**
```
project-A/
└── .clinerules/hooks/PreToolUse

project-B/
└── .clinerules/hooks/PreToolUse

# Each project has its own hooks
```

**Approach 2: Conditional logic**
```bash
#!/usr/bin/env bash
input=$(cat)
workspace=$(echo "$input" | jq -r '.workspaceRoots[0]')

if [[ "$workspace" == *"project-A"* ]]; then
    # Project A specific logic
elif [[ "$workspace" == *"project-B"* ]]; then
    # Project B specific logic
fi

echo '{"cancel": false}'
```

### How do I share hooks with my team?

1. **Commit hooks to version control:**
```bash
git add .clinerules/hooks/
git commit -m "Add Cline hooks for project standards"
git push
```

2. **Document hook behavior:**
```markdown
# README.md

## Cline Hooks

This project uses Cline hooks to enforce:
- TypeScript-only files
- Security checks on commands
- Automatic project detection

Hooks are in `.clinerules/hooks/`
```

3. **Make hooks executable in setup:**
```bash
# setup.sh
chmod +x .clinerules/hooks/*
```

4. **Consider global organization hooks:**
```
~/Documents/Cline/Rules/Hooks/
└── PreToolUse  # Enforces org-wide standards
```

---

## Troubleshooting

### Hook works manually but not with Cline

**Check:**
1. Restart VS Code / Reload window
2. Verify "Enable Hooks" is checked
3. Check VS Code Output panel for errors
4. Ensure hook has shebang line
5. Test with fresh Cline task

### Hook blocks too much / too little

**Too Much:**
- Check your conditions are specific enough
- Add logging to see what's matching
- Test with different inputs

**Too Little:**
- Conditions might be too narrow
- Check for typos in patterns
- Verify you're checking the right tool type

### Different behavior on different machines

**Common causes:**
- Different `jq` versions
- Different shell versions (bash vs zsh)
- Different file paths
- Missing dependencies

**Solutions:**
- Use `#!/usr/bin/env bash` explicitly
- Check for required commands: `command -v jq >/dev/null`
- Make paths absolute where needed
- Document dependencies

---

## Best Practices

### Do's ✅

- Start simple (logging only)
- Test hooks manually first
- Use clear, descriptive error messages
- Log to stderr for debugging
- Keep hooks fast (< 1 second)
- Handle errors gracefully
- Document hook behavior
- Version control your hooks
- Review hooks from others before using

### Don'ts ❌

- Don't block everything
- Don't make network calls without timeouts
- Don't use hooks for long-running tasks
- Don't output non-JSON to stdout
- Don't trust user input (validate it)
- Don't modify files Cline is writing
- Don't use hooks for malicious purposes
- Don't run untrusted hooks

---

## Getting Help

### Resources

- **Documentation:** See included guides (cheatsheet.md, debugging-guide.md, etc.)
- **Examples:** Check the exercise files
- **Community:** Cline GitHub discussions
- **VS Code Output:** View > Output > Select "Cline"

### Common Questions to Ask

1. "What hook type should I use for X?"
2. "How do I match pattern Y?"
3. "Why isn't my context working?"
4. "How do I test this hook?"
5. "Is this hook pattern secure?"

---

**Remember:** Hooks are powerful tools. Start simple, test thoroughly, and build complexity gradually!
