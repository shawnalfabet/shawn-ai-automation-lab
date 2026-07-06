# Alfabet Playwright Testing Lab

Public R&D lab for **deep Alfabet-style Playwright E2E testing**.

This is not a shallow smoke-test repo. It is a testing architecture lab for betting/trading-style interfaces: state changes, live markets, navigation complexity, betslip safety, observability, flake forensics, and agent-assisted test design.

## Core idea

```text
creative test design → public-safe experiment → normal Playwright code → deterministic CI signal
```

The repo is intentionally public-safe. It documents reusable strategies, examples, templates, and workflows without exposing proprietary source code, private selectors, credentials, traces, screenshots, or environment details.

## Start here

| Document | Purpose |
|---|---|
| [`alfabet/testing-lab-manifesto.md`](alfabet/testing-lab-manifesto.md) | Why this lab exists and what “deep testing” means |
| [`alfabet/deep-testing-roadmap.md`](alfabet/deep-testing-roadmap.md) | Roadmap for advanced Alfabet Playwright experiments |
| [`alfabet/testing-strategy.md`](alfabet/testing-strategy.md) | Practical testing strategy foundation |
| [`playwright/stacked-pr-workflow.md`](playwright/stacked-pr-workflow.md) | Stacked PR workflow for test batches |
| [`templates/playwright-agent-batch-pr.md`](templates/playwright-agent-batch-pr.md) | PR template for generated test batches |

## Workstreams

- State-model testing for market and betslip state
- Synthetic fixture worlds for deterministic test data
- Property-based UI action exploration
- Playwright observability and failure evidence
- Flake forensics and triage loops
- Risk-based coverage for money-impacting UI
- Agent-assisted planner/generator/healer workflows
- Public-safe example specs for Alfabet-style flows

## Not in scope

- generic AI market research
- revenue recovery notes
- private ParleyIt/Alfabet source code
- real internal URLs, credentials, traces, screenshots, videos, or auth state
- noisy contribution farming
