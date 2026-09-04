# Setup

## 1. One-time change to the target repo

Apply `demo/prereq_add_speed_floor.patch` to `taxi-analytics-pipeline` and
merge it to `main` (normal PR). It upgrades
`dbt_project/tests/assert_int_trips_enriched_speed_within_bounds.sql` with a
1 mph plausibility floor for sustained trips. Without it, the demo's seeded
units bug passes silently — an upper bound alone cannot detect a uniform
shrink.

```bash
cd taxi-analytics-pipeline
git apply ../sentinel-agent-loop/demo/prereq_add_speed_floor.patch
```

## 2. Secrets (on this repo)

| Secret | What it is |
|---|---|
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude Code OAuth token for `anthropics/claude-code-action` (same one the jira-agent workflow uses) |
| `ANTHROPIC_API_KEY` | Used by `sentinel analyze` in the detect job |
| `TARGET_REPO_TOKEN` | Fine-grained PAT, **scoped to `qraza/taxi-analytics-pipeline` only**, permissions: Contents (read/write), Pull requests (read/write). Nothing else. |

The narrow PAT scope is part of the security story: even a fully compromised
agent run can only touch one repo, on a branch, behind review.

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
- Scheduled runs on a green build exit quietly by design — the loop is a
  tripwire, not a chatterbox.
