# Deep Test Charter Template for an Alfabet-Style Area

Use this charter before writing Playwright code for a complex Alfabet-style betting or trading workflow. The goal is to turn product risk into observable states, deterministic fixtures, and reviewable evidence instead of starting with a shallow click path.

This template is public-safe by design. Keep examples synthetic and generalized. Do not include proprietary selectors, internal routes, credentials, environment variables, customer data, screenshots, traces, videos, HAR files, storage state, or private implementation details.

## Charter header

| Field | Notes |
|---|---|
| Area under test | Example: market detail, navigation shell, betslip review, account-safe read-only flow |
| Primary risk | The user-impacting or money-impacting failure this charter protects against |
| Test depth | smoke, navigation, state-model, contract-observed UI, visual semantics, flake-forensics |
| Fixture world | Synthetic and deterministic data profile used to force important states |
| Evidence target | The trace annotations, console categories, network observations, or screenshots policy needed for triage |
| Out of scope | Explicitly list private data, live volatility, real accounts, and non-deterministic dependencies to avoid |

## Product risk statement

Write one sentence that explains why this test area deserves deeper E2E attention.

```md
If [state or transition] occurs while [user intent] is in progress, the UI must [safe behavior], otherwise [user-impacting risk].
```

Example:

```md
If a synthetic market changes while a user has a pending selection, the UI must require visible review before any primary money-impacting action is enabled, otherwise the user may act on stale context.
```

## User-visible model

Describe the behavior as states and transitions that can be observed by a user or by public-safe test adapters.

| State | How the user recognizes it | Allowed transitions | Forbidden transitions |
|---|---|---|---|
| Initial stable state |  |  |  |
| User intent started |  |  |  |
| External or fixture transition observed |  |  |  |
| Review or validation required |  |  |  |
| Safe recovery state |  |  |  |

### Candidate state path

```text
stable area → user intent started → deterministic fixture transition → review/validation state → safe recovery
```

Keep the first automated path short enough that a failure can be understood from the test name, `test.step` labels, and annotations.

## Invariants

List the safety properties that should hold after every relevant step, not only at the final assertion.

- **Action safety:** unavailable, suspended, stale, or invalid states do not expose an enabled primary action.
- **Causal validation:** warnings appear near the affected control and clear when the cause is removed.
- **Context coherence:** badges, summaries, panels, and detail views agree on the same selection or market state.
- **Review explicitness:** changed context requires an intentional accept, clear, or retry path.
- **Recovery clarity:** reloads, retries, and fallback states explain what happened and do not strand the user.

Add area-specific invariants here:

1. `<area-specific invariant>`
2. `<area-specific invariant>`
3. `<area-specific invariant>`

## Fixture design

Define the smallest deterministic world that can force the risk.

| Fixture capability | Required? | Notes |
|---|---:|---|
| Stable initial entity | yes | Synthetic sport/event/market/account-safe entity |
| Status transition |  | available, suspended, closed, changed, unavailable |
| Validation boundary |  | low/high stake, empty input, duplicate action, stale state |
| Recovery trigger |  | reload, retry, route fallback, transient error category |
| Seed or scenario ID | yes | Log this as an annotation for reproducibility |

Public-safe rule: the fixture should describe behavior, not reveal internal endpoints or payloads.

## Playwright evidence plan

A deep charter should decide what a useful failure must explain before the test is written.

| Evidence | Use when | Public-safe handling |
|---|---|---|
| `test.step` labels | Every state transition | Name user intent and expected state |
| `testInfo.annotations` | Risk, model, fixture, seed | Use synthetic IDs only |
| Trace on retry | Debugging timing or state mismatch | Do not commit trace artifacts |
| Console/page errors | Triage unexpected client failures | Classify categories; avoid private payloads |
| Network observations | Correlating UI with observed contract category | Record route category/status, not private URLs or bodies |
| Screenshot | Visual semantic mismatch | Use only synthetic/public-safe data; do not commit raw artifacts |

## Assertion strategy

Prefer user-facing semantics and stable adapters over implementation details.

```ts
test('synthetic changed selection requires explicit review', async ({ page }, testInfo) => {
  testInfo.annotations.push(
    { type: 'risk', description: 'stale context before money-impacting action' },
    { type: 'fixture', description: 'synthetic-market-transition-v1' },
    { type: 'model', description: 'stable → selected → changed → review-required' },
  );

  const ui = createPublicSafeAreaAdapter(page);
  const world = createSyntheticFixtureWorld({ seed: 'charter-seed', transition: 'changed' });

  await test.step('open stable synthetic area', async () => {
    await world.route(page);
    await ui.openStableArea();
    await ui.expectStableArea();
  });

  await test.step('start user intent', async () => {
    await ui.startIntent();
    await ui.expectIntentSummary();
  });

  await test.step('force deterministic transition', async () => {
    await world.emitTransition('changed');
    await ui.expectTransitionEvidence('changed');
  });

  await test.step('enforce safe review state', async () => {
    await ui.expectReviewRequired();
    await ui.expectPrimaryActionDisabled();
  });
});
```

The adapter names above are placeholders. Replace them with project-appropriate, public-safe abstractions that hide selectors and preserve readable business behavior.

## Flake controls

Before committing automation from this charter, answer:

- Is the test using deterministic synthetic data instead of live volatility?
- Are waits tied to observable UI or fixture events rather than arbitrary timeouts?
- Does every generated path have a seed, path summary, or scenario ID?
- Can the failure be reproduced from the test output without private artifacts?
- Are animations, responsive layout, and async loading states accounted for where relevant?

## Review checklist

- [ ] The charter protects a named Alfabet-style risk.
- [ ] States, transitions, and invariants are explicit.
- [ ] Test data is synthetic and deterministic.
- [ ] Assertions focus on user-visible semantics.
- [ ] Evidence is sufficient for triage and safe for a public repository.
- [ ] No private selectors, URLs, credentials, storage state, traces, screenshots, videos, or reports are committed.
- [ ] The first automated version is small enough to review and maintain.

## Related lab notes

- [Playwright State-Model Testing](../playwright/state-model-testing.md)
- [Playwright Observability Playbook](../playwright/observability-playbook.md)
- [Flake Forensics Guide](../playwright/flake-forensics.md)
- [Synthetic Fixture Experiment](../experiments/synthetic-fixtures/README.md)
