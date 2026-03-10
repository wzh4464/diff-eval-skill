# diff-eval

A Claude Code skill for evaluating agent-generated code changes against human-approved pull requests.

## What it does

`/diff-eval` compares a local git diff (agent-generated changes) with a ground truth PR, producing a structured evaluation report:

- **Scoring** on 3 dimensions (0-5 each): Functional Correctness, Completeness & Coverage, Behavioral Equivalence
- **File-level coverage analysis**: which files match, which are missing, which are extra
- **Deep analysis**: approach comparison, missing logic, test coverage gaps, unnecessary changes
- **Verdict**: PASS / PARTIAL / FAIL
- **Auto-saves** report to `eval_report.md` in the project directory

## Install

```bash
git clone https://github.com/wzh4464/diff-eval-skill.git /tmp/diff-eval-skill
cp -r /tmp/diff-eval-skill/diff-eval ~/.claude/skills/diff-eval
rm -rf /tmp/diff-eval-skill
```

Or manually copy the `diff-eval/` directory to `~/.claude/skills/`.

## Usage

```
/diff-eval <pr-link> <requirements-link> <repo-path>
```

**Example:**
```
/diff-eval https://github.com/kubernetes/kubernetes/pull/132807 https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/5365-ImageVolume-with-image-digest/README.md /home/user/codes/my-project
```

### Inputs

| Parameter | Required | Description |
|-----------|----------|-------------|
| `pr-link` | Yes | GitHub PR URL (ground truth) |
| `requirements-link` | Yes | URL to the issue/spec/requirements document |
| `repo-path` | Yes | Local path to the repo with agent-generated changes |

### Output

The skill generates `eval_report.md` in the repo directory and displays the report, including:

- Per-dimension scores with justification
- Stats comparison table (files, lines added/deleted, test files)
- File-level coverage table
- Deep analysis sections (approach, missing logic, test gaps, etc.)
- Strengths, weaknesses, and recommendations

## Requirements

- `gh` CLI (authenticated with GitHub)
- Git repository with uncommitted changes (the agent-generated diff)

## License

MIT
