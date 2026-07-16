# Risk-Based Playwright Coverage for Alfabet-Style Interfaces

Risk-based coverage keeps E2E automation focused on the failures that would matter most in an Alfabet-style betting/trading product. The goal is not to automate every click path. The goal is to protect money-impacting decisions, live-market state transitions, validation boundaries, and recovery behavior with tests that produce useful evidence.

This document is public-safe: it uses generalized product language, synthetic examples, and reusable Playwright testing patterns. It does not include proprietary selectors, private URLs, credentials, traces, screenshots, videos, storage state, customer data, or internal API contracts.

## Coverage principle

Prioritize a flow when at least one of these is true:

1. a user can make or lose value because of the UI state;
2. stale or delayed market data can change the correct action;
3. a disabled, blocked, or review-required action must be enforced;
4. latency or recovery behavior changes user trust;
5. responsive layout changes the meaning or availability of a control.

A strong suite should make these risks visible through named invariants, deterministic fixtures, and failure artifacts. A weak suite only proves that a happy path rendered once.

## Risk matrix

| Risk area | Why it matters | Example invariant | Primary Playwright signal | Suggested depth |
|---|---|---|---|---|
| Money-impacting continuation | The UI must not allow continuation from an invalid, stale, or unreviewed selection | A changed selection requires explicit review before the primary action is enabled | role-based action state, visible review banner, annotation in `test.step` | state-model or transition test |
| Stale odds or market data | Users need to see whether the displayed price/status is current enough to act on | The displayed state changes from available to review/closed without allowing the previous action | synthetic transition, visible status, sanitized network category | deterministic fixture + retry evidence |
| Disabled or blocked actions | A blocked action must be semantically unavailable, not merely styled differently | Disabled controls are inaccessible to pointer/keyboard and explain the reason | `toBeDisabled`, accessible description, keyboard attempt | accessibility-aware E2E |
| Validation boundaries | Boundary values must produce clear, stable validation and recovery | Invalid input cannot proceed; corrected input clears exactly the relevant error | input state, error text, focus behavior | table-driven UI tests |
| Latency and retry | Loading/retry states must not duplicate actions or show contradictory affordances | During delayed response, only a pending/retry affordance is available and repeated clicks are safe | route delay, actionability log, console summary | controlled network test |
| Responsive layout | Mobile/tablet layouts can hide or reorder controls that affect decisions | The same invariant is reachable and enforceable across viewport projects | project matrix, stable landmarks, no viewport-only selector assumptions | cross-project smoke + risk path |
| Navigation recovery | Back/forward, refresh, and deep links must preserve or safely reset state | Refresh does not silently submit, lose critical warnings, or enable invalid continuation | navigation step, visible state reconstruction | recovery scenario |
| Error copy and evidence | Failures must be understandable without private artifacts | Every blocked state has a user-visible reason and a redacted test annotation | `testInfo.annotations`, screenshot/trace retained outside git | observability check |

## Coverage layers

Use layered tests instead of forcing every risk into a single large E2E script.

### 1. Risk smoke

Fast checks that the most important user journeys render and expose stable landmarks.

```ts
test('market journey exposes decision landmarks', async ({ page }, testInfo) => {
  testInfo.annotations.push({ type: 'risk', description: 'money-impacting navigation shell' });

  await test.step('open synthetic market area', async () => {
    // Public-safe adapter action. Do not hard-code private URLs or selectors.
  });

  await test.step('verify decision landmarks are visible', async () => {
    // Expect accessible headings, status landmarks, and primary action regions.
  });
});
```

### 2. Transition invariants

Stateful tests that drive a synthetic event from available to changed, suspended, closed, or recovered.

```ts
test('changed selection requires explicit review', async ({ page }, testInfo) => {
  testInfo.annotations.push(
    { type: 'risk', description: 'stale value must not continue silently' },
    { type: 'fixture', description: 'synthetic-market-transition' },
  );

  await test.step('start from an available synthetic selection', async () => {
    // Arrange the public-safe fixture world.
  });

  await test.step('move the selection to changed state', async () => {
    // Trigger a deterministic public-safe transition.
  });

  await test.step('assert review state gates continuation', async () => {
    // Assert the visible review requirement and primary action state.
  });
});
```

### 3. Recovery and resilience

Tests that verify the product tells the truth during slow, failed, or retried interactions.

```ts
test('delayed continuation cannot create duplicate intent', async ({ page }, testInfo) => {
  testInfo.annotations.push({ type: 'risk', description: 'latency must not duplicate money-impacting intent' });

  await test.step('delay the synthetic continuation response', async () => {
    // Use Playwright routing or a fixture adapter with redacted categories.
  });

  await test.step('attempt repeated user input', async () => {
    // Exercise pointer and keyboard behavior against stable user-facing semantics.
  });

  await test.step('assert only one pending intent is represented', async () => {
    // Assert visible pending/retry state rather than private request details.
  });
});
```

## Prioritization score

A lightweight score helps decide what deserves E2E depth:

| Factor | 1 | 3 | 5 |
|---|---|---|---|
| User impact | cosmetic or informational | confusing but recoverable | money-impacting or compliance-sensitive |
| State complexity | static page | one async transition | multiple live transitions or recovery paths |
| Observability need | unit/contract signal is enough | UI confirms an integration | only E2E reveals the failure mode |
| Flake risk | deterministic | timing-sensitive | live-like data, animation, cross-browser, or responsive |

Suggested rule:

```text
score = impact + state complexity + observability need + flake risk

4-7   → document or unit/contract coverage may be enough
8-13  → add a focused Playwright scenario
14-20 → add deterministic fixture support, annotations, and failure-evidence expectations
```

The score is not a management metric. It is a forcing function for test design conversations.

## Test design checklist

Before adding a high-risk Playwright scenario, write down:

- **Risk protected:** the user or business harm the test prevents.
- **Invariant:** the stable truth the UI must preserve.
- **Synthetic world:** fixture profile, seed, and transition path.
- **Observable evidence:** visible state, accessible semantics, redacted network category, trace retention policy.
- **Failure class to watch:** stale data, async propagation, disabled-action leak, layout drift, or recovery failure.
- **Minimum assertion set:** no incidental copy, pixel, or private selector coupling.
- **Recovery expectation:** what the user can safely do after failure, refresh, or retry.

## Public-safe reporting pattern

Use names that communicate risk without exposing implementation details:

```md
### Coverage note

- Area: market detail + decision panel
- Risk: changed state must require explicit review
- Fixture: synthetic-live-market-v1
- Viewports: desktop and mobile
- Invariant: primary action is unavailable until the review state is acknowledged
- Evidence: Playwright trace retained on failure; network details summarized by category only
- Follow-up: add state-model coverage for suspended and recovered transitions
```

## Anti-patterns

Avoid these shortcuts:

- automating only the happy path because it is easy to stabilize;
- asserting private DOM structure instead of user-visible semantics;
- using production-like data that cannot be replayed deterministically;
- committing traces, screenshots, videos, storage state, or network payloads;
- treating retries as proof that the risk is covered;
- making one giant scenario responsible for smoke, validation, latency, and recovery at once.

Risk-based coverage should make the Playwright suite smaller, sharper, and more credible. Each test should explain what Alfabet-style product risk it protects and what evidence a failure would leave behind.
