# Marty Issue Triage Workflow

Automatically analyze and categorize new issues with Marty AI.

## Overview

This workflow triggers when an issue is opened or edited, and Marty will:
- Categorize the issue (bug/feature/question/refactoring)
- Check if requirements are clear
- Ask clarifying questions if needed
- Estimate complexity
- Add appropriate labels
- Post a comment with analysis

---

## Workflow File

Save this as `.github/workflows/marty-issue-triage.yml`:

```yaml
name: Marty Issue Triage

on:
  issues:
    types: [opened, edited]

permissions:
  contents: read
  issues: write
  id-token: write

jobs:
  triage:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 1  # Fast checkout, only latest commit

      - name: Generate Marty token
        id: marty-token
        uses: actions/create-github-app-token@v2
        with:
          app-id: ${{ secrets.MARTY_APP_ID }}
          private-key: ${{ secrets.MARTY_APP_PRIVATE_KEY }}

      - name: Marty Issue Analysis
        uses: nesalia-inc/marty-action@v1.0.0
        with:
          github_token: ${{ steps.marty-token.outputs.token }}
          prompt: |
            REPO: ${{ github.repository }}
            ISSUE NUMBER: ${{ github.event.issue.number }}
            ISSUE TITLE: ${{ github.event.issue.title }}
            ISSUE BODY: ${{ github.event.issue.body }}
            AUTHOR: ${{ github.event.issue.user.login }}

            You are Marty, an intelligent development assistant. Analyze this issue:

            ## Analysis Tasks

            1. **Categorize**: Is this a bug, feature request, question, or refactoring?
            2. **Clarity Check**: Is the request clear and specific enough?
            3. **Missing Info**: What additional information is needed?
            4. **Complexity**: Estimate (trivial/simple/complex/very-complex)
            5. **Feasibility**: Can this be implemented automatically?

            ## Labels to Add

            Add appropriate labels using:
            `gh issue edit $NUMBER --add-label "label1,label2,label3"`

            Suggested labels:
            - Type: `bug`, `feature`, `question`, `refactoring`, `documentation`
            - Complexity: `complexity-trivial`, `complexity-simple`, `complexity-complex`
            - Priority: `priority-critical`, `priority-high`, `priority-medium`, `priority-low`
            - Component: Add component-specific labels if applicable

            ## Response Requirements

            If the issue is UNCLEAR:
            - Ask specific questions in a comment
            - Don't add labels yet (wait for clarification)

            If the issue is CLEAR:
            - Add appropriate labels
            - Post a comment confirming understanding
            - If complex, suggest breaking it down into sub-tasks
            - Estimate implementation time

            Keep responses concise and friendly.

          claude_args: |
            --allowedTools "Bash(gh issue:*),Bash(gh label:*),Bash(gh api:*)"
            --max-turns 10

        env:
          # Z.ai Configuration
          ANTHROPIC_BASE_URL: https://api.z.ai/api/anthropic
          ANTHROPIC_AUTH_TOKEN: ${{ secrets.ZAI_API_KEY }}
          ANTHROPIC_DEFAULT_SONNET_MODEL: glm-4.7
          ANTHROPIC_DEFAULT_HAIKU_MODEL: glm-4.5-air
          ANTHROPIC_DEFAULT_OPUS_MODEL: glm-4.7
```

---

## Required Setup

### 1. GitHub Secrets

Add these secrets to your repository (Settings → Secrets and variables → Actions):

| Secret Name | Description | Example |
|-------------|-------------|---------|
| `MARTY_APP_ID` | Marty GitHub App ID | `2883066` |
| `MARTY_APP_PRIVATE_KEY` | Private key from Marty GitHub App | Download from GitHub App settings |
| `ZAI_API_KEY` | Z.ai API key | Your Z.ai platform key |

### 2. GitHub App Installation

1. Go to https://github.com/apps/marty
2. Click "Install"
3. Select your repository/organization
4. Grant permissions:
   - Contents: Read/Write
   - Issues: Read/Write
   - Pull requests: Read/Write
   - Metadata: Read

### 3. Permissions

The workflow requires:
- `contents: read` - To read repository files
- `issues: write` - To add labels and post comments
- `id-token: write` - For GitHub App authentication

---

## Why Checkout is Required

⚠️ **Important:** The `actions/checkout@v4` step is **mandatory** even though issue triage doesn't modify code.

**Why?**
- GitHub CLI (`gh`) requires a git repository context to know which repository to operate on
- Without checkout, there's no `.git` directory
- Without `.git`, commands like `gh issue edit` fail with: `fatal: not a git repository`

**Performance:** Using `fetch-depth: 1` makes the checkout very fast by only downloading the latest commit, not the full history.

---

## Example: Clear Issue

### User Opens Issue

```markdown
## Bug: Login fails with special characters in password

**Problem:**
When a password contains special characters like `@` or `#`, the login endpoint returns a 500 error.

**Steps to reproduce:**
1. Create user with password: `Test@123#`
2. Try to login with these credentials
3. Receive 500 error

**Expected:** Login should succeed
**Actual:** 500 error

**File:** `/packages/api/src/auth/login.ts`
```

### Marty's Response

**Labels added:**
- `bug`
- `complexity-simple`
- `priority-medium`
- `auth`

**Comment posted:**
> Hi! Thanks for the detailed bug report.
>
> **Analysis:**
> - ✅ **Category:** Bug
> - ✅ **Clarity:** Very clear, good reproduction steps
> - ✅ **Complexity:** Simple - likely an encoding/validation issue
> - ✅ **Feasibility:** Can be auto-implemented
>
> **Assessment:**
> This appears to be a password encoding issue. The special characters are probably not being properly URL-encoded or escaped in the API request.
>
> **Estimated fix time:** 15-30 minutes
>
> Ready to implement when you are! Just say `@marty implements this`.

---

## Example: Unclear Issue

### User Opens Issue

```markdown
## Feature: Add caching

We need caching for performance.
```

### Marty's Response

**Labels added:** None yet

**Comment posted:**
> Hi! I'd like to help implement caching, but I need a bit more information:
>
> **Questions:**
> 1. What specifically should be cached? (API responses, database queries, computed results?)
> 2. Which cache backend? (Redis, Memcached, in-memory?)
> 3. Cache TTL? (How long should data be cached?)
> 4. Cache invalidation strategy? (Time-based, manual, event-based?)
> 5. Which endpoints or features need caching?
>
> Once you provide these details, I can categorize and estimate the implementation.

---

## Label System

### Type Labels
- `bug` - Software defect
- `feature` - New functionality
- `question` - Question or discussion
- `refactoring` - Code restructuring
- `documentation` - Documentation improvements

### Complexity Labels
- `complexity-trivial` - < 30 minutes, 1-2 files
- `complexity-simple` - 30min - 2 hours, few files
- `complexity-complex` - 2-8 hours, multiple files/components
- `complexity-very-complex` - > 8 hours, architectural changes

### Priority Labels
- `priority-critical` - Breaking production, security issue, data loss
- `priority-high` - Major functionality broken
- `priority-medium` - Important but not blocking
- `priority-low` - Nice to have, minor issues

### Component Labels (Optional)
Add your own based on your project:
- `auth`, `api`, `database`, `ui`, `performance`, etc.

---

## Customization

### Add Component-Specific Labels

Edit the prompt to include your project's components:

```yaml
prompt: |
  ## Component Labels

  Add appropriate component labels:
  - `auth` - Authentication/authorization
  - `api` - API endpoints
  - `database` - Database queries and schema
  - `ui` - User interface components
  - `performance` - Performance optimizations
```

### Adjust Max Turns

For quick triage:
```yaml
claude_args: |
  --max-turns 5
```

For detailed analysis:
```yaml
claude_args: |
  --max-turns 15
```

### Use Faster Model for Quick Triage

```yaml
env:
  ANTHROPIC_DEFAULT_HAIKU_MODEL: glm-4.5-air  # Use for all operations
```

This reduces cost and latency for simple categorization tasks.

---

## Workflow Triggers

### Current Triggers

```yaml
on:
  issues:
    types: [opened, edited]
```

- `opened` - Triggers when a new issue is created
- `edited` - Triggers when an issue is modified (title/body changes)

### Optional: Manual Trigger

```yaml
on:
  issues:
    types: [opened, edited]
  workflow_dispatch:  # Add manual trigger
```

This allows you to manually run triage from the Actions tab.

### Optional: Exclude Bot Edits

```yaml
on:
  issues:
    types: [opened, edited]
jobs:
  triage:
    if: github.event.sender.type == 'User'  # Only run for human users
```

This prevents the workflow from running when bots edit issues.

---

## Troubleshooting

### Issue: "fatal: not a git repository"

**Cause:** Missing `actions/checkout@v4` step

**Solution:** Add the checkout step before Marty runs:
```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 1
```

---

### Issue: "Resource not accessible by integration"

**Cause:** Marty GitHub App doesn't have write permissions

**Solution:**
1. Go to GitHub App settings
2. Edit repository permissions
3. Set "Issues" to "Read/Write"

---

### Issue: Marty doesn't add labels

**Cause:** Command syntax error or missing label creation

**Solution:**
1. Create labels in repository settings first
2. Or use `gh label create` before adding:
```yaml
prompt: |
  Create label if needed, then add it:
  gh label create "complexity-simple" --color "#f4d5b1" || true
  gh issue edit $NUMBER --add-label "complexity-simple"
```

---

### Issue: Workflow is slow

**Cause:** Full repository checkout or too many max-turns

**Solutions:**
1. Use `fetch-depth: 1` in checkout
2. Reduce `max-turns` to 5-8
3. Use faster model (glm-4.5-air)

---

## Performance Tips

### 1. Shallow Checkout

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 1  # Only latest commit
```

### 2. Limit Max Turns

For simple triage:
```yaml
claude_args: |
  --max-turns 5
```

### 3. Use Fast Model

```yaml
env:
  ANTHROPIC_DEFAULT_HAIKU_MODEL: glm-4.5-air  # Use Haiku for everything
```

### 4. Skip for Edits

Only triage on issue creation, not edits:
```yaml
on:
  issues:
    types: [opened]  # Remove "edited"
```

---

## Advanced: Conditional Triage

### Only Triage Specific Repos

```yaml
jobs:
  triage:
    if: github.repository == 'nesalia-inc/production-repo'
```

### Only Triage Complex Issues

```yaml
jobs:
  triage:
    if: contains(github.event.issue.body, 'complexity-complex') == false
```

### Require Minimum Issue Length

```yaml
jobs:
  triage:
    if: ${{ github.event.issue.body != null && github.event.issue.body.length > 50 }}
```

---

## Monitoring

### Track Triage Effectiveness

Create metrics dashboard showing:

| Metric | How to Measure |
|--------|----------------|
| Triage accuracy | Compare Marty's labels vs. human edits |
| Response time | Average workflow duration |
| Questions asked | % of issues needing clarification |
| Auto-implementation rate | % of triaged issues that get implemented |

### Review Logs Regularly

Check workflow runs periodically to:
- Identify common confusion patterns
- Improve prompts based on feedback
- Add missing labels
- Adjust complexity estimates

---

## Related Workflows

- **[Issue Response](./ISSUE_RESPONSE_WORKFLOW.md)** - Interactive dialogue via @marty mentions
- **[Implementation](./IMPLEMENTATION_WORKFLOW.md)** - Auto-implement issues when requested
- **[PR Review](./PR_REVIEW_WORKFLOW.md)** - Auto-review pull requests

---

## Next Steps

1. **Create the workflow file** in `.github/workflows/marty-issue-triage.yml`
2. **Add required secrets** (MARTY_APP_ID, MARTY_APP_PRIVATE_KEY, ZAI_API_KEY)
3. **Install Marty GitHub App** on your repository
4. **Test with a sample issue**
5. **Customize labels** based on your project structure
6. **Monitor and iterate** based on results

---

## Support

For issues or questions:
- Check the main [Marty documentation](./README.md)
- Review [troubleshooting guide](./docs/faq.md)
- Open an issue on the [Marty repository](https://github.com/nesalia-inc/marty-action)
