# Setup

## 1. One-time change to the target repo

Apply `demo/prereq_add_median_guard.patch` to `taxi-analytics-pipeline` and
merge it to `main` (normal PR). It adds
`assert_int_trips_enriched_speed_distribution_sane.sql`, a distribution-level
guard asserting the median trip speed stays plausible. Without it, a uniform
units regression passes silently: row-level bounds can't detect a systemic
shrink, and a naive row-level lower bound false-positives on legitimate
gridlock crawls.

```bash
cd taxi-analytics-pipeline
git apply ../sentinel-agent-loop/demo/prereq_add_median_guard.patch
```

## 2. Secrets (on this repo)

| Secret | What it is |
|---|---|
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude Code OAuth token for `anthropics/claude-code-action` (same one the jira-agent workflow uses) |
| `ANTHROPIC_API_KEY` | Used by `sentinel analyze` in the detect job |
| `TARGET_REPO_TOKEN` | Fine-grained PAT, **scoped to `qraza/taxi-analytics-pipeline` only**, permissions: Contents (read/write), Pull requests (read/write). Nothing else. |

The narrow PAT scope is part of the security story — and the agent itself
never sees it: the PAT lives only in workflow steps (checkout, push, PR
creation), while the agent has no GitHub API access at all.

Because the PAT acts as the repo owner, GitHub treats the owner as the PR
author and blocks self-approval; on a single-maintainer repo, merge the
agent's PRs with the audited "bypass rules" option after reviewing. A team
reviewer, or a dedicated GitHub App identity for the loop, removes that
limitation.

## 3. The human gate (target repo)

On `taxi-analytics-pipeline` → Settings → Branches → protect `main`:

- Require a pull request before merging, **1 approval**
- Require status checks: the existing `CI` workflow
- Do not allow bypassing the above

This is the approval gate. It is enforced by GitHub, so no prompt injection,
agent error, or workflow bug can route around it.

## 4. Sanity checks before the first real run

- `workflow_dispatch` the loop **without** `demo_break`: it should build green
  and stop (no agent invocation, no PR).
- Then run with `demo_break: true` and follow the run summary link to the PR.

## Known sharp edges

- The detect job uploads `capstone.duckdb` as an artifact so the propose job's
  MCP server samples the same warehouse state that failed. If the fixture
  grows, switch to rebuilding it in the propose job instead.
- `sentinel-mcp` is installed from GitHub (it isn't on PyPI). When you
  publish it, swap the install line in `dq-loop.yml` back to
  `uv pip install --system dbt-sentinel sentinel-mcp` — publishing is worth
  doing anyway; it strengthens the "reusable components" story.
- Also install the Claude Code GitHub App (github.com/apps/claude) on BOTH repos — the action refuses to run where it isn't installed.
- Scheduled runs on a green build exit quietly by design — the loop is a
  tripwire, not a chatterbox.
