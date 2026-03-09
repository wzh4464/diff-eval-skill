---
name: diff-eval
description: Evaluate agent-generated code changes against a human-approved PR. Compares the local git diff (modified + untracked files) with the ground truth PR diff, scores using a structured rubric (functional correctness, completeness, behavioral equivalence), and provides deep coverage analysis. Use when evaluating agent code generation quality, benchmarking AI coding tools, or comparing generated patches against human-approved pull requests. Trigger on "/diff-eval", "evaluate diff", "compare diff with PR", "score the changes".
argument-hint: <pr-link> <requirements-link> <repo-path>
---

# Diff Eval: Agent-Generated Code Evaluation

Evaluate how well agent-generated code changes match a human-approved PR, with structured scoring and deep coverage analysis.

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

## Workflow

### Step 1: Gather the Three Inputs

#### 1a. Extract the Generated Patch (local diff)

Run these commands in the repo directory to get the agent's changes:

```bash
# Get modified/staged changes vs HEAD
git diff HEAD

# Get list of untracked files (excluding .claude directory)
git ls-files --others --exclude-standard | grep -v '^\.claude'

# For each untracked file, show its content as a diff-like addition
# (prepend each line with "+")
```

Combine the output of `git diff HEAD` with the untracked file contents into a single "Generated Patch" string. Exclude any files under `.claude/`.

Also collect file-level stats:
```bash
# Files modified/added in the generated patch
git diff HEAD --name-only
git ls-files --others --exclude-standard | grep -v '^\.claude'
```

#### 1b. Fetch the Ground Truth Patch (PR diff)

Use `gh` CLI to get the approved PR diff:

```bash
# Extract owner/repo and PR number from the URL
gh pr diff <PR_NUMBER> --repo <OWNER/REPO>
```

Also collect the PR file list:
```bash
gh pr diff <PR_NUMBER> --repo <OWNER/REPO> --name-only
```

And fetch PR metadata for context:
```bash
gh pr view <PR_NUMBER> --repo <OWNER/REPO> --json title,body,files,additions,deletions,changedFiles
```

#### 1c. Fetch the Issue Statement (requirements)

Use WebFetch to retrieve the requirements document. If it's a GitHub issue, prefer:
```bash
gh issue view <ISSUE_NUMBER> --repo <OWNER/REPO> --json title,body
```

### Step 2: Coverage Analysis

Before scoring, perform a structural comparison of what was changed.

Compute and present:

| Metric | Generated Patch | Ground Truth PR |
|--------|----------------|-----------------|
| Files changed | count | count |
| Lines added | count | count |
| Lines deleted | count | count |
| Test files changed | count | count |

Then produce a file-level diff table:

| File | In Generated? | In Ground Truth? | Status |
|------|:---:|:---:|--------|
| `src/foo.py` | Y | Y | Covered |
| `src/bar.py` | N | Y | **Missing** |
| `tests/test_foo.py` | N | Y | **Missing** |
| `src/extra.py` | Y | N | Extra |

Highlight:
- **Missing files**: Files in the PR but not in the generated patch — these represent gaps in coverage
- **Extra files**: Files in the generated patch but not in the PR — may be unnecessary or over-engineered changes
- **Partial overlap files**: Files touched by both but with significantly different change scope

### Step 3: Deep Analysis

Go beyond surface-level file counts. For each file that appears in both patches, compare:

1. **Change scope within shared files**: Does the generated patch modify the same functions/classes/blocks? Or does it touch different parts of the same file?

2. **Approach differences**: Does the generated patch use a fundamentally different approach to solve the problem? (e.g., different algorithm, different API, different pattern)

3. **Missing logic**: Key logic present in the ground truth but absent from the generated patch. Call out specific functions, conditions, or error handling that were missed.

4. **Unnecessary changes**: Changes in the generated patch that have no counterpart in the ground truth. Assess whether they add value or are noise.

5. **Test coverage gap**: If the ground truth includes test changes, analyze whether the generated patch's lack of tests (or different tests) is a significant gap.

6. **Dependency/config changes**: Note any differences in package dependencies, configuration files, or build scripts.

### Step 4: Scoring

Read the scoring rubric from [references/scoring_rubric.md](references/scoring_rubric.md).

Score each criterion independently on a 0-5 scale. Do NOT compute a weighted total or overall score — each dimension is reported separately so downstream consumers can apply their own weighting.

- **A. Functional Correctness** (0-5)
- **B. Completeness & Coverage** (0-5)
- **C. Behavioral Equivalence to Ground Truth** (0-5)

For each score, provide a 2-4 sentence justification referencing specific code changes or gaps.

Determine verdict per rubric rules (PASS / PARTIAL / FAIL), based on the individual scores.

### Step 5: Save and Output Report

Save the report as a markdown file at `<repo-path>/eval_report.md`. Then present the same content to the user.

Report structure:

```
## Evaluation Report

### Summary
[1-3 sentence high-level summary]

### Verdict: [PASS / PARTIAL / FAIL]

### Scores

#### A. Functional Correctness: [X]/5
[2-4 sentence justification. Reference specific functions, logic, or code paths.]

#### B. Completeness & Coverage: [X]/5
[2-4 sentence justification. Reference specific missing/extra files, tests, or config.]

#### C. Behavioral Equivalence to Ground Truth: [X]/5
[2-4 sentence justification. Reference specific semantic differences or similarities.]

### Coverage Analysis

#### Stats Comparison
| Metric | Generated Patch | Ground Truth PR |
|--------|----------------|-----------------|
| Files changed | X | Y |
| Lines added | X | Y |
| Lines deleted | X | Y |
| Test files changed | X | Y |

#### File Coverage Rate
Generated patch covers X out of Y ground truth files (XX%).

#### File-Level Comparison
| File | Generated? | Ground Truth? | Status |
|------|:---:|:---:|--------|
| ... | ... | ... | ... |

### Deep Analysis

#### Approach Comparison
[How does the generated patch's approach differ from the ground truth?
Compare at the level of algorithms, patterns, and architectural decisions.]

#### Shared Files: Scope Comparison
[For files modified by both patches, compare which functions/classes/blocks
are touched. Highlight differences in granularity.]

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
[Any differences in package.json, requirements.txt, Makefile, CI config, etc.]

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
- If the PR or requirements link is inaccessible, inform the user and ask for alternatives
- For very large diffs, focus analysis on the most significant changes rather than exhaustively listing every line
- When the generated patch takes a completely different approach, still evaluate whether it achieves the same functional outcome
- Score honestly — a fundamentally different but equally correct approach can still score well on A and B, even if C is lower
