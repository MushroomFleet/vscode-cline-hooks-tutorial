# Exercise 3.1: "Project Detective" - Auto-Discovering Tech Stack

**Module:** Context Injection & Learning  
**Duration:** 35 minutes  
**Hook Types:** TaskStart + PostToolUse  
**Difficulty:** ⭐⭐ Intermediate

## Learning Objectives

By completing this exercise, you will:
- Understand context injection timing (affects future decisions)
- Build dynamic project knowledge from file system inspection
- Create intelligent guidance for AI based on project structure
- Master multi-file detection patterns
- Learn to combine multiple hooks for compound effects

## Scenario

When Cline starts working on a project, it doesn't automatically know:
- What programming language you're using
- What frameworks are present
- What coding standards to follow
- What file structure conventions exist

Instead of manually explaining your project every time, you'll build a hook that automatically detects these details and injects them as context, helping Cline make better decisions from the start.

## Prerequisites

- Completed Modules 1 and 2
- Understanding of `contextModification` field
- Familiarity with project structure files (package.json, requirements.txt, etc.)
- Basic file system navigation

## Understanding Context Timing

**CRITICAL CONCEPT:**

Context injection affects **FUTURE** decisions, not the current one:

```
1. AI decides to do something
2. Hook runs (can block or allow)
3. Context gets added to conversation
4. NEXT AI request sees that context ← Important!
```

This means:
- `PreToolUse` blocks bad actions NOW
- `PostToolUse` teaches Cline for NEXT actions
- `TaskStart` sets up context for ALL subsequent actions

## Step-by-Step Instructions

### Step 1: Create Basic Project Detector

```bash
cd .clinerules/hooks

cat > TaskStart << 'EOF'
#!/usr/bin/env bash

input=$(cat)

# Get workspace root
workspace=$(echo "$input" | jq -r '.workspaceRoots[0]')

# Initialize context
context=""

# Detect Node.js project
if [ -f "$workspace/package.json" ]; then
    context="WORKSPACE_RULES: This is a Node.js project."
    
    # Check for specific dependencies
    if grep -q '"typescript"' "$workspace/package.json"; then
        context="$context Use TypeScript (.ts/.tsx) for all new files."
    fi
    
    if grep -q '"react"' "$workspace/package.json"; then
        context="$context This project uses React."
    fi
fi

# Detect Python project
if [ -f "$workspace/pyproject.toml" ] || [ -f "$workspace/requirements.txt" ] || [ -f "$workspace/setup.py" ]; then
    context="WORKSPACE_RULES: This is a Python project. Follow PEP 8 style guidelines. Use type hints for function signatures."
    
    # Check for specific frameworks
    if [ -f "$workspace/pyproject.toml" ] && grep -q "django" "$workspace/pyproject.toml"; then
        context="$context This is a Django project. Follow Django conventions."
    fi
    
    if grep -q "flask" "$workspace/requirements.txt" 2>/dev/null; then
        context="$context This is a Flask application."
    fi
fi

# Detect Rust project
if [ -f "$workspace/Cargo.toml" ]; then
    context="WORKSPACE_RULES: This is a Rust project. Follow Rust idioms. Use cargo for all dependency management."
fi

# Detect Go project
if [ -f "$workspace/go.mod" ]; then
    context="WORKSPACE_RULES: This is a Go project. Follow Go conventions. Use gofmt style."
fi

# Output with context
echo "{
  \"cancel\": false,
  \"contextModification\": \"$context\"
}"
EOF

chmod +x TaskStart
```

### Step 2: Test Basic Detection

Create test projects:

```bash
# Test Node.js detection
mkdir -p /tmp/test-nodejs
cd /tmp/test-nodejs
echo '{"name": "test", "dependencies": {"typescript": "^5.0.0"}}' > package.json

# Open this in VS Code and start a Cline task
# You should see TypeScript guidance appear
```

### Step 3: Enhanced Detection with Framework Details

```bash
cat > TaskStart << 'EOF'
#!/usr/bin/env bash

input=$(cat)

workspace=$(echo "$input" | jq -r '.workspaceRoots[0]')

context=""

# === NODE.JS / JAVASCRIPT / TYPESCRIPT ===
if [ -f "$workspace/package.json" ]; then
    pkg_content=$(cat "$workspace/package.json")
    
    context="WORKSPACE_RULES: This is a Node.js project."
    
    # TypeScript detection
    if echo "$pkg_content" | grep -q '"typescript"'; then
        context="$context\n\n📘 TYPESCRIPT PROJECT:\n- Use .ts/.tsx extensions for all new files\n- Include type annotations\n- Avoid 'any' types where possible"
    fi
    
    # React detection
    if echo "$pkg_content" | grep -q '"react"'; then
        context="$context\n\n⚛️  REACT PROJECT:\n- Use functional components with hooks\n- Follow component file structure\n- Use TypeScript interfaces for props"
        
        # Next.js specific
        if echo "$pkg_content" | grep -q '"next"'; then
            context="$context\n\n📦 NEXT.JS CONVENTIONS:\n- Use app/ directory structure\n- Create page.tsx for routes\n- Use layout.tsx for shared layouts\n- Follow Next.js 14+ patterns"
        fi
    fi
    
    # Vue detection
    if echo "$pkg_content" | grep -q '"vue"'; then
        context="$context\n\n💚 VUE PROJECT:\n- Use Composition API with <script setup>\n- Single File Components (.vue)\n- Follow Vue 3 conventions"
    fi
    
    # Testing frameworks
    if echo "$pkg_content" | grep -q '"jest"'; then
        context="$context\n\n🧪 TESTING: Jest is configured. Create .test.ts files alongside source files."
    fi
    
    if echo "$pkg_content" | grep -q '"vitest"'; then
        context="$context\n\n🧪 TESTING: Vitest is configured. Create .test.ts files."
    fi
    
    # Linting
    if echo "$pkg_content" | grep -q '"eslint"'; then
        context="$context\n\n✅ LINTING: ESLint is configured. Follow linting rules."
    fi
fi

# === PYTHON ===
if [ -f "$workspace/pyproject.toml" ] || [ -f "$workspace/requirements.txt" ]; then
    context="WORKSPACE_RULES: This is a Python project."
    context="$context\n\n🐍 PYTHON STANDARDS:\n- Follow PEP 8 style guide\n- Use type hints for functions\n- Docstrings for public functions\n- Use f-strings for formatting"
    
    # Django
    if [ -f "$workspace/manage.py" ]; then
        context="$context\n\n🎸 DJANGO PROJECT:\n- Follow Django app structure\n- Models in models.py\n- Views in views.py\n- URLs in urls.py\n- Use Django ORM conventions"
    fi
    
    # FastAPI
    if grep -q "fastapi" "$workspace/requirements.txt" 2>/dev/null || grep -q "fastapi" "$workspace/pyproject.toml" 2>/dev/null; then
        context="$context\n\n⚡ FASTAPI PROJECT:\n- Use async/await for endpoints\n- Pydantic models for validation\n- Type hints required\n- Follow REST conventions"
    fi
    
    # pytest
    if grep -q "pytest" "$workspace/requirements.txt" 2>/dev/null; then
        context="$context\n\n🧪 TESTING: pytest configured. Create test_*.py files."
    fi
fi

# === RUST ===
if [ -f "$workspace/Cargo.toml" ]; then
    context="WORKSPACE_RULES: This is a Rust project."
    context="$context\n\n🦀 RUST CONVENTIONS:\n- Follow Rust naming conventions (snake_case)\n- Use Result<T, E> for error handling\n- Implement Display and Debug traits\n- Write unit tests in same file\n- Use cargo fmt and clippy"
fi

# === GO ===
if [ -f "$workspace/go.mod" ]; then
    context="WORKSPACE_RULES: This is a Go project."
    context="$context\n\n🐹 GO CONVENTIONS:\n- Follow effective Go guidelines\n- Use gofmt formatting\n- Error handling: if err != nil\n- Interfaces for abstractions\n- Use go mod for dependencies"
fi

# === JAVA ===
if [ -f "$workspace/pom.xml" ] || [ -f "$workspace/build.gradle" ]; then
    context="WORKSPACE_RULES: This is a Java project."
    context="$context\n\n☕ JAVA CONVENTIONS:\n- Follow Java naming conventions\n- Use meaningful package names\n- Implement proper exception handling\n- Write Javadoc comments"
    
    if [ -f "$workspace/pom.xml" ]; then
        context="$context\n- Maven build system"
    fi
    
    if [ -f "$workspace/build.gradle" ]; then
        context="$context\n- Gradle build system"
    fi
fi

# === C/C++ ===
if [ -f "$workspace/CMakeLists.txt" ] || [ -f "$workspace/Makefile" ]; then
    context="WORKSPACE_RULES: This is a C/C++ project."
    context="$context\n\n⚙️  C/C++ CONVENTIONS:\n- Header guards in .h files\n- Proper memory management\n- const correctness\n- Follow project's style guide"
fi

# === DOCUMENTATION ===
if [ -f "$workspace/README.md" ]; then
    context="$context\n\n📚 DOCUMENTATION: README.md exists. Keep it updated with changes."
fi

if [ -d "$workspace/docs" ]; then
    context="$context\n\n📖 Documentation directory found. Add relevant docs for new features."
fi

# === CI/CD ===
if [ -f "$workspace/.github/workflows" ] || [ -d "$workspace/.github/workflows" ]; then
    context="$context\n\n🔄 CI/CD: GitHub Actions configured. Consider CI impact."
fi

if [ -f "$workspace/.gitlab-ci.yml" ]; then
    context="$context\n\n🔄 CI/CD: GitLab CI configured."
fi

# === CONTAINERIZATION ===
if [ -f "$workspace/Dockerfile" ]; then
    context="$context\n\n🐳 DOCKER: Dockerfile present. Consider containerization."
fi

if [ -f "$workspace/docker-compose.yml" ]; then
    context="$context\n\n🐳 DOCKER COMPOSE: Multi-container setup defined."
fi

# Output
echo "{
  \"cancel\": false,
  \"contextModification\": \"$context\"
}"
EOF

chmod +x TaskStart
```

### Step 4: Add Learning from File Operations

Create a PostToolUse hook that learns patterns:

```bash
cat > PostToolUse << 'EOF'
#!/usr/bin/env bash

input=$(cat)

tool_name=$(echo "$input" | jq -r '.postToolUse.toolName')
success=$(echo "$input" | jq -r '.postToolUse.success')

context=""

if [[ "$tool_name" == "write_to_file" ]] && [[ "$success" == "true" ]]; then
    file_path=$(echo "$input" | jq -r '.postToolUse.parameters.path')
    
    # Learn from file structure
    dir_name=$(dirname "$file_path")
    file_name=$(basename "$file_path")
    extension="${file_name##*.}"
    
    # Detect pattern: tests alongside source
    if [[ "$file_path" == *"test"* ]] || [[ "$file_path" == *"spec"* ]]; then
        context="PROJECT_PATTERN: Tests are co-located with source files. Continue this pattern for new code."
    fi
    
    # Detect pattern: component structure
    if [[ "$dir_name" == *"components"* ]] && [[ "$extension" == "tsx" ]]; then
        # Check if there's an index file pattern
        if [ -f "$dir_name/index.ts" ] || [ -f "$dir_name/index.tsx" ]; then
            context="PROJECT_PATTERN: Components use index files for exports. When creating new components, add to index file."
        fi
    fi
    
    # Detect pattern: API route structure
    if [[ "$dir_name" == *"api"* ]] || [[ "$dir_name" == *"routes"* ]]; then
        context="PROJECT_PATTERN: API routes detected. Follow RESTful conventions and existing route structure."
    fi
    
    # Detect pattern: utility/helper structure
    if [[ "$dir_name" == *"utils"* ]] || [[ "$dir_name" == *"helpers"* ]]; then
        context="PROJECT_PATTERN: Utility functions organized in dedicated directory. Place helper functions here."
    fi
fi

# Learn from command execution
if [[ "$tool_name" == "execute_command" ]] && [[ "$success" == "true" ]]; then
    command=$(echo "$input" | jq -r '.postToolUse.parameters.command')
    result=$(echo "$input" | jq -r '.postToolUse.result')
    
    # Learn from test commands
    if [[ "$command" == *"npm test"* ]] || [[ "$command" == *"pytest"* ]] || [[ "$command" == *"cargo test"* ]]; then
        if echo "$result" | grep -q "passed"; then
            context="✅ TESTS PASSING: Current test suite passes. Maintain this when making changes."
        elif echo "$result" | grep -q "failed"; then
            context="⚠️  TEST FAILURES: Tests are failing. Fix these before proceeding with new features."
        fi
    fi
    
    # Learn from linting
    if [[ "$command" == *"lint"* ]] || [[ "$command" == *"eslint"* ]]; then
        if echo "$result" | grep -q "error"; then
            context="⚠️  LINTING ERRORS: Address linting issues found."
        fi
    fi
fi

echo "{
  \"cancel\": false,
  \"contextModification\": \"$context\"
}"
EOF

chmod +x PostToolUse
```

### Step 5: Create Project Analysis Tool

```bash
cat > analyze-project.sh << 'EOF'
#!/usr/bin/env bash

echo "╔════════════════════════════════════════╗"
echo "║     PROJECT STRUCTURE ANALYZER         ║"
echo "╚════════════════════════════════════════╝"
echo ""

# Current directory
workspace=$(pwd)

echo "📁 Analyzing: $workspace"
echo ""

# Language detection
echo "🔍 Language Detection:"

if [ -f "package.json" ]; then
    echo "  ✓ Node.js (package.json found)"
    
    if grep -q '"typescript"' package.json; then
        echo "  ✓ TypeScript"
    fi
    
    if grep -q '"react"' package.json; then
        echo "  ✓ React"
    fi
    
    if grep -q '"next"' package.json; then
        echo "  ✓ Next.js"
    fi
fi

if [ -f "requirements.txt" ] || [ -f "pyproject.toml" ]; then
    echo "  ✓ Python"
fi

if [ -f "Cargo.toml" ]; then
    echo "  ✓ Rust"
fi

if [ -f "go.mod" ]; then
    echo "  ✓ Go"
fi

echo ""

# Project structure
echo "📂 Project Structure:"

if [ -d "src" ]; then
    echo "  ✓ src/ directory"
    
    if [ -d "src/components" ]; then
        echo "    ✓ src/components/ ($(find src/components -name '*.tsx' -o -name '*.ts' -o -name '*.jsx' -o -name '*.js' 2>/dev/null | wc -l) files)"
    fi
    
    if [ -d "src/utils" ]; then
        echo "    ✓ src/utils/ (helper functions)"
    fi
    
    if [ -d "src/api" ]; then
        echo "    ✓ src/api/ (API routes)"
    fi
fi

if [ -d "tests" ] || [ -d "test" ]; then
    test_dir="tests"
    [ -d "test" ] && test_dir="test"
    test_count=$(find "$test_dir" -name '*.test.*' -o -name '*.spec.*' 2>/dev/null | wc -l)
    echo "  ✓ $test_dir/ directory ($test_count test files)"
fi

echo ""

# Tooling detection
echo "🛠️  Tooling:"

if [ -f ".eslintrc.js" ] || [ -f ".eslintrc.json" ]; then
    echo "  ✓ ESLint (linting)"
fi

if [ -f ".prettierrc" ] || [ -f ".prettierrc.json" ]; then
    echo "  ✓ Prettier (formatting)"
fi

if [ -f "tsconfig.json" ]; then
    echo "  ✓ TypeScript config"
fi

if [ -f "jest.config.js" ] || [ -f "jest.config.ts" ]; then
    echo "  ✓ Jest (testing)"
fi

if [ -f "vitest.config.ts" ]; then
    echo "  ✓ Vitest (testing)"
fi

echo ""

# Version control
echo "📝 Version Control:"

if [ -d ".git" ]; then
    echo "  ✓ Git repository"
    branch=$(git branch --show-current 2>/dev/null)
    echo "    Current branch: $branch"
fi

if [ -f ".gitignore" ]; then
    echo "  ✓ .gitignore configured"
fi

echo ""

# CI/CD
echo "🔄 CI/CD:"

if [ -d ".github/workflows" ]; then
    workflow_count=$(find .github/workflows -name '*.yml' 2>/dev/null | wc -l)
    echo "  ✓ GitHub Actions ($workflow_count workflows)"
fi

if [ -f ".gitlab-ci.yml" ]; then
    echo "  ✓ GitLab CI"
fi

echo ""

# Containerization
echo "🐳 Containerization:"

if [ -f "Dockerfile" ]; then
    echo "  ✓ Dockerfile"
fi

if [ -f "docker-compose.yml" ]; then
    echo "  ✓ Docker Compose"
fi

echo ""
echo "Analysis complete!"
EOF

chmod +x analyze-project.sh
```

**Run the analyzer:**
```bash
./analyze-project.sh
```

## Testing the Project Detective

### Test 1: TypeScript React Next.js Project

```bash
mkdir -p /tmp/test-nextjs
cd /tmp/test-nextjs

cat > package.json << 'JSON'
{
  "name": "test-nextjs",
  "dependencies": {
    "react": "^18.0.0",
    "next": "^14.0.0",
    "typescript": "^5.0.0"
  },
  "devDependencies": {
    "eslint": "^8.0.0",
    "jest": "^29.0.0"
  }
}
JSON

echo '{"compilerOptions": {"strict": true}}' > tsconfig.json
mkdir -p src/components
```

Start a Cline task here. You should see context like:
```
WORKSPACE_RULES: This is a Node.js project.

📘 TYPESCRIPT PROJECT:
- Use .ts/.tsx extensions for all new files
- Include type annotations
- Avoid 'any' types where possible

⚛️  REACT PROJECT:
- Use functional components with hooks
- Follow component file structure
- Use TypeScript interfaces for props

📦 NEXT.JS CONVENTIONS:
- Use app/ directory structure
- Create page.tsx for routes
- Use layout.tsx for shared layouts
- Follow Next.js 14+ patterns

🧪 TESTING: Jest is configured. Create .test.ts files alongside source files.

✅ LINTING: ESLint is configured. Follow linting rules.
```

### Test 2: Python Django Project

```bash
mkdir -p /tmp/test-django
cd /tmp/test-django

cat > requirements.txt << 'TXT'
Django==4.2.0
pytest==7.0.0
TXT

touch manage.py
```

Start a Cline task. You should see:
```
WORKSPACE_RULES: This is a Python project.

🐍 PYTHON STANDARDS:
- Follow PEP 8 style guide
- Use type hints for functions
- Docstrings for public functions
- Use f-strings for formatting

🎸 DJANGO PROJECT:
- Follow Django app structure
- Models in models.py
- Views in views.py
- URLs in urls.py
- Use Django ORM conventions

🧪 TESTING: pytest configured. Create test_*.py files.
```

## Understanding Context Injection Patterns

### Pattern 1: Project Type Detection
```bash
if [ -f "$workspace/package.json" ]; then
    context="This is a Node.js project"
fi
```

### Pattern 2: Framework Detection
```bash
if grep -q '"react"' "$workspace/package.json"; then
    context="$context This uses React"
fi
```

### Pattern 3: Convention Detection
```bash
if [ -d "$workspace/src/components" ]; then
    context="$context Components in src/components/"
fi
```

### Pattern 4: Tooling Detection
```bash
if [ -f "$workspace/.eslintrc.js" ]; then
    context="$context ESLint configured"
fi
```

## Creating a Persistent Knowledge Base

Store learned patterns across sessions:

```bash
cat > save-project-knowledge.sh << 'EOF'
#!/usr/bin/env bash

knowledge_file=".clinerules/project-knowledge.json"

# Create knowledge base
cat > "$knowledge_file" << 'JSON'
{
  "detectedAt": "'$(date -Iseconds)'",
  "language": "typescript",
  "frameworks": ["react", "next.js"],
  "testFramework": "jest",
  "linter": "eslint",
  "conventions": {
    "componentLocation": "src/components",
    "testLocation": "colocated",
    "styleGuide": "airbnb"
  },
  "cicd": "github-actions",
  "containerized": true
}
JSON

echo "Project knowledge saved to $knowledge_file"
EOF

chmod +x save-project-knowledge.sh
```

## Advanced Detection Patterns

### Detect Coding Style

```bash
# Check for semicolons
if find . -name '*.ts' -o -name '*.js' | head -5 | xargs grep -q ';$'; then
    context="$context\nCODE STYLE: Project uses semicolons."
else
    context="$context\nCODE STYLE: Project does not use semicolons."
fi

# Check for single vs double quotes
if find . -name '*.ts' -o -name '*.js' | head -5 | xargs grep -q "\""; then
    context="$context Use double quotes for strings."
else
    context="$context Use single quotes for strings."
fi
```

### Detect Test Patterns

```bash
# Are tests colocated or separate?
if [ -d "tests" ] || [ -d "test" ]; then
    context="$context\nTEST PATTERN: Tests in separate directory."
elif find src -name '*.test.*' | head -1 | grep -q .; then
    context="$context\nTEST PATTERN: Tests colocated with source."
fi
```

### Detect Import Style

```bash
# Check for absolute vs relative imports
if grep -r "import.*from '\.\./" src/ | head -1 | grep -q .; then
    context="$context\nIMPORT STYLE: Uses relative imports."
elif grep -r "import.*from '@/" src/ | head -1 | grep -q .; then
    context="$context\nIMPORT STYLE: Uses absolute imports with @ alias."
fi
```

## Common Issues and Solutions

### Issue: Context too long

**Solution:** Prioritize most important info:
```bash
# Keep context under 500 characters
if [ ${#context} -gt 500 ]; then
    context=$(echo "$context" | head -c 500)
    context="$context..."
fi
```

### Issue: Detection not working

**Debug:**
```bash
# Add logging
echo "Checking for package.json..." >&2
if [ -f "$workspace/package.json" ]; then
    echo "Found package.json" >&2
else
    echo "No package.json" >&2
fi
```

### Issue: Wrong framework detected

**Solution:** Be more specific:
```bash
# Check package.json dependencies, not devDependencies
deps=$(jq -r '.dependencies | keys[]' package.json 2>/dev/null)
if echo "$deps" | grep -q "react"; then
    # React is a real dependency
fi
```

## Validation Checklist

- [ ] Detects Node.js projects
- [ ] Detects TypeScript usage
- [ ] Detects Python projects
- [ ] Detects React framework
- [ ] Detects Next.js
- [ ] Detects testing frameworks
- [ ] Detects linting tools
- [ ] Context is clear and actionable
- [ ] PostToolUse learns patterns
- [ ] Works with multiple project types

## What You Learned

✅ **Context Timing**: How context affects future decisions  
✅ **File System Inspection**: Reading project files to detect patterns  
✅ **Framework Detection**: Identifying tools and frameworks  
✅ **Pattern Learning**: Building knowledge from operations  
✅ **Multi-Hook Coordination**: TaskStart + PostToolUse working together  
✅ **Dynamic Context**: Building guidance based on project state

## Challenge Extensions

### Challenge 1: Machine Learning from History

Track what Cline does and learn from it:

```bash
# In PostToolUse
if [[ "$file_path" == src/components/* ]]; then
    # Learn: components go in src/components
    echo '{"pattern": "components", "location": "src/components"}' >> .clinerules/learned-patterns.jsonl
fi
```

### Challenge 2: Team Conventions

Detect team-specific patterns:

```bash
# Check commit messages for conventions
if git log --oneline | head -10 | grep -q "^feat:"; then
    context="$context\nCOMMIT STYLE: Use conventional commits (feat:, fix:, docs:)"
fi
```

### Challenge 3: Smart Defaults

Set defaults based on project size:

```bash
file_count=$(find . -name '*.ts' -o -name '*.js' | wc -l)
if [ "$file_count" -gt 100 ]; then
    context="$context\nLARGE PROJECT: Consider modular architecture."
fi
```

### Challenge 4: Integration with External Tools

Query package registries for best practices:

```bash
# Check npm for package info
if command -v npm &>/dev/null; then
    npm_info=$(npm view react version 2>/dev/null)
    context="$context\nREACT VERSION: Using $npm_info"
fi
```

## Next Steps

You've built an intelligent project detection system! Cline now adapts to your project automatically.

**Next Exercise:** [Exercise 3.2: Performance Tracker](exercise-3.2-performance-tracker.md)

Learn how to track performance metrics and optimize Cline's operations.

## Reference

**Hook Types:** TaskStart + PostToolUse  
**Can Block:** No  
**Timing:** Task initialization + After operations  
**Use Case:** Dynamic context injection  

**Detection Priorities:**
1. Language (package.json, requirements.txt, etc.)
2. Framework (react, django, etc.)
3. Tooling (eslint, pytest, etc.)
4. Conventions (file structure, naming)
5. Patterns (learned from operations)

**Files Created:**
- `.clinerules/hooks/TaskStart` - Initial detection
- `.clinerules/hooks/PostToolUse` - Pattern learning
- `analyze-project.sh` - Analysis tool
- `.clinerules/project-knowledge.json` - Knowledge base
