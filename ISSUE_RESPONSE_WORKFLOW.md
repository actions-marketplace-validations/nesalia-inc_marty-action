# Marty Issue Response Workflow

Enable interactive dialogue with Marty via mentions on issue comments.

## ⚠️ Important: Marty's Username

**Marty's GitHub username is `@martyy-code`** (not `@marty`).

All triggers should use `@martyy-code` instead of `@marty`:

```yaml
# ❌ Wrong
if: contains(github.event.comment.body, '@marty')

# ✅ Correct
if: contains(github.event.comment.body, '@martyy-code')
```

---

## Overview

This workflow enables interactive dialogue on issues. When someone mentions `@martyy-code` in a comment, Marty:
- Reads the entire issue history
- Understands the full context
- Responds intelligently to questions
- Provides technical insights
- Offers implementation help when requested

---

## Workflow File

Save this as `.github/workflows/marty-issue-response.yml`:

```yaml
name: Marty Issue Response

on:
  issue_comment:
    types: [created]

permissions:
  contents: read
  issues: write
  id-token: write

jobs:
  respond:
    # Trigger only when @martyy-code is mentioned
    if: contains(github.event.comment.body, '@martyy-code')
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

      - name: Marty Responds to Comment
        uses: nesalia-inc/marty-action@v1.0.0
        with:
          github_token: ${{ steps.marty-token.outputs.token }}
          prompt: |
            REPO: ${{ github.repository }}
            ISSUE NUMBER: ${{ github.event.issue.number }}
            COMMENT AUTHOR: ${{ github.event.comment.user.login }}
            COMMENT: ${{ github.event.comment.body }}

            You are Marty, mentioned in an issue comment.

            ## ⚡ PROGRESSION DISPLAY (Required)

            CRITICAL: Show users that Marty is working while thinking!

            Step 1: Post a progress comment IMMEDIATELY:
            ```bash
            gh issue comment $ISSUE_NUMBER --body "🤖 **Marty is analyzing your request...** ⏳"
            ```

            Capture the comment ID from the response (it will be in the JSON output).

            Step 2: Read the ENTIRE issue history (title, body, all previous comments)
            Step 3: Understand the full context and current state of discussion
            Step 4: Formulate your intelligent response
            Step 5: EDIT the progress comment with your actual response:
            ```bash
            gh issue edit comment $COMMENT_ID --body "Your final response here ✅"
            ```

            This way, users see Marty is working while you think, instead of wondering if anything is happening!

            ## Your Task

            1. Read the ENTIRE issue history (title, body, all previous comments)
            2. Understand the full context and current state of discussion
            3. Respond intelligently to the latest comment
            4. If someone asks questions, answer them clearly
            5. If technical discussion, provide helpful insights
            6. If implementation is discussed, offer to help implement when ready

            ## Important Guidelines

            - Be concise but thorough
            - Reference specific code or files when helpful
            - Ask clarifying questions if something is unclear
            - Be friendly and professional
            - Don't mention that you're an AI (just be helpful)

            ## How to Extract Comment ID

            When posting a comment, `gh issue comment` returns JSON like:
            ```json
            {
              "id": "1234567890",
              "body": "🤖 **Marty is analyzing your request...** ⏳",
              ...
            }
            ```

            Extract the ID using:
            ```bash
            RESPONSE=$(gh issue comment $ISSUE_NUMBER --body "...")
            COMMENT_ID=$(echo "$RESPONSE" | jq -r '.id')
            ```

            Then edit it later:
            ```bash
            gh issue edit comment $COMMENT_ID --body "New content"
            ```

          claude_args: |
            --allowedTools "Bash(gh:*),Bash(jq:*),Read"
            --max-turns 8

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

Same as other workflows:

| Secret Name | Description | Example |
|-------------|-------------|---------|
| `MARTY_APP_ID` | Marty GitHub App ID | `2883066` |
| `MARTY_APP_PRIVATE_KEY` | Private key from Marty GitHub App | Download from GitHub App settings |
| `ZAI_API_KEY` | Z.ai API key | Your Z.ai platform key |

### 2. GitHub App Installation

Ensure Marty GitHub App is installed with:
- Contents: Read/Write
- Issues: Read/Write
- Pull requests: Read/Write
- Metadata: Read

### 3. Permissions

- `contents: read` - To read repository files for context
- `issues: write` - To post comments
- `id-token: write` - For GitHub App authentication

---

## Why Checkout is Required

Even though responding doesn't modify code, `actions/checkout@v4` is required because:
- Marty needs to read code files to provide technical context
- GitHub CLI (`gh`) requires git repository context
- Without `.git` directory, `gh issue comment` fails

---

## Usage Examples

### Example 1: Technical Question

**User comments:**
> @martyy-code How does the `invalidateByTag()` method work internally?

**Marty responds:**
> Looking at `packages/functions/src/events/stream.ts`, the `invalidateByTag()` method currently iterates over subscriptions with matching tag filters.
>
> However, I notice there's a bug mentioned in issue #45 - it only checks subscriptions, not actual cache entries. The method should track all cache entries by tag, not just those with active subscriptions.
>
> Would you like me to explain the proposed fix in detail?

---

### Example 2: Clarification Request

**User comments:**
> @martyy-code Can you clarify what you meant by "complexity-simple" in your triage comment?

**Marty responds:**
> Sure! By "complexity-simple", I mean:
>
> - **Estimated time:** 30 minutes to 2 hours
> - **Files affected:** Few files (typically 1-5)
> - **Risk level:** Low
> - **Testing required:** Unit tests for new/modified code
> - **Auto-implementable:** ✅ Yes, I can implement this automatically
>
> For this particular bug fix, it involves:
> 1. Adding a `Map` to track tagged entries
> 2. Modifying 2-3 methods
> 3. Adding a few unit tests
>
> Ready to implement when you are!

---

### Example 3: Implementation Discussion

**User comments:**
> @martyy-code Should we use Redis or Memcached for caching?

**Marty responds:**
> Great question! Here's my analysis:
>
> **Redis:**
> - ✅ More feature-rich (pub/sub, transactions, data structures)
> - ✅ Better persistence options
> - ✅ Larger community and ecosystem
> - ❌ Slightly more complex setup
> - ❌ More memory overhead
>
> **Memcached:**
> - ✅ Simpler, lighter
> - ✅ Very fast for simple key-value
> - ❌ No persistence
> - ❌ Limited data structures
>
> **Recommendation:** Unless you need advanced features, **Redis** is usually the better choice due to:
> - Better ecosystem and tooling
> - Easier to scale
> - More future-proof
>
> Also consider that many hosting providers (AWS ElastiCache, Heroku Redis) have managed Redis services.
>
> Want me to help implement Redis caching?

---

### Example 4: Request for Implementation

**User comments:**
> @martyy-code implements this issue

**Marty responds:**
> On it! I'll:
> 1. Create a branch `marty/issue-XX`
> 2. Implement the fix
> 3. Add tests
> 4. Create a PR
> 5. Self-review the PR
>
> Estimated time: 30-60 minutes
>
> Starting now... ✅

*(Note: This triggers the implementation workflow if you have it set up)*

---

## 🎨 Progress Display System

### Why Progress Display Matters

When someone mentions `@martyy-code`, they expect a quick response. However:
- LLM generation can take 30 seconds to 2 minutes
- Users might wonder if Marty is working
- A progress indicator improves UX significantly

### How It Works

```
User: "@martyy-code how does X work?"
    ↓
Workflow triggers
    ↓
Marty posts: "🤖 Marty is analyzing... ⏳"  (User sees this immediately!)
    ↓
Marty reads issue history (5-10s)
    ↓
Marty formulates response (20-60s)
    ↓
Marty edits comment with actual response ✅
```

### Visual Example

**Initial comment (appears in 2-3 seconds):**
> 🤖 **Marty is analyzing your request...** ⏳

**After ~30-60 seconds (edited):**
> Great question! Looking at the code... [full response] ✅

### Technical Implementation

The prompt instructs Marty to:

1. **Post progress immediately**
   ```bash
   gh issue comment $NUMBER --body "🤖 **Marty is analyzing...** ⏳"
   ```

2. **Capture comment ID** from JSON response
   ```bash
   RESPONSE=$(gh issue comment $NUMBER --body "...")
   COMMENT_ID=$(echo "$RESPONSE" | jq -r '.id')
   ```

3. **Edit with real response**
   ```bash
   gh issue edit comment $COMMENT_ID --body "Actual response ✅"
   ```

### Progress Message Variations

You can customize the progress messages:

```yaml
prompt: |
  Post a progress comment (choose based on what you're doing):
  - "🤖 **Marty is reading the issue...** 📖"
  - "🤖 **Marty is analyzing the code...** 🔍"
  - "🤖 **Marty is thinking...** 🤔"
  - "🤖 **Marty is formulating a response...** ✍️"
```

### Animated Progress (Optional)

For a more dynamic feel, update the progress multiple times:

```yaml
prompt: |
  Post initial progress: "🤖 **Marty is analyzing...** ⏳"

  Then update while working:
  - After reading: "🤖 **Marty has read the issue, now thinking...** 🤔"
  - After analyzing: "🤖 **Marty is preparing the response...** ✍️"
  - Final: Edit with actual response ✅
```

### Benefits

| Benefit | Description |
|---------|-------------|
| **User confidence** | Users see Marty is working |
| **Reduced duplicates** | Fewer "anyone here?" follow-ups |
| **Better UX** | Clear communication of state |
| **Professional** | Shows activity, not silence |

---

## 🔧 Required Tools Configuration

### Why `Bash(gh:*)` and `Bash(jq:*)` are Required

The progress display system requires Marty to:

1. **Post comments** using `gh issue comment`
2. **Capture IDs** using `jq` to parse JSON responses
3. **Edit comments** using `gh issue edit`

Without these tools, Marty cannot:
- Post the progress indicator
- Extract the comment ID for editing
- Update the comment with the final response

### Allowed Tools Breakdown

```yaml
--allowedTools "Bash(gh:*),Bash(jq:*),Read"
```

| Tool | Purpose | Example Commands |
|------|---------|------------------|
| `Bash(gh:*)` | GitHub CLI operations | `gh issue comment`, `gh issue edit`, `gh issue view` |
| `Bash(jq:*)` | JSON parsing | `jq -r '.id'`, `jq -r '.body'` |
| `Read` | Read repository files | Understanding code context |

### Security Considerations

**With `Bash(gh:*)`, Marty can:**
- ✅ Post, edit, delete comments
- ✅ Add/remove labels
- ✅ Edit issues
- ❌ Cannot modify code (no `Write` tool)
- ❌ Cannot run arbitrary bash commands (only `gh` and `jq`)

**If you need even more permissive:**

```yaml
--allowedTools "Bash(*),Read"
```

This allows ANY bash command but is **less secure**. Use only for trusted repositories.

**For maximum security (no progress display):**

```yaml
--allowedTools "Bash(gh issue comment:*)"
```

This only allows posting comments, not editing (so no progress display).

### What Happened Without Proper Tools

**Original problematic config:**
```yaml
--allowedTools "Bash(gh issue:*),Bash(gh api:*)"
```

**Problem:** Commands with variable assignments don't match the pattern:

```bash
# ❌ Doesn't match "Bash(gh issue:*)" pattern
ISSUE_NUMBER=46 RESPONSE=$(gh issue comment ...) && ...

# ✅ Would match
gh issue comment $ISSUE_NUMBER --body "..."
```

**Solution:** Use `Bash(gh:*)` to match ANY command starting with `gh`, regardless of variable assignments or pipes.

---

## Advanced Configuration

### Adjust Response Length

For shorter responses:
```yaml
claude_args: |
  --max-turns 5
```

For more detailed responses:
```yaml
claude_args: |
  --max-turns 12
```

### Use Faster Model

For quick questions:
```yaml
env:
  ANTHROPIC_DEFAULT_HAIKU_MODEL: glm-4.5-air  # Use Haiku for everything
```

### Custom Personality

```yaml
prompt: |
  You are Marty, a senior software engineer with 10 years of experience.
  You're helpful, concise, and technical.

  Tone: Professional but friendly
  Style: Direct and practical
  Expertise: Full-stack development, architecture, best practices
```

---

## Trigger Variations

### Multiple Trigger Options

```yaml
if: |
  contains(github.event.comment.body, '@martyy-code') ||
  contains(github.event.comment.body, '@martyy')
```

**Note:** This allows both `@martyy-code` and the shorthand `@marty`.

### Command-Specific Triggers

Trigger only for specific commands:

```yaml
# For implementation requests
if: contains(github.event.comment.body, '@martyy-code implements')

# For explanations only
if: contains(github.event.comment.body, '@martyy-code explain')
```

### Case-Insensitive Trigger

```yaml
if: |
  contains(lower(github.event.comment.body), '@martyy-code') ||
  contains(lower(github.event.comment.body), '@marty')
```

---

## Troubleshooting

### Issue: Workflow doesn't trigger

**Cause:** Comment doesn't exactly match `@martyy-code`

**Examples that won't trigger:**
- `@martyy` (one 'y')
- `@Martyy-Code` (capitalized)
- `@martyy -code` (space)

**Solution:** Remind users to use exact username `@martyy-code`

---

### Issue: Marty responds but comment doesn't appear

**Cause:** Missing `issues: write` permission or authentication failure

**Solution:**
1. Check GitHub App has `Issues: Read/Write` permission
2. Verify secrets are correctly set
3. Check workflow logs for authentication errors

---

### Issue: Marty doesn't have code context

**Cause:** Missing checkout or wrong file paths

**Solution:**
```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 1  # Ensure this is present
```

---

### Issue: Response is too generic

**Cause:** Marty isn't reading the full issue history

**Solution:** Update prompt to emphasize reading history:
```yaml
prompt: |
  ## CRITICAL: Read Everything

  Before responding, you MUST:
  1. Read the issue title
  2. Read the issue body
  3. Read ALL previous comments
  4. Understand the full context
  5. Reference specific points from the discussion

  Only then, respond to the latest comment.
```

---

## Best Practices

### 1. Be Specific in Mentions

❌ **Too vague:**
> @martyy-code help

✅ **Specific:**
> @martyy-code How should I handle JWT token rotation in the auth middleware?

### 2. Provide Context

❌ **No context:**
> @martyy-code this doesn't work

✅ **With context:**
> @martyy-code I'm getting a 500 error when calling POST /api/auth/login with password containing `@` symbol. The error is in the password validation. Any ideas?

### 3. Ask Follow-up Questions

Good Marty responses often ask clarifying questions:
- "Which file are you working in?"
- "What error message do you see?"
- "Can you share the relevant code snippet?"
- "What have you tried so far?"

### 4. Reference Code When Helpful

Marty can reference specific files:
- "Looking at `src/auth/login.ts:45`..."
- "The bug is in the `CacheInvalidationStream` class..."
- "Similar to how it's done in `api/user/routes.ts`..."

---

## Integration with Other Workflows

This workflow works seamlessly with:

### Issue Triage Workflow

1. **Issue created** → Triage workflow categorizes
2. **Marty asks questions** → User responds
3. **User asks @martyy-code** → Response workflow answers
4. **Iterative dialogue** → Issue becomes clear
5. **User says "@martyy-code implements"** → Implementation workflow triggers

### Implementation Workflow

When someone says `@martyy-code implements`:

```yaml
# In implementation workflow
if: |
  contains(github.event.comment.body, '@martyy-code') &&
  (contains(github.event.comment.body, 'implements') ||
   contains(github.event.comment.body, 'implémentes'))
```

This detects the implementation request and triggers auto-implementation.

---

## Performance Optimization

### Reduce Max Turns for Quick Responses

```yaml
claude_args: |
  --max-turns 5  # Faster, shorter responses
```

### Use Faster Model for Simple Questions

```yaml
env:
  # Override to use Haiku for everything
  ANTHROPIC_DEFAULT_SONNET_MODEL: glm-4.5-air
  ANTHROPIC_DEFAULT_HAIKU_MODEL: glm-4.5-air
  ANTHROPIC_DEFAULT_OPUS_MODEL: glm-4.5-air
```

### Skip for Bot Comments

```yaml
jobs:
  respond:
    if: |
      contains(github.event.comment.body, '@martyy-code') &&
      github.event.sender.type == 'User'  # Only humans
```

---

## Monitoring

### Track Response Quality

Metrics to monitor:

| Metric | How to Measure |
|--------|----------------|
| Response helpfulness | User reactions (thumbs up/down) |
| Response time | Average workflow duration |
| Code accuracy | How often Marty references correct files |
| Follow-up questions | % of responses that ask for clarification |

### Review Conversation Patterns

Check workflow runs periodically for:
- Common questions (add to FAQ)
- Confusion points (improve documentation)
- Repeated topics (create issue templates)

---

## Advanced: Conditional Responses

### Only Respond on Specific Repos

```yaml
if: |
  contains(github.event.comment.body, '@martyy-code') &&
  github.repository == 'nesalia-inc/production-repo'
```

### Different Behavior Based on Comment Content

```yaml
- name: Determine response type
  id: response-type
  run: |
    if contains(github.event.comment.body, 'explain'); then
      echo "type=explain" >> $GITHUB_OUTPUT
    elif contains(github.event.comment.body, 'implements'); then
      echo "type=implement" >> $GITHUB_OUTPUT
    else
      echo "type=respond" >> $GITHUB_OUTPUT
    fi

- uses: nesalia-inc/marty-action@v1.0.0
  with:
    prompt: |
      RESPONSE_TYPE: ${{ steps.response-type.outputs.type }}
      # ... rest of prompt
```

---

## Username Mapping Summary

| Context | Username to Use |
|---------|-----------------|
| **GitHub App/Bot** | `martyy-code` (GitHub username) |
| **Trigger detection** | `@martyy-code` |
| **In prompts** | Marty (friendly name) |
| **Git commits** | `marty[bot]` (automatic) |

**Important:** Never use `@marty` in triggers - it won't match the actual bot username!

---

## Related Workflows

- **[Issue Triage](./ISSUE_TRIAGE_WORKFLOW.md)** - Auto-analyze new issues
- **[Implementation](./IMPLEMENTATION_WORKFLOW.md)** - Auto-implement when requested
- **[PR Review](./PR_REVIEW_WORKFLOW.md)** - Auto-review pull requests
- **[End-to-End AI Workflow](./docs/END_TO_END_AI_WORKFLOW.md)** - Complete autonomous development

---

## Next Steps

1. **Create the workflow file** in `.github/workflows/marty-issue-response.yml`
2. **Update all `@marty` references** to `@martyy-code` in existing workflows
3. **Verify GitHub App username** is indeed `@martyy-code`
4. **Test by commenting** on an issue with `@martyy-code`
5. **Customize the prompt** based on your team's needs
6. **Monitor response quality** and iterate

---

## Quick Reference Card

Print this for your team:

```
🤖 How to Talk to Marty

Username: @martyy-code (not @marty!)

Example uses:
- "@martyy-code how does this function work?"
- "@martyy-code what's the best approach for X?"
- "@martyy-code implements this issue"

Tips:
✓ Be specific
✓ Provide context
✓ Share error messages
✓ Reference file names

Marty will:
✓ Read full issue history
✓ Understand context
✓ Provide helpful answers
✓ Offer implementation help
```

---

## Support

For issues or questions:
- Check main [Marty documentation](./README.md)
- Review [troubleshooting guide](./docs/faq.md)
- Open an issue on [Marty repository](https://github.com/nesalia-inc/marty-action)
