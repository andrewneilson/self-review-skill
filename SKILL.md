---
name: self-review
description: Local code review comparing branches/diffs without GitHub integration. Use when the user wants to review their code, self-review changes, check for bugs before committing, or mentions code review, diff review, or branch comparison.
allowed-tools: AskUserQuestion Bash(git diff:*) Bash(git log:*) Bash(git branch:*) Bash(git blame:*) Bash(git show:*) Bash(git for-each-ref:*) Bash(git rev-parse:*) Bash(git status:*) Bash(git ls-files:*) Bash(git cat-file:*) Bash(grep:*) Bash(head:*) Bash(wc:*) Read Glob Grep Task
---

# Self-Review

Perform local code review of git changes using parallel agents.

## Quick Start

When invoked, follow these steps in order.

## Step 1: Gather Context

Get current branch and recent branches for comparison:

```bash
git branch --show-current
git for-each-ref --sort=-committerdate --format='%(refname:short)' refs/heads/ | grep -v "$(git branch --show-current)" | head -3
```

## Step 2: Ask User Questions

### Question 1: Review Scope

```
AskUserQuestion:
- question: "What would you like to review?"
- header: "Scope"
- multiSelect: false
- options:
  - label: "Both"
    description: "Review branch commits + working changes (Recommended)"
  - label: "Branch commits"
    description: "Review commits on current branch vs base branch"
  - label: "Working changes"
    description: "Review staged and unstaged changes only"
```

### Question 2: Base Branch

Present the 3 most recent branches as options (user can select "Other" to type any branch).

## Step 3: Gather the Diff

Based on selections:

- **Branch commits**: `git diff <base-branch>..HEAD`
- **Working changes**: `git diff` + `git diff --cached`
- **Both**: `git diff <base-branch>`

Also get changed files: `git diff --name-only <base-branch>`

## Step 4: Find CLAUDE.md Files

Use a Haiku agent to find relevant CLAUDE.md files in the repo root and modified directories. Read their contents.

## Step 5: Summarize Changes

Use a Haiku agent to create a 2-3 sentence summary of the changes.

## Step 6: Multi-Agent Review

Launch 4 parallel Sonnet agents (single message with 4 Task calls):

### Agent 1: CLAUDE.md Compliance
Audit changes against CLAUDE.md guidelines. Only flag issues where guidelines EXPLICITLY mention the rule.

### Agent 2: Bug Detection
Shallow scan for obvious bugs. Focus on LARGE bugs only: null pointers, off-by-one, race conditions, resource leaks, incorrect logic.

### Agent 3: History Analysis
Use `git blame` and `git log` to identify issues informed by history (reverted fixes, broken patterns).

### Agent 4: Code Comments Compliance
Check that changes comply with inline code comments (TODOs, warnings, documentation).

Each agent returns issues in format:
```
- file: <path>
- lines: <line range>
- reason: "[Category]: [description]"
- code_snippet: <code>
```

## Step 7: Confidence Scoring

For each issue found, launch parallel Haiku agents to score confidence 0-100:
- 0: False positive
- 25: Might be real
- 50: Real but minor
- 75: Real and important
- 100: Definitely real

## Step 8: Filter and Output

Filter out issues with confidence < 80.

Also filter:
- Pre-existing issues (not from this diff)
- Pedantic nitpicks
- Linter/typechecker issues
- Lines user didn't modify

### Output Format

```markdown
## Self-Review Results

**Comparing:** [current-branch] vs [base-branch]
**Scope:** [scope]
**Files changed:** [count]

---

Found [N] issues:

### 1. [Issue title] (confidence: [score])
**File:** [path]:[lines]
**Reason:** [reason]

\`\`\`[language]
[code snippet]
\`\`\`
```

If no issues >= 80 confidence:

```markdown
## Self-Review Results

**Comparing:** [current-branch] vs [base-branch]
**Scope:** [scope]
**Files changed:** [count]

---

No significant issues found. Checked for:
- CLAUDE.md compliance
- Obvious bugs
- Historical context issues
- Code comment compliance
```
