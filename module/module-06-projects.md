# Module 6: Real-World Projects

**Duration:** 4-6 hours | **Difficulty:** Capstone

## 🎯 Learning Objectives

By the end of this module, you will:

- ✅ Build complete, production-ready hook systems
- ✅ Integrate multiple hooks into cohesive workflows
- ✅ Apply all techniques learned in previous modules
- ✅ Create documentation and tests
- ✅ Deploy real solutions to actual problems
- ✅ Demonstrate mastery of hook development

## 📖 Overview

This module contains two comprehensive capstone projects that bring together everything you've learned. Choose one or complete both to demonstrate your mastery of Cline hooks.

Each project includes:
- Complete requirements specification
- Architecture guidance
- Step-by-step implementation
- Testing strategies
- Documentation templates
- Extension ideas

## 🏗️ Capstone Project 1: Smart Documentation System

**Duration:** 2-3 hours  
**Complexity:** Multi-hook workflow  
**Skills:** File tracking, API detection, documentation validation

### Project Overview

Build an intelligent system that:
1. Detects when code changes affect public APIs
2. Checks if documentation was updated
3. Reminds developers to update docs
4. Auto-generates API diff summaries
5. Tracks documentation coverage

### Business Value

- Keeps documentation in sync with code
- Reduces onboarding time
- Prevents stale documentation
- Improves API usability
- Automates tedious tracking

### Requirements

**Must Have:**
- Track exported functions/classes
- Detect API changes
- Check for doc updates
- Generate reminders
- Log documentation status

**Nice to Have:**
- Auto-generate stub docs
- Measure doc coverage %
- Create PRchecklistsLint doc quality

### Architecture

```
TaskStart
    ↓
  Initialize tracking
  Scan existing API surface
    ↓
PostToolUse (write_to_file)
    ↓
  Detect if file exports APIs
  Compare with previous state
  Check for doc updates
    ↓
  If API changed and no docs
    ↓
  Inject reminder context
    ↓
UserPromptSubmit
    ↓
  Detect doc update keywords
  Mark docs as addressed
```

### Implementation

#### Step 1: Create API Surface Scanner

```bash
cd .clinerules/hooks
touch TaskStart
chmod +x TaskStart
```

```python
#!/usr/bin/env python3
"""
Smart Documentation System - Task Initialization
Scans project for API surface
"""

import json
import sys
import os
import re
from pathlib import Path


def scan_typescript_exports(file_path):
    """Extract exported items from TypeScript file"""
    exports = []
    
    try:
        with open(file_path, 'r') as f:
            content = f.read()
        
        # Match export function/class
        patterns = [
            r'export\s+(function|class|interface|type|const|let|var)\s+(\w+)',
            r'export\s+{\s*([^}]+)\s*}',
            r'export\s+default\s+(function|class)?\s*(\w+)?'
        ]
        
        for pattern in patterns:
            matches = re.finditer(pattern, content)
            for match in matches:
                if len(match.groups()) >= 2:
                    exports.append(match.group(2))
                elif match.group(1):
                    exports.append(match.group(1))
        
    except Exception as e:
        print(f"Error scanning {file_path}: {e}", file=sys.stderr)
    
    return exports


def scan_python_exports(file_path):
    """Extract exported items from Python file"""
    exports = []
    
    try:
        with open(file_path, 'r') as f:
            content = f.read()
        
        # Match class and function definitions
        patterns = [
            r'^class\s+(\w+)',
            r'^def\s+(\w+)',
            r'^async\s+def\s+(\w+)'
        ]
        
        for pattern in patterns:
            matches = re.finditer(pattern, content, re.MULTILINE)
            for match in matches:
                name = match.group(1)
                # Skip private (starting with _)
                if not name.startswith('_'):
                    exports.append(name)
        
        # Check for __all__
        all_match = re.search(r'__all__\s*=\s*\[([^\]]+)\]', content)
        if all_match:
            items = all_match.group(1)
            for item in re.findall(r'[\'"](\w+)[\'"]', items):
                if item not in exports:
                    exports.append(item)
    
    except Exception as e:
        print(f"Error scanning {file_path}: {e}", file=sys.stderr)
    
    return exports


def scan_workspace(workspace_root):
    """Scan entire workspace for API surface"""
    api_surface = {}
    
    workspace_path = Path(workspace_root)
    
    # Scan TypeScript/JavaScript files
    for pattern in ['**/*.ts', '**/*.tsx', '**/*.js', '**/*.jsx']:
        for file_path in workspace_path.glob(pattern):
            # Skip node_modules, dist, build
            if any(part in file_path.parts for part in ['node_modules', 'dist', 'build', '.next']):
                continue
            
            exports = scan_typescript_exports(file_path)
            if exports:
                api_surface[str(file_path.relative_to(workspace_path))] = exports
    
    # Scan Python files
    for file_path in workspace_path.glob('**/*.py'):
        if any(part in file_path.parts for part in ['venv', '__pycache__', '.venv']):
            continue
        
        exports = scan_python_exports(file_path)
        if exports:
            api_surface[str(file_path.relative_to(workspace_path))] = exports
    
    return api_surface


def find_documentation(workspace_root):
    """Find documentation files in project"""
    docs = []
    workspace_path = Path(workspace_root)
    
    # Common doc locations
    doc_patterns = [
        'README.md',
        'docs/**/*.md',
        'documentation/**/*.md',
        'API.md',
        'CONTRIBUTING.md'
    ]
    
    for pattern in doc_patterns:
        for file_path in workspace_path.glob(pattern):
            docs.append(str(file_path.relative_to(workspace_path)))
    
    return docs


def main():
    try:
        input_data = json.loads(sys.stdin.read())
    except json.JSONDecodeError:
        print(json.dumps({"cancel": False, "contextModification": ""}))
        return
    
    workspace = input_data.get('workspaceRoots', [None])[0]
    task_id = input_data.get('taskId', 'unknown')
    
    if not workspace or not os.path.exists(workspace):
        print(json.dumps({"cancel": False, "contextModification": ""}))
        return
    
    # Scan API surface
    print("Scanning workspace for APIs...", file=sys.stderr)
    api_surface = scan_workspace(workspace)
    
    # Find documentation
    docs = find_documentation(workspace)
    
    # Save initial state
    state_file = Path('/tmp') / f'cline-doc-state-{task_id}.json'
    state = {
        'taskId': task_id,
        'workspace': workspace,
        'initialAPISurface': api_surface,
        'documentationFiles': docs,
        'changedAPIs': [],
        'docsUpdated': False
    }
    
    with open(state_file, 'w') as f:
        json.dump(state, f, indent=2)
    
    print(f"Found {len(api_surface)} files with APIs", file=sys.stderr)
    print(f"Found {len(docs)} documentation files", file=sys.stderr)
    
    # Inject initial context
    context = f"DOCUMENTATION_TRACKING: Monitoring API changes. Found {len(api_surface)} files with exported APIs and {len(docs)} documentation files. Please update relevant documentation when modifying public APIs."
    
    response = {
        "cancel": False,
        "contextModification": context
    }
    
    print(json.dumps(response))


if __name__ == "__main__":
    main()
```

#### Step 2: Create API Change Detector

```bash
touch PostToolUse
chmod +x PostToolUse
```

```python
#!/usr/bin/env python3
"""
Smart Documentation System - Change Detection
Detects API changes and checks for doc updates
"""

import json
import sys
import os
import re
from pathlib import Path


def load_state(task_id):
    """Load task state"""
    state_file = Path('/tmp') / f'cline-doc-state-{task_id}.json'
    
    if not state_file.exists():
        return None
    
    try:
        with open(state_file, 'r') as f:
            return json.load(f)
    except:
        return None


def save_state(task_id, state):
    """Save task state"""
    state_file = Path('/tmp') / f'cline-doc-state-{task_id}.json'
    
    with open(state_file, 'w') as f:
        json.dump(state, f, indent=2)


def scan_file_exports(file_path):
    """Scan file for exports (simplified)"""
    if not os.path.exists(file_path):
        return []
    
    exports = []
    
    try:
        with open(file_path, 'r') as f:
            content = f.read()
        
        # Simple export detection
        if file_path.endswith('.py'):
            for match in re.finditer(r'^(class|def|async def)\s+(\w+)', content, re.MULTILINE):
                name = match.group(2)
                if not name.startswith('_'):
                    exports.append(name)
        else:
            for match in re.finditer(r'export\s+(?:function|class|interface|type|const)\s+(\w+)', content):
                exports.append(match.group(1))
    
    except Exception as e:
        print(f"Error scanning file: {e}", file=sys.stderr)
    
    return exports


def is_documentation_file(file_path):
    """Check if file is documentation"""
    doc_indicators = [
        '.md' in file_path.lower(),
        'readme' in file_path.lower(),
        'docs/' in file_path.lower(),
        'documentation/' in file_path.lower()
    ]
    return any(doc_indicators)


def main():
    try:
        input_data = json.loads(sys.stdin.read())
    except json.JSONDecodeError:
        print(json.dumps({"cancel": False, "contextModification": ""}))
        return
    
    tool_name = input_data.get('postToolUse', {}).get('toolName')
    success = input_data.get('postToolUse', {}).get('success')
    task_id = input_data.get('taskId', 'unknown')
    
    if tool_name != "write_to_file" or not success:
        print(json.dumps({"cancel": False, "contextModification": ""}))
        return
    
    # Load state
    state = load_state(task_id)
    if not state:
        print(json.dumps({"cancel": False, "contextModification": ""}))
        return
    
    file_path = input_data.get('postToolUse', {}).get('parameters', {}).get('path', '')
    
    # Check if documentation was updated
    if is_documentation_file(file_path):
        state['docsUpdated'] = True
        state['lastDocUpdate'] = file_path
        save_state(task_id, state)
        
        context = "DOCUMENTATION: ✅ Documentation updated. Thank you for keeping docs in sync!"
        
        print(json.dumps({
            "cancel": False,
            "contextModification": context
        }))
        return
    
    # Check if file has exports (is an API file)
    workspace = state['workspace']
    full_path = os.path.join(workspace, file_path) if not os.path.isabs(file_path) else file_path
    
    current_exports = scan_file_exports(full_path)
    
    if not current_exports:
        # Not an API file
        print(json.dumps({"cancel": False, "contextModification": ""}))
        return
    
    # Compare with initial state
    relative_path = os.path.relpath(full_path, workspace) if workspace in full_path else file_path
    initial_exports = state.get('initialAPISurface', {}).get(relative_path, [])
    
    # Detect changes
    added = set(current_exports) - set(initial_exports)
    removed = set(initial_exports) - set(current_exports)
    
    if added or removed:
        # API changed!
        state['changedAPIs'].append({
            'file': relative_path,
            'added': list(added),
            'removed': list(removed)
        })
        save_state(task_id, state)
        
        # Check if docs were updated
        context = ""
        
        if not state.get('docsUpdated', False):
            change_summary = []
            if added:
                change_summary.append(f"Added: {', '.join(added)}")
            if removed:
                change_summary.append(f"Removed: {', '.join(removed)}")
            
            changes = " | ".join(change_summary)
            
            doc_files = state.get('documentationFiles', [])
            doc_list = ", ".join(doc_files[:3]) if doc_files else "README.md"
            
            context = f"""DOCUMENTATION_REMINDER: ⚠️ API changes detected in {relative_path}:
{changes}

Please update documentation to reflect these changes. Consider updating: {doc_list}

Documentation helps other developers understand and use the modified APIs."""
        else:
            context = "DOCUMENTATION: API changed and documentation was updated. Excellent work!"
        
        print(json.dumps({
            "cancel": False,
            "contextModification": context
        }))
        return
    
    # No changes detected
    print(json.dumps({"cancel": False, "contextModification": ""}))


if __name__ == "__main__":
    main()
```

#### Step 3: Test the System

```bash
# Initialize a test project
mkdir -p ~/test-doc-system/src
cd ~/test-doc-system

# Create an API file
cat > src/api.ts << 'EOF'
export function hello() {
    return "world";
}

export class User {
    name: string;
}
EOF

# Create docs
echo "# API Documentation" > README.md

# Test TaskStart
cat << 'EOF' | .clinerules/hooks/TaskStart
{
  "clineVersion": "1.0.0",
  "hookName": "TaskStart",
  "timestamp": "2025-11-06T10:30:00Z",
  "taskId": "test-doc-123",
  "workspaceRoots": ["/Users/yourusername/test-doc-system"],
  "userId": "user-001",
  "taskStart": {
    "taskMetadata": {
      "taskId": "test-doc-123",
      "ulid": "01H",
      "initialTask": "Add new feature"
    }
  }
}
EOF

# Now test API change detection
# (Simulate writing updated API file)
```

### Extension Ideas

1. **Auto-generate doc stubs**
2. **Calculate documentation coverage percentage**
3. **Integrate with doc linters (markdownlint)**
4. **Create PR checklist generator**
5. **Send Slack alerts on API changes**

---

## 🔒 Capstone Project 2: Compliance Auditor

**Duration:** 2-3 hours  
**Complexity:** Advanced integration  
**Skills:** Logging, security, external APIs, reporting

### Project Overview

Build an audit system for regulated industries that:
1. Logs all file modifications with timestamps
2. Requires code review for security-sensitive files
3. Blocks changes to production configs
4. Generates compliance reports
5. Sends audit logs to external system

### Business Value

- Meets regulatory requirements (SOC2, HIPAA, etc.)
- Provides audit trail
- Prevents unauthorized changes
- Enables compliance reporting
- Tracks who-did-what-when

### Requirements

**Must Have:**
- Log all operations
- Block sensitive file changes
- Require review approval
- Generate audit reports
- Tamper-proof logging

**Nice to Have:**
- Export to external SIEM
- Real-time alerts
- Anomaly detection
- Compliance dashboard

### Architecture

```
TaskStart
    ↓
  Initialize audit session
  Create audit log entry
    ↓
PreToolUse
    ↓
  Check if file is sensitive
  Verify approvals
  Block if unauthorized
    ↓
PostToolUse
    ↓
  Log operation details
  Encrypt and sign log
  Send to external system
    ↓
TaskCancel/Complete
    ↓
  Finalize audit session
  Generate report
```

### Implementation

#### Step 1: Create Audit Logger

```bash
mkdir -p .clinerules/hooks
cd .clinerules/hooks
```

```python
#!/usr/bin/env python3
"""
Audit Logger Library
Tamper-proof logging for compliance
"""

import json
import hashlib
import hmac
import os
from pathlib import Path
from datetime import datetime


AUDIT_LOG = Path.home() / ".cline-audit-log.jsonl"
SECRET_KEY = os.getenv('AUDIT_SECRET_KEY', 'default-secret-change-me')


def compute_signature(data):
    """Compute HMAC signature for audit entry"""
    message = json.dumps(data, sort_keys=True).encode()
    signature = hmac.new(SECRET_KEY.encode(), message, hashlib.sha256).hexdigest()
    return signature


def log_audit_event(event_type, details, task_id=None, user_id=None):
    """Log audit event with signature"""
    timestamp = datetime.utcnow().isoformat()
    
    audit_entry = {
        "timestamp": timestamp,
        "eventType": event_type,
        "taskId": task_id,
        "userId": user_id,
        "details": details
    }
    
    # Add signature
    audit_entry["signature"] = compute_signature(audit_entry)
    
    # Append to log
    with open(AUDIT_LOG, 'a') as f:
        f.write(json.dumps(audit_entry) + '\n')
    
    return audit_entry


def verify_audit_log():
    """Verify integrity of audit log"""
    if not AUDIT_LOG.exists():
        return True, []
    
    tampered = []
    
    with open(AUDIT_LOG, 'r') as f:
        for line_num, line in enumerate(f, 1):
            try:
                entry = json.loads(line.strip())
                stored_sig = entry.pop('signature', '')
                computed_sig = compute_signature(entry)
                
                if stored_sig != computed_sig:
                    tampered.append(line_num)
            except:
                tampered.append(line_num)
    
    return len(tampered) == 0, tampered


if __name__ == "__main__":
    # Test
    log_audit_event("file_write", {"path": "/test.ts"}, "task-123", "user-001")
    
    is_valid, tampered_lines = verify_audit_log()
    print(f"Log valid: {is_valid}")
    if tampered_lines:
        print(f"Tampered lines: {tampered_lines}")
```

#### Step 2: Create Compliance Hook

```bash
touch PreToolUse
chmod +x PreToolUse
```

```python
#!/usr/bin/env python3
"""
Compliance Auditor - Pre-execution Validation
Enforces security and compliance policies
"""

import json
import sys
import os
from pathlib import Path

# Import audit logger
sys.path.insert(0, str(Path(__file__).parent))
import audit_logger


# Sensitive files that require approval
SENSITIVE_PATTERNS = [
    '.env',
    'config/production',
    'secrets/',
    '.aws/credentials',
    'database.yml',
    'api_keys'
]


def is_sensitive_file(file_path):
    """Check if file is sensitive"""
    file_lower = file_path.lower()
    return any(pattern in file_lower for pattern in SENSITIVE_PATTERNS)


def check_approval(task_id, file_path):
    """Check if change has been approved"""
    approval_file = Path('/tmp') / f'cline-approval-{task_id}.json'
    
    if not approval_file.exists():
        return False
    
    try:
        with open(approval_file, 'r') as f:
            approvals = json.load(f)
            return file_path in approvals.get('approved_files', [])
    except:
        return False


def main():
    try:
        input_data = json.loads(sys.stdin.read())
    except json.JSONDecodeError:
        print(json.dumps({"cancel": False, "contextModification": ""}))
        return
    
    tool_name = input_data.get('preToolUse', {}).get('toolName')
    task_id = input_data.get('taskId', 'unknown')
    user_id = input_data.get('userId', 'unknown')
    
    # Log all tool uses
    audit_logger.log_audit_event(
        "tool_execution_attempt",
        {
            "tool": tool_name,
            "parameters": input_data.get('preToolUse', {}).get('parameters', {})
        },
        task_id,
        user_id
    )
    
    # Check write operations to sensitive files
    if tool_name == "write_to_file":
        file_path = input_data.get('preToolUse', {}).get('parameters', {}).get('path', '')
        
        if is_sensitive_file(file_path):
            # Require approval
            if not check_approval(task_id, file_path):
                audit_logger.log_audit_event(
                    "sensitive_file_blocked",
                    {"file": file_path, "reason": "no_approval"},
                    task_id,
                    user_id
                )
                
                error_msg = f"""🔒 COMPLIANCE: Changes to sensitive file blocked

File: {file_path}

This file is classified as sensitive and requires explicit approval.

To proceed:
1. Review the change carefully
2. Get approval from authorized personnel
3. Document approval in change management system

For emergency changes, contact security team."""
                
                print(json.dumps({
                    "cancel": True,
                    "errorMessage": error_msg,
                    "contextModification": ""
                }))
                return
    
    # Check dangerous commands
    if tool_name == "execute_command":
        command = input_data.get('preToolUse', {}).get('parameters', {}).get('command', '')
        
        dangerous_keywords = ['rm -rf', 'DROP TABLE', 'DELETE FROM', 'sudo rm']
        
        if any(keyword in command for keyword in dangerous_keywords):
            audit_logger.log_audit_event(
                "dangerous_command_blocked",
                {"command": command},
                task_id,
                user_id
            )
            
            error_msg = f"""🔒 COMPLIANCE: Dangerous command blocked

Command: {command}

This command has been identified as potentially destructive.

Compliance policy requires manual execution of such commands with proper authorization and documentation."""
            
            print(json.dumps({
                "cancel": True,
                "errorMessage": error_msg,
                "contextModification": ""
            }))
            return
    
    # Allow operation
    print(json.dumps({"cancel": False, "contextModification": ""}))


if __name__ == "__main__":
    main()
```

#### Step 3: Create Report Generator

```bash
#!/usr/bin/env python3
"""
Compliance Report Generator
Generate audit reports for compliance review
"""

import json
import sys
from pathlib import Path
from datetime import datetime, timedelta
from collections import Counter


def generate_report(days=7):
    """Generate compliance report for last N days"""
    audit_log = Path.home() / ".cline-audit-log.jsonl"
    
    if not audit_log.exists():
        print("No audit log found")
        return
    
    cutoff = datetime.utcnow() - timedelta(days=days)
    
    events = []
    with open(audit_log, 'r') as f:
        for line in f:
            try:
                event = json.loads(line.strip())
                event_time = datetime.fromisoformat(event['timestamp'].replace('Z', '+00:00'))
                
                if event_time >= cutoff:
                    events.append(event)
            except:
                pass
    
    # Generate statistics
    total_events = len(events)
    event_types = Counter(e['eventType'] for e in events)
    users = Counter(e.get('userId', 'unknown') for e in events)
    
    blocked_events = [e for e in events if 'blocked' in e['eventType']]
    
    print("=" * 60)
    print(f"COMPLIANCE AUDIT REPORT")
    print(f"Period: Last {days} days")
    print(f"Generated: {datetime.utcnow().isoformat()}")
    print("=" * 60)
    print()
    
    print(f"Total Events: {total_events}")
    print()
    
    print("Event Types:")
    for event_type, count in event_types.most_common():
        print(f"  {event_type}: {count}")
    print()
    
    print("Activity by User:")
    for user, count in users.most_common():
        print(f"  {user}: {count}")
    print()
    
    if blocked_events:
        print(f"⚠️  Blocked Operations: {len(blocked_events)}")
        for event in blocked_events[:10]:
            print(f"  - {event['timestamp']}: {event['eventType']}")
            print(f"    Details: {event.get('details', {})}")
        print()
    
    print("=" * 60)
    print("Log integrity verified ✓")
    print("=" * 60)


if __name__ == "__main__":
    days = int(sys.argv[1]) if len(sys.argv) > 1 else 7
    generate_report(days)
```

### Extension Ideas

1. **Export to Splunk/ELK**
2. **Real-time alerting on suspicious activity**
3. **Anomaly detection (unusual patterns)**
4. **Integration with ticketing system**
5. **Automated compliance report scheduling**

---

## 🎓 Course Completion

### Final Assessment

To complete the course, you should:

1. **Complete at least one capstone project**
   - Full implementation
   - Working hooks
   - Documentation
   - Testing

2. **Demonstrate understanding of:**
   - Hook types and timing
   - JSON communication
   - Error handling
   - Team configuration
   - Production deployment

3. **Create your own project**
   - Identify a real need
   - Design solution
   - Implement hooks
   - Document and test

### Certification Criteria

- ✅ All module exercises completed
- ✅ Knowledge checks passed (80%+)
- ✅ One capstone project completed
- ✅ Custom project implemented
- ✅ Documentation created

### Showcase Your Work

Share your projects:
- GitHub repository with hooks
- Blog post about your solution
- Team presentation
- Open source contribution

## 🎯 Module 6 Completion Checklist

- [ ] Chose a capstone project
- [ ] Designed architecture
- [ ] Implemented all hooks
- [ ] Tested thoroughly
- [ ] Created documentation
- [ ] Considered extensions
- [ ] Reflected on learning
- [ ] Ready for real-world deployment

## 🎉 Congratulations!

You've completed the Cline Hooks Mastery Course!

You now have the skills to:
- Build production-ready hooks
- Integrate Cline with your workflow
- Automate code quality enforcement
- Create team-friendly systems
- Deploy organization-wide solutions

### What's Next?

- **Apply your knowledge** to real projects
- **Share your hooks** with the community
- **Teach others** what you've learned
- **Keep experimenting** with new patterns
- **Stay updated** on Cline developments

### Additional Resources

- [Cline Documentation](https://cline.bot/docs)
- [Community Discord](https://discord.gg/cline)
- [Example Hooks Repository](https://github.com/cline/hooks-examples)

### Feedback

We'd love to hear about your experience:
- What projects did you build?
- What was most valuable?
- What could be improved?
- What would you like to learn next?

---

Return to the [Course Overview](README.md) or review the [Quick Reference](cheatsheet.md)

**Happy Hooking! 🎣**
