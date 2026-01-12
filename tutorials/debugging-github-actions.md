# How to Debug GitHub Actions Workflows

## Overview
Debugging GitHub Actions can be challenging since workflows run remotely. This guide covers techniques and tools to effectively debug your workflows.

## Enable Debug Logging

### Method 1: Repository Secrets (Recommended)
1. Go to **Settings** → **Secrets and variables** → **Actions**
2. Add these secrets:
   - `ACTIONS_STEP_DEBUG` = `true` (verbose logging)
   - `ACTIONS_RUNNER_DEBUG` = `true` (runner diagnostic logging)

### Method 2: Re-run with Debug Logging
1. Navigate to failed workflow run
2. Click **Re-run jobs** dropdown
3. Select **Re-run jobs with debug logging**

### View Debug Logs
Debug logs appear with `::debug::` prefix in workflow output.

## Common Debugging Techniques

### 1. Print Environment Information
```yaml
steps:
  - name: Debug - Print environment
    run: |
      echo "Event: ${{ github.event_name }}"
      echo "Ref: ${{ github.ref }}"
      echo "Actor: ${{ github.actor }}"
      echo "Repository: ${{ github.repository }}"
      echo "Working directory: $(pwd)"
      echo "Runner OS: ${{ runner.os }}"
```

### 2. Print All Environment Variables
```yaml
steps:
  - name: Debug - Environment variables
    run: env | sort
```

### 3. Check Context Values
```yaml
steps:
  - name: Debug - GitHub context
    run: echo '${{ toJSON(github) }}'
    
  - name: Debug - Job context
    run: echo '${{ toJSON(job) }}'
    
  - name: Debug - Steps context
    run: echo '${{ toJSON(steps) }}'
    
  - name: Debug - Runner context
    run: echo '${{ toJSON(runner) }}'
```

### 4. List Files and Permissions
```yaml
steps:
  - name: Debug - List files
    run: |
      ls -la
      pwd
      find . -type f -name "*.yml" -o -name "*.yaml"
```

### 5. Check Installed Tools
```yaml
steps:
  - name: Debug - Tool versions
    run: |
      echo "Git: $(git --version)"
      echo "Node: $(node --version)"
      echo "NPM: $(npm --version)"
      echo "Python: $(python --version)"
      echo "Docker: $(docker --version)"
```

## Interactive Debugging

### Using tmate (SSH into Runner)
```yaml
name: Debug with tmate
on: [workflow_dispatch]

jobs:
  debug:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup tmate session
        uses: mxschmitt/action-tmate@v3
        timeout-minutes: 30
        # Session will be available for 30 minutes
```

**Access the session:**
1. Go to workflow run
2. Find SSH connection string in logs
3. Use the provided command to SSH into the runner

### Using debug-via-ssh
```yaml
- name: Start SSH session
  uses: luchihoratiu/debug-via-ssh@main
  with:
    NGROK_AUTH_TOKEN: ${{ secrets.NGROK_AUTH_TOKEN }}
    SSH_PASS: ${{ secrets.SSH_PASS }}
```

## Debugging Specific Issues

### Checkout Problems
```yaml
steps:
  - name: Debug - Before checkout
    run: ls -la
    
  - uses: actions/checkout@v3
  
  - name: Debug - After checkout
    run: |
      ls -la
      git status
      git log --oneline -5
```

### Permission Issues
```yaml
steps:
  - name: Debug - File permissions
    run: |
      ls -la /path/to/file
      stat /path/to/file
      
  - name: Fix permissions if needed
    run: chmod +x /path/to/file
```

### Path Issues
```yaml
steps:
  - name: Debug - PATH variable
    run: echo $PATH
    
  - name: Debug - Find executable
    run: which node || echo "node not found in PATH"
    
  - name: Add to PATH
    run: echo "/custom/path" >> $GITHUB_PATH
```

### Secret Issues
```yaml
steps:
  - name: Debug - Check if secret exists
    run: |
      if [ -z "${{ secrets.MY_SECRET }}" ]; then
        echo "Secret is empty or not set"
        exit 1
      else
        echo "Secret is set"
      fi
```

### Dependency Issues
```yaml
steps:
  - name: Debug - NPM dependencies
    run: |
      npm list --depth=0
      cat package.json
      cat package-lock.json
```

## Conditional Debugging

### Debug Only on Failure
```yaml
steps:
  - name: Run tests
    id: tests
    run: npm test
    
  - name: Debug on failure
    if: failure()
    run: |
      echo "Tests failed, debugging..."
      cat test-results.log
      env | sort
```

### Debug Specific Branch
```yaml
steps:
  - name: Debug (main branch only)
    if: github.ref == 'refs/heads/main'
    run: echo "Debugging main branch"
```

### Debug for Specific Actor
```yaml
steps:
  - name: Debug (for specific user)
    if: github.actor == 'your-username'
    run: |
      echo "Extra debugging for your-username"
      env
```

## Log Grouping
```yaml
steps:
  - name: Grouped logs
    run: |
      echo "::group::System Information"
      uname -a
      echo "::endgroup::"
      
      echo "::group::Environment Variables"
      env | sort
      echo "::endgroup::"
      
      echo "::group::Installed Packages"
      npm list --depth=0
      echo "::endgroup::"
```

## Annotations

### Error Annotation
```yaml
steps:
  - name: Custom error
    run: |
      echo "::error file=app.js,line=10,col=5::Missing semicolon"
```

### Warning Annotation
```yaml
steps:
  - name: Custom warning
    run: |
      echo "::warning file=README.md,line=1::Title should be capitalized"
```

### Notice Annotation
```yaml
steps:
  - name: Custom notice
    run: |
      echo "::notice::Build completed successfully"
```

## Debugging Matrix Builds

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node: [14, 16, 18]
    
steps:
  - name: Debug matrix
    run: |
      echo "OS: ${{ matrix.os }}"
      echo "Node: ${{ matrix.node }}"
      echo "Job index: ${{ strategy.job-index }}"
```

## Artifact Debugging

### Save Debug Artifacts
```yaml
steps:
  - name: Run process
    run: ./script.sh > output.log 2>&1
    continue-on-error: true
    
  - name: Upload logs
    if: always()
    uses: actions/upload-artifact@v3
    with:
      name: debug-logs
      path: |
        output.log
        *.log
        debug/
```

### Download and Inspect
```bash
# Using GitHub CLI
gh run download <run-id>

# Or download from UI
# Go to workflow run → Artifacts → Download
```

## Common Issues and Solutions

### Issue: Workflow Doesn't Trigger
**Debug:**
```yaml
# Add workflow_dispatch for manual testing
on:
  push:
    branches: [main]
  workflow_dispatch:  # Add this
```

### Issue: Step Fails Silently
**Debug:**
```yaml
steps:
  - name: Problematic step
    run: |
      set -x  # Enable command echoing
      ./script.sh
```

### Issue: Different Behavior Locally vs CI
**Debug:**
```yaml
steps:
  - name: Compare environments
    run: |
      echo "=== Local vs CI Comparison ==="
      echo "OS: $(uname -a)"
      echo "Shell: $SHELL"
      echo "User: $(whoami)"
      echo "Home: $HOME"
      echo "Workspace: ${{ github.workspace }}"
```

### Issue: Timeout
**Debug:**
```yaml
steps:
  - name: Long running task
    timeout-minutes: 10  # Add timeout
    run: |
      echo "Starting task at $(date)"
      ./long-task.sh
      echo "Finished task at $(date)"
```

## Best Practices

### Create Reusable Debug Action
```yaml
# .github/actions/debug/action.yml
name: Debug Environment
description: Print debug information

runs:
  using: composite
  steps:
    - name: System info
      shell: bash
      run: |
        echo "::group::System Information"
        uname -a
        echo "::endgroup::"
        
    - name: Environment
      shell: bash
      run: |
        echo "::group::Environment Variables"
        env | sort
        echo "::endgroup::"
```

**Use it:**
```yaml
steps:
  - uses: ./.github/actions/debug
```

### Use Continue-on-error for Debugging
```yaml
steps:
  - name: Experimental step
    continue-on-error: true
    run: ./experimental-script.sh
    
  - name: Debug if previous failed
    if: failure()
    run: echo "Experimental step failed, continuing anyway"
```

### Set Up Slack/Discord Notifications
```yaml
- name: Notify on failure
  if: failure()
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

## Tools and Extensions

### GitHub CLI
```bash
# View workflow runs
gh run list

# View specific run
gh run view <run-id>

# View run logs
gh run view <run-id> --log

# Watch run in real-time
gh run watch <run-id>
```

### act (Run Actions Locally)
```bash
# Install act
brew install act

# Run workflow locally
act push

# Run specific job
act -j test

# Use specific runner image
act -P ubuntu-latest=nektos/act-environments-ubuntu:18.04
```

### nektos/act for Local Testing
```bash
# Run all workflows
act

# Run with secrets
act -s GITHUB_TOKEN=your_token

# Dry run
act -n
```

## Resources
- [Workflow commands](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions)
- [Debugging workflows](https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/enabling-debug-logging)
- [act - Local GitHub Actions](https://github.com/nektos/act)
