# Property-Based UI Testing for Alfabet-Style Playwright Flows

Property-based UI testing explores many small variations of user actions and checks that core invariants always hold. For Alfabet-style betting or trading interfaces, this is useful when the exact path is less important than the safety property: stale markets stay disabled, betslip counts remain coherent, validation is causal, and recovery never leaves a half-enabled money-impacting action.

This experiment is public-safe by design. It uses generalized Playwright patterns, fictional data, and pseudocode. It does not include proprietary selectors, private routes, internal APIs, credentials, traces, screenshots, customer data, or implementation details.

## What property-based testing adds

Classic E2E checks often encode one representative journey:

```text
open market → add selection → enter stake → assert review state
```

Property-based testing starts from a stronger question:

> Across many valid action sequences, what must always remain true about the visible UI?

That makes it a good fit for risk areas where a single happy path hides edge cases: rapid add/remove actions, navigation while the betslip is populated, market status changes, validation boundaries, responsive layout changes, and retry flows.

## Candidate properties

| Property | Why it matters | Example invariant |
|---|---|---|
| Action safety | Prevents unsafe continuation when state is stale, closed, or invalid | primary money-impacting action is disabled unless the model says the state is actionable |
| Count coherence | Catches duplicated or orphaned selections | market row, badge, and betslip panel agree on empty vs populated state |
| Validation causality | Avoids confusing or sticky warnings | validation appears only after a triggering input and clears when the trigger is removed |
| Review explicitness | Protects users from changed context | changed or stale selections require a visible review/clear decision before continuation |
| Navigation recovery | Catches deep-link and back-button regressions | after refresh/back, the UI returns to a safe state or explains the fallback |
| Responsive semantics | Prevents layout-only bugs from hiding controls | the same state exposes equivalent semantic controls across viewport classes |

## Public-safe generator sketch

The generator should produce user-intent actions, not private implementation details. Keep the domain vocabulary generic and route product-specific behavior through adapters.

```ts
type UiAction =
  | { kind: 'openMarket'; market: 'synthetic-a' | 'synthetic-b' }
  | { kind: 'toggleSelection'; index: 0 | 1 | 2 }
  | { kind: 'enterStake'; value: 'empty' | 'min' | 'typical' | 'tooHigh' }
  | { kind: 'clearBetslip' }
  | { kind: 'refreshPage' }
  | { kind: 'changeViewport'; size: 'mobile' | 'desktop' }
  | { kind: 'fixtureTransition'; event: 'priceChanged' | 'marketSuspended' | 'marketRecovered' };

type GeneratedCase = {
  seed: number;
  actions: UiAction[];
  protectedRisk: 'stale-action' | 'validation' | 'selection-coherence' | 'recovery';
};
```

The action list is intentionally small. A useful property-based UI test is not random clicking; it is constrained exploration of realistic user intent under deterministic fixture worlds.

## Playwright test shape

```ts
import { expect, test } from '@playwright/test';

test('generated UI action paths preserve betslip safety invariants', async ({ page }, testInfo) => {
  const generated = generatePublicSafeActionCase({
    seed: 73041,
    maxActions: 8,
    allowedRisks: ['stale-action', 'validation', 'selection-coherence'],
  });

  testInfo.annotations.push(
    { type: 'property-seed', description: String(generated.seed) },
    { type: 'risk', description: generated.protectedRisk },
  );

  const world = createSyntheticWorldForGeneratedCase(generated);
  const ui = createPublicSafeBettingUiAdapter(page, world);

  await world.route(page);
  await page.goto(world.publicSafeEntryPath());

  for (const [index, action] of generated.actions.entries()) {
    await test.step(`${index + 1}. ${describeAction(action)}`, async () => {
      await ui.perform(action);
      await expectGlobalSafetyInvariants(ui, generated);
    });
  }
});
```

The important rule is the invariant check after every generated action. The failure should identify the seed, action index, action description, protected risk, and invariant that broke.

## Shrinking failures into debuggable cases

A property-based UI failure is only useful if it can be minimized. When a generated path fails, shrink it into the smallest sequence that still reproduces the invariant violation.

```text
Original failing path, seed 73041:
openMarket(A) → toggleSelection(0) → enterStake(typical) → changeViewport(mobile) → fixtureTransition(priceChanged) → refreshPage

Shrunk repro:
toggleSelection(0) → fixtureTransition(priceChanged)

Invariant:
changed selection requires review message and disabled primary action
```

Practical shrinking strategy:

1. Remove one action at a time and rerun the invariant.
2. Prefer deleting layout and navigation noise before deleting the suspected state transition.
3. Replace boundary values with simpler equivalents when the same invariant still fails.
4. Stop when removing any remaining action loses the failure.
5. Commit the minimized case as a normal deterministic regression test, not as an opaque random test.

## Guardrails against noisy randomness

Property-based UI testing can become flaky if it is unconstrained. Keep the first implementation boring and reproducible:

- fixed seed in CI, optional rotating seed in scheduled experiments;
- short action paths, usually five to ten actions;
- synthetic fixture worlds rather than volatile live data;
- semantic adapters instead of raw CSS selectors;
- explicit action preconditions so impossible actions are skipped or transformed;
- invariant failures with clear evidence, not just `expect` line numbers;
- generated traces and videos retained only by the test system, never committed to the public repo.

## Action preconditions

Generated actions should respect what a user could reasonably attempt. Preconditions keep the test focused on meaningful state exploration.

| Action | Example precondition | If unmet |
|---|---|---|
| `toggleSelection` | market is visible and selection index exists | replace with `openMarket` or skip |
| `enterStake` | betslip has at least one selection | replace with `toggleSelection` |
| `fixtureTransition(priceChanged)` | a selected item exists | use `marketSuspended` or skip |
| `clearBetslip` | betslip is populated | no-op with annotation |
| `refreshPage` | current route is recoverable | assert safe fallback after reload |
| `changeViewport` | viewport class is relevant to the test | keep only if responsive semantics are in scope |

## Evidence to attach on failure

A failure report should be small enough for a human to act on:

```md
### Property-Based UI Failure Note

- Property: stale selections require explicit review
- Seed: 73041
- Action index: 5
- Failing action: fixtureTransition(priceChanged)
- Shrunk path: toggleSelection(0) → fixtureTransition(priceChanged)
- Expected invariant: review message visible; primary action disabled
- Observed category: action remained enabled after stale state
- Evidence: Playwright trace on retry, console errors, sanitized fixture transition list
- Regression candidate: deterministic spec from shrunk path
```

## Promotion path

1. Start as a scheduled experiment with one fixed synthetic world and one risk category.
2. Record generated actions and seeds in test annotations.
3. Convert useful failures into deterministic Playwright regression specs.
4. Add rotating seeds only after the invariant messages and shrinker are reliable.
5. Keep public examples generalized; keep product selectors, endpoints, traces, and private data out of the repository.

The end state is not a suite of random browser clicks. It is a disciplined way to discover unexpected state combinations, reduce them to readable regressions, and strengthen Alfabet-style Playwright coverage around the user-facing safety properties that matter most.

## Related lab notes

- [Playwright State-Model Testing for Alfabet-Style UIs](../playwright/state-model-testing.md)
- [Synthetic Fixture Worlds for Alfabet-Style Playwright Tests](synthetic-fixtures/README.md)
- [Playwright Flake Forensics for Alfabet-Style E2E](../playwright/flake-forensics.md)
