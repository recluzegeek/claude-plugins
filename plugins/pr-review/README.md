# pr-review

Codebase-aware pull request review for Claude Code. It reads the code around a change, not only the diff:
- **callers** of what changed
- **callees** it now depends on
- **similar code** elsewhere in the repository, to spot inconsistencies

It then checks whether the change does what it claims.

## How it works

1. **Scope.** It works out the diff against the base branch. On a follow-up push, it reviews only the new commits, while still reading the whole PR for context.
2. **Rules.** It loads the repository's own rules from `.pr-review/rules.md`, plus `AGENTS.md`, `CLAUDE.md` and `CONTRIBUTING.md` when present.
3. **Triage.** It skips generated files, states the change's intent, and tags each file by risk: security, data, concurrency, performance, contract.
4. **Specialist agents run in parallel:**
   - `correctness`: compares old and new behaviour, edge inputs, and intent mismatches
   - `impact`: callers, callees and similar code
   - `security`: traces untrusted input to where it's used
   - `performance`: only on hot paths it has confirmed
   - `data-concurrency`: schemas, migrations, transactions, races
   - `rules`: the repository's written rules
5. **Validation.** A separate `validator` agent re-checks every finding against the code and scores it 1–10. Findings scoring 8 or more are kept. P0 findings scoring 6–7 are kept and marked as unconfirmed.
6. **Report.**
   - Inline comments for P0 and P1 findings.
   - One summary comment, updated on each push, with:
     - what the PR does and whether it matches its description
     - a confidence score from 1 to 5
     - files needing attention
     - a table of the files changed
     - a table of findings
     - the last commit reviewed

Text in the PR (title, description, code, comments) is treated as data to review, never as instructions.

## Use it locally

```bash
claude --plugin-dir ./plugins/pr-review -p "/pr-review:review --local origin/main"
```

`--local <base-ref>` reviews `HEAD` against `<base-ref>`, posts nothing, and prints the findings as JSON.

In a session with the plugin installed, a PR is reviewed with:

```
/pr-review:review owner/repo/pull/123
```

## Use it in GitHub Actions

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]

permissions:
  contents: read
  pull-requests: read
  issues: read
  id-token: write

jobs:
  review:
    if: github.event.pull_request.draft == false
    runs-on: ubuntu-latest
    timeout-minutes: 45
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0   # the review diffs against the base branch and reads history
      - uses: anthropics/claude-code-action@v1
        env:
          # Required: subagents otherwise run in the background, and the action
          # ends the run before they report, so nothing gets posted.
          CLAUDE_CODE_DISABLE_BACKGROUND_TASKS: "1"
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}   # or anthropic_api_key
          plugin_marketplaces: "https://github.com/recluzegeek/claude-plugins.git"
          plugins: "pr-review@recluzegeek"
          prompt: "/pr-review:review ${{ github.repository }}/pull/${{ github.event.pull_request.number }} --comment"
          claude_args: >-
            --model sonnet --max-turns 100
            --allowedTools "Read,Grep,Glob,Task,TodoWrite,Bash(git diff:*),Bash(git log:*),Bash(git show:*),Bash(git rev-parse:*),Bash(git merge-base:*),Bash(git fetch:*),Bash(git ls-files:*),Bash(gh pr view:*),Bash(gh pr diff:*),Bash(gh pr comment:*),mcp__github_inline_comment__create_inline_comment"
```

`--model` sets the coordinator's model. The agents set their own: Opus for correctness, security, data-concurrency and the validator, and Sonnet for impact, performance and rules.

## Repository rules

Add `.pr-review/rules.md` to a repository to teach the review what matters there:

```markdown
---
ignore:
  - "**/*.snap"
  - "fixtures/**"
---

## Money is always in minor units
Applies to: src/billing/**, src/payments/**

Amounts are integers in cents. A float or a division by 100 in these paths is a P0.

## New indexes need a migration
Applies to: src/models/**

Production runs with automatic index creation off. A new or changed index needs a migration in `migrations/`.
```

- `ignore:` lists glob patterns of files not to review.
- `Applies to:` limits a rule to matching paths. A rule without it applies everywhere.
- Rules about process (branch names, PR text, test commands) are ignored. Only rules about the code count.

## Evaluation (v0.1.0)

**Planted-bug cases** in a Node.js/MongoDB API: 7 bugs across 4 cases, plus 1 clean refactor. All 7 bugs were found. No false P0 or P1 findings; the clean refactor got none.
- Inverted condition
- Undefined variable
- Silent default change in a "refactor"
- Unawaited write inside a transaction
- Units change that breaks callers outside the diff
- Index spec the database rejects

**[goosy-benchmark](https://github.com/goosy-benchmark) grafana PRs #27–#36.** Each benchmark PR reverts a real upstream fix while titled as that fix. All 10 were caught with 0 noise, as judged by a separate grading agent against the upstream fix descriptions. On the same PRs, using the benchmark's own forks for each tool:
- CodeAnt caught 7 of 10.
- Greptile reviewed only 6 (it hit trial limits on the rest) and caught 5 of those, plus 1 partial.
- CodeRabbit and Graphite left no reviews to compare. The original fixes are in the repository's git history, which the reviewer can read.

Each review took 20–55 s and cost about $0.35–0.50 at API prices.
