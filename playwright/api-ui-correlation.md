# API/UI Correlation Strategy for Alfabet Playwright Tests

Public Playwright suites for Alfabet-style betting and trading products should not expose private API routes, payloads, selectors, or production evidence. They can still demonstrate a strong strategy: correlate user-visible behavior with sanitized network observations so failures explain whether the UI, data feed, validation layer, or rendering contract drifted.

This note describes a public-safe pattern for treating network traffic as test evidence without turning E2E tests into brittle private API assertions.

## Why correlate UI and observed contracts?

Betting and trading interfaces are highly stateful. A visible price, suspended market, disabled action, or validation warning often depends on multiple asynchronous updates. A UI-only assertion can prove what the user saw, but not why the product reached that state. A raw API assertion can prove a response shape, but not whether the customer experience honored it.

A good Playwright test records both sides:

- **Observed contract:** sanitized facts emitted by route, websocket, or polling responses.
- **User surface:** text, roles, enabled states, banners, and betslip affordances visible in the browser.
- **Correlation rule:** a small invariant connecting the two, such as "a suspended selection cannot remain actionable".

The artifact should be useful in a failure report while remaining safe to publish.

## Public-safe boundaries

Do not commit or publish:

- real endpoint paths, hostnames, tokens, cookies, or environment variables;
- raw request/response bodies from private systems;
- customer, account, wallet, ticket, or trading data;
- screenshots, videos, traces, HAR files, or storage state from private environments;
- proprietary selectors or product-specific DOM structure.

Do publish:

- generalized event names such as `marketSnapshot`, `priceUpdate`, or `validationResult`;
- redacted payload sketches with fake identifiers;
- correlation heuristics that explain the testing idea;
- reusable adapters that accept caller-provided sanitizers.

## Correlation layers

| Layer | What the test observes | What the UI proves | Example invariant |
| --- | --- | --- | --- |
| Navigation readiness | Document, route, and critical data completion | The target area is usable, not merely loaded | A page is not ready until the shell and first market snapshot are both represented on screen |
| Market status | Sanitized active/suspended/closed statuses | Disabled actions, labels, and recovery messages | Suspended selections are visible but not actionable |
| Price movement | Direction and timestamp of price updates | Price text changes, movement hints, stale-state handling | A stale price must either refresh or show an explicit stale indicator before submission |
| Betslip validation | Public-safe validation category | User-facing error, warning, or disabled submit state | Rejected input is explained before the user can submit again |
| Recovery path | Retry/fallback event category | User can reattempt or navigate safely | A transient feed failure does not leave the UI in a silently disabled state |

## Harness sketch

The test should not assert private routes directly. Instead, collect sanitized observations through an adapter owned by the test harness.

```ts
import { expect, type Page, test } from '@playwright/test';

type MarketObservation = {
  kind: 'marketSnapshot' | 'priceUpdate' | 'validationResult';
  marketAlias: string;
  status?: 'active' | 'suspended' | 'closed';
  priceDirection?: 'up' | 'down' | 'unchanged';
  validationCategory?: 'accepted' | 'rejected' | 'stale' | 'unavailable';
  observedAt: number;
};

async function observePublicSafeMarketEvents(page: Page) {
  const events: MarketObservation[] = [];

  page.on('response', async response => {
    const sanitized = await sanitizeIfKnownMarketEvent(response);
    if (sanitized) events.push(sanitized);
  });

  return {
    events,
    latestFor(alias: string) {
      return events.filter(event => event.marketAlias === alias).at(-1);
    },
  };
}

async function sanitizeIfKnownMarketEvent(response: unknown): Promise<MarketObservation | null> {
  // Repository examples intentionally omit private URL matching and payload parsing.
  // Real projects inject a sanitizer that maps internal traffic to public-safe facts.
  return null;
}

test('suspended selection is represented as non-actionable UI', async ({ page }) => {
  const observations = await observePublicSafeMarketEvents(page);

  await page.goto('/public-safe-demo/markets');

  const selection = page.getByRole('button', { name: /example selection/i });
  await expect(selection).toBeVisible();

  const latest = observations.latestFor('example-market');
  test.info().annotations.push({
    type: 'market-observation',
    description: latest ? `${latest.kind}:${latest.status ?? 'unknown'}` : 'none-recorded',
  });

  if (latest?.status === 'suspended') {
    await expect(selection).toBeDisabled();
    await expect(page.getByText(/temporarily unavailable|suspended/i)).toBeVisible();
  }
});
```

The example is intentionally incomplete: the missing piece is the private sanitizer, which belongs in the consuming project and should never be copied into this public lab.

## Correlation rules worth testing

### 1. Readiness requires both UI shell and data evidence

Avoid `networkidle` as a blanket readiness signal for live betting/trading surfaces. A page can keep streaming after it is ready, or appear idle before the critical data has rendered. Prefer a domain readiness rule:

1. route has reached the intended public-safe area;
2. key landmark or heading is visible;
3. at least one sanitized domain event has been observed;
4. the UI representation of that event is visible and stable long enough for interaction.

### 2. Disabled states must have an explainable cause

For money-impacting controls, disabled is not enough. A correlated assertion should ask:

- Did the observed status make the action unsafe?
- Does the UI explain why the action is unavailable?
- Does the control recover when a later active event arrives?

This catches silent dead ends where the button state changes but the user gets no reason or recovery path.

### 3. Stale data needs a user-visible policy

When the observed event timestamp is older than the freshness budget, the UI should do one of three things:

- refresh and display the new value;
- mark the value as stale or unavailable;
- prevent submission until validation completes.

The test should not require a private timeout value in public docs. It can model the expectation as a named policy such as `freshnessBudget.marketPrice` in the real harness.

### 4. Validation failures should align with visible guidance

If an observed validation category is `rejected`, the UI should expose an actionable message and preserve safe user recovery. The assertion should focus on categories rather than raw backend messages:

- `stale`: refresh or reselect;
- `unavailable`: market or selection no longer accepts action;
- `limit`: entered value is outside the allowed range;
- `accepted`: submission can proceed or confirmation is shown.

## Failure report checklist

A useful correlated Playwright failure should answer:

- What public-safe event category was last observed?
- What did the user-visible surface show at the same time?
- Was the mismatch likely data freshness, rendering, validation, or interaction timing?
- Did the test attach sanitized annotations rather than raw private traffic?
- What next diagnostic artifact is needed: trace, console log, network summary, or product rule review?

## Anti-patterns

- Asserting exact private endpoint paths in E2E tests when a public-safe semantic event would be more stable.
- Copying raw network payloads into fixtures or failure comments.
- Treating HTTP success as proof that the UI is correct.
- Treating visible text as proof that the underlying market state is current.
- Adding broad waits instead of correlating readiness to a specific observed event.

## Practical adoption path

1. Start with one money-impacting flow and define three sanitized event categories.
2. Add a tiny observation adapter that records only public-safe facts.
3. Attach observations as Playwright annotations on failure.
4. Write one invariant that pairs an observed status with a user-visible affordance.
5. Expand only after the failure output is understandable to engineers, QA, and product stakeholders.

The result is not an API test hidden inside a browser test. It is a browser test with enough domain evidence to explain high-risk UI behavior without exposing private implementation details.
