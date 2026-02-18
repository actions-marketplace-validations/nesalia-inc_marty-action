# Marty Troubleshooting Guide

Complete guide for debugging and fixing common Marty workflow errors.

---

## 🔍 Quick Diagnosis

### Step 1: Enable Full Output

Always start by enabling full output to see the actual error:

```yaml
- uses: nesalia-inc/marty-action@v1.0.0
  with:
    github_token: ${{ steps.marty-token.outputs.token }}
    show_full_output: true  # ← Critical for debugging
    prompt: |
      Your prompt here...
```

This shows:
- Full Claude Code execution logs
- Error messages (not truncated)
- Tool usage and permissions
- Token usage and costs

---

## 🚨 Common Errors and Solutions

### Error 1: "Claude Code process exited with code 1"

**Symptoms:**
```
SDK execution error: Error: Claude Code process exited with code 1
```

**Causes & Solutions:**

#### Cause 1A: MCP Server Error

**Problem:** GitHub inline comment MCP server crashes

**Detection:**
```json
{
  "mcp-config": "{...github_inline_comment...}"
}
```

**Solution A - Disable MCP temporarily:**
```yaml
claude_args: |
  --allowedTools "Bash(gh pr comment:*)"
  --max-turns 15
  # Remove: mcp__github_inline_comment__create_inline_comment
```

**Solution B - Fix MCP server:**
- Check MCP server logs
- Verify GITHUB_TOKEN has correct permissions
- Ensure PR_NUMBER is valid

---

#### Cause 1B: Timeout

**Problem:** Workflow exceeds time limit

**Detection:**
```
duration_ms: 478630  # 8 minutes
```

**Solution:**
```yaml
- uses: nesalia-inc/marty-action@v1.0.0
  timeout-minutes: 30  # Increase from default 15
```

**Or reduce complexity:**
```yaml
claude_args: |
  --max-turns 10  # Reduce from 25
```

---

#### Cause 1C: Memory/Resource Issues

**Problem:** Runner runs out of memory

**Detection:**
```
exitCode: 1
No specific error message
```

**Solution:**
```yaml
jobs:
  review:
    runs-on: ubuntu-latest
    # Or use larger runner:
    # runs-on: ubuntu-latest-large
```

---

#### Cause 1D: Syntax Error in Generated Content

**Problem:** Marty generates invalid code/markdown

**Detection:**
```json
{
  "permission_denials": [],
  "is_error": true
}
```

**Solution:**
```yaml
prompt: |
  ## Output Format Requirements

  CRITICAL: Your output must be valid:
  - No unclosed code blocks
  - No unescaped special characters
  - Valid JSON if using jq
  - Valid markdown syntax

  Test your command before running:
  echo "test" | gh comment edit $COMMENT_ID --body-file -
```

---

### Error 2: Permission Denials

**Symptoms:**
```json
{
  "permission_denials": [
    {
      "tool_name": "Bash",
      "tool_input": "gh issue comment ..."
    }
  ]
}
```

**Causes & Solutions:**

#### Cause 2A: Wrong allowedTools Pattern

**Problem:** Pattern doesn't match command

**Example:**
```yaml
# ❌ WRONG - Only matches commands STARTING with "gh issue"
--allowedTools "Bash(gh issue:*)"

# Command that fails:
ISSUE_NUMBER=46 RESPONSE=$(gh issue comment ...) && ...
# Starts with "ISSUE_NUMBER" not "gh issue"
```

**Solution:**
```yaml
# ✅ CORRECT - Matches any command containing "gh"
--allowedTools "Bash(gh:*)"

# ✅ OR - Allow all bash (less secure)
--allowedTools "Bash(*)"
```

---

#### Cause 2B: Missing Tools

**Problem:** Required tool not in allowedTools

**Detection:**
```json
{
  "tool_name": "Read",
  "permission_denials": [...]
}
```

**Solution:**
```yaml
claude_args: |
  --allowedTools "Bash(gh:*),Bash(jq:*),Read"
  # Add missing tools
```

**Common tools needed:**
| Tool | When Needed |
|------|-------------|
| `Bash(gh:*)` | Any GitHub CLI operation |
| `Bash(jq:*)` | Parsing JSON responses |
| `Bash(git:*)` | Git operations |
| `Read` | Reading files |
| `Write` | Creating files |
| `Edit` | Modifying files |

---

### Error 3: Authentication Failures

**Symptoms:**
```
Error: 401 Unauthorized
failed to run git: Permission denied
```

**Causes & Solutions:**

#### Cause 3A: Missing GitHub App Installation

**Detection:**
```
Failed to create token for "repo-name" (attempt 1): Not Found
```

**Solution:**
1. Go to https://github.com/apps/marty
2. Click "Install"
3. Select your repository
4. Grant required permissions

---

#### Cause 3B: Wrong Token Passing

**Detection:**
```
Requesting OIDC token...
App token exchange failed: 401 Unauthorized
```

**Solution:**
```yaml
# ❌ WRONG - env doesn't propagate to TypeScript
env:
  OVERRIDE_GITHUB_TOKEN: ${{ steps.marty-token.outputs.token }}

# ✅ CORRECT - Use with input
with:
  github_token: ${{ steps.marty-token.outputs.token }}
```

---

#### Cause 3C: Insufficient Permissions

**Detection:**
```
Resource not accessible by integration
```

**Solution:**
```yaml
permissions:
  contents: write  # Or read for some workflows
  pull-requests: write
  issues: write
  id-token: write  # Critical for OIDC
```

---

### Error 4: "fatal: not a git repository"

**Symptoms:**
```
fatal: not a git repository
gh issue comment failed
```

**Cause:** Missing checkout step

**Solution:**
```yaml
jobs:
  triage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4  # ← REQUIRED!
        with:
          fetch-depth: 1

      - uses: nesalia-inc/marty-action@v1.0.0
        # ...
```

**Why:** GitHub CLI (`gh`) needs `.git` directory to identify the repository.

---

### Error 5: "Claude Code is not installed"

**Symptoms:**
```
Claude Code is not installed on this repository
```

**Cause:** GitHub App not installed on repository

**Solution:** Install Marty GitHub App on the repository

---

### Error 6: Environment Variable Validation Failed

**Symptoms:**
```
Environment variable validation failed:
  - Either ANTHROPIC_API_KEY or CLAUDE_CODE_OAUTH_TOKEN is required
```

**Cause:** Custom provider not supported in validation

**Solution:**
The validation in `base-action/src/validate-env.ts` needs to support custom providers.

**If you control marty-action:**
```typescript
// In validate-env.ts
const anthropicBaseUrl = process.env.ANTHROPIC_BASE_URL;
const anthropicAuthToken = process.env.ANTHROPIC_AUTH_TOKEN;
const hasCustomProvider = anthropicBaseUrl && (anthropicAuthToken || anthropicApiKey);

if (!anthropicApiKey && !claudeCodeOAuthToken && !hasCustomProvider) {
    errors.push("Either ANTHROPIC_API_KEY, CLAUDE_CODE_OAUTH_TOKEN, or ANTHROPIC_BASE_URL...");
}
```

**If using published action:** Use environment variables that bypass validation:

```yaml
env:
  ANTHROPIC_BASE_URL: https://openrouter.ai/api
  ANTHROPIC_AUTH_TOKEN: ${{ secrets.OPENROUTER_API_KEY }}
  ANTHROPIC_API_KEY: "dummy"  # Satisfies validation
```

---

### Error 7: Model Not Found

**Symptoms:**
```
Error: Model 'glm-4.7' not found
```

**Cause:** Wrong model name for provider

**Solution:** Use correct model names for your provider

**Z.ai models:**
```yaml
env:
  ANTHROPIC_DEFAULT_SONNET_MODEL: glm-4.7
  ANTHROPIC_DEFAULT_HAIKU_MODEL: glm-4.5-air
```

**OpenRouter models:**
```yaml
env:
  ANTHROPIC_DEFAULT_SONNET_MODEL: anthropic/claude-sonnet-4-20250514
  ANTHROPIC_DEFAULT_HAIKU_MODEL: anthropic/claude-3-5-haiku-20241022
```

**Anthropic models:**
```yaml
env:
  ANTHROPIC_DEFAULT_SONNET_MODEL: claude-sonnet-4-20250514
  ANTHROPIC_DEFAULT_HAIKU_MODEL: claude-3-5-haiku-20241022
```

---

### Error 8: Max Turns Exceeded

**Symptoms:**
```json
{
  "subtype": "error_max_turns",
  "num_turns": 25
}
```

**Cause:** Task too complex for max-turns limit

**Solution:**
```yaml
claude_args: |
  --max-turns 40  # Increase from 25
```

**Or simplify the task:**
```yaml
prompt: |
  Focus on the most critical aspects only:
  1. Security vulnerabilities
  2. Breaking changes

  Skip: style suggestions, minor optimizations
```

---

## 🔧 Diagnostic Commands

### Check Workflow Logs

```bash
# Using gh CLI
gh run view <run-id> --log

# Or in browser:
https://github.com/<owner>/<repo>/actions/runs/<run-id>
```

### Test Locally with act

```bash
# Install act
brew install act  # macOS
# Or: https://github.com/nektos/act

# Run workflow locally
act -j <job-name> --secret MARTY_APP_ID=<id> --secret MARTY_APP_PRIVATE_KEY=<key>
```

### Validate YAML Syntax

```bash
# Check workflow syntax
yamllint .github/workflows/marty-*.yml

# Or use GitHub CLI
gh workflow view <workflow-name> --yaml
```

---

## 📊 Error Patterns by Workflow Type

### Issue Triage Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `fatal: not a git repository` | Missing checkout | Add `actions/checkout@v4` |
| `gh issue edit failed` | Wrong allowedTools | Use `Bash(gh:*)` |
| No labels added | Prompt unclear | Simplify instructions |
| Timeout | Too many issues to analyze | Reduce scope or max-turns |

### Issue Response Errors

| Error | Cause | Solution |
|-------|-------|----------|
| Progress comment not edited | jq permission denied | Add `Bash(jq:*)` |
| Response too generic | Not reading full history | Emphasize reading all comments |
| No response posted | gh comment failed | Check `issues: write` permission |

### Implementation Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `git push failed` | No write permissions | Add `contents: write` |
| Branch not created | git config failed | Check git authentication |
| PR not created | gh pr failed | Check `pull-requests: write` |
| Timeout | Task too complex | Break into smaller issues |

### PR Review Errors

| Error | Cause | Solution |
|-------|-------|----------|
| MCP inline comment fails | Server crash | Disable MCP temporarily |
| No review posted | Wrong allowedTools | Ensure `gh pr comment` allowed |
| Timeout | Large PR / low max-turns | Increase max-turns or timeout |
| Generic review | Prompt too complex | Simplify review criteria |

---

## 🛠️ Debugging Workflow

### Step 1: Identify the Error Type

```yaml
# Enable full output
- uses: nesalia-inc/marty-action@v1.0.0
  with:
    show_full_output: true
```

Look for:
- `permission_denials` → Tools/permissions issue
- `exitCode: 1` → Execution error
- `timeout` → Time limit exceeded
- HTTP error codes → Authentication/permissions

---

### Step 2: Check Permissions

```yaml
permissions:
  contents: read       # Or write for implementation
  pull-requests: write # For PR reviews
  issues: write        # For issue triage/response
  id-token: write      # Required for OIDC
```

**Verify in GitHub App settings:**
1. Go to repository Settings → Applications
2. Click "Marty" GitHub App
3. Check permissions match workflow

---

### Step 3: Verify Secrets

```bash
# List secrets
gh secret list

# Check specific secret exists
gh secret get MARTY_APP_ID
gh secret get MARTY_APP_PRIVATE_KEY
gh secret get OPENROUTER_API_KEY  # Or ZAI_API_KEY
```

---

### Step 4: Test with Minimal Workflow

Create a test workflow with minimal config:

```yaml
name: Marty Test

on:
  workflow_dispatch:

permissions:
  contents: read
  issues: write
  id-token: write

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Testing Marty"
      - uses: nesalia-inc/marty-action@v1.0.0
        with:
          github_token: ${{ secrets.MARTY_APP_ID }}  # Wrong token type
        env:
          ANTHROPIC_BASE_URL: https://openrouter.ai/api
          ANTHROPIC_AUTH_TOKEN: ${{ secrets.OPENROUTER_API_KEY }}
```

**Expected result:** Should fail with clear error

---

### Step 5: Incremental Complexity

Once minimal works, add features one at a time:

1. ✅ Basic test (simple prompt)
2. ✅ Add progress display
3. ✅ Add allowedTools
4. ✅ Add file operations
5. ✅ Add git operations

---

## 📝 Logging Best Practices

### 1. Progress Markers

```yaml
prompt: |
  Step 1: Post progress
  gh issue comment $NUMBER --body "Working..."

  Step 2: Do work
  # ... work ...

  Step 3: Update progress
  gh issue edit comment $ID --body "Still working..."

  Step 4: Final update
  gh issue edit comment $ID --body "Done!"
```

### 2. Error Handling in Prompt

```yaml
prompt: |
  If a command fails:
  1. Check the error message
  2. Try alternative approach
  3. If still failing, post error as comment
  4. Don't crash the workflow
```

### 3. Verbose Mode for Debugging

```yaml
prompt: |
  ## DEBUG MODE

  Before each command, log what you're doing:
  echo "About to run: <command>"

  After each command, log result:
  echo "Result: <output>"

  This helps identify where failures occur.
```

---

## 🚀 Performance Tuning

### Reduce Execution Time

```yaml
# 1. Reduce max-turns
claude_args: |
  --max-turns 10  # Instead of 25

# 2. Use faster model
env:
  ANTHROPIC_DEFAULT_HAIKU_MODEL: anthropic/claude-3-5-haiku-20241022

# 3. Limit scope
prompt: |
  Only check these files:
  - src/api/*.ts
  - lib/services/*.ts
```

### Increase Reliability

```yaml
# 1. Add timeout
- uses: nesalia-inc/marty-action@v1.0.0
  timeout-minutes: 30

# 2. Add retry logic
prompt: |
  If a command fails, wait 5s and retry once:
  command || (sleep 5 && command)

# 3. Use OpenRouter for failover
env:
  ANTHROPIC_BASE_URL: https://openrouter.ai/api
```

---

## 📞 Getting Help

### Before Asking for Help

1. ✅ Enable `show_full_output: true`
2. ✅ Check all secrets are set
3. ✅ Verify GitHub App is installed
4. ✅ Validate permissions
5. ✅ Test with minimal workflow
6. ✅ Search existing issues

### When Opening an Issue

Include:
- **Workflow file** (full YAML)
- **Error message** (complete, not truncated)
- **Workflow run link** (GitHub Actions URL)
- **Repository type** (public/private)
- **Provider** (Z.ai, OpenRouter, Anthropic)
- **What you tried** (debugging steps)

### Useful Resources

- [Marty Documentation](./README.md)
- [GitHub Actions Logs](https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/using-workflow-run-logs)
- [GitHub CLI](https://cli.github.com/)
- [act (local testing)](https://github.com/nektos/act)

---

## 🎯 Quick Reference Card

### Common Fixes

| Error | Quick Fix |
|-------|-----------|
| `fatal: not a git repository` | Add `actions/checkout@v4` |
| `permission_denials` | Change `Bash(gh issue:*)` → `Bash(gm:*)` |
| `401 Unauthorized` | Install GitHub App / Check permissions |
| `Model not found` | Use correct model name for provider |
| `exit code 1` | Enable `show_full_output: true` |
| Timeout | Increase `timeout-minutes` or reduce `max-turns` |

### Essential Config

```yaml
# Always include these
permissions:
  contents: read/write
  issues: write
  pull-requests: write
  id-token: write

steps:
  - uses: actions/checkout@v4
  - uses: nesalia-inc/marty-action@v1.0.0
    with:
      show_full_output: true  # When debugging
```

### Provider Config

```yaml
# Z.ai
env:
  ANTHROPIC_BASE_URL: https://api.z.ai/api/anthropic
  ANTHROPIC_AUTH_TOKEN: ${{ secrets.ZAI_API_KEY }}

# OpenRouter
env:
  ANTHROPIC_BASE_URL: https://openrouter.ai/api
  ANTHROPIC_AUTH_TOKEN: ${{ secrets.OPENROUTER_API_KEY }}
  ANTHROPIC_API_KEY: ""  # Critical!

# Anthropic
env:
  ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

---

## Checklist: Is Your Workflow Ready?

- [ ] GitHub App installed on repository
- [ ] All secrets configured (MARTY_APP_ID, MARTY_APP_PRIVATE_KEY, API key)
- [ ] Permissions correct in workflow YAML
- [ ] `actions/checkout@v4` included (if needed)
- [ ] `allowedTools` includes required tools
- [ ] Provider models correctly named
- [ ] `max-turns` appropriate for task complexity
- [ ] `timeout-minutes` set (if needed)
- [ ] Tested with simple prompt first
- [ ] `show_full_output: true` when debugging

---

**Remember:** 90% of errors are due to:
1. Missing `actions/checkout@v4`
2. Wrong `allowedTools` patterns
3. GitHub App not installed
4. Wrong provider/model names

Start with these! 🚀
