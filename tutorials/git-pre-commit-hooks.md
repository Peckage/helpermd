# How to Set Up Git Pre-commit Hooks

## Overview
Pre-commit hooks are scripts that run automatically before each commit, helping you catch issues early and maintain code quality.

## What Are Pre-commit Hooks?

Git hooks are custom scripts that Git executes before or after events like commit, push, and merge. Pre-commit hooks run before the commit is finalized, allowing you to:

- Lint code for style issues
- Run tests
- Check for secrets or sensitive data
- Format code automatically
- Validate commit messages

## Method 1: Manual Setup (Simple)

### Step 1: Create Hook Script
```bash
# Navigate to your repository
cd /path/to/your/repo

# Create the pre-commit hook
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash

echo "Running pre-commit checks..."

# Run linter
npm run lint
if [ $? -ne 0 ]; then
    echo "❌ Linting failed. Please fix errors before committing."
    exit 1
fi

# Run tests
npm test
if [ $? -ne 0 ]; then
    echo "❌ Tests failed. Please fix tests before committing."
    exit 1
fi

echo "✅ All checks passed!"
exit 0
EOF

# Make it executable
chmod +x .git/hooks/pre-commit
```

### Step 2: Test the Hook
```bash
# Try to make a commit
git add .
git commit -m "Test commit"
# Hook will run automatically
```

## Method 2: Using pre-commit Framework (Recommended)

### Why Use pre-commit?
- Multi-language support
- Shareable configurations
- Large library of existing hooks
- Easy to manage and version control

### Installation
```bash
# Using pip
pip install pre-commit

# Using Homebrew (macOS)
brew install pre-commit

# Using conda
conda install -c conda-forge pre-commit
```

### Step 1: Create Configuration File
Create `.pre-commit-config.yaml` in your repository root:

```yaml
# .pre-commit-config.yaml
repos:
  # General file checks
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-added-large-files
        args: ['--maxkb=500']
      - id: check-merge-conflict
      - id: detect-private-key

  # Python-specific
  - repo: https://github.com/psf/black
    rev: 23.12.1
    hooks:
      - id: black
        language_version: python3

  - repo: https://github.com/pycqa/flake8
    rev: 7.0.0
    hooks:
      - id: flake8
        args: ['--max-line-length=88']

  # JavaScript/TypeScript
  - repo: https://github.com/pre-commit/mirrors-eslint
    rev: v8.56.0
    hooks:
      - id: eslint
        files: \.[jt]sx?$
        types: [file]
        
  # Prettier for formatting
  - repo: https://github.com/pre-commit/mirrors-prettier
    rev: v3.1.0
    hooks:
      - id: prettier
        files: \.(js|jsx|ts|tsx|json|css|md)$
```

### Step 2: Install Git Hook Scripts
```bash
pre-commit install
```

### Step 3: Run Against All Files (Optional)
```bash
# Run hooks on all files (useful for first-time setup)
pre-commit run --all-files
```

### Step 4: Update Hooks
```bash
# Update hooks to latest versions
pre-commit autoupdate
```

## Common Hook Examples

### Check for Debug Statements
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: debug-statements  # Prevent Python debugger imports
```

### Prevent Commits to Main Branch
```bash
#!/bin/bash
# .git/hooks/pre-commit

branch="$(git rev-parse --abbrev-ref HEAD)"

if [ "$branch" = "main" ] || [ "$branch" = "master" ]; then
  echo "❌ Direct commits to $branch are not allowed!"
  echo "Please create a feature branch."
  exit 1
fi
```

### Check Commit Message Format
```bash
#!/bin/bash
# .git/hooks/commit-msg

commit_msg=$(cat "$1")

# Require conventional commit format
if ! echo "$commit_msg" | grep -qE "^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .+"; then
    echo "❌ Invalid commit message format!"
    echo "Format: <type>: <description>"
    echo "Example: feat: add user authentication"
    exit 1
fi
```

### Secret Detection
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

## Language-Specific Examples

### Python Project
```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      
  - repo: https://github.com/psf/black
    rev: 23.12.1
    hooks:
      - id: black
      
  - repo: https://github.com/pycqa/isort
    rev: 5.13.2
    hooks:
      - id: isort
      
  - repo: https://github.com/pycqa/flake8
    rev: 7.0.0
    hooks:
      - id: flake8
```

### JavaScript/Node.js Project
```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-json
      
  - repo: https://github.com/pre-commit/mirrors-eslint
    rev: v8.56.0
    hooks:
      - id: eslint
        additional_dependencies:
          - eslint@8.56.0
          - eslint-config-airbnb@19.0.4
          
  - repo: https://github.com/pre-commit/mirrors-prettier
    rev: v3.1.0
    hooks:
      - id: prettier
```

## Skipping Hooks

### Skip All Hooks for One Commit
```bash
git commit --no-verify -m "Emergency fix"
# or
git commit -n -m "Emergency fix"
```

### Skip Specific Hook
```bash
SKIP=flake8 git commit -m "Skip flake8 only"
```

### Disable Pre-commit Temporarily
```bash
pre-commit uninstall
# Make commits...
pre-commit install  # Re-enable
```

## Best Practices

- ✅ Keep hooks fast (< 10 seconds)
- ✅ Version control your `.pre-commit-config.yaml`
- ✅ Share hooks with team
- ✅ Run hooks in CI/CD as well
- ✅ Document any skip reasons
- ❌ Don't skip hooks without good reason
- ❌ Don't make hooks too strict initially

## Troubleshooting

### Hook Not Running
```bash
# Check if hooks are installed
ls -la .git/hooks/

# Reinstall hooks
pre-commit install
```

### Permission Denied
```bash
# Make hook executable
chmod +x .git/hooks/pre-commit
```

### Hook Fails on CI
```bash
# Run pre-commit in CI
# .github/workflows/ci.yml
- name: Run pre-commit
  run: |
    pip install pre-commit
    pre-commit run --all-files
```

## Resources
- [pre-commit framework](https://pre-commit.com/)
- [Git Hooks Documentation](https://git-scm.com/docs/githooks)
- [Supported Hooks List](https://pre-commit.com/hooks.html)
