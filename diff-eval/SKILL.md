---
name: diff-eval
description: Evaluate agent-generated code changes against a human-approved PR. Compares the local git diff (modified + untracked files, relative to the PR's merge base commit) with the ground truth PR diff, filters out auto-generated files, scores using a structured rubric (functional correctness, completeness, behavioral equivalence), and provides both semantic and data-based coverage analysis. Use when evaluating agent code generation quality, benchmarking AI coding tools, or comparing generated patches against human-approved pull requests. Trigger on "/diff-eval", "evaluate diff", "compare diff with PR", "score the changes".
argument-hint: <pr-link> <requirements-link> <repo-path>
---

# Diff Eval: Agent-Generated Code Evaluation

Evaluate how well agent-generated code changes match a human-approved PR, with structured scoring, auto-generated file filtering, and dual semantic + data-based coverage analysis.

## Inputs

The user provides:
1. **PR link** (required) - GitHub PR URL, e.g. `https://github.com/org/repo/pull/123`
2. **Requirements link** (required) - URL to the issue/spec/requirements document
3. **Repo path** (required) - Local path to the repo containing agent-generated changes

Parse these from ``:
- URL containing `/pull/` -> PR link
- Other URL -> requirements link
- Non-URL path -> repo path
- If any required input is missing, ask the user for it before proceeding

## CRITICAL: Working Directory Rule

**All git commands MUST use `git -C <REPO_PATH>` to explicitly specify the repository directory.**

The shell working directory resets between tool calls. If you run `git diff` without `-C`, it may execute outside the repo and fail with `fatal: not a git repository`. Never rely on `cd` persisting across separate Bash tool calls.

```bash
# WRONG — will fail if cwd is not the repo
git diff <BASE_COMMIT>

# CORRECT — always use -C to specify the repo
git -C <REPO_PATH> diff <BASE_COMMIT>

# ALSO CORRECT — cd && git in the same command
cd <REPO_PATH> && git diff <BASE_COMMIT>
```

Apply this rule to EVERY `git` command in the entire workflow below.

## Workflow

### Step 1: Gather the Three Inputs

#### 1a. Determine the Base Commit (merge base from PR)

First, fetch PR metadata to find the base commit — the commit the PR was merged onto (or branched from):

```bash
# Get the base ref name and merge commit details
gh pr view <PR_NUMBER> --repo <OWNER/REPO> --json baseRefName,mergeCommit,headRefOid,title,body,files,additions,deletions,changedFiles
```

Then determine the merge base commit hash in the local repo. **All git commands must use `-C <REPO_PATH>`**:

```bash
# Ensure remote refs are up to date
git -C <REPO_PATH> fetch origin

# Option 1: If the PR is merged, use the merge commit's first parent
# (the commit on the base branch just before the merge)
git -C <REPO_PATH> rev-parse <mergeCommit>^1

# Option 2: If the PR's base branch is available locally, find the merge base
# between the PR's base branch and the PR's head commit
git -C <REPO_PATH> merge-base origin/<baseRefName> <headRefOid>

# Option 3: Fall back to finding the fork point
git -C <REPO_PATH> merge-base origin/<baseRefName> HEAD
```

Store this as `BASE_COMMIT`. This is the reference point for the generated diff — it represents the state of the code before the PR's changes were applied.

**Important**: If `BASE_COMMIT` cannot be determined (e.g., remote branches not fetched), run `git -C <REPO_PATH> fetch origin` and retry. If still unavailable, ask the user for the base commit hash.

#### 1b. Extract the Generated Patch (local diff vs base commit)

Get the agent's changes **relative to the base commit**:

```bash
# Get modified/staged changes vs the base commit (NOT HEAD)
git -C <REPO_PATH> diff <BASE_COMMIT>

# Get list of untracked files (excluding .claude directory)
git -C <REPO_PATH> ls-files --others --exclude-standard | grep -v '^\.claude'

# For each untracked file, show its content as a diff-like addition
# (prepend each line with "+")
```

Combine the output of `git -C <REPO_PATH> diff <BASE_COMMIT>` with the untracked file contents into a single "Generated Patch" string. Exclude any files under `.claude/`.

Also collect file-level stats:
```bash
# Files modified/added in the generated patch
git -C <REPO_PATH> diff <BASE_COMMIT> --name-only
git -C <REPO_PATH> ls-files --others --exclude-standard | grep -v '^\.claude'
```

#### 1c. Fetch the Ground Truth Patch (PR diff)

Use `gh` CLI to get the approved PR diff:

```bash
# Extract owner/repo and PR number from the URL
gh pr diff <PR_NUMBER> --repo <OWNER/REPO>
```

Also collect the PR file list:
```bash
gh pr diff <PR_NUMBER> --repo <OWNER/REPO> --name-only
```

#### 1d. Fetch the Issue Statement (requirements)

Use WebFetch to retrieve the requirements document. If it's a GitHub issue, prefer:
```bash
gh issue view <ISSUE_NUMBER> --repo <OWNER/REPO> --json title,body
```

### Step 2: Classify Auto-Generated vs Non-Auto-Generated Files

Before any comparison, classify every file in both the generated patch and ground truth PR into two categories.

**Classification method**: Two simple checks, in order:

#### 2a. Path-Based Quick Filter (obvious cases only)

These are unambiguously auto-generated by file name alone — no content check needed:
- **Lock files**: `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Pipfile.lock`, `poetry.lock`, `Gemfile.lock`, `composer.lock`, `Cargo.lock`, `go.sum`
- **Build output directories**: `dist/`, `build/`, `.next/`, `__pycache__/`, `node_modules/`
- **Compiled artifacts**: `*.pyc`, `*.min.js`, `*.min.css`
- **IDE/OS artifacts**: `.idea/`, `.DS_Store`

Only apply this for clear-cut cases. When in doubt, proceed to step 2b.

#### 2b. Content-Based Check (read the actual file)

For every file **not** already classified by path, **read the first 5 lines of the file from the GT PR** and check whether they contain auto-generated markers (e.g. `@generated`, `DO NOT EDIT`, `auto-generated`, `Generated by`, etc.).

Use the PR's ref to read the file content directly from GitHub:
```bash
# Read first 5 lines of a file at the PR's head commit
gh api repos/<OWNER>/<REPO>/contents/<FILE_PATH>?ref=<PR_HEAD_SHA> \
  --jq '.content' | base64 -d | head -5
```

If the first few lines contain any auto-generated marker, classify as auto-generated. Otherwise, classify as non-auto-generated.

**This is the authoritative check** — it catches generated files with normal-looking paths (e.g. `src/schema.ts` that starts with `// Generated by ...`).

#### 2c. Classification Output

**Non-auto-generated files** (core comparison targets):
- Source code, test files, configuration files, documentation, CI/CD configs, Dockerfiles, Makefiles
- Manually written migration files
- Everything not classified as auto-generated above

Present the classification:

| File | Source | Classification | Reason |
|------|--------|---------------|--------|
| `src/foo.py` | Both | Non-auto | Source code |
| `package-lock.json` | Generated only | **Auto-generated** | Lock file (path) |
| `dist/bundle.js` | PR only | **Auto-generated** | Build output (path) |
| `src/schema.ts` | Both | **Auto-generated** | Header: `// Generated by ...` (content) |

**Important**: If uncertain about a file, default to non-auto-generated (conservative approach). If the project has unusual conventions, note them.

### Step 3: Coverage Analysis (Data-Based)

Perform a quantitative comparison using **only non-auto-generated files**.

#### 3a. File Set Coverage Rate

Compute the core metric:

```
Non-auto PR files:  {set of non-auto-generated files in ground truth PR}
Non-auto Diff files: {set of non-auto-generated files in generated patch}
Intersection:        Non-auto PR files ∩ Non-auto Diff files
Coverage Rate:       |Intersection| / |Non-auto PR files| × 100%
```

This is the primary data-based score — what fraction of the PR's meaningful files did the agent also touch?

#### 3b. Stats Comparison (non-auto files only)

| Metric | Generated Patch | Ground Truth PR |
|--------|----------------|-----------------|
| Non-auto files changed | count | count |
| Lines added (non-auto) | count | count |
| Lines deleted (non-auto) | count | count |
| Test files changed | count | count |
| Auto-generated files (excluded) | count | count |

#### 3c. File-Level Comparison Table

| File | Auto? | In Generated? | In Ground Truth? | Status |
|------|:---:|:---:|:---:|--------|
| `src/foo.py` | N | Y | Y | Covered |
| `src/bar.py` | N | N | Y | **Missing** |
| `tests/test_foo.py` | N | N | Y | **Missing** |
| `src/extra.py` | N | Y | N | Extra |
| `package-lock.json` | Y | Y | Y | *(auto, excluded)* |

Highlight:
- **Missing files**: Non-auto files in the PR but not in the generated patch — these represent real gaps
- **Extra files**: Non-auto files in the generated patch but not in the PR — may be unnecessary
- **Auto-generated**: Listed for completeness but excluded from coverage rate calculation

### Step 4: Deep Analysis (Semantic-Based)

This is the qualitative counterpart to Step 3. Instead of counting files, evaluate **what requirements/changes were actually accomplished**.

#### 4a. Requirements Checklist

Based on the issue statement and PR changes, decompose the task into discrete requirements or change items. Then check each:

| # | Requirement / Change Item | In PR? | In Generated? | Status |
|---|--------------------------|:---:|:---:|--------|
| 1 | Fix null pointer in `processOrder()` | Y | Y | Done |
| 2 | Add validation for negative quantities | Y | Y | Done |
| 3 | Update error messages to include order ID | Y | N | **Missing** |
| 4 | Add unit tests for edge cases | Y | N | **Missing** |
| 5 | (Extra) Add logging to `processOrder()` | N | Y | Extra |

Summarize: **X out of Y requirements completed** (semantic completion rate).

#### 4b. Detailed Comparison for Shared Files

For each non-auto file that appears in both patches, compare:

1. **Change scope**: Does the generated patch modify the same functions/classes/blocks? Or does it touch different parts of the same file?

2. **Approach differences**: Does the generated patch use a fundamentally different approach? (e.g., different algorithm, different API, different pattern)

3. **Missing logic**: Key logic present in the ground truth but absent from the generated patch. Be precise — name the function, the file, the condition.

4. **Unnecessary changes**: Changes in the generated patch with no counterpart in the ground truth. Assess: harmless, over-engineering, or potentially harmful?

#### 4c. Test Coverage Gap

If the ground truth includes test changes, analyze whether the generated patch's lack of tests (or different tests) is a significant gap. List specific test scenarios covered by the PR but missing from the generated patch.

#### 4d. Dependency & Config Differences

Note any differences in non-auto-generated package dependencies, configuration files, or build scripts.

### Step 5: Scoring

Read the scoring rubric from [references/scoring_rubric.md](references/scoring_rubric.md).

Score each criterion independently on a 0-5 scale. Do NOT compute a weighted total or overall score — each dimension is reported separately so downstream consumers can apply their own weighting.

- **A. Functional Correctness** (0-5)
- **B. Completeness & Coverage** (0-5)
- **C. Behavioral Equivalence to Ground Truth** (0-5)

For each score, provide a 2-4 sentence justification referencing specific code changes or gaps.

Determine verdict per rubric rules (PASS / PARTIAL / FAIL), based on the individual scores.

### Step 6: Save and Output Report

Save the report as a markdown file at `<repo-path>/eval_report.md`. Then present the same content to the user.

Report structure:

```
## Evaluation Report

### Summary
[1-3 sentence high-level summary]

### Verdict: [PASS / PARTIAL / FAIL]

### Base Commit
`<BASE_COMMIT>` — determined from PR merge base on `<baseRefName>`

### Scores

#### A. Functional Correctness: [X]/5
[2-4 sentence justification. Reference specific functions, logic, or code paths.]

#### B. Completeness & Coverage: [X]/5
[2-4 sentence justification. Reference specific missing/extra files, tests, or config.]

#### C. Behavioral Equivalence to Ground Truth: [X]/5
[2-4 sentence justification. Reference specific semantic differences or similarities.]

### Auto-Generated File Classification

| File | Source | Classification | Reason |
|------|--------|---------------|--------|
| ... | ... | ... | ... |

[X auto-generated files excluded from comparison. Y non-auto files used for analysis.]

### Data-Based Coverage (Non-Auto Files Only)

#### File Set Coverage Rate
Non-auto PR files: X | Non-auto Generated files: Y | Intersection: Z
**Coverage Rate: Z/X = XX%**

#### Stats Comparison (Non-Auto Files)
| Metric | Generated Patch | Ground Truth PR |
|--------|----------------|-----------------|
| Non-auto files changed | X | Y |
| Lines added (non-auto) | X | Y |
| Lines deleted (non-auto) | X | Y |
| Test files changed | X | Y |
| Auto-generated files (excluded) | X | Y |

#### File-Level Comparison
| File | Auto? | Generated? | Ground Truth? | Status |
|------|:---:|:---:|:---:|--------|
| ... | ... | ... | ... | ... |

### Semantic Coverage (Requirements-Based)

#### Requirements Checklist
| # | Requirement / Change Item | In PR? | In Generated? | Status |
|---|--------------------------|:---:|:---:|--------|
| ... | ... | ... | ... | ... |

**Semantic Completion: X/Y requirements completed (XX%)**

### Deep Analysis

#### Approach Comparison
[How does the generated patch's approach differ from the ground truth?
Compare at the level of algorithms, patterns, and architectural decisions.]

#### Shared Files: Scope Comparison
[For non-auto files modified by both patches, compare which
functions/classes/blocks are touched. Highlight differences in granularity.]

#### Missing Logic
[Specific functions, conditions, error handling, or edge cases present in
the ground truth but absent from the generated patch. Be precise — name
the function, the file, the condition.]

#### Unnecessary Changes
[Changes in the generated patch with no ground truth counterpart.
Assess: harmless noise, over-engineering, or potentially harmful?]

#### Test Coverage Gap
[If ground truth includes tests: what test scenarios are covered by
ground truth but missing from generated? If ground truth has no tests,
note that.]

#### Dependency & Config Differences
[Any differences in non-auto-generated package configs, CI, Makefile, etc.]

### Strengths
- [Bullet list of what the generated patch does well]

### Weaknesses
- [Bullet list of what the generated patch misses or does poorly]

### Recommendations
- [Concrete, actionable suggestions for improving the generated patch]

### Confidence: [0.0-1.0]
[Why this confidence level — what information was clear vs ambiguous?]
```

## Important Notes

- Always exclude `.claude/` directory from the generated diff
- The base commit for diffing is derived from the PR's merge base, NOT from HEAD — this ensures the generated diff captures the same scope of changes as the PR
- Auto-generated files are classified and excluded from the core coverage metrics, but listed for transparency
- Two complementary coverage views are provided:
  - **Data-based**: File set intersection ratio (objective, quantitative)
  - **Semantic-based**: Requirements completion checklist (qualitative, intent-focused)
- If the PR or requirements link is inaccessible, inform the user and ask for alternatives
- For very large diffs, focus analysis on the most significant non-auto-generated changes
- When the generated patch takes a completely different approach, still evaluate whether it achieves the same functional outcome
- Score honestly — a fundamentally different but equally correct approach can still score well on A and B, even if C is lower
