---
name: self-review
description: Local code review comparing branches/diffs without GitHub integration. Use when the user wants to review their code, self-review changes, check for bugs before committing, or mentions code review, diff review, or branch comparison.
allowed-tools: AskUserQuestion shell(git diff:*) shell(git log:*) shell(git branch:*) shell(git blame:*) shell(git show:*) shell(git for-each-ref:*) shell(git rev-parse:*) shell(git status:*) shell(git ls-files:*) shell(git cat-file:*) shell(git merge-base:*) shell(git diff:*) shell(git log:*) shell(git branch:*) shell(git blame:*) shell(git show:*) shell(git for-each-ref:*) shell(git rev-parse:*) shell(git status:*) shell(git ls-files:*) shell(git cat-file:*) shell(git merge-base:*) Read Glob Grep Task
---

# Self-Review

Perform local code review of git changes using parallel agents with high-reasoning verification.

## Quick Start

When invoked, follow these steps in order.

## Step 1: Gather Context

Get current branch and recent branches for comparison:

```bash
git branch --show-current
```

```bash
git for-each-ref --sort=-committerdate --format='%(refname:short)' refs/heads/
```

From the second command's output, filter out the current branch and take the first 3 remaining branches. Do this filtering in your logic, not with shell pipes.

## Step 1.5: High-Reasoning Model Selection

You know your own model ID. If you are a Claude model (`claude-*`), skip this step and use `opus` as `<high-reasoning-model>`.

If you are NOT a Claude model, ask the user:

```
AskUserQuestion:
- question: "Which model for the high-reasoning verification pass?"
- header: "Verifier"
- multiSelect: false
- options:
  - label: "opus"
    description: "Claude Opus 4.5 (Recommended)"
  - label: "gpt-5.2-codex-xhigh"
    description: "GPT 5.2 Codex (xhigh reasoning)"
  - label: "gpt-5.2-codex-high"
    description: "GPT 5.2 Codex (high reasoning)"
  - label: "gemini-3-pro"
    description: "Gemini 3 Pro"
```

Store the response as `<high-reasoning-model>`.

Low and medium reasoning tasks always use Claude Haiku and Sonnet.

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

```
AskUserQuestion:
- question: "Which base branch should we compare against?"
- header: "Base Branch"
- multiSelect: false
- options:
  - label: "<recent-branch-1>"
    description: "Recent branch"
  - label: "<recent-branch-2>"
    description: "Recent branch"
  - label: "<recent-branch-3>"
    description: "Recent branch"
  - label: "Other"
    description: "Type a different base branch"
- freeTextOn: "Other"
- freeTextPlaceholder: "e.g. main"
```

## Step 3: Gather the Diff

Based on selections, run simple individual commands. Handle fallback logic in your decision-making, not shell scripts.

### For Branch commits (compare against base branch):

1. Get the merge-base:
```bash
git merge-base <base-branch> HEAD
```

2. If merge-base succeeds (returns a commit hash), use it for the diff:
```bash
git diff <merge-base-result>..HEAD
```

3. If merge-base fails or returns empty, fall back to direct diff:
```bash
git diff <base-branch>..HEAD
```

4. Get the list of changed files (using same comparison):
```bash
git diff --name-only <merge-base-result>..HEAD
```

### For Working changes (staged + unstaged + untracked):

Run these commands separately:

```bash
git diff --cached
```

```bash
git diff
```

```bash
git ls-files --others --exclude-standard
```

For untracked files from the last command, read their contents using the Read tool (they're new files, so the entire file is the "diff").

### For Both:

Combine branch diff with working changes. Run the commands separately as listed above.

## Step 4: Find CLAUDE.md and AGENTS.md Files

Use an Explore agent with `model: "<low-reasoning-model>"` to:
1. Use Glob tool with patterns `**/CLAUDE.md` and `**/AGENTS.md` to find guidance files
2. Read contents of found files, prioritizing repo root and directories containing changed files

## Step 5: Summarize Changes

Use an Explore agent with `model: "<low-reasoning-model>"` to create a 2-3 sentence summary of the changes.

## Step 6: Multi-Agent Review

Launch 5 parallel Explore agents with `model: "<medium-reasoning-model>"` and thoroughness "very thorough" (single message with 5 Task calls). Use `subagent_type: "Explore"` to ensure agents cannot access Edit/Write tools.

### Read-Only Mode (include in ALL agent prompts)

```
CRITICAL: This is a READ-ONLY code review task.

DO NOT:
- Edit, modify, or fix any code
- Use the Edit, Write, or MultiEdit tools
- Write files to /tmp/ or any other location
- Suggest applying fixes directly
- Attempt to "help" by making changes

DO:
- Report findings in the specified output format
- Return all output as markdown text in your response (not as files)
- Describe issues and how to confirm them
- Suggest fixes in the "fix_suggestion" field (text only)

Your job is to REPORT, not REPAIR. Return structured findings as markdown text only.
```

### Bug Qualification Criteria (include in Agents 1-4 prompts)

```
Only flag issues where:
1. It meaningfully impacts accuracy, performance, security, or maintainability
2. The issue is discrete and actionable (not general or compound)
3. It matches the rigor level present in the codebase
4. It was introduced in this diff (not pre-existing)
5. The author would likely fix if aware
6. It doesn't rely on unstated assumptions
7. You can identify provably affected code (no speculation)
8. It's not an intentional change by the author
9. Parameter correctness across the call stack:
   a) Arguments represent the correct entities, not just matching types
   b) If a parameter is added, removed, or ignored, check that callers reflect the same intent
```

### Design Finding Criteria (include in Agent 5 prompt)

```
Only flag design issues where:
1. It meaningfully impacts maintainability, readability, or evolvability
2. The issue is discrete and actionable (one clear recommendation per finding)
3. The concern was introduced or materially worsened by this diff
4. The author would likely agree it's worth addressing (not a subjective preference)
5. A concrete alternative exists (even if details are left to the author)
6. The scale of the issue warrants flagging -- minor imperfections are not findings
7. For duplication: both sides are identifiable, and the repeated structure is
   substantial enough that extraction would be an improvement
8. For naming: the current name actively misleads, not merely "could be better"
9. For excessive responsibility: the concerns are genuinely unrelated, not just
   "a lot of code doing one complex thing"
```

### Comment Quality Guidelines (include in ALL agent prompts)

```
When describing an issue:
- Clearly explain WHY it's a problem
- Communicate severity accurately (don't exaggerate)
- Keep explanation to 1 paragraph max
- "Issue" paragraph: NO fenced code blocks, max 3 inline `code` snippets
- Context snippet (separate section): 1-10 lines fenced (as needed)
- Explicitly state conditions where bug arises
- Matter-of-fact tone (no flattery, not accusatory)
- Include "How to confirm" with specific steps
```

### Diff-Overlap Requirement (include in ALL agent prompts)

```
Line ranges must:
- Overlap with the diff (not pre-existing code)
- Be 1-10 lines (pick the tightest subrange that captures the issue)
- For untracked files: overlap is the entire file (via --no-index diff)

For multi-file issues (e.g., missing tests, contract changes):
- Create ONE finding with primary file/line as location
- Primary file MUST still overlap the diff
- List related files in the explanation body
- Use category tag "cross-cutting"

For duplication findings:
- The primary file/line range must overlap the diff
- The second instance may be pre-existing (not in the diff)
  as long as the diff introduces or extends the duplicated code
- List both file locations in the explanation body
```

### Required Category Tag

Each agent must tag findings with a category from:
`null-pointer | buffer-overflow | integer-overflow | race-condition | resource-leak | logic-error | security | missing-test | type-error | api-misuse | wrong-value | cross-cutting | naming | duplication | excessive-responsibility | other`

Note: `wrong-value` is for bugs where a parameter is provided but represents the wrong entity/data (correct type, wrong semantic meaning).

Design categories: `naming` is for identifiers that mislead about behavior or scope.
`duplication` is for substantial repeated logic that should be shared.
`excessive-responsibility` is for units handling multiple unrelated concerns.

### Agent 1: CLAUDE.md/AGENTS.md Compliance

Audit changes against CLAUDE.md and AGENTS.md guidelines. Only flag issues where guidelines EXPLICITLY mention the rule.

### Agent 2: Bug Detection

Scan for bugs with focus on:
- Null pointers, off-by-one, race conditions, resource leaks, incorrect logic
- **Wrong value passed**: A parameter is provided but represents the wrong entity/data
  (e.g., using X's ID when Y's ID was needed, passing stale data, wrong enum value)

#### Systematic Wrong-Value Detection Procedure

For each function/method call in the diff where arguments are passed:

1. **DETERMINE** what entity each argument SHOULD represent:
   - Read the function signature and parameter names
   - Check function documentation/comments if available
   - Infer from function name (e.g., `checkPermission($userId)` → userId should be the user being checked)

2. **VERIFY** the actual argument matches the expected entity:
   - What variable is being passed?
   - What does that variable represent in the CALLING context?
   - Does the calling context's meaning match the function's expected meaning?

3. **PAY SPECIAL ATTENTION** to:
   - Same-type parameters representing different entities (e.g., sourceId vs targetId, ownerId vs viewerId)
   - Variables passed from different scopes
   - Functions with multiple parameters of the same type
   - Gating/permission functions (the "who" being checked matters critically)

### Agent 3: History Analysis

Use `git blame` and `git log` to identify issues informed by history (reverted fixes, broken patterns).

### Agent 4: Code Comments Compliance

Check that changes comply with inline code comments (TODOs, warnings, documentation).

### Agent 5: Design & Organization

Review changes for design and organizational concerns. Include the Design Finding Criteria (not Bug Qualification Criteria) in this agent's prompt.

Focus areas:

- **Naming clarity**: Identifiers whose names actively mislead about behavior or scope. Not "could be slightly better" names -- names where the implied abstraction differs materially from actual behavior.
- **Significant duplication**: Substantial blocks (~20+ lines) of repeated logic across files indicating a missing shared abstraction. Must identify both sides concretely.
- **Excessive responsibility**: A single file/class/function handling multiple unrelated concerns. Threshold scales with complexity (50-line file with two related concerns is fine).

Constraints:
- Only flag issues introduced or significantly worsened by this diff
- Diff-overlap requirement applies
- Don't flag stylistic preferences (brace style, import ordering)
- Don't flag trivially short files/functions
- Duplication in test files is lower priority than production code

### Agent Output Format

Each agent returns issues in format:
```
- file: <path>
- lines: <start>-<end> (must be 1-10 lines, must overlap diff)
- category: <category tag>
- title: <short title>
- issue: <1 paragraph, no fenced code, max 3 inline code snippets>
- how_to_confirm: <specific steps>
- code_snippet: <1-10 lines from the file at the reported line range>
```

## Step 6.5: Dedupe Findings

After multi-agent review, before scoring.

### Dedupe Criteria (ALL must match):

- Same file
- Overlapping line ranges
- Same category tag

DO NOT merge based on title similarity alone.

### Merge Rules:

- Keep tightest subrange for file location (MUST be ≤10 lines even if union would be larger)
- Mention other relevant lines in explanation body
- Keep most detailed description
- Preserve all unique context

## Step 7: Confidence Scoring (Ranking Only)

Use Explore agents with `model: "<low-reasoning-model>"` to score each candidate. This is ranking only - NO filtering at this step.

For each candidate:
- Assign confidence 0-100 (likelihood it's real)
- Do NOT filter any candidates
- Pass full list to verifier

Confidence scale:
- 0-25: Likely false positive
- 26-50: Uncertain
- 51-75: Probably real
- 76-100: Almost certainly real

## Step 8: High-Reasoning Verification

Launch an Explore agent with `model: "<high-reasoning-model>"` and thoroughness "very thorough" that receives (use `subagent_type: "Explore"` to prevent edit attempts):
- Full diff with hunks (from Step 3) - not just filenames, the actual diff content
- CLAUDE.md/AGENTS.md files (from Step 4)
- Ranked candidate list with confidence scores (from Step 7)

### Critical: Independent Diff Scan

The verifier must independently scan the entire diff, not just validate candidates. The candidate list is a starting point, not the universe of issues.

### Large Diff Handling

If diff exceeds context limits (~100k tokens), chunk by file:
- Prioritize files that had candidate findings
- Include "highest-risk" files (security-related, core logic) even without candidates
- **Required:** If any files were skipped, output must include a list of skipped files

### Verifier Prompt Instructions

```
You are performing a final verification pass on code review findings.

CRITICAL: This is a READ-ONLY review task. Do NOT edit, modify, or fix any code.
Do NOT use Edit, Write, or MultiEdit tools. Report findings only - do not repair.

Bug Qualification Criteria (for correctness categories):
1. It meaningfully impacts accuracy, performance, security, or maintainability
2. The issue is discrete and actionable (not general or compound)
3. It matches the rigor level present in the codebase
4. It was introduced in this diff (not pre-existing)
5. The author would likely fix if aware
6. It doesn't rely on unstated assumptions
7. You can identify provably affected code (no speculation)
8. It's not an intentional change by the author
9. Parameter correctness across the call stack:
   a) Arguments represent the correct entities, not just matching types
   b) If a parameter is added, removed, or ignored, check that callers reflect the same intent

Design Finding Criteria (for design categories: naming, duplication, excessive-responsibility):
1. It meaningfully impacts maintainability, readability, or evolvability
2. The issue is discrete and actionable (one clear recommendation per finding)
3. The concern was introduced or materially worsened by this diff
4. The author would likely agree it's worth addressing (not a subjective preference)
5. A concrete alternative exists (even if details are left to the author)
6. The scale of the issue warrants flagging -- minor imperfections are not findings
7. For duplication: both sides are identifiable, and the repeated structure is
   substantial enough that extraction would be an improvement
8. For naming: the current name actively misleads, not merely "could be better"
9. For excessive responsibility: the concerns are genuinely unrelated, not just
   "a lot of code doing one complex thing"

Apply Bug Qualification for correctness categories. Apply Design Finding for
design categories (naming, duplication, excessive-responsibility).

Comment Quality Guidelines:
- Clearly explain WHY it's a problem
- Communicate severity accurately (don't exaggerate)
- Keep explanation to 1 paragraph max
- "Issue" paragraph: NO fenced code blocks, max 3 inline `code` snippets
- Context snippet (separate section): 1-10 lines fenced (as needed)
- Explicitly state conditions where bug arises
- Matter-of-fact tone (no flattery, not accusatory)
- Include "How to confirm" with specific steps

TASK 1: Review all candidates, marking each verified: true/false/unsure
- true: Confirms this is a real finding per the applicable criteria set
- false: Rejects as false positive or pre-existing; set confidence: 0
- unsure: Can't conclusively verify; may adjust confidence up/down

TASK 2: Set confidence for ALL findings (0 for rejected, assessed value for others)

TASK 3: Independently scan the full diff for issues not in the candidate list
- Scan for both correctness bugs AND design concerns (naming, duplication, excessive responsibility)
- New findings may REPLACE lower-quality candidates if they cover the same issue better
- All new findings must also have verified status
- Apply systematic wrong-value detection (see below)

TASK 4: Assign priority based on IMPACT (not confidence)
Correctness priority:
- P0: Drop everything. Blocking. Universal, assumption-free.
- P1: Urgent. Address in next cycle.
- P2: Normal. Fix eventually.
- P3: Low. Nice to have.
Design priority:
- P0: Not applicable for design findings (no runtime failures)
- P1: Significant ongoing maintenance burden if not addressed before merge
- P2: Worth addressing but mergeable with follow-up
- P3: Minor organizational improvement

TASK 5: Assign required category tag for each finding

Output Format for each finding:
- file: <path>
- lines: <start>-<end>
- category: <category tag>
- title: <short title>
- priority: P0/P1/P2/P3
- confidence: 0-100
- verified: true/false/unsure
- issue: <1 paragraph explanation>
- how_to_confirm: <specific steps>
- fix_suggestion: <if applicable>
- code_snippet: <1-10 lines from file at reported line range>

If any files were skipped due to size:
- skipped_files: [list of skipped file paths]

Systematic Wrong-Value Detection:
For each function call in the diff where arguments are passed:
1. DETERMINE what each argument SHOULD represent (from signature, docs, function name)
2. VERIFY the actual argument matches: what does the variable mean in the CALLING context?
3. FLAG when calling context meaning differs from function's expected meaning
4. Priority attention: gating/permission functions, same-type multi-parameter functions
```

## Step 9: Filter and Output

### Filter Rules

After verification review:
- **verified: true** (any confidence): Show in Verified Findings, contribute to verdict
- **verified: unsure** + confidence ≥50: Show in Tentative Findings (doesn't affect verdict)
- **verified: unsure** + confidence <50: Filter out
- **verified: false**: Filter out (rejected by verifier)

### Output Ordering

- Verified findings: Sort by decreasing severity (P0 first, then P1, P2, P3), then by confidence (high→low)
- Tentative findings: Sort by confidence (high→low)

### Output Format

```markdown
## Self-Review Results

**Comparing:** [current-branch] vs [base-branch]
**Scope:** [scope]
**Files changed:** [count]

---

## Verified Findings

### 1. [P0] Issue title (confidence: 85%)
**File:** path/to/file.ext:10-15
**Category:** logic-error
**Context:**
\`\`\`language
[1-10 line snippet from the file at the reported line range]
\`\`\`
**Issue:** [1 paragraph, code refs ≤3 lines, no fenced code blocks]
**How to confirm:** [specific steps]
**Fix suggestion:** [if applicable]

### 2. [P1] Another issue (confidence: 75%)
...

---

**Correctness:** Patch is [correct/incorrect]
**Design:** [clean/has concerns]
**Explanation:** [1-2 sentence justification covering both]

---

## Tentative Findings (verifier unsure, confidence ≥50%)

### 1. [P2] Uncertain issue (confidence: 55%)
**File:** path/to/file.ext:20-25
**Category:** null-pointer
**Context:**
\`\`\`language
[snippet]
\`\`\`
**Issue:** [explanation]
**How to confirm:** [steps]
**Fix suggestion:** [if applicable]
```

### Verdict Rules

- **Correctness**: "incorrect" if ANY verified correctness finding exists (any priority level)
  - Correctness categories: all categories except `naming`, `duplication`, `excessive-responsibility`
- **Design**: "has concerns" if ANY verified design finding exists
  - Design categories: `naming`, `duplication`, `excessive-responsibility`
- If files were skipped due to size:
  - Correctness: "correct (with skipped files)" if no verified correctness bugs
  - Explanation MUST list which files weren't reviewed
  - Skipped-files caveat applies to correctness only
- Tentative Findings section still appears even if Verified Findings is empty

### No-Findings Output

```markdown
## Self-Review Results

**Comparing:** [current-branch] vs [base-branch]
**Scope:** [scope]
**Files changed:** [count]

---

## Verified Findings

No verified findings.

---

**Correctness:** Patch is correct
**Design:** clean
**Explanation:** No issues met verification criteria.

---

## Tentative Findings (verifier unsure, confidence ≥50%)

No tentative findings.
```

### Skipped Files Output (when applicable)

```markdown
**Note:** The following files were skipped due to diff size limits:
- [file1]
- [file2]
Consider running `/self-review` again with these files specifically.
```
