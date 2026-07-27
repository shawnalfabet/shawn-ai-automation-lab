# Locator Contract Strategy for Alfabet-Style Playwright Suites

Locator contracts define how tests find user-facing behavior without binding the suite to private DOM structure. For an Alfabet-style betting/trading product, this matters because the highest-value tests often touch fast-moving UI: market lists, selection rows, decision panels, validation states, pending actions, and recovery paths. A weak locator strategy turns those tests into noise; a strong one makes failures explain whether the product behavior changed or the test contract was broken.

This document is public-safe. It uses generalized terms such as market, selection, decision panel, status, review, and recovery. It does not include proprietary selectors, internal URLs, credentials, screenshots, traces, HAR files, customer data, or implementation details.

## Principle

```text
locator = public user meaning + stable test contract + clear ownership
```

A Playwright locator should answer three questions for a reviewer:

1. **What user meaning is being exercised?** Prefer role, accessible name, visible text, and semantic regions before structural CSS.
2. **What stability contract does the application intentionally provide?** Use test ids only when they are named as product-facing test contracts, not as leaked implementation paths.
3. **Who owns a break?** A locator failure should point to either an intentional UX change, a missing accessibility contract, or a test contract that needs versioning.

## Locator hierarchy

| Preference | Locator style | Good fit | Risk if overused |
|---:|---|---|---|
| 1 | `getByRole` with accessible name | Buttons, status regions, alerts, navigation, tabs, dialogs | Requires accessible names to be intentional and reviewed |
| 2 | `getByLabel`, `getByPlaceholder`, `getByText` | Form controls, validation copy, user-visible recovery messages | Brittle if copy changes frequently without UX intent |
| 3 | Semantic container then role/text within it | Decision panel, market card, selection summary | Needs stable region names and careful scoping |
| 4 | `getByTestId` for explicit contracts | Repeated rows, virtualized lists, synthetic fixture handles | Can become private DOM shorthand if naming is careless |
| 5 | CSS/XPath | Last resort for public-safe harness internals | Tends to encode implementation and raise flake cost |

The default should be semantic. Test ids are still valuable when they describe a durable behavior contract, especially for repeated market rows where visible names can collide.

## Contract naming rules

Use names that describe the user-observable concept, not the component tree.

| Avoid | Prefer | Why |
|---|---|---|
| `div-odds-cell-3` | `selection-price` | The test cares about price meaning, not DOM shape |
| `BetSlipFooterSubmitButton` | `decision-panel-primary-action` | The contract survives refactors and layout changes |
| `marketItem_42` | `market-row` scoped by visible market name | The id is not tied to fixture data or private identifiers |
| `modal-error-x` | `recovery-dialog-dismiss` | The action and state are clear to a reviewer |

For Alfabet-style flows, useful public-safe contract terms include `market-row`, `selection-price`, `selection-status`, `decision-panel`, `review-required-message`, `pending-decision-status`, `primary-action`, `recovery-alert`, and `selection-summary`.

## Scoping pattern

Repeated betting/trading UI often contains many similar labels: multiple markets, selections, prices, and actions. Scope locators through a visible semantic parent before interacting with a child.

```ts
const market = page
  .getByRole('region', { name: /match result market/i })
  .filter({ hasText: /home team/i });

await expect(market.getByTestId('selection-status')).toHaveText(/available/i);
await market.getByRole('button', { name: /add home team/i }).click();

const decisionPanel = page.getByRole('region', { name: /decision panel/i });
await expect(decisionPanel.getByTestId('selection-summary')).toContainText(/home team/i);
await expect(decisionPanel.getByRole('button', { name: /continue/i })).toBeEnabled();
```

This keeps the test public-safe and reviewable: the scenario says which region, which selection meaning, and which decision affordance it expects. It does not expose private selectors or endpoint-derived identifiers.

## When to add a test id

Add or request a test id when all of these are true:

- the element represents a stable user concept rather than a styling wrapper;
- role/name/text alone cannot disambiguate repeated instances reliably;
- the id can be named without private business identifiers;
- the id will be treated as a compatibility contract for tests;
- the failure message will help diagnose product behavior rather than implementation shape.

Do not add a test id merely to avoid improving accessibility. If an element is a button, alert, dialog, status, tab, or form control, the better first fix is usually a clear accessible role and name.

## Anti-patterns

1. **DOM archaeology:** `page.locator('div > div:nth-child(4) button')` proves knowledge of markup, not behavior.
2. **Private vocabulary leakage:** IDs copied from internal API fields, trading codes, or environment names should not appear in public examples.
3. **Unscoped text clicks:** `page.getByText('Continue').click()` can hit the wrong region when multiple panels or dialogs are visible.
4. **Fixture-coupled ids:** `selection-12345` makes synthetic examples look like production data and prevents deterministic reuse.
5. **Hidden assertion targets:** asserting on non-visible technical fields misses whether the user saw the right state.

## Review checklist

Before committing a Playwright locator for a money-impacting or trust-sensitive flow, ask:

- Does the locator describe user meaning rather than component structure?
- Is the search scoped to the smallest stable semantic region?
- Would a UX copy change, accessibility change, or test-contract change explain a failure clearly?
- Does the test avoid private selectors, internal URLs, credentials, storage state, screenshots, traces, HAR files, and customer data?
- Are repeated market/selection cases deterministic in the synthetic fixture world?
- Does the failure message tell whether the unsafe action was visible, enabled, blocked, pending, or recoverable?

## Practical standard

For this lab, public examples should use this order:

```text
role/name first → scoped semantic region → public-safe test id → CSS only for harness internals
```

That standard keeps Alfabet-style Playwright suites close to user risk while remaining stable enough for CI, useful during refactors, and safe to publish as portfolio work.
