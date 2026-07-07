# Playwright State-Model Testing for Alfabet-Style UIs

State-model testing treats an Alfabet-style betting or trading interface as a set of visible states and allowed transitions, not as one long happy-path script. This is useful when navigation, market availability, betslip behavior, validation, and recovery can change independently while the user is already on the page.

This note is public-safe by design: it uses generic product language, pseudocode, and reusable Playwright patterns. It does not depend on private selectors, internal routes, proprietary APIs, credentials, screenshots, traces, or real customer data.

## Why model the UI as states?

Traditional E2E tests often read like:

```text
open page → click selection → enter stake → assert action is enabled
```

That proves one path, but it does not explain what should happen when a market suspends, a selection changes price, navigation refreshes context, validation appears, or the user tries to recover from an interrupted flow.

A state model makes the test design explicit:

- **states** describe what the user can see or do;
- **events** describe public-safe actions or observed changes;
- **transitions** describe legal movement between states;
- **invariants** describe safety properties that must always hold.

For Alfabet-style products, this is especially valuable because the riskiest bugs often live in the gaps between visible UI, live data freshness, and money-impacting action affordances.

## Candidate model areas

| Model area | Example states | Example events | High-value invariant |
|---|---|---|---|
| Navigation | shell loaded, sport selected, event detail, empty route, unavailable route | open sport, open event, refresh, back, deep link | the user always has a clear route back to a stable product area |
| Market status | loading, available, suspended, reopened, closed, unavailable | fixture update, manual refresh, observed status change | unavailable or suspended markets never expose an enabled primary action |
| Betslip | empty, selection added, stake entered, validation required, review required, cleared | add selection, remove selection, change stake, clear, accept change | stale or changed selections require explicit review before proceeding |
| Validation | no errors, field error, flow-level warning, blocking error, resolved | invalid stake, boundary stake, status change, retry | validation messages are near the affected control and clear when the cause is removed |
| Recovery | transient loading, recoverable error, retrying, stable fallback | network interruption, reload, retry, logout boundary | recovery either returns to a safe state or explains why the action cannot continue |

## Minimal state-machine sketch

This is intentionally pseudocode. It describes a test-design shape, not a committed dependency or private implementation.

```ts
type UiState =
  | 'eventVisible'
  | 'marketAvailable'
  | 'marketSuspended'
  | 'betslipEmpty'
  | 'selectionAdded'
  | 'stakeEntered'
  | 'reviewRequired'
  | 'validationBlocking'
  | 'safeFallback';

type UiEvent =
  | 'openEvent'
  | 'addSelection'
  | 'enterStake'
  | 'marketSuspends'
  | 'priceChanges'
  | 'clearBetslip'
  | 'retryRecovery';

const transitions = [
  ['eventVisible', 'addSelection', 'selectionAdded'],
  ['selectionAdded', 'enterStake', 'stakeEntered'],
  ['stakeEntered', 'priceChanges', 'reviewRequired'],
  ['selectionAdded', 'marketSuspends', 'marketSuspended'],
  ['reviewRequired', 'clearBetslip', 'betslipEmpty'],
  ['marketSuspended', 'retryRecovery', 'safeFallback'],
] as const;
```

A real implementation can start much smaller: one model, three to five states, and a few invariants that matter enough to keep in CI.

## Playwright implementation pattern

The model runner should keep the product-specific details behind public-safe adapter functions. The test reads as business behavior; the adapter owns how to interact with the generic UI surface.

```ts
test('market and betslip state model keeps unsafe actions disabled', async ({ page }) => {
  const ui = createPublicSafeBettingUiAdapter(page);
  const model = createSmallMarketBetslipModel();

  await ui.openSyntheticEventWorld('available-market');

  for (const step of model.generatePath({ maxSteps: 6 })) {
    await test.step(`${step.from} --${step.event}--> ${step.to}`, async () => {
      await ui.perform(step.event);
      await ui.expectState(step.to);
      await expectCoreInvariants(ui);
    });
  }
});
```

The important design choice is the invariant call after every transition. It prevents the suite from only checking the final screen.

## Core invariants to check after every transition

Start with invariants that protect user trust and money-impacting behavior:

1. **Disabled means disabled.** If the model says a market is suspended, closed, unavailable, or stale, the corresponding action is not clickable and communicates why.
2. **Selection count is coherent.** Empty, one-selection, and cleared states agree across the market row, betslip badge, and betslip panel.
3. **Validation is causal.** Stake or flow warnings appear only when their trigger exists, and disappear when the user removes that trigger.
4. **Review is explicit.** Changed-price or stale-selection states require a visible review/accept/clear decision before any money-impacting continuation.
5. **Recovery is safe.** A retry, reload, or transient network failure cannot leave a half-enabled action with stale context.
6. **Navigation preserves orientation.** Back, refresh, and deep-link recovery either restore the selected context or show an understandable fallback.

## Choosing paths without creating flake

State-model testing does not mean random clicking in production-like live data. Keep the first version deterministic:

- run against synthetic fixture worlds or stable test data;
- generate short paths with a fixed seed;
- log the seed and transition list as test annotations;
- avoid real money, real accounts, and private production data;
- prefer semantic locators and adapter methods over raw CSS selectors;
- cap path length so failure triage remains readable.

Example annotation shape:

```text
model: market-betslip-v1
seed: 2026-07-07
path: eventVisible/openEvent → marketAvailable/addSelection → selectionAdded/priceChanges → reviewRequired/clearBetslip
risk: stale-action-affordance
```

## Triage signal from failures

A good state-model failure should answer:

- Which state was expected?
- Which event just ran?
- Which invariant failed?
- Was the failure caused by UI semantics, fixture setup, timing, selector drift, or a real product regression?
- Is the model too strict, or did the UI allow an unsafe transition?

Recommended Playwright evidence for this layer:

- trace on retry;
- screenshot only when it contains public-safe synthetic data;
- console and page error capture;
- sanitized network category notes, not raw private payloads;
- transition path and seed in the test output.

## Promotion checklist

Before promoting a model into a scheduled or pre-merge lane, verify:

- the model has a named product risk;
- every state is observable through public-safe semantics;
- the fixture world can deterministically force important states;
- invariant failures are specific enough to debug;
- the path length is short enough for maintainers to understand;
- no private Alfabet selector, endpoint, credential, trace, screenshot, or customer data is committed.

## Related lab notes

- [Alfabet Playwright Test Architecture Map](../alfabet/test-architecture-map.md)
- [Alfabet Testing Lab Manifesto](../alfabet/testing-lab-manifesto.md)
