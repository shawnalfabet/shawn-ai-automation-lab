# Latency Resilience Testing for Alfabet-Style Playwright Suites

Latency resilience testing checks whether an Alfabet-style betting/trading interface remains truthful, safe, and recoverable when async work is slow, retried, interrupted, or partially stale. These tests are deeper than generic loading-spinner checks: they protect decision points where timing can change whether a user should continue.

This document is public-safe. It uses generalized UI concepts such as market, selection, decision panel, validation, pending intent, and recovery. It does not include private URLs, proprietary selectors, credentials, storage state, screenshots, traces, HAR files, customer data, or internal API contracts.

## Design goal

A latency scenario should prove one stable product truth:

```text
when the system is uncertain, the UI must not imply certainty or allow unsafe continuation
```

For Playwright, that means a test should intentionally control timing, drive a user-relevant transition, and assert visible semantics rather than private implementation details.

## Failure modes worth modeling

| Failure mode | User risk | Public-safe invariant | Playwright signal |
|---|---|---|---|
| Slow decision response | A user may click repeatedly or believe the action failed | Only one pending intent is represented, and repeat input cannot create a second visible intent | route delay, disabled/actionable state, visible pending copy |
| Stale market transition | A previously valid selection may no longer be valid | Changed, suspended, or closed state gates continuation until review or recovery | synthetic transition, status region, primary action state |
| Partial panel refresh | Some UI regions update while others still show old information | Mixed state is labeled as loading/reviewing instead of actionable | landmark assertions, status annotations, no contradictory affordances |
| Retry after recoverable error | Retry may duplicate action or clear important context | Retry keeps user context and emits one clear recovery path | retry control, preserved selection summary, redacted console/network category |
| Navigation during pending state | Back/refresh/deep link may resurrect an unsafe action | Return path reconstructs safe pending/review/retry state, never silent success | navigation step, visible reconstruction, no enabled unsafe action |

## Harness pattern

Keep latency control behind a public-safe adapter so tests describe behavior, not private endpoints.

```ts
type LatencyWorld = {
  openMarket(): Promise<void>;
  delayDecisionResponse(label: string): Promise<void>;
  releaseDecisionResponse(label: string): Promise<void>;
  moveSelectionToReviewRequired(): Promise<void>;
};

test('pending decision cannot create duplicate intent', async ({ page }, testInfo) => {
  testInfo.annotations.push(
    { type: 'risk', description: 'latency must not duplicate a money-impacting intent' },
    { type: 'fixture', description: 'synthetic-latency-world' },
  );

  const world: LatencyWorld = createSyntheticLatencyWorld(page);

  await test.step('open a deterministic market state', async () => {
    await world.openMarket();
    await expect(page.getByRole('region', { name: /decision panel/i })).toBeVisible();
  });

  await test.step('hold the decision response open', async () => {
    await world.delayDecisionResponse('primary-decision');
    await page.getByRole('button', { name: /continue/i }).click();
  });

  await test.step('repeat user input while pending', async () => {
    await expect(page.getByRole('button', { name: /continue/i })).toBeDisabled();
    await page.keyboard.press('Enter');
    await expect(page.getByRole('status', { name: /pending decision/i })).toBeVisible();
  });

  await test.step('release and verify a single resolved intent', async () => {
    await world.releaseDecisionResponse('primary-decision');
    await expect(page.getByText(/decision received/i)).toBeVisible();
  });
});
```

The adapter can be implemented with Playwright routing, a local fixture server, or an in-process mock boundary. The committed test should still read like ordinary Playwright and should not expose private request paths or payloads.

## Assertion strategy

Prefer assertions that tell a reviewer what the user could safely do:

1. **Actionability:** the primary action is disabled, hidden, or replaced while the system is uncertain.
2. **Visible reason:** a status, alert, or review message explains the pending/retry/review state.
3. **Input idempotence:** repeated pointer or keyboard input does not create duplicate visible intent.
4. **State reconstruction:** refresh, back, or retry returns to a safe state with context preserved.
5. **Evidence:** the test records sanitized annotations for risk, fixture profile, and latency phase.

Avoid relying on exact milliseconds unless timing itself is the product contract. Use controlled release points and semantic expectations instead of sleeps.

## Latency phases to name in `test.step`

Use consistent step names so traces and reports are easy to scan:

```text
arrange synthetic available state
hold primary decision response
attempt repeat input during pending state
move selection to review-required while pending
release response and assert safe reconstruction
retry after recoverable error
```

These names make a failure report useful without revealing implementation details.

## Review checklist

Before merging a latency resilience test, confirm:

- The scenario protects a named user or product risk, not just a spinner.
- The fixture world is deterministic and replayable in CI.
- The test does not commit traces, screenshots, videos, HAR files, storage state, or payload dumps.
- Assertions use accessible roles, labels, status regions, or public adapter methods.
- Repeat input is tested through at least one realistic channel: pointer, keyboard, refresh, or retry.
- Failure output includes sanitized risk and fixture annotations.

Latency resilience coverage should make the suite more trustworthy by proving that uncertainty is represented honestly. For Alfabet-style interfaces, the safest UI is often the one that blocks, explains, and recovers rather than continuing from stale confidence.
