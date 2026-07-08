# Playwright Observability Playbook for Alfabet-Style E2E Tests

A useful Alfabet-style Playwright failure should explain more than "the button was not visible." It should preserve enough public-safe evidence to answer three questions quickly:

1. **What did the user see?**
2. **What did the browser observe?**
3. **Which product risk was protected by this assertion?**

This playbook describes a lightweight observability standard for betting/trading UI tests without exposing proprietary selectors, private API shapes, credentials, screenshots, traces, videos, storage state, or customer data.

## Evidence principles

| Principle | Practice | Why it matters |
|---|---|---|
| Capture behavior, not secrets | Record public-safe labels, generic route names, sanitized request categories, and invariant names | Keeps the repo useful without leaking internals |
| Prefer structured clues | Add test steps, annotations, and failure notes before relying on raw artifacts | Reviewers can triage failures before opening a trace |
| Keep artifacts scoped | Collect traces/videos/screenshots only for failures or focused debugging | Avoids noisy CI storage and accidental private data retention |
| Link evidence to risk | Name the money-impacting or safety invariant being protected | Prevents E2E tests from becoming decorative smoke checks |
| Make retry context explicit | Report attempt number, seed, fixture profile, and browser/project | Separates product defects from environment or data variance |

## Baseline Playwright configuration

For a public-safe lab, the default policy is evidence on failure, not evidence for every passing run.

```ts
// playwright.config.ts sketch — intentionally generic
export default defineConfig({
  retries: process.env.CI ? 1 : 0,
  use: {
    trace: 'retain-on-failure',
    video: 'retain-on-failure',
    screenshot: 'only-on-failure',
  },
  reporter: [
    ['list'],
    ['html', { open: 'never' }],
  ],
});
```

Do not commit generated `test-results/`, traces, videos, screenshots, HTML reports, storage state, HAR files, or raw console/network logs. Treat them as local or CI artifacts with retention rules, not repository content.

## Test-level annotations

Every deep E2E test should say what risk it covers. This makes reports searchable and helps failure triage start from intent.

```ts
test('selection review is required after observed price movement', async ({ page }, testInfo) => {
  testInfo.annotations.push(
    { type: 'risk', description: 'money-impacting stale price acceptance' },
    { type: 'area', description: 'market detail + betslip' },
    { type: 'fixture', description: 'synthetic-live-market-v1' },
  );

  await test.step('open a synthetic market with one available selection', async () => {
    // public-safe adapter action
  });

  await test.step('observe a price movement before final action', async () => {
    // sanitized network/UI observation
  });

  await test.step('assert the primary action requires explicit review', async () => {
    // invariant assertion
  });
});
```

A good annotation avoids private implementation detail. Prefer `market detail + betslip` over internal page names, and `synthetic-live-market-v1` over real fixture identifiers.

## Trace review checklist

When a failure retains a Playwright trace, inspect it in this order:

1. **Timeline:** Did the UI reach the intended state before the failing assertion?
2. **Actionability:** Did Playwright wait for visibility, stability, enabled state, and hit target correctly?
3. **DOM snapshot:** Is the expected user-visible state absent, hidden, duplicated, or late?
4. **Console:** Are there uncaught errors, hydration warnings, or client-side validation failures?
5. **Network:** Did a relevant request fail, lag, repeat, or return a stale category of response?
6. **Attachments:** Did the test include a custom note that names the invariant and fixture profile?

The trace should support a short diagnosis such as:

```text
Invariant: suspended market does not expose enabled primary action
Observed: market status changed to suspended, but the action remained enabled for one render cycle
Likely class: async UI/state propagation timing
Next step: add a deterministic fixture transition and assert disabled affordance after status observation
```

## Console evidence

Console logs are valuable only when they are filtered and summarized. A public-safe listener can classify messages without preserving sensitive payloads.

```ts
const consoleEvents: Array<{ type: string; text: string }> = [];

page.on('console', message => {
  const text = message.text();
  if (looksSensitive(text)) return;

  consoleEvents.push({
    type: message.type(),
    text: sanitizeForReport(text),
  });
});
```

Recommended classifications:

| Signal | Example interpretation |
|---|---|
| `error` | client exception, failed hydration, blocked script, unhandled promise |
| `warning` | deprecation, accessibility warning, suspicious recovery path |
| repeated info/debug | polling loop, duplicated render, noisy state sync |

Attach only sanitized summaries to reports. Avoid raw object dumps because they can contain URLs, tokens, account identifiers, or internal response fragments.

## Network evidence without leaking contracts

Network observation should answer product questions without publishing private API details. Store categories and timing, not endpoints or payloads.

```ts
const networkSummary = createNetworkSummary({
  redactUrl: true,
  redactHeaders: true,
  redactBodies: true,
});

page.on('requestfinished', async request => {
  networkSummary.record({
    category: classifyRequest(request), // e.g. 'market-data', 'navigation', 'static-asset'
    method: request.method(),
    status: request.response() ? (await request.response())?.status() : undefined,
    durationBucket: 'lt-500ms',
  });
});
```

Useful public-safe fields:

- request category (`market-data`, `bet-slip-read`, `navigation`, `static-asset`);
- method;
- response status class (`2xx`, `4xx`, `5xx`);
- duration bucket (`lt-500ms`, `500ms-2s`, `gt-2s`);
- retry count;
- whether the UI showed a loading, stale, recovered, or failed state.

Avoid publishing exact endpoint paths, query parameters, headers, response bodies, fixture IDs, customer IDs, auth state, or production hostnames.

## Failure note template

A failing test should produce a concise note that a reviewer can read before opening artifacts.

```md
### E2E Failure Note

- Area: market detail + betslip
- Risk: stale or unavailable selection allows unsafe action
- Fixture profile: synthetic-live-market-v1
- Seed: 18421
- Browser/project: chromium-desktop
- Last successful step: selection added to betslip
- Failed invariant: suspended selection disables primary action
- Evidence available: trace, screenshot, sanitized console summary, sanitized network summary
- First triage guess: async UI propagation or stale fixture transition
```

This note can be attached in Allure-style reporters, Playwright attachments, CI summaries, or issue comments. Keep it short and deliberately sanitized.

## Screenshot and video policy

Screenshots and videos are powerful but risky for public repositories. Use this policy:

- capture on failure only;
- never commit screenshots or videos;
- avoid using real accounts, real balances, real fixture names, or private URLs in captured sessions;
- prefer synthetic data with clearly fake teams, markets, and amounts;
- crop or mask if the CI/reporting system supports it;
- set retention windows for CI artifacts.

For public examples, describe the evidence shape instead of uploading real product images.

## What a useful failure should explain

A deep Alfabet-style E2E failure is useful when it identifies at least one of these:

| Question | Good failure evidence |
|---|---|
| Was the user blocked from unsafe action? | visible affordance state, invariant name, current market/betslip state |
| Was the data fresh enough for the assertion? | sanitized market-data category, status class, duration bucket, observed UI timestamp label |
| Did navigation preserve context? | route category, selected area label, recovery path, breadcrumb/menu state |
| Did validation help the user recover? | field-level message, action disabled/enabled state, clearing behavior |
| Is this likely a flaky test? | retry comparison, trace timing, console/network variance, seed stability |

## Triage loop

Use this loop before changing a selector or adding a timeout:

```text
failure appears
  ↓
read failure note and annotations
  ↓
open trace only if the note is insufficient
  ↓
classify failure: product behavior, test design, data/fixture, environment, or unknown
  ↓
if product behavior: preserve evidence and create a focused bug report
if test design: tighten invariant, adapter, or waiting condition
if data/fixture: make the synthetic world deterministic
if environment: quarantine only with an expiry and evidence link
  ↓
add prevention: assertion, fixture transition, helper, or flake-forensics entry
```

## Definition of done for observable tests

A new or changed Playwright test is observable enough when it has:

- a risk annotation;
- named `test.step` boundaries for setup, action, and invariant;
- deterministic synthetic or sanitized fixture context;
- trace/video/screenshot policy inherited from config;
- sanitized console and network summaries when relevant;
- a failure note that explains the protected behavior;
- no committed private artifacts or raw runtime evidence.

The goal is not to collect more artifacts. The goal is to make every failing Alfabet-style E2E test faster to understand, safer to share, and more directly connected to product risk.
