# Agent Test Designer Workflow for Alfabet-Style Playwright Suites

Agent-assisted test design is useful when it expands the thinking around risk, states, fixtures, and evidence without hiding the final suite behind generated noise. For Alfabet-style betting or trading interfaces, the workflow should produce normal Playwright tests that a human can review, debug, and maintain.

This note is public-safe by design. It uses generalized roles, synthetic examples, and reusable Playwright patterns. It does not include proprietary source code, private selectors, internal URLs, credentials, environment variables, screenshots, traces, customer data, or implementation details.

## Goal

Use an agent as a disciplined test-design collaborator, not as an unchecked code generator.

```text
risk brief → planner → human-readable charter → generator → normal Playwright spec → evidence review → healer notes
```

The committed artifact should be boring in the best way: deterministic tests, semantic page adapters, explicit fixtures, clear assertions, and no dependence on a hidden agent runtime.

## Role split

| Role | Responsibility | Output |
|---|---|---|
| Planner | Converts product risk into testable states and invariants | test charter, model states, fixture needs |
| Generator | Drafts Playwright test skeletons from approved charters | public-safe spec or helper sketch |
| Reviewer | Checks risk coverage, public safety, selector quality, and maintainability | review notes and requested changes |
| Healer | Diagnoses failed tests and proposes minimal fixes | failure classification, smaller repro, candidate patch |
| Human owner | Decides what ships and keeps the suite coherent | committed normal code and docs |

The important boundary: the agent may propose, but the repository stores only readable testing assets.

## Input contract for the planner

A good planner prompt is structured and constrained. It should not ask for private source code or real environment data.

```md
### Test Design Brief

- Area: market detail + betslip
- Public-safe context: synthetic live market with available, suspended, and changed selections
- Primary risk: money-impacting action remains enabled after stale state
- Secondary risks: validation confusion, duplicated selection count, poor refresh recovery
- Allowed evidence: annotations, sanitized console category, synthetic fixture transition log
- Forbidden evidence: real endpoints, tokens, storage state, screenshots, traces, internal selectors
- Desired output: deterministic Playwright charter and candidate test steps
```

This input style keeps the agent focused on the testing problem instead of scraping implementation detail from the application.

## Planner output shape

The planner should produce artifacts that can be reviewed before code exists.

```md
### Candidate Invariant

A changed or suspended selection must require explicit review before any primary money-impacting action is enabled.

### State Model

- Empty betslip
- Selection added from available market
- Stake entered
- Fixture transition observed: price changed or market suspended
- Review required
- Safe recovery: clear, accept review, or navigate away with explanation

### Test Steps

1. Route a synthetic market fixture.
2. Open the market detail page through a public-safe adapter.
3. Add one selection and enter a typical stake.
4. Trigger a deterministic fixture transition.
5. Assert visible review semantics and disabled primary action.
6. Attach sanitized evidence naming the invariant and transition.
```

If the output is not reviewable in markdown, it is not ready to become code.

## Generator guardrails

The generator is allowed to create a draft, but the draft must obey normal Playwright engineering standards:

- use semantic adapter methods such as `market.openSyntheticMarket()` instead of raw private selectors;
- keep fixture names fictional and deterministic;
- place assertions after every state-changing step when the protected risk requires it;
- add `test.step` names that describe user intent;
- add `testInfo.annotations` for risk, fixture profile, state model, and seed when applicable;
- avoid sleeps, random live data, hidden retries, and broad locator fallbacks;
- do not commit generated traces, videos, screenshots, storage state, HAR files, or reports.

A generator that cannot explain the invariant should not write the spec.

## Public-safe Playwright sketch

```ts
import { expect, test } from '@playwright/test';

test('changed synthetic selection requires review before primary action', async ({ page }, testInfo) => {
  testInfo.annotations.push(
    { type: 'risk', description: 'money-impacting stale selection acceptance' },
    { type: 'fixture', description: 'synthetic-market-transition-v1' },
    { type: 'state-model', description: 'available → selected → stake-entered → changed → review-required' },
  );

  const world = createSyntheticMarketWorld({ transition: 'selectionChanged' });
  const ui = createPublicSafeTradingUi(page, world);

  await test.step('open a deterministic synthetic market', async () => {
    await world.route(page);
    await ui.openMarketDetail();
  });

  await test.step('build a user-visible pending action', async () => {
    await ui.addSelection({ index: 0 });
    await ui.enterStake('typical');
    await expect(ui.betslipSummary()).toContainText('1 selection');
  });

  await test.step('observe a deterministic market transition', async () => {
    await world.emitTransition('selectionChanged');
    await ui.expectTransitionEvidence('selection changed');
  });

  await test.step('assert safe review state', async () => {
    await expect(ui.reviewMessage()).toBeVisible();
    await expect(ui.primaryMoneyAction()).toBeDisabled();
  });
});
```

This sketch intentionally leaves product-specific details behind adapters. The useful part is the design shape: fixture transition, semantic UI actions, explicit evidence, and a safety invariant.

## Healer workflow

A healer should not automatically patch around failures. Its job is to classify and reduce.

| Failure signal | Healer question | Acceptable next action |
|---|---|---|
| Timeout waiting for review message | Did the fixture transition happen, or did the UI miss it? | inspect trace locally, add transition evidence, reduce steps |
| Primary action remained enabled | Is the model state stale, changed, or still valid? | create minimized regression around the transition |
| Locator ambiguity | Is the user-facing role/name duplicated? | improve semantic adapter or accessible naming expectation |
| Console exception | Is this a product bug or test harness bug? | attach sanitized classification and isolate reproducer |
| Network delay | Is the test using live volatility instead of deterministic fixture data? | replace with synthetic route or explicit observed contract category |

The healer output should read like a failure postmortem draft, not like a magical retry patch.

## Review checklist before commit

- Does the test protect a named Alfabet-style risk rather than a decorative click path?
- Are all selectors, routes, fixture names, and evidence public-safe?
- Can the test run without an agent service at runtime?
- Are generated ideas converted into normal Playwright code or documentation?
- Does the assertion verify user-visible semantics, not private implementation state?
- Would a reviewer understand the failure from test steps and annotations?
- Is the smallest useful artifact being committed today?

## Promotion path

1. Start with a markdown charter from a risk brief.
2. Generate a small deterministic spec or harness sketch.
3. Review public safety and replace product details with semantic adapters.
4. Run the normal project checks.
5. Commit the human-readable artifact.
6. If a failure is found later, use the healer workflow to classify and minimize before changing code.

The long-term value is not that an agent can write many tests. The value is a repeatable design loop that turns high-risk Alfabet-style states into maintainable Playwright coverage with clear evidence and public-safe documentation.

## Related lab notes

- [Property-Based UI Testing for Alfabet-Style Playwright Flows](property-based-ui-testing.md)
- [Synthetic Fixture Worlds for Alfabet-Style Playwright Tests](synthetic-fixtures/README.md)
- [Playwright Observability Playbook for Alfabet-Style E2E Tests](../playwright/observability-playbook.md)
- [Playwright State-Model Testing for Alfabet-Style UIs](../playwright/state-model-testing.md)
