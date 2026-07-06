# Alfabet Playwright Testing Lab Manifesto

This repo is a public R&D lab for testing Alfabet-style betting and trading interfaces with Playwright.

The point is not to collect shallow smoke tests. The point is to explore how a serious E2E practice can make a high-change, real-time product safer without making the test suite slow, brittle, or impossible to maintain.

## What makes Alfabet-style testing hard

Betting/trading frontends are not normal CRUD apps. They have product behavior that changes while the user is looking at it:

- live markets appear, suspend, update, and disappear;
- prices and statuses can change between click and assertion;
- navigation depends on sport, region, league, event, market type, and viewport;
- some UI states are money-impacting and must not be tested casually;
- mobile and desktop often expose the same domain through different interaction models;
- flakiness can come from data freshness, network timing, animations, feature flags, or real-time feeds.

A good test lab should treat these as first-class design constraints.

## Lab philosophy

### 1. Test behaviors, not implementation details

A test should explain the behavior it protects:

```text
when a market is unavailable → the user cannot place an action against stale state
```

Not:

```text
click the third div in the second panel
```

### 2. Keep public examples public-safe

This repo may mention Alfabet as the testing context, but examples must avoid private selectors, private URLs, credentials, screenshots, traces, and implementation details.

### 3. Make the hard parts visible

The interesting work is not just writing `.spec.ts` files. It is designing:

- stable selectors;
- synthetic test worlds;
- state models;
- observability hooks;
- flake triage loops;
- risk-based coverage;
- safe boundaries for write flows;
- agent-assisted planning that still commits normal Playwright code.

### 4. Prefer small artifacts with strong ideas

Each contribution should add a reusable idea:

- a test architecture note;
- a model of a tricky state machine;
- a flake-forensics playbook;
- a public-safe example spec;
- a template that improves test reviews;
- a checklist that prevents bad automation.

### 5. Separate R&D from production code

The lab can be creative. Production tests should be boring:

```text
creative planning → normal Playwright test → deterministic CI signal
```

No runtime AI dependency. No hidden magic. No test nobody can review.

## Deep workstreams

| Workstream | Question |
|---|---|
| State-model testing | Can we model betslip, market, and navigation state transitions? |
| Synthetic fixtures | Can we build deterministic test worlds without real customer data? |
| Observability | Can a failed test explain what changed: UI, network, console, state, timing? |
| Flake forensics | Can failures be classified and fixed systematically? |
| Risk-based coverage | Which UI behaviors deserve E2E protection because they are money-impacting? |
| Agent-assisted test design | Can agents draft plans/tests while humans keep the committed code boring? |
| Visual semantics | Can we test visible meaning without brittle screenshots? |

## Success criteria

This repo is working if it helps answer:

- What should we test before release?
- What should we avoid testing through E2E?
- How do we know a failure is real?
- How do we keep tests stable as the product changes?
- How do we make automation creative without making CI weird?
