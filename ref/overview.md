# Cline Hooks Educational Course Plan

## Course Overview & Learning Path

This comprehensive course will teach developers how to leverage Cline's hooks system to create intelligent, automated workflows. The course progresses from basic hook mechanics to advanced integration patterns.

---

## Module 1: Foundation - Understanding Hook Mechanics

### Learning Objectives
- Understand hook execution model and timing
- Master JSON input/output patterns
- Learn hook file setup and permissions
- Grasp the difference between blocking and learning hooks

### Exercise 1.1: "Hello Hooks" - Your First Hook
**Duration:** 15 minutes  
**Hook Type:** TaskStart  
**Difficulty:** Beginner

**Scenario:** Create a simple logging hook that announces when a task starts.

**Step-by-Step:**
```bash
# 1. Create hooks directory
mkdir -p .clinerules/hooks
cd .clinerules/hooks

# 2. Create TaskStart hook
cat > TaskStart << 'EOF'
#!/usr/bin/env bash

# Read the JSON input
input=$(cat)

# Extract the task ID
task_id=$(echo "$input" | jq -r '.taskId')
initial_task=$(echo "$input" | jq -r '.taskStart.taskMetadata.initialTask')

# Log to a file (for learning purposes)
echo "[$(date)] Task Started: $task_id" >> ~/cline-hooks.log
echo "Initial request: $initial_task" >> ~/cline-hooks.log

# Return success (don't block)
echo '{
  "cancel": false,
  "contextModification": ""
}'
EOF

# 3. Make executable
chmod +x TaskStart

# 4. Test by viewing the log structure
echo '{"clineVersion":"1.0","hookName":"TaskStart","timestamp":"2025-11-06","taskId":"test-123","workspaceRoots":["/workspace"],"userId":"user1","taskStart":{"taskMetadata":{"taskId":"test-123","ulid":"01H","initialTask":"Create a hello world app"}}}' | ./TaskStart
```

**What You'll Learn:**
- Hook file naming convention (no extension)
- Reading JSON from stdin with `$(cat)`
- Parsing JSON with `jq`
- Returning proper JSON response
- Making hooks executable

**Validation:**
- Check `~/cline-hooks.log` for output
- Start a real Cline task and verify logging

---

### Exercise 1.2: "Inspector Gadget" - Understanding Hook Data
**Duration:** 20 minutes  
**Hook Type:** PreToolUse  
**Difficulty:** Beginner

**Scenario:** Create a diagnostic hook that shows you exactly what data each tool operation contains.

**Step-by-Step:**
```bash
cat > PreToolUse << 'EOF'
#!/usr/bin/env bash

input=$(cat)

# Create a timestamped log file
log_file="/tmp/cline-tool-inspector-$(date +%s).json"

# Pretty-print the entire payload
echo "$input" | jq '.' > "$log_file"

# Extract key information
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')
timestamp=$(echo "$input" | jq -r '.timestamp')

# Print summary to console (will appear in Cline output)
echo "=== Tool Inspection ===" >&2
echo "Tool: $tool_name" >&2
echo "Time: $timestamp" >&2
echo "Full data saved to: $log_file" >&2
echo "=====================" >&2

# Don't block anything, just observe
echo '{
  "cancel": false,
  "contextModification": ""
}'
EOF

chmod +x PreToolUse
```

**What You'll Learn:**
- How to inspect all available data fields
- Difference between stdout (JSON) and stderr (logging)
- Tool operation structure
- Debugging techniques for hooks

**Validation Tasks:**
1. Trigger various tool operations (write_to_file, execute_command, etc.)
2. Examine the JSON files in `/tmp/`
3. Identify common vs. tool-specific fields

---

## Module 2: Validation & Enforcement

### Learning Objectives
- Implement blocking logic with `cancel: true`
- Create validation rules for file operations
- Handle error messages effectively
- Build confidence in pre-execution hooks

### Exercise 2.1: "TypeScript Guardian" - Preventing Wrong File Types
**Duration:** 25 minutes  
**Hook Type:** PreToolUse  
**Difficulty:** Intermediate

**Scenario:** You're working on a TypeScript project. Block any attempt to create `.js` files, enforcing `.ts` usage.

**Step-by-Step:**
```bash
cat > PreToolUse << 'EOF'
#!/usr/bin/env bash

input=$(cat)

tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

# Only check file write operations
if [[ "$tool_name" == "write_to_file" ]]; then
    # Extract the file path
    file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
    
    # Check if it's a .js file
    if [[ "$file_path" == *.js ]]; then
        echo '{
          "cancel": true,
          "contextModification": "",
          "errorMessage": "🚫 JavaScript files are not allowed in this TypeScript project. Please use .ts extension instead. File blocked: '"$file_path"'"
        }'
        exit 0
    fi
fi

# Allow everything else
echo '{
  "cancel": false,
  "contextModification": ""
}'
EOF

chmod +x PreToolUse
```

**What You'll Learn:**
- Conditional logic in hooks
- Blocking operations with `cancel: true`
- Providing helpful error messages
- Tool-specific validation

**Challenge Extensions:**
1. Also block `.jsx` files, suggest `.tsx`
2. Allow `.js` in a `scripts/` directory
3. Check for matching `.test.js` / `.test.ts` patterns

---

### Exercise 2.2: "The Security Sentinel" - Preventing Dangerous Commands
**Duration:** 30 minutes  
**Hook Type:** PreToolUse  
**Difficulty:** Intermediate

**Scenario:** Block dangerous shell commands that could harm the system.

**Step-by-Step:**
```bash
cat > PreToolUse << 'EOF'
#!/usr/bin/env bash

input=$(cat)

tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

# Check execute_command operations
if [[ "$tool_name" == "execute_command" ]]; then
    command=$(echo "$input" | jq -r '.preToolUse.parameters.command')
    
    # List of dangerous patterns
    dangerous_patterns=(
        "rm -rf /"
        "rm -rf /*"
        ":(){ :|:& };:"  # fork bomb
        "mkfs"
        "dd if=/dev/zero"
        "> /dev/sda"
    )
    
    # Check each pattern
    for pattern in "${dangerous_patterns[@]}"; do
        if [[ "$command" == *"$pattern"* ]]; then
            echo '{
              "cancel": true,
              "contextModification": "",
              "errorMessage": "⛔ BLOCKED: Potentially dangerous command detected. Pattern: '"$pattern"'"
            }'
            exit 0
        fi
    done
    
    # Warn about sudo usage but don't block
    if [[ "$command" == sudo* ]]; then
        echo '{
          "cancel": false,
          "contextModification": "⚠️ WARNING: A sudo command was executed. Command: '"$command"'"
        }'
        exit 0
    fi
fi

echo '{
  "cancel": false,
  "contextModification": ""
}'
EOF

chmod +x PreToolUse
```

**What You'll Learn:**
- Pattern matching in bash
- Building blocklists
- Difference between blocking and warning
- Security-focused validation

---

## Module 3: Context Injection & Learning

### Learning Objectives
- Understand context timing (future decisions)
- Build project knowledge dynamically
- Use PostToolUse for learning patterns
- Create intelligent context modifications

### Exercise 3.1: "Project Detective" - Auto-Discovering Tech Stack
**Duration:** 35 minutes  
**Hook Type:** TaskStart + PostToolUse  
**Difficulty:** Intermediate

**Scenario:** Automatically detect the project's tech stack and inject guidance for Cline.

**Step-by-Step:**
```bash
# First, create TaskStart to initialize detection
cat > TaskStart << 'EOF'
#!/usr/bin/env bash

input=$(cat)

workspace=$(echo "$input" | jq -r '.workspaceRoots[0]')

# Detect project type
context=""

if [ -f "$workspace/package.json" ]; then
    # It's a Node.js project
    has_typescript=$(grep -q "typescript" "$workspace/package.json" && echo "yes" || echo "no")
    has_react=$(grep -q "react" "$workspace/package.json" && echo "yes" || echo "no")
    has_next=$(grep -q "next" "$workspace/package.json" && echo "yes" || echo "no")
    
    context="WORKSPACE_RULES: This is a Node.js project."
    
    if [ "$has_typescript" == "yes" ]; then
        context="$context Use TypeScript (.ts/.tsx) for all new files."
    fi
    
    if [ "$has_react" == "yes" ]; then
        context="$context This project uses React. Follow React best practices."
    fi
    
    if [ "$has_next" == "yes" ]; then
        context="$context This is a Next.js project. Use app router conventions."
    fi
fi

if [ -f "$workspace/pyproject.toml" ] || [ -f "$workspace/setup.py" ]; then
    context="WORKSPACE_RULES: This is a Python project. Follow PEP 8 style guidelines."
fi

if [ -f "$workspace/Cargo.toml" ]; then
    context="WORKSPACE_RULES: This is a Rust project. Follow Rust idioms and use cargo for dependencies."
fi

echo "{
  \"cancel\": false,
  \"contextModification\": \"$context\"
}"
EOF

chmod +x TaskStart
```

**What You'll Learn:**
- File system inspection in hooks
- Building contextual guidance
- Multi-condition logic
- Dynamic rule generation

**Challenge Extensions:**
1. Detect testing frameworks and suggest test file patterns
2. Check for linting configs and reference them
3. Identify CI/CD files and mention deployment practices

---

### Exercise 3.2: "Performance Tracker" - Learning from Operation Times
**Duration:** 30 minutes  
**Hook Type:** PostToolUse  
**Difficulty:** Intermediate

**Scenario:** Track which operations are slow and inject performance guidance.

**Step-by-Step:**
```bash
cat > PostToolUse << 'EOF'
#!/usr/bin/env bash

input=$(cat)

tool_name=$(echo "$input" | jq -r '.postToolUse.toolName')
exec_time=$(echo "$input" | jq -r '.postToolUse.executionTimeMs')
success=$(echo "$input" | jq -r '.postToolUse.success')

# Define slow thresholds (in ms)
SLOW_THRESHOLD=5000  # 5 seconds

context=""

if [ "$success" == "true" ] && [ "$exec_time" -gt "$SLOW_THRESHOLD" ]; then
    exec_seconds=$((exec_time / 1000))
    
    case "$tool_name" in
        "execute_command")
            command=$(echo "$input" | jq -r '.postToolUse.parameters.command')
            context="PERFORMANCE: Command took ${exec_seconds}s: $command. Consider optimizing or breaking into smaller steps."
            ;;
        "write_to_file")
            file_path=$(echo "$input" | jq -r '.postToolUse.parameters.path')
            context="PERFORMANCE: Writing to $file_path took ${exec_seconds}s. Large file detected."
            ;;
        "read_file")
            file_path=$(echo "$input" | jq -r '.postToolUse.parameters.path')
            context="PERFORMANCE: Reading $file_path took ${exec_seconds}s. Consider reading in chunks or streaming."
            ;;
    esac
fi

# Log all operations to performance DB
echo "$input" >> /tmp/cline-performance-log.jsonl

echo "{
  \"cancel\": false,
  \"contextModification\": \"$context\"
}"
EOF

chmod +x PostToolUse
```

**What You'll Learn:**
- Performance monitoring
- Conditional context injection
- Building historical data
- Threshold-based alerts

---

## Module 4: Advanced Integrations

### Learning Objectives
- Connect hooks to external tools
- Build multi-hook workflows
- Implement persistent state
- Create production-ready hooks

### Exercise 4.1: "The Quality Gate" - Automated Code Review
**Duration:** 45 minutes  
**Hook Types:** PreToolUse + PostToolUse  
**Difficulty:** Advanced

**Scenario:** Run linters before committing code and track code quality metrics.

**Step-by-Step:**
```bash
# PreToolUse: Run linters before file writes
cat > PreToolUse << 'EOF'
#!/usr/bin/env bash

input=$(cat)

tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

if [[ "$tool_name" == "write_to_file" ]]; then
    file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
    file_content=$(echo "$input" | jq -r '.preToolUse.parameters.content')
    
    # Check TypeScript files
    if [[ "$file_path" == *.ts ]] || [[ "$file_path" == *.tsx ]]; then
        # Create temp file
        temp_file="/tmp/cline-lint-$$.ts"
        echo "$file_content" > "$temp_file"
        
        # Run ESLint if available
        if command -v eslint &> /dev/null; then
            lint_output=$(eslint "$temp_file" 2>&1 || true)
            
            if [ ! -z "$lint_output" ]; then
                rm "$temp_file"
                echo "{
                  \"cancel\": true,
                  \"contextModification\": \"\",
                  \"errorMessage\": \"❌ Linting errors detected in $file_path:\n$lint_output\n\nPlease fix these issues before saving.\"
                }"
                exit 0
            fi
        fi
        
        rm "$temp_file"
    fi
    
    # Check Python files
    if [[ "$file_path" == *.py ]]; then
        temp_file="/tmp/cline-lint-$$.py"
        echo "$file_content" > "$temp_file"
        
        # Run flake8 if available
        if command -v flake8 &> /dev/null; then
            lint_output=$(flake8 "$temp_file" 2>&1 || true)
            
            if [ ! -z "$lint_output" ]; then
                rm "$temp_file"
                echo "{
                  \"cancel\": true,
                  \"contextModification\": \"\",
                  \"errorMessage\": \"❌ Style errors detected in $file_path:\n$lint_output\"
                }"
                exit 0
            fi
        fi
        
        rm "$temp_file"
    fi
fi

echo '{
  "cancel": false,
  "contextModification": ""
}'
EOF

chmod +x PreToolUse

# PostToolUse: Track code quality metrics
cat > PostToolUse << 'EOF'
#!/usr/bin/env bash

input=$(cat)

tool_name=$(echo "$input" | jq -r '.postToolUse.toolName')
success=$(echo "$input" | jq -r '.postToolUse.success')

if [[ "$tool_name" == "write_to_file" ]] && [[ "$success" == "true" ]]; then
    file_path=$(echo "$input" | jq -r '.postToolUse.parameters.path')
    
    # Count lines of code
    if [[ -f "$file_path" ]]; then
        loc=$(wc -l < "$file_path")
        
        # Log to metrics file
        echo "{\"timestamp\":\"$(date -Iseconds)\",\"file\":\"$file_path\",\"loc\":$loc}" >> /tmp/cline-code-metrics.jsonl
        
        # Alert on very large files
        if [ "$loc" -gt 500 ]; then
            echo "{
              \"cancel\": false,
              \"contextModification\": \"CODE_QUALITY: File $file_path has $loc lines. Consider breaking it into smaller modules.\"
            }"
            exit 0
        fi
    fi
fi

echo '{
  "cancel": false,
  "contextModification": ""
}'
EOF

chmod +x PostToolUse
```

**What You'll Learn:**
- Running external tools from hooks
- Temp file handling
- Combining pre and post hooks
- Building quality metrics

---

### Exercise 4.2: "The Integration Hub" - Connecting to External Services
**Duration:** 50 minutes  
**Hook Type:** PostToolUse  
**Difficulty:** Advanced

**Scenario:** Send notifications to Slack/Discord when important operations complete, and update Linear/Jira tickets.

**Step-by-Step:**
```bash
cat > PostToolUse << 'EOF'
#!/usr/bin/env python3
import json
import sys
import requests
import os
from datetime import datetime

# Read JSON from stdin
input_data = json.loads(sys.stdin.read())

tool_name = input_data['postToolUse']['toolName']
success = input_data['postToolUse']['success']
task_id = input_data['taskId']

# Configuration (use environment variables in production)
SLACK_WEBHOOK = os.getenv('CLINE_SLACK_WEBHOOK', '')
DISCORD_WEBHOOK = os.getenv('CLINE_DISCORD_WEBHOOK', '')

def send_slack_notification(message):
    if SLACK_WEBHOOK:
        try:
            requests.post(SLACK_WEBHOOK, json={"text": message}, timeout=5)
        except:
            pass  # Silent fail, don't block Cline

def send_discord_notification(message):
    if DISCORD_WEBHOOK:
        try:
            requests.post(DISCORD_WEBHOOK, json={"content": message}, timeout=5)
        except:
            pass

# Track completed tasks
if tool_name == "execute_command" and success:
    command = input_data['postToolUse']['parameters'].get('command', '')
    
    # Notify on test runs
    if 'test' in command or 'pytest' in command or 'jest' in command:
        result = input_data['postToolUse'].get('result', '')
        
        # Parse for pass/fail
        if 'passed' in result.lower() or 'ok' in result.lower():
            message = f"✅ Tests passed in task {task_id[:8]}"
            send_slack_notification(message)
        elif 'failed' in result.lower() or 'error' in result.lower():
            message = f"❌ Tests failed in task {task_id[:8]}"
            send_slack_notification(message)
    
    # Notify on deployments
    if any(keyword in command for keyword in ['deploy', 'publish', 'release']):
        message = f"🚀 Deployment triggered: {command}"
        send_discord_notification(message)

# Log significant file operations
if tool_name == "write_to_file" and success:
    file_path = input_data['postToolUse']['parameters'].get('path', '')
    
    # Track migrations or schema changes
    if 'migration' in file_path or 'schema' in file_path:
        message = f"📝 Database change: {file_path}"
        send_slack_notification(message)

# Always allow
response = {
    "cancel": False,
    "contextModification": ""
}

print(json.dumps(response))
EOF

chmod +x PostToolUse
```

**What You'll Learn:**
- Using Python for complex hooks
- HTTP requests from hooks
- Async notification patterns
- Production configuration practices

**Setup Instructions:**
```bash
# Set environment variables
export CLINE_SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
export CLINE_DISCORD_WEBHOOK="https://discord.com/api/webhooks/YOUR/WEBHOOK"
```

---

### Exercise 4.3: "State Machine" - Persistent Hook State
**Duration:** 40 minutes  
**Hook Types:** TaskStart + PostToolUse + TaskCancel  
**Difficulty:** Advanced

**Scenario:** Track project state across multiple tasks and sessions using a persistent database.

**Step-by-Step:**
```bash
# Create shared state manager
cat > /tmp/cline-state-manager.py << 'EOF'
#!/usr/bin/env python3
import json
import sqlite3
from pathlib import Path

DB_PATH = Path.home() / ".cline-hooks-state.db"

def init_db():
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS task_state (
            task_id TEXT PRIMARY KEY,
            started_at TEXT,
            file_writes INTEGER DEFAULT 0,
            commands_run INTEGER DEFAULT 0,
            status TEXT DEFAULT 'active'
        )
    ''')
    conn.commit()
    conn.close()

def update_state(task_id, **kwargs):
    init_db()
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    
    # Check if task exists
    cursor.execute('SELECT task_id FROM task_state WHERE task_id = ?', (task_id,))
    if not cursor.fetchone():
        cursor.execute('INSERT INTO task_state (task_id, started_at) VALUES (?, datetime("now"))', (task_id,))
    
    # Update fields
    for key, value in kwargs.items():
        if key in ['file_writes', 'commands_run']:
            cursor.execute(f'UPDATE task_state SET {key} = {key} + ? WHERE task_id = ?', (value, task_id))
        else:
            cursor.execute(f'UPDATE task_state SET {key} = ? WHERE task_id = ?', (value, task_id))
    
    conn.commit()
    conn.close()

def get_state(task_id):
    init_db()
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM task_state WHERE task_id = ?', (task_id,))
    row = cursor.fetchone()
    conn.close()
    return row

if __name__ == "__main__":
    import sys
    action = sys.argv[1]
    task_id = sys.argv[2]
    
    if action == "init":
        update_state(task_id)
    elif action == "file_write":
        update_state(task_id, file_writes=1)
    elif action == "command":
        update_state(task_id, commands_run=1)
    elif action == "complete":
        update_state(task_id, status='completed')
    elif action == "get":
        print(json.dumps(get_state(task_id)))
EOF

chmod +x /tmp/cline-state-manager.py

# TaskStart: Initialize state
cat > TaskStart << 'EOF'
#!/usr/bin/env bash
input=$(cat)
task_id=$(echo "$input" | jq -r '.taskId')

/tmp/cline-state-manager.py init "$task_id"

echo '{
  "cancel": false,
  "contextModification": ""
}'
EOF

chmod +x TaskStart

# PostToolUse: Track operations
cat > PostToolUse << 'EOF'
#!/usr/bin/env bash
input=$(cat)
task_id=$(echo "$input" | jq -r '.taskId')
tool_name=$(echo "$input" | jq -r '.postToolUse.toolName')

case "$tool_name" in
    "write_to_file")
        /tmp/cline-state-manager.py file_write "$task_id"
        ;;
    "execute_command")
        /tmp/cline-state-manager.py command "$task_id"
        ;;
esac

# Get current state
state=$(/tmp/cline-state-manager.py get "$task_id")

# Provide summary periodically
file_writes=$(echo "$state" | jq -r '.[2]' 2>/dev/null || echo "0")
commands=$(echo "$state" | jq -r '.[3]' 2>/dev/null || echo "0")

context=""
if [ "$file_writes" -gt 20 ]; then
    context="PROJECT_STATUS: High activity - $file_writes files modified, $commands commands executed. Consider breaking into smaller tasks."
fi

echo "{
  \"cancel\": false,
  \"contextModification\": \"$context\"
}"
EOF

chmod +x PostToolUse
```

**What You'll Learn:**
- Persistent state management
- SQLite integration
- Cross-hook communication
- Activity tracking patterns

---

## Module 5: Production Patterns

### Learning Objectives
- Error handling and resilience
- Performance optimization
- Security best practices
- Team collaboration patterns

### Exercise 5.1: "Bulletproof Hooks" - Error Handling
**Duration:** 35 minutes  
**Difficulty:** Advanced

**Comprehensive error handling template:**

```bash
cat > PreToolUse << 'EOF'
#!/usr/bin/env bash

# Enable strict error handling
set -euo pipefail

# Logging function
log_error() {
    echo "[ERROR $(date -Iseconds)] $*" >> /tmp/cline-hooks-errors.log
}

# Main logic wrapped in try-catch pattern
main() {
    local input
    input=$(cat) || {
        log_error "Failed to read stdin"
        echo '{"cancel": false, "contextModification": ""}'
        exit 0
    }
    
    # Validate JSON
    if ! echo "$input" | jq empty 2>/dev/null; then
        log_error "Invalid JSON input"
        echo '{"cancel": false, "contextModification": ""}'
        exit 0
    fi
    
    # Extract with defaults
    local tool_name
    tool_name=$(echo "$input" | jq -r '.preToolUse.toolName // "unknown"')
    
    # Your logic here with error handling
    if [[ "$tool_name" == "write_to_file" ]]; then
        local file_path
        file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path // ""')
        
        if [[ -z "$file_path" ]]; then
            log_error "Empty file path in write_to_file"
            echo '{"cancel": false, "contextModification": ""}'
            exit 0
        fi
        
        # Add your validation logic
    fi
    
    # Success
    echo '{
      "cancel": false,
      "contextModification": ""
    }'
}

# Run main with timeout
timeout 10s bash -c "$(declare -f main); $(declare -f log_error); main" || {
    log_error "Hook timeout exceeded"
    echo '{"cancel": false, "contextModification": ""}'
}
EOF

chmod +x PreToolUse
```

**What You'll Learn:**
- Defensive programming
- Timeout handling
- Graceful degradation
- Logging strategies

---

### Exercise 5.2: "Team Hooks" - Collaborative Configuration
**Duration:** 30 minutes  
**Difficulty:** Advanced

**Scenario:** Create hooks that work across a team with individual preferences.

```bash
# Create team-wide config
cat > .clinerules/hooks-config.json << 'EOF'
{
  "team": {
    "linting": {
      "enabled": true,
      "tools": ["eslint", "prettier"]
    },
    "security": {
      "blockDangerousCommands": true,
      "allowedSudoCommands": []
    },
    "notifications": {
      "slackWebhook": "${TEAM_SLACK_WEBHOOK}"
    }
  },
  "personalOverrides": {
    "enabled": true,
    "configPath": "~/.cline-personal-config.json"
  }
}
EOF

# Hook that respects team + personal config
cat > PreToolUse << 'EOF'
#!/usr/bin/env python3
import json
import sys
import os
from pathlib import Path

def load_config():
    """Load team config with personal overrides"""
    # Load team config
    team_config_path = Path('.clinerules/hooks-config.json')
    config = {}
    
    if team_config_path.exists():
        with open(team_config_path) as f:
            config = json.load(f)['team']
    
    # Load personal overrides
    personal_config_path = Path.home() / '.cline-personal-config.json'
    if personal_config_path.exists():
        with open(personal_config_path) as f:
            personal = json.load(f)
            config.update(personal)
    
    return config

def main():
    input_data = json.loads(sys.stdin.read())
    config = load_config()
    
    tool_name = input_data['preToolUse']['toolName']
    
    # Example: Honor linting preferences
    if tool_name == "write_to_file" and config.get('linting', {}).get('enabled'):
        # Run linters...
        pass
    
    # Always allow by default
    print(json.dumps({
        "cancel": False,
        "contextModification": ""
    }))

if __name__ == "__main__":
    main()
EOF

chmod +x PreToolUse
```

**What You'll Learn:**
- Configuration management
- Team collaboration
- Personal preferences
- Config merging strategies

---

## Module 6: Real-World Projects

### Capstone Project 1: "Smart Documentation System"
**Duration:** 2-3 hours  
**Complexity:** Multi-hook workflow

**Objective:** Build a system that:
1. Detects when code changes affect public APIs
2. Checks if documentation was updated
3. Reminds developers to update docs
4. Auto-generates API diff summaries

**Hooks Required:**
- PostToolUse (file tracking)
- TaskStart (initialization)
- UserPromptSubmit (doc update detection)

---

### Capstone Project 2: "Compliance Auditor"
**Duration:** 2-3 hours  
**Complexity:** Advanced integration

**Objective:** Build an audit system for regulated industries:
1. Log all file modifications with timestamps
2. Require code review for security-sensitive files
3. Block changes to production configs
4. Generate compliance reports
5. Send audit logs to external system

**Hooks Required:**
- All tool execution hooks
- Task lifecycle hooks
- External API integration

---

## Course Materials Package

### For Each Exercise:
1. **README.md** - Exercise overview and learning objectives
2. **starter-template/** - Initial hook scaffolding
3. **solution/** - Complete working implementation
4. **tests/** - Validation scripts
5. **docs/** - Detailed explanations

### Additional Resources:
1. **cheatsheet.md** - Quick reference for all hook types
2. **debugging-guide.md** - Common issues and solutions
3. **patterns-library.md** - Reusable hook patterns
4. **faq.md** - Frequently asked questions

---

## Assessment Strategy

### Knowledge Checks (After each module):
- Quiz on hook mechanics
- Short answer questions
- Code reading exercises

### Practical Assessments:
- Build a hook for a given scenario
- Debug a broken hook
- Optimize a slow hook
- Code review other students' hooks

### Final Project:
Students create a complete hooks system for their own project with:
- At least 3 different hook types
- Documentation
- Tests
- Team usage guide

---

## Appendix: Quick Reference

### All Hook Types Summary:

| Hook | Timing | Primary Use | Can Block? |
|------|--------|-------------|-----------|
| TaskStart | Task begins | Initialize, detect project | No |
| TaskResume | Task resumes | Restore state | No |
| TaskCancel | Task cancelled | Cleanup | No |
| PreToolUse | Before tool runs | Validation, blocking | Yes ✅ |
| PostToolUse | After tool runs | Learning, metrics | No |
| UserPromptSubmit | User sends message | Input validation | Partial |

### Essential jq Commands:
```bash
# Extract value
echo "$input" | jq -r '.path.to.value'

# Check if key exists
echo "$input" | jq 'has("key")'

# Get array length
echo "$input" | jq '.array | length'

# Pretty print
echo "$input" | jq '.'
```

### Common Patterns:
```bash
# Read and validate
input=$(cat)
if ! echo "$input" | jq empty 2>/dev/null; then
    echo '{"cancel": false}'; exit 0
fi

# Extract with default
value=$(echo "$input" | jq -r '.key // "default"')

# Conditional blocking
if [[ condition ]]; then
    echo '{"cancel": true, "errorMessage": "reason"}'
    exit 0
fi
```

---

This comprehensive course plan provides a complete learning journey from basic hook concepts to production-ready implementations. Each exercise builds on previous knowledge while introducing new concepts and real-world applications.