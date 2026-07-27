# Visual Semantics Testing for Alfabet-Style Playwright Suites

Visual semantics testing checks whether an Alfabet-style betting/trading interface communicates the right meaning, priority, and safety state without turning the suite into brittle pixel comparison. The goal is to assert user-facing meaning: unavailable actions look and behave unavailable, changed selections require review, live updates are distinguishable from settled decisions, and responsive layouts preserve decision context.

This document is public-safe. It uses generalized concepts such as market, selection, decision panel, status region, and review-required state. It does not include private selectors, internal URLs, credentials, screenshots, traces, HAR files, customer data, proprietary styling tokens, or implementation details.

## Testing thesis

```text
visual assertion = semantic state + accessible affordance + bounded layout evidence
```

For Alfabet-style products, visual defects are risky when they change interpretation. A failed icon, hidden status, clipped price, or misleading disabled state can affect whether a user trusts a decision. Playwright should therefore verify the visual language around risk, not only the exact pixels of a page.

## What belongs in visual semantics

| Product signal | User risk | Public-safe assertion | Evidence to keep |
|---|---|---|---|
| Market suspended or closed | User thinks a selection is still actionable | Status text is visible and the primary action is blocked | trace step with redacted fixture name |
| Selection changed while in panel | User continues from stale confidence | Review-required message appears near the decision context | role/text assertion and annotation |
| Pending async decision | User repeats input or assumes success | Pending status is visible and repeat input cannot create duplicate intent | status region assertion |
| Responsive panel collapse | User loses context on mobile width | Selection summary and risk message remain reachable by keyboard | viewport-specific step names |
| Error or retry state | User cannot distinguish retry from final failure | Alert explains recoverability and keeps prior context visible | sanitized console/network category |

## Prefer semantic snapshots over pixel snapshots

Pixel screenshots are sometimes useful, but they are expensive to maintain and easy to misuse. Start with assertions that describe meaning:

```ts
await test.step('suspended market communicates blocked action', async () => {
  await world.openMarket({ status: 'suspended' });

  await expect(page.getByRole('status', { name: /market suspended/i })).toBeVisible();
  await expect(page.getByRole('button', { name: /continue/i })).toBeDisabled();
  await expect(page.getByRole('region', { name: /decision panel/i })).toContainText(/review/i);
});
```

This pattern avoids private selectors and styling details while still proving the user cannot safely continue from a misleading visual state.

## When a screenshot is justified

Use a screenshot or visual diff only when the risk cannot be expressed through roles, text, actionability, or layout bounds. Good candidates:

1. **Dense responsive layout:** critical selection context can be clipped or reordered.
2. **Contrast/state distinction:** changed, suspended, and unavailable states must not look identical.
3. **Icon-only affordances:** an icon conveys review, removal, or warning semantics.
4. **Cross-browser rendering:** a browser-specific layout bug changes visible decision context.

Even then, prefer synthetic fixture worlds and scrubbed screenshots. Do not commit production captures, customer information, private branding assets, trace bundles, videos, or HAR files.

## Bounded layout checks

A small geometry assertion can be more stable than a full screenshot when the concern is clipping or overlap:

```ts
async function expectContainedWithin(parent: Locator, child: Locator) {
  const parentBox = await parent.boundingBox();
  const childBox = await child.boundingBox();

  expect(parentBox, 'parent region should be measurable').not.toBeNull();
  expect(childBox, 'child element should be measurable').not.toBeNull();

  expect(childBox!.x).toBeGreaterThanOrEqual(parentBox!.x);
  expect(childBox!.y).toBeGreaterThanOrEqual(parentBox!.y);
  expect(childBox!.x + childBox!.width).toBeLessThanOrEqual(parentBox!.x + parentBox!.width);
  expect(childBox!.y + childBox!.height).toBeLessThanOrEqual(parentBox!.y + parentBox!.height);
}
```

Use this sparingly. Geometry checks should protect a named risk such as clipped status copy, overlapping action controls, or unreachable review information.

## Viewport matrix

A focused visual semantics suite should cover representative layout transitions rather than every device size:

| Viewport class | Primary question | Example invariant |
|---|---|---|
| Narrow mobile | Is decision context preserved when panels collapse? | Selection summary, status, and primary action remain in a logical keyboard path |
| Tablet / small desktop | Do side panels and content columns avoid overlap? | Market list and decision panel expose consistent status semantics |
| Desktop | Does density hide risk states? | Changed or suspended status remains adjacent to the affected selection |

Name viewport steps explicitly in reports:

```text
mobile: preserve review-required status in collapsed panel
tablet: prevent market list and decision panel overlap
desktop: keep changed selection status adjacent to selection row
```

## Anti-patterns

- Asserting exact CSS class names or proprietary design tokens in public examples.
- Using screenshots as a substitute for roles, labels, status regions, and actionability checks.
- Capturing real production screens or trace artifacts.
- Approving broad visual baselines without naming the product risk they protect.
- Hiding unstable animation or timing problems behind arbitrary waits.

## Review checklist

Before adding a visual semantics test, confirm:

- The test protects a named interpretation risk, not a cosmetic preference.
- Assertions use accessible roles, visible text, actionability, or bounded geometry before screenshots.
- Screenshots, if used, are synthetic, scrubbed, and not committed with private data.
- The scenario includes the viewport or layout condition that makes the risk plausible.
- Failure output explains the semantic mismatch a reviewer should investigate.

Visual semantics coverage should make the suite better at detecting misleading states. For Alfabet-style interfaces, the most important visual question is not “did the page match an image?” but “did the UI communicate the safe next action truthfully?”
