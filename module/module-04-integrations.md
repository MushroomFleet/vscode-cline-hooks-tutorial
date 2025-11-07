# Module 4: Advanced Integrations

**Duration:** 3-4 hours | **Difficulty:** Advanced

## 🎯 Learning Objectives

By the end of this module, you will:

- ✅ Integrate hooks with external linters and code quality tools
- ✅ Connect to external services (Slack, Discord, etc.)
- ✅ Build multi-hook workflows
- ✅ Implement persistent state across hook executions
- ✅ Use Python for complex hook logic
- ✅ Handle HTTP requests and API integrations

## 📖 Theory: Hooks as Integration Points

### The Power of Integration

Hooks become truly powerful when they connect Cline to your broader development ecosystem:

**Quality Tools** - ESLint, Prettier, Pylint, cargo clippy  
**Services** - Slack, Discord, PagerDuty  
**Platforms** - Jira, Linear, GitHub, GitLab  
**Databases** - Log to PostgreSQL, analyze with SQL  
**Monitoring** - Datadog, New Relic, custom dashboards  

###Bash vs Python for Hooks

**Use Bash when:**
- Simple file/command operations
- String manipulation
- Quick checks and validations
- < 50 lines of logic

**Use Python when:**
- Complex data structures
- HTTP/API requests
- JSON manipulation beyond jq
- Error handling needs
- > 50 lines of logic

### External Tool Integration Patterns

**Pre-execution validation:**
```
User requests change → PreToolUse → Run linter → Block if fails
```

**Post-execution notifications:**
```
Operation completes → PostToolUse → Send to Slack → Continue
```

**State tracking:**
```
TaskStart → Initialize DB → PostToolUse updates → TaskCancel cleanup
```

## 🎨 Exercise 4.1: "The Quality Gate" - Automated Code Review

**Duration:** 45 minutes  
**Hook Types:** PreToolUse + PostToolUse  
**Difficulty:** ⭐⭐⭐ Advanced

### Scenario

You want to maintain high code quality without manual review of every change. Build a system that automatically runs linters before files are saved and tracks code quality metrics over time.

### What You'll Build

A two-hook system:
- **PreToolUse**: Runs linters before file writes, blocks if errors
- **PostToolUse**: Tracks code quality metrics and trends

### Real-World Value

- Prevent linting errors from entering codebase
- Maintain consistent code style
- Track quality metrics
- Catch issues before commit
- Save code review time

### Prerequisites

Install linting tools:
```bash
# For TypeScript/JavaScript
npm install -g eslint prettier

# For Python
pip install flake8 pylint black --break-system-packages

# For Rust (if applicable)
# cargo has clippy built-in
```

### Step-by-Step Instructions

#### 1. Create PreToolUse - The Linter Gate

```bash
cd .clinerules/hooks
mv PreToolUse PreToolUse.backup  # if exists
touch PreToolUse
code PreToolUse
```

#### 2. Write the Linter Integration

```bash
#!/usr/bin/env bash

# Quality Gate - Pre-execution Linting
# Prevents low-quality code from being written

set -euo pipefail

input=$(cat)
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

# Only check file writes
if [[ "$tool_name" != "write_to_file" ]]; then
    echo '{"cancel": false, "contextModification": ""}'; exit 0
fi

file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
file_content=$(echo "$input" | jq -r '.preToolUse.parameters.content')

# Create temp file for linting
temp_file="/tmp/cline-lint-$$.tmp"
echo "$file_content" > "$temp_file"

# Track if we should block
should_block=false
error_message=""

# ═══════════════════════════════════════════════════════════
# TYPESCRIPT / JAVASCRIPT LINTING
# ═══════════════════════════════════════════════════════════

if [[ "$file_path" =~ \.(ts|tsx|js|jsx)$ ]]; then
    # Copy with correct extension for linter
    lint_file="/tmp/cline-lint-$$${file_path##*.}"
    cp "$temp_file" "$lint_file"
    
    # Run ESLint if available
    if command -v eslint &> /dev/null; then
        echo "Running ESLint on ${file_path}..." >&2
        
        lint_output=$(eslint "$lint_file" --format compact 2>&1 || true)
        
        # Check if there are errors (not just warnings)
        if echo "$lint_output" | grep -q "error"; then
            should_block=true
            
            # Extract error messages
            errors=$(echo "$lint_output" | grep "error" | head -10)
            
            error_message="❌ ESLint errors detected in $file_path:

$errors

Please fix these errors before saving. You can:
• Address the errors manually
• Run: eslint --fix $file_path
• Disable specific rules if they're not applicable

Linting ensures code quality and prevents bugs."
        elif echo "$lint_output" | grep -q "warning"; then
            # Warnings don't block but inject context
            warnings=$(echo "$lint_output" | grep "warning" | wc -l)
            echo '{"cancel": false, "contextModification": "CODE_QUALITY: '"$file_path"' has '"$warnings"' ESLint warnings. Consider addressing them for better code quality."}' 
            rm -f "$temp_file" "$lint_file"
            exit 0
        fi
    fi
    
    # Run Prettier check if available
    if command -v prettier &> /dev/null && [ "$should_block" = false ]; then
        echo "Checking formatting with Prettier..." >&2
        
        if ! prettier --check "$lint_file" &> /dev/null; then
            # Format issues - warning only
            echo '{"cancel": false, "contextModification": "CODE_QUALITY: '"$file_path"' has formatting issues. Consider running: prettier --write '"$file_path"'"}' 
            rm -f "$temp_file" "$lint_file"
            exit 0
        fi
    fi
    
    rm -f "$lint_file"
fi

# ═══════════════════════════════════════════════════════════
# PYTHON LINTING
# ═══════════════════════════════════════════════════════════

if [[ "$file_path" =~ \.py$ ]]; then
    lint_file="/tmp/cline-lint-$$.py"
    cp "$temp_file" "$lint_file"
    
    # Run flake8 if available
    if command -v flake8 &> /dev/null; then
        echo "Running flake8 on ${file_path}..." >&2
        
        lint_output=$(flake8 "$lint_file" --max-line-length=100 2>&1 || true)
        
        if [ -n "$lint_output" ]; then
            # Count errors
            error_count=$(echo "$lint_output" | wc -l)
            
            if [ "$error_count" -gt 10 ]; then
                should_block=true
                error_message="❌ Flake8 found $error_count style issues in $file_path:

$(echo "$lint_output" | head -15)

... and more. Please fix these issues:
• Follow PEP 8 style guidelines
• Run: autopep8 --in-place $file_path
• Or: black $file_path

Clean code is maintainable code."
            else
                # Few errors - inject context but don't block
                echo '{"cancel": false, "contextModification": "CODE_QUALITY: '"$file_path"' has '"$error_count"' flake8 issues. Consider fixing: '"${lint_output:0:200}"'..."}' 
                rm -f "$temp_file" "$lint_file"
                exit 0
            fi
        fi
    fi
    
    rm -f "$lint_file"
fi

# ═══════════════════════════════════════════════════════════
# RUST LINTING
# ═══════════════════════════════════════════════════════════

if [[ "$file_path" =~ \.rs$ ]]; then
    # Rust files need project context for clippy
    # We'll do a simpler check or skip for now
    echo "Rust linting requires project context - consider running 'cargo clippy' manually" >&2
fi

# ═══════════════════════════════════════════════════════════
# GENERAL CODE CHECKS
# ═══════════════════════════════════════════════════════════

# Check for common issues
if grep -q "console.log" "$temp_file" && [[ "$file_path" =~ \.(ts|tsx)$ ]]; then
    # console.log in TypeScript - warn but don't block
    echo '{"cancel": false, "contextModification": "CODE_QUALITY: File contains console.log statements. Consider removing or using a proper logging library for production code."}' 
    rm -f "$temp_file"
    exit 0
fi

# Check for TODO/FIXME in new code
if grep -qi "TODO\|FIXME" "$temp_file"; then
    todo_count=$(grep -ci "TODO\|FIXME" "$temp_file")
    echo '{"cancel": false, "contextModification": "CODE_QUALITY: File contains '"$todo_count"' TODO/FIXME comments. Consider addressing these before finalizing."}' 
    rm -f "$temp_file"
    exit 0
fi

# ═══════════════════════════════════════════════════════════
# CLEANUP AND RETURN
# ═══════════════════════════════════════════════════════════

rm -f "$temp_file"

if [ "$should_block" = true ]; then
    echo "{
      \"cancel\": true,
      \"contextModification\": \"\",
      \"errorMessage\": $(echo "$error_message" | jq -Rs .)
    }"
else
    echo '{"cancel": false, "contextModification": ""}'
fi
```

#### 3. Create PostToolUse - Quality Metrics Tracker

```bash
touch PostToolUse
code PostToolUse
```

```bash
#!/usr/bin/env bash

# Quality Metrics Tracker
# Records code quality metrics after file operations

input=$(cat)
tool_name=$(echo "$input" | jq -r '.postToolUse.toolName')
success=$(echo "$input" | jq -r '.postToolUse.success')

if [[ "$tool_name" != "write_to_file" ]] || [[ "$success" != "true" ]]; then
    echo '{"cancel": false, "contextModification": ""}'; exit 0
fi

file_path=$(echo "$input" | jq -r '.postToolUse.parameters.path')

# Skip non-code files
if [[ ! "$file_path" =~ \.(ts|tsx|js|jsx|py|rs|go)$ ]]; then
    echo '{"cancel": false, "contextModification": ""}'; exit 0
fi

# ═══════════════════════════════════════════════════════════
# COLLECT METRICS
# ═══════════════════════════════════════════════════════════

if [ -f "$file_path" ]; then
    # Lines of code
    loc=$(wc -l < "$file_path" 2>/dev/null || echo "0")
    
    # Complexity indicators
    if_count=$(grep -c "if " "$file_path" 2>/dev/null || echo "0")
    function_count=$(grep -cE "function |def |fn " "$file_path" 2>/dev/null || echo "0")
    
    # Comments
    comment_count=$(grep -cE "^[[:space:]]*(//|#|/\*)" "$file_path" 2>/dev/null || echo "0")
    
    # Calculate comment ratio
    if [ "$loc" -gt 0 ]; then
        comment_ratio=$((comment_count * 100 / loc))
    else
        comment_ratio=0
    fi
    
    # Log metrics
    metrics_log="/tmp/cline-code-metrics.jsonl"
    
    echo "{
      \"timestamp\": \"$(date -Iseconds)\",
      \"file\": \"$file_path\",
      \"loc\": $loc,
      \"functions\": $function_count,
      \"conditionals\": $if_count,
      \"comments\": $comment_count,
      \"commentRatio\": $comment_ratio
    }" >> "$metrics_log"
    
    # ═══════════════════════════════════════════════════════════
    # PROVIDE GUIDANCE
    # ═══════════════════════════════════════════════════════════
    
    context=""
    
    # Large file warning
    if [ "$loc" -gt 500 ]; then
        context="CODE_QUALITY: ⚠️ $file_path has $loc lines. Consider breaking it into smaller, more focused modules (target: <300 lines per file)."
    fi
    
    # Low comment ratio
    if [ "$comment_ratio" -lt 5 ] && [ "$loc" -gt 100 ]; then
        if [ -n "$context" ]; then
            context="$context "
        fi
        context="${context}CODE_QUALITY: Low comment ratio (${comment_ratio}%) in $file_path. Consider adding documentation for complex logic."
    fi
    
    # High complexity
    complexity_score=$((if_count + function_count))
    if [ "$complexity_score" -gt 50 ]; then
        if [ -n "$context" ]; then
            context="$context "
        fi
        context="${context}CODE_QUALITY: High complexity detected in $file_path ($function_count functions, $if_count conditionals). Consider refactoring for maintainability."
    fi
    
    echo "{
      \"cancel\": false,
      \"contextModification\": \"$context\"
    }"
else
    echo '{"cancel": false, "contextModification": ""}'
fi
```

#### 4. Make Both Executable

```bash
chmod +x PreToolUse PostToolUse
```

#### 5. Test the Quality Gate

Create a test file with linting errors:

```bash
mkdir -p ~/test-quality-gate
cd ~/test-quality-gate

# Create a TypeScript file with errors
cat > bad-code.ts << 'EOF'
function test() {
var x = 1
var y = 2
if(x==y){
console.log("test")
}
}
EOF

# Test with PreToolUse
cat << 'EOF' | ~/.clinerules/hooks/PreToolUse
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
      "path": "/tmp/bad-code.ts",
      "content": "function test() {\nvar x = 1\nvar y = 2\nif(x==y){\nconsole.log(\"test\")\n}\n}"
    }
  }
}
EOF
```

#### 6. View Quality Metrics

```bash
# View metrics log
cat /tmp/cline-code-metrics.jsonl | jq '.'

# Calculate average LOC
cat /tmp/cline-code-metrics.jsonl | jq -s 'map(.loc) | add / length'

# Find files with low comment ratios
cat /tmp/cline-code-metrics.jsonl | jq -s '.[] | select(.commentRatio < 10) | {file, commentRatio}'
```

### Key Takeaways

✅ **Run external tools from hooks** - Shell out to linters  
✅ **Temp files for linting** - Don't modify actual files  
✅ **Error vs warning distinction** - Block on errors, warn on warnings  
✅ **Multi-hook workflows** - PreToolUse blocks, PostToolUse learns  
✅ **Metrics over time** - JSONL perfect for tracking trends

---

## 🔔 Exercise 4.2: "The Integration Hub" - External Service Notifications

**Duration:** 50 minutes  
**Hook Type:** PostToolUse (Python)  
**Difficulty:** ⭐⭐⭐ Advanced

### Scenario

Your team wants visibility into AI-generated changes. Build a hook that sends notifications to Slack/Discord when important operations complete, keeping the team informed.

### What You'll Build

A Python-based PostToolUse hook that:
- Sends Slack notifications
- Sends Discord messages
- Filters important events
- Handles API errors gracefully
- Configures via environment variables

### Real-World Applications

- CI/CD notifications
- Team awareness
- Audit logging
- Integration monitoring
- Change tracking

### Prerequisites

```bash
# Python should be available
python3 --version

# No additional packages needed (using only stdlib)
```

### Step-by-Step Instructions

#### 1. Set Up Webhook URLs

**For Slack:**
1. Go to https://api.slack.com/messaging/webhooks
2. Create an incoming webhook
3. Copy the webhook URL

**For Discord:**
1. Open Discord server settings → Integrations
2. Create webhook
3. Copy the webhook URL

```bash
# Set environment variables (add to ~/.bashrc or ~/.zshrc)
export CLINE_SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
export CLINE_DISCORD_WEBHOOK="https://discord.com/api/webhooks/YOUR/WEBHOOK"
```

#### 2. Create the Integration Hub Hook

```bash
cd .clinerules/hooks
mv PostToolUse PostToolUse.metrics  # backup previous
touch PostToolUse
chmod +x PostToolUse
code PostToolUse
```

#### 3. Write the Python Integration

```python
#!/usr/bin/env python3
"""
Integration Hub Hook
Sends notifications to external services for important events
"""

import json
import sys
import os
import urllib.request
import urllib.error
from datetime import datetime

def send_slack_notification(message):
    """Send message to Slack via webhook"""
    webhook_url = os.getenv('CLINE_SLACK_WEBHOOK')
    
    if not webhook_url:
        return False
    
    payload = json.dumps({"text": message}).encode('utf-8')
    
    req = urllib.request.Request(
        webhook_url,
        data=payload,
        headers={'Content-Type': 'application/json'}
    )
    
    try:
        with urllib.request.urlopen(req, timeout=5) as response:
            return response.status == 200
    except (urllib.error.URLError, TimeoutError) as e:
        print(f"Slack notification failed: {e}", file=sys.stderr)
        return False


def send_discord_notification(message, embed=None):
    """Send message to Discord via webhook"""
    webhook_url = os.getenv('CLINE_DISCORD_WEBHOOK')
    
    if not webhook_url:
        return False
    
    payload = {"content": message}
    if embed:
        payload["embeds"] = [embed]
    
    data = json.dumps(payload).encode('utf-8')
    
    req = urllib.request.Request(
        webhook_url,
        data=data,
        headers={'Content-Type': 'application/json'}
    )
    
    try:
        with urllib.request.urlopen(req, timeout=5) as response:
            return response.status in (200, 204)
    except (urllib.error.URLError, TimeoutError) as e:
        print(f"Discord notification failed: {e}", file=sys.stderr)
        return False


def format_timestamp():
    """Format current timestamp for display"""
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")


def main():
    # Read JSON input from stdin
    try:
        input_data = json.loads(sys.stdin.read())
    except json.JSONDecodeError as e:
        print(f"Failed to parse JSON: {e}", file=sys.stderr)
        print(json.dumps({"cancel": False, "contextModification": ""}))
        return
    
    tool_name = input_data.get('postToolUse', {}).get('toolName')
    success = input_data.get('postToolUse', {}).get('success')
    task_id = input_data.get('taskId', 'unknown')
    
    # Only process successful operations
    if not success:
        print(json.dumps({"cancel": False, "contextModification": ""}))
        return
    
    # Short task ID for display
    short_task_id = task_id[:8] if len(task_id) > 8 else task_id
    
    # ════════════════════════════════════════════════════════
    # COMMAND EXECUTION EVENTS
    # ════════════════════════════════════════════════════════
    
    if tool_name == "execute_command":
        command = input_data.get('postToolUse', {}).get('parameters', {}).get('command', '')
        result = input_data.get('postToolUse', {}).get('result', '')
        
        # Test executions
        if any(test_cmd in command.lower() for test_cmd in ['test', 'jest', 'pytest', 'cargo test']):
            # Parse test results
            if any(word in result.lower() for word in ['passed', 'ok', '✓', 'success']):
                message = f"✅ Tests passed in task `{short_task_id}`"
                send_slack_notification(message)
                
                # Discord with embed
                embed = {
                    "title": "Tests Passed",
                    "description": f"```\n{command}\n```",
                    "color": 3066993,  # Green
                    "timestamp": datetime.utcnow().isoformat()
                }
                send_discord_notification(message, embed)
                
            elif any(word in result.lower() for word in ['failed', 'error', 'fail']):
                message = f"❌ Tests failed in task `{short_task_id}`\nCommand: `{command}`"
                send_slack_notification(message)
                
                embed = {
                    "title": "Tests Failed",
                    "description": f"```\n{command}\n```",
                    "color": 15158332,  # Red
                    "timestamp": datetime.utcnow().isoformat()
                }
                send_discord_notification(message, embed)
        
        # Deployment commands
        if any(keyword in command.lower() for keyword in ['deploy', 'publish', 'release']):
            message = f"🚀 Deployment triggered: `{command}`\nTask: `{short_task_id}`"
            send_slack_notification(message)
            send_discord_notification(message)
        
        # Installation commands
        if any(keyword in command.lower() for keyword in ['npm install', 'pip install', 'cargo add']):
            # Extract package name
            parts = command.split()
            if len(parts) > 2:
                package = parts[2]
                message = f"📦 Dependency added: `{package}`"
                send_slack_notification(message)
    
    # ════════════════════════════════════════════════════════
    # FILE OPERATIONS
    # ════════════════════════════════════════════════════════
    
    if tool_name == "write_to_file":
        file_path = input_data.get('postToolUse', {}).get('parameters', {}).get('path', '')
        
        # Migration or schema files
        if any(keyword in file_path.lower() for keyword in ['migration', 'schema', '.sql']):
            message = f"🗃️ Database change detected: `{os.path.basename(file_path)}`\nTask: `{short_task_id}`"
            send_slack_notification(message)
        
        # Configuration files
        if any(keyword in file_path.lower() for keyword in ['.env', 'config', 'settings']):
            message = f"⚙️ Configuration modified: `{os.path.basename(file_path)}`\nTask: `{short_task_id}`"
            send_slack_notification(message)
        
        # Docker files
        if 'dockerfile' in file_path.lower() or 'docker-compose' in file_path.lower():
            message = f"🐳 Docker configuration updated: `{os.path.basename(file_path)}`"
            send_slack_notification(message)
    
    # ════════════════════════════════════════════════════════
    # RETURN RESPONSE
    # ════════════════════════════════════════════════════════
    
    response = {
        "cancel": False,
        "contextModification": ""
    }
    
    print(json.dumps(response))


if __name__ == "__main__":
    main()
```

#### 4. Test the Integration

```bash
# Test Slack notification
cat << 'EOF' | ./PostToolUse
{
  "clineVersion": "1.0.0",
  "hookName": "PostToolUse",
  "timestamp": "2025-11-06T10:30:00Z",
  "taskId": "test-123456",
  "workspaceRoots": ["/workspace"],
  "userId": "user-001",
  "postToolUse": {
    "toolName": "execute_command",
    "parameters": {
      "command": "npm test"
    },
    "result": "Tests: 15 passed, 15 total",
    "success": true,
    "executionTimeMs": 3500
  }
}
EOF

# Check Slack/Discord for the notification
```

### Enhancement: Rich Discord Embeds

Add more sophisticated Discord formatting:

```python
def create_rich_embed(title, description, color="blue", fields=None):
    """Create a rich Discord embed"""
    colors = {
        "green": 3066993,
        "red": 15158332,
        "blue": 3447003,
        "yellow": 16776960,
        "purple": 10181046
    }
    
    embed = {
        "title": title,
        "description": description,
        "color": colors.get(color, colors["blue"]),
        "timestamp": datetime.utcnow().isoformat(),
        "footer": {
            "text": "Cline AI Assistant"
        }
    }
    
    if fields:
        embed["fields"] = fields
    
    return embed

# Usage:
embed = create_rich_embed(
    title="Test Results",
    description="All tests passed successfully",
    color="green",
    fields=[
        {"name": "Duration", "value": "3.5s", "inline": True},
        {"name": "Tests", "value": "15", "inline": True}
    ]
)
```

### Key Takeaways

✅ **Python for complex logic** - Better than bash for APIs  
✅ **Environment variables for config** - Keep secrets out of code  
✅ **Fail gracefully** - Don't break hooks on network errors  
✅ **Filter important events** - Don't spam notifications  
✅ **Use urllib (stdlib)** - No external dependencies needed

---

## 💾 Exercise 4.3: "State Machine" - Persistent Hook State

**Duration:** 40 minutes  
**Hook Types:** TaskStart + PostToolUse + TaskCancel  
**Difficulty:** ⭐⭐⭐ Advanced

### Scenario

You want to track project state across multiple tasks and sessions. Build a state management system using SQLite that persists between Cline executions.

### What You'll Build

A complete state management system:
- SQLite database for persistence
- Task initialization and tracking
- Operation counting
- State cleanup
- Historical analysis

### Real-World Applications

- Project analytics
- Resource tracking
- Compliance logging
- Audit trails
- Performance baselines

### Step-by-Step Instructions

#### 1. Create the State Manager Library

```bash
touch /tmp/cline-state-manager.py
chmod +x /tmp/cline-state-manager.py
code /tmp/cline-state-manager.py
```

```python
#!/usr/bin/env python3
"""
Cline State Manager
Persistent state tracking across hook executions
"""

import sqlite3
import json
import sys
from pathlib import Path
from datetime import datetime

DB_PATH = Path.home() / ".cline-hooks-state.db"


def init_db():
    """Initialize the database schema"""
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    
    # Tasks table
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS tasks (
            task_id TEXT PRIMARY KEY,
            started_at TEXT,
            last_updated TEXT,
            file_writes INTEGER DEFAULT 0,
            file_reads INTEGER DEFAULT 0,
            commands_run INTEGER DEFAULT 0,
            total_time_ms INTEGER DEFAULT 0,
            status TEXT DEFAULT 'active'
        )
    ''')
    
    # Operations table
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS operations (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            task_id TEXT,
            timestamp TEXT,
            tool_name TEXT,
            execution_ms INTEGER,
            success BOOLEAN,
            FOREIGN KEY (task_id) REFERENCES tasks (task_id)
        )
    ''')
    
    # Metrics table
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS metrics (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            task_id TEXT,
            timestamp TEXT,
            metric_name TEXT,
            metric_value REAL,
            FOREIGN KEY (task_id) REFERENCES tasks (task_id)
        )
    ''')
    
    conn.commit()
    conn.close()


def create_task(task_id, initial_task=""):
    """Create a new task record"""
    init_db()
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    
    cursor.execute('''
        INSERT OR IGNORE INTO tasks (task_id, started_at, last_updated, status)
        VALUES (?, ?, ?, 'active')
    ''', (task_id, datetime.now().isoformat(), datetime.now().isoformat()))
    
    conn.commit()
    conn.close()


def update_task(task_id, **kwargs):
    """Update task statistics"""
    init_db()
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    
    # Ensure task exists
    cursor.execute('SELECT task_id FROM tasks WHERE task_id = ?', (task_id,))
    if not cursor.fetchone():
        create_task(task_id)
    
    # Update last_updated
    cursor.execute('''
        UPDATE tasks SET last_updated = ? WHERE task_id = ?
    ''', (datetime.now().isoformat(), task_id))
    
    # Update specific fields
    for key, value in kwargs.items():
        if key in ['file_writes', 'file_reads', 'commands_run', 'total_time_ms']:
            # Increment counters
            cursor.execute(f'''
                UPDATE tasks SET {key} = {key} + ? WHERE task_id = ?
            ''', (value, task_id))
        elif key == 'status':
            # Direct update
            cursor.execute('''
                UPDATE tasks SET status = ? WHERE task_id = ?
            ''', (value, task_id))
    
    conn.commit()
    conn.close()


def log_operation(task_id, tool_name, execution_ms, success):
    """Log an operation to the operations table"""
    init_db()
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    
    cursor.execute('''
        INSERT INTO operations (task_id, timestamp, tool_name, execution_ms, success)
        VALUES (?, ?, ?, ?, ?)
    ''', (task_id, datetime.now().isoformat(), tool_name, execution_ms, success))
    
    conn.commit()
    conn.close()


def get_task_state(task_id):
    """Retrieve task state"""
    init_db()
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    
    cursor.execute('SELECT * FROM tasks WHERE task_id = ?', (task_id,))
    row = cursor.fetchone()
    conn.close()
    
    if row:
        return {
            "task_id": row[0],
            "started_at": row[1],
            "last_updated": row[2],
            "file_writes": row[3],
            "file_reads": row[4],
            "commands_run": row[5],
            "total_time_ms": row[6],
            "status": row[7]
        }
    return None


def get_task_stats():
    """Get overall statistics"""
    init_db()
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    
    cursor.execute('''
        SELECT 
            COUNT(*) as total_tasks,
            SUM(file_writes) as total_writes,
            SUM(commands_run) as total_commands,
            AVG(total_time_ms) as avg_time
        FROM tasks
    ''')
    
    row = cursor.fetchone()
    conn.close()
    
    return {
        "total_tasks": row[0] or 0,
        "total_writes": row[1] or 0,
        "total_commands": row[2] or 0,
        "avg_time_ms": row[3] or 0
    }


def main():
    if len(sys.argv) < 3:
        print("Usage: state-manager.py <action> <task_id> [args...]")
        sys.exit(1)
    
    action = sys.argv[1]
    task_id = sys.argv[2]
    
    if action == "create":
        create_task(task_id)
        print(f"Created task: {task_id}")
    
    elif action == "file_write":
        update_task(task_id, file_writes=1)
        print(f"Incremented file_writes for {task_id}")
    
    elif action == "file_read":
        update_task(task_id, file_reads=1)
        print(f"Incremented file_reads for {task_id}")
    
    elif action == "command":
        exec_time = int(sys.argv[3]) if len(sys.argv) > 3 else 0
        update_task(task_id, commands_run=1, total_time_ms=exec_time)
        print(f"Logged command for {task_id}")
    
    elif action == "complete":
        update_task(task_id, status='completed')
        print(f"Marked {task_id} as completed")
    
    elif action == "cancel":
        update_task(task_id, status='cancelled')
        print(f"Marked {task_id} as cancelled")
    
    elif action == "get":
        state = get_task_state(task_id)
        if state:
            print(json.dumps(state, indent=2))
        else:
            print(f"No state found for {task_id}")
    
    elif action == "stats":
        stats = get_task_stats()
        print(json.dumps(stats, indent=2))
    
    else:
        print(f"Unknown action: {action}")
        sys.exit(1)


if __name__ == "__main__":
    main()
```

#### 2. Create TaskStart Hook

```bash
cd .clinerules/hooks
touch TaskStart
chmod +x TaskStart
```

```bash
#!/usr/bin/env bash

# TaskStart - Initialize task state

input=$(cat)
task_id=$(echo "$input" | jq -r '.taskId')

# Initialize task in database
/tmp/cline-state-manager.py create "$task_id" 2>&1 | while read line; do
    echo "$line" >&2
done

echo '{"cancel": false, "contextModification": ""}'
```

#### 3. Create PostToolUse Hook

```bash
touch PostToolUse
chmod +x PostToolUse
```

```bash
#!/usr/bin/env bash

# PostToolUse - Track operations

input=$(cat)
task_id=$(echo "$input" | jq -r '.taskId')
tool_name=$(echo "$input" | jq -r '.postToolUse.toolName')
exec_time=$(echo "$input" | jq -r '.postToolUse.executionTimeMs')
success=$(echo "$input" | jq -r '.postToolUse.success')

if [ "$success" != "true" ]; then
    echo '{"cancel": false, "contextModification": ""}'; exit 0
fi

# Update state based on tool
case "$tool_name" in
    "write_to_file")
        /tmp/cline-state-manager.py file_write "$task_id" 2>&1 >&2
        ;;
    "read_file")
        /tmp/cline-state-manager.py file_read "$task_id" 2>&1 >&2
        ;;
    "execute_command")
        /tmp/cline-state-manager.py command "$task_id" "$exec_time" 2>&1 >&2
        ;;
esac

# Get current state
state=$(/tmp/cline-state-manager.py get "$task_id" 2>/dev/null)

# Extract metrics
file_writes=$(echo "$state" | jq -r '.file_writes // 0')
commands=$(echo "$state" | jq -r '.commands_run // 0')

# Provide periodic feedback
context=""

if [ "$file_writes" -gt 20 ]; then
    context="PROJECT_STATUS: High activity detected - $file_writes files modified. Consider reviewing changes and breaking into smaller tasks if needed."
elif [ "$commands" -gt 15 ]; then
    context="PROJECT_STATUS: Many commands executed ($commands). Ensure all operations are necessary and efficient."
fi

echo "{
  \"cancel\": false,
  \"contextModification\": \"$context\"
}"
```

#### 4. Test the State System

```bash
# Create a task
/tmp/cline-state-manager.py create test-task-001

# Log some operations
/tmp/cline-state-manager.py file_write test-task-001
/tmp/cline-state-manager.py file_write test-task-001
/tmp/cline-state-manager.py command test-task-001 1500

# Check state
/tmp/cline-state-manager.py get test-task-001

# Get overall stats
/tmp/cline-state-manager.py stats
```

#### 5. Query the Database

```bash
# Open the database
sqlite3 ~/.cline-hooks-state.db

# Run queries
.mode column
.headers on

SELECT * FROM tasks;
SELECT task_id, COUNT(*) as ops FROM operations GROUP BY task_id;
SELECT tool_name, AVG(execution_ms) FROM operations GROUP BY tool_name;
```

### Key Takeaways

✅ **SQLite for persistence** - Perfect for local state  
✅ **Multiple tables** - Normalize data properly  
✅ **State manager library** - Reusable across hooks  
✅ **Task lifecycle tracking** - Create → Update → Complete  
✅ **Historical analysis** - Query for insights

---

## 🎓 Module 4 Review

### What You've Learned

- ✅ External tool integration (linters)
- ✅ HTTP requests and webhooks
- ✅ Python for complex hooks
- ✅ Multi-hook workflows
- ✅ Persistent state with SQLite
- ✅ Error handling and graceful degradation

### Practical Skills

You can now:
- Integrate any external tool
- Send notifications to services
- Build stateful applications
- Use Python for advanced logic
- Create production-ready hooks
- Handle errors gracefully

## 🎯 Module 4 Completion Checklist

- [ ] Created Quality Gate with linting
- [ ] Integrated external linters (ESLint/flake8)
- [ ] Built Integration Hub with webhooks
- [ ] Tested Slack/Discord notifications
- [ ] Implemented State Machine with SQLite
- [ ] Can query and analyze state data
- [ ] Understand when to use Python vs Bash
- [ ] Passed knowledge check

## 🚀 Next Steps

You've mastered advanced integrations. Next, learn production patterns including error handling, team collaboration, and deployment strategies:

**[Module 5: Production Patterns →](module-05-production.md)**

Or return to the [Course Overview](README.md)
