# Module 2: Validation & Enforcement

**Duration:** 2-3 hours | **Difficulty:** Intermediate

## 🎯 Learning Objectives

By the end of this module, you will:

- ✅ Implement blocking logic using `cancel: true`
- ✅ Create sophisticated validation rules
- ✅ Handle and communicate errors effectively
- ✅ Build confidence in pre-execution enforcement
- ✅ Understand when to block vs. when to warn
- ✅ Protect your codebase from problematic operations

## 📖 Theory: The Power of Prevention

### Why Validation Matters

Hooks that run *before* operations execute (PreToolUse) give you the power to prevent problems before they happen. This is incredibly valuable for:

- **Enforcing standards** - Ensure code style and project conventions
- **Preventing mistakes** - Block actions that violate rules
- **Security** - Stop dangerous operations before execution
- **Consistency** - Maintain project structure and patterns

### The Blocking Mechanism

When your hook returns `"cancel": true`, Cline:
1. Stops the current operation
2. Shows your error message to the user
3. Gives the AI the error message as feedback
4. Waits for the user or AI to revise the approach

This creates a feedback loop where the AI learns from rejections and adjusts its strategy.

### Validation Best Practices

**Be specific in error messages**
```bash
# ❌ Bad
"errorMessage": "File not allowed"

# ✅ Good
"errorMessage": "🚫 JavaScript files are not allowed in this TypeScript project. Please use .ts extension instead. File blocked: src/app.js"
```

**Validate early, fail fast**
- Check the most important rules first
- Return immediately on validation failure
- Don't waste time on additional checks after blocking

**Provide guidance**
- Tell users what's wrong
- Suggest the correct approach
- Include examples when helpful

## 🛡️ Exercise 2.1: "TypeScript Guardian" - Preventing Wrong File Types

**Duration:** 25 minutes  
**Hook Type:** PreToolUse  
**Difficulty:** ⭐⭐ Intermediate

### Scenario

You're working on a TypeScript project, but Cline occasionally tries to create JavaScript files. This causes type checking issues and violates project standards. Build a hook that enforces TypeScript file usage.

### What You'll Build

A validation hook that:
- Detects file write operations
- Checks file extensions
- Blocks `.js` files
- Suggests `.ts` alternatives
- Provides helpful error messages

### Real-World Use Case

This pattern applies to many scenarios:
- Enforcing TypeScript in TS projects
- Requiring tests for new features
- Maintaining naming conventions
- Preventing files in wrong directories

### Step-by-Step Instructions

#### 1. Create the Hook (or Update Existing)

```bash
cd .clinerules/hooks

# If you have PreToolUse from Module 1, back it up
mv PreToolUse PreToolUse.backup

# Create new PreToolUse
touch PreToolUse
code PreToolUse
```

#### 2. Write the TypeScript Guardian

```bash
#!/usr/bin/env bash

# TypeScript Guardian Hook
# Enforces TypeScript file extensions in TypeScript projects

# Read input
input=$(cat)

# Extract tool information
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

# Only validate file write operations
if [[ "$tool_name" != "write_to_file" ]]; then
    # Not a file write, allow everything else
    echo '{
      "cancel": false,
      "contextModification": ""
    }'
    exit 0
fi

# Extract file path from parameters
file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')

# Log for debugging (optional)
echo "Checking file: $file_path" >&2

# Check if it's a JavaScript file
if [[ "$file_path" == *.js ]]; then
    # Block the operation!
    echo '{
      "cancel": true,
      "contextModification": "",
      "errorMessage": "🚫 JavaScript files are not allowed in this TypeScript project.\n\n❌ Blocked: '"$file_path"'\n✅ Please use: '"${file_path%.js}.ts"'\n\nThis project uses TypeScript for type safety. All source files must use .ts or .tsx extensions."
    }'
    exit 0
fi

# Also check for JSX files
if [[ "$file_path" == *.jsx ]]; then
    echo '{
      "cancel": true,
      "contextModification": "",
      "errorMessage": "🚫 JSX files are not allowed in this TypeScript project.\n\n❌ Blocked: '"$file_path"'\n✅ Please use: '"${file_path%.jsx}.tsx"'\n\nThis project uses TypeScript. Use .tsx for React components with JSX syntax."
    }'
    exit 0
fi

# File passes validation
echo '{
  "cancel": false,
  "contextModification": ""
}'
```

#### 3. Make It Executable

```bash
chmod +x PreToolUse
```

#### 4. Test the Guardian

Let's test it manually first:

```bash
# Test 1: Try to create a JavaScript file (should block)
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
      "path": "/workspace/src/app.js",
      "content": "console.log('test');"
    }
  }
}
EOF
```

**Expected Output:**
```json
{
  "cancel": true,
  "errorMessage": "🚫 JavaScript files are not allowed..."
}
```

```bash
# Test 2: Try to create a TypeScript file (should allow)
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
      "path": "/workspace/src/app.ts",
      "content": "console.log('test');"
    }
  }
}
EOF
```

**Expected Output:**
```json
{
  "cancel": false,
  "contextModification": ""
}
```

#### 5. Test with Real Cline

1. Start a Cline task: "Create a new JavaScript file called utils.js with helper functions"
2. Watch Cline attempt to create the file
3. See the hook block it with your error message
4. Observe Cline adjust and create utils.ts instead

### Understanding the Code

Let's break down the key techniques:

```bash
# 1. Early exit pattern
if [[ "$tool_name" != "write_to_file" ]]; then
    echo '{"cancel": false}'
    exit 0
fi
# Only continue for file writes - efficient!

# 2. String pattern matching
if [[ "$file_path" == *.js ]]; then
# Matches any path ending with .js

# 3. String manipulation
"${file_path%.js}.ts"
# %.js removes .js from the end
# .ts adds the new extension
# example.js → example.ts

# 4. Multi-line error messages
"errorMessage": "Line 1\n\nLine 2\n\nLine 3"
# \n creates line breaks
# \n\n creates paragraph breaks
```

### Enhancement: Directory Exceptions

Let's make it smarter by allowing `.js` in certain directories (like `scripts/`):

```bash
#!/usr/bin/env bash

input=$(cat)
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

if [[ "$tool_name" != "write_to_file" ]]; then
    echo '{"cancel": false, "contextModification": ""}'; exit 0
fi

file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')

# Allow .js files in scripts/ directory
if [[ "$file_path" == *"/scripts/"* ]]; then
    echo '{"cancel": false, "contextModification": ""}'; exit 0
fi

# Block .js files elsewhere
if [[ "$file_path" == *.js ]]; then
    echo '{
      "cancel": true,
      "errorMessage": "🚫 JavaScript files only allowed in scripts/ directory.\n\n❌ Blocked: '"$file_path"'\n✅ For source files, use: '"${file_path%.js}.ts"'\n✅ For scripts, move to: scripts/'"$(basename "$file_path")
    }'
    exit 0
fi

echo '{"cancel": false, "contextModification": ""}'
```

### Challenge Tasks

Enhance your guardian to:

1. **Allow Test Files**
   - Permit `*.test.js` files (testing legacy code)
   - Still require `.test.ts` for new tests

2. **Check Configuration**
   - Read `tsconfig.json` to determine if project is TypeScript
   - Only enforce if TypeScript is configured

3. **Suggest Alternatives**
   - If blocking `component.jsx`, suggest using `.tsx`
   - Provide boilerplate TypeScript interfaces if useful

### Key Takeaways

✅ **Validation happens before execution** - No harm done  
✅ **Error messages guide users** - Be helpful, not just restrictive  
✅ **Pattern matching is powerful** - `[[ "$var" == *.ext ]]`  
✅ **Early exits optimize** - Don't check what you don't need  
✅ **Context matters** - Directory location can change rules

---

## 🔒 Exercise 2.2: "The Security Sentinel" - Preventing Dangerous Commands

**Duration:** 30 minutes  
**Hook Type:** PreToolUse  
**Difficulty:** ⭐⭐ Intermediate

### Scenario

AI agents can be incredibly helpful, but they can also execute dangerous commands if not supervised. Build a security hook that prevents potentially harmful operations while still allowing legitimate work.

### What You'll Build

A security validation hook that:
- Monitors command execution attempts
- Blocks dangerous patterns
- Warns on risky operations
- Allows safe commands
- Logs security events

### Real-World Use Case

This is critical for:
- Protecting production systems
- Preventing accidental data loss
- Enforcing security policies
- Compliance requirements
- Team safety standards

### Step-by-Step Instructions

#### 1. Create the Security Sentinel

```bash
cd .clinerules/hooks

# Back up existing PreToolUse if needed
mv PreToolUse PreToolUse.typescript-guardian

# Create new security-focused hook
touch PreToolUse
code PreToolUse
```

#### 2. Write the Sentinel Script

```bash
#!/usr/bin/env bash

# Security Sentinel Hook
# Prevents dangerous command execution

input=$(cat)
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

# Only monitor command execution
if [[ "$tool_name" != "execute_command" ]]; then
    echo '{"cancel": false, "contextModification": ""}'; exit 0
fi

# Extract the command
command=$(echo "$input" | jq -r '.preToolUse.parameters.command')

# Log for security audit
security_log="$HOME/.cline-security-audit.log"
echo "[$(date -Iseconds)] Command attempted: $command" >> "$security_log"

# ═══════════════════════════════════════════════════════════
# CRITICAL DANGER - BLOCK IMMEDIATELY
# ═══════════════════════════════════════════════════════════

# Prevent destructive filesystem operations
if [[ "$command" =~ rm[[:space:]]+-rf[[:space:]]+/([[:space:]]|$) ]] || \
   [[ "$command" =~ rm[[:space:]]+-rf[[:space:]]+/\* ]]; then
    echo '{
      "cancel": true,
      "errorMessage": "🛑 CRITICAL: Destructive filesystem operation blocked!\n\n❌ Command: '"$command"'\n\n⚠️  This command would delete system files.\nThis operation is NEVER allowed."
    }'
    echo "[$(date -Iseconds)] BLOCKED - Destructive rm: $command" >> "$security_log"
    exit 0
fi

# Prevent fork bombs
if [[ "$command" =~ :\(\)\{.*:\|:.*\}\;: ]]; then
    echo '{
      "cancel": true,
      "errorMessage": "🛑 CRITICAL: Fork bomb detected and blocked!\n\n❌ Command: '"$command"'\n\nThis is a denial of service attack."
    }'
    echo "[$(date -Iseconds)] BLOCKED - Fork bomb: $command" >> "$security_log"
    exit 0
fi

# Prevent filesystem formatting
if [[ "$command" =~ ^mkfs ]] || [[ "$command" =~ [[:space:]]mkfs ]]; then
    echo '{
      "cancel": true,
      "errorMessage": "🛑 CRITICAL: Filesystem format operation blocked!\n\n❌ Command: '"$command"'\n\nmkfs operations could destroy data."
    }'
    echo "[$(date -Iseconds)] BLOCKED - mkfs: $command" >> "$security_log"
    exit 0
fi

# Prevent direct disk operations
if [[ "$command" =~ /dev/sd[a-z] ]] || [[ "$command" =~ /dev/nvme ]]; then
    echo '{
      "cancel": true,
      "errorMessage": "🛑 CRITICAL: Direct disk access blocked!\n\n❌ Command: '"$command"'\n\nDirect disk operations could cause data loss."
    }'
    echo "[$(date -Iseconds)] BLOCKED - Disk access: $command" >> "$security_log"
    exit 0
fi

# ═══════════════════════════════════════════════════════════
# HIGH RISK - BLOCK WITH EXPLANATION
# ═══════════════════════════════════════════════════════════

# Block rm -rf without absolute path
if [[ "$command" =~ rm[[:space:]]+-rf ]]; then
    echo '{
      "cancel": true,
      "errorMessage": "⛔ Blocked: rm -rf detected\n\n❌ Command: '"$command"'\n\n⚠️  This command is too dangerous to run automatically.\n\nIf you need to delete:\n• Use specific file paths\n• Run manually in terminal\n• Use safer alternatives like: rm -r <specific-dir>"
    }'
    echo "[$(date -Iseconds)] BLOCKED - rm -rf: $command" >> "$security_log"
    exit 0
fi

# Block chmod 777
if [[ "$command" =~ chmod[[:space:]]+777 ]]; then
    echo '{
      "cancel": true,
      "errorMessage": "⛔ Blocked: chmod 777 is a security risk\n\n❌ Command: '"$command"'\n\n⚠️  This grants read/write/execute to everyone.\n\n✅ Better alternatives:\n• chmod 755 - Read and execute for everyone, write for owner\n• chmod 644 - Read for everyone, write for owner"
    }'
    echo "[$(date -Iseconds)] BLOCKED - chmod 777: $command" >> "$security_log"
    exit 0
fi

# Block curl piped to shell
if [[ "$command" =~ curl.*\|.*sh ]] || [[ "$command" =~ wget.*\|.*sh ]]; then
    echo '{
      "cancel": true,
      "errorMessage": "⛔ Blocked: Piping downloaded content to shell\n\n❌ Command: '"$command"'\n\n⚠️  This could execute malicious code.\n\n✅ Safer approach:\n1. Download the script first\n2. Review the contents\n3. Execute if safe"
    }'
    echo "[$(date -Iseconds)] BLOCKED - Curl pipe: $command" >> "$security_log"
    exit 0
fi

# ═══════════════════════════════════════════════════════════
# MEDIUM RISK - WARN BUT ALLOW
# ═══════════════════════════════════════════════════════════

# Warn about sudo usage
if [[ "$command" =~ ^sudo ]] || [[ "$command" =~ [[:space:]]sudo ]]; then
    echo '{
      "cancel": false,
      "contextModification": "⚠️ SECURITY NOTICE: A sudo command was executed: '"$command"'. Sudo grants elevated privileges and should be used carefully."
    }'
    echo "[$(date -Iseconds)] ALLOWED with warning - sudo: $command" >> "$security_log"
    exit 0
fi

# Warn about password in command
if [[ "$command" =~ password=|passwd=|pwd= ]] || [[ "$command" =~ -p[[:space:]]*[^[:space:]] ]]; then
    echo '{
      "cancel": false,
      "contextModification": "⚠️ SECURITY NOTICE: Command may contain password in plaintext: '"$command"'. Consider using environment variables or secure credential storage."
    }'
    echo "[$(date -Iseconds)] ALLOWED with warning - password: $command" >> "$security_log"
    exit 0
fi

# Warn about sensitive file access
if [[ "$command" =~ /etc/passwd ]] || \
   [[ "$command" =~ /etc/shadow ]] || \
   [[ "$command" =~ \.ssh/id_rsa ]] || \
   [[ "$command" =~ \.aws/credentials ]]; then
    echo '{
      "cancel": false,
      "contextModification": "⚠️ SECURITY NOTICE: Command accesses sensitive file: '"$command"'. Ensure this is intentional."
    }'
    echo "[$(date -Iseconds)] ALLOWED with warning - sensitive file: $command" >> "$security_log"
    exit 0
fi

# ═══════════════════════════════════════════════════════════
# SAFE - ALLOW
# ═══════════════════════════════════════════════════════════

# Log and allow safe commands
echo "[$(date -Iseconds)] ALLOWED: $command" >> "$security_log"

echo '{
  "cancel": false,
  "contextModification": ""
}'
```

#### 3. Make Executable

```bash
chmod +x PreToolUse
```

#### 4. Test the Sentinel

Let's test various scenarios:

```bash
# Test 1: Dangerous command (should block)
cat << 'EOF' | ./PreToolUse
{
  "clineVersion": "1.0.0",
  "hookName": "PreToolUse",
  "timestamp": "2025-11-06T10:30:00Z",
  "taskId": "test-123",
  "workspaceRoots": ["/workspace"],
  "userId": "user-001",
  "preToolUse": {
    "toolName": "execute_command",
    "parameters": {
      "command": "rm -rf /"
    }
  }
}
EOF

# Test 2: Sudo command (should warn but allow)
cat << 'EOF' | ./PreToolUse
{
  "clineVersion": "1.0.0",
  "hookName": "PreToolUse",
  "timestamp": "2025-11-06T10:30:00Z",
  "taskId": "test-123",
  "workspaceRoots": ["/workspace"],
  "userId": "user-001",
  "preToolUse": {
    "toolName": "execute_command",
    "parameters": {
      "command": "sudo npm install -g typescript"
    }
  }
}
EOF

# Test 3: Safe command (should allow)
cat << 'EOF' | ./PreToolUse
{
  "clineVersion": "1.0.0",
  "hookName": "PreToolUse",
  "timestamp": "2025-11-06T10:30:00Z",
  "taskId": "test-123",
  "workspaceRoots": ["/workspace"],
  "userId": "user-001",
  "preToolUse": {
    "toolName": "execute_command",
    "parameters": {
      "command": "npm test"
    }
  }
}
EOF
```

#### 5. Check the Security Audit Log

```bash
# View the security log
cat ~/.cline-security-audit.log

# View recent entries
tail -20 ~/.cline-security-audit.log
```

### Understanding Security Levels

The hook implements three security levels:

**CRITICAL (Block)** - Immediate danger
- System file deletion
- Fork bombs
- Disk formatting
- Direct disk access

**HIGH RISK (Block with guidance)** - Too risky for automation
- `rm -rf` operations
- `chmod 777` permissions
- Curl piped to shell

**MEDIUM RISK (Warn)** - Allowed but logged
- Sudo usage
- Passwords in commands
- Sensitive file access

### Regular Expression Patterns Explained

```bash
# Match rm -rf /
[[ "$command" =~ rm[[:space:]]+-rf[[:space:]]+/ ]]
# Breaks down to:
# - rm: literal text "rm"
# - [[:space:]]+: one or more spaces
# - -rf: literal flags
# - [[:space:]]+: one or more spaces
# - /: root directory

# Match sudo anywhere in command
[[ "$command" =~ ^sudo ]] || [[ "$command" =~ [[:space:]]sudo ]]
# ^sudo: sudo at start of command
# [[:space:]]sudo: sudo after a space (middle of command)
```

### Enhancement: Whitelist Approach

For even tighter security, consider a whitelist:

```bash
# Define allowed commands
allowed_commands=("npm" "yarn" "git" "ls" "cat" "grep" "find" "echo")

command_base=$(echo "$command" | awk '{print $1}')

is_allowed=false
for allowed in "${allowed_commands[@]}"; do
    if [[ "$command_base" == "$allowed" ]]; then
        is_allowed=true
        break
    fi
done

if [[ "$is_allowed" == false ]]; then
    echo '{
      "cancel": true,
      "errorMessage": "⛔ Command not in whitelist: '"$command_base"'"
    }'
    exit 0
fi
```

### Challenge Tasks

Enhance your sentinel to:

1. **Environment-Based Rules**
   - Stricter in production
   - More permissive in development
   - Check environment variables

2. **Rate Limiting**
   - Block if >10 sudo commands in 5 minutes
   - Prevent command spam

3. **User Approval**
   - Pause and require confirmation for risky commands
   - Log approval decisions

### Key Takeaways

✅ **Layer security checks** - Critical → High → Medium → Allow  
✅ **Log everything** - Audit trail is essential  
✅ **Explain blocks** - Help users understand  
✅ **Regular expressions** - Powerful pattern matching  
✅ **Balance safety and usability** - Don't break legitimate workflows

---

## 🎓 Module 2 Review

### What You've Learned

- ✅ How to block operations with `cancel: true`
- ✅ Writing effective error messages
- ✅ Pattern matching and validation logic
- ✅ Security best practices
- ✅ Risk-based decision making
- ✅ Audit logging
- ✅ Conditional rules and exceptions

### Practical Skills

You can now:
- Enforce file naming conventions
- Prevent dangerous operations
- Build flexible validation rules
- Create security policies
- Log and audit operations
- Provide helpful error feedback

## 📊 Knowledge Check

1. **What happens when a hook returns `"cancel": true`?**
   <details>
   <summary>Answer</summary>
   
   Cline stops the operation, shows the error message to the user, and gives the AI the error message as feedback. The operation does not execute.
   </details>

2. **Should you block or warn for `sudo` commands?**
   <details>
   <summary>Answer</summary>
   
   Warn but allow. Sudo is often necessary for legitimate operations. Log it for audit purposes and inject context warning, but don't block unless you have specific sudo restrictions.
   </details>

3. **How do you make error messages helpful?**
   <details>
   <summary>Answer</summary>
   
   Include:
   - What was blocked and why
   - What the user should do instead
   - Examples of correct usage
   - Emojis for visual clarity
   </details>

4. **When should you use early exit in validation?**
   <details>
   <summary>Answer</summary>
   
   As soon as you determine the hook doesn't need to process the operation (wrong tool type) or as soon as a validation check fails. No need to run additional checks after blocking.
   </details>

## 🎯 Module 2 Completion Checklist

- [ ] Created TypeScript Guardian hook
- [ ] Tested blocking JavaScript files
- [ ] Added directory exceptions
- [ ] Created Security Sentinel hook
- [ ] Tested dangerous command blocking
- [ ] Implemented warning for sudo
- [ ] Set up audit logging
- [ ] Can write effective error messages
- [ ] Understand validation patterns
- [ ] Passed knowledge check

## 🚀 Next Steps

You now know how to enforce rules and prevent problems. Next, learn how to make hooks that learn from operations and inject intelligent context:

**[Module 3: Context Injection & Learning →](module-03-context.md)**

Or return to the [Course Overview](README.md)
