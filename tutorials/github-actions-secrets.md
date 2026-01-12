# How to Use GitHub Actions Secrets Securely

## Overview
GitHub Actions secrets allow you to store sensitive information like API keys, passwords, and tokens securely. This guide shows you how to use them properly.

## What Are GitHub Actions Secrets?

Secrets are encrypted environment variables that you can use in GitHub Actions workflows. They are:
- Encrypted at rest
- Never exposed in logs
- Accessible only during workflow execution
- Scoped to repositories or organizations

## Types of Secrets

### 1. Repository Secrets
Available to workflows in a single repository.

### 2. Environment Secrets
Available to workflows targeting specific deployment environments.

### 3. Organization Secrets
Shared across multiple repositories in an organization.

## Creating Repository Secrets

### Via GitHub UI
1. Go to your repository
2. Click **Settings** → **Secrets and variables** → **Actions**
3. Click **New repository secret**
4. Enter:
   - **Name**: `API_KEY` (uppercase with underscores)
   - **Value**: Your secret value
5. Click **Add secret**

### Via GitHub CLI
```bash
# Set a secret
gh secret set API_KEY

# Set from file
gh secret set SSH_KEY < ~/.ssh/id_rsa

# Set with value
echo "my-secret-value" | gh secret set API_KEY
```

## Using Secrets in Workflows

### Basic Usage
```yaml
name: Deploy
on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to production
        env:
          API_KEY: ${{ secrets.API_KEY }}
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: |
          echo "Deploying with API key"
          # API_KEY is available as environment variable
          ./deploy.sh
```

### Multiple Secrets
```yaml
steps:
  - name: Configure AWS credentials
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_REGION: ${{ secrets.AWS_REGION }}
    run: |
      aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
      aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
      aws configure set region $AWS_REGION
```

### In Action Inputs
```yaml
steps:
  - name: Deploy to Vercel
    uses: amondnet/vercel-action@v20
    with:
      vercel-token: ${{ secrets.VERCEL_TOKEN }}
      vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
      vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
```

## Environment Secrets

### Create Environment
1. **Settings** → **Environments** → **New environment**
2. Name it (e.g., `production`, `staging`)
3. Add secrets specific to that environment

### Use in Workflow
```yaml
name: Deploy to Production
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # References the environment
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy
        env:
          # Environment secrets override repository secrets
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
        run: ./deploy.sh
```

### Environment Protection Rules
```yaml
# Settings → Environments → production → Protection rules
# - Required reviewers: Require manual approval
# - Wait timer: Delay deployment
# - Deployment branches: Restrict to specific branches
```

## Organization Secrets

### Create Organization Secret
1. Organization **Settings** → **Secrets and variables** → **Actions**
2. Click **New organization secret**
3. Configure:
   - **Name**: Secret name
   - **Value**: Secret value
   - **Repository access**: 
     - All repositories
     - Private repositories
     - Selected repositories
4. Click **Add secret**

### Access in Workflow
```yaml
# Same syntax as repository secrets
env:
  ORG_WIDE_TOKEN: ${{ secrets.ORG_WIDE_TOKEN }}
```

## Security Best Practices

### ✅ Do's

#### Use Descriptive Names
```yaml
# Good
secrets.AWS_ACCESS_KEY_ID
secrets.PROD_DATABASE_URL
secrets.SLACK_WEBHOOK_URL

# Avoid
secrets.KEY1
secrets.SECRET
secrets.TOKEN
```

#### Limit Secret Scope
```yaml
# Use environment secrets for deployment
environment: production
env:
  SECRET: ${{ secrets.PRODUCTION_SECRET }}

# Not repository-wide when only one env needs it
```

#### Rotate Secrets Regularly
```bash
# Update secret value
gh secret set API_KEY
# Enter new value
```

#### Use Separate Secrets for Different Environments
```yaml
# Different secrets for each environment
STAGING_API_KEY
PRODUCTION_API_KEY
DEVELOPMENT_API_KEY
```

### ❌ Don'ts

#### Never Echo Secrets
```yaml
# BAD - Exposes secret in logs
- run: echo "Secret is ${{ secrets.API_KEY }}"

# BAD - Also exposes secret
- run: |
    echo "API Key: $API_KEY"
  env:
    API_KEY: ${{ secrets.API_KEY }}

# GOOD - Secrets are automatically masked
- run: ./script.sh
  env:
    API_KEY: ${{ secrets.API_KEY }}
```

#### Don't Pass Secrets Between Jobs Unsafely
```yaml
# BAD - Exposes secret in artifact
- run: echo "${{ secrets.API_KEY }}" > secret.txt
- uses: actions/upload-artifact@v3
  with:
    name: secrets
    path: secret.txt

# GOOD - Keep secrets in environment variables
```

#### Avoid Committing Secrets
```yaml
# BAD - In workflow file
env:
  API_KEY: "hardcoded-key-123"

# GOOD - Use secrets
env:
  API_KEY: ${{ secrets.API_KEY }}
```

## Secret Masking

GitHub automatically masks secrets in logs:

```yaml
steps:
  - name: Test masking
    env:
      MY_SECRET: ${{ secrets.MY_SECRET }}
    run: |
      # This will be masked: ***
      echo "Secret value: $MY_SECRET"
      
      # Output from commands is also masked
      curl -H "Authorization: Bearer $MY_SECRET" api.example.com
```

### Register Custom Masks
```yaml
- name: Mask custom value
  run: |
    echo "::add-mask::my-custom-secret-value"
    # Now this value will be masked in all subsequent steps
```

## Advanced Patterns

### Conditional Secrets
```yaml
steps:
  - name: Use different secrets based on branch
    env:
      API_KEY: ${{ github.ref == 'refs/heads/main' && secrets.PROD_API_KEY || secrets.DEV_API_KEY }}
    run: ./deploy.sh
```

### JSON Secrets
```yaml
# Store JSON as secret
# Secret name: FIREBASE_CONFIG
# Secret value: {"apiKey":"xxx","authDomain":"yyy"}

steps:
  - name: Use JSON secret
    env:
      FIREBASE_CONFIG: ${{ secrets.FIREBASE_CONFIG }}
    run: |
      echo $FIREBASE_CONFIG > firebase-config.json
      # Use config file
```

### Multi-line Secrets
```yaml
# Store multi-line secrets (like SSH keys or certificates)
# Just paste the entire multi-line content as the secret value

steps:
  - name: Use SSH key
    run: |
      echo "${{ secrets.SSH_PRIVATE_KEY }}" > ssh_key
      chmod 600 ssh_key
      ssh -i ssh_key user@server
```

### Secret in File
```yaml
steps:
  - name: Create credentials file
    run: |
      echo "${{ secrets.SERVICE_ACCOUNT_JSON }}" > credentials.json
      
  - name: Use credentials
    run: ./app --credentials=credentials.json
    
  - name: Cleanup
    if: always()
    run: rm -f credentials.json
```

## Troubleshooting

### Secret Not Available
**Problem**: `secrets.MY_SECRET` is empty in workflow

**Solutions**:
1. Check secret name matches exactly (case-sensitive)
2. Verify secret is created in correct repository/organization
3. For organization secrets, check repository access settings
4. For environment secrets, ensure job specifies `environment:`

### Secret Exposed in Logs
**Problem**: Secret appears in workflow logs

**Solutions**:
1. Remove any `echo` or `print` statements with secrets
2. Check tool output isn't logging the secret
3. Use `::add-mask::` to mask custom values
4. Rotate the exposed secret immediately

### Can't Update Secret
**Problem**: Need to change secret value

**Solutions**:
```bash
# Via CLI
gh secret set SECRET_NAME

# Via UI
# Settings → Secrets → Click secret name → Update secret
```

## Alternatives to Secrets

### For Public Values
Use variables instead of secrets:
```yaml
# Settings → Secrets and variables → Actions → Variables tab
env:
  API_ENDPOINT: ${{ vars.API_ENDPOINT }}  # Not sensitive
  DEBUG_MODE: ${{ vars.DEBUG_MODE }}
```

### For External Secret Management
```yaml
# AWS Secrets Manager
- uses: aws-actions/aws-secretsmanager-get-secrets@v1
  with:
    secret-ids: |
      my-secret-1
      my-secret-2

# HashiCorp Vault
- uses: hashicorp/vault-action@v2
  with:
    url: ${{ secrets.VAULT_URL }}
    token: ${{ secrets.VAULT_TOKEN }}
    secrets: |
      secret/data/production api_key | API_KEY
```

## Auditing

### View Secret Usage
1. **Settings** → **Secrets and variables** → **Actions**
2. Click on secret name
3. View "Last used" timestamp

### Check Workflow Runs
```bash
# List workflow runs
gh run list

# View run details
gh run view <run-id>
```

## Resources
- [Encrypted secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
