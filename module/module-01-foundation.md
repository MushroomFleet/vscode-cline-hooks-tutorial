# Module 1: Foundation - Understanding Hook Mechanics

**Duration:** 1-2 hours | **Difficulty:** Beginner

## 🎯 Learning Objectives

By the end of this module, you will:

- ✅ Understand how hooks execute and when they trigger
- ✅ Master JSON input/output patterns for hook communication
- ✅ Know how to create, configure, and test hook files
- ✅ Grasp the critical timing difference between PreToolUse and PostToolUse
- ✅ Be able to read and parse hook data structures

## 📖 Theory: How Hooks Work

### The Hook Lifecycle

1. **Event Trigger** - Something happens (task starts, tool runs, etc.)
2. **Hook Invocation** - Cline executes your hook script
3. **Data Input** - JSON payload sent to your hook via stdin
4. **Processing** - Your script processes the data
5. **Response Output** - Your script returns JSON via stdout
6. **Action** - Cline responds based on your output

### Critical Concepts

**Hooks are synchronous** - They block execution until they complete
- Keep them fast (<5 seconds recommended)
- Avoid expensive operations
- Use background processes for heavy lifting

**Context timing matters** - Understanding when your context takes effect
- PreToolUse: Can block the current operation
- PostToolUse: Context affects the NEXT AI decision
- TaskStart: Context shapes the entire task

**JSON is the language** - All communication happens via JSON
- Input: Cline sends JSON to your hook's stdin
- Output: Your hook writes JSON to stdout
- Logging: Use stderr for debug output

## 🔧 Exercise 1.1: "Hello Hooks" - Your First Hook

**Duration:** 15 minutes  
**Hook Type:** TaskStart  
**Difficulty:** ⭐ Beginner

### Scenario

You want to know when Cline starts working on a new task. Create a simple logging hook that records task starts to a file, helping you understand the basics of hook execution.

### What You'll Build

A TaskStart hook that:
- Receives task metadata
- Extracts key information
- Logs to a file
- Returns success response

### Step-by-Step Instructions

#### 1. Create Your Hooks Directory

```bash
# Navigate to your project
cd ~/cline-hooks-practice

# Create the hooks directory
mkdir -p .clinerules/hooks

# Navigate to hooks directory
cd .clinerules/hooks
```

#### 2. Create the TaskStart Hook File

```bash
# Create the file (note: NO file extension)
touch TaskStart

# Open in your editor
vim TaskStart
# or
code TaskStart
```

#### 3. Write the Hook Script

Copy this script into your `TaskStart` file:

```bash
#!/usr/bin/env bash

# This is your first hook!
# The shebang line (above) tells the system how to execute this file

# STEP 1: Read the JSON input that Cline sends us
# The input comes via stdin (standard input)
input=$(cat)

# STEP 2: Parse the JSON to extract useful information
# We use 'jq' (a JSON processor) to extract specific fields

# Get the task ID
task_id=$(echo "$input" | jq -r '.taskId')

# Get the initial task description
initial_task=$(echo "$input" | jq -r '.taskStart.taskMetadata.initialTask')

# Get the timestamp
timestamp=$(echo "$input" | jq -r '.timestamp')

# STEP 3: Do something useful - log to a file
log_file="$HOME/cline-hooks.log"

echo "========================================" >> "$log_file"
echo "Task Started at: $timestamp" >> "$log_file"
echo "Task ID: $task_id" >> "$log_file"
echo "Initial Request: $initial_task" >> "$log_file"
echo "========================================" >> "$log_file"
echo "" >> "$log_file"

# STEP 4: Return a JSON response to Cline
# This tells Cline whether to continue and optionally adds context
echo '{
  "cancel": false,
  "contextModification": ""
}'

# That's it! Your first hook is complete.
```

#### 4. Make the Hook Executable

```bash
# Give the hook execute permissions
chmod +x TaskStart

# Verify it's executable
ls -l TaskStart
# Should show: -rwxr-xr-x (the x means executable)
```

#### 5. Test Your Hook Manually

Before testing with Cline, let's test the hook directly:

```bash
# Create a test JSON payload
cat << 'EOF' | ./TaskStart
{
  "clineVersion": "1.0.0",
  "hookName": "TaskStart",
  "timestamp": "2025-11-06T10:30:00Z",
  "taskId": "test-task-123",
  "workspaceRoots": ["/workspace"],
  "userId": "user-001",
  "taskStart": {
    "taskMetadata": {
      "taskId": "test-task-123",
      "ulid": "01HFAKE",
      "initialTask": "Create a hello world application"
    }
  }
}
EOF
```

**Expected Output:**
- You should see the JSON response: `{"cancel": false, "contextModification": ""}`
- Check the log file: `cat ~/cline-hooks.log`
- You should see your logged task information

#### 6. Enable Hooks in Cline

1. Open VSCode with Cline installed
2. Click the Cline icon in the sidebar
3. Click "Settings" (gear icon, top right)
4. Navigate to "Feature" section
5. Check "Enable Hooks"
6. Restart VSCode to ensure settings take effect

#### 7. Test with Real Cline Task

1. Open Cline in your VSCode
2. Start a new task: "Create a simple README.md file"
3. Let Cline begin working
4. Check your log file:

```bash
cat ~/cline-hooks.log
```

You should see a new entry with the real task information!

### Understanding the Output

Let's break down what happened:

```bash
# Your hook received this structure:
{
  "clineVersion": "1.0.0",        # Version of Cline
  "hookName": "TaskStart",         # Which hook was triggered
  "timestamp": "2025-11-06...",    # When it triggered
  "taskId": "abc123...",           # Unique task identifier
  "workspaceRoots": ["/path"],     # Your project path
  "userId": "user123",             # Your user ID
  "taskStart": {                   # Hook-specific data
    "taskMetadata": {
      "taskId": "abc123",
      "ulid": "01H...",
      "initialTask": "Create..."   # What user asked for
    }
  }
}

# Your hook returned:
{
  "cancel": false,                 # Don't stop execution
  "contextModification": ""        # No context to add
}
```

### Key Takeaways

✅ **Hook files have NO extension** - Just the exact name (TaskStart)  
✅ **Must start with shebang** - `#!/usr/bin/env bash` or `#!/usr/bin/env python3`  
✅ **Must be executable** - `chmod +x TaskStart`  
✅ **Read from stdin** - `input=$(cat)`  
✅ **Parse with jq** - `echo "$input" | jq -r '.field'`  
✅ **Return JSON** - Always return valid JSON with cancel and contextModification

### Common Issues

**Hook not running?**
- Check "Enable Hooks" is checked in Cline settings
- Verify file is executable: `ls -l TaskStart`
- Check for syntax errors: `bash -n TaskStart`
- Look in VSCode Output panel (View → Output → Cline)

**No log file created?**
- Check file permissions on home directory
- Try using /tmp/cline-hooks.log instead
- Verify hook is actually executing (add `echo "test" >&2`)

**JSON parse errors?**
- Test jq queries separately: `echo "$input" | jq '.'`
- Use `-r` flag for raw output: `jq -r '.field'`
- Check field exists before accessing: `jq -r '.field // "default"'`

---

## 🔍 Exercise 1.2: "Inspector Gadget" - Understanding Hook Data

**Duration:** 20 minutes  
**Hook Type:** PreToolUse  
**Difficulty:** ⭐ Beginner

### Scenario

Before you can build sophisticated hooks, you need to understand exactly what data Cline provides. Create a diagnostic hook that captures and displays all the data available during tool operations.

### What You'll Build

A PreToolUse hook that:
- Captures complete JSON payloads
- Saves them for inspection
- Shows summaries in real-time
- Teaches you the data structure

### Learning Goals

- Understand tool operation data structure
- Learn what information is available
- Practice JSON navigation
- Build debugging skills

### Step-by-Step Instructions

#### 1. Create the PreToolUse Hook

```bash
cd .clinerules/hooks

# Create PreToolUse hook (we're using a different hook type now!)
touch PreToolUse

# Open in editor
code PreToolUse
```

#### 2. Write the Inspector Script

```bash
#!/usr/bin/env bash

# Inspector Gadget Hook
# This hook captures and displays all tool operation data

# Read the complete JSON input
input=$(cat)

# Create a timestamped log file in /tmp
timestamp=$(date +%Y%m%d_%H%M%S)
log_file="/tmp/cline-inspector-${timestamp}.json"

# Save the complete JSON for later inspection
echo "$input" | jq '.' > "$log_file"

# Extract key information for console display
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')
task_id=$(echo "$input" | jq -r '.taskId')
timestamp_iso=$(echo "$input" | jq -r '.timestamp')

# Print inspection summary to stderr (will show in Cline output)
echo "╔══════════════════════════════════════════╗" >&2
echo "║        TOOL INSPECTION REPORT            ║" >&2
echo "╚══════════════════════════════════════════╝" >&2
echo "" >&2
echo "🔧 Tool: $tool_name" >&2
echo "🆔 Task: ${task_id:0:8}..." >&2
echo "⏰ Time: $timestamp_iso" >&2
echo "📄 Full data saved to:" >&2
echo "   $log_file" >&2
echo "" >&2

# Let's inspect the parameters based on tool type
echo "📋 Parameters:" >&2

case "$tool_name" in
  "write_to_file")
    file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
    content_length=$(echo "$input" | jq -r '.preToolUse.parameters.content | length')
    echo "   - File: $file_path" >&2
    echo "   - Content length: $content_length characters" >&2
    ;;
  
  "read_file")
    file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
    echo "   - File: $file_path" >&2
    ;;
  
  "execute_command")
    command=$(echo "$input" | jq -r '.preToolUse.parameters.command')
    echo "   - Command: $command" >&2
    ;;
  
  "list_files")
    directory=$(echo "$input" | jq -r '.preToolUse.parameters.path')
    echo "   - Directory: $directory" >&2
    ;;
  
  *)
    echo "   - Check JSON file for full parameters" >&2
    ;;
esac

echo "" >&2
echo "──────────────────────────────────────────" >&2
echo "" >&2

# Always allow the operation to continue
echo '{
  "cancel": false,
  "contextModification": ""
}'
```

#### 3. Make It Executable

```bash
chmod +x PreToolUse
```

#### 4. Test the Inspector

Let's test with a sample tool operation:

```bash
# Simulate a write_to_file operation
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
      "path": "/workspace/src/index.ts",
      "content": "console.log('Hello, World!');"
    }
  }
}
EOF
```

You should see a nicely formatted inspection report!

#### 5. Test with Real Cline Operations

1. Start a task in Cline: "Create a new file called test.txt with some content"
2. Watch the VSCode Output panel (View → Output → Select "Cline")
3. You'll see inspection reports as tools are used
4. Check `/tmp/` for saved JSON files:

```bash
ls -lt /tmp/cline-inspector-*.json | head -5
```

#### 6. Explore the Captured Data

Open one of the captured JSON files:

```bash
# Find the most recent inspector file
latest_file=$(ls -t /tmp/cline-inspector-*.json | head -1)

# View it with jq for nice formatting
cat "$latest_file" | jq '.'
```

### Data Structure Exploration

Now let's understand what you're seeing. Every hook receives **base fields** plus **hook-specific data**:

#### Base Fields (Present in ALL hooks)

```json
{
  "clineVersion": "string",      // Cline version number
  "hookName": "PreToolUse",       // Which hook triggered
  "timestamp": "2025-11-06...",   // ISO 8601 timestamp
  "taskId": "unique-id",          // Current task identifier
  "workspaceRoots": ["/path"],    // Project directories
  "userId": "user-id"             // Your user identifier
}
```

#### PreToolUse Specific Fields

```json
{
  "preToolUse": {
    "toolName": "write_to_file",  // Which tool is about to run
    "parameters": {                // Tool-specific parameters
      // Contents vary by tool...
    }
  }
}
```

### Tool-Specific Parameters

Different tools have different parameters. Here are common ones:

**write_to_file:**
```json
{
  "path": "/path/to/file.txt",
  "content": "File contents here..."
}
```

**read_file:**
```json
{
  "path": "/path/to/file.txt"
}
```

**execute_command:**
```json
{
  "command": "npm install lodash"
}
```

**list_files:**
```json
{
  "path": "/path/to/directory",
  "recursive": true
}
```

### Discovery Exercise

**Task:** Run these operations in Cline and inspect the captured JSON:

1. **File Operations**
   - Ask Cline: "Read the package.json file"
   - Inspect: What fields does read_file have?

2. **Command Execution**
   - Ask Cline: "Run 'ls -la'"
   - Inspect: What does the command parameter look like?

3. **Multiple Operations**
   - Ask Cline: "Create three files: a.txt, b.txt, c.txt"
   - Inspect: Do you get separate JSON files for each operation?

### Understanding stdout vs stderr

Your hook uses two output streams:

```bash
# stdout (standard output) - for JSON response
echo '{"cancel": false}'

# stderr (standard error) - for logging/debugging
echo "Debug message" >&2
```

**Why?**
- Cline reads only stdout for the JSON response
- stderr output appears in VSCode Output panel
- This lets you log without breaking JSON parsing

### Key Takeaways

✅ **PreToolUse runs BEFORE tools execute** - You can inspect and block  
✅ **Complete data is available** - Everything you need to make decisions  
✅ **stderr is for debugging** - Use it for console output  
✅ **stdout is for JSON** - Must be valid JSON  
✅ **Different tools, different parameters** - Learn by inspection  
✅ **Timestamps help debugging** - Track hook execution timing

### Challenge: Build Your Own Inspector

Modify the inspector to:
1. Count how many times each tool is used
2. Save statistics to a summary file
3. Alert when unusual patterns occur (e.g., 10+ file writes in a row)

Hint: You'll need to maintain state between hook executions. Consider using a file in /tmp to track counts.

---

## 📊 Knowledge Check

Before moving to Module 2, ensure you understand:

1. **What is the purpose of the shebang line in a hook file?**
   <details>
   <summary>Click to reveal answer</summary>
   
   The shebang (`#!/usr/bin/env bash`) tells the operating system which interpreter to use when executing the script. Without it, the system won't know how to run your hook.
   </details>

2. **Why do hook files not have file extensions?**
   <details>
   <summary>Click to reveal answer</summary>
   
   Cline looks for hooks by exact filename (e.g., "TaskStart", "PreToolUse"). Extensions would prevent Cline from finding and executing your hooks.
   </details>

3. **What's the difference between stdout and stderr in hooks?**
   <details>
   <summary>Click to reveal answer</summary>
   
   - stdout: Used for JSON response to Cline (must be valid JSON)
   - stderr: Used for logging and debugging (appears in VSCode Output panel)
   </details>

4. **When does a TaskStart hook run vs a PreToolUse hook?**
   <details>
   <summary>Click to reveal answer</summary>
   
   - TaskStart: Once when a new task begins
   - PreToolUse: Before every tool operation during the task
   </details>

5. **What does `jq -r` do vs just `jq`?**
   <details>
   <summary>Click to reveal answer</summary>
   
   - `jq`: Returns JSON-formatted output (with quotes)
   - `jq -r`: Returns raw output (without quotes, just the value)
   </details>

## 🎯 Module 1 Completion Checklist

- [ ] Created and tested TaskStart hook
- [ ] Log file contains task information
- [ ] Created and tested PreToolUse hook
- [ ] Successfully captured tool operation data
- [ ] Can read and interpret captured JSON
- [ ] Understand stdin/stdout/stderr differences
- [ ] Know how to make hooks executable
- [ ] Passed knowledge check questions

## 🚀 Next Steps

Congratulations! You now understand the fundamentals of hook mechanics.

Ready to learn how to validate and block operations? Continue to:

**[Module 2: Validation & Enforcement →](module-02-validation.md)**

Or return to the [Course Overview](README.md)

---

## 📚 Additional Resources

- [jq Tutorial](https://stedolan.github.io/jq/tutorial/)
- [Bash Scripting Cheatsheet](https://devhints.io/bash)
- [JSON.org](https://www.json.org/) - Understanding JSON structure
- [Cline Official Hooks Documentation](https://cline.bot/docs/hooks)
