# How to Setup a GitHub Actions Bot with Proper Permissions

## Overview
This guide walks you through setting up a GitHub Actions bot with the correct permissions to perform automated tasks in your repository.

## Prerequisites
- A GitHub repository
- Admin access to repository settings
- Basic understanding of GitHub Actions

## Step 1: Configure Workflow Permissions

### Option A: Repository-Level Settings (Recommended)
1. Go to your repository on GitHub
2. Navigate to **Settings** → **Actions** → **General**
3. Scroll to **Workflow permissions**
4. Choose one of:
   - **Read repository contents and packages permissions** (default, most restrictive)
   - **Read and write permissions** (allows bot to make changes)
5. Optionally enable **Allow GitHub Actions to create and approve pull requests**
6. Click **Save**

### Option B: Workflow File Configuration
Add permissions directly in your workflow YAML file:

```yaml
name: Bot Workflow
on: [push]

permissions:
  contents: write      # Read and write to repository contents
  pull-requests: write # Create and modify PRs
  issues: write        # Create and modify issues
  checks: write        # Create check runs
  
jobs:
  bot-job:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run bot actions
        run: echo "Bot running with proper permissions"
```

## Step 2: Common Permission Scopes

| Permission | Access Level | Use Case |
|------------|-------------|----------|
| `contents: write` | Read/write repository files | Committing changes, creating tags |
| `contents: read` | Read repository files | Checking out code |
| `pull-requests: write` | Create/modify PRs | Automated PR creation, reviews |
| `issues: write` | Create/modify issues | Issue management, labeling |
| `checks: write` | Create check runs | Status checks |
| `deployments: write` | Create deployments | Deployment workflows |
| `packages: write` | Publish packages | Package registry publishing |

## Step 3: Using GITHUB_TOKEN

The `GITHUB_TOKEN` is automatically created for each workflow run:

```yaml
steps:
  - name: Create PR
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    run: |
      gh pr create --title "Automated PR" --body "Created by bot"
```

## Step 4: Custom Personal Access Token (PAT)

For actions that need elevated permissions:

1. Create a PAT:
   - Go to **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**
   - Click **Generate new token**
   - Select required scopes (e.g., `repo`, `workflow`)
   - Generate and copy the token

2. Add as repository secret:
   - Repository **Settings** → **Secrets and variables** → **Actions**
   - Click **New repository secret**
   - Name: `BOT_PAT`
   - Value: paste your token

3. Use in workflow:
```yaml
steps:
  - uses: actions/checkout@v3
    with:
      token: ${{ secrets.BOT_PAT }}
```

## Step 5: Best Practices

- ✅ **Principle of least privilege**: Only grant permissions needed for the task
- ✅ **Use job-level permissions**: Override at job level for granular control
- ✅ **Audit regularly**: Review bot actions and permissions periodically
- ✅ **Rotate tokens**: If using PATs, rotate them regularly
- ❌ **Avoid**: Using `permissions: write-all` unless absolutely necessary

## Example: Complete Bot Workflow

```yaml
name: PR Labeler Bot
on:
  pull_request:
    types: [opened, edited]

permissions:
  contents: read
  pull-requests: write

jobs:
  label:
    runs-on: ubuntu-latest
    steps:
      - name: Label PR based on files
        uses: actions/labeler@v4
        with:
          repo-token: ${{ secrets.GITHUB_TOKEN }}
```

## Troubleshooting

### "Resource not accessible by integration" error
- **Cause**: Insufficient permissions
- **Solution**: Add required permission to workflow file or update repository settings

### Bot can't push commits
- **Cause**: `contents: write` permission missing
- **Solution**: Add `contents: write` to permissions block

### Bot can't create PRs
- **Cause**: Either permission missing or repository setting disabled
- **Solution**: Add `pull-requests: write` permission AND enable "Allow GitHub Actions to create and approve pull requests" in repository settings

## Resources
- [GitHub Actions Permissions](https://docs.github.com/en/actions/security-guides/automatic-token-authentication)
- [Workflow Syntax for Permissions](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#permissions)
