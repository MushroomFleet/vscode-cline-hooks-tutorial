# Exercise 1.1: "Hello Hooks" - Your First Hook

**Module:** Foundation - Understanding Hook Mechanics  
**Duration:** 15 minutes  
**Hook Type:** TaskStart  
**Difficulty:** ⭐ Beginner

## Learning Objectives

By completing this exercise, you will:
- Understand the basic hook execution model
- Learn proper hook file naming conventions (no file extension)
- Master reading JSON input from stdin
- Practice parsing JSON with `jq`
- Create properly formatted JSON responses
- Understand file permissions for hooks

## Scenario

You're setting up your first Cline hook to get familiar with the system. You'll create a simple logging hook that announces when a task starts and records basic information about each task. This hook won't block anything - it's purely observational, making it perfect for learning.

## Prerequisites

- Cline installed and "Enable Hooks" checked in Settings
- Basic command line knowledge
- `jq` installed (check with `which jq`)

If `jq` is not installed:
```bash
# macOS
brew install jq

# Linux (Ubuntu/Debian)
sudo apt-get install jq

# Linux (Fedora)
sudo dnf install jq
```

## Step-by-Step Instructions

### Step 1: Create the Hooks Directory

First, create the directory where your hook files will live:

```bash
# Navigate to your project root
cd /path/to/your/project

# Create the hooks directory
mkdir -p .clinerules/hooks

# Navigate into it
cd .clinerules/hooks
```

**What's happening:**
- `.clinerules/hooks/` is a project-specific hooks location
- Hooks here only apply to this workspace
- The directory name is case-sensitive

### Step 2: Create the TaskStart Hook File

Create a file named exactly `TaskStart` (no extension):

```bash
cat > TaskStart << 'EOF'
#!/usr/bin/env bash

# Read the JSON input from stdin
input=$(cat)

# Extract specific fields using jq
task_id=$(echo "$input" | jq -r '.taskId')
initial_task=$(echo "$input" | jq -r '.taskStart.taskMetadata.initialTask')
timestamp=$(echo "$input" | jq -r '.timestamp')

# Create a log entry
log_message="[${timestamp}] Task Started: ${task_id}"
echo "$log_message" >> ~/cline-hooks.log
echo "Initial request: $initial_task" >> ~/cline-hooks.log
echo "---" >> ~/cline-hooks.log

# Return proper JSON response
# cancel: false means "don't block this action"
# contextModification: empty means "don't inject any context"
echo '{
  "cancel": false,
  "contextModification": ""
}'
EOF
```

**Code Breakdown:**

```bash
#!/usr/bin/env bash
```
This is the "shebang" - it tells the system this is a bash script. **Every hook must start with a shebang**.

```bash
input=$(cat)
```
This reads all data from stdin and stores it in the `input` variable. Cline sends hook data via stdin as JSON.

```bash
task_id=$(echo "$input" | jq -r '.taskId')
```
This uses `jq` to extract the `taskId` field from the JSON. The `-r` flag means "raw output" (without quotes).

```bash
echo '{
  "cancel": false,
  "contextModification": ""
}'
```
This is the required response. Every hook must output valid JSON with at least a `cancel` field.

### Step 3: Make the Hook Executable

```bash
chmod +x TaskStart
```

**Why this matters:**
- Hooks must be executable files
- Without execute permission, Cline won't run them
- You'll need to do this for every hook you create

### Step 4: Verify the Hook Structure

Let's test the hook manually before using it with Cline:

```bash
# Create sample input that matches what Cline sends
echo '{
  "clineVersion": "1.0.0",
  "hookName": "TaskStart",
  "timestamp": "2025-11-07T10:30:00Z",
  "taskId": "test-task-123",
  "workspaceRoots": ["/home/user/myproject"],
  "userId": "user-456",
  "taskStart": {
    "taskMetadata": {
      "taskId": "test-task-123",
      "ulid": "01HZXY123456789",
      "initialTask": "Create a hello world application"
    }
  }
}' | ./TaskStart
```

**Expected output:**
```json
{
  "cancel": false,
  "contextModification": ""
}
```

**Check the log file:**
```bash
cat ~/cline-hooks.log
```

You should see:
```
[2025-11-07T10:30:00Z] Task Started: test-task-123
Initial request: Create a hello world application
---
```

### Step 5: Test with Real Cline Usage

Now test it with actual Cline:

1. Open Cline in VS Code
2. Start any new task (e.g., "Create a simple README file")
3. Let Cline begin processing
4. Check your log file:

```bash
cat ~/cline-hooks.log
```

You should see new entries with real task data!

## Understanding the Data Structure

When TaskStart runs, Cline sends this JSON structure:

```json
{
  "clineVersion": "string",      // Version of Cline
  "hookName": "TaskStart",        // Which hook this is
  "timestamp": "ISO8601 string",  // When this happened
  "taskId": "string",             // Unique task identifier
  "workspaceRoots": ["string"],   // Project paths
  "userId": "string",             // User identifier
  "taskStart": {                  // TaskStart-specific data
    "taskMetadata": {
      "taskId": "string",         // Task ID (same as above)
      "ulid": "string",           // Another unique identifier
      "initialTask": "string"     // The user's original request
    }
  }
}
```

**Key fields explained:**
- `taskId`: Unique identifier for this task session
- `initialTask`: The exact text the user typed to start the task
- `timestamp`: ISO 8601 format timestamp
- `workspaceRoots`: Array of workspace paths (usually just one)

## Response Format

Every hook must return JSON with this structure:

```json
{
  "cancel": boolean,              // true = block action, false = allow
  "contextModification": "string", // Text to inject into conversation
  "errorMessage": "string"        // Optional, shown when cancel=true
}
```

For TaskStart hooks:
- `cancel` is typically `false` (you can't block task start)
- `contextModification` is usually empty (you'll learn this in Module 3)
- `errorMessage` is not used for TaskStart

## Common Issues and Solutions

### Issue: Hook doesn't run

**Check:**
```bash
# Is it executable?
ls -l .clinerules/hooks/TaskStart

# Should show: -rwxr-xr-x (the 'x' means executable)
```

**Fix:**
```bash
chmod +x .clinerules/hooks/TaskStart
```

### Issue: "jq: command not found"

**Fix:** Install jq:
```bash
# macOS
brew install jq

# Linux
sudo apt-get install jq
```

### Issue: No output in log file

**Check permissions:**
```bash
# Can you write to the log?
touch ~/cline-hooks.log
echo "test" >> ~/cline-hooks.log
```

### Issue: Syntax error in hook

**Test the hook manually:**
```bash
bash -n .clinerules/hooks/TaskStart
```
This checks syntax without running the script.

## Validation Checklist

- [ ] Hook file exists at `.clinerules/hooks/TaskStart`
- [ ] Hook file is executable (`chmod +x`)
- [ ] Hook starts with `#!/usr/bin/env bash`
- [ ] Hook outputs valid JSON
- [ ] Log file gets created at `~/cline-hooks.log`
- [ ] Starting a Cline task creates log entries
- [ ] JSON response includes `cancel` field

## What You Learned

✅ **Hook Naming**: Files must have exact names with no extensions  
✅ **Shebang Lines**: Every hook needs `#!/usr/bin/env bash` (or appropriate interpreter)  
✅ **Reading Input**: Use `input=$(cat)` to capture stdin  
✅ **JSON Parsing**: Use `jq` to extract values from JSON  
✅ **JSON Output**: Hooks must return valid JSON via stdout  
✅ **Permissions**: Hooks must be executable with `chmod +x`  
✅ **File Locations**: `.clinerules/hooks/` for project-specific hooks

## Challenge Extensions

Ready for more? Try these modifications:

### Challenge 1: Add More Details
Extract and log the `clineVersion` and `userId` fields.

### Challenge 2: Structured Logging
Instead of plain text, write JSON to the log file:
```bash
log_entry=$(jq -n \
  --arg timestamp "$timestamp" \
  --arg task_id "$task_id" \
  --arg task "$initial_task" \
  '{timestamp: $timestamp, taskId: $task_id, task: $task}')
echo "$log_entry" >> ~/cline-hooks.jsonl
```

### Challenge 3: Pretty Console Output
Also log to stderr so you see it in Cline's output panel:
```bash
echo "🚀 Starting task: $initial_task" >&2
```

### Challenge 4: Count Tasks
Keep a counter of how many tasks have started:
```bash
count=$(wc -l < ~/cline-hooks.log 2>/dev/null || echo "0")
echo "This is task #$count" >> ~/cline-hooks.log
```

## Next Steps

You've created your first hook! You now understand:
- The basic hook file structure
- How to read and parse JSON input
- How to return properly formatted responses
- The execution model for hooks

**Next Exercise:** [Exercise 1.2: Inspector Gadget](exercise-1.2-inspector-gadget.md)

You'll learn how to inspect all the data that hooks receive and understand different tool operations.

## Reference

**Hook Type:** TaskStart  
**Can Block:** No  
**Timing:** When a task begins  
**Common Uses:** Initialization, project detection, logging  
**Response Required:** `{"cancel": false, "contextModification": ""}`

**Files Created:**
- `.clinerules/hooks/TaskStart` - The hook script
- `~/cline-hooks.log` - Log output

**Further Reading:**
- [Cline Hooks Documentation](https://github.com/cline/cline/docs/hooks)
- [jq Manual](https://jqlang.github.io/jq/manual/)
- [Bash Scripting Guide](https://www.gnu.org/software/bash/manual/)
