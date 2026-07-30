# Temporal Consistency Testing for Alfabet-Style Playwright Suites

Temporal consistency tests verify that an Alfabet-style betting/trading UI tells a coherent story as time passes: countdowns expire, market freshness changes, async validation settles, and user actions remain safe when the world moves underneath the page.

This guide is public-safe. It uses generalized terms such as market, selection, quote, decision panel, freshness indicator, and validation result. It does not include private selectors, internal URLs, production payloads, credentials, browser storage state, screenshots, traces, HAR files, customer data, or proprietary implementation details.

## Why time deserves its own test layer

Many UI bugs are not wrong at first render. They become wrong after a delay, refresh, timeout, retry, tab switch, or race between user intent and market updates.

```text
initial truth + elapsed time + async evidence + user intent = safe or unsafe continuation
```

For Alfabet-style interfaces, time-related defects can create high-impact risks:

- a price or availability indicator looks fresh after its freshness window has expired;
- a primary action remains enabled while validation is unresolved;
- a countdown reaches zero but the page still presents the previous market state;
- a retry path duplicates an unresolved intent;
- a recovered page loses the decision context that explains what changed.

## Temporal risk matrix

| Temporal condition | User risk | Playwright oracle | Public-safe evidence |
|---|---|---|---|
| Fresh quote within window | User can make a decision from current information | Freshness indicator, selection summary, and actionability agree | scoped role/text assertions, fixture clock label |
| Quote expires | User may continue from stale confidence | Expired/stale state is visible and continuation is blocked or requires refresh/review | status region, disabled action, test annotation |
| Validation pending | User may double-submit or infer success | Pending state blocks repeat intent and eventually resolves deterministically | controlled fixture latency, actionability checks |
| Market update during review | User may acknowledge the wrong version | Review copy identifies the changed selection and preserves context | decision panel assertions, sanitized update category |
| Retry after timeout | User may create duplicate intent | Retry explains unresolved prior state and remains idempotent | retry affordance, no second visible intent |
| Restore after navigation | User may return to stale or orphaned context | Restored state either refreshes safely or demands explicit review | navigation step, visible freshness rule |

## Deterministic clock pattern

Avoid tests that sleep for real time whenever the behavior can be controlled through a synthetic fixture world. The test should describe the product rule while the adapter hides how time is advanced.

```ts
type TemporalMarketWorld = {
  openSelectionWithFreshQuote(): Promise<void>;
  advanceFreshnessWindow(ms: number): Promise<void>;
  resolveValidationAs(result: 'accepted' | 'changed' | 'timeout'): Promise<void>;
};

test('expired quote blocks continuation until the user reviews freshness', async ({ page }, testInfo) => {
  testInfo.annotations.push(
    { type: 'risk', description: 'stale market information must not allow confident continuation' },
    { type: 'time-control', description: 'synthetic freshness window' },
  );

  const world: TemporalMarketWorld = createTemporalMarketWorld(page);

  await test.step('arrange a fresh selection', async () => {
    await world.openSelectionWithFreshQuote();
    await expect(page.getByRole('region', { name: /decision panel/i })).toBeVisible();
    await expect(page.getByRole('status', { name: /fresh/i })).toBeVisible();
    await expect(page.getByRole('button', { name: /continue/i })).toBeEnabled();
  });

  await test.step('advance beyond the freshness window', async () => {
    await world.advanceFreshnessWindow(30_000);
    await expect(page.getByRole('status', { name: /stale|expired/i })).toBeVisible();
    await expect(page.getByRole('button', { name: /continue/i })).toBeDisabled();
  });
});
```

The committed test stays ordinary Playwright. The fixture may use route interception, fake timers, a local sandbox, component harness state, or a test-only adapter, but the public artifact should not reveal private API contracts.

## Assertions that make time visible

Temporal tests are strongest when they assert more than elapsed milliseconds:

1. **Clock or freshness signal:** the UI exposes a visible status such as fresh, stale, validating, expired, retrying, or review required.
2. **Actionability alignment:** primary actions match the visible time state; pending, stale, or uncertain states should not permit confident continuation.
3. **Context preservation:** affected selection, quote, or market summary remains visible when time changes the user decision.
4. **Resolution evidence:** the test records which synthetic condition was advanced or resolved without dumping raw network data.
5. **Recovery path:** if the user can refresh, retry, or acknowledge, the follow-up state is asserted too.

## Anti-patterns

- Waiting with `page.waitForTimeout()` and hoping the environment is fast enough.
- Treating `networkidle` as proof that validation, freshness, or recovery is semantically complete.
- Asserting only a countdown number without checking the action it controls.
- Allowing a stale state to be visually subtle while the primary action remains prominent.
- Replaying production traces, HAR files, screenshots, or payloads in a public repository.
- Testing only the happy fresh path and never the expired, pending, changed, timeout, or recovery paths.

## Review checklist

Before adding a temporal consistency test, confirm:

- The protected risk is named: stale confidence, duplicate intent, unresolved validation, unsafe retry, or lost context.
- Time is controlled deterministically by a public-safe fixture or harness.
- Assertions cover visible state, actionability, and decision context.
- The test avoids fixed sleeps unless it is explicitly validating a UI delay boundary and remains bounded.
- Trace/report annotations explain the synthetic temporal condition without exposing private implementation details.
- The recovery path after expiration, timeout, or update is asserted, not just the failure state.

Temporal consistency turns Playwright from a page-load verifier into a safety harness for moving market truth. In Alfabet-style products, credible E2E coverage must prove not only that the UI is correct now, but that it stays honest when time changes the decision.
