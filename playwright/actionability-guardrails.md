# Playwright Actionability Guardrails for Betting UI

Alfabet-style betting and trading products need more than “button is visible” checks. A strong E2E suite should prove that money-impacting actions are only possible when the UI is in a safe, explainable state.

This note describes a public-safe Playwright strategy for testing actionability without exposing private selectors, internal APIs, production data, or implementation details.

## Risk model

Actionability bugs are high severity because they let a user act during ambiguous state:

| Risk | Example public-safe symptom | Expected guardrail |
|---|---|---|
| Stale market data | A visible price has changed but the primary action still appears available. | Require user acknowledgement or disable the action until state is fresh. |
| Pending transition | Selection is still being validated while a second click is accepted. | Show pending state and reject duplicate continuation. |
| Closed/suspended market | A selection remains clickable after the market is no longer available. | Disable action and expose a clear unavailable reason. |
| Validation mismatch | UI accepts a combination that visible validation says is invalid. | Block continuation and attach validation text to the relevant region. |
| Recovery ambiguity | A failed action leaves the UI half-enabled with no next step. | Provide retry, clear, or navigation recovery with deterministic state. |

## Test design principle

Avoid asserting raw CSS state as the source of truth. Prefer user-facing semantics:

1. locate the meaningful region, such as a market card, selection row, or betslip summary;
2. assert the user-visible state label, validation message, or status affordance;
3. assert whether the next user action is available;
4. attempt the action only when the scenario expects it to be safe;
5. record enough evidence to explain why the action was blocked or allowed.

```ts
// Public-safe sketch: names are generic and not copied from any private product.
const market = page.getByRole('region', { name: /match winner/i });
const selection = market.getByRole('button', { name: /home team/i });
const betslip = page.getByRole('region', { name: /betslip/i });

await expect(market.getByText(/suspended/i)).toBeVisible();
await expect(selection).toBeDisabled();
await expect(betslip.getByText(/selection unavailable/i)).toBeVisible();
```

## Scenario matrix

| Scenario | Setup in a synthetic world | Playwright evidence |
|---|---|---|
| Fresh selection | Market is open, price is current, validation passes. | Selection is enabled; betslip updates; confirmation copy reflects accepted state. |
| Price changed | Price changes after selection but before confirmation. | Continue action is blocked or confirmation is required; trace annotation records the transition. |
| Suspended during pending | Selection enters pending state, then market becomes suspended. | Pending resolves to blocked state; no duplicate click path succeeds. |
| Invalid combination | User combines incompatible selections in a deterministic fixture. | Validation is visible; submit/continue is disabled; invalid item is identified. |
| Network recovery | Safe stub returns a retryable error for a non-production endpoint. | User sees retry/clear path; retry does not create duplicate accepted state. |

## Guardrail assertions

Useful assertions should combine actionability and explanation:

```ts
await test.step('blocked selections explain why they are blocked', async () => {
  const selection = page.getByRole('button', { name: /draw/i });
  await expect(selection).toBeDisabled();
  await expect(page.getByText(/market suspended/i)).toBeVisible();
});
```

For enabled actions, verify the inverse as well:

```ts
await test.step('enabled selection has no stale-state warning', async () => {
  const selection = page.getByRole('button', { name: /away team/i });
  await expect(selection).toBeEnabled();
  await expect(page.getByText(/price changed|market suspended|unavailable/i)).toHaveCount(0);
});
```

## Anti-patterns

- Clicking through disabled or loading states with `force: true` in normal coverage.
- Treating visibility as actionability.
- Checking only the final page state after a click while ignoring pending, stale, and validation transitions.
- Asserting private API payload shapes in public examples.
- Using real screenshots, traces, HAR files, customer data, or production market names as fixtures.

## Observability hooks

When a guardrail fails, the test report should answer:

- which synthetic state world was active;
- which action was attempted;
- whether the UI claimed the action was enabled or disabled;
- which user-visible reason was present;
- which sanitized network category was observed, such as `accepted`, `price_changed`, `suspended`, or `retryable_error`.

A small annotation pattern keeps this reviewable:

```ts
test.info().annotations.push({
  type: 'synthetic-state',
  description: 'market=suspended, selection=blocked, betslip=explains-unavailable',
});
```

## Definition of done

An actionability guardrail test is useful when it proves all three points:

1. the action state matches the visible product state;
2. blocked actions explain the reason in user-facing language;
3. the trace gives enough context to debug stale, pending, unavailable, or validation behavior without private artifacts.
