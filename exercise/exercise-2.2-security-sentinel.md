# Exercise 2.2: "The Security Sentinel" - Preventing Dangerous Commands

**Module:** Validation & Enforcement  
**Duration:** 30 minutes  
**Hook Type:** PreToolUse  
**Difficulty:** ⭐⭐ Intermediate

## Learning Objectives

By completing this exercise, you will:
- Build security-focused validation hooks
- Implement pattern matching for dangerous commands
- Distinguish between blocking and warning scenarios
- Create graduated responses (block vs warn)
- Handle complex string matching in bash

## Scenario

Cline has access to execute shell commands on your system. While this is powerful and necessary, it also creates potential risks. A misguided AI decision, a misunderstood request, or a simple mistake could result in:

- Deleting important files (`rm -rf /`)
- Overwriting disk partitions (`dd if=/dev/zero of=/dev/sda`)
- Creating system instability (fork bombs)
- Exposing sensitive data
- Installing unwanted software

Your mission: Create a security guardian that blocks genuinely dangerous operations while allowing legitimate development work to proceed.

## Prerequisites

- Completed Exercise 2.1
- Understanding of shell commands and their risks
- Familiarity with pattern matching
- Basic security awareness

## Security Philosophy

**Important Principles:**

1. **Defense in Depth**: Hooks are ONE layer of security, not the only one
2. **False Positives Are OK**: Better to block a legitimate command and explain than allow a dangerous one
3. **Education Over Obstruction**: Error messages should teach, not just block
4. **Graduated Responses**: Some commands deserve warnings, not blocks

## Step-by-Step Instructions

### Step 1: Create the Basic Security Sentinel

```bash
cd .clinerules/hooks

cat > PreToolUse << 'EOF'
#!/usr/bin/env bash

input=$(cat)

tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

# Only check command execution
if [[ "$tool_name" == "execute_command" ]]; then
    command=$(echo "$input" | jq -r '.preToolUse.parameters.command')
    
    # Critical danger patterns - BLOCK IMMEDIATELY
    dangerous_patterns=(
        "rm -rf /"
        "rm -rf /*"
        "rm -rf ~"
        "rm -rf *"
        "> /dev/sda"
        "mkfs"
        "dd if=/dev/zero"
        ":(){ :|:& };:"  # fork bomb
        "chmod -R 777 /"
        "chown -R"
    )
    
    # Check each pattern
    for pattern in "${dangerous_patterns[@]}"; do
        if [[ "$command" == *"$pattern"* ]]; then
            echo '{
              "cancel": true,
              "contextModification": "",
              "errorMessage": "⛔ SECURITY BLOCK\n\nThis command was blocked for safety:\n'"$command"'\n\nDangerous pattern detected: '"$pattern"'\n\nThis operation could cause serious system damage."
            }' | jq -c '.'
            exit 0
        fi
    done
fi

echo '{
  "cancel": false,
  "contextModification": ""
}'
EOF

chmod +x PreToolUse
```

### Step 2: Add Warning Level for Sudo

Not all risky commands should be blocked. Some need review:

```bash
cat > PreToolUse << 'EOF'
#!/usr/bin/env bash

input=$(cat)

tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

if [[ "$tool_name" == "execute_command" ]]; then
    command=$(echo "$input" | jq -r '.preToolUse.parameters.command')
    
    # Critical patterns - BLOCK
    dangerous_patterns=(
        "rm -rf /"
        "rm -rf /*"
        "rm -rf ~"
        "> /dev/sda"
        "mkfs"
        "dd if=/dev/zero"
        ":(){ :|:& };:"
        "chmod -R 777 /"
    )
    
    for pattern in "${dangerous_patterns[@]}"; do
        if [[ "$command" == *"$pattern"* ]]; then
            echo '{
              "cancel": true,
              "contextModification": "",
              "errorMessage": "⛔ SECURITY BLOCK: '"$pattern"'\n\nCommand blocked: '"$command"'"
            }' | jq -c '.'
            exit 0
        fi
    done
    
    # Warning patterns - ALLOW but WARN
    if [[ "$command" == sudo* ]]; then
        echo '{
          "cancel": false,
          "contextModification": "⚠️ SECURITY WARNING: A sudo command was executed:\n'"$command"'\n\nThis command ran with elevated privileges. Review the action carefully."
        }' | jq -c '.'
        exit 0
    fi
    
    # Check for other elevated privilege commands
    if [[ "$command" == su* ]] || [[ "$command" == doas* ]]; then
        echo '{
          "cancel": false,
          "contextModification": "⚠️ SECURITY WARNING: Privilege elevation command executed: '"$command"'"
        }' | jq -c '.'
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

### Step 3: Add Network Security Checks

```bash
cat > PreToolUse << 'EOF'
#!/usr/bin/env bash

input=$(cat)

tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

if [[ "$tool_name" == "execute_command" ]]; then
    command=$(echo "$input" | jq -r '.preToolUse.parameters.command')
    
    # Critical patterns - BLOCK
    dangerous_patterns=(
        "rm -rf /"
        "rm -rf /*"
        "rm -rf ~"
        "> /dev/sda"
        "mkfs"
        "dd if=/dev/zero"
        ":(){ :|:& };:"
        "chmod -R 777 /"
    )
    
    for pattern in "${dangerous_patterns[@]}"; do
        if [[ "$command" == *"$pattern"* ]]; then
            echo '{
              "cancel": true,
              "contextModification": "",
              "errorMessage": "⛔ SECURITY BLOCK\n\nDangerous pattern: '"$pattern"'\nCommand: '"$command"'\n\nThis operation could cause serious system damage and was blocked automatically."
            }' | jq -c '.'
            exit 0
        fi
    done
    
    # Network operations that might expose services
    if [[ "$command" == *"nc -l"* ]] || [[ "$command" == *"netcat -l"* ]]; then
        echo '{
          "cancel": false,
          "contextModification": "⚠️ SECURITY: Network listener started with command: '"$command"'\n\nThis opens a port on your system. Ensure this is intentional."
        }' | jq -c '.'
        exit 0
    fi
    
    # Downloading and executing scripts
    if [[ "$command" == *"curl"*"| bash"* ]] || [[ "$command" == *"wget"*"| bash"* ]] || [[ "$command" == *"curl"*"| sh"* ]]; then
        echo '{
          "cancel": true,
          "contextModification": "",
          "errorMessage": "⛔ SECURITY BLOCK\n\nBlocked: Piping download to shell\nCommand: '"$command"'\n\nDownloading and executing scripts without inspection is dangerous.\n\nSafer approach:\n1. Download: curl -o script.sh <URL>\n2. Review: cat script.sh\n3. Execute: bash script.sh"
        }' | jq -c '.'
        exit 0
    fi
    
    # Sudo operations
    if [[ "$command" == sudo* ]]; then
        echo '{
          "cancel": false,
          "contextModification": "⚠️ SECURITY: Sudo command executed: '"$command"'"
        }' | jq -c '.'
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

### Step 4: Add File System Protection

```bash
cat > PreToolUse << 'EOF'
#!/usr/bin/env bash

input=$(cat)

tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

if [[ "$tool_name" == "execute_command" ]]; then
    command=$(echo "$input" | jq -r '.preToolUse.parameters.command')
    
    # Log all commands for audit trail
    echo "[$(date -Iseconds)] $command" >> ~/.cline-security-audit.log
    
    # === CRITICAL BLOCKS ===
    
    dangerous_patterns=(
        "rm -rf /"
        "rm -rf /*"
        "rm -rf ~"
        "rm -rf *"
        "> /dev/sda"
        "> /dev/sd"
        "mkfs"
        "dd if=/dev/zero"
        "dd if=/dev/random"
        ":(){ :|:& };:"
        "chmod -R 777 /"
        "chmod 777 /etc"
    )
    
    for pattern in "${dangerous_patterns[@]}"; do
        if [[ "$command" == *"$pattern"* ]]; then
            echo "[BLOCKED] $command" >> ~/.cline-security-audit.log
            echo '{
              "cancel": true,
              "contextModification": "",
              "errorMessage": "⛔ SECURITY BLOCK\n\n🚨 Critical danger pattern detected: '"$pattern"'\n\nCommand blocked: '"$command"'\n\nThis command could:\n- Delete critical system files\n- Corrupt disk partitions\n- Crash your system\n- Cause data loss\n\nIf you believe this is incorrect, review the command carefully and consider alternative approaches."
            }' | jq -c '.'
            exit 0
        fi
    done
    
    # === DANGEROUS FILE OPERATIONS ===
    
    # Protect critical system directories
    protected_dirs=(
        "/etc"
        "/usr"
        "/bin"
        "/sbin"
        "/boot"
        "/sys"
        "/proc"
    )
    
    for dir in "${protected_dirs[@]}"; do
        if [[ "$command" == *"rm -rf $dir"* ]] || [[ "$command" == *"rm -r $dir"* ]]; then
            echo '[BLOCKED] Protected directory deletion' >> ~/.cline-security-audit.log
            echo '{
              "cancel": true,
              "contextModification": "",
              "errorMessage": "⛔ SECURITY BLOCK\n\n🔒 Protected system directory: '"$dir"'\n\nCommand blocked: '"$command"'\n\nDeleting system directories can break your OS."
            }' | jq -c '.'
            exit 0
        fi
    done
    
    # === NETWORK SECURITY ===
    
    # Block download-and-execute patterns
    if [[ "$command" == *"curl"*"| bash"* ]] || [[ "$command" == *"wget"*"| sh"* ]] || [[ "$command" == *"curl"*"| sh"* ]]; then
        echo '[BLOCKED] Download and execute' >> ~/.cline-security-audit.log
        echo '{
          "cancel": true,
          "contextModification": "",
          "errorMessage": "⛔ SECURITY BLOCK\n\n🌐 Blocked: Download and execute pattern\n\nCommand: '"$command"'\n\nThis pattern is risky because:\n- You cannot review the script before execution\n- The script could be malicious\n- The source could be compromised\n\nSafer approach:\n1. curl -o script.sh <URL>\n2. cat script.sh  # Review it\n3. bash script.sh"
        }' | jq -c '.'
        exit 0
    fi
    
    # === WARNINGS (Allow but notify) ===
    
    # Sudo usage
    if [[ "$command" == sudo* ]]; then
        echo '[WARN] Sudo used' >> ~/.cline-security-audit.log
        echo '{
          "cancel": false,
          "contextModification": "⚠️ SECURITY WARNING: Sudo command executed with elevated privileges:\n\n'"$command"'\n\nReview this action carefully."
        }' | jq -c '.'
        exit 0
    fi
    
    # Package installation
    if [[ "$command" == *"apt install"* ]] || [[ "$command" == *"npm install -g"* ]] || [[ "$command" == *"pip install"* ]] || [[ "$command" == *"brew install"* ]]; then
        echo '[WARN] Package installation' >> ~/.cline-security-audit.log
        echo '{
          "cancel": false,
          "contextModification": "⚠️ NOTICE: Installing packages:\n\n'"$command"'\n\nVerify that these packages are from trusted sources."
        }' | jq -c '.'
        exit 0
    fi
    
    # Network listeners
    if [[ "$command" == *"nc -l"* ]] || [[ "$command" == *"netcat -l"* ]] || [[ "$command" == *"python -m http.server"* ]]; then
        echo '[WARN] Network listener' >> ~/.cline-security-audit.log
        echo '{
          "cancel": false,
          "contextModification": "⚠️ SECURITY: Opening network port:\n\n'"$command"'\n\nThis makes your system accessible over the network."
        }' | jq -c '.'
        exit 0
    fi
    
    # SSH/SCP operations
    if [[ "$command" == ssh* ]] || [[ "$command" == scp* ]]; then
        echo '[INFO] SSH/SCP used' >> ~/.cline-security-audit.log
        echo '{
          "cancel": false,
          "contextModification": "ℹ️ Network operation: '"$command"'"
        }' | jq -c '.'
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

### Step 5: Create Security Audit Viewer

```bash
cat > view-security-audit.sh << 'EOF'
#!/usr/bin/env bash

audit_log="$HOME/.cline-security-audit.log"

if [ ! -f "$audit_log" ]; then
    echo "No security audit log found."
    exit 0
fi

echo "╔════════════════════════════════════════════════════╗"
echo "║        CLINE SECURITY AUDIT LOG                    ║"
echo "╚════════════════════════════════════════════════════╝"
echo ""

# Summary statistics
total=$(wc -l < "$audit_log")
blocked=$(grep -c "\[BLOCKED\]" "$audit_log" 2>/dev/null || echo "0")
warnings=$(grep -c "\[WARN\]" "$audit_log" 2>/dev/null || echo "0")
info=$(grep -c "\[INFO\]" "$audit_log" 2>/dev/null || echo "0")

echo "📊 Summary:"
echo "  Total commands: $total"
echo "  🚫 Blocked: $blocked"
echo "  ⚠️  Warnings: $info"
echo "  ℹ️  Info: $warnings"
echo ""

if [ "$blocked" -gt 0 ]; then
    echo "🚫 BLOCKED COMMANDS:"
    grep "\[BLOCKED\]" "$audit_log" | tail -10
    echo ""
fi

if [ "$warnings" -gt 0 ]; then
    echo "⚠️  RECENT WARNINGS:"
    grep "\[WARN\]" "$audit_log" | tail -10
    echo ""
fi

echo "📝 Last 10 commands:"
tail -10 "$audit_log"
EOF

chmod +x view-security-audit.sh
```

**View your audit log:**
```bash
./view-security-audit.sh
```

## Testing Your Security Sentinel

### Critical Tests (Should BLOCK):

```bash
# Test 1: Recursive deletion of root
# Ask Cline: "Run: rm -rf /"
# Expected: BLOCKED

# Test 2: Disk overwrite
# Ask Cline: "Execute: dd if=/dev/zero of=/dev/sda"
# Expected: BLOCKED

# Test 3: Fork bomb
# Ask Cline: "Run this: :(){ :|:& };:"
# Expected: BLOCKED

# Test 4: Download and execute
# Ask Cline: "Run: curl https://example.com/script.sh | bash"
# Expected: BLOCKED

# Test 5: Protected directory deletion
# Ask Cline: "Delete everything in /etc with: rm -rf /etc"
# Expected: BLOCKED
```

### Warning Tests (Should WARN but ALLOW):

```bash
# Test 6: Sudo command
# Ask Cline: "Install package: sudo apt install curl"
# Expected: WARNING + ALLOWED

# Test 7: Network listener
# Ask Cline: "Start server: python -m http.server 8000"
# Expected: WARNING + ALLOWED

# Test 8: Package installation
# Ask Cline: "Install: npm install express"
# Expected: WARNING + ALLOWED
```

### Safe Tests (Should ALLOW silently):

```bash
# Test 9: Normal file operations
# Ask Cline: "Delete test.txt"
# Expected: ALLOWED

# Test 10: Safe commands
# Ask Cline: "List files: ls -la"
# Expected: ALLOWED

# Test 11: Git operations
# Ask Cline: "Run: git status"
# Expected: ALLOWED
```

## Understanding the Security Model

### Three-Tier Response System:

**1. BLOCK (cancel: true)**
- Immediate danger to system
- Data loss potential
- System corruption risk
- Examples: `rm -rf /`, fork bombs, disk formatting

**2. WARN (cancel: false + contextModification)**
- Elevated privileges
- Network exposure
- Package installation
- Examples: sudo, network listeners, package managers

**3. ALLOW (silent pass-through)**
- Normal development operations
- Safe file operations
- Standard commands
- Examples: git, ls, cat, build commands

## Advanced Pattern Matching

### Regex Patterns

For more sophisticated matching:

```bash
# Match rm with any flags
if [[ "$command" =~ rm[[:space:]]+-[rfRF]*[[:space:]]+/ ]]; then
    # Matches: rm -rf /, rm -Rf /, rm -r /, rm -f /
fi

# Match commands with output redirection to devices
if [[ "$command" =~ >[[:space:]]*/dev/(sd|hd|nvme) ]]; then
    # Matches: > /dev/sda, > /dev/nvme0n1
fi
```

### Context-Aware Checks

```bash
# Check command context
workspace=$(echo "$input" | jq -r '.workspaceRoots[0]')

# Only block rm -rf * if NOT in a test directory
if [[ "$command" == *"rm -rf *"* ]] && [[ "$workspace" != *"/test"* ]]; then
    # Block it
fi
```

## Creating a Security Configuration File

```bash
cat > .clinerules/security-config.json << 'EOF'
{
  "blocklist": {
    "patterns": [
      "rm -rf /",
      "mkfs",
      "dd if=/dev/zero",
      ":(){ :|:& };:"
    ],
    "protectedPaths": [
      "/etc",
      "/usr",
      "/bin",
      "/sbin"
    ]
  },
  "warnlist": {
    "patterns": [
      "sudo",
      "curl.*| bash",
      "nc -l"
    ]
  },
  "allowedSudoCommands": [
    "sudo npm install",
    "sudo apt update"
  ]
}
EOF
```

Load it in your hook:

```bash
config_file=".clinerules/security-config.json"
if [ -f "$config_file" ]; then
    blocklist=$(jq -r '.blocklist.patterns[]' "$config_file")
fi
```

## Common Issues and Solutions

### Issue: Too many false positives

**Solution:** Add more specific patterns:
```bash
# Instead of blocking all "rm"
# Block only dangerous combinations
if [[ "$command" == *"rm -rf /"* ]]; then
```

### Issue: Legitimate commands getting blocked

**Solution:** Add to allowlist:
```bash
# Allow specific sudo commands
allowed_sudo=(
    "sudo npm install"
    "sudo apt update"
)

for allowed in "${allowed_sudo[@]}"; do
    if [[ "$command" == "$allowed"* ]]; then
        # Allow it
        echo '{"cancel": false}' | jq -c '.'
        exit 0
    fi
done
```

### Issue: Not catching variations

**Solution:** Use more flexible patterns:
```bash
# Catch multiple spacing variations
if [[ "$command" =~ rm[[:space:]]+-[rfRF]+[[:space:]]+/ ]]; then
    # This catches: rm -rf /, rm  -rf   /, rm -Rf /
fi
```

## Validation Checklist

- [ ] Blocks `rm -rf /` and variations
- [ ] Blocks disk formatting commands
- [ ] Blocks fork bombs
- [ ] Blocks download-and-execute
- [ ] Warns on sudo usage
- [ ] Warns on network operations
- [ ] Allows normal development commands
- [ ] Audit log working
- [ ] Error messages are clear
- [ ] All tests pass

## What You Learned

✅ **Security Patterns**: Identifying dangerous command patterns  
✅ **Graduated Responses**: Block vs warn vs allow  
✅ **Pattern Matching**: Complex string matching in bash  
✅ **Audit Logging**: Tracking security events  
✅ **Defense in Depth**: Multi-layered protection  
✅ **User Education**: Teaching through error messages  

## Challenge Extensions

### Challenge 1: Machine Learning Detector

Use command frequency to detect anomalies:

```bash
# Track command patterns
echo "$command" >> ~/.cline-command-history.txt

# If a command appears rarely, warn
count=$(grep -c "^$command$" ~/.cline-command-history.txt)
if [ "$count" -eq 1 ]; then
    # First time seeing this command - extra scrutiny
fi
```

### Challenge 2: Environment-Based Rules

Different rules for production vs development:

```bash
if [ -f ".env" ] && grep -q "ENVIRONMENT=production" ".env"; then
    # Stricter rules for production
    if [[ "$command" == *"npm install"* ]]; then
        echo '{"cancel": true, "errorMessage": "No npm install in production"}' | jq -c '.'
        exit 0
    fi
fi
```

### Challenge 3: Time-Based Restrictions

Block dangerous operations during off-hours:

```bash
hour=$(date +%H)
if [ "$hour" -ge 22 ] || [ "$hour" -le 6 ]; then
    # After 10 PM or before 6 AM
    if [[ "$command" == sudo* ]]; then
        echo '{"cancel": true, "errorMessage": "Sudo blocked during off-hours"}' | jq -c '.'
        exit 0
    fi
fi
```

### Challenge 4: Notification Integration

Send alerts to Slack/email when dangerous commands are blocked:

```bash
if [ "$SLACK_WEBHOOK" ]; then
    curl -X POST "$SLACK_WEBHOOK" \
        -H 'Content-Type: application/json' \
        -d '{"text":"🚨 Dangerous command blocked: '"$command"'"}' \
        2>/dev/null &
fi
```

## Next Steps

You've built a comprehensive security system! You now understand how to protect against dangerous operations while allowing legitimate work.

**Next Exercise:** [Exercise 3.1: Project Detective](exercise-3.1-project-detective.md)

You'll learn context injection to help Cline make better decisions automatically.

## Reference

**Hook Type:** PreToolUse  
**Can Block:** Yes ✅  
**Timing:** Before command execution  
**Use Case:** Security enforcement  

**Critical Patterns to Block:**
- `rm -rf /` and variations
- Disk operations (`mkfs`, `dd`)
- Fork bombs
- Download-and-execute patterns
- System directory deletion

**Files Created:**
- `.clinerules/hooks/PreToolUse` - Security sentinel
- `~/.cline-security-audit.log` - Audit trail
- `view-security-audit.sh` - Audit viewer
- `.clinerules/security-config.json` - Configuration
