# Playwright stacked PR workflow

Use this when an existing Playwright-agent proof-of-concept PR is open and new E2E test batches should avoid piling more commits directly onto the parent branch.

## Goal

Create many small, reviewable Playwright batches with minimal production impact:

```text
staging
  ↑
parent PR branch: poc/playwright-agents
  ↑
child PR branch: agent/playwright-batch-YYYYMMDD-HHMM
```

Each batch is a normal PR whose base is the existing parent branch.

## Why this helps

- Keeps each test batch reviewable.
- Avoids destabilizing the parent PR with many unrelated commits.
- Lets CI/review happen per batch.
- Makes it easy to retarget child PRs to `staging` after the parent PR merges.
- Keeps committed artifacts as normal Playwright tests — no runtime AI dependency.

## Command skeleton

```bash
git fetch origin --prune
git status --short

BRANCH="agent/playwright-batch-$(date -u +%Y%m%d-%H%M)"
git checkout -B "$BRANCH" origin/poc/playwright-agents

# Add e2e/specs/<batch>.md
# Add e2e/tests/.../<batch>.spec.ts
# Run safe checks / targeted tests where possible

git diff --stat
git add e2e/specs e2e/tests
git commit -m "test(e2e): add agent generated <area> batch"
git push -u origin HEAD

gh pr create \
  --base poc/playwright-agents \
  --head "$BRANCH" \
  --title "test(e2e): add Playwright agent batch $(date -u +%Y-%m-%d)" \
  --body-file /tmp/playwright-agent-batch-pr.md
```

## Guardrails

- Do not commit auth state, traces, screenshots, `.env` files, reports, or node_modules.
- Prefer existing page objects and fixtures.
- Avoid brittle `waitForTimeout` waits.
- Keep batches to 2–3 tests.
- Do not merge or force-push without explicit approval.
