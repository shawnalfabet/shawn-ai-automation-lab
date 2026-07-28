# Alfabet Playwright Testing Lab

Public R&D lab for **deep Alfabet-style Playwright E2E testing**: stateful UI behavior, live-market risk, deterministic fixture worlds, observability, flake forensics, and agent-assisted test design.

This is not a smoke-test collection or contribution-farming repo. It is a public-safe testing architecture portfolio for betting/trading-style products where UI correctness affects trust, timing, validation, and money-impacting decisions.

## Lab thesis

```text
risk model → synthetic world → Playwright invariant → observable evidence → maintainable signal
```

Strong Playwright coverage for Alfabet-style interfaces should answer more than “did the page load?” It should explain:

- which user or business risk a test protects;
- which state transition, validation boundary, or recovery path was exercised;
- what evidence a reviewer gets when the test fails;
- why the test is deterministic enough to trust in CI;
- how public examples avoid private product details.

The repository intentionally documents reusable strategies, templates, and harness sketches without exposing proprietary ParleyIt/Alfabet source code, private selectors, internal URLs, credentials, browser storage state, screenshots, traces, HAR files, customer data, or implementation details.

## Roadmap and index

| Area | Artifact | Status | What it contributes |
|---|---|---:|---|
| Lab direction | [`alfabet/testing-lab-manifesto.md`](alfabet/testing-lab-manifesto.md) | Published | Defines the standard for deep E2E R&D rather than shallow browser checks. |
| Architecture | [`alfabet/test-architecture-map.md`](alfabet/test-architecture-map.md) | Published | Maps smoke, navigation, state-model, observed-contract, visual semantics, flake, and agent-plan layers. |
| Strategy foundation | [`alfabet/testing-strategy.md`](alfabet/testing-strategy.md) | Published | Establishes practical public-safe testing principles for Alfabet-style products. |
| Roadmap | [`alfabet/deep-testing-roadmap.md`](alfabet/deep-testing-roadmap.md) | Published | Tracks larger R&D directions and candidate future experiments. |
| State models | [`playwright/state-model-testing.md`](playwright/state-model-testing.md) | Published | Models navigation, market status, betslip state, validation, and recovery as transitions. |
| Observability | [`playwright/observability-playbook.md`](playwright/observability-playbook.md) | Published | Shows how traces, video, network, console, annotations, and evidence aid triage. |
| Flake forensics | [`playwright/flake-forensics.md`](playwright/flake-forensics.md) | Published | Classifies flaky E2E failures and gives a debugging decision tree. |
| Synthetic data | [`experiments/synthetic-fixtures/README.md`](experiments/synthetic-fixtures/README.md) | Published | Describes deterministic public-safe fixture worlds for market and betslip behavior. |
| Property exploration | [`experiments/property-based-ui-testing.md`](experiments/property-based-ui-testing.md) | Published | Explores generated UI action sequences with invariant checks. |
| Agent workflow | [`experiments/agent-test-designer.md`](experiments/agent-test-designer.md) | Published | Defines planner/generator/healer roles while keeping committed tests normal and reviewable. |
| Charters | [`templates/deep-test-charter.md`](templates/deep-test-charter.md) | Published | Provides a reusable deep exploratory charter template for one product area. |
| Postmortems | [`templates/failure-postmortem.md`](templates/failure-postmortem.md) | Published | Structures E2E failure analysis around root cause, signal, and prevention. |
| Risk coverage | [`playwright/risk-based-coverage.md`](playwright/risk-based-coverage.md) | Published | Prioritizes money-impacting flows, stale data, blocked actions, latency, and responsive layout. |
| API/UI correlation | [`playwright/api-ui-correlation.md`](playwright/api-ui-correlation.md) | Published | Correlates visible UI assertions with sanitized network observations without exposing private APIs. |
| Latency resilience | [`playwright/latency-resilience.md`](playwright/latency-resilience.md) | Published | Models slow, pending, stale, retry, and navigation-during-latency behavior with deterministic Playwright evidence. |
| Visual semantics | [`playwright/visual-semantics.md`](playwright/visual-semantics.md) | Published | Defines semantic visual assertions for status, actionability, responsive layout, and safe interpretation. |
| Locator contracts | [`playwright/locator-contracts.md`](playwright/locator-contracts.md) | Published | Sets a public-safe locator strategy for semantic roles, scoped regions, and durable test ids. |
| Actionability guardrails | [`playwright/actionability-guardrails.md`](playwright/actionability-guardrails.md) | Published | Defines how to prove enabled, disabled, pending, stale, and validation states are safe before money-impacting actions. |
| PR workflow | [`playwright/stacked-pr-workflow.md`](playwright/stacked-pr-workflow.md) | Published | Documents a disciplined stacked-PR approach for Playwright automation batches. |
| Batch template | [`templates/playwright-agent-batch-pr.md`](templates/playwright-agent-batch-pr.md) | Published | Gives a reviewable PR description format for agent-assisted test batches. |

## How to read the lab

### If you are designing a suite

Start with the [testing lab manifesto](alfabet/testing-lab-manifesto.md), then use the [test architecture map](alfabet/test-architecture-map.md) to decide which layer owns each risk. Promote only the behaviors that need browser-level evidence into Playwright E2E.

### If you are modeling live-market behavior

Read the [state-model testing notes](playwright/state-model-testing.md), [synthetic fixture experiment](experiments/synthetic-fixtures/README.md), and [property-based UI testing experiment](experiments/property-based-ui-testing.md). Together they show how to move from fixed happy paths to transition-oriented tests with invariants.

### If you are debugging failures

Use the [observability playbook](playwright/observability-playbook.md), [flake forensics guide](playwright/flake-forensics.md), and [failure postmortem template](templates/failure-postmortem.md). The goal is to classify failures into actionable evidence, not hide them behind retries.

### If you are planning coverage

Use the [risk-based coverage matrix](playwright/risk-based-coverage.md), [API/UI correlation strategy](playwright/api-ui-correlation.md), [visual semantics guide](playwright/visual-semantics.md), and [deep test charter template](templates/deep-test-charter.md) to connect each test to product risk, expected evidence, and maintenance cost.

### If you are using agents

Use the [agent test designer workflow](experiments/agent-test-designer.md) and [batch PR template](templates/playwright-agent-batch-pr.md). Agents can propose charters, invariants, and skeletons, but committed code should remain ordinary Playwright that can be reviewed without a hidden runtime.

## Contribution principles

1. **Public-safe by default.** Use generalized names such as market, event, selection, betslip, validation, and recovery. Do not publish real endpoints, selectors, payloads, credentials, screenshots, videos, traces, HAR files, storage state, customer data, or private implementation details.
2. **Risk before mechanics.** Every artifact should identify the risk it protects: stale market data, invalid continuation, blocked actions, latency, recovery, responsive semantics, or observability gaps.
3. **Prefer deterministic worlds.** Live data can inspire scenarios, but reusable examples should run against synthetic fixtures, adapters, or redacted observations.
4. **Make failures useful.** Tests should leave enough annotations, trace context, console/network summaries, or postmortem structure for a reviewer to classify the failure quickly.
5. **Keep committed assets reviewable.** Agent-generated ideas are acceptable only when the final repository content is clear, maintainable, and normal for a Playwright engineer.

## Not in scope

- generic AI market research;
- arbitrary daily filler or contribution farming;
- private ParleyIt/Alfabet source code;
- real internal URLs, credentials, cookies, tokens, traces, screenshots, videos, HAR files, or auth state;
- proprietary selectors, product-specific DOM structure, or customer/account data;
- fabricated test results or unverified claims.
