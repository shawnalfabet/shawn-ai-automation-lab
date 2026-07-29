# Market State Oracles for Alfabet-Style Playwright Suites

Market state oracles are public-safe rules that decide whether an Alfabet-style betting/trading UI is telling the same story across visible status, actionability, decision context, and observed async evidence. They are useful when a page can be technically rendered while still being unsafe or misleading because the market state changed underneath the user.

This document uses generalized terms such as market, selection, decision panel, price, status region, and review-required state. It does not include private selectors, internal URLs, request paths, credentials, storage state, screenshots, traces, HAR files, customer data, or proprietary implementation details.

## Oracle thesis

```text
oracle = visible state + allowed action + preserved context + sanitized evidence
```

A Playwright test should not ask only “is the button enabled?” or “does the row exist?” It should ask whether all user-facing signals agree about the safe next action.

For Alfabet-style products, the important oracle is often negative:

```text
if the market state is uncertain, changed, closed, or invalid, the UI must not present a confident continuation path
```

## State-to-oracle matrix

| Market condition | User risk | Oracle rule | Playwright evidence |
|---|---|---|---|
| Available selection | User should be able to continue from current information | Selection summary is visible, price/context is present, and primary action is enabled only after required input | role/text assertions, scoped decision panel |
| Price changed | User may continue from stale confidence | Review-required message is adjacent to affected selection and continuation requires explicit acknowledgement | status region, changed selection copy, gated action |
| Suspended market | User may believe the selection is still actionable | Suspended/closed state is visible and money-impacting actions are blocked | status assertion, disabled primary action |
| Closed event | User may try to recover into an impossible flow | Navigation or panel explains closure and offers a safe route out | alert/status, preserved non-sensitive context |
| Pending validation | User may double-submit or assume success | Pending state is visible, repeat input is idempotent, and no second intent appears | controlled latency, actionability check, annotation |
| Recoverable failure | User may lose context or duplicate intent on retry | Retry path preserves selection context and explains whether the prior intent is unresolved | alert text, retry control, redacted network category |

## Public-safe oracle harness

Keep implementation details behind a fixture-world adapter. The test should express the product truth without committing private endpoints or selectors.

```ts
type MarketOracleWorld = {
  openSelection(condition: 'available' | 'price-changed' | 'suspended' | 'pending-validation'): Promise<void>;
  acknowledgeReviewIfPresent(): Promise<void>;
  releaseValidation(): Promise<void>;
};

test('price change requires explicit review before continuation', async ({ page }, testInfo) => {
  testInfo.annotations.push(
    { type: 'risk', description: 'changed market data must not allow stale continuation' },
    { type: 'oracle', description: 'market-state-visible-actionability-context' },
  );

  const world: MarketOracleWorld = createSyntheticMarketOracleWorld(page);

  await test.step('arrange changed selection in the decision panel', async () => {
    await world.openSelection('price-changed');
    await expect(page.getByRole('region', { name: /decision panel/i })).toBeVisible();
  });

  await test.step('assert changed state and blocked continuation agree', async () => {
    await expect(page.getByRole('status', { name: /price changed/i })).toBeVisible();
    await expect(page.getByRole('button', { name: /continue/i })).toBeDisabled();
    await expect(page.getByRole('region', { name: /decision panel/i })).toContainText(/review/i);
  });

  await test.step('acknowledge review and regain a safe continuation path', async () => {
    await world.acknowledgeReviewIfPresent();
    await expect(page.getByRole('button', { name: /continue/i })).toBeEnabled();
  });
});
```

The adapter can be backed by route interception, a local fixture server, component test fixtures, or a deterministic sandbox. The committed test remains normal Playwright and avoids exposing real product contracts.

## Oracle composition pattern

A durable oracle should combine at least three independent signals:

1. **Visible state:** status, alert, label, or nearby explanatory copy communicates the condition.
2. **Allowed action:** the primary action is enabled, disabled, hidden, or replaced consistently with that condition.
3. **Context preservation:** the affected selection, market, or decision summary remains visible enough for the user to understand what changed.
4. **Evidence annotation:** test output names the synthetic fixture and risk without dumping private network payloads.

When only one signal is asserted, the test can pass while the experience is still unsafe. For example, a disabled button without visible explanation may block the user but fail the product communication requirement.

## Anti-oracles to avoid

- Treating any enabled button as proof that the market is safe.
- Waiting for network idle as a substitute for asserting the visible settled state.
- Asserting private CSS classes, endpoint paths, or payload shapes in public examples.
- Ignoring adjacency: a warning far away from the affected selection may not protect the user.
- Checking only the happy state and never the changed, suspended, pending, or recoverable paths.
- Capturing production traces or screenshots to prove a rule that can be modeled with synthetic fixtures.

## Review checklist

Before adding or approving a market oracle test, confirm:

- The oracle is tied to a named risk such as stale confidence, invalid continuation, duplicate intent, or unclear recovery.
- The fixture world can deterministically create the relevant market condition.
- Assertions combine visible state, actionability, and decision context.
- Step names and annotations would help a reviewer classify a failure from trace/report output.
- The test does not expose proprietary selectors, private routes, raw payloads, credentials, storage state, traces, screenshots, videos, or customer data.
- The oracle is reusable across product areas because it describes behavior, not implementation.

Market state oracles make Playwright suites more credible by turning ambiguous UI checks into explicit safety rules. For Alfabet-style interfaces, the strongest test is one that proves the UI blocks, explains, and preserves context whenever the market truth changes.
