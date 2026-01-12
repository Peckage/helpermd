# How to Create a GitHub Personal Access Token (PAT)

## Overview
Personal Access Tokens (PATs) allow you to authenticate to GitHub APIs and Git operations over HTTPS without using your password.

## When You Need a PAT
- Accessing GitHub API
- Git operations over HTTPS
- GitHub Actions needing elevated permissions
- Third-party applications and scripts
- CI/CD pipelines

## Creating a Classic PAT

### Step 1: Navigate to Settings
1. Click your profile photo (top right) → **Settings**
2. Scroll down left sidebar → **Developer settings**
3. Click **Personal access tokens** → **Tokens (classic)**

### Step 2: Generate Token
1. Click **Generate new token** → **Generate new token (classic)**
2. Give your token a descriptive name (e.g., "CI/CD Pipeline Token")
3. Set expiration (recommendation: 90 days or less for security)
4. Select scopes based on your needs (see below)
5. Click **Generate token**

### Step 3: Save Token Immediately
⚠️ **Important**: Copy the token immediately. You won't be able to see it again!

```bash
# Store in a secure location, not in your code
export GITHUB_TOKEN="ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

## Common Permission Scopes

### Repository Access
- `repo` - Full control of private repositories
  - `repo:status` - Access commit status
  - `repo_deployment` - Access deployment status
  - `public_repo` - Access public repositories only
  - `repo:invite` - Access repository invitations
  - `security_events` - Read and write security events

### Workflow
- `workflow` - Update GitHub Actions workflows

### Packages
- `write:packages` - Upload packages
- `read:packages` - Download packages

### Organizations
- `admin:org` - Full control of organizations
- `read:org` - Read organization data

### User Data
- `read:user` - Read profile information
- `user:email` - Access email addresses

## Creating a Fine-Grained PAT (Beta)

### Advantages
- Repository-specific access
- More granular permissions
- Better security posture

### Steps
1. **Settings** → **Developer settings** → **Personal access tokens** → **Fine-grained tokens**
2. Click **Generate new token**
3. Configure:
   - **Token name**: Descriptive name
   - **Expiration**: Set expiration date
   - **Repository access**: 
     - All repositories
     - Only select repositories
   - **Permissions**: Choose specific permissions per category
4. Click **Generate token**

## Using Your PAT

### Git Clone/Push/Pull
```bash
# Clone with PAT
git clone https://ghp_YOUR_TOKEN@github.com/username/repo.git

# Or configure credential helper
git config --global credential.helper store
git clone https://github.com/username/repo.git
# Enter username and PAT as password
```

### GitHub CLI
```bash
# Authenticate
echo "ghp_YOUR_TOKEN" | gh auth login --with-token

# Use CLI
gh repo list
```

### API Requests
```bash
# Using curl
curl -H "Authorization: token ghp_YOUR_TOKEN" \
  https://api.github.com/user/repos

# Using wget
wget --header="Authorization: token ghp_YOUR_TOKEN" \
  https://api.github.com/user/repos
```

### GitHub Actions
```yaml
# Use as secret in workflows
steps:
  - uses: actions/checkout@v3
    with:
      token: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
```

## Best Practices

### Security
- ✅ Set expiration dates (avoid "no expiration")
- ✅ Use fine-grained tokens when possible
- ✅ Grant minimum required permissions
- ✅ Use different tokens for different purposes
- ✅ Rotate tokens regularly
- ✅ Store tokens securely (password managers, secret stores)
- ❌ Never commit tokens to repositories
- ❌ Never share tokens via email/chat

### Token Management
```bash
# Use environment variables
export GITHUB_TOKEN="ghp_xxxx"

# Or use git credential helper
git config --global credential.helper cache
git config --global credential.helper 'cache --timeout=3600'
```

### Revoking Tokens
1. **Settings** → **Developer settings** → **Personal access tokens**
2. Find the token to revoke
3. Click **Delete** or **Revoke**
4. Confirm the action

## Troubleshooting

### "Bad credentials" error
- Token may be expired
- Token may have been revoked
- Incorrect token format
- **Solution**: Generate a new token

### "Resource not accessible" error
- Token lacks required permissions
- **Solution**: Update token scopes

### Token not working with Git
- Token may not have `repo` scope
- **Solution**: Regenerate with correct scopes

## Alternatives to PATs

### SSH Keys
```bash
# Generate SSH key
ssh-keygen -t ed25519 -C "your_email@example.com"

# Add to GitHub: Settings → SSH and GPG keys
```

### GitHub Apps
- For organizational automation
- Better audit trail
- More granular permissions

### Deploy Keys
- Read-only or read-write access to single repository
- Better for CI/CD deployment scenarios

## Resources
- [Creating a personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)
- [Token scopes](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/scopes-for-oauth-apps)
