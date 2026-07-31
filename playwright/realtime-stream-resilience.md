# Realtime Stream Resilience for Alfabet-Style Playwright Suites

Realtime stream resilience testing checks whether an Alfabet-style betting/trading UI remains safe when live updates are delayed, duplicated, reordered, disconnected, or resumed from a snapshot. These scenarios matter because a page can look active while the decision context is no longer current enough for a user to continue confidently.

This guide is public-safe. It uses generalized concepts such as market stream, snapshot, sequence number, selection, decision panel, stale banner, reconnect, and recovery. It does not include private WebSocket URLs, endpoint paths, selectors, payloads, credentials, storage state, screenshots, traces, HAR files, customer data, or proprietary implementation details.

## Testing thesis

```text
live UI safety = ordered evidence + freshness boundary + visible recovery + gated action
```

A realtime Playwright test should prove the product rule a user depends on, not the private shape of the transport. For Alfabet-style interfaces, the core rule is:

```text
if live market truth cannot be proven fresh and ordered, the UI must label uncertainty and block unsafe continuation
```

## Stream failure modes to model

| Stream condition | User risk | Public-safe invariant | Playwright signal |
|---|---|---|---|
| Delayed update | User continues from stale price or status | Freshness boundary is visible before continuation is allowed | controlled stream hold, stale/status region, disabled action |
| Duplicate update | UI may repeat alerts or duplicate a decision intent | Replayed event is idempotent in visible state | repeated synthetic event, one visible warning/selection |
| Out-of-order update | Older market truth may overwrite newer truth | Lower-sequence event cannot regress the visible market state | sequence-aware fixture, unchanged latest status |
| Stream disconnect | User may believe the market is still live | Disconnected/stale state is communicated and primary action is gated | disconnect trigger, alert/status, actionability check |
| Snapshot resume | Reconnect may restore partial or contradictory state | Resume reconciles snapshot and stream into one coherent decision context | reconnect step, visible context preservation |
| Stream-to-navigation race | User navigates while updates are pending | Destination reconstructs safe state from latest accepted truth | navigation, restored warning/context, no unsafe enabled action |

## Public-safe harness sketch

Keep transport mechanics behind a synthetic stream adapter. The committed Playwright test should describe ordered user-visible truth, not private message formats.

```ts
type RealtimeMarketWorld = {
  openLiveMarket(): Promise<void>;
  holdStream(label: string): Promise<void>;
  emitMarketUpdate(update: 'available' | 'price-changed' | 'suspended', options?: { sequence?: number }): Promise<void>;
  disconnectStream(): Promise<void>;
  resumeFromSnapshot(snapshot: 'available' | 'review-required' | 'suspended'): Promise<void>;
};

test('out-of-order market update cannot reopen a suspended selection', async ({ page }, testInfo) => {
  testInfo.annotations.push(
    { type: 'risk', description: 'older live data must not restore unsafe continuation' },
    { type: 'fixture', description: 'synthetic-realtime-market-world' },
  );

  const world: RealtimeMarketWorld = createSyntheticRealtimeMarketWorld(page);

  await test.step('open a deterministic live market', async () => {
    await world.openLiveMarket();
    await expect(page.getByRole('region', { name: /decision panel/i })).toBeVisible();
  });

  await test.step('accept the newer suspended state', async () => {
    await world.emitMarketUpdate('suspended', { sequence: 42 });
    await expect(page.getByRole('status', { name: /suspended/i })).toBeVisible();
    await expect(page.getByRole('button', { name: /continue/i })).toBeDisabled();
  });

  await test.step('deliver an older available update', async () => {
    await world.emitMarketUpdate('available', { sequence: 41 });
  });

  await test.step('prove the UI did not regress to unsafe availability', async () => {
    await expect(page.getByRole('status', { name: /suspended/i })).toBeVisible();
    await expect(page.getByRole('button', { name: /continue/i })).toBeDisabled();
    await expect(page.getByRole('region', { name: /decision panel/i })).toContainText(/review|closed|suspended/i);
  });
});
```

The adapter can be implemented with a local fixture server, browser-side event bridge, route-backed polling simulation, component fixture, or a test-only stream broker. The important boundary is that public examples name behavior and invariants instead of publishing real transport contracts.

## Assertion strategy

A credible realtime test should combine these signals:

1. **Freshness signal:** a visible status, timestamp label, stale banner, or review message communicates whether live truth is current enough.
2. **Ordering rule:** older or duplicate events do not overwrite newer user-visible safety state.
3. **Actionability gate:** money-impacting or commitment-like actions are disabled, hidden, or replaced while the stream is stale, disconnected, or reconciling.
4. **Context preservation:** the affected market, selection, or decision summary remains visible so the user understands what changed.
5. **Recovery evidence:** reconnect, retry, or snapshot resume creates one coherent state rather than contradictory panels.
6. **Sanitized annotation:** traces and reports include risk and fixture names without raw stream payloads.

Avoid sleeping for arbitrary milliseconds to infer realtime behavior. Prefer explicit fixture controls such as `holdStream`, `emitMarketUpdate`, `disconnectStream`, and `resumeFromSnapshot` so the failure evidence is deterministic.

## Stream resilience test matrix

| Scenario | Arrange | Act | Expected evidence |
|---|---|---|---|
| Stale while held | Open live market and hold updates | Attempt to continue after freshness boundary | Stale/reconnecting status appears and primary action is blocked |
| Duplicate price change | Emit the same changed update twice | Inspect decision panel | One review-required state, no duplicate warning stack |
| Out-of-order suspend | Emit suspended at sequence N, then available at N-1 | Inspect visible status and action | Suspended state remains authoritative |
| Disconnect during pending decision | Start a pending decision, then disconnect stream | Attempt repeat input | Pending/disconnected uncertainty is visible and idempotent |
| Resume from snapshot | Disconnect after a change, resume from synthetic snapshot | Review decision panel | Snapshot and visible state agree; no contradictory continuation |
| Navigate during reconnect | Move away and return while stream resumes | Inspect reconstructed page | Latest safe state is restored with context preserved |

## Anti-patterns

- Treating an open WebSocket as proof that market data is fresh enough for action.
- Asserting raw private event names, endpoint paths, payload fields, or sequence formats in public examples.
- Using `waitForTimeout` to guess that a stream has settled.
- Allowing older events to re-enable a primary action without an explicit review path.
- Capturing production traces, videos, screenshots, HAR files, or storage state to demonstrate a rule that can be modeled synthetically.
- Verifying only reconnect success while ignoring what the user can safely do during reconnect.

## Review checklist

Before adding a realtime resilience test, confirm:

- The scenario protects a named risk such as stale confidence, out-of-order market truth, duplicate intent, or unsafe recovery.
- Stream behavior is deterministic and controlled by a public-safe fixture or adapter.
- Assertions combine visible freshness, actionability, ordering, and preserved context.
- `test.step` names make the trace readable without exposing internal transport details.
- The test does not commit secrets, endpoints, payload dumps, traces, screenshots, videos, HAR files, storage state, customer data, or proprietary selectors.
- Failure output would help a reviewer decide whether the bug is transport ordering, UI reconciliation, action gating, or observability.

Realtime stream resilience turns live-update uncertainty into reviewable Playwright evidence. In Alfabet-style interfaces, the safest realtime UI is one that can say “not fresh enough yet,” preserve the decision context, and refuse unsafe continuation until ordered truth is restored.
