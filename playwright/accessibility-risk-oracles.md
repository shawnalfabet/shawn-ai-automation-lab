# Accessibility Risk Oracles for Alfabet-Style Playwright Suites

Accessibility assertions in an Alfabet-style betting/trading UI are not only compliance checks. They are risk oracles: if status, disabled state, validation, freshness, or receipt semantics are not exposed accessibly, a user or a test may interpret a money-impacting state incorrectly.

This guide is public-safe. It uses generalized concepts such as market, selection, decision panel, stake input, quote state, status region, receipt, and recovery. It does not include private selectors, internal URLs, payloads, credentials, screenshots, traces, HAR files, customer data, or proprietary implementation details.

## Testing thesis

```text
accessible semantics = user-interpretable state + durable Playwright signal
```

A credible Playwright suite should use accessibility semantics to prove that critical UI states are both perceivable and machine-checkable. For Alfabet-style interfaces, the central rule is:

```text
if a state changes what the user may safely do, that state must be exposed through visible and accessible semantics
```

This makes accessibility an engineering control, not a separate audit afterthought.

## Risk oracles worth modeling

| Oracle | User risk | Public-safe invariant | Playwright signal |
|---|---|---|---|
| Market status semantics | A suspended, closed, or stale market appears actionable | Status is announced or labelled before the primary action can continue | `getByRole('status')`, scoped region text, disabled action |
| Stake validation semantics | Invalid input looks accepted or error copy is detached from the field | The input exposes invalid state and points to actionable guidance | accessible name, `aria-invalid`, error relationship, visible message |
| Pending decision semantics | Repeated clicks or keyboard submits create duplicate intent | Pending state is represented once and the action is unavailable | disabled button, live status, single receipt/pending region |
| Freshness semantics | A stale quote is read as current | Freshness or review-required state is discoverable near the decision | status/alert region, timestamp label, action gate |
| Receipt semantics | A generic toast hides whether the intended action was accepted, declined, or pending | Terminal state is tied to the synthetic context and exposed as a receipt | named receipt region, status category, scoped assertions |
| Recovery semantics | Reconnect, refresh, or retry returns contradictory UI | Recovery state explains what changed and what the user can do next | alert/status region, preserved context, gated primary action |

## Harness sketch

Keep selectors semantic and product-neutral. The test should read as a user-state proof rather than a DOM implementation probe.

```ts
import { expect, test } from '@playwright/test';

type AccessibleMarketWorld = {
  openAvailableMarket(): Promise<void>;
  moveMarketToSuspended(): Promise<void>;
  typeStake(value: string): Promise<void>;
};

test('suspended market exposes accessible status and blocks continuation', async ({ page }, testInfo) => {
  testInfo.annotations.push(
    { type: 'risk', description: 'suspended market must not remain semantically actionable' },
    { type: 'oracle', description: 'accessibility-risk-oracle' },
  );

  const world: AccessibleMarketWorld = createSyntheticAccessibleMarketWorld(page);

  await test.step('open a deterministic available market', async () => {
    await world.openAvailableMarket();
    await expect(page.getByRole('region', { name: /decision panel/i })).toBeVisible();
    await expect(page.getByRole('button', { name: /continue/i })).toBeEnabled();
  });

  await test.step('transition to suspended state', async () => {
    await world.moveMarketToSuspended();
  });

  await test.step('assert visible and accessible safety semantics', async () => {
    const panel = page.getByRole('region', { name: /decision panel/i });
    await expect(panel.getByRole('status', { name: /market status/i })).toContainText(/suspended|closed|review/i);
    await expect(panel.getByRole('button', { name: /continue/i })).toBeDisabled();
    await expect(panel).toContainText(/selection.*unavailable|review required|market closed/i);
  });
});
```

The adapter can use a component fixture, local fixture server, or route-backed synthetic world. The important constraint is that the public example verifies semantics without exposing private DOM structure or production data.

## Assertion strategy

### 1. Scope every assertion to a decision region

Money-impacting UI often has repeated labels: multiple markets, selections, buttons, and summaries. Use a named region first, then assert inside it.

```ts
const decisionPanel = page.getByRole('region', { name: /decision panel/i });
await expect(decisionPanel.getByRole('button', { name: /continue/i })).toBeDisabled();
```

This prevents a passing assertion against the wrong repeated control.

### 2. Pair actionability with explanation

A disabled button alone is weak evidence. A status message alone is also weak evidence. For safety states, assert both:

```text
primary action is gated + the reason is perceivable near the decision
```

That pairing helps catch regressions where a visual change appears but keyboard or screen-reader users do not receive the same state.

### 3. Treat validation as a field contract

For stake or quantity inputs, a good validation oracle checks:

- the invalid value remains visible or is intentionally normalized;
- the field exposes invalid semantics;
- the error message is associated with the field or placed in a nearby alert/status region;
- the primary action remains gated until the value is valid or reviewed.

Avoid relying only on color, border style, animation, or a screenshot diff for validation state.

### 4. Prefer status categories over private copy

Exact product copy changes often. Public-safe tests can assert status categories such as `available`, `review-required`, `suspended`, `pending`, `declined`, or `accepted` through accessible regions, stable labels, or sanitized fixture metadata. Copy assertions should focus on meaning, not private wording.

### 5. Use accessibility snapshots as review aids, not golden truth

Playwright accessibility snapshots or aria snapshots can help debug semantic regressions, but they should not become broad brittle goldens for a whole page. Capture or compare a small decision region when the semantics themselves are the contract.

## Accessibility risk matrix

| Scenario | Arrange | Act | Expected evidence |
|---|---|---|---|
| Suspended selection | Open available synthetic market | Move selection to suspended | Status announces suspended/review state and continue is disabled |
| Invalid stake | Open decision panel | Enter invalid stake class | Input exposes invalid state, guidance is visible, submit is gated |
| Pending submit | Start single synthetic intent | Press click and keyboard submit while pending | One pending status, no duplicate receipt, primary action unavailable |
| Quote freshness expired | Hold quote beyond freshness boundary | Attempt continuation | Freshness/review status appears before action can proceed |
| Recover after reconnect | Disconnect and resume from synthetic snapshot | Inspect decision panel | Recovery explanation is visible and context is preserved or safely reset |

## Anti-patterns

- Testing only CSS classes for disabled, error, stale, or pending states.
- Checking that a toast appeared without tying it to the decision panel or receipt context.
- Using unscoped `getByText` assertions on pages with repeated markets or selections.
- Treating a successful mouse click as proof the keyboard or assistive-technology path is safe.
- Publishing accessibility snapshots that include private product copy, customer data, internal labels, or proprietary state names.

## Review checklist

Before adding an accessibility-risk oracle, confirm:

- The scenario protects a named safety risk, not only a generic accessibility rule.
- Assertions combine semantic discoverability, visible explanation, and actionability.
- Locators use accessible roles, names, labels, and scoped regions where possible.
- Any snapshots, attachments, or annotations are synthetic and public-safe.
- Failure output would help a reviewer distinguish a semantic regression from a visual-only regression.

Accessibility-focused Playwright tests are strongest when they make the same safety contract legible to users, reviewers, and automation. In Alfabet-style products, semantic correctness is part of transactional trust.
