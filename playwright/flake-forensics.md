# Playwright Flake Forensics for Alfabet-Style E2E Tests

Flaky E2E tests are expensive because they blur the line between product risk and test noise. In an Alfabet-style betting/trading UI, a failure might be caused by a legitimate stale-market bug, a late UI transition, a weak fixture, selector drift, or an unstable environment. This guide turns each repeat failure into evidence instead of guesswork.

This document is public-safe: it uses generalized product language, synthetic examples, and reusable Playwright investigation patterns. It does not include proprietary selectors, private URLs, credentials, traces, screenshots, HAR files, storage state, customer data, or internal API contracts.

## Forensics goal

Do not start flake triage by adding a timeout. Start by answering:

1. **What invariant was being protected?**
2. **What observable state did Playwright see?**
3. **Did the failure reproduce with the same seed, fixture, browser, and transition path?**
4. **Which failure class explains the evidence best?**
5. **What prevention makes the same class less likely next time?**

A useful flake investigation produces a small diagnosis, not just a green rerun.

## Flake taxonomy

| Class | Typical Alfabet-style symptom | Evidence to inspect | Better fix than waiting |
|---|---|---|---|
| Data freshness | selection price, market status, or event availability changes between action and assertion | sanitized network category, visible freshness label, fixture seed, trace timeline | deterministic synthetic transition or explicit stale/review invariant |
| Async UI propagation | status changed, but affordance, badge, or validation text updates one render later | trace snapshots, actionability log, console warnings, step timing | wait on a user-visible state boundary, not a fixed delay |
| Network timing | loading, retry, or recovered state races the assertion | response status class, duration bucket, retry count, loading indicator state | assert the recovery contract and use route-level fixture control |
| Animation/actionability | Playwright click or hover targets an element during motion or layout shift | actionability log, video/trace, layout shift clues | disable non-essential animation in test mode or click stable semantics |
| Selector drift | locator matches zero, too many, or the wrong repeated market row | locator strictness error, DOM snapshot, accessible names | prefer role/test semantics through an adapter; add uniqueness assertions |
| Auth/setup boundary | test starts in login, expired session, wrong account shell, or empty permissions | first trace step, storage-state age, setup project logs | isolate setup, validate preconditions, avoid committing auth state |
| Environment instability | browser crash, CI resource pressure, DNS/service outage, port conflict | runner logs, worker ID, machine metrics, retry pattern across specs | mark infrastructure separately; quarantine only with expiry and owner |
| Test order leakage | passes alone, fails after another spec mutates shared state | shard/order comparison, worker-scoped artifacts, fixture cleanup logs | make fixtures isolated and idempotent; reset synthetic world per test |
| Overbroad assertion | test expects exact copy/order/timing unrelated to the protected risk | failure diff, product intent, recent copy/layout changes | assert the invariant and accessible behavior, not incidental details |

## Decision tree

```text
failure observed
  ↓
Was the same risk/invariant named in the test output?
  ├─ no → improve annotation before changing behavior
  └─ yes
       ↓
Did retry fail at the same step with the same fixture/seed?
  ├─ yes → likely deterministic product, fixture, or assertion issue
  │        inspect trace + sanitized network + DOM snapshot
  └─ no
       ↓
Did the UI state eventually become correct after the assertion window?
  ├─ yes → async propagation, network timing, animation, or weak wait boundary
  └─ no
       ↓
Did setup/auth/navigation enter the expected public-safe starting state?
  ├─ no → setup boundary or test order leakage
  └─ yes
       ↓
Is the locator semantically unique and tied to user-visible intent?
  ├─ no → selector drift or overbroad selector
  └─ yes
       ↓
Check environment signals: browser crash, CI load, service outage, worker-only pattern
```

The tree is intentionally biased toward evidence. A timeout is only a valid fix after the class is known and the wait condition represents a real user-visible boundary.

## Minimum evidence bundle

For every recurring E2E flake, capture a sanitized bundle in the test report or investigation note:

- test title and risk annotation;
- browser/project, worker, retry attempt, shard, and CI/local marker;
- synthetic fixture profile and seed;
- final transition or last successful `test.step`;
- failed invariant;
- trace/video/screenshot availability, without committing artifacts;
- console summary after redaction;
- network category summary after redaction;
- first diagnosis and proposed prevention.

Example note:

```md
### Flake Forensics Note

- Risk: stale selection must require explicit review
- Area: market detail + betslip
- Fixture: synthetic-live-market-v1
- Seed: 18421
- Browser/project: chromium-desktop
- Retry pattern: failed once, passed on retry
- Last step: observed price movement before final action
- Failed invariant: primary action remained enabled while review was required
- Evidence: trace retained on failure; sanitized market-data category showed one slow response bucket
- Suspected class: async UI propagation or data freshness boundary
- Prevention: wait for visible review state and add a deterministic fixture transition for changed price
```

## Playwright investigation patterns

### Name the invariant inside the test

```ts
test('changed selection requires explicit review before continuation', async ({ page }, testInfo) => {
  testInfo.annotations.push(
    { type: 'risk', description: 'money-impacting stale selection' },
    { type: 'flake-class-watch', description: 'data freshness + async UI propagation' },
    { type: 'fixture', description: 'synthetic-live-market-v1' },
  );

  await test.step('open synthetic available market', async () => {
    // adapter action, public-safe
  });

  await test.step('force a changed-price transition', async () => {
    // deterministic synthetic transition, not a private API call
  });

  await test.step('assert review is required before continuing', async () => {
    // invariant assertion
  });
});
```

Annotations make the failure searchable. They also prevent a triage conversation from collapsing into "the click was flaky" when the real topic is stale-action safety.

### Wait on behavior, not time

Prefer waits that describe what a user can observe:

```ts
await expect(ui.reviewBanner()).toBeVisible();
await expect(ui.primaryAction()).toBeDisabled();
await expect(ui.selectionSummary()).toContainText(/review required/i);
```

Avoid waits that only describe elapsed time:

```ts
await page.waitForTimeout(1000); // hides the class of failure
```

If a fixed pause appears during triage, treat it as a diagnostic experiment only. Replace it with a state boundary before committing.

### Compare first attempt and retry

Retry behavior is signal:

| Pattern | Likely interpretation |
|---|---|
| same assertion fails on every attempt | deterministic product behavior, fixture setup, or wrong expectation |
| first attempt fails, retry passes | timing, async propagation, data freshness, environment, or order leakage |
| only one browser/project fails | browser-specific rendering, actionability, responsive layout, or feature support |
| only CI fails | resource pressure, service dependency, headless-only behavior, or missing setup |
| only after a preceding test | shared state, fixture cleanup, storage state, or order dependency |

Log the retry attempt and seed so the evidence survives outside the trace viewer.

## Class-specific playbooks

### Data freshness

Ask whether the test is asserting a state that can legitimately change while the user is on the page.

Checks:

- Did the synthetic world force the market or selection state deterministically?
- Did the UI show stale, suspended, changed, or review-required state before the assertion?
- Did the test assume a live value would remain stable without pinning the fixture?

Prevention:

- use synthetic fixture worlds for volatile states;
- encode stale/review behavior as an invariant;
- expose only sanitized network categories in reports;
- record fixture seed and transition list.

### Async UI propagation

Ask whether one part of the UI updated before another.

Checks:

- Did the market row show one state while the betslip action showed another?
- Did the trace show the correct UI a few frames after the failure?
- Was the assertion tied to a stable state boundary?

Prevention:

- assert the causal chain: status label, review message, disabled action;
- use `expect` auto-waiting on semantic locators;
- avoid reading raw DOM state before the user-visible state settles.

### Selector drift

Ask whether the locator still represents the user-visible thing the test cares about.

Checks:

- Is the locator role-based or adapter-backed?
- Does it match exactly one element in the relevant area?
- Did a repeated market row, responsive layout, or copy change make it ambiguous?

Prevention:

- centralize product-specific locators behind adapters;
- prefer roles, labels, and stable public-safe test semantics;
- add local uniqueness expectations around repeated lists.

### Auth and setup boundaries

Ask whether the test actually started in the intended world.

Checks:

- Was storage state created by a setup project in the same run?
- Did the first step validate the expected shell, account mode, or permissions?
- Did another test mutate shared state?

Prevention:

- keep setup isolated and observable;
- validate starting state before the risk flow;
- never commit storage state, cookies, tokens, or environment files.

## Flake issue template

Use this when turning a recurring failure into a tracked item:

```md
## E2E Flake Investigation

- Test:
- Risk protected:
- First seen:
- Browsers/projects affected:
- Retry pattern:
- Fixture profile + seed:
- Last successful step:
- Failed invariant:
- Suspected class:
- Evidence reviewed:
  - Trace:
  - Console summary:
  - Network category summary:
  - Setup logs:
- Decision:
  - Product bug / test design / fixture / selector / environment / unknown
- Prevention to add:
- Quarantine needed?
  - Owner:
  - Expiry date:
  - Re-enable condition:
```

Quarantine is a temporary containment tool, not a fix. Every quarantine should have an owner, expiry date, and re-enable condition.

## Definition of done

A flake investigation is complete when:

- the failure is classified using evidence;
- the protected Alfabet-style risk is still clear;
- the prevention is smaller than the problem;
- no private selectors, URLs, traces, screenshots, videos, storage state, credentials, or customer data are committed;
- the next failure of the same class will produce better diagnostics.

## Related lab notes

- [Playwright Observability Playbook](observability-playbook.md)
- [Playwright State-Model Testing](state-model-testing.md)
- [Alfabet Playwright Test Architecture Map](../alfabet/test-architecture-map.md)
