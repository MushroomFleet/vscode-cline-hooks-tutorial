# Module 3: Context Injection & Learning

**Duration:** 2-3 hours | **Difficulty:** Intermediate

## 🎯 Learning Objectives

By the end of this module, you will:

- ✅ Understand context timing and how it affects AI decisions
- ✅ Build hooks that learn from operations
- ✅ Use PostToolUse effectively for dynamic learning
- ✅ Create intelligent, project-aware context
- ✅ Implement performance monitoring
- ✅ Build cumulative knowledge systems

## 📖 Theory: Context is King

### Understanding Context Timing

This is the most important concept in hooks:

**Context affects FUTURE decisions, not the current one.**

When a hook runs:
1. ✅ The AI has already decided what to do
2. ✅ Your hook can block or allow it (PreToolUse)
3. ✅ Your hook can inject context
4. ✅ The NEXT AI request sees that context
5. ❌ The current operation doesn't see the context

### Context Flow Example

```
User: "Create a new component"
    ↓
AI Decision: write_to_file component.js
    ↓
PreToolUse Hook: (can block, adds context about TypeScript)
    ↓
Operation executes (or is blocked)
    ↓
PostToolUse Hook: (adds context about what just happened)
    ↓
NEXT AI Decision: (sees all the context from hooks)
```

### When to Use Which Hook

**TaskStart** - Set the stage
- Detect project type
- Establish conventions
- Initialize tracking
- Context: Shapes entire task

**PreToolUse** - Prevent and guide
- Block bad operations
- Warn before risky actions
- Context: Affects next tool use

**PostToolUse** - Learn and adapt
- Learn from results
- Track patterns
- Build knowledge
- Context: Affects subsequent decisions

### Context Best Practices

**Be specific and actionable**
```bash
# ❌ Vague
"contextModification": "This is important"

# ✅ Specific
"contextModification": "WORKSPACE_RULES: This Next.js project uses the app/ directory structure. Place all routes under app/ not pages/."
```

**Use prefixes for organization**
```bash
"WORKSPACE_RULES: ..."    # Project conventions
"PERFORMANCE: ..."         # Speed and optimization  
"SECURITY: ..."           # Security concerns
"ERROR_PATTERN: ..."      # Common mistakes
"BEST_PRACTICE: ..."      # Recommendations
```

**Keep it concise**
- Context has a 50KB limit
- Be clear but brief
- Focus on actionable guidance

## 🔍 Exercise 3.1: "Project Detective" - Auto-Discovering Tech Stack

**Duration:** 35 minutes  
**Hook Type:** TaskStart  
**Difficulty:** ⭐⭐ Intermediate

### Scenario

Every project is different. Instead of manually telling Cline about your tech stack, build a hook that automatically detects it and injects appropriate guidance. This makes Cline immediately aware of project conventions.

### What You'll Build

An intelligent TaskStart hook that:
- Detects programming languages
- Identifies frameworks
- Finds configuration files
- Injects contextual rules
- Adapts to project type

### Real-World Use Case

This is valuable for:
- Onboarding Cline to new projects
- Ensuring consistent practices
- Reducing initial setup
- Team knowledge sharing
- Multi-project workflows

### Step-by-Step Instructions

#### 1. Create the Project Detective

```bash
cd .clinerules/hooks

# Back up existing TaskStart if needed
mv TaskStart TaskStart.backup

touch TaskStart
code TaskStart
```

#### 2. Write the Detection Script

```bash
#!/usr/bin/env bash

# Project Detective Hook
# Automatically detect tech stack and inject project context

input=$(cat)

# Extract workspace path
workspace=$(echo "$input" | jq -r '.workspaceRoots[0]')

# If workspace is null or doesn't exist, skip detection
if [ -z "$workspace" ] || [ ! -d "$workspace" ]; then
    echo '{"cancel": false, "contextModification": ""}'; exit 0
fi

# Initialize context string
context=""

# ═══════════════════════════════════════════════════════════
# DETECT JAVASCRIPT / TYPESCRIPT PROJECTS
# ═══════════════════════════════════════════════════════════

if [ -f "$workspace/package.json" ]; then
    # Read package.json
    package_json=$(cat "$workspace/package.json")
    
    # Base detection
    context="WORKSPACE_RULES: This is a Node.js project."
    
    # Check for TypeScript
    if echo "$package_json" | grep -q "\"typescript\""; then
        has_tsconfig=false
        if [ -f "$workspace/tsconfig.json" ]; then
            has_tsconfig=true
        fi
        
        if [ "$has_tsconfig" = true ]; then
            context="$context Use TypeScript (.ts/.tsx) for all new files. Follow TypeScript best practices and include proper type annotations."
        fi
    fi
    
    # Check for React
    if echo "$package_json" | grep -q "\"react\""; then
        context="$context This project uses React. Use functional components with hooks. Follow React best practices including proper key props, memo optimization where appropriate, and accessibility standards."
    fi
    
    # Check for Next.js
    if echo "$package_json" | grep -q "\"next\""; then
        # Detect Next.js version / structure
        if [ -d "$workspace/app" ]; then
            context="$context This is a Next.js 13+ project using App Router. Place routes in app/ directory. Use server components by default and mark client components with 'use client'. Utilize server actions for mutations."
        elif [ -d "$workspace/pages" ]; then
            context="$context This is a Next.js project using Pages Router. Place routes in pages/ directory. Use getServerSideProps or getStaticProps for data fetching."
        fi
    fi
    
    # Check for Vue
    if echo "$package_json" | grep -q "\"vue\""; then
        context="$context This project uses Vue.js. Follow Vue composition API patterns. Use <script setup> syntax."
    fi
    
    # Check for testing frameworks
    if echo "$package_json" | grep -q "\"jest\""; then
        context="$context Jest is configured for testing. Write tests in *.test.ts or *.spec.ts files."
    fi
    
    if echo "$package_json" | grep -q "\"vitest\""; then
        context="$context Vitest is configured for testing. Write tests in *.test.ts files."
    fi
    
    # Check for linters
    if echo "$package_json" | grep -q "\"eslint\""; then
        context="$context ESLint is configured. Follow linting rules defined in the project."
    fi
    
    if echo "$package_json" | grep -q "\"prettier\""; then
        context="$context Prettier is configured for code formatting. Format code according to .prettierrc settings."
    fi
fi

# ═══════════════════════════════════════════════════════════
# DETECT PYTHON PROJECTS
# ═══════════════════════════════════════════════════════════

if [ -f "$workspace/pyproject.toml" ] || [ -f "$workspace/setup.py" ] || [ -f "$workspace/requirements.txt" ]; then
    context="WORKSPACE_RULES: This is a Python project. Follow PEP 8 style guidelines. Use type hints for function parameters and return values."
    
    # Check for common frameworks
    if [ -f "$workspace/pyproject.toml" ]; then
        pyproject=$(cat "$workspace/pyproject.toml")
        
        if echo "$pyproject" | grep -q "django"; then
            context="$context This is a Django project. Follow Django conventions: apps for modular functionality, models in models.py, views in views.py, and URLs in urls.py."
        fi
        
        if echo "$pyproject" | grep -q "fastapi"; then
            context="$context This is a FastAPI project. Use Pydantic models for request/response validation. Utilize dependency injection and async endpoints where appropriate."
        fi
        
        if echo "$pyproject" | grep -q "flask"; then
            context="$context This is a Flask project. Use blueprints for modular routes. Follow Flask application factory pattern."
        fi
    fi
    
    # Check for pytest
    if [ -f "$workspace/pytest.ini" ] || grep -q "pytest" "$workspace/pyproject.toml" 2>/dev/null; then
        context="$context Pytest is configured for testing. Write tests in test_*.py files."
    fi
    
    # Check for Poetry
    if [ -f "$workspace/poetry.lock" ]; then
        context="$context Poetry is used for dependency management. Use 'poetry add' for dependencies."
    fi
fi

# ═══════════════════════════════════════════════════════════
# DETECT RUST PROJECTS
# ═══════════════════════════════════════════════════════════

if [ -f "$workspace/Cargo.toml" ]; then
    context="WORKSPACE_RULES: This is a Rust project. Follow Rust idioms and conventions. Use cargo for all build and dependency operations. Ensure code compiles without warnings. Use proper error handling with Result types."
    
    # Check for workspace
    if grep -q "\[workspace\]" "$workspace/Cargo.toml"; then
        context="$context This is a Cargo workspace with multiple crates. Consider dependencies and build ordering."
    fi
fi

# ═══════════════════════════════════════════════════════════
# DETECT GO PROJECTS
# ═══════════════════════════════════════════════════════════

if [ -f "$workspace/go.mod" ]; then
    context="WORKSPACE_RULES: This is a Go project. Follow Go conventions: use gofmt for formatting, organize code in packages, use proper error handling. Run go mod tidy after adding dependencies."
fi

# ═══════════════════════════════════════════════════════════
# DETECT MONOREPO STRUCTURES
# ═══════════════════════════════════════════════════════════

if [ -f "$workspace/lerna.json" ]; then
    context="$context This is a Lerna monorepo. Be aware of package dependencies and workspace structure."
fi

if [ -f "$workspace/nx.json" ]; then
    context="$context This is an Nx monorepo. Use Nx commands for builds and tests. Respect project boundaries."
fi

if [ -f "$workspace/pnpm-workspace.yaml" ]; then
    context="$context This is a pnpm workspace. Use pnpm for all package management operations."
fi

# ═══════════════════════════════════════════════════════════
# DETECT BUILD TOOLS
# ═══════════════════════════════════════════════════════════

if [ -f "$workspace/vite.config.ts" ] || [ -f "$workspace/vite.config.js" ]; then
    context="$context Vite is configured as the build tool. Development server uses Vite's fast HMR."
fi

if [ -f "$workspace/webpack.config.js" ]; then
    context="$context Webpack is configured as the build tool."
fi

# ═══════════════════════════════════════════════════════════
# DETECT DOCUMENTATION
# ═══════════════════════════════════════════════════════════

if [ -f "$workspace/docs" ]; then
    context="$context This project has a docs/ directory. Maintain documentation for new features and API changes."
fi

# ═══════════════════════════════════════════════════════════
# DETECT CI/CD
# ═══════════════════════════════════════════════════════════

if [ -f "$workspace/.github/workflows" ]; then
    context="$context GitHub Actions is configured for CI/CD. Ensure changes pass CI checks."
fi

if [ -f "$workspace/.gitlab-ci.yml" ]; then
    context="$context GitLab CI is configured. Ensure pipeline passes."
fi

# ═══════════════════════════════════════════════════════════
# LOG DETECTION RESULTS
# ═══════════════════════════════════════════════════════════

echo "Project Detective Results:" >&2
if [ -n "$context" ]; then
    echo "✅ Detected project type and injected context" >&2
    echo "$context" | head -c 200 >&2
    echo "..." >&2
else
    echo "ℹ️  No specific project type detected" >&2
fi

# Return context
echo "{
  \"cancel\": false,
  \"contextModification\": \"$context\"
}"
```

#### 3. Make Executable

```bash
chmod +x TaskStart
```

#### 4. Test the Detective

Create a test project structure:

```bash
# Create a test Next.js project structure
mkdir -p ~/test-nextjs-project
cd ~/test-nextjs-project

# Create package.json
cat > package.json << 'EOF'
{
  "name": "test-project",
  "dependencies": {
    "react": "^18.0.0",
    "next": "^14.0.0",
    "typescript": "^5.0.0"
  },
  "devDependencies": {
    "eslint": "^8.0.0",
    "prettier": "^3.0.0",
    "jest": "^29.0.0"
  }
}
EOF

# Create tsconfig.json
touch tsconfig.json

# Create app directory (Next.js 13+ indicator)
mkdir app

# Now test the hook
cat << 'EOF' | ~/cline-hooks-practice/.clinerules/hooks/TaskStart
{
  "clineVersion": "1.0.0",
  "hookName": "TaskStart",
  "timestamp": "2025-11-06T10:30:00Z",
  "taskId": "test-123",
  "workspaceRoots": ["/Users/yourusername/test-nextjs-project"],
  "userId": "user-001",
  "taskStart": {
    "taskMetadata": {
      "taskId": "test-123",
      "ulid": "01H",
      "initialTask": "Add a new feature"
    }
  }
}
EOF
```

**Expected Output:**
You should see context mentioning:
- Node.js project
- TypeScript usage
- React with functional components
- Next.js 13+ with App Router
- ESLint and Prettier
- Jest for testing

#### 5. Test with Real Projects

Open different projects in VSCode and start Cline tasks. Check VSCode Output panel to see what the detective discovers about each project.

### Understanding the Detection Logic

Key patterns used:

```bash
# File existence check
if [ -f "$workspace/package.json" ]; then
    # File exists
fi

# Directory existence check
if [ -d "$workspace/app" ]; then
    # Directory exists
fi

# Content search with grep
if echo "$package_json" | grep -q "\"react\""; then
    # Found "react" in content
    # -q means quiet (no output, just exit code)
fi

# Multiple conditions
if [ -f "$workspace/pyproject.toml" ] || [ -f "$workspace/setup.py" ]; then
    # Either file exists
fi
```

### Enhancement: User Preferences

Allow developers to add personal preferences:

```bash
# Check for user preference file
user_prefs="$HOME/.cline-project-prefs.json"

if [ -f "$user_prefs" ]; then
    # Load user preferences
    prefs=$(cat "$user_prefs")
    
    # Extract custom rules
    custom_rules=$(echo "$prefs" | jq -r '.customRules // empty')
    
    if [ -n "$custom_rules" ]; then
        context="$context $custom_rules"
    fi
fi
```

Create preferences file:
```bash
cat > ~/.cline-project-prefs.json << 'EOF'
{
  "customRules": "PERSONAL_PREFERENCE: I prefer arrow functions over function declarations. Use const for all variables unless reassignment is needed. Add JSDoc comments for exported functions.",
  "strictMode": true
}
EOF
```

### Challenge Tasks

Enhance your detective to:

1. **Version Detection**
   - Check Node.js version from `.nvmrc`
   - Suggest using correct versions

2. **Dependency Analysis**
   - Detect outdated dependencies
   - Warn about security vulnerabilities

3. **Coding Standards Detection**
   - Parse `.prettierrc` for formatting rules
   - Read `.eslintrc` for linting config
   - Inject specific rules into context

### Key Takeaways

✅ **TaskStart shapes the entire task** - Perfect for project detection  
✅ **File system inspection is powerful** - Check files and directories  
✅ **Layer detections** - Build context progressively  
✅ **Be specific** - Generic rules don't help  
✅ **Log discoveries** - Use stderr for debugging

---

## 📊 Exercise 3.2: "Performance Tracker" - Learning from Operation Times

**Duration:** 30 minutes  
**Hook Type:** PostToolUse  
**Difficulty:** ⭐⭐ Intermediate

### Scenario

Operations that take too long can indicate problems: inefficient commands, large file operations, or expensive computations. Build a hook that learns from execution times and provides performance guidance.

### What You'll Build

A PostToolUse hook that:
- Tracks operation execution times
- Identifies slow operations
- Injects performance guidance
- Builds historical data
- Suggests optimizations

### Real-World Use Case

This helps with:
- Identifying bottlenecks
- Optimizing workflows
- Catching inefficient patterns
- Performance regression detection
- Developer awareness

### Step-by-Step Instructions

#### 1. Create the Performance Tracker

```bash
cd .clinerules/hooks

# Back up existing PostToolUse if needed
mv PostToolUse PostToolUse.backup

touch PostToolUse
code PostToolUse
```

#### 2. Write the Tracker Script

```bash
#!/usr/bin/env bash

# Performance Tracker Hook
# Monitor operation execution times and provide optimization guidance

input=$(cat)

# Extract performance data
tool_name=$(echo "$input" | jq -r '.postToolUse.toolName')
exec_time=$(echo "$input" | jq -r '.postToolUse.executionTimeMs')
success=$(echo "$input" | jq -r '.postToolUse.success')
task_id=$(echo "$input" | jq -r '.taskId')

# Only track successful operations
if [ "$success" != "true" ]; then
    echo '{"cancel": false, "contextModification": ""}'; exit 0
fi

# ═══════════════════════════════════════════════════════════
# LOG TO PERFORMANCE DATABASE
# ═══════════════════════════════════════════════════════════

perf_log="/tmp/cline-performance.jsonl"
timestamp=$(date -Iseconds)

# Create log entry
log_entry=$(echo "$input" | jq -c '{
  timestamp: "'"$timestamp"'",
  taskId: .taskId,
  tool: .postToolUse.toolName,
  executionMs: .postToolUse.executionTimeMs,
  parameters: .postToolUse.parameters
}')

echo "$log_entry" >> "$perf_log"

# ═══════════════════════════════════════════════════════════
# DEFINE PERFORMANCE THRESHOLDS
# ═══════════════════════════════════════════════════════════

# Thresholds in milliseconds
SLOW_THRESHOLD=5000      # 5 seconds
VERY_SLOW_THRESHOLD=10000  # 10 seconds

# Initialize context
context=""

# ═══════════════════════════════════════════════════════════
# ANALYZE SPECIFIC TOOL PERFORMANCE
# ═══════════════════════════════════════════════════════════

if [ "$exec_time" -gt "$VERY_SLOW_THRESHOLD" ]; then
    # Very slow operation
    exec_seconds=$((exec_time / 1000))
    
    case "$tool_name" in
        "execute_command")
            command=$(echo "$input" | jq -r '.postToolUse.parameters.command')
            context="PERFORMANCE: ⚠️ Command took ${exec_seconds}s to execute: \`$command\`

Optimization suggestions:
• Break into smaller, incremental steps
• Consider background execution for long-running tasks
• Check if operation can be made more efficient
• Consider caching results if operation repeats"
            ;;
            
        "write_to_file")
            file_path=$(echo "$input" | jq -r '.postToolUse.parameters.path')
            content_size=$(echo "$input" | jq -r '.postToolUse.parameters.content | length')
            
            context="PERFORMANCE: ⚠️ Writing to $file_path took ${exec_seconds}s

File size: ~$((content_size / 1024))KB

Optimization suggestions:
• Consider if entire file needs to be rewritten
• Break large files into smaller modules
• Use streaming for very large content
• Check for unnecessary duplication in content"
            ;;
            
        "read_file")
            file_path=$(echo "$input" | jq -r '.postToolUse.parameters.path')
            
            context="PERFORMANCE: ⚠️ Reading $file_path took ${exec_seconds}s

This indicates a very large file.

Optimization suggestions:
• Read specific sections instead of entire file
• Consider file streaming for large files
• Check if file content can be cached
• Verify file size is necessary for the task"
            ;;
            
        *)
            context="PERFORMANCE: ⚠️ Operation '$tool_name' took ${exec_seconds}s

Consider optimizing this operation or breaking it into smaller steps."
            ;;
    esac
    
elif [ "$exec_time" -gt "$SLOW_THRESHOLD" ]; then
    # Moderately slow operation
    exec_seconds=$((exec_time / 1000))
    
    case "$tool_name" in
        "execute_command")
            command=$(echo "$input" | jq -r '.postToolUse.parameters.command')
            
            # Check for specific slow patterns
            if echo "$command" | grep -q "npm install"; then
                context="PERFORMANCE: npm install took ${exec_seconds}s. This is normal for package installation. Consider using npm ci for faster installs in CI environments."
                
            elif echo "$command" | grep -q "pip install"; then
                context="PERFORMANCE: pip install took ${exec_seconds}s. Consider using pip install with --no-cache-dir in CI or checking for large dependencies."
                
            elif echo "$command" | grep -qE "(build|compile)"; then
                context="PERFORMANCE: Build/compile took ${exec_seconds}s. Consider using incremental builds or checking build configuration for optimization opportunities."
                
            else
                context="PERFORMANCE: Command took ${exec_seconds}s: \`$command\`. Monitor if this consistently slow."
            fi
            ;;
    esac
fi

# ═══════════════════════════════════════════════════════════
# PATTERN DETECTION
# ═══════════════════════════════════════════════════════════

# Check for repeated slow operations in current task
slow_operations_count=0

if [ -f "$perf_log" ]; then
    # Count operations over threshold in current task
    slow_operations_count=$(grep "\"taskId\":\"$task_id\"" "$perf_log" | \
                           jq -s "[.[] | select(.executionMs > $SLOW_THRESHOLD)] | length")
    
    # Warn if many slow operations
    if [ "$slow_operations_count" -gt 5 ]; then
        if [ -z "$context" ]; then
            context="PERFORMANCE: ⚠️ Multiple slow operations detected in this task ($slow_operations_count operations over 5s).

Consider:
• Breaking the task into smaller, focused subtasks
• Reviewing approach for efficiency
• Checking for repetitive or unnecessary operations"
        fi
    fi
fi

# ═══════════════════════════════════════════════════════════
# STATISTICS (for debugging)
# ═══════════════════════════════════════════════════════════

echo "Performance Stats:" >&2
echo "  Tool: $tool_name" >&2
echo "  Time: ${exec_time}ms" >&2

if [ "$slow_operations_count" -gt 0 ]; then
    echo "  Slow ops in task: $slow_operations_count" >&2
fi

# ═══════════════════════════════════════════════════════════
# RETURN RESPONSE
# ═══════════════════════════════════════════════════════════

echo "{
  \"cancel\": false,
  \"contextModification\": \"$context\"
}"
```

#### 3. Make Executable

```bash
chmod +x PostToolUse
```

#### 4. Test the Tracker

Simulate some operations:

```bash
# Simulate a slow command
cat << 'EOF' | ./PostToolUse
{
  "clineVersion": "1.0.0",
  "hookName": "PostToolUse",
  "timestamp": "2025-11-06T10:30:00Z",
  "taskId": "test-123",
  "workspaceRoots": ["/workspace"],
  "userId": "user-001",
  "postToolUse": {
    "toolName": "execute_command",
    "parameters": {
      "command": "npm install"
    },
    "result": "added 237 packages",
    "success": true,
    "executionTimeMs": 12500
  }
}
EOF

# Check the output - should see performance warning
```

#### 5. Analyze Performance Data

```bash
# View performance log
cat /tmp/cline-performance.jsonl | jq '.'

# Find slowest operations
cat /tmp/cline-performance.jsonl | jq -s 'sort_by(.executionMs) | reverse | .[0:10]'

# Average execution time by tool
cat /tmp/cline-performance.jsonl | jq -s 'group_by(.tool) | map({tool: .[0].tool, avgMs: (map(.executionMs) | add / length), count: length})'
```

### Building Performance Analytics

Create a helper script to analyze performance:

```bash
cat > /tmp/analyze-cline-performance.sh << 'EOF'
#!/usr/bin/env bash

perf_log="/tmp/cline-performance.jsonl"

if [ ! -f "$perf_log" ]; then
    echo "No performance data found"
    exit 1
fi

echo "═══════════════════════════════════════════"
echo "Cline Performance Analysis"
echo "═══════════════════════════════════════════"
echo ""

# Total operations
total=$(wc -l < "$perf_log")
echo "Total operations tracked: $total"
echo ""

# Average by tool
echo "Average execution time by tool:"
cat "$perf_log" | jq -s '
  group_by(.tool) | 
  map({
    tool: .[0].tool,
    avgMs: (map(.executionMs) | add / length | round),
    count: length
  }) | 
  sort_by(.avgMs) | 
  reverse | 
  .[]' | jq -r '"\(.tool): \(.avgMs)ms (n=\(.count))"'

echo ""

# Slowest operations
echo "Top 5 slowest operations:"
cat "$perf_log" | jq -s 'sort_by(.executionMs) | reverse | .[0:5]' | \
  jq -r '.[] | "\(.executionMs)ms - \(.tool) at \(.timestamp)"'

echo ""

# Operations over threshold
slow_count=$(cat "$perf_log" | jq -s '[.[] | select(.executionMs > 5000)] | length')
echo "Operations over 5s threshold: $slow_count"

EOF

chmod +x /tmp/analyze-cline-performance.sh

# Run it
/tmp/analyze-cline-performance.sh
```

### Understanding PostToolUse Data

PostToolUse provides rich information:

```json
{
  "postToolUse": {
    "toolName": "execute_command",     // What ran
    "parameters": {...},                // What was passed
    "result": "output here...",        // What it produced
    "success": true,                    // Did it succeed?
    "executionTimeMs": 1234            // How long it took
  }
}
```

### Challenge Tasks

Enhance your tracker to:

1. **Trend Analysis**
   - Track if operations are getting slower over time
   - Alert on performance regressions

2. **Tool Comparison**
   - Compare similar operations
   - Suggest faster alternatives

3. **Contextual Optimization**
   - Provide tool-specific optimization tips
   - Link to relevant documentation

4. **Performance Budget**
   - Set time budgets for different operations
   - Alert when budgets are exceeded

### Key Takeaways

✅ **PostToolUse is for learning** - Can't block, but can guide future  
✅ **Track execution times** - Performance data is valuable  
✅ **Context guides next decisions** - AI learns from feedback  
✅ **Build historical data** - JSONL format is perfect for logs  
✅ **Provide actionable advice** - Tell how to improve

---

## 🎓 Module 3 Review

### What You've Learned

- ✅ Context timing and its effects
- ✅ TaskStart for project initialization
- ✅ PostToolUse for learning patterns
- ✅ Building intelligent, adaptive hooks
- ✅ Performance monitoring techniques
- ✅ Data collection and analysis

### Practical Skills

You can now:
- Auto-detect project types
- Inject contextual guidance
- Track operation performance
- Build learning systems
- Analyze hook data
- Provide intelligent feedback

## 📊 Knowledge Check

1. **When does context from a hook take effect?**
   <details>
   <summary>Answer</summary>
   
   Context affects FUTURE AI decisions, not the current operation. The next time the AI makes a decision, it will see the context that was injected.
   </details>

2. **Which hook is best for enforcing project conventions?**
   <details>
   <summary>Answer</summary>
   
   TaskStart is ideal because it runs once at the beginning and can inject project-wide rules that shape all subsequent AI decisions throughout the task.
   </details>

3. **Why use PostToolUse instead of PreToolUse for performance tracking?**
   <details>
   <summary>Answer</summary>
   
   PostToolUse runs AFTER operations complete and includes executionTimeMs data. PreToolUse runs before execution, so timing data isn't available yet.
   </details>

4. **What format is good for performance logs?**
   <details>
   <summary>Answer</summary>
   
   JSONL (JSON Lines) - one JSON object per line. Easy to append, stream-process, and analyze with tools like jq.
   </details>

## 🎯 Module 3 Completion Checklist

- [ ] Created Project Detective hook
- [ ] Tested auto-detection on different project types
- [ ] Successfully injected project-specific context
- [ ] Created Performance Tracker hook
- [ ] Logged performance data to JSONL
- [ ] Analyzed performance with jq
- [ ] Understand context timing
- [ ] Can differentiate when to use each hook type
- [ ] Passed knowledge check

## 🚀 Next Steps

You now understand how to build learning hooks that make Cline smarter. Ready to integrate with external tools and build sophisticated workflows?

**[Module 4: Advanced Integrations →](module-04-integrations.md)**

Or return to the [Course Overview](README.md)
