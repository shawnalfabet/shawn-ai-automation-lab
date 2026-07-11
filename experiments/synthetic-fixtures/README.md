# Synthetic Fixture Worlds for Alfabet-Style Playwright Tests

Synthetic fixtures make volatile betting/trading UI behavior testable without depending on production data, private APIs, or lucky timing. The goal is to model enough public-safe behavior to verify critical user-facing invariants: market availability, changing prices, disabled actions, validation, recovery, and evidence capture.

This experiment is public-safe by design. It uses generalized terms and harness patterns only. It does not include proprietary selectors, internal URLs, credentials, environment variables, real fixtures, traces, screenshots, customer data, or private network contracts.

## Why synthetic worlds matter

Live market interfaces are hard to test because the state can move while the browser is asserting it. A useful synthetic fixture world gives the test author controlled transitions instead of hoping the real system is in the right moment.

| Problem in live E2E | Synthetic-world response | Test value |
|---|---|---|
| Available selection becomes suspended mid-flow | Schedule a named `available → suspended` transition | Verify disabled actions and recovery copy |
| Price changes between selection and final action | Emit a deterministic `priceChanged` event | Verify explicit review before continuation |
| Validation depends on stake, limits, or market status | Model validation outcomes as state rules | Assert user-facing prevention, not backend details |
| Network timing makes assertions racey | Delay by category and bucket, not endpoint | Exercise loading/recovered states safely |
| Flakes are hard to reproduce | Record fixture profile, seed, and transition list | Re-run the same world locally or in CI |

The point is not to replace integrated tests. It is to make the riskiest UI states observable, repeatable, and explainable.

## Fixture world contract

A synthetic world should expose a small public-safe contract:

```ts
type FixtureWorld = {
  profile: 'synthetic-live-market-v1';
  seed: number;
  startingState: 'available' | 'suspended' | 'closed';
  transitions: Array<{
    atStep: string;
    event: 'priceChanged' | 'marketSuspended' | 'marketRecovered' | 'validationRejected';
    userVisibleOutcome: string;
  }>;
};
```

The contract intentionally names user-visible outcomes instead of private API responses. A test should be able to say, "after this transition, the UI must require review," without exposing how the application receives the event.

## Harness sketch

```ts
import { test, expect } from '@playwright/test';

test('changed selection requires explicit review before continuation', async ({ page }, testInfo) => {
  const world = createSyntheticWorld({
    profile: 'synthetic-live-market-v1',
    seed: 41207,
    startingState: 'available',
    transitions: [
      {
        atStep: 'after-selection-added',
        event: 'priceChanged',
        userVisibleOutcome: 'review required before primary action',
      },
    ],
  });

  testInfo.annotations.push(
    { type: 'fixture-profile', description: world.profile },
    { type: 'fixture-seed', description: String(world.seed) },
    { type: 'risk', description: 'money-impacting stale selection review' },
  );

  await test.step('open synthetic market in available state', async () => {
    await world.route(page);
    await page.goto(world.publicSafeEntryPath());
  });

  await test.step('add selection and trigger deterministic price movement', async () => {
    await marketPage(page).selectFirstAvailableSelection();
    await world.transition('after-selection-added');
  });

  await test.step('assert the UI requires explicit review', async () => {
    await expect(betslip(page).reviewMessage()).toBeVisible();
    await expect(betslip(page).primaryAction()).toBeDisabled();
  });
});
```

The example is deliberately adapter-based. `marketPage`, `betslip`, `createSyntheticWorld`, and route details are placeholders for public-safe boundaries, not exported product selectors.

## Design rules

1. **Name the invariant first.** Start with the user safety property, such as "stale selection cannot continue without review."
2. **Keep fixture data synthetic.** Use fictional event names, market labels, and account states.
3. **Route by category.** Prefer categories like `market-data` or `validation` in documentation; do not publish private endpoint paths.
4. **Make transitions explicit.** A fixture world should list the state changes the test depends on.
5. **Record the seed.** If the world uses randomness, the seed belongs in test annotations and failure notes.
6. **Assert visible behavior.** Verify disabled affordances, review copy, loading states, and recovery messages.
7. **Do not commit artifacts.** Traces, videos, screenshots, HAR files, storage state, and generated reports stay out of the repo.

## Transition matrix

| Transition | Starting state | Expected user-visible invariant | Evidence to capture on failure |
|---|---|---|---|
| `priceChanged` | selected available item | review message visible; primary action disabled until acknowledgement | fixture seed, transition name, visible summary text category |
| `marketSuspended` | market detail open | selection controls unavailable; existing selection marked unavailable | final step, status label category, actionability log |
| `marketRecovered` | suspended market | recovered status visible before selection is re-enabled | transition list, loading/recovered state timing |
| `validationRejected` | stake entered | validation message shown; unsafe submission prevented | validation category, user-visible message class |
| `navigationInterrupted` | moving between market and slip | no duplicated selection or stale enabled action | trace retained on failure, page object state summary |

## Failure note template

```md
### Synthetic Fixture Failure Note

- Fixture profile: synthetic-live-market-v1
- Seed: 41207
- Transition list: priceChanged after-selection-added
- Protected invariant: stale selection requires explicit review
- Observed UI state: review message absent; primary action remained enabled
- Evidence retained: trace/video on failure, sanitized network categories
- Suspected class: async UI propagation or fixture transition bug
- Prevention candidate: assert intermediate changed-price state before final action
```

## When not to use a synthetic world

Synthetic worlds are a tool, not a blanket replacement for integration coverage. Keep at least a small number of true end-to-end checks for authentication, routing, and real service compatibility in private systems where that is appropriate. Use synthetic worlds when the goal is to exercise rare, risky, or timing-sensitive UI states repeatedly and safely.

A strong Alfabet-style Playwright suite uses both: integrated confidence for the happy path and synthetic determinism for the edge states that protect users from stale or unsafe actions.
