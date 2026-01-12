# How to Setup a GitHub Actions Bot with Proper Permissions

## Overview
This comprehensive guide walks you through setting up a GitHub Actions bot with the correct permissions to perform automated tasks in your repository. It covers both GitHub-hosted and self-hosted runners, including the full command flow with proper privilege separation (what to run as root vs. regular user).

## Prerequisites
- A GitHub repository
- Admin access to repository settings
- Basic understanding of GitHub Actions
- For self-hosted runners: A Linux server with sudo access
- SSH access to your server (if setting up self-hosted runner)

## Table of Contents
1. [Quick Start: GitHub-Hosted Runners](#step-1-configure-workflow-permissions)
2. [Self-Hosted Runner Setup](#self-hosted-runner-setup)
3. [Permission Scopes](#step-2-common-permission-scopes)
4. [Using GITHUB_TOKEN](#step-3-using-github_token)
5. [Personal Access Tokens](#step-4-custom-personal-access-token-pat)
6. [Best Practices](#step-5-best-practices)
7. [Complete Examples](#example-complete-bot-workflow)
8. [Troubleshooting](#troubleshooting)

---

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

---

## Self-Hosted Runner Setup

If you need more control or want to run workflows on your own infrastructure, set up a self-hosted runner. This section covers the complete flow with proper permission handling.

### System Requirements
- Linux (Ubuntu 20.04+ recommended), macOS, or Windows
- At least 2GB RAM, 10GB disk space
- User account (non-root) for running the runner
- Sudo access for initial setup

### Step 1: Create a Dedicated User (Run as ROOT)

**⚠️ Run these commands as root or with sudo:**

```bash
# Create a dedicated user for the GitHub Actions runner
sudo useradd -m -s /bin/bash github-runner

# Set a password (optional, needed if you want to sudo as this user)
sudo passwd github-runner

# Add to docker group if you'll run Docker containers (optional)
sudo usermod -aG docker github-runner

# Verify user creation
id github-runner
```

**Why create a dedicated user?**
- Security isolation: Runner processes don't run as root
- Easier permission management
- Clear audit trail for runner actions

### Step 2: Install Dependencies (Run as ROOT)

**⚠️ Run these commands as root or with sudo:**

```bash
# Update package lists
sudo apt update

# Install required dependencies
sudo apt install -y curl jq wget git

# Install Docker (if needed for containerized actions)
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker

# Verify installations
docker --version
git --version
```

### Step 3: Download and Configure Runner (Run as REGULAR USER)

**✅ Switch to the github-runner user for these commands:**

```bash
# Switch to the runner user
sudo su - github-runner

# Create a directory for the runner
mkdir -p ~/actions-runner && cd ~/actions-runner

# Download the latest runner package
# Get the latest version from: https://github.com/actions/runner/releases
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz

# Extract the installer
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz

# Clean up the archive
rm actions-runner-linux-x64-2.311.0.tar.gz
```

### Step 4: Get Runner Registration Token (Via GitHub UI)

1. Go to your repository on GitHub
2. Navigate to **Settings** → **Actions** → **Runners**
3. Click **New self-hosted runner**
4. Select your OS (Linux/macOS/Windows)
5. Copy the registration token (it will look like: `AABBC...XYZ`)

**Alternatively, use GitHub CLI (requires PAT with `admin:org` or `repo` scope):**

```bash
# For repository runner
gh api -X POST /repos/OWNER/REPO/actions/runners/registration-token

# For organization runner
gh api -X POST /orgs/ORG/actions/runners/registration-token
```

### Step 5: Configure the Runner (Run as REGULAR USER)

**✅ Run as github-runner user:**

```bash
# Configure the runner (interactive)
./config.sh --url https://github.com/OWNER/REPO --token YOUR_REGISTRATION_TOKEN

# You'll be prompted for:
# - Runner group: Press Enter for default
# - Runner name: Give it a meaningful name (e.g., "production-server-1")
# - Runner labels: Add custom labels (e.g., "self-hosted,linux,production")
# - Work folder: Press Enter for default (_work)

# Example non-interactive configuration
./config.sh \
  --url https://github.com/OWNER/REPO \
  --token YOUR_REGISTRATION_TOKEN \
  --name "prod-runner-1" \
  --labels "self-hosted,linux,x64,production" \
  --work "_work" \
  --unattended
```

### Step 6: Set Up Runner as a Service (Run as ROOT)

**⚠️ Run these commands as root or with sudo:**

```bash
# Install the runner service
cd /home/github-runner/actions-runner
sudo ./svc.sh install github-runner

# Start the runner service
sudo ./svc.sh start

# Check service status
sudo ./svc.sh status

# Enable runner to start on boot
sudo systemctl enable actions.runner.*
```

**Alternative: Run interactively (for testing, run as REGULAR USER):**

```bash
# Run the runner in the foreground (useful for debugging)
./run.sh
```

### Step 7: Verify Runner Registration

1. Go back to **Settings** → **Actions** → **Runners** in your repository
2. You should see your runner listed with a green "Idle" status
3. Note the labels assigned to your runner

### Step 8: Set File Permissions (Run as ROOT if needed)

**⚠️ Run these commands as root or with sudo:**

```bash
# Ensure github-runner owns all runner files
sudo chown -R github-runner:github-runner /home/github-runner/actions-runner

# Set proper permissions for the work directory
sudo chmod -R 755 /home/github-runner/actions-runner/_work

# If you need the runner to write to specific directories
sudo mkdir -p /opt/deploy
sudo chown github-runner:github-runner /opt/deploy
sudo chmod 755 /opt/deploy
```

### Step 9: Configure Workflow to Use Self-Hosted Runner

**✅ Update your workflow file:**

```yaml
name: Deploy on Self-Hosted Runner
on: [push]

jobs:
  deploy:
    # Use your self-hosted runner
    runs-on: [self-hosted, linux, production]
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Check runner environment
        run: |
          echo "Running on: $(hostname)"
          echo "User: $(whoami)"
          echo "Working directory: $(pwd)"
          
      - name: Deploy application
        run: |
          # Your deployment commands here
          echo "Deploying..."
```

### Security Considerations for Self-Hosted Runners

**⚠️ IMPORTANT: Self-hosted runners have different security implications**

1. **Never use self-hosted runners with public repositories**
   - Anyone can fork and create PRs that run code on your infrastructure
   - Only use with private repositories

2. **Limit runner access:**
```bash
# Run as ROOT - Restrict network access if needed
sudo ufw enable
sudo ufw allow ssh
sudo ufw allow 443  # HTTPS for GitHub
```

3. **Regular updates (Run as ROOT):**
```bash
# Update the runner periodically
cd /home/github-runner/actions-runner
sudo ./svc.sh stop
sudo su - github-runner
./config.sh remove --token YOUR_REMOVAL_TOKEN
# Then repeat steps 3-6 with latest version
```

4. **Monitor runner activity:**
```bash
# Check runner service logs
sudo journalctl -u actions.runner.* -f

# Check what the runner user is doing
ps aux | grep github-runner
```

### Managing Runner Lifecycle

**Stop the runner (Run as ROOT):**
```bash
sudo ./svc.sh stop
```

**Remove the runner (Run as ROOT and REGULAR USER):**
```bash
# Stop the service first (as root)
cd /home/github-runner/actions-runner
sudo ./svc.sh stop
sudo ./svc.sh uninstall

# Remove the runner configuration (as github-runner user)
sudo su - github-runner
cd ~/actions-runner
./config.sh remove --token YOUR_REMOVAL_TOKEN
```

---

## Step 2: Common Permission Scopes

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

### Command-Line Examples with GITHUB_TOKEN

**Push commits to repository:**
```yaml
steps:
  - uses: actions/checkout@v3
    with:
      token: ${{ secrets.GITHUB_TOKEN }}
      
  - name: Make changes and push
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    run: |
      # Configure git
      git config user.name "github-actions[bot]"
      git config user.email "github-actions[bot]@users.noreply.github.com"
      
      # Make changes
      echo "Updated at $(date)" >> updates.txt
      
      # Commit and push
      git add .
      git commit -m "Automated update"
      git push
```

**Create and manage issues:**
```yaml
steps:
  - name: Create issue
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    run: |
      gh issue create \
        --title "Automated Issue" \
        --body "This issue was created by GitHub Actions" \
        --label "automation"
      
      # List issues
      gh issue list --state open
      
      # Close an issue
      gh issue close 123 --comment "Resolved by automation"
```

**Work with pull requests:**
```yaml
steps:
  - name: Manage PRs
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    run: |
      # Create a PR
      gh pr create \
        --title "Automated PR" \
        --body "Changes made by bot" \
        --base main \
        --head feature-branch
      
      # Add reviewers
      gh pr edit 123 --add-reviewer username
      
      # Merge a PR
      gh pr merge 123 --squash --delete-branch
```

## Step 4: Custom Personal Access Token (PAT)

For actions that need elevated permissions or cross-repository access:

### Creating a PAT (Via GitHub UI)

1. Go to **Settings** (your personal settings, not repository)
2. Navigate to **Developer settings** → **Personal access tokens** → **Tokens (classic)**
3. Click **Generate new token** → **Generate new token (classic)**
4. Configure the token:
   - **Note**: "GitHub Actions Bot Token"
   - **Expiration**: Choose appropriate duration (90 days recommended)
   - **Select scopes**:
     - `repo` - Full control of private repositories
     - `workflow` - Update GitHub Action workflows
     - `write:packages` - Upload packages
     - `delete:packages` - Delete packages
     - `admin:org` - Full control of orgs (if needed)
5. Click **Generate token**
6. **IMPORTANT**: Copy the token immediately (you won't see it again)

### Creating a PAT (Via GitHub CLI)

```bash
# Login to GitHub CLI (run this locally, not in CI)
gh auth login

# Create a token with specific scopes
gh auth token

# For more control, use the API
gh api -X POST /user/tokens \
  -f note="GitHub Actions Bot" \
  -f scopes='["repo","workflow"]'
```

### Adding PAT as Repository Secret

**Via GitHub UI:**
1. Go to your repository **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Name: `BOT_PAT` (or any descriptive name)
4. Value: Paste your token
5. Click **Add secret**

**Via GitHub CLI (from your local machine):**
```bash
# Set a secret in your repository
gh secret set BOT_PAT --body "ghp_your_token_here"

# Or read from file
cat token.txt | gh secret set BOT_PAT

# Set secret for a specific repository
gh secret set BOT_PAT --repo OWNER/REPO --body "ghp_your_token_here"

# List secrets (values are hidden)
gh secret list
```

**Via API with curl:**
```bash
# This requires encrypting the secret first
# Use gh CLI instead for simplicity, or see:
# https://docs.github.com/en/rest/actions/secrets
```

### Using PAT in Workflows

**For checkout with extended permissions:**
```yaml
steps:
  - uses: actions/checkout@v3
    with:
      token: ${{ secrets.BOT_PAT }}
      # This allows pushing to protected branches
      # and triggering workflows from commits
```

**For cross-repository operations:**
```yaml
steps:
  - name: Clone another repository
    env:
      PAT: ${{ secrets.BOT_PAT }}
    run: |
      git clone https://x-access-token:${PAT}@github.com/owner/other-repo.git
      cd other-repo
      # Make changes
      git config user.name "bot"
      git config user.email "bot@example.com"
      echo "update" >> file.txt
      git add .
      git commit -m "Update from bot"
      git push
```

**For GitHub API calls:**
```yaml
steps:
  - name: Call GitHub API
    env:
      GITHUB_TOKEN: ${{ secrets.BOT_PAT }}
    run: |
      # Create a repository
      gh api -X POST /user/repos \
        -f name="new-repo" \
        -f private=true
      
      # Update repository settings
      gh api -X PATCH /repos/owner/repo \
        -f has_issues=true \
        -f has_wiki=false
```

### PAT Best Practices

1. **Use fine-grained tokens when possible** (Beta):
```bash
# Fine-grained tokens offer better security
# Create via: Settings → Developer settings → Personal access tokens → Fine-grained tokens
# Benefits:
# - Repository-specific access
# - More granular permissions
# - Shorter expiration times
```

2. **Rotate tokens regularly:**
```bash
# Set reminder to rotate every 90 days
# Update the secret when you rotate:
gh secret set BOT_PAT --body "ghp_new_token_here"
```

3. **Use separate tokens for different purposes:**
```yaml
# Example: Different tokens for different jobs
jobs:
  deploy:
    env:
      DEPLOY_TOKEN: ${{ secrets.DEPLOY_PAT }}
  
  release:
    env:
      RELEASE_TOKEN: ${{ secrets.RELEASE_PAT }}
```

## Step 5: Best Practices

### Permission Management

- ✅ **Principle of least privilege**: Only grant permissions needed for the task
- ✅ **Use job-level permissions**: Override at job level for granular control
  ```yaml
  permissions:
    contents: read  # Default for all jobs
  
  jobs:
    deploy:
      permissions:
        contents: write  # Override for specific job
  ```
- ✅ **Explicit permissions**: Always specify permissions in workflow files
- ✅ **Audit regularly**: Review bot actions and permissions quarterly
- ✅ **Rotate tokens**: If using PATs, rotate them every 90 days
- ❌ **Avoid**: Using `permissions: write-all` unless absolutely necessary
- ❌ **Never**: Commit tokens to the repository or logs

### Security Best Practices

**1. Protect secrets in logs:**
```yaml
steps:
  - name: Use secret safely
    env:
      SECRET_VALUE: ${{ secrets.MY_SECRET }}
    run: |
      # ✅ Good: Use secret without printing
      curl -H "Authorization: token $SECRET_VALUE" https://api.github.com
      
      # ❌ Bad: Never echo secrets
      # echo $SECRET_VALUE
```

**2. Mask custom values:**
```yaml
steps:
  - name: Mask sensitive output
    run: |
      API_KEY=$(generate-api-key)
      echo "::add-mask::$API_KEY"
      # Now API_KEY will be masked in logs
      echo "Generated key: $API_KEY"
```

**3. Use environment protection:**
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # Requires approval
    steps:
      - name: Deploy to production
        run: echo "Deploying..."
```

**4. Validate inputs:**
```yaml
steps:
  - name: Validate before execution
    run: |
      if [[ ! "${{ github.event.pull_request.head.ref }}" =~ ^[a-zA-Z0-9_-]+$ ]]; then
        echo "Invalid branch name"
        exit 1
      fi
```

### File System Permissions (Self-Hosted Runners)

**Setting up workspace permissions (Run as ROOT):**
```bash
# Create a workspace directory
sudo mkdir -p /opt/github-actions-workspace
sudo chown github-runner:github-runner /opt/github-actions-workspace
sudo chmod 755 /opt/github-actions-workspace

# For deployments that need specific permissions
sudo mkdir -p /var/www/html
sudo chown github-runner:www-data /var/www/html
sudo chmod 775 /var/www/html
```

**In your workflow:**
```yaml
jobs:
  deploy:
    runs-on: [self-hosted]
    steps:
      - name: Deploy files
        run: |
          # Copy files to deployment directory
          cp -r dist/* /var/www/html/
          
          # Verify permissions
          ls -la /var/www/html/
```

### Monitoring and Auditing

**1. Enable audit logging (Organization level):**
- Go to **Organization Settings** → **Audit log**
- Review actions performed by bots
- Export logs for compliance

**2. Monitor workflow runs:**
```bash
# Using GitHub CLI
gh run list --limit 50
gh run view RUN_ID --log

# Check for failures
gh run list --status failure
```

**3. Set up notifications:**
```yaml
jobs:
  notify-on-failure:
    runs-on: ubuntu-latest
    if: failure()
    steps:
      - name: Send notification
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
        run: |
          curl -X POST $SLACK_WEBHOOK \
            -H 'Content-Type: application/json' \
            -d '{"text":"Workflow failed!"}'
```

### Permission Scope Reference Table

| Scope | Read | Write | Use Cases |
|-------|------|-------|-----------|
| `actions` | View workflows | Modify workflows | Workflow management |
| `checks` | View check runs | Create checks | Status checks, CI/CD |
| `contents` | Read files | Write files, tags | Code changes, releases |
| `deployments` | View deployments | Create deployments | Deployment tracking |
| `issues` | View issues | Create/edit issues | Issue management |
| `packages` | Download packages | Publish packages | Package registry |
| `pages` | View Pages | Deploy Pages | GitHub Pages |
| `pull-requests` | View PRs | Create/edit PRs | PR automation |
| `repository-projects` | View projects | Manage projects | Project boards |
| `security-events` | View alerts | Manage alerts | Security scanning |
| `statuses` | View statuses | Create statuses | Commit statuses |

## Example: Complete Bot Workflow

### Example 1: PR Labeler Bot
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

### Example 2: Automated Release Bot
```yaml
name: Automated Release
on:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write
  issues: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0  # Get all history for changelog
          
      - name: Configure Git
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
      
      - name: Determine version bump
        id: semver
        run: |
          # Analyze commits to determine version bump
          if git log -1 --pretty=%B | grep -q "BREAKING CHANGE"; then
            echo "bump=major" >> $GITHUB_OUTPUT
          elif git log -1 --pretty=%B | grep -q "feat:"; then
            echo "bump=minor" >> $GITHUB_OUTPUT
          else
            echo "bump=patch" >> $GITHUB_OUTPUT
          fi
      
      - name: Create release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          # Get current version
          CURRENT_VERSION=$(git describe --tags --abbrev=0 2>/dev/null || echo "v0.0.0")
          
          # Calculate new version (simplified)
          NEW_VERSION=$(echo $CURRENT_VERSION | awk -F. '{$NF = $NF + 1;} 1' | sed 's/ /./g')
          
          # Create tag
          git tag -a "$NEW_VERSION" -m "Release $NEW_VERSION"
          git push origin "$NEW_VERSION"
          
          # Create GitHub release
          gh release create "$NEW_VERSION" \
            --title "Release $NEW_VERSION" \
            --generate-notes
```

### Example 3: Automated Code Review Bot
```yaml
name: Code Review Bot
on:
  pull_request:
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Check for TODOs
        id: todos
        run: |
          TODOS=$(grep -r "TODO" --include="*.js" --include="*.py" . || true)
          if [ -n "$TODOS" ]; then
            echo "found=true" >> $GITHUB_OUTPUT
            echo "$TODOS" >> todos.txt
          else
            echo "found=false" >> $GITHUB_OUTPUT
          fi
      
      - name: Comment on PR
        if: steps.todos.outputs.found == 'true'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          TODO_LIST=$(cat todos.txt)
          gh pr comment ${{ github.event.pull_request.number }} \
            --body "⚠️ Found TODOs in this PR:
          
          \`\`\`
          $TODO_LIST
          \`\`\`
          
          Please address these before merging."
```

### Example 4: Self-Hosted Deployment Bot
```yaml
name: Deploy to Production
on:
  push:
    branches: [main]

permissions:
  contents: read
  deployments: write

jobs:
  deploy:
    runs-on: [self-hosted, linux, production]
    environment: production
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Create deployment
        id: deployment
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          DEPLOYMENT_ID=$(gh api \
            -X POST /repos/${{ github.repository }}/deployments \
            -f ref=${{ github.sha }} \
            -f environment=production \
            -f auto_merge=false \
            --jq '.id')
          echo "deployment_id=$DEPLOYMENT_ID" >> $GITHUB_OUTPUT
      
      - name: Deploy application
        run: |
          # Build the application
          npm ci
          npm run build
          
          # Deploy (commands run as github-runner user)
          sudo systemctl stop myapp
          sudo cp -r dist/* /var/www/html/
          sudo systemctl start myapp
      
      - name: Update deployment status
        if: always()
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          STATUS=${{ job.status == 'success' && 'success' || 'failure' }}
          gh api -X POST \
            /repos/${{ github.repository }}/deployments/${{ steps.deployment.outputs.deployment_id }}/statuses \
            -f state=$STATUS \
            -f environment=production
```

### Example 5: Multi-Repository Sync Bot
```yaml
name: Sync Configuration
on:
  push:
    paths:
      - 'config/**'

permissions:
  contents: read

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Sync to other repositories
        env:
          PAT: ${{ secrets.BOT_PAT }}
        run: |
          # List of repositories to sync to
          REPOS=("org/repo1" "org/repo2" "org/repo3")
          
          for REPO in "${REPOS[@]}"; do
            echo "Syncing to $REPO..."
            
            # Clone the target repository
            git clone "https://x-access-token:${PAT}@github.com/${REPO}.git" target
            
            # Copy configuration files
            cp -r config/* target/config/
            
            # Commit and push
            cd target
            git config user.name "config-sync-bot"
            git config user.email "bot@example.com"
            git add .
            
            if git diff --staged --quiet; then
              echo "No changes for $REPO"
            else
              git commit -m "Sync configuration from main repo"
              git push
            fi
            
            cd ..
            rm -rf target
          done
```

## Troubleshooting

### "Resource not accessible by integration" error
- **Cause**: Insufficient permissions for GITHUB_TOKEN
- **Solution**: Add required permission to workflow file or update repository settings
  ```yaml
  permissions:
    contents: write  # Add the required permission
  ```

### Bot can't push commits
- **Cause**: `contents: write` permission missing
- **Solution**: Add `contents: write` to permissions block
  ```yaml
  permissions:
    contents: write
  ```
- **Alternative cause**: Protected branch rules
- **Solution**: Either use a PAT or exempt the GitHub Actions bot in branch protection settings

### Bot can't create PRs
- **Cause**: Either permission missing or repository setting disabled
- **Solution**: 
  1. Add `pull-requests: write` permission:
     ```yaml
     permissions:
       pull-requests: write
       contents: write  # Also needed to create branches
     ```
  2. Enable in repository settings: **Settings** → **Actions** → **General** → Check "Allow GitHub Actions to create and approve pull requests"

### "fatal: could not read Password" error
- **Cause**: Git credential issue
- **Solution**: Use token in checkout:
  ```yaml
  - uses: actions/checkout@v3
    with:
      token: ${{ secrets.GITHUB_TOKEN }}
      persist-credentials: true
  ```

### Self-hosted runner shows "Offline"
- **Cause**: Runner service not running or network issue
- **Solution**: Check and restart the service
  ```bash
  # Check status (as root)
  sudo systemctl status actions.runner.*
  
  # Restart service (as root)
  cd /home/github-runner/actions-runner
  sudo ./svc.sh stop
  sudo ./svc.sh start
  
  # Check logs
  sudo journalctl -u actions.runner.* -n 50
  ```

### Self-hosted runner permission denied errors
- **Cause**: Runner user doesn't have permission to access files/directories
- **Solution**: Fix file permissions
  ```bash
  # Check current permissions
  ls -la /path/to/directory
  
  # Fix ownership (as root)
  sudo chown -R github-runner:github-runner /path/to/directory
  
  # Fix permissions (as root)
  sudo chmod -R 755 /path/to/directory
  ```

### "API rate limit exceeded" error
- **Cause**: Too many API calls with GITHUB_TOKEN
- **Solution**: Use a PAT which has higher rate limits
  ```yaml
  env:
    GITHUB_TOKEN: ${{ secrets.BOT_PAT }}
  ```

### Secrets not available in fork PRs
- **Cause**: Security feature to prevent secret exposure
- **Solution**: This is intentional. For public repos, avoid running sensitive workflows on fork PRs:
  ```yaml
  on:
    pull_request_target:  # Has access to secrets but runs on base branch
  ```
  ⚠️ **Warning**: Be very careful with `pull_request_target` - validate all inputs!

### Token expired error
- **Cause**: PAT has expired
- **Solution**: Create a new token and update the secret
  ```bash
  # Generate new token via GitHub UI or CLI
  gh auth login
  
  # Update secret
  gh secret set BOT_PAT --body "ghp_new_token_here"
  ```

### Docker permission denied (self-hosted)
- **Cause**: Runner user not in docker group
- **Solution**: Add user to docker group (as root)
  ```bash
  sudo usermod -aG docker github-runner
  
  # Restart runner service for changes to take effect
  sudo ./svc.sh restart
  
  # Verify
  sudo su - github-runner
  docker ps
  ```

### Workflow not triggering
- **Cause**: Commits pushed by GITHUB_TOKEN don't trigger new workflows (prevents infinite loops)
- **Solution**: Use a PAT for commits that should trigger workflows
  ```yaml
  - uses: actions/checkout@v3
    with:
      token: ${{ secrets.BOT_PAT }}
  ```

### Debug workflow permissions
Add this step to debug what permissions your workflow has:
```yaml
steps:
  - name: Debug permissions
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    run: |
      echo "Repository: ${{ github.repository }}"
      echo "Actor: ${{ github.actor }}"
      echo "Event: ${{ github.event_name }}"
      
      # Check token permissions
      gh api /repos/${{ github.repository }} --jq '.permissions'
      
      # Test specific permissions
      gh api /repos/${{ github.repository }}/issues || echo "No issues permission"
      gh api /repos/${{ github.repository }}/pulls || echo "No PR permission"
```

---

## Quick Reference Commands

### GitHub CLI Commands (with proper permissions)

```bash
# Authentication
gh auth login
gh auth status

# Repository operations
gh repo create my-repo --private
gh repo view owner/repo

# Issue management
gh issue create --title "Issue" --body "Description"
gh issue list --state open
gh issue close 123

# Pull request operations
gh pr create --title "PR Title" --body "Description"
gh pr list --state open
gh pr merge 123 --squash
gh pr review 123 --approve

# Workflow operations
gh workflow list
gh workflow run workflow.yml
gh run list
gh run view RUN_ID

# Secret management
gh secret list
gh secret set SECRET_NAME
gh secret delete SECRET_NAME
```

### Git Commands for Bot Workflows

```bash
# Configure Git identity
git config user.name "github-actions[bot]"
git config user.email "github-actions[bot]@users.noreply.github.com"

# Commit and push
git add .
git commit -m "Automated commit"
git push

# Create and push tags
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0

# Create branch and push
git checkout -b feature-branch
git push -u origin feature-branch
```

### Self-Hosted Runner Commands

```bash
# As ROOT/sudo:
sudo ./svc.sh start    # Start runner service
sudo ./svc.sh stop     # Stop runner service
sudo ./svc.sh status   # Check runner status
sudo ./svc.sh restart  # Restart runner service

# As github-runner user:
./run.sh              # Run runner interactively
./config.sh --help    # Show configuration options

# Monitoring:
sudo journalctl -u actions.runner.* -f  # Follow logs
ps aux | grep Runner.Listener            # Check if running
```

## Resources
- [GitHub Actions Permissions](https://docs.github.com/en/actions/security-guides/automatic-token-authentication)
- [Workflow Syntax for Permissions](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#permissions)
- [Self-hosted Runners Documentation](https://docs.github.com/en/actions/hosting-your-own-runners)
- [GitHub CLI Manual](https://cli.github.com/manual/)
- [Security Hardening for GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
