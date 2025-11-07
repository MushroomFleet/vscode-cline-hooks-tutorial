# Module 5: Production Patterns

**Duration:** 2-3 hours | **Difficulty:** Advanced

## 🎯 Learning Objectives

By the end of this module, you will:

- ✅ Implement robust error handling
- ✅ Build hooks that never break Cline
- ✅ Create team-friendly configurations
- ✅ Handle edge cases gracefully
- ✅ Optimize hook performance
- ✅ Establish testing strategies
- ✅ Deploy hooks organization-wide

## 📖 Theory: Production-Grade Hooks

### The Production Mindset

Development hooks can fail loudly. Production hooks must fail gracefully.

**Key Principles:**

**Never break the user's flow** - Hooks should enhance, not interrupt  
**Fail silently when appropriate** - Log errors, don't crash  
**Timeout protection** - Don't hang indefinitely  
**Resource cleanup** - Clean up temp files and connections  
**Clear error messages** - Help users understand what went wrong

### Error Handling Hierarchy

```
1. Critical errors → Block operation with clear message
2. Recoverable errors → Log and continue
3. Warnings → Inject context, don't block
4. Info → Log only, silent to user
```

### Performance Considerations

**Fast hooks are good hooks:**
- Target: <1s for most operations
- Maximum: <5s (Cline timeout)
- Async for expensive operations
- Cache when possible
- Batch operations

### Team Deployment

**Organization-wide hooks:**
- Place in `~/Documents/Cline/Rules/Hooks/`
- Apply to all projects
- Enforce company standards

**Project-specific hooks:**
- Place in `.clinerules/hooks/`
- Version control with project
- Team-specific workflows

## 🛡️ Exercise 5.1: "Bulletproof Hooks" - Comprehensive Error Handling

**Duration:** 35 minutes  
**Difficulty:** ⭐⭐⭐ Advanced

### Scenario

Create a production-ready hook template that handles all edge cases: invalid JSON, missing fields, timeouts, external tool failures, and more.

### What You'll Build

A robust hook template featuring:
- Comprehensive error handling
- Timeout protection
- Input validation
- Graceful degradation
- Detailed logging
- Safe defaults

### Step-by-Step Instructions

#### 1. Create the Bulletproof Template

```bash
cd .clinerules/hooks
touch PreToolUse
chmod +x PreToolUse
```

```bash
#!/usr/bin/env bash

#═══════════════════════════════════════════════════════════════════════════
# BULLETPROOF HOOK TEMPLATE
# Comprehensive error handling for production environments
#═══════════════════════════════════════════════════════════════════════════

# Enable strict error handling
set -euo pipefail

# ═══════════════════════════════════════════════════════════
# CONFIGURATION
# ═══════════════════════════════════════════════════════════

HOOK_NAME="PreToolUse"
HOOK_TIMEOUT=5  # seconds
ERROR_LOG="${HOME}/.cline-hooks-errors.log"
DEBUG_MODE="${CLINE_HOOK_DEBUG:-false}"

# ═══════════════════════════════════════════════════════════
# LOGGING FUNCTIONS
# ═══════════════════════════════════════════════════════════

log_error() {
    local timestamp=$(date -Iseconds)
    echo "[ERROR $timestamp] $HOOK_NAME: $*" >> "$ERROR_LOG"
    
    if [ "$DEBUG_MODE" = "true" ]; then
        echo "[ERROR] $*" >&2
    fi
}

log_warning() {
    local timestamp=$(date -Iseconds)
    echo "[WARN $timestamp] $HOOK_NAME: $*" >> "$ERROR_LOG"
    
    if [ "$DEBUG_MODE" = "true" ]; then
        echo "[WARN] $*" >&2
    fi
}

log_info() {
    if [ "$DEBUG_MODE" = "true" ]; then
        echo "[INFO] $*" >&2
    fi
}

log_debug() {
    if [ "$DEBUG_MODE" = "true" ]; then
        echo "[DEBUG] $*" >&2
    fi
}

# ═══════════════════════════════════════════════════════════
# SAFE RESPONSE FUNCTION
# ═══════════════════════════════════════════════════════════

safe_response() {
    # Always return valid JSON that allows execution
    local context="${1:-}"
    
    if [ -n "$context" ]; then
        echo "{\"cancel\": false, \"contextModification\": \"$context\"}"
    else
        echo '{"cancel": false, "contextModification": ""}'
    fi
}

blocking_response() {
    local error_message="$1"
    local context="${2:-}"
    
    # Escape special characters in error message
    error_message=$(echo "$error_message" | jq -Rs .)
    context=$(echo "$context" | jq -Rs .)
    
    echo "{\"cancel\": true, \"errorMessage\": $error_message, \"contextModification\": $context}"
}

# ═══════════════════════════════════════════════════════════
# INPUT VALIDATION
# ═══════════════════════════════════════════════════════════

validate_input() {
    local input="$1"
    
    # Check if input is empty
    if [ -z "$input" ]; then
        log_error "Empty input received"
        return 1
    fi
    
    # Validate JSON
    if ! echo "$input" | jq empty 2>/dev/null; then
        log_error "Invalid JSON input"
        return 1
    fi
    
    log_debug "Input validation passed"
    return 0
}

# ═══════════════════════════════════════════════════════════
# SAFE FIELD EXTRACTION
# ═══════════════════════════════════════════════════════════

safe_get() {
    local input="$1"
    local path="$2"
    local default="${3:-}"
    
    local value=$(echo "$input" | jq -r "$path // \"$default\"" 2>/dev/null || echo "$default")
    echo "$value"
}

# ═══════════════════════════════════════════════════════════
# MAIN LOGIC (with timeout protection)
# ═══════════════════════════════════════════════════════════

main() {
    log_debug "Hook execution started"
    
    # ───────────────────────────────────────────────────────
    # STEP 1: Read and validate input
    # ───────────────────────────────────────────────────────
    
    local input
    if ! input=$(cat); then
        log_error "Failed to read stdin"
        safe_response
        return 0
    fi
    
    if ! validate_input "$input"; then
        log_error "Input validation failed"
        safe_response
        return 0
    fi
    
    # ───────────────────────────────────────────────────────
    # STEP 2: Extract fields with safe defaults
    # ───────────────────────────────────────────────────────
    
    local tool_name=$(safe_get "$input" '.preToolUse.toolName' 'unknown')
    local task_id=$(safe_get "$input" '.taskId' 'unknown')
    local timestamp=$(safe_get "$input" '.timestamp' '')
    
    log_info "Processing $tool_name for task $task_id"
    
    # ───────────────────────────────────────────────────────
    # STEP 3: Tool-specific validation logic
    # ───────────────────────────────────────────────────────
    
    case "$tool_name" in
        "write_to_file")
            local file_path=$(safe_get "$input" '.preToolUse.parameters.path' '')
            
            if [ -z "$file_path" ]; then
                log_warning "Empty file path in write_to_file"
                safe_response "⚠️ Warning: Empty file path detected"
                return 0
            fi
            
            log_debug "File path: $file_path"
            
            # Add your validation logic here
            # Example: Check file extension
            if [[ "$file_path" == *.js ]]; then
                log_info "JavaScript file detected: $file_path"
                # Add your .js validation
            fi
            ;;
            
        "execute_command")
            local command=$(safe_get "$input" '.preToolUse.parameters.command' '')
            
            if [ -z "$command" ]; then
                log_warning "Empty command in execute_command"
                safe_response
                return 0
            fi
            
            log_debug "Command: $command"
            
            # Add your command validation logic here
            ;;
            
        "read_file")
            local file_path=$(safe_get "$input" '.preToolUse.parameters.path' '')
            log_debug "Reading file: $file_path"
            ;;
            
        *)
            log_debug "Unhandled tool type: $tool_name"
            ;;
    esac
    
    # ───────────────────────────────────────────────────────
    # STEP 4: External tool integration (with error handling)
    # ───────────────────────────────────────────────────────
    
    if command -v some_external_tool &> /dev/null; then
        log_debug "External tool available, running checks"
        
        # Run external tool with timeout and error handling
        local output
        if ! output=$(timeout 3s some_external_tool 2>&1); then
            log_warning "External tool failed or timed out"
            # Don't block on external tool failure
            safe_response "⚠️ External validation unavailable"
            return 0
        fi
        
        log_debug "External tool output: ${output:0:100}..."
    fi
    
    # ───────────────────────────────────────────────────────
    # STEP 5: Return success
    # ───────────────────────────────────────────────────────
    
    safe_response
    log_debug "Hook execution completed successfully"
}

# ═══════════════════════════════════════════════════════════
# TIMEOUT WRAPPER
# ═══════════════════════════════════════════════════════════

# Run main with timeout protection
if timeout "${HOOK_TIMEOUT}s" bash -c "$(declare -f main); $(declare -f safe_response); $(declare -f safe_get); $(declare -f log_error); $(declare -f log_warning); $(declare -f log_info); $(declare -f log_debug); main"; then
    log_debug "Hook completed within timeout"
else
    log_error "Hook timeout exceeded (${HOOK_TIMEOUT}s)"
    safe_response "⚠️ Hook execution timeout"
fi

# Always exit successfully so we don't break Cline
exit 0
```

#### 2. Enable Debug Mode for Testing

```bash
# Enable debug output
export CLINE_HOOK_DEBUG=true

# Test with valid input
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

# Test with invalid JSON
echo "not json" | ./PreToolUse

# Test with empty input
echo "" | ./PreToolUse

# Test with missing fields
echo '{"hookName": "PreToolUse"}' | ./PreToolUse
```

#### 3. Review Error Log

```bash
# Check what errors were logged
cat ~/.cline-hooks-errors.log

# Monitor in real-time
tail -f ~/.cline-hooks-errors.log
```

### Key Error Handling Patterns

**1. Input Validation**
```bash
# Always validate before processing
if ! validate_input "$input"; then
    safe_response  # Allow execution despite error
    return 0
fi
```

**2. Safe Field Extraction**
```bash
# Use defaults for missing fields
local value=$(safe_get "$input" '.path.to.field' 'default')
```

**3. External Tool Protection**
```bash
# Timeout + error handling
if ! output=$(timeout 3s external_tool 2>&1); then
    log_warning "Tool failed"
    safe_response  # Continue despite failure
    return 0
fi
```

**4. Always Exit Successfully**
```bash
# Never exit with non-zero, it breaks Cline
exit 0
```

### Testing Your Error Handling

Create a comprehensive test suite:

```bash
#!/usr/bin/env bash
# test-hook-errors.sh

echo "Testing error handling..."

tests_passed=0
tests_failed=0

run_test() {
    local name="$1"
    local input="$2"
    local expected_pattern="$3"
    
    echo "Running: $name"
    output=$(echo "$input" | ./PreToolUse 2>&1)
    
    if echo "$output" | grep -q "$expected_pattern"; then
        echo "✅ PASS: $name"
        ((tests_passed++))
    else
        echo "❌ FAIL: $name"
        echo "Output: $output"
        ((tests_failed++))
    fi
}

# Test 1: Empty input
run_test "Empty input" "" '"cancel": false'

# Test 2: Invalid JSON
run_test "Invalid JSON" "not json" '"cancel": false'

# Test 3: Missing fields
run_test "Missing fields" '{"hookName":"test"}' '"cancel": false'

# Test 4: Valid input
run_test "Valid input" '{"clineVersion":"1.0","hookName":"PreToolUse","timestamp":"2025-11-06T10:30:00Z","taskId":"test","workspaceRoots":["/workspace"],"userId":"user-001","preToolUse":{"toolName":"write_to_file","parameters":{"path":"/test.ts","content":"test"}}}' '"cancel": false'

echo ""
echo "Tests passed: $tests_passed"
echo "Tests failed: $tests_failed"
```

### Key Takeaways

✅ **Validate all inputs** - Never assume correct format  
✅ **Use safe defaults** - Missing fields shouldn't crash  
✅ **Timeout protection** - Don't hang indefinitely  
✅ **Log errors** - Don't fail silently  
✅ **Always exit 0** - Never break Cline's flow  
✅ **Graceful degradation** - Continue when possible

---

## 👥 Exercise 5.2: "Team Hooks" - Collaborative Configuration

**Duration:** 30 minutes  
**Difficulty:** ⭐⭐⭐ Advanced

### Scenario

Your team needs shared hooks with some flexibility for individual preferences. Build a configuration system that balances team standards with personal customization.

### What You'll Build

A configuration-driven hook system:
- Team-wide base configuration
- Personal override system
- Environment-specific rules
- Shared + individual preferences

### Step-by-Step Instructions

#### 1. Create Team Configuration

```bash
mkdir -p .clinerules
cat > .clinerules/hooks-config.json << 'EOF'
{
  "version": "1.0",
  "team": {
    "linting": {
      "enabled": true,
      "blockOnError": true,
      "warnOnWarning": true,
      "tools": {
        "eslint": {
          "enabled": true,
          "autoFix": false
        },
        "prettier": {
          "enabled": true,
          "autoFix": false
        },
        "flake8": {
          "enabled": true,
          "maxLineLength": 100
        }
      }
    },
    "security": {
      "blockDangerousCommands": true,
      "requireApprovalFor": ["sudo", "rm -rf"],
      "allowedSudoCommands": ["npm install -g", "apt-get install"]
    },
    "codeStandards": {
      "maxFileLines": 500,
      "minCommentRatio": 5,
      "requiredFileHeaders": true
    },
    "notifications": {
      "enabled": true,
      "channels": {
        "slack": {
          "enabled": true,
          "webhookEnv": "TEAM_SLACK_WEBHOOK"
        },
        "discord": {
          "enabled": false
        }
      },
      "notifyOn": ["test_failure", "deployment", "error"]
    }
  },
  "personalOverrides": {
    "enabled": true,
    "configPath": "~/.cline-personal-config.json",
    "allowedOverrides": [
      "linting.warnOnWarning",
      "codeStandards.maxFileLines",
      "notifications.channels"
    ]
  }
}
EOF

# Commit this to version control
git add .clinerules/hooks-config.json
```

#### 2. Create Personal Config Template

```bash
cat > ~/.cline-personal-config.json << 'EOF'
{
  "version": "1.0",
  "overrides": {
    "linting": {
      "warnOnWarning": false
    },
    "codeStandards": {
      "maxFileLines": 750
    },
    "notifications": {
      "channels": {
        "slack": {
          "enabled": false
        }
      }
    }
  },
  "personalRules": {
    "preferredCodeStyle": "functional",
    "customPatterns": [
      "Avoid class components, prefer hooks",
      "Use async/await over promises.then()"
    ]
  }
}
EOF
```

#### 3. Create Configuration-Aware Hook

```bash
cd .clinerules/hooks
touch PreToolUse
chmod +x PreToolUse
```

```python
#!/usr/bin/env python3
"""
Configuration-Driven Hook
Loads team config with personal overrides
"""

import json
import sys
import os
from pathlib import Path


def load_config():
    """Load team config with personal overrides"""
    config = {
        "linting": {"enabled": True},
        "security": {"blockDangerousCommands": True},
        "codeStandards": {}
    }
    
    # Load team configuration
    team_config_path = Path('.clinerules/hooks-config.json')
    if team_config_path.exists():
        try:
            with open(team_config_path) as f:
                team_data = json.load(f)
                config.update(team_data.get('team', {}))
        except Exception as e:
            print(f"Warning: Failed to load team config: {e}", file=sys.stderr)
    
    # Load personal overrides if allowed
    personal_enabled = config.get('personalOverrides', {}).get('enabled', False)
    
    if personal_enabled:
        personal_path = Path.home() / '.cline-personal-config.json'
        if personal_path.exists():
            try:
                with open(personal_path) as f:
                    personal_data = json.load(f)
                    
                    # Merge allowed overrides
                    allowed = config.get('personalOverrides', {}).get('allowedOverrides', [])
                    overrides = personal_data.get('overrides', {})
                    
                    # Deep merge allowed fields
                    for override_path in allowed:
                        keys = override_path.split('.')
                        
                        # Navigate to the override value
                        override_value = overrides
                        for key in keys:
                            override_value = override_value.get(key, {})
                        
                        # Apply override if found
                        if override_value:
                            current = config
                            for key in keys[:-1]:
                                current = current.setdefault(key, {})
                            current[keys[-1]] = override_value
                    
                    # Add personal rules
                    config['personalRules'] = personal_data.get('personalRules', {})
                    
            except Exception as e:
                print(f"Warning: Failed to load personal config: {e}", file=sys.stderr)
    
    return config


def should_lint(config, file_path):
    """Check if linting is enabled for this file"""
    linting = config.get('linting', {})
    
    if not linting.get('enabled', True):
        return False
    
    # Check file type
    ext = Path(file_path).suffix
    
    if ext in ['.ts', '.tsx', '.js', '.jsx']:
        return linting.get('tools', {}).get('eslint', {}).get('enabled', True)
    elif ext == '.py':
        return linting.get('tools', {}).get('flake8', {}).get('enabled', True)
    
    return False


def get_personal_guidance(config):
    """Get personal rules as context"""
    personal_rules = config.get('personalRules', {})
    custom_patterns = personal_rules.get('customPatterns', [])
    
    if custom_patterns:
        return "PERSONAL_PREFERENCES: " + " | ".join(custom_patterns)
    
    return ""


def main():
    try:
        input_data = json.loads(sys.stdin.read())
    except json.JSONDecodeError:
        print(json.dumps({"cancel": False, "contextModification": ""}))
        return
    
    # Load configuration
    config = load_config()
    
    tool_name = input_data.get('preToolUse', {}).get('toolName')
    
    # Handle based on configuration
    context = ""
    
    if tool_name == "write_to_file":
        file_path = input_data.get('preToolUse', {}).get('parameters', {}).get('path', '')
        
        # Check if linting is enabled
        if should_lint(config, file_path):
            # Linting logic would go here
            pass
        
        # Add personal guidance
        personal_context = get_personal_guidance(config)
        if personal_context:
            context = personal_context
    
    # Return response
    response = {
        "cancel": False,
        "contextModification": context
    }
    
    print(json.dumps(response))


if __name__ == "__main__":
    main()
```

#### 4. Test Configuration Loading

```bash
# Test the hook
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

# Should see personal preferences in output
```

### Configuration Best Practices

**1. Version Your Configs**
```json
{
  "version": "1.0",  // Track config schema version
  "team": { ... }
}
```

**2. Document Available Options**
```bash
# Create .clinerules/hooks-config-schema.md
```

**3. Provide Examples**
```bash
# Create .clinerules/hooks-config-example.json
```

**4. Validate Configs**
```python
def validate_config(config):
    """Ensure config has required fields"""
    required = ['version', 'team']
    for field in required:
        if field not in config:
            return False
    return True
```

### Team Documentation Template

Create `.clinerules/HOOKS.md`:

```markdown
# Team Hooks Configuration

## Overview
This project uses Cline hooks to enforce code quality and team standards.

## Configuration
Edit `.clinerules/hooks-config.json` to modify team-wide settings.

## Personal Overrides
Create `~/.cline-personal-config.json` to customize:
- Linting severity
- File size limits
- Notification preferences

See `.clinerules/hooks-config-example.json` for format.

## Available Hooks
- **PreToolUse**: Validates files before writing
- **PostToolUse**: Tracks metrics and sends notifications

## Troubleshooting
Check `~/.cline-hooks-errors.log` for error details.

## Questions?
Contact: team-lead@company.com
```

### Key Takeaways

✅ **Team config in repo** - Version controlled standards  
✅ **Personal overrides** - Balance structure with flexibility  
✅ **Document everything** - Make it easy for team  
✅ **Validate configs** - Fail gracefully on errors  
✅ **Provide examples** - Show don't just tell

---

## 🎓 Module 5 Review

### What You've Learned

- ✅ Comprehensive error handling
- ✅ Timeout protection
- ✅ Input validation patterns
- ✅ Graceful degradation
- ✅ Team configuration systems
- ✅ Personal override patterns
- ✅ Production deployment strategies

### Practical Skills

You can now:
- Build bulletproof hooks
- Handle all edge cases
- Create team-friendly configs
- Deploy organization-wide
- Test thoroughly
- Debug effectively

## 📊 Knowledge Check

1. **Why should hooks always exit with 0?**
   <details>
   <summary>Answer</summary>
   
   Non-zero exit codes signal failure to Cline and can break the user's workflow. Hooks should handle errors internally and always exit successfully, using JSON response to communicate validation results.
   </details>

2. **What's the difference between team and personal configs?**
   <details>
   <summary>Answer</summary>
   
   Team configs are version-controlled in the project repo and apply to everyone. Personal configs are user-specific overrides stored in home directory, allowing individual customization within team boundaries.
   </details>

3. **How long should hooks take to execute?**
   <details>
   <summary>Answer</summary>
   
   Target <1s for most operations, maximum <5s to avoid timeouts. Use timeout protection and async operations for anything longer.
   </details>

## 🎯 Module 5 Completion Checklist

- [ ] Created bulletproof error handling template
- [ ] Tested with invalid inputs
- [ ] Implemented timeout protection
- [ ] Created team configuration system
- [ ] Set up personal overrides
- [ ] Documented team hooks
- [ ] Understand production deployment
- [ ] Passed knowledge check

## 🚀 Next Steps

You've mastered production patterns. Ready to build complete, real-world systems?

**[Module 6: Real-World Projects →](module-06-projects.md)**

Or return to the [Course Overview](README.md)
