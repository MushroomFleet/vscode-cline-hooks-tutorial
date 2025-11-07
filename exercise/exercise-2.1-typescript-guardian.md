# Exercise 2.1: "TypeScript Guardian" - Preventing Wrong File Types

**Module:** Validation & Enforcement  
**Duration:** 25 minutes  
**Hook Type:** PreToolUse  
**Difficulty:** ⭐⭐ Intermediate

## Learning Objectives

By completing this exercise, you will:
- Implement blocking logic using `cancel: true`
- Create file extension validation rules
- Provide helpful error messages to users
- Understand conditional hook execution
- Build confidence in enforcement patterns

## Scenario

You're working on a TypeScript project and want to maintain consistency. The team has agreed that all new code should use TypeScript (`.ts`, `.tsx`), not JavaScript (`.js`, `.jsx`). However, Cline sometimes creates `.js` files, especially when working with legacy examples or when not given clear guidance.

Your job: Build a hook that automatically prevents JavaScript files from being created and guides Cline to use TypeScript instead.

## Prerequisites

- Completed Exercises 1.1 and 1.2
- Understanding of `cancel: true` response
- Basic knowledge of file extensions
- Familiarity with bash string matching

## Why This Pattern Matters

This exercise teaches one of the most powerful hook patterns: **preventive validation**. Instead of fixing problems after they happen, you stop them before they occur. This pattern applies to:

- File type enforcement
- Naming convention validation
- Security policy enforcement
- Project structure rules

## Step-by-Step Instructions

### Step 1: Create the Basic Guardian Hook

```bash
cd .clinerules/hooks

cat > PreToolUse << 'EOF'
#!/usr/bin/env bash

input=$(cat)

# Extract tool information
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

# Only check file write operations
if [[ "$tool_name" == "write_to_file" ]]; then
    # Extract the file path
    file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
    
    # Check if it's a .js file (but not .json)
    if [[ "$file_path" == *.js ]] && [[ "$file_path" != *.json ]]; then
        echo '{
          "cancel": true,
          "contextModification": "",
          "errorMessage": "🚫 JavaScript files (.js) are not allowed in this TypeScript project.\n\nPlease use .ts extension instead.\n\nBlocked file: '"$file_path"'\nSuggested: '"${file_path%.js}.ts"'"
        }' | jq -c '.'
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

### Step 2: Understand the Blocking Mechanism

Let's break down the key parts:

**Conditional Check:**
```bash
if [[ "$tool_name" == "write_to_file" ]]; then
```
Only inspect file write operations. Other tools pass through without checks.

**Extension Matching:**
```bash
if [[ "$file_path" == *.js ]] && [[ "$file_path" != *.json ]]; then
```
- `*.js` matches files ending in `.js`
- `&& [[ "$file_path" != *.json ]]` excludes `.json` files (they should be allowed!)

**The Block Response:**
```json
{
  "cancel": true,
  "contextModification": "",
  "errorMessage": "Your message here"
}
```
- `"cancel": true` - This STOPS the operation
- `errorMessage` - Shown to the user explaining why

**Path Transformation:**
```bash
"${file_path%.js}.ts"
```
This bash parameter expansion removes `.js` and adds `.ts`, suggesting the correct filename.

### Step 3: Test the Guardian

#### Test 1: Try to Create a JavaScript File

In Cline, ask:
```
Create a new file called utils.js with a simple helper function
```

**Expected Result:**
- Cline attempts to write `utils.js`
- Your hook blocks it
- Error message appears in Cline
- File is NOT created

**What You Should See:**
```
🚫 JavaScript files (.js) are not allowed in this TypeScript project.

Please use .ts extension instead.

Blocked file: utils.js
Suggested: utils.ts
```

#### Test 2: Verify TypeScript Files Work

In Cline, ask:
```
Create a new file called utils.ts with a simple helper function
```

**Expected Result:**
- File is created successfully
- No blocking occurs
- Cline proceeds normally

#### Test 3: Ensure JSON Files Pass Through

In Cline, ask:
```
Create a package.json file
```

**Expected Result:**
- File is created successfully
- Hook doesn't block `.json` files
- Everything works normally

### Step 4: Enhance with JSX Detection

Now let's also block `.jsx` files:

```bash
cat > PreToolUse << 'EOF'
#!/usr/bin/env bash

input=$(cat)

tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

if [[ "$tool_name" == "write_to_file" ]]; then
    file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
    
    # Check for .js files (excluding .json)
    if [[ "$file_path" == *.js ]] && [[ "$file_path" != *.json ]]; then
        echo '{
          "cancel": true,
          "contextModification": "",
          "errorMessage": "🚫 JavaScript files (.js) are not allowed.\n\nUse .ts instead: '"${file_path%.js}.ts"'"
        }' | jq -c '.'
        exit 0
    fi
    
    # Check for .jsx files
    if [[ "$file_path" == *.jsx ]]; then
        echo '{
          "cancel": true,
          "contextModification": "",
          "errorMessage": "🚫 JSX files (.jsx) are not allowed.\n\nUse .tsx instead: '"${file_path%.jsx}.tsx"'"
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

### Step 5: Add Exception Handling

Sometimes you need to allow JavaScript in specific directories (like `scripts/` or `config/`):

```bash
cat > PreToolUse << 'EOF'
#!/usr/bin/env bash

input=$(cat)

tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')

if [[ "$tool_name" == "write_to_file" ]]; then
    file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
    
    # Define allowed directories for JavaScript
    allowed_js_dirs=("scripts/" "config/" "webpack/" ".eslintrc.js" "jest.config.js")
    
    # Check if path matches any allowed directory or config file
    is_allowed=false
    for allowed in "${allowed_js_dirs[@]}"; do
        if [[ "$file_path" == *"$allowed"* ]]; then
            is_allowed=true
            break
        fi
    done
    
    # If not allowed and is a .js file (not .json), block it
    if [[ "$is_allowed" == false ]] && [[ "$file_path" == *.js ]] && [[ "$file_path" != *.json ]]; then
        echo '{
          "cancel": true,
          "contextModification": "",
          "errorMessage": "🚫 JavaScript files not allowed outside scripts/, config/, or config files.\n\nBlocked: '"$file_path"'\nUse .ts instead: '"${file_path%.js}.ts"'\n\nAllowed JS locations:\n- scripts/\n- config/\n- webpack/\n- Root config files (*.config.js, .eslintrc.js)"
        }' | jq -c '.'
        exit 0
    fi
    
    # Same for .jsx
    if [[ "$is_allowed" == false ]] && [[ "$file_path" == *.jsx ]]; then
        echo '{
          "cancel": true,
          "contextModification": "",
          "errorMessage": "🚫 JSX files not allowed.\n\nUse .tsx instead: '"${file_path%.jsx}.tsx"'"
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

### Step 6: Add Logging for Tracking

Let's track both blocked and allowed operations:

```bash
cat > PreToolUse << 'EOF'
#!/usr/bin/env bash

input=$(cat)

tool_name=$(echo "$input" | jq -r '.preToolUse.toolName')
timestamp=$(date -Iseconds)

if [[ "$tool_name" == "write_to_file" ]]; then
    file_path=$(echo "$input" | jq -r '.preToolUse.parameters.path')
    
    # Check for .js files (excluding .json)
    if [[ "$file_path" == *.js ]] && [[ "$file_path" != *.json ]]; then
        # Log the block
        echo "$timestamp | BLOCKED | $file_path" >> ~/cline-ts-guardian.log
        
        echo '{
          "cancel": true,
          "contextModification": "",
          "errorMessage": "🚫 JavaScript files (.js) are not allowed.\n\nUse .ts instead: '"${file_path%.js}.ts"'"
        }' | jq -c '.'
        exit 0
    fi
    
    # Check for .jsx files
    if [[ "$file_path" == *.jsx ]]; then
        echo "$timestamp | BLOCKED | $file_path" >> ~/cline-ts-guardian.log
        
        echo '{
          "cancel": true,
          "contextModification": "",
          "errorMessage": "🚫 JSX files (.jsx) are not allowed.\n\nUse .tsx instead: '"${file_path%.jsx}.tsx"'"
        }' | jq -c '.'
        exit 0
    fi
    
    # Log allowed TypeScript files
    if [[ "$file_path" == *.ts ]] || [[ "$file_path" == *.tsx ]]; then
        echo "$timestamp | ALLOWED | $file_path" >> ~/cline-ts-guardian.log
    fi
fi

echo '{
  "cancel": false,
  "contextModification": ""
}'
EOF

chmod +x PreToolUse
```

**View the log:**
```bash
cat ~/cline-ts-guardian.log

# Example output:
# 2025-11-07T14:23:15+00:00 | BLOCKED | src/utils.js
# 2025-11-07T14:23:45+00:00 | ALLOWED | src/utils.ts
# 2025-11-07T14:24:10+00:00 | BLOCKED | components/Button.jsx
# 2025-11-07T14:24:30+00:00 | ALLOWED | components/Button.tsx
```

## Understanding Error Messages

Good error messages should:

1. **Explain what was blocked:**
   ```
   🚫 JavaScript files (.js) are not allowed.
   ```

2. **Provide the reason:**
   ```
   This is a TypeScript project.
   ```

3. **Suggest a solution:**
   ```
   Use .ts instead: utils.ts
   ```

4. **Be actionable:**
   - Show the exact blocked path
   - Provide the corrected filename
   - Explain any exceptions

**Bad error message:**
```
Error: Invalid file type
```

**Good error message:**
```
🚫 JavaScript files (.js) are not allowed in this TypeScript project.

Please use .ts extension instead.

Blocked file: src/utils.js
Suggested: src/utils.ts
```

## Testing Your Hook Thoroughly

### Test Suite:

```bash
# Test 1: Block .js
# Ask Cline: "Create index.js"
# Expected: BLOCKED

# Test 2: Block .jsx
# Ask Cline: "Create App.jsx"
# Expected: BLOCKED

# Test 3: Allow .ts
# Ask Cline: "Create utils.ts"
# Expected: ALLOWED

# Test 4: Allow .tsx
# Ask Cline: "Create Button.tsx"
# Expected: ALLOWED

# Test 5: Allow .json
# Ask Cline: "Create data.json"
# Expected: ALLOWED

# Test 6: Allow .js in scripts/
# Ask Cline: "Create scripts/build.js"
# Expected: ALLOWED (if you have exceptions)

# Test 7: Allow config files
# Ask Cline: "Create jest.config.js"
# Expected: ALLOWED (if you have exceptions)
```

### Automated Test Script

Create a test script:

```bash
cat > test-guardian.sh << 'EOF'
#!/usr/bin/env bash

echo "Testing TypeScript Guardian Hook"
echo "================================"

test_hook() {
    local file_path="$1"
    local expected="$2"
    
    # Create test input
    test_input=$(cat <<JSON
{
  "clineVersion": "3.1.0",
  "hookName": "PreToolUse",
  "timestamp": "2025-11-07T10:00:00Z",
  "taskId": "test-123",
  "workspaceRoots": ["/test"],
  "userId": "test-user",
  "preToolUse": {
    "toolName": "write_to_file",
    "parameters": {
      "path": "$file_path",
      "content": "test content"
    }
  }
}
JSON
)
    
    # Run the hook
    result=$(echo "$test_input" | .clinerules/hooks/PreToolUse)
    is_blocked=$(echo "$result" | jq -r '.cancel')
    
    # Check result
    if [[ "$is_blocked" == "$expected" ]]; then
        echo "✅ PASS: $file_path (blocked=$is_blocked)"
    else
        echo "❌ FAIL: $file_path (expected blocked=$expected, got blocked=$is_blocked)"
    fi
}

# Run tests
test_hook "utils.js" "true"
test_hook "App.jsx" "true"
test_hook "utils.ts" "false"
test_hook "Button.tsx" "false"
test_hook "data.json" "false"
test_hook "package.json" "false"

echo ""
echo "Test complete!"
EOF

chmod +x test-guardian.sh
./test-guardian.sh
```

## Common Issues and Solutions

### Issue: Hook blocks everything

**Diagnosis:**
```bash
# Check the logic
cat .clinerules/hooks/PreToolUse
```

**Cause:** Condition is too broad or missing proper checks.

**Solution:** Make sure you have:
```bash
if [[ "$tool_name" == "write_to_file" ]]; then
    if [[ "$file_path" == *.js ]] && [[ "$file_path" != *.json ]]; then
        # Block only here
    fi
fi
```

### Issue: JSON files get blocked

**Cause:** Not excluding `.json` from `.js` check.

**Solution:** Always use:
```bash
if [[ "$file_path" == *.js ]] && [[ "$file_path" != *.json ]]; then
```

### Issue: Error message not showing

**Cause:** JSON formatting issue or missing `jq -c` flag.

**Solution:**
```bash
# Use jq -c for compact JSON
echo '{"cancel": true, "errorMessage": "text"}' | jq -c '.'
```

### Issue: Hook allows .js files

**Check:**
1. Is hook executable? `ls -l .clinerules/hooks/PreToolUse`
2. Is "Enable Hooks" checked in Cline settings?
3. Test manually: `echo '{"preToolUse":{"toolName":"write_to_file","parameters":{"path":"test.js"}}}' | .clinerules/hooks/PreToolUse`

### Issue: Path transformations not working

**Debug:**
```bash
# Test bash parameter expansion
file_path="utils.js"
echo "${file_path%.js}.ts"
# Should output: utils.ts
```

## Validation Checklist

- [ ] Hook blocks `.js` files (except `.json`)
- [ ] Hook blocks `.jsx` files
- [ ] Hook allows `.ts` files
- [ ] Hook allows `.tsx` files
- [ ] Hook allows `.json` files
- [ ] Error messages are clear and helpful
- [ ] Suggested filenames are correct
- [ ] Exceptions work (if implemented)
- [ ] Logging works (if implemented)
- [ ] All tests pass

## What You Learned

✅ **Blocking Operations**: Using `cancel: true` to prevent actions  
✅ **Pattern Matching**: File extension checks with bash  
✅ **Error Messages**: Creating helpful user feedback  
✅ **Conditional Logic**: Tool-specific validation  
✅ **Exception Handling**: Allowing specific cases  
✅ **String Manipulation**: Path transformations  
✅ **Testing**: Validating hook behavior

## Challenge Extensions

### Challenge 1: Configuration File

Create a `.ts-guardian.json` config file:

```json
{
  "blocked": [".js", ".jsx"],
  "allowed": [".ts", ".tsx"],
  "exceptions": ["scripts/", "config/"],
  "message": "This project uses TypeScript"
}
```

Load it in your hook:
```bash
config=$(cat .ts-guardian.json 2>/dev/null || echo '{}')
blocked=$(echo "$config" | jq -r '.blocked[]')
```

### Challenge 2: Auto-Convert Mode

Instead of just blocking, offer to automatically fix:

```bash
if [[ "$file_path" == *.js ]]; then
    fixed_path="${file_path%.js}.ts"
    echo "{
      \"cancel\": true,
      \"contextModification\": \"USER_FEEDBACK: File $file_path was blocked. Please create $fixed_path instead.\",
      \"errorMessage\": \"...\"
    }" | jq -c '.'
fi
```

### Challenge 3: File Content Analysis

Check if the content is actually TypeScript:

```bash
content=$(echo "$input" | jq -r '.preToolUse.parameters.content')

# Look for TypeScript indicators
if echo "$content" | grep -q ": string\|: number\|interface \|type "; then
    # Content looks like TypeScript
    if [[ "$file_path" == *.js ]]; then
        # TypeScript content in .js file - block it!
    fi
fi
```

### Challenge 4: Statistics Dashboard

Create a summary command:

```bash
cat > show-guardian-stats.sh << 'EOF'
#!/usr/bin/env bash

echo "TypeScript Guardian Statistics"
echo "=============================="
echo ""

total=$(wc -l < ~/cline-ts-guardian.log)
blocked=$(grep "BLOCKED" ~/cline-ts-guardian.log | wc -l)
allowed=$(grep "ALLOWED" ~/cline-ts-guardian.log | wc -l)

echo "Total operations: $total"
echo "Blocked: $blocked"
echo "Allowed: $allowed"
echo ""
echo "Most blocked files:"
grep "BLOCKED" ~/cline-ts-guardian.log | awk '{print $NF}' | sort | uniq -c | sort -rn | head -5
EOF

chmod +x show-guardian-stats.sh
```

## Next Steps

Congratulations! You've built a real validation hook that enforces project standards.

**Next Exercise:** [Exercise 2.2: Security Sentinel](exercise-2.2-security-sentinel.md)

You'll build a hook that blocks dangerous shell commands to protect your system.

## Reference

**Hook Type:** PreToolUse  
**Can Block:** Yes ✅  
**Timing:** Before file write  
**Use Case:** File type enforcement  

**Key Patterns:**
```bash
# Check tool type
if [[ "$tool_name" == "write_to_file" ]]; then

# Check extension
if [[ "$file_path" == *.js ]]; then

# Exclude patterns
if [[ "$file_path" != *.json ]]; then

# Block operation
echo '{"cancel": true, "errorMessage": "..."}'

# Allow operation
echo '{"cancel": false}'
```

**Files Created:**
- `.clinerules/hooks/PreToolUse` - The guardian hook
- `~/cline-ts-guardian.log` - Block/allow log
- `test-guardian.sh` - Test script
