# How to Write Better Git Commit Messages

## Overview
Good commit messages are essential for maintaining a clean and understandable project history. They help teammates understand changes and make code review easier.

## The Seven Rules (Summary)

1. Separate subject from body with a blank line
2. Limit the subject line to 50 characters
3. Capitalize the subject line
4. Do not end the subject line with a period
5. Use the imperative mood in the subject line
6. Wrap the body at 72 characters
7. Use the body to explain what and why vs. how

## Basic Format

```
<type>: <subject>

<body>

<footer>
```

## Conventional Commits

### Format
```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

### Common Types
- `feat` - A new feature
- `fix` - A bug fix
- `docs` - Documentation changes
- `style` - Code style changes (formatting, semicolons, etc.)
- `refactor` - Code refactoring without changing functionality
- `test` - Adding or modifying tests
- `chore` - Maintenance tasks, dependency updates
- `perf` - Performance improvements
- `ci` - CI/CD configuration changes
- `build` - Build system or external dependencies

### Examples

#### Simple Feature
```
feat: add user authentication

Implement JWT-based authentication for API endpoints.
Users can now register and login with email/password.
```

#### Bug Fix
```
fix: resolve null pointer exception in user service

The getUserById method was not checking for null before
accessing user properties, causing crashes when user
doesn't exist.

Fixes #123
```

#### Breaking Change
```
feat!: redesign API response format

BREAKING CHANGE: API now returns data in camelCase instead
of snake_case. All client applications need to update their
response parsing logic.
```

#### With Scope
```
feat(auth): implement OAuth2 provider support

Add support for Google and GitHub OAuth2 authentication
alongside existing email/password login.
```

## Good Examples

### ✅ Clear and Concise
```
feat: add password reset functionality

Implement password reset via email token:
- Generate secure reset tokens
- Send reset email with expiration link
- Validate token and allow password update
- Expire tokens after 1 hour

Closes #456
```

### ✅ Explains Why
```
refactor: switch from Redux to Zustand

Redux was adding unnecessary complexity for our small
state management needs. Zustand provides simpler API
and reduces boilerplate by ~300 lines.

Performance impact: bundle size reduced by 15KB.
```

### ✅ Multi-file Change
```
feat: implement dark mode

Add dark mode support across the application:
- Create theme context and provider
- Update all components with theme-aware styling
- Add theme toggle in settings
- Persist theme preference in localStorage

Affects: components/, styles/, hooks/
```

## Bad Examples

### ❌ Vague
```
fix: fix bug
```
**Better:**
```
fix: prevent duplicate form submissions

Add disabled state to submit button while form is
processing to prevent multiple API calls.
```

### ❌ Too Long Subject
```
feat: add user authentication with JWT tokens and implement login and registration endpoints and add middleware
```
**Better:**
```
feat: implement user authentication

Add JWT-based authentication system:
- Login and registration endpoints
- Auth middleware for protected routes
- Token refresh mechanism
```

### ❌ Past Tense
```
fixed login issue
```
**Better:**
```
fix: resolve login redirect loop

Replace window.location with router.push() to prevent
navigation issues after successful authentication.
```

## Subject Line Guidelines

### Use Imperative Mood
Think: "This commit will..."

✅ Good:
- Add feature
- Fix bug
- Update docs
- Remove deprecated code

❌ Bad:
- Added feature
- Fixes bug
- Updating docs
- Removed code

### Be Specific
✅ Good:
- `fix: prevent race condition in data fetching`
- `feat: add email validation to registration form`

❌ Bad:
- `fix: bug`
- `feat: update form`

## Body Guidelines

### Explain Context
```
refactor: extract database queries to repository layer

The service layer was directly accessing the database,
making it difficult to test and maintain. This change
introduces a repository pattern that:

1. Separates data access logic
2. Makes services easier to unit test
3. Provides consistent error handling
4. Enables easier database migration in future

No functional changes to API behavior.
```

### List Breaking Changes
```
feat!: upgrade to React Router v6

BREAKING CHANGES:
- <Switch> is now <Routes>
- <Route> component prop changed to element
- useHistory() is now useNavigate()
- Nested routes use relative paths

Migration guide: https://link-to-migration-guide.com
```

### Reference Issues
```
fix: resolve memory leak in WebSocket connection

WebSocket connections were not being properly closed
on component unmount, leading to memory accumulation
over time.

Fixes #789
Related to #654
```

## Footer Examples

### Issue References
```
Fixes #123
Closes #456, #789
Resolves #234
See also #555
```

### Co-authors
```
Co-authored-by: Jane Doe <jane@example.com>
Co-authored-by: John Smith <john@example.com>
```

### Reviewers
```
Reviewed-by: Alice Johnson <alice@example.com>
Acked-by: Bob Williams <bob@example.com>
```

## Templates

### Create Git Commit Template
```bash
# Create template file
cat > ~/.gitmessage << 'EOF'
# <type>: <subject> (max 50 chars)

# Body (wrap at 72 chars)
# - Explain what and why, not how
# - Use imperative mood

# Footer
# Fixes #issue
EOF

# Configure Git to use template
git config --global commit.template ~/.gitmessage
```

### VSCode Integration
```json
// settings.json
{
  "git.inputValidationLength": 50,
  "git.inputValidationSubjectLength": 50
}
```

## Commit Message Linting

### Install commitlint
```bash
npm install --save-dev @commitlint/cli @commitlint/config-conventional

# Create config file
echo "module.exports = {extends: ['@commitlint/config-conventional']}" > commitlint.config.js
```

### Add Commit Message Hook
```bash
# Install husky
npm install --save-dev husky

# Enable Git hooks
npx husky install

# Add commit-msg hook
npx husky add .husky/commit-msg 'npx --no -- commitlint --edit "$1"'
```

## Team Guidelines

### Document Your Convention
Create `CONTRIBUTING.md`:
```markdown
## Commit Message Format

We follow Conventional Commits:
- feat: New features
- fix: Bug fixes
- docs: Documentation only changes

Examples:
- `feat: add dark mode support`
- `fix: resolve login redirect issue`
```

### Enforce in CI
```yaml
# .github/workflows/commit-lint.yml
name: Lint Commits
on: [pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - uses: wagoid/commitlint-github-action@v5
```

## Best Practices

- ✅ Commit often, with logical changes
- ✅ One concern per commit
- ✅ Write for future you and your team
- ✅ Use the body for context
- ✅ Reference relevant issues
- ❌ Don't commit commented-out code
- ❌ Don't mix unrelated changes
- ❌ Don't use generic messages

## Amending Commits

### Fix Last Commit Message
```bash
git commit --amend -m "New commit message"
```

### Fix Older Commit Messages
```bash
# Interactive rebase
git rebase -i HEAD~3

# In editor, change 'pick' to 'reword' for commits to edit
# Save and close, then edit each message
```

## Resources
- [Conventional Commits](https://www.conventionalcommits.org/)
- [How to Write a Git Commit Message](https://chris.beams.io/posts/git-commit/)
- [commitlint](https://commitlint.js.org/)
